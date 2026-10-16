参照 `73379762` 中 UB 隔离方案的分析方式，负载感知调度在 Mooncake 中应形成这条链路：**请求侧采集成功 READ 耗时 → Master 维护节点成本 → Master 在新写入分配时消费；健康状态复用隔离模块。** 注释 1

这里的“调度”主要是**选择数据放置节点**。首期不改 SchedulerCore 的任务公平调度，也不调整 Jetty 池大小。

**1．先对应 yuanrong 与 Mooncake 的角色变化**

| 环节 | yuanrong 参考方案 | Mooncake 接入方式 |
| --- | --- | --- |
| 读服务耗时采集 | Worker 执行 GET 的 UB WRITE 回写，Worker 计时 | Client 执行远端内存 READ，在实际 TE 执行端计时 |
| 性能反馈 | Publish 响应反馈统计 | 新增异步样本控制通道，报告给 Master |
| 成本维护和选择 | Client 收到反馈后选择 Worker | Master 汇总目标节点成本，选择新写入目标 |
| 端口健康输入 | Worker 健康视图 | 复用 `NodeHealthRegistry` 的确认事实 |
| 实际传输 | 选择 Worker 后提交 | Master 返回副本地址，Client 通过 TE WRITE |

因此，不能把 yuanrong 的 Client 调度器直接放进 Mooncake Client。**普通 PUT 的目标空间由 Master 分配，性能选点的主要决策位置应在 Master。**

**2．谁维护状态，消费者是谁**

| 拟新增或复用组件 | 所在层 | 维护内容 | 消费者 |
| --- | --- | --- | --- |
| `ReadTransferObserver` | TE 传输执行层 | 单次 READ 组的开始、硬件完成、字节数和目标身份 | Reporter |
| `ReadLatencyReporter` | 请求侧 Client/实际 TE 执行进程 | 有界待上报样本 | Master Tracker |
| `NodeReadLatencyTracker` | 每个 Master | 各节点有界窗口、归一化 P50/P99、统计版本 | 放置适配器 |
| `NodeHealthRegistry` | 复用故障隔离模块 | 节点准入、端口健康比例、身份代际 | 放置适配器和硬过滤 |
| `LatencyHealthWeightPolicy` | 公共纯策略组件 | 成本、接受率和有界选择算法；不拥有健康事实 | 放置适配器 |
| `NodePlacementAdapter` | Master 分配层 | 当前请求的候选快照、排除集合及选择过程 | 实际 allocator |

状态所有权应保持清晰：**Tracker 不决定隔离，Registry 不计算时延，Policy 不调用 provider、RPC 或 allocator。**

**3．采样接入点：在实际 UB READ 完成链上**

业务来源可从 [Client::TransferRead](D:/Lingqu/Supercache/mooncake-store/src/client_service.cpp:4385) 识别，传输组与目标关联可在 [TransferSubmitter::submitTransfer](D:/Lingqu/Supercache/mooncake-store/src/transfer_task.cpp:1231) 附近建立，但精确计时必须落到 UB post/CQE 层：

| 接入位置 | 需要做什么 |
| --- | --- |
| TransferSubmitter / operation state | 建立 READ 样本组，绑定目标节点身份、字节数和操作 owner |
| UB 实际 post 路径 | 首个确认接受的 WR 记录开始时间，不能用上层 submit 时间代替 |
| [UrmaContext::poll](D:/Lingqu/Supercache/mooncake-transfer-engine/src/transport/kunpeng_transport/urma/urma_endpoint.cpp:636) | 记录最后一个硬件完成时间；在发布任务完成前复制必要信息 |
| 异步 Reporter | 将有效样本放入有界队列，批量上报，不阻塞 CQE 和业务完成 |

样本口径沿用现有方案：

- 成功 UB READ 组，至少 256 KiB，归一化到 4 MiB。
- 从 first accepted post 到 last hardware completion。
- 排除 Master RPC、提交前排队、H2D、失败、重试及 partial post 样本。
- 不在 `markSuccess()` 后继续访问可能已回收的 Slice/Task。

