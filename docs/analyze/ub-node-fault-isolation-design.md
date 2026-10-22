# TENT 与 Mooncake Store 的 UB 节点级故障隔离方案

> 日期：2026-10-22。
>
> Supercache 分析基线：`main_learn`，`c03a339f`。UMDK 基线：`09086da5d86c53f83eb395109d636957d3efcc79`。
>
> 本文是待实现方案，不表示相关接口或行为已合入。当前范围为 CPU DRAM 上的普通 UB READ/WRITE。
>
> 前置约定：能够准确归属端口、属于当前实例和资源代次的 status 4，可以直接构成该端口的有效故障证据。节点隔离不再要求全部证据必须来自 provider 状态查询。
>
> 基础资源、分片、重试与 DMA 安全设计见 [物理 UB 可靠性设计](tent-physical-ub-reliability-design.md)。本文负责节点级故障证据、准入、传播、恢复与 Store 接入；路径排空和设备重建继续沿用该设计。

## 1. 目标与核心决策

节点级 UB 阻断条件为：

```text
相关端口清单经过配置与发现结果核验，完整且非空
AND 每个相关端口都具有当前实例、当前设备集合下的有效 BAD 证据
    → 阻断该服务实例的 UB 节点准入
```

BAD 证据可以来自 status 4、合格的 provider 查询、具有明确端口故障语义的设备事件；允许混合使用。只要满足有效性条件，status 4 不需要再经 provider 查询确认才能计入。

例如实例只使用 P0、P1：

| 当前事实 | 节点级处理 | 路径级处理 |
| --- | --- | --- |
| P0=status 4，P1=UNKNOWN | 不新增节点阻断，标记待核验 | 隔离 P0；P1 只能按请求级规则尝试或探测 |
| P0=status 4，P1=有效 GOOD | 节点不阻断，处于退化状态 | 禁用 P0，继续使用满足请求条件的 P1 |
| P0、P1=当前有效 status 4 | 立即阻断本地节点 UB，无需等待查询 | 两个端口均不可承接新业务 |
| P0=status 4，P1=有效查询 BAD | 可以阻断，证据来源可以混合 | 同上 |
| 已阻断后 BAD 证据过期或查询失败 | 保持阻断，标记证据待刷新 | 不因时间流逝重新启用端口 |
| 已阻断后 P0 获得有效恢复证据 | 解除“全部端口故障”阻断 | 仍检查 P0 的 peer 可达性、资源与排空条件 |

隔离只改变 UB 数据面 eligibility，不直接改变 membership、对象所有权、hash ring、Segment 挂载、ReplicaStatus 或节点进程存活状态。RPC/TCP 等其他服务按各自健康和策略继续工作。

本方案不负责负载权重算法。它提供端口与节点健康视图，供后续读副本、新副本放置和迁移调度使用。

## 2. 分层模型与状态所有者

### 2.1 “节点”的定义

本文节点指一个 UB 服务实例及其配置使用的设备/端口集合，而不是整个 IP 地址或物理主机：

```text
NodeIdentity = service_instance_id + incarnation
PortInventory = NodeIdentity + inventory_generation + physical_ports[]
PortIdentity = stable_device_identity + device_epoch + chip/die/physical_port
```

`service_instance_id` 对应当前服务；`incarnation` 在进程重启后变化。`inventory_generation` 在使用的端口集合发生受控变化时变化。

同一物理端口的多个 EID 只计一个端口。同一机器多个服务实例可以有不同端口集合；一个实例全故障不能直接隔离同机器的所有实例。

若多个 TENT engine 共用该实例的端口集合，应共享一个进程级证据管理器，或由一个明确的聚合所有者合并。不得让各 engine 把自己的局部集合都声明为完整节点清单。跨进程的整机隔离需要额外节点代理，不在本次默认范围。

### 2.2 三种状态分开维护

| 层次 | 回答的问题 | 状态所有者 |
| --- | --- | --- |
| 本地端口事实 | P0、P1 分别有哪些有效健康/故障证据？ | TENT `UbPortEvidenceManager` |
| 节点级准入 | 是否已经确认该实例全部相关端口故障？ | TENT `UbNodeAdmission`；远端维护只读核验视图 |
| 请求级可用性 | 对当前 peer、profile、buffer 是否有可安全提交的路径？ | TENT PathManager、Workers、EndpointStore |

节点解除阻断只是允许进入请求级检查。不能因此跳过建链、注册、配额、旧 DMA 排空或 peer 可达性检查。

节点仍未阻断，也不保证一定有可用业务路径。例如 P0 已 BAD、P1 UNKNOWN 时，P1 可以进入受控验证；若没有合格路径，请求有界失败或选择允许的其他 Transport，不能无限等待。

