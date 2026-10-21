# TENT 直接使用物理 UB 设备的可靠性分析与增强设计

> 文档性质：基于源码的现状说明和待实现设计。本文不代表下述增强已经合入，也不代表已经通过真实 UB 硬件验证。
>
> 分析时间：2026-10-21。
>
> Supercache 基线：`main_learn`，`a6f6672974b553a401e5c6f54599c04cd91ee94d`。
>
> 同级 UMDK 基线：`09086da5d86c53f83eb395109d636957d3efcc79`。
>
> 范围：TENT 原生 `UBTransport`、CPU DRAM、单边 READ/WRITE。Classic TE、SEND/RECV、原子操作和 GPU 内存不在本次可靠性承诺范围内。

## 1. 结论与目标边界

TENT 已经在 Transport 层实现多设备资源管理、内存注册与导入、路径调度、配额、分片重试、完成聚合及失败资源隔离。因此，去掉 bonding 的正确方向，是在现有机制上补充物理设备路径管理和故障恢复闭环，而不是在 TENT 中重新实现一个 bonding provider。

当前代码存在选择物理设备的入口，但“能够选择物理设备并下发 READ/WRITE”还不能等同于“与使用 bonding 时具有相同的可用性”。主要缺口是：

1. 物理 EID 之间的可达性、传输参数和故障域没有形成完整模型。
2. 健康恢复主要依靠流量反馈、JFC poll 和冷却时间，缺少真实数据面的主动探测。
3. 致命设备故障后的 Context 重建、重新注册和元数据发布没有完整闭环。
4. 重试目前主要依赖粗粒度错误类别，没有明确区分未执行、结果未知和是否允许安全重放。
5. 资源争用和候选 Endpoint 失效的部分路径可能反复 defer，缺少统一请求期限。
6. 真实 UDMA 的排空语义、跨设备恢复和上层业务语义仍需硬件与故障注入验证。

“可靠性不劣化”需要同时满足以下三个维度：

| 维度 | 应满足的条件 |
| --- | --- |
| 内存与结果安全 | 无释放后 DMA、无重复终结/配额释放、无旧重试覆盖新数据、不把结果未知报告为确定未执行 |
| 可用性与进展 | 在相同可独立失效资源条件下，有等价备用路径、故障检测和恢复能力；请求不会无限等待 |
| 业务契约 | 两组方案使用相同的源数据稳定性、目的区域独占、发布规则和顺序要求 |

两条 EID 如果落在同一物理设备或同一故障域，不能作为两条独立高可用路径。只有一条物理路径时，可以保证安全失败，但无法凭软件恢复成多设备 bonding 的故障覆盖范围。

UMDK bonding 自己也没有提供通用的 exactly-once 或跨队列严格顺序保证。本文要求把业务重试契约说清楚，不把完整事务协议作为所有 READ/WRITE 场景的强制前提。

## 2. 源码依据与引用方式

本仓库链接使用相对路径；UMDK 以同级仓库中的相对文件路径和函数定位。UMDK 不属于本仓库，离开双仓库工作区后，应在上面记录的 UMDK 提交中查找对应文件。

| 编号 | 源码入口 | 主要内容 |
| --- | --- | --- |
| T1 | [device_selection.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/device_selection.h)、[ub_transport.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/ub_transport.cpp) | bonding 优先选择、设备过滤、初始化、Bootstrap |
| T2 | [urma_adapter.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/urma_adapter.cpp) | 原生 URMA 资源、WR、CR 分类、绑定和排空 |
| T3 | [buffers.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/buffers.cpp)、[buffers.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/buffers.h) | 多 Context 注册、远端导入和代际缓存 |
| T4 | [workers.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/workers.cpp)、[workers.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/workers.h) | 选路、提交、完成、超时和 defer |
| T5 | [quota.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/quota.cpp)、[rail_monitor.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/rail_monitor.cpp) | 物理路径配额、带宽与错误反馈 |
| T6 | [endpoint.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/endpoint.cpp)、[endpoint_store.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/endpoint_store.cpp) | Endpoint 状态机、代际、淘汰和隔离 |
| T7 | [context.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/context.cpp)、[slice.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/slice.h) | Context 恢复条件、分片尝试与任务终结 |
| T8 | [control_plane.h](../../mooncake-transfer-engine/tent/include/tent/runtime/control_plane.h)、[topology.cpp](../../mooncake-transfer-engine/tent/src/runtime/topology.cpp) | UB 专用握手、EID 拓扑和 NUMA 排序 |
| T9 | [transfer_engine_impl.cpp](../../mooncake-transfer-engine/tent/src/runtime/transfer_engine_impl.cpp)、[admission_queue.cpp](../../mooncake-transfer-engine/tent/src/runtime/admission_queue.cpp) | Transport 失效转移、全局准入与期限 |
| T10 | [params.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/params.h) | 当前实际支持的配置 |
| U1 | `src/urma/hw/udma/udma_u_ops.c` | 物理 provider 完整操作表 |
| U2 | `src/urma/hw/udma/udma_u_jfs.c`：`udma_u_post_sq_wr` | 部分提交、队列锁、WQE/doorbell |
| U3 | `src/urma/hw/udma/udma_u_jetty.c`：`udma_u_bind_jetty_ex` | RC 绑定和 EEXIST |
| U4 | `src/urma/lib/urma/bond/bondp_datapath_schedule.c`：`schedule_send_balance` | bonding 路径调度 |
| U5 | `src/urma/lib/urma/bond/bondp_dp_health.c`、`src/urma/lib/urma/bond/bondp_env.c` | 主动健康探测、默认开关 |
| U6 | `src/urma/lib/urma/bond/bondp_datapath.c` | 故障重发、完成去重、顺序边界 |
| U7 | `src/urma/lib/urma/core/include/urma_opcode.h`、`src/urma/lib/urma/core/urma_main.c` | CR 状态、聚合专用选项 |
| U8 | `doc/ch/urma/URMA User Guide.ch.md`、`doc/ch/urma/URMA API Guide.ch.md` | EID 可达性、模式与排序说明 |

辅助学习材料：[TENT 四项核心能力](../../learn/tent-four-core-capabilities.md)。

## 3. TENT 已有能力：实现原理与适用边界

### 3.1 统一逻辑对象与多设备资源展开

应用提交的目标是 Segment 和内存区间。UBTransport 将逻辑请求映射到具体设备 Context、Endpoint 和远端注册 Segment，应用不需要持有 UDMA Jetty 等原生对象。