首期应明确限定为**远端内存读服务样本**。SSD/offload、warmup、后台复制不能仅因底层也是 READ 就混入，否则同一个 P99 会混合不同服务语义。

**4．决策接入点：Master 分配前，而不是分配后**

主要入口是 [MasterService::PutStart](D:/Lingqu/Supercache/mooncake-store/src/master_service.cpp:5445)，实际空间分配集中在 [AllocateAndInsertMetadata](D:/Lingqu/Supercache/mooncake-store/src/master_service.cpp:5164) 及其调用的 `AllocationStrategy::Allocate()`。

推荐流程：

```text
确认请求属于首期范围
  → 原策略产生首选和合法候选
  → 检查原首选是否有已知坏端口
  → 无坏口/健康未知：保持原策略
  → 有坏口：读取固定健康、成本快照，执行加权选择
  → 复查身份、硬准入和副本约束
  → 实际分配空间
```

**当前代码这里存在一个必须处理的结构问题：选点和分配耦合。**

[AllocationStrategy](D:/Lingqu/Supercache/mooncake-store/include/allocation_strategy.h:195) 直接返回已分配的 Replica；部分策略在遍历候选时立即申请内存。因此不能简单在 `PutStart` 调用 `Allocate()` 后重排结果，也不能先分配再撤销来模拟选择。

更合理的调整是：**在分配策略内部，把“产生候选”和“实际 allocate”之间留出一个小型选择接口**，由 `NodePlacementAdapter` 接入。原有容量检查、副本分布和分配失败回退继续保留。

另一个重要边界是：**成本按节点维护，而 allocator 面向 segment。** 同一进程挂载多个 segment 时，应先按节点选择，再在该节点内按原策略分配，避免“segment 越多，抽中概率越高”的意外偏置。

**5．消费者如何使用健康和成本**

首选存在已知坏口时，候选接受率仍按现有文档计算：

```text
C = (P50 + 3 × P99) / 4 + 非原首选惩罚
h = good / total
a = R / (R + C) × h

非原首选惩罚 = 200 us
R = 1000 us
```

P99 门限、防饥饿和有界抽样继续保留。这里的顺序是：

1. 隔离模块提供硬准入，确认隔离的节点不可选。
2. 部分端口故障触发性能选点。
3. 时延成本和健康比例共同影响候选接受率。
4. 实际分配前再次检查身份和可用性。

四端口域和共享 context 的计数应直接复用健康模块，不能另建一套端口监控。高时延也不能反过来制造节点隔离。

**6．哪些入口首期启用，哪些只做健康过滤**

| 入口 | 首期处理 |
| --- | --- |
| 普通单键 UB MEMORY 新写入放置 | 启用成本加权 |
| Client GET 副本选择 | 原策略＋健康硬过滤，负责产生有效读样本 |
| 批量写入 | 原策略＋健康过滤 |
| Copy/Move、Promotion、硬目标 AllocateFrom | 保留原选择约束，不自动启用性能加权 |
| UB context/HCA/Jetty 选择 | 保留设备策略和资源约束 |
| CVM 远端分配 | 若属于首期请求，由实际分配的目的 Master 本地执行 |
| HA 接管 | 成本窗口作为可重建软状态；初始化成本不能解除健康围栏 |

实现时尤其需要注意：**Upsert 和其他入口也会复用 `AllocateAndInsertMetadata()`，不能在公共函数里无条件启用新算法。** 应传入明确的内部放置上下文，限定操作范围，并携带请求超时、目标协议和亲和信息；跨 Master 转发也要保留该上下文。

整体上，公共组件只需覆盖**样本契约、节点成本快照和纯选择策略**。实际接入集中在 TE 完成观测、控制上报及 Master 分配三个位置，不需要把整个 TE 或所有业务操作都改成新的调度框架。

本次为接入分析，未修改文档或代码。