### 2.3 组件关系

```text
Completion / async event / provider query / recovery probe
                  |
                  v
       UbPortEvidenceManager
       - 清单校验、端口归属、证据归并、代次与新鲜度
                  |
                  v
           UbNodeAdmission
           - 本地全部故障阻断与恢复
                  |
        +---------+----------+
        |                    |
        v                    v
TENT 提交门禁         QueryUbNodeHealth / 健康通知
        |                    |
        v                    v
请求级选路          Store HealthRegistry / Master / Client
```

Manager 是端口事实的唯一发布者，Admission 是节点阻断状态的唯一发布者。Store 不再根据错误码独立维护第二套端口判断。

## 3. 相关端口清单必须完整且可核验

### 3.1 清单生成

1. 读取配置允许的物理设备/EID 和服务用途。
2. 用设备发现结果核验设备身份、EID 和启用资源。
3. 将 EID 映射到物理端口，处理多个 EID 共享一个端口、primary EID 覆盖多个端口的情况。
4. 去重形成服务实例真正使用的端口集合。
5. 记录发现版本、映射能力和 `inventory_generation`，然后原子发布清单。

清单只能覆盖部署声明且被发现结果核验的服务资源。不能在某个端口失败后通过重新枚举“当前活跃设备”把它删除，从而错误缩小分母。

禁用端口或变更服务设备范围必须走受控重配置，不能把暂时发现不到设备等同于配置移除。启用设备发生变化但清单尚未完整核验时，进入 Reconfiguring，不宣称新的完整端口集合已就绪。

### 3.2 不能由 EID 列表直接推断物理端口数量

当前 `DeviceInfo` 表示设备/EID 对，没有完整的物理端口身份和映射信息。一个 EID 可能覆盖多个物理端口。（T1）

因此需要新增 inventory 和映射能力。若当前 primary EID 上的 WR 无法确定实际经过哪个物理端口：

- 可以隔离该次路径或 Endpoint。
- 可以触发查询或受控探测。
- 不能把一个 status 4 复制成该 EID 覆盖的所有端口 BAD。
- 不能因为这是唯一发现的 EID，就认定整个节点全部端口故障。

严格模式应在启动时验证可归属能力；缺少能力时明确显示降级原因，不能对外宣称已实现 status 4 的完整端口隔离。

## 4. 证据模型与有效性

### 4.1 建议数据结构

以下是设计示意，并非当前代码：

```cpp
enum class PortFact { Unknown, Good, Bad };

enum class EvidenceSource {
    Status4,
    ProviderQuery,
    DeviceEvent,
    RecoveryProbe,
    DataCompletion
};

struct PortEvidence {
    NodeIdentity node;
    uint64_t inventory_generation;
    PortIdentity port;
    uint64_t port_recovery_epoch;
    uint64_t operation_id;
    EvidenceSource source;
    PortFact fact;
    uint64_t owner_sequence;
    uint64_t observed_steady_ns;
    uint64_t valid_until_steady_ns;
    bool exact_attribution;
    int raw_status;
};
```

端口状态由 Manager 归并，业务线程不能直接改 `bad_count`。每次有意义的端口事实变化递增 `health_epoch`；观测刷新序号与事实 epoch 分开，避免“事实没变”被误读成“查询没刷新”。

### 4.2 status 4 成为有效 BAD 的条件

必须同时满足：

1. provider/profile 按本文约定将本次 status 4 解释为对应端口故障。
2. 能从该次 WR/Jetty 的绑定与实际路径信息准确归属单个物理端口。
3. 对应当前 Node incarnation、inventory generation 和 device epoch。
4. 对应有效的 attempt/token，并且不是 entity marker 或重复完成。
5. 没有被该端口较新的恢复屏障淘汰。
6. 在配置的证据有效窗口内，且序列、时间与映射校验通过。

这些条件是确保“这是 P0 当前的 status 4”所需的归属和生命周期检查，不要求另做 provider 查询。

当前代码中 status 4 对应的符号被归类为 MEMORY_ERROR；需按已选 provider 的明确契约增加端口故障分类，并保留原始状态。不应简单把所有平台的数值 4 无条件重解释。（T2）

端口故障观测与请求结果是两条链路：记录 BAD 不代表该请求确定未执行，也不自动授权 WRITE 重放。

### 4.3 其他证据的作用