初始化时，TENT 为选中的设备/EID 建立 Context 和 JFC，创建 BufferRegistry、EndpointStore、RailMonitor、QuotaManager 和 Workers。每个原生句柄仍属于自己的 Context，不是把一个设备的句柄直接复用到另一设备。

当前发现逻辑会枚举设备的 EID 列表，并将不同 EID 表示为不同 NIC。默认过滤为空时，如果发现 bonding 设备，会优先只选 bonding；显式 `device_filter` 可以关闭这种自动优先行为。过滤器可匹配设备名，也可匹配 `ub:<native>:eid<index>` 形式的拓扑名。（T1、T2、T8）

其价值在于：应用的逻辑 Segment 不必随实际选用的设备改变。需要增强的是物理身份和可达性，而不是再设计一个虚拟 bonding EID。

边界：当前“设备活跃”判断是设备有任一端口活跃且 EID 非零，甚至 `port_cnt == 0` 时也会通过设备活跃判断。这不证明所选 EID 到某个远端 EID 的数据路径可达。

### 3.2 多 Context 内存注册与远端导入缓存

BufferRegistry 为同一应用缓冲区维护不同 Context 对应的本地注册资源。路径确定后，`findLocal` 找到该路径可用的本地 Segment，`importRemote` 在选中的本地 Context 中导入远端 Segment。（T3）

导入缓存的区分信息包括：

- 本地 topology ID。
- 远端逻辑 Segment ID。
- 远端 topology ID。
- 远端 buffer base。
- 远端注册 generation。

这样可以避免把某次远端注册的导入句柄误用于另一次注册。元数据还检查版本、权限、地址范围和相关 ABI 信息；失败清理不能完成时，保留资源以继续重试清理。

Adapter 提交前进一步检查 Jetty、本地 Segment 和导入 Segment 属于同一个本地 Context，并检查访问范围。成功提交前持有 Segment 引用，未被接受的 WR 才释放相应引用。（T2）

这已经覆盖 bonding “虚拟 Segment 展开到多设备”中 TENT 所需的主要职责。直接物理 UB 不需要另补一层聚合 Segment 缓存。

边界：远端 generation 有助于拒绝陈旧注册资源，但不是端到端操作编号，也不能使一次 WRITE 自动具备 exactly-once 语义。设备重建后新增注册、旧导入撤销和新元数据发布仍需统一编排。

### 3.3 多路径选择、负载反馈与物理配额

Workers 先依据本地内存位置、设备 mask、活跃 Context 和 NUMA 排名构造本地设备候选，再从目标 Segment 的已注册 UB 内存中构造远端设备候选。（T4）

对本地和远端组合，现有评分使用配额是否可用、拓扑排名、归一化在途负载、Endpoint 在途 WR/字节数以及带宽估计等信息；重试次数也影响候选顺序。确定候选后，由 QuotaManager 原子预留设备和物理 rail 的资源。

配额使用唯一 reservation 标识，重复释放不会重复扣减。物理 rail 配额不因为 Endpoint 换了 generation 就凭空重置，避免旧请求仍在途时新连接绕过容量限制。（T4、T5）

这是应用层已有的多设备调度能力。没有必要为了去 bonding，复制 UMDK 的另一套调度算法。UMDK Balance 当前以较少 SQE 的路径为候选，同负载时随机选择，并非固定轮询。（U4）

边界有两点：

1. 当前候选主要由 NIC 元数据、NUMA 和资源状况推导，没有完整的“这个本端 EID 可以到这个对端 EID”的证明。
2. TENT 在 bonding Context 上设置的是 `STANDALONE + IODIE`。验收时必须比较实际配置，不能把 bonding provider 所有模式的能力都算作当前 TENT 的默认基线。（T2）

### 3.4 UB 专用建链、Endpoint 代际与复用

TENT 通过 IP 控制面传递 `UbBootstrapDesc`，交换 EID、Jetty ID、UASID、Endpoint generation 和 Segment generation 等信息，导入并绑定对端 Jetty。（T1、T8）

Endpoint 状态包括准备、握手、绑定、Ready、失败和销毁等阶段。失效实例进入退休流程，新的连接使用新的 generation；Store 按本地 NIC、远端 Segment/NIC 等信息复用 Endpoint。（T6）

因此，直接使用物理 UB 时，不需要仅为元数据交换再实现 URMA public Jetty 自举。

边界：当前 Bootstrap 解析仅接受协议版本 1；虽然有 `capabilities` 字段，但没有完整协商物理路径的 `trans_mode / tp_type / priority / ordering`。缓存中的 Ready Endpoint 直接复用也不能替代对端重启通知和代际失效机制。

### 3.5 分片、原生下发和部分提交处理

请求拆为 UbSlice，每个尝试拥有自己的 attempt 标识。初始化会根据设备能力限制消息长度、READ/WRITE 长度、Jetty 深度和 SGE 数量。（T1、T7）

真实 Adapter 生成普通 URMA READ/WRITE WR，设置 completion 和数值 token，使用本地与远端 SGE，然后调用 `urma_post_jetty_send_wr`。应用不需要直接编写 WQE 或敲 doorbell；这些由 UDMA provider 完成。（T2、U1、U2）

UDMA 批量 post 发生错误时，前缀 WR 可能已被硬件接受，`bad_wr` 指向首个未接受项。TENT Adapter 会计算已接受数量；当前 Worker 每次下发一个 WR，若 `posted_count == 1`，即使调用同时返回错误，也按已提交等待完成，不盲目重投。（T2、T4）

这项能力应原样保留。未来批量优化必须继续遵守“已接受前缀/未接受后缀”的契约。

### 3.6 完成聚合与重复、迟到完成隔离

提交时，Inflight 记录数值 completion token、attempt、Endpoint、Segment 句柄、配额和截止时间。原生 CR 通过 token 找到对应记录；不存在的旧 token 不再推进当前任务。（T4）

资源释放使用原子一次性标志；Slice 在互斥保护下核对 attempt，保证一个逻辑尝试只终结一次，最后再聚合为 Task 状态。（T4、T7）

这已承担 bonding 中 TENT 所需的完成归并和用户上下文恢复职责，无需再做虚拟 JFC ID 重写。

边界：本地完成去重只消除重复记账和重复回调，不能阻止远端已执行 WRITE 在重试时再次执行。

### 3.7 超时后的排空、资源保留与隔离