| 来源 | 能否标记端口 BAD | 能否作为恢复 GOOD |
| --- | --- | --- |
| 合格且精确归属的 status 4 | 可以，直接计入 | 不适用 |
| provider 完整状态查询 | 可以，对返回覆盖范围内的端口更新 | 可以，但需确认其 GOOD 语义与新鲜度 |
| 具有明确端口范围的故障事件 | 可以 | 恢复事件只有在保证可服务语义时可直接作为 GOOD |
| status 9 / ACK timeout | 仅作为路径异常与核验触发器 | 不可以 |
| 普通 RPC/TCP 成功 | 不可以 | 不可以证明 UB 端口恢复 |
| 当前代次、准确归属端口的数据探测成功 | 不适用 | 可以，证明所验证范围可用 |
| 当前恢复屏障后的合格业务完成 | 不适用 | 可以在语义允许时使用 |
| 旧 CQE、旧探测、身份不匹配的查询 | 不计入 | 不计入 |

设备 Link Up 不一定证明数据面可服务。若 provider 只能提供 Link Up，则标记待验证，再通过受控数据探测形成合格 GOOD。

数据探测应使用专用注册区域，不修改业务内存。成功证据有方向与可达范围；节点解除阻断后仍通过请求级路径规则限制未验证方向。

### 4.4 时效与冲突

BAD 与 GOOD 都有有效期和观测范围，具体窗口由目标 provider 与恢复 SLO 配置，并通过故障实验标定。生产启用不能只定义永久 BAD 而没有主动恢复通道。

归并规则：

- 不同 incarnation、inventory 或 device epoch 的证据不能混合组成“全故障”。
- 在同一清单快照下，只有所有端口都持有尚未失效的有效 BAD，才允许首次进入阻断。
- 先后很久发生、没有重叠有效期的两个 status 4，不能简单累加成当前全故障。
- 证据过期进入待核验，不自动产生 GOOD。
- 端口自身的失败隔离也保留到合格恢复，不能让 BAD 过期变 UNKNOWN 后直接重回业务候选；过期只影响能否用该证据新增节点全故障结论。
- 已确认全故障后的证据过期只改变核验状态，不解除阻断。
- 无法确定因果顺序的 GOOD/BAD 冲突，保留安全隔离并启动新的定向核验。

使用所有者本地单调时钟判断有效期。跨节点传递采样年龄、剩余有效预算和请求关联信息，接收端用自己的单调时钟计时；不能直接比较两台机器的 steady_clock 值。

### 4.5 迟到 CQE 与恢复屏障

每次开始有效恢复验证时，冻结本端口的旧提交代次，排空或隔离旧资源。成功验证后推进 `port_recovery_epoch`；提交时把该代次写入 Inflight。

恢复后的迟到 status 4 如果属于旧提交代次：

- 继续完成旧请求的清理和必要记账。
- 不覆盖新代次的 GOOD，也不参与新一轮节点阻断。
- 如仍有疑点，触发新代次核验。

旧 WR 的成功完成同样不能清除新代次故障。真正新的 status 4 会产生新的 BAD 并重新隔离。

恢复验证预留候选恢复代次和 probe ID，普通业务仍受门禁约束；只有验证通过并复核期间没有更晚的有效故障，才发布新代次。不能一开始探测就先清除 BAD 或解除节点阻断。

provider 查询启动时记录目标端口的当前 revision；查询完成时若期间已收到较新 status 4，不能仅因“查询返回得晚”就覆盖它。只有具备可验证采样因果关系的结果才提交，否则重新核验。

## 5. 节点状态机与准入

### 5.1 状态定义

| 状态 | 含义 | 节点层是否阻断 UB |
| --- | --- | --- |
| Unknown | 清单或端口事实尚不足以判断 | 不因“全端口故障”新增阻断；严格初始化/重配置门禁另行控制 |
| Available | 当前有充分健康事实，无已知有效 BAD | 否 |
| Degraded | 有 BAD、UNKNOWN 或待核验，但未满足首次全故障条件 | 否 |
| Unavailable | 已由完整有效事实确认全部 BAD，尚无合格恢复证据 | 是 |
| Reconfiguring | 正在更换设备集合或重建服务资源 | 由独立配置/资源门禁阻断，不伪称全部端口 BAD |

清单完整性与可服务初始化是不同门禁。首次启动不能因为 Unknown“不新增故障阻断”而跳过配置校验直接下发。

### 5.2 进入阻断与解除阻断分开

```text
CanEnterAllPortsDown(snapshot):
    inventory_complete
    AND inventory_nonempty
    AND for every relevant port:
        current-incarnation/current-inventory valid BAD

CanClearAllPortsDown(latched_block, update):
    same authorized service scope
    AND at least one relevant port has qualified current GOOD
    AND evidence is newer than the block/recovery barrier
    AND update is not stale, pending-only or identity-mismatched
```

Admission 保存 `latched_block`、`block_id`、阻断时的 inventory/health epoch 和原因端口集合。

状态迁移：

```text
Available / Unknown / Degraded
    -- 当前完整非空集合全部有效 BAD --> Unavailable

Unavailable
    -- 查询失败、证据过期、status 9、旧结果 --> 保持 Unavailable

Unavailable
    -- 任一相关端口获得合格恢复 GOOD --> Degraded 或 Available

任意状态
    -- 受控设备集合变化 --> Reconfiguring
```

第一次阻断使用“当前全部有效 BAD”的严格条件；阻断后的保持使用已确认事实的锁存。对外显示应区分“当前仍全部 BAD”和“曾确认全部 BAD、当前核验待完成”，不能把历史锁存冒充新查询结果。

### 5.3 端口与请求门禁

每个 BAD 端口从新业务路径候选中排除。节点层尚未 Unavailable 时，只要仍有合格路径，就继续服务。

提交必须至少复核：

```text
本地节点 UB admission
AND 已核验的 peer UB admission
AND 当前 inventory / device / endpoint / registration generations
AND 选定物理端口未被隔离
AND peer-specific 路径和 profile 合格
AND 资源可用、提交期限尚未超出
```

状态检查与实际 post 之间需要提交门禁同步，避免检查后端口进入 BAD，但新 WR 仍越过原生提交边界。

节点阻断停止新的 UB 尝试，不直接丢弃已接受 WR；在途 WR 继续完成或进入排空。当前请求仍使用原始执行结果，不因健康状态变化伪造成功或未执行。

## 6. TENT 的具体改造点

### 6.1 Adapter：增加语义与物理归属

当前 Completion 有 `native_status`、`token`、`local_jetty_id`，但没有完整物理端口归属；DeviceInfo 主要为设备/EID。（T1、T2）

建议：

1. 增加 provider fault profile，声明 status 4 的端口故障语义和归属能力。
2. 增加 PhysicalPortIdentity、EID-to-port 映射，以及必要的原生完成扩展。
3. 如果一个 Jetty/WR 严格固定物理端口，可在创建/提交时保存映射；否则必须由 provider 输出实际路径，不能从设备名猜测。
4. 保留原错误语义供业务结果处理；新增独立 PortObservation。
5. 抽象查询能力，bonding 和物理 UDMA 各自适配，不把 BONDP 私有 ABI 暴露到 Store。

### 6.2 Workers：在完成分支汇合处生成观测

利用 Inflight 的 token、attempt、path 和 Endpoint 引用，补充提交时捕获的 node/inventory/device/port recovery epochs。（T3）

处理顺序建议为：

```text
核对 token 和 attempt
→ 构造带代次的健康观测
→ 幂等投递到证据管理器
→ 按原规则结算 completion、quota 与任务结果
```

超时后收到真实错误完成、quiesce 返回的真实完成、正常 poll 完成，都经过相同入口。当前 `timed_out` 分支可能提前返回，因此观测入口应位于该分支之前，但仍执行代次过滤。

重复或未知 token 不能反复推进健康 epoch。entity marker 的 token 无效，不可当作普通 status 4 端口证据。独立设备事件走自己的事件 ID 和归属规则。

### 6.3 健康管理与恢复

建议新增以下逻辑组件，可按规模合并实现：

| 组件 | 主要职责 |
| --- | --- |
| PhysicalPortInventory | 完整清单、映射、版本与受控重配置 |
| UbPortEvidenceManager | 接收、校验、归并和发布端口事实 |
| UbNodeAdmission | 全故障锁存、恢复与 O(1) 快照读取 |
| UbRecoveryCoordinator | 探测、排空、资源重建和恢复屏障 |
| PeerUbHealthVerifier | peer 查询去重、摘要校验和远端 admission |

本地首次满足全 BAD 后立即发布阻断，不把慢查询加入这个决定的必要路径。可以并行触发查询作诊断与恢复准备。

BAD、UNKNOWN、pending 或已阻断时，启动有界后台恢复工作。若以一秒核验为部署目标，应保证固定周期加错峰；不能在全故障节点恢复核验上无限指数退避。具体数据路径探测可独立退避，但必须给最大恢复检测间隔。

节点阻断后应能从健康控制面创建受限恢复探测。恢复探测不受普通业务 admission 阻断，但只能使用专用资源、明确白名单和限额，不能成为业务绕过隔离的后门。

### 6.4 公共结果与接口

建议新增版本化接口，避免直接改变现有 C ABI：

```cpp
// 建议接口，并非当前已有 API。
getLocalUbNodeHealth(...);
queryPeerUbNodeHealth(peer, expected_incarnation, ...);
subscribeUbNodeHealth(...);
getTransferFailureDetail(batch, task, ...);
```

故障详情至少包含 raw status、opcode、端口归属有效性、故障范围、执行结果是否未知、DMA 是否安全及建议恢复动作。