超时不是立即释放内存的依据。现有流程先将请求标记为超时，对该 Endpoint 的 Jetty 执行 quiesce，处理排空期间返回的真实完成，然后决定是否能回收和重试。（T4）

真实 Adapter 大致执行：

1. 与 JFC poll、Jetty post 同步。
2. 将 Jetty 置 ERROR。
3. 等待本 Jetty 的 `URMA_CR_WR_FLUSH_ERR_DONE` 标记。
4. 使用 `urma_flush_jetty` 排空未处理项。
5. 达到 fence 条件后，才把该 Jetty 视作已排空。（T2）

如果 fence 失败，代码保留 Inflight、Endpoint 和 Segment 引用，后续继续尝试排空，或等待自然完成；不会仅凭“CQ 暂时为空”释放资源。Endpoint 清理失败也会进入隔离集合，继续占用资源预算。（T4、T6）

这是最需要保留的安全机制。直接物理设备的可靠性设计应补充 provider 语义验证，而不是删除排空以缩短超时。

边界：软件执行了上述步骤，不等于已经证明所有硬件/驱动版本都满足所需 DMA 与远端副作用隔离语义。能否安全复用本地地址、重放远端 WRITE，必须分别验证。

### 3.8 基础故障反馈与重试框架

当前 CR 分类区分 SUCCESS、TIMEOUT、本地设备错误、远端路径错误、Endpoint 错误和内存错误。Worker 对可重试类别退休 Endpoint、更新 rail 统计，并由 Slice 预算决定后续尝试。（T2、T4）

本地设备错误会暂停对应 Context；其恢复要求故障清理完成、无在途 WR、所有 JFC 在相应失败 epoch 下 poll 成功，并经过冷却期。RailMonitor 会统计错误与超时，达到阈值后暂停路径。（T5、T7）

这些机制可复用，但目前并不是完整主动健康检查：

- 成功 poll 可以返回 0 条完成，不能证明数据链路可达。
- rail 冷却到期会解除暂停，不等于真实 half-open 探测成功。
- 一个 EID 对应 Context 失效时，没有自动按底层物理设备归并所有相关 EID。
- Context 恢复可能复用旧原生 Context，没有通用致命故障重建流程。

## 4. 需要增强的现状与优先级

本文 P0 表示物理 UB 高可用方案投产前的安全/语义门槛；P1 表示完成故障覆盖和恢复闭环所必需；P2 表示按部署条件或性能目标启用。

| 编号 | 现状或缺口 | 影响与判断 | 优先级 |
| --- | --- | --- | --- |
| G1 | NIC 活跃与 NUMA 排名不能证明 peer-specific 可达性 | 跨 plane、port EID 或错误 profile 可反复建链失败 | P0 |
| G2 | 固定 RC、导入固定 CTP，物理设备优先级默认值；无双端 profile 协商 | 参数合法性和可用性不能由 bonding 路径的成功推导 | P0 |
| G3 | bind 将 EEXIST 视作成功 | UDMA 中也可能表示 Jetty 已绑定另一目标，需核对身份 | P0 |
| G4 | 重试未细分结果未知、永久权限错误和 DMA 状态 | WRITE 可能重复/覆盖，权限错误可能被当路径故障重试 | P0 |
| G5 | 真实物理 provider 的 fence 边界缺硬件证明 | 错误释放、旧 DMA 与新尝试重叠是内存安全风险 | P0 |
| G6 | 候选 Endpoint 无调度租约；资源压力下反复 defer | 小缓存下可能持续选到已淘汰对象；不是断言每次必现 | P0 |
| G7 | 缺主动数据面探测和 async event 闭环 | 空闲断链、单向故障发现慢，回切缺证据 | P1 |
| G8 | 缺故障域模型和致命 Context 重建闭环 | 同设备多 EID 误作备用，无法可靠热恢复 | P1 |
| G9 | 排队、配额等待等路径缺统一 deadline | 单次 WR 有超时，整个请求仍可能长期不终结 | P1 |
| G10 | Runtime 仅对 FAILED 进入既有 failover，TIMEOUT 不同 | 不能假定 UB 超时会自动切到其他 Transport | P1；启用跨 Transport 恢复时 |
| G11 | 准入预测使用跨 Transport 的 dispatching bytes | 混合 RDMA/UB 且启用期限降级时，估时可能失真 | P2；对应部署启用时提升为必修 |

G3 依据 U3：相同绑定目标可返回成功，但已绑定不同目标会返回 `URMA_EEXIST`。不能把运行库初始化时的 EEXIST 语义照搬到 Jetty bind。

TENT 的 `RealJetty::bind` 已经在互斥锁下检查已记录的 EID、Jetty ID 和 UASID，相同目标复用、不同目标拒绝。因此 G3 是原生返回码处理与异常状态一致性的加固项，不能据此断言正常重绑流程必然绑定错误目标。

G4 依据 T2：当前 `classifyCompletion` 将远端权限/操作/不支持请求等多个状态放入可重试的 REMOTE_PATH_ERROR。其具体可重试性必须结合原始 CR、opcode 和 metadata generation 判断。

G6 依据 T4、T6：候选上限至少为 2，而允许 `max_endpoints = 1`。构造候选 B 可能淘汰尚无在途 WR 的 A，保留在候选数组中的 A 随后申请 outstanding 失败，进入不消耗错误重试预算的 defer。需要回归测试验证调度结果与终结边界。

## 5. 增强设计总览

保留现有 BufferRegistry、Workers、QuotaManager、EndpointStore 和 Adapter 分工，新增物理路径管理与恢复编排。下面所有新类型、状态、接口、字段及配置均为建议，当前分支尚未实现。

```text
ControlPlane / Metadata
  ├─ 身份、profile、注册 generation、探测资源描述
  └─ peer restart / generation 变化
                 |
PhysicalPathManager
  ├─ ConnectivityTable：允许通信的 EID 对
  ├─ HealthManager：事件、探测、half-open
  └─ RecoveryCoordinator：故障域隔离、排空、重建和发布
                 |
Workers + EndpointStore + QuotaManager + BufferRegistry
                 |
UrmaAdapter：原生查询、事件、创建、提交、完成、fence
                 |
UDMA / 物理 UB 设备
```

PathManager 只向 Workers 提供带版本的路径快照和候选租约。Adapter 不做跨设备选路；控制面不直接调用持锁的原生销毁流程。

可靠性不变量：