公开 TransferStatus 当前主要只有状态和字节数，不足以承载上述事实。（T4）

## 7. 远端传播与独立核验

### 7.1 端口所有者发布事实

本地 Manager 是事实源。其他节点的一次 status 9、RPC 超时或路径探测失败，只能触发核验，不能直接把目标节点的所有端口标 BAD。

对端实例自己汇总的 status 4 证据可以通过健康摘要和查询传播。这里“查询”读取的是该实例的统一证据账本，不意味着必须重新执行 provider 端口状态查询。

因此，本方案允许：

```text
目标实例 P0、P1 都收到有效 status 4
→ 目标实例本地立即阻断
→ QueryUbNodeHealth 读取该完整证据快照
→ 调用方核验身份、清单和有效性后阻断目标实例 UB
```

整个链路可以不依赖 provider 状态查询。

### 7.2 建议摘要字段

| 字段 | 作用 |
| --- | --- |
| service_instance_id、incarnation | 确认端口事实属于哪个服务实例 |
| inventory_generation、inventory_digest、complete | 确认使用的是完整、非空且一致的服务端口集合 |
| total、bad、good、unknown | 区分完全故障、部分故障和未知 |
| health_epoch、observation_sequence | 分离事实变化和刷新进度 |
| node_admission、block_id、block_reason | 表示当前锁存及原因 |
| evidence_age、remaining_validity、verification_pending | 防止陈旧结果伪装为新结果 |
| evidence_source_mask、capabilities | 标明可使用 status 4、查询、事件或探测等来源 |

本地保留逐端口完整事实；跨节点默认传聚合摘要与清单摘要，必要时通过受控诊断接口读取明细。请求与响应关联 nonce/请求 ID，结合现有控制面身份校验，防止把其他实例的响应应用到当前 peer。

聚合事实与锁存状态必须分开传。BAD 已过期但节点仍锁存时，`bad` 可以不等于 `total`；此时消费者不能只凭计数自行解除阻断。

### 7.3 远端准入策略

为延续独立核验原则，建议：

- 被动摘要更新健康视图，触发立即 Query。
- 本地所有者无需 Query，直接使用自己的证据账本。
- 远端通过独立控制面 Query 确认 owner 当前 Unavailable 后建立本地 peer 阻断。
- Query 可以确认由 status 4 形成的事实，不强制替换成 provider 查询证据。
- 远端已阻断时，只有独立 Query 返回合格的新恢复事实才解除；旧摘要、pending-only 或 RPC/TCP 成功不能解除。
- Query 失败时，未阻断的 peer 保持 Suspect，已阻断的 peer 保持阻断。

每个进程按 peer/incarnation 去重任务；健康恢复摘要可以触发一次提前核验，不能制造多个并行轮询器。

远端查询只针对相关或已隔离 peer，不全量周期 all-to-all。节点自有摘要传播可以集中汇聚，但活跃关系极密时定向查询总量仍需预算，不能笼统宣称所有控制面开销都是 O(N)。

## 8. Mooncake Store 的配合

### 8.1 健康表与身份映射

Store 增加 HealthRegistry，将 TENT service instance 映射到 Segment、transport endpoint、owner client 和 host。一个节点实例拥有多个 Segment 时，其 UB 阻断覆盖这些 Segment 的 UB 访问，但不改变对象位置。

健康变化与拓扑版本解耦：端口变化不需要等待成员增删才同步。Master 重启或 HA 切换后，失去的新鲜事实标记待核验，不能恢复出陈旧的“健康”结论。

### 8.2 读副本选择

读候选先做对象版本、COMPLETE、lease 和传输 eligibility 检查，再做 locality/负载评分，最后在实际提交前复核。

当前 `SelectBestReplica` 在评分未选出候选后可能返回 `first_memory`；部分底层 Get 直接找第一个 COMPLETE。（S1、S2）

因此必须引入独立 eligibility，并覆盖所有回退分支。不能只给阻断节点一个极差分数。

本地 memcpy 不依赖 UB；目标节点 UB 被阻断，但相同副本通过其他已验证 Transport 可读时，可以按策略使用它。没有任何可用方式时返回可识别的不可用/待恢复错误，不能当作对象不存在。

### 8.3 新写入与 Master 分配

Store 的典型 Put 是：

```text
PutStart 分配目标地址
→ Client WRITE
→ PutEnd 或 PutRevoke
```

Master 在 allocation 的 preferred、排序、随机回退和批量路径上统一执行目标传输 eligibility，排除已核验阻断且没有允许替代方式的目标。