1. 被接受的 WR 在获得可靠完成或合格 fence 前，保留其资源与必要的应用缓冲区使用约束。
2. 每个 attempt 至多一次终结；每个 reservation 至多一次释放。
3. 只有身份、generation、profile 和健康状态匹配的路径可以提交。
4. 旧 epoch 的事件、探测和完成不能把新实例判定为健康或释放新实例资源。
5. 未知结果不能被包装成“确定未执行”。
6. 对外释放缓冲区使用权的时间点，必须有明确可验证的安全依据。

## 6. 物理身份、可达性和 profile 设计

### 6.1 身份模型

建议新增以下逻辑数据；具体 C++ 存储可沿用现有类型：

| 类型 | 字段 | 目的 |
| --- | --- | --- |
| DeviceIdentity | provider、稳定 native device 标识、设备实例 epoch | 区分同名设备重建前后 |
| EidIdentity | DeviceIdentity、EID 值、EID 类型、当前 index、plane/port/IODie 信息 | index 仅用于当次打开，不能作为跨重启身份 |
| FailureDomain | device、port、IODie、共享链路或部署提供的 domain ID | 选择真正独立的备路 |
| PathKey | 双端 EidIdentity、peer incarnation、profile ID | 隔离不同远端与参数组合 |
| PathSnapshot | PathKey、connectivity version、health epoch、状态 | 提交前进行版本复核 |

设备名或 EID 的稳定性必须由实际 provider/部署保证；缺少唯一硬件 ID 时，使用部署声明标识加本进程实例 epoch，不能假装 native 数字序号跨重启稳定。

UMDK U8 中，port EID、primary EID、bonding EID 有不同覆盖范围；primary EID 也不意味着跨所有 plane 可达。未知 EID 类型不得按“全连通”处理。

### 6.2 可达性矩阵

ConnectivityTable 为每个 EID 对记录 `Unknown / Allowed / Denied`，附带证据来源、版本和有效期：

- 来源一：经过校验的部署拓扑清单。
- 来源二：provider 能力与拓扑查询的组合；查询接口和字段需以目标 UMDK 版本为准。
- 来源三：实际建链和数据探测结果，用于验证已允许路径的当前可用性。

拓扑允许与链路健康分开：`Allowed` 只是允许尝试，只有健康状态合格才接收业务。建链成功也不能永久覆盖后续物理故障。

提交候选条件为：

```text
可达性允许
AND profile 双端一致且本地支持
AND 双端注册描述匹配当前 generation
AND 健康状态允许业务
AND 故障域/设备未进入隔离
AND 能获取 Endpoint 租约与配额
```

严格物理模式初始化时拒绝 bonding 设备；无匹配设备、无合法 profile 或无法满足要求的独立备路数量时，返回具体原因。若允许降级为单路径，必须由配置明确声明，不能对外继续承诺同等高可用。

### 6.3 传输参数协商

新增 `PhysicalLinkProfile`，至少包含：

- 事务模式 `trans_mode` 和传输类型 `tp_type`。
- 各端适用的 priority 与其映射/能力依据。
- 所需 ordering 语义、completion 语义和可支持的 fence 契约版本。
- 可用消息大小、READ/WRITE 支持、相关资源限制。
- profile ID 和配置来源。

两端 priority 的数值不必强行相等，必须验证它们各自映射到兼容的传输能力与服务配置。

不能仅因为当前代码固定 RC/CTP，就认定该组合在所有物理设备都有效；也不能仅凭通用文档描述认定所有物理 RC/CTP 都无效。应查询目标设备能力，并对目标驱动、拓扑与版本中的选定组合做建链和读写验证。

建立 Endpoint 前先确定 profile，随后用它创建本地资源、导入远端描述和绑定；不能先用默认参数创建，再在握手后发现双方不一致。

当前 WR 没有显式设置 `place_order`。若业务要求明确顺序，应按 provider 能力设置，并在接口契约中说明作用域。单个队列上的顺序不能自动扩展为多 Jetty、多路径或多个写者的全局顺序。

### 6.4 Bootstrap 升级与幂等绑定

现有 v1 解析器拒绝其他协议版本，不能仅修改发送的版本号就认为兼容。

建议增加能力发现 RPC，升级端在明确对端支持后使用 v2 物理握手；旧端点保持原 v1 路径。若无法安全发现能力，使用显式版本配置并将不匹配作为失败。严格物理模式不得静默降回缺失必要字段的 v1。

v2 消息携带双方身份、profile、设备 epoch、Endpoint generation、注册 generation、探测区描述及请求 ID。绑定事务使用这些字段形成指纹：

1. 校验消息、范围、注册和目标身份。
2. 若相同请求 ID 与相同指纹已完成，返回原结果。
3. 若 Jetty 已绑定相同目标，按本端记录与 provider 语义确认可复用。
4. 若 EEXIST 不能证明同一目标，销毁或退休该实例，返回冲突；不得报告 Ready。
5. 双端确认后公布 Ready；握手中断进入有限期清理。

对端重启或 generation 变化时，失效对应 Ready Endpoint 和导入缓存。控制面通知是加速路径，重连握手仍须校验 generation，避免依赖通知必达。

## 7. 主动健康检查、事件与恢复设计

### 7.1 状态机与 half-open

建议每条物理路径维护：

```text
Unknown → Probing → Healthy
Healthy → Suspect → Draining → Down
Down --冷却结束--> HalfOpen --数据探测成功达到门槛--> Healthy
HalfOpen --探测失败--> Down
任意状态 --身份/profile/注册改变--> 旧实例 Retired，创建新实例
```

`HalfOpen` 仅允许有限探测，不立即恢复业务流量。冷却到期只允许开始验证，不计作“恢复成功”。

同一设备的 fatal event 将受影响的路径组整体移出候选；单条 peer 路径失败不能直接宣布所有远端都故障。故障范围由事件性质、设备归属和数据面证据决定。

### 7.2 异步事件处理

建议给 Adapter 增加事件读取、确认和设备健康查询接口，具体映射到当前 UMDK 的 get/ack async event 能力。当前 UB Adapter 只有相关 fd 入口，没有完整事件消费闭环。（T2、U1）

事件线程负责：

1. 等待事件并立即拷贝类型、对象标识、原始状态和设备 epoch。
2. 按 API 生命周期要求 ACK，确保每个事件恰好确认一次；未知事件也记录并按规定确认。
3. 将不可变事件记录发给 RecoveryCoordinator，避免在事件回调中直接退休整个资源图。
4. 区分 Jetty、JFC、端口和设备级错误；无法精确定位时保守扩大隔离范围，但不误用旧 epoch。
5. 事件丢失、fd 异常或不支持事件时，继续依靠主动探测和业务完成形成兜底。