不要通过永久 Unmount 或全局 `IsServing=false` 代替 UB 专用隔离，以免误伤其他传输和已有副本。Client 得到分配结果后仍需再次复核，处理分配后节点才故障的竞争。（S3）

Master 不掌握客户端每条 peer 路径的全部信息，因此其节点过滤是第一道门禁，Client/TENT 的请求级检查仍必需。

### 8.4 迁移、复制与批量路径

迁移源和目标使用同一份节点健康表，分别检查读源、写目标和所用 Transport。短期 UB 全故障不自动触发对象所有权切换、删除副本或大规模迁移。

覆盖单键、Batch、range/session、复制、迁移和后台任务，避免只有一种入口遵守隔离。恢复后可逐步增加业务量，后台任务使用独立额度，防止回切突发。

### 8.5 状态和资源契约

Store 适配器必须保留 TENT 的故障详情及 DMA 安全状态。当前普通传输适配的 switch 没有把 TIMEOUT 作为终态，可能拖到外层 60 秒超时，应作为接入时的独立修复项。（S4）

WRITE 全端口故障后：

- 不能把 status 4 自动解释成远端未写入。
- 不能在旧 DMA 未安全结束时立即 PutRevoke 并复用目标地址。
- 换节点、换 Transport 或重新分配仍要满足重放契约。
- 只有数据传输与发布条件满足后才能 PutEnd。

READ 换副本前，也须保证旧 READ 不再写同一个接收区，并继续维护对象版本、lease 与 checksum。节点准入状态不替代这些契约。

## 9. 关键场景时序

### 9.1 两个 status 4 直接触发本地节点隔离

1. 完整清单为 P0、P1，当前 incarnation 和 inventory 固定。
2. P0 的合格完成返回 status 4；Manager 记录 P0=BAD，暂停 P0 新业务。
3. P1 尚 UNKNOWN，节点为 Degraded，不新增全故障阻断。
4. P1 随后返回合格 status 4，且 P0 的 BAD 仍有效。
5. Manager 在同一清单版本下确认 P0/P1 全 BAD，原子发布 Unavailable 和新 block_id。
6. TENT 拒绝该实例新的普通 UB 提交；在途请求继续完成或排空。
7. Store 读取新摘要，远端消费者发起 Query；Query 读取证据账本后确认隔离。
8. 此流程不等待 provider 状态查询。

### 9.2 全故障后单端口恢复

1. 节点已锁存 Unavailable。
2. 恢复 Coordinator 针对 P0 创建受限探测，处理旧资源和恢复屏障。
3. 当前代次的探测成功，或取得等价的合格 GOOD。
4. P0=GOOD，推进端口恢复 epoch 与 health epoch，解除节点全故障阻断。
5. P1 仍 BAD，节点为 Degraded；业务只能使用合格的 P0 路径。
6. 远端收到恢复摘要后立即 Query，取得同一实例的新恢复事实后解除 peer 阻断。
7. P0 的旧代次 status 4 晚到，只参与旧请求清理，不再次隔离新实例。
8. 如果当前代次再出现新的 status 4，则重新按完整集合判定。

### 9.3 没有 status 4，但路径持续超时

status 9 或 ACK timeout 触发 peer 查询、路径隔离和探测，不直接标记全部端口 BAD。可使用合格 provider 查询或具有端口语义的事件补充事实。

若只知道某个 peer 方向不可达，而端口本身没有 BAD 证据，应按路径故障处理。即使所有当前业务路径都失败，也不能伪称已确认所有物理端口故障；请求仍应有界失败、使用其他合法传输或等待恢复。

### 9.4 清单变化或进程重启

进程重启产生新 incarnation，旧证据与旧查询任务不得迁移到新实例。远端先等待新实例的清单和请求级验证，不能用旧实例的一条 GOOD 解封新实例。

同一实例受控变更端口集合时，进入 Reconfiguring，冻结相关提交，核验完整新清单，再原子切换代次。原本阻断的实例不能仅因扩大清单、把新增端口置 UNKNOWN 就自动恢复；要有新集合的合格 GOOD 或明确的受控重配置完成依据。

如果新集合为空，应标记未配置/不可服务 UB，而不是构造“零个端口全部故障”。

## 10. 并发、资源与观测

### 10.1 并发规则

- completion 线程只做有限校验和有界事件投递，不同步调用慢 provider 查询或 RPC。
- 权威证据投递不能静默丢失；队列满时设置按端口合并的 pending latch/dirty 标志，由 Manager 补读，或走有界同步原子更新。
- Manager 单点归并 revision，发布不可变快照；读者不在热路径扫描所有端口。
- 不持有 JFC/Jetty、EndpointStore 或 Store 路由锁做查询、RPC、回调或 join。
- 状态回调在锁外执行；接收端检查 epoch，允许通知重传且幂等。
- shutdown 先关闭入口、排空事件并停止探测/查询线程，再释放 Context。
- 阻断与恢复不清零物理配额；旧在途资源继续记账。