事件处理不能与普通 poll 同时无序消费同一类完成；共享 JFC 的排空仍沿用 Adapter 中的同步规则。

### 7.3 数据面探测

每个需要验证的路径使用独立 Jetty、受控配额和专用注册探测区，不读写业务区。推荐使用 READ 已发布的只读小块，校验目标 incarnation 和 nonce/内容标识。

探测区应由控制面发布有效注册描述与生命周期；远端重启后旧描述失效，不能继续读旧地址。

设计要求：

- 每条路径限制并发探测数；使用 jitter，避免大量路径同步探测。
- 独立预留少量资源，业务队列满时仍能检测；探测自身也有资源上限。
- 区分“因本地资源争用未提交”和“已提交后超时”，避免把资源不足全记为断链。
- READ 证明指定方向的读数据面可用；需要双向服务时由双端分别探测。WRITE 的权限与语义仍通过建链校验和专项测试覆盖。
- 控制面 ping 只能证明控制面可达，不能替代 UB 数据探测。
- 探测失败也必须遵守排空和内存生命周期规则。

新参数如探测周期、超时、失败阈值、恢复连续成功次数、最大探测并发均需可配置，部署根据规模与恢复 SLO 标定，不在本设计中宣称未经验证的毫秒级恢复指标。

### 7.4 设备故障隔离与 Context 重建

对于需要重建的致命故障，RecoveryCoordinator 按设备故障域执行：

1. 原子切换设备 epoch 的状态为 Draining，冻结相关新提交。
2. 收集所有依附该设备的 EID Context 和 Endpoint，防止只隔离其中一个 EID。
3. 排空已接受 WR；得到 fence 或可靠完成后结算配额和逻辑尝试。
4. fence 失败的资源留在隔离区，不释放、不复用、不假装恢复成功。
5. 在满足 provider 销毁条件后，释放旧资源；重新枚举并验证 EID、端口与 profile。
6. 创建新 Context/JFC/探测资源，为现有本地注册内存建立新设备注册。
7. 生成新设备 epoch 和注册 generation；构造完整元数据后原子发布，旧对象由引用与安全屏障延迟回收。
8. 建立新 Endpoint，进入 HalfOpen，探测达标后逐步恢复业务候选。

永久损坏或 provider 无法证明排空时，保持该设备 Down，依靠其他独立设备继续服务。恢复可以停留在安全隔离并要求运维介入，不能以“重新 poll 成功”冒充完成致命 Context 重建。

### 7.5 并发与锁

建议保留 Adapter 当前对 Jetty 和 JFC 的锁顺序，并为新层建立约束：

- 路径表锁仅用于快照和状态更新，不持锁执行 RPC、原生 poll/fence 或资源创建销毁。
- EndpointStore 锁下只修改索引、租约与退休队列，慢清理移交 Coordinator。
- 先取得版本快照再执行慢操作，提交结果时复核 epoch；过期结果只清理自己的实例。
- 新注册资源未完整准备前不发布；失败回滚只处理本次生成的资源。
- 销毁与提交使用同一个提交门禁；无法仅靠检查一个健康布尔值防止 TOCTOU。
- quota 的物理资源账本覆盖旧实例排空期间的占用，新 epoch 不清零旧账。

## 8. 错误语义、重试与内存生命周期设计

### 8.1 统一 RetryDecision

新增结构表达重试依据，贯穿 Adapter、Workers、Slice 和 Runtime：

```cpp
// 设计示意；不是当前接口。
struct RetryDecision {
    ExecutionOutcome outcome;  // NotExecuted / Succeeded / Unknown
    DmaSafety dma;              // Fenced / Pending / Unproven
    ErrorScope scope;           // Request / Memory / Endpoint / Path / Device
    RetryAction action;         // Fail / Refresh / SamePath / AlternatePath
    int raw_status;
};
```

`Fenced` 的含义必须明确覆盖哪些资源与副作用，不能仅依据状态名推导。原始 CR 保留用于排障，但上层不能通过魔法数字决定语义。

| 情况 | 结果判断 | 建议处理 |
| --- | --- | --- |
| 未被接受的 WR | 确定未提交此 WR | 可在期限内退避或换路 |
| WR 成功完成 | 成功 | 正常终结，按完成契约释放 |
| 本地地址/权限/长度错误 | 请求或注册问题 | 失败；只有明确陈旧注册证据时刷新后有限重试 |
| 远端不支持请求、确定权限错误 | 通常不是路径瞬态故障 | 不盲目换路；返回具体错误 |
| ACK timeout、部分连接故障 | 远端结果可能未知 | 排空后结合操作契约决定是否重放 |
| RNR retry exhausted | 需结合模式与实际资源原因 | 有界退避或失败，不能无限重试 |
| flush/unhandled 标记 | 按 provider 对该 WR 的执行语义判断 | 不将所有 flush 一概当作未执行 |
| 致命设备错误 | 可能涉及整个设备 | 故障域隔离、排空或留存、另选独立路径 |

特别是 `URMA_CR_ACK_TIMEOUT_ERR`，远端 WRITE 可能已生效但确认未返回。把它归类为 TIMEOUT 并换设备，不会消除这种不确定性。（T2、U7）

### 8.2 READ 与 WRITE 分开制定重放契约

READ：只有旧尝试不再写本地目标缓冲区后才允许重试。若要求读取一致快照，远端源必须固定版本或由上层 lease/快照保护；重试读到新内容不是 Transport 能自动解决的一致性问题。

WRITE 的最小可重放契约：

1. 源缓冲区在所有相关尝试安全结束前内容稳定。
2. 目的区在操作结束前由本次传输独占，不与新版本或其他写者竞争。
3. 只有所有必要分片成功后才向读者发布。
4. 在允许上层复用目的地址前，旧尝试已排空，或存在等价且经验证的隔离机制。

在这个契约下，向同一区域重写相同字节可以是安全选择；但仅“值相同”不足以证明 DMA 生命周期安全。

对于普通共享原地 WRITE，如果旧数据 A 超时后新写 B 已发布，再重试 A，就可能覆盖 B。必须拒绝透明重放，或者由上层采用下一节的版本化发布。

### 8.3 可选的版本化发布协议

对并发更新或强语义业务，建议把传输和发布分开：