### 10.2 配置建议

以下为新增设计项，并非当前可用配置：

| 配置 | 目的 |
| --- | --- |
| status4_port_fault_profile | 选择已验证的 provider/status 4 端口语义 |
| physical_port_inventory | 服务实例的端口范围与映射来源 |
| evidence_validity | GOOD/BAD 的有效窗口及最大组合窗口 |
| node_verify_interval / rpc_timeout | 节点核验与目标恢复检测窗口 |
| recovery_probe_budget | 限制恢复探测并发和资源 |
| failure_detail_retention | 保留旧尝试诊断与归属记录的有界时间 |
| strict_inventory_validation | 缺少完整清单时明确拒绝承诺该隔离能力 |

status 4 作为证据本身不需要重复配置批准；profile 负责将本文约定准确落实到目标 provider。参数默认值应由目标硬件测试确定，不能把示意的一秒周期直接解释成一秒必然恢复。

### 10.3 指标

至少记录：

- 按来源的有效 BAD/GOOD、被拒绝的旧证据、归属失败、清单不完整。
- 本地阻断次数、远端阻断次数、status 4 全覆盖触发次数。
- 第一个 BAD 到全部 BAD、到本地阻断、到远端停止提交的分阶段耗时。
- 证据过期但保持阻断、查询失败、恢复探测与重建失败。
- 首个合格 GOOD 到本地/远端解除阻断的时间。
- 请求被节点门禁拒绝和被请求级路径门禁拒绝的数量，二者分开。
- 隔离中的资源、在途 WR、保留内存和恢复探测开销。
- 调度分数、TCP 成功等非权威信号不得计为端口恢复次数。

## 11. 验证矩阵与交付顺序

### 11.1 必须通过的确定性测试

| 用例 | 预期结果 |
| --- | --- |
| P0=status 4，P1 UNKNOWN | 只隔离 P0，不新增节点阻断 |
| P0=status 4，P1 GOOD | 节点退化，P1 可走请求级检查 |
| P0/P1 当前有效 status 4 | 不调用 provider 查询也能本地立即阻断 |
| status 4 与查询/事件 BAD 混合全覆盖 | 可以阻断，不要求相同证据来源 |
| 清单不完整、空清单、重复 EID | 不误判完整全故障；配置门禁给出明确原因 |
| primary EID 无法归属真实端口 | 仅路径隔离，不扩散成所有端口 BAD |
| 两个 BAD 有效期不重叠 | 不新增当前全故障阻断 |
| 已阻断后所有 BAD 过期 | 保持阻断，进入待核验 |
| 未阻断时查询失败 | 不新增节点硬隔离 |
| 已阻断时查询失败/返回旧缓存 | 不解除阻断 |
| 一个当前合格 GOOD | 解除全故障阻断，其他路径仍受限 |
| 新 GOOD 后旧 status 4 晚到 | 不覆盖新恢复代次 |
| 新 BAD 后旧成功完成/旧查询晚到 | 不误恢复 |
| 超时/quiesce 分支产生真实 status 4 | 统一入口记录有效观测，不重复记账 |
| entity marker、重复 token | 不生成重复端口证据 |
| 证据队列满 | 权威信号被合并保留，不静默丢失 |
| 远端 Query 只读取 status 4 账本 | 可以确认隔离，不要求 provider 查询 |
| provider 只支持 Link Up | 不把单纯 Link Up 当已恢复数据面 |
| 节点恢复但 peer 路径未恢复 | 节点解除，具体请求仍被正确拦截 |
| Store scorer 全部拒绝后的回退 | 不能重新选择 UB 已阻断目标 |
| TIMEOUT、取消、WRITE 未知完成 | 有界处理且不提前复用缓冲区或注册地址 |
| Batch/session/迁移路径 | 使用同一健康视图和门禁 |

### 11.2 硬件与端到端验收

在目标 provider、驱动和 profile 上验证实际 status 4 语义与端口归属。至少注入单端口、全部端口、单向黑洞、peer 重启、设备重建、延迟 CQE、恢复抖动和长时间故障。

验收分别比较：

- 不误隔离：单端口故障仍能使用其他合格端口。
- 不漏隔离：全部有效 status 4 后，本地新 UB 提交及时停止，远端在约定核验窗口内收敛。
- 不误恢复：未知、旧 CQE、旧 Query、TCP 成功都不能解封。
- 能恢复：无业务流量时仍能取得合格 GOOD，单端口恢复不必等待所有端口恢复。
- 数据和内存安全：隔离、换路、换节点及回收不破坏原有 DMA/发布契约。
- 业务覆盖：读、写、批量和迁移不绕过门禁，UB 隔离不导致错误的对象删除。