1. 上层分配 operation ID、目标对象 version 和不会与旧操作复用的暂存区。
2. TENT 将数据写入该版本的未发布区域；重试仅落在相同版本区域。
3. 完成并验证后，通过控制面提交 `commit(operation_id, version, checksum)`。
4. 远端对 operation ID 幂等处理，并检查当前允许发布的 version。
5. 旧 version 的迟到提交被拒绝；旧暂存区满足 DMA 安全条件后才回收。

该协议需要远端参与，不是当前原始单边 WRITE 已有能力。控制面幂等提交也不自动证明底层原地内存安全；lease 超时本身不是 DMA fence。

UMDK 的 MSN 去重针对相应消息编码，不应被当作原始 WRITE 的通用防重放协议。因此，不能通过简单移植 MSN 来解决上述问题。

### 8.4 fence 与对外完成契约

定义两个不同状态：

- `LogicalDeadlineExceeded`：业务期限已超，停止新尝试。
- `ResourcesSafe`：相关 DMA 与资源使用已达到可释放条件。

如果现有公开完成状态意味着调用者可以立即复用缓冲区，就只能在 ResourcesSafe 后返回该最终状态。不能为了“有界响应”提前返回普通 TIMEOUT，之后仍让硬件访问调用者会释放的地址。

若希望提前报告，需要新增显式的 pending-drain 结果和等待安全释放的句柄，或改为 Transport 拥有的 staging 内存；这是 API/资源成本变更，必须单独评审。当前接口契约未扩展前，优先保留现有安全等待与隔离行为，并暴露排空阻塞告警。

对物理 UDMA，验收需分别证明：

1. 本地源/目标与注册内存在完成或排空后可安全复用。
2. 不会发生旧远端写副作用在新尝试或新版本之后继续到达。
3. 错误/flush/重复 CR 不会错误释放另一个 WR 的资源。
4. 如果 provider 只能证明本地安全，原地 WRITE 的自动恢复就不能声称已获得完整隔离；需要更强 provider 保证或版本化暂存协议。

### 8.5 统一跨 Transport failover

当前 Runtime 的 `updateTaskStatusAfterPoll` 以 `FAILED` 为既有 failover 入口；UB 任务最终可能保留 `TIMEOUT`。不能假定启用通用 failover 就覆盖所有 UB 超时。（T7、T9）

设计上将 RetryDecision 提升到运行时决策：

- 先确认前一 Transport 的 DMA 安全和重放契约。
- 使用同一个 request deadline 和总尝试预算。
- 保留 Unknown 等结果语义，不把 TIMEOUT 无条件改成 FAILED。
- 对需要操作隔离的任务，必须在旧任务安全退出后提交新的 Transport 尝试。
- 保持 alias/task 终结与统计一次性，不能并行存在两个仍可触碰同一区域的 owner。

TCP 或 RDMA fallback 只在部署实际具备、且策略明确允许时生效，不可作为物理 UB 备用路径缺失的默认补偿。

## 9. 有界进展、Endpoint 容量与准入设计

### 9.1 请求级期限

保留当前已提交 WR 的 `slice_timeout_ms`，另引入绝对 request deadline：

```text
总预算覆盖：
admission → 排队 → 可达性/握手 → 导入 → 配额等待
→ 下发 → 完成等待 → 排空 → 允许的重试/Transport failover
```

所有 defer 与重新入队检查 deadline、取消状态和剩余预算。错误尝试次数、未提交的资源等待次数分别统计，但不能靠“不消耗 max_retries”获得无限生命周期。

截止后移出待提交队列，不再创建新 Endpoint。对已提交 WR 进入安全排空，按 8.4 节区分逻辑期限与缓冲区安全释放。无法排空时必须有明确隔离资源上限和告警，不能通过不安全释放来强行满足期限。

### 9.2 Endpoint 候选租约

新增 `CandidateLease`：

1. EndpointStore 返回候选时增加调度 pin，禁止普通容量淘汰。
2. 选路只在可承担的候选数内解析 Endpoint；根据空闲槽和已有缓存对象调整数量。
3. 候选状态或 generation 改变时丢弃该候选并尝试其他候选，不持续重选同一无效引用。
4. 在提交门禁中复核健康、generation、Ready 和租约，再获取/确认 quota 与 outstanding；任一步失败回滚本次预留。
5. 下发被接受后将租约转交 Inflight 所持引用/计数；未选中的候选立即释放 pin。
6. 淘汰条件变成 `outstanding == 0 && scheduling_pins == 0`，隔离实例仍计入容量。

为避免两个调度器互相 pin 满缓存，候选扩展不能阻塞等待更多容量：拿不到第二个槽时立即使用已有合格候选或释放租约后退避。单 Endpoint 容量配置应能正确工作，或在启用特定并行策略时明确拒绝，而不是隐式无限 defer。

初步修复可以先采取容量感知解析和逐个尝试；租约仍是完整并发方案，不能只把 `resolve_limit` 改成 1 就宣称全部竞争问题解决。

### 9.3 混合 Transport 准入

当前 `AdmissionQueue` 给不同 owner 预测时延时，会使用全局 `dispatching_bytes_`。RDMA 与 UB 的带宽模型不同，全局队列字节不一定是某个 Transport 的有效排队量。（T9）

建议：

- 保留全局内存/并发上限。
- 新增按 Transport 统计的在途与队列字节，用于该 Transport 的期限预测。
- 如果多个 Transport 共享同一瓶颈，明确引入共享资源组，不能简单认为它们完全独立。
- owner 在 failover 时从旧 Transport 账本扣除，再计入新 Transport，保证取消和重复完成幂等。
- 对纯 UB 且未启用 deadline 降级的部署，不把此项算作去 bonding 的先决条件。

## 10. 配置与兼容性

### 10.1 当前可以使用的配置

T10 已有 `transports/ub/enable`、`device_filter`、worker/poller 数、JFC/Jetty 数、`max_endpoints`、`slice_size`、`max_retries`、`slice_timeout_ms` 和 `endpoint_cooldown_ms` 等。

当前显式选择真实物理设备/EID 的过滤器可以避开自动 bonding 优先。过滤器中的拓扑名必须来自实际发现结果；不能把本文示意名称复制为现场设备名。

但显式过滤只解决设备选择，不代表 G1—G10 已解决。

### 10.2 建议新增的配置结构

以下 JSON 是配置设计示意，不是当前分支可运行配置。所有 `physical` 子字段尚待实现；数值仅用于表达结构，生产取值需测试标定。

```json
{
  "transports": {
    "ub": {
      "enable": true,
      "device_filter": [
        "ub:example_device_a:eid0",
        "ub:example_device_b:eid0"
      ],
      "physical": {
        "strict": true,
        "topology_file": "physical-ub-topology.json",
        "profile_id": "deployment-validated-profile",
        "required_independent_paths": 2,
        "probe": {
          "interval_ms": 1000,
          "timeout_ms": 2000,
          "max_inflight_per_path": 1,
          "recover_success_threshold": 3
        },
        "write_replay_policy": "exclusive_immutable",
        "fatal_recovery": "recreate_or_keep_down"
      }
    }
  }
}
```

两个过滤项只是结构示意，部署仍需验证它们属于独立故障域并且都能到达目标。若实际只得到一条独立路径，严格校验应报错，不能静默降低要求。

拓扑文件至少声明稳定设备身份、EID、故障域和允许的 peer 配对。敏感授权信息继续由原注册与安全机制处理，不把 topology 文件当作远端访问授权。

兼容原则：

- 默认保持现有行为；严格物理模式显式启用。
- 新 profile/协议需双端支持，混部时明确拒绝不满足契约的路径。
- 设备、profile、注册 generation 改变均触发相关缓存失效。
- 保留 bonding 路径用于 A/B 与回滚，但回滚配置也不能跳过旧 WR 排空。
- 编译时移除 `urma_ubagg.h` 等依赖是可选构建整理，不等同于运行时不再选择 bonding，也不是本方案可靠性门槛。

## 11. 典型 WRITE 故障恢复时序

以独占、不可变源 WRITE 为例，假设路径 A、B 属于独立故障域：

1. 双端发现并验证 A/B profile，发布每个 Context 的注册描述。
2. 请求绑定绝对 deadline；Workers 选择 A，获得 Endpoint 租约和物理配额。
3. 建立 Inflight token，持有 Segment 与缓冲区约束，post 一个 WRITE。
4. ACK 超时。标记此次尝试 Unknown，不立即向 B 重投。
5. 将 A 所属故障范围移出候选，对旧 Endpoint quiesce；继续处理真实 CR。
6. 若 fence 不能建立，保留资源并告警；不得把 B 的重试成功作为旧 A 已安全的证明。
7. fence 合格后，按独占、不可变源契约允许重放；释放旧尝试资源计数，生成新 attempt。
8. 重新取元数据与路径快照，在 B 建链/导入，预留配额并提交。
9. B 成功后以 attempt/token 核对终结；迟到的 A 完成只影响其旧实例，不重复完成业务。
10. Task 所有分片成功且安全条件满足后，应用才发布目的数据。
11. A 经设备恢复和 HalfOpen 探测后逐步回到候选，不因为冷却时间到期直接恢复业务。

若目的区已有另一个写者发布新版本，则不能套用上述透明重放，应采用版本化暂存/发布或返回结果未知。READ 时序类似，但需要保护本地目标与远端源的快照要求。

## 12. 无需为本范围补齐的 bonding 能力

| bonding 能力 | 本范围为什么不用照搬 | 需要保留的实际责任 |
| --- | --- | --- |
| 统一 bonding EID 和虚拟设备 | 应用已使用 TENT Segment，底层设备可被 Transport 隐藏 | 稳定身份、可达性与故障域 |
| 虚拟 Context/Jetty/JFC 资源展开 | TENT 已管理多套原生资源 | 重建、注册与安全回收闭环 |
| 聚合完成 ID 和用户上下文恢复 | TENT token/attempt 已完成映射和一次性终结 | 迟到完成、代际和配额正确性 |
| 聚合 Segment 展开缓存 | TENT 有按 Context 与 generation 区分的导入缓存 | 新设备 epoch 失效与元数据刷新 |
| Balance 原样调度算法 | TENT 已有 NUMA、quota、负载与带宽评分 | 先过滤不可达/不健康路径，再调度 |
| MSN 接收消息编码去重 | 当前是单边 READ/WRITE；消息去重不解决原始 WRITE 重放 | 明确 WRITE 契约或版本化发布 |
| 聚合接收队列的倍数预投递 | 单边 READ/WRITE 不需要为每个请求投递 recv | 原生 JFR 等资源按实际 API 创建维护 |
| 完整软件 RNR 重试模块 | bonding 默认未开启软件 RNR；范围也不同于 SEND/RECV | 仍需分类 RNR 错误并有界处理 |
| 全部 SEND/RECV、atomic 能力 | 不在当前 TENT 原生 UB 数据接口范围 | 将来扩展时单独设计接收与不可重放语义 |
| 用户态 WQE/doorbell 实现 | 直接物理 UDMA 已提供普通 URMA 快路径 | 使用 post/poll 并遵守 provider 契约 |
| 虚拟 JFCE 事件合并 | 当前可靠性不要求从 polling 改成中断 | async 错误事件处理仍需补齐 |

UMDK 当前默认启用 failover、failback 和健康检查；软件 RNR 默认关闭。默认值不代表所有模式、所有资源类型都具备相同能力。尤其不能把 Jetty 主路径的健康恢复直接推及所有独立 JFS/JFR 场景。（U5）

## 13. 观测与验收设计

### 13.1 必需观测

建议新增指标并限定标签基数：

- 按故障域和有限错误类别：事件数、路径暂停、HalfOpen 成败、真实恢复次数。
- 按阶段：检测耗时、排空耗时、重建耗时、重新注册耗时、恢复业务耗时。
- 请求：排队/握手/配额等待、重试次数、结果未知数、因契约禁止重放数。
- 资源：在途 WR/字节、candidate pins、隔离 Endpoint/Context/Segment 数量及保留字节。
- 配额：申请失败、重复释放被抑制、失败恢复后是否回归基线。
- 元数据：generation 冲突、旧事件/旧探测被拒绝、profile 不匹配。
- 错误日志保留 raw CR、opcode、attempt、epoch、路径与操作关联 ID；不把高基数字段全部做成指标标签。

“冷却到期”和“真实探测恢复”应使用不同计数器。不能沿用现有 recoveries 名称却把它直接解释为硬件恢复次数。

### 13.2 无硬件的确定性测试