### 11.3 分阶段实现

| 阶段 | 内容 | 优先级 |
| --- | --- | --- |
| A | inventory、端口归属、status 4 语义、代次与统一观测入口 | P0 |
| B | 证据归并、全故障锁存、提交门禁、迟到结果过滤 | P0 |
| C | 独立查询、恢复探测、资源重建与恢复屏障 | P0；投产前必须形成闭环 |
| D | Store eligibility、Master 分配、批量/迁移接入、状态适配 | P0 |
| E | 规模化控制面预算、负载调度复用、指标与调优 | P1；负载权重可另行上线 |

A/B 可以验证“status 4 足以触发隔离”；不能只实现阻断而没有 C/D 就对外宣称可靠方案完成。

## 12. 源码依据与本次交付范围

以下路径按本次基线定位。UMDK 文件位于同级仓库，路径相对 UMDK 根目录。

| 编号 | 文件与定位 | 当前事实 |
| --- | --- | --- |
| T1 | [urma_adapter.h](../../mooncake-transfer-engine/tent/include/tent/transport/ub/urma_adapter.h)：DeviceInfo、Completion | 当前主要有设备/EID、原始 CR、token、Jetty ID，缺完整端口账本 |
| T2 | [urma_adapter.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/urma_adapter.cpp)：classifyCompletion、convertCompletion | 原始状态保留；LOC_ACCESS_ERR 当前归入 MEMORY_ERROR |
| T3 | [workers.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/workers.cpp)：handleCompletion、scanTimeouts | 有 Inflight、超时排空、完成和配额幂等机制 |
| T4 | [types.h](../../mooncake-transfer-engine/tent/include/tent/common/types.h)：TransferStatus、NicLoadStats | 公共完成结果与 NIC 负载结构 |
| T5 | [context.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/context.cpp)：markUnavailable、recordPollSuccess | Context 局部失败与 poll/冷却恢复，不等同节点端口事实 |
| T6 | [rail_monitor.cpp](../../mooncake-transfer-engine/tent/src/transport/ub/rail_monitor.cpp)：refreshCooldownLocked | 冷却结束会解除 rail 暂停，不应直接作为节点恢复证据 |
| T7 | [control_plane.h](../../mooncake-transfer-engine/tent/include/tent/runtime/control_plane.h)：UbBootstrapDesc、ControlClient/Server | 可扩展的 IP 控制面；新健康协议需版本兼容 |
| S1 | [replica_selection.h](../../mooncake-store/include/replica_selection.h)：SelectBestReplica | 评分之外仍有 first_memory 回退，需独立 eligibility |
| S2 | [client_service.cpp](../../mooncake-store/src/client_service.cpp)：FindFirstCompleteReplica、Put、TransferRead/Write | 底层选副本与数据/发布调用链 |
| S3 | [master_service.cpp](../../mooncake-store/src/master_service.cpp)、[allocation_strategy.h](../../mooncake-store/include/allocation_strategy.h) | Master 分配及 preferred/回退路径 |
| S4 | [transfer_task.cpp](../../mooncake-store/src/transfer_task.cpp)：TransferEngineOperationState::check_task_status | 普通状态适配需要完善 TIMEOUT 与故障详情传递 |
| U1 | `src/urma/lib/urma/bond/bondp_cp_user_ctl.c`：bondp_user_ctl_query_port_status | 本地已有查询，读取启用成员的健康标记，不是通用物理 UDMA 查询 |
| U2 | `src/urma/lib/urma/bond/include/urma_ubagg.h` | 查询 ABI、GOOD/BAD 与 chip/die/port 字段 |
| U3 | `src/urma/lib/urma/bond/bondp_datapath.c`、`src/urma/lib/urma/bond/bondp_dp_health.c` | bonding 中端口 BAD 标记与恢复更新 |
| U4 | `src/urma/lib/urma/core/include/urma_opcode.h` | 原始 CR 定义；provider 契约与归属适配不可省略 |

本次仅提交设计文档与基础设计中的关联链接，没有实现上述代码、修改运行配置或开展 UB 硬件测试。文档的可执行验收要求列在第 11 节，不能用源码分析代替硬件结果。

文档检查覆盖 UTF-8、文件结尾、空白、冲突标记、代码围栏、本仓库相对链接和 UMDK 引用路径。已尝试 `docs/make.bat html`，但本机缺少 `sphinx-build`；`pre-commit` 及其 Python 模块也未安装，相关检查未完成。文件位于 `docs/analyze/`，未修改 Sphinx 导航。