优先扩展现有
[ub_core_test.cpp](../../mooncake-transfer-engine/tent/tests/ub_core_test.cpp)、
[ub_native_data_path_test.cpp](../../mooncake-transfer-engine/tent/tests/ub_native_data_path_test.cpp) 和
[ub_teardown_test.cpp](../../mooncake-transfer-engine/tent/tests/ub_teardown_test.cpp)，新增设计对应测试。这里是未来实现的验收计划，本次文档提交不代表已运行或新增这些测试。

| 测试场景 | 必须验证 |
| --- | --- |
| 同设备多 EID、跨 plane、未知 EID | 不把不可达组合选为业务路径，不把共享故障域计为独立备路 |
| profile 不匹配、旧 peer、EEXIST 不同目标 | 不进入 Ready，错误原因明确，无资源泄漏 |
| posted prefix 与 bad_wr | 仅未接受项可重投，已接受项持有资源 |
| 重复/迟到完成与 attempt 竞争 | 只终结一次、只释放一次、旧 token 不影响新实例 |
| fence 成功、超时、部分成功 | 部分完成不丢；没有安全依据时既不释放，也不并行重试 |
| 远端权限错误与 ACK timeout | 永久错误有限失败；Unknown 不伪装成 NotExecuted |
| JFC 空 poll 和冷却结束 | 不直接恢复业务健康状态 |
| HalfOpen 探测失败与旧 epoch 探测成功 | 前者仍 Down，后者不能恢复新实例 |
| 设备重建中断、部分注册失败 | 旧元数据不误用，新元数据不部分发布 |
| max_endpoints=1、多候选、并发淘汰 | 有界成功或失败，不反复命中退休候选 |
| quota 满、导入持续忙、握手重试 | deadline 覆盖所有等待路径 |
| 混合 RDMA/UB 期限准入、Transport failover | 账本不串用；DMA 安全和总预算不丢失 |
| 取消与 shutdown 同时发生 | 不丢失已接受 WR，不提前释放缓冲区 |

### 13.3 真实 UDMA 的必测项

至少覆盖目标 provider/驱动/固件版本与合法 profile，并记录设备、EID、拓扑和所有开关：

1. 正常双向 READ/WRITE、多设备并发、分片边界和最大资源限制。
2. 拔链/端口 down、单向黑洞、peer 退出与重启、设备级故障。
3. 在远端可能已写入后丢 ACK，验证 Unknown 和重放契约。
4. 在 fence 前后用隔离测试内存检测迟到访问；分别观测本地 DMA 与远端副作用。
5. poll、事件、timeout、quiesce 并发，重复完成和不同 Jetty 的完成交织。
6. 故障后所有相关 EID 的隔离与新 Context 注册/元数据生效。
7. 长时间故障、隔离资源满、恢复抖动和反复回切。
8. 在持续负载与资源紧张下验证探测仍有进展，但不会无界挤占业务。
9. 再次注册相同地址与设备重建后的旧 generation 拒绝。
10. 同一应用区的新版本发布测试，证明旧尝试不能覆盖已发布新数据。

### 13.4 与 bonding 的 A/B 判定

A 组为当前 TENT 实际 bonding 模式和配置，B 组为物理设备增强方案。保持物理端口、故障域、内存、请求分布、上层契约和允许的 fallback 相同。

验收前由部署方确定恢复时延、成功率、尾延迟和隔离资源上限。比较至少包括：

| 项目 | 判定 |
| --- | --- |
| 数据正确性 | 无静默损坏、旧写覆盖、重复发布或错误成功 |
| 内存安全 | 无释放后访问；不能证明时安全隔离而非强制释放 |
| 单故障覆盖 | A 能覆盖的约定故障，B 有相同独立备用资源并能完成恢复 |
| 故障检测与恢复 | 检测、排空、切换、回切分别满足约定 SLO |
| 长故障资源 | 隔离对象有上限、告警明确，不拖垮无关健康路径 |
| 终结与语义 | 成功、失败、超时和结果未知均符合相同业务契约 |
| 健康与故障性能 | 吞吐和尾延迟分别比较；不能只测无故障带宽 |

只有完成上述测试，才能对已测试配置给出“不劣化”的工程结论。单元测试或源码阅读无法替代这一结论。

## 14. 实施拆分与交付门槛

| 阶段 | 主要修改位置 | 交付内容 | 门槛 |
| --- | --- | --- | --- |
| A：安全与路径契约 | params、Adapter、Bootstrap、PathManager | 严格选择、身份/profile、绑定校验、RetryDecision、provider fence 契约 | G1—G5；不允许未经定义的原地 WRITE 自动重放 |
| B：调度进展 | EndpointStore、Workers、Slice | 候选租约、容量感知、统一期限、安全取消/终结 | G6、G9；小缓存与争用回归通过 |
| C：高可用闭环 | Adapter、Context、PathManager、BufferRegistry | async event、主动探测、故障域隔离、设备重建、代际发布 | G7、G8；独立备路和 HalfOpen 可证明 |
| D：运行时联动 | TransferEngineImpl、AdmissionQueue | 安全跨 Transport 决策、对应部署的准入账本修正 | 启用相关策略前解决 G10/G11 |
| E：硬件验收与灰度 | 部署配置、故障测试、指标 | bonding 对照、故障注入、回滚与运维手册 | 满足第 13 节后扩大流量 |

阶段拆分是实现依赖安排，不表示仅完成 A 就可宣称高可用等价。对需要去 bonding 且维持跨设备恢复能力的生产部署，A、B、C 与硬件验收均是必要交付。

首轮应优先澄清 fence 和 WRITE 契约，因为它们决定哪些重试能开放；随后在现有资源管理与调度上增加物理路径、健康与重建机制。完成这些闭环后，TENT 可以直接依赖 UDMA 的基础通信能力，而无需复刻 bonding 的全部资源虚拟化与消息协议。

## 15. 本次文档交付的验证范围

本次仅新增分析文档，没有实现上述增强，也没有改变运行时配置。已核对核心源码入口、本文的本仓库相对链接、UMDK 引用路径、JSON 示例语法，以及 UTF-8、围栏代码块、尾随空白和冲突标记。

执行 `docs/make.bat html` 时，本机缺少 `sphinx-build`，未能构建 HTML；`pre-commit` 及其 Python 模块同样未安装。本文件位于 `docs/analyze/`，没有加入 `docs/source/` 的 Sphinx 导航。

本次没有运行 C++ 测试、UB 硬件测试或 bonding A/B 测试。第 13 节描述的是未来实现的验收要求，可靠性等价结论仍以目标部署完成该验收为前提。
