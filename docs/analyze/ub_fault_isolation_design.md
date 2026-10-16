# Mooncake UB 多端口故障隔离适配设计

> 设计状态：待实现；更新日期：2026-10-16。
> 运行时代码基线：`supercache-snapshot-930` / `a2f25168`；文档提交不改变该代码基线。
> 结构参考：认证读取的 [yuanrong issue #28](https://gitcode.com/hc147/yuanrong-datasystem/issues/28)；实现对照：同级 yuanrong-datasystem `eacc9857`。
> 关联：[Jetty 池化与生命周期](ub_jetty_pool_lifecycle_design.md)、[端口故障触发的时延成本调度](ub_load_scheduling_design.md)。本文只设计故障事实、业务准入与恢复，不接管资源池或调度权重。

## 0. yuanrong 与 Supercache 的读写变化及适配原则

### 0.1 角色职责对应

业务仍是写入 key/value、按 key 读取，但这是职责映射，不是把 Worker 类改名为 Store。

| yuanrong 角色 | Supercache 中的对应职责 |
| --- | --- |
| Client：发起 GET/SET、选择 Worker、管理发送/接收 buffer | 请求侧 Client：向 Master 查询副本或申请空间，再调用 TE 传输 |
| Worker：管理对象、内存、业务 RPC，并回写数据 | Master 管对象元数据和分配；提供内存的 Store 进程承载注册内存及数据服务 |
| Client/Worker 内部 UB 组件 | TE 的 UbTransport / UrmaContext / UrmaEndpoint |

Supercache 源码中的 Client 也可提供存储内存。同一进程可以同时发起 PUT/GET、挂载 segment 供他人读写；`global_segment_size=0` 时可以只做请求客户端。独立 Store 服务也复用 RealClient 实现。因此本文始终区分请求发起角色与数据持有角色，节点隔离按进程身份覆盖其两种角色，而不按 Client 类名判断。参见 [RealClient 初始化](../../mooncake-store/src/real_client.cpp)。

### 0.2 SET/PUT：分配和发布协调发生变化

```mermaid
sequenceDiagram
    participant YC as yuanrong Client
    participant YW as yuanrong Worker
    participant C as Supercache Client
    participant M as Supercache Master
    participant S as Supercache Store
    rect rgb(240, 245, 255)
        YC->>YW: 申请或准备对象空间（控制）
        YW-->>YC: 目标空间与 UB 信息
        YC->>YW: UB WRITE 数据
        YC->>YW: Publish（控制）
        YW-->>YC: 发布结果
    end
    rect rgb(240, 255, 245)
        C->>M: PutStart（控制）
        M-->>C: 副本地址
        C->>S: TE WRITE 数据
        C->>M: PutEnd 或 PutRevoke（控制）
        Note over M,S: Master 协调分配与确认，Store 承载数据
    end
```

Supercache 的分配和确认主要与 Master 交互，目标 Store 通常不执行每次 WRITE 对应的对象级 Publish RPC。当前 `Put()` 遇到 OBJECT_ALREADY_EXISTS 返回成功且不覆盖旧值，更新有独立 `Upsert()` 路径；迁移 yuanrong SET 时须按存在性/覆盖语义选择 API。故障 gate 必须同时覆盖 Put/Upsert，而非只拦截一种写入口。

代码对照：[yuanrong UB 写入](../../../yuanrong-datasystem/src/datasystem/client/object_cache/client_worker_api/client_worker_base_api.cpp)、[Supercache Put/Upsert](../../mooncake-store/src/client_service.cpp)。

### 0.3 GET：Worker WRITE 回写变为 Client 主动 READ

```mermaid
sequenceDiagram
    participant YC as yuanrong Client
    participant YW as yuanrong Worker
    participant C as Supercache Client / TE
    participant M as Supercache Master
    participant S as Supercache Store
    rect rgb(240, 245, 255)
        YC->>YW: Get / QueryAndGet，提供接收 buffer
        YW->>YC: Worker 发起 UB WRITE 数据回写
        Note over YW: Worker 提交 WR、处理 CQE
        YW-->>YC: GET 业务结果，可附健康摘要
    end
    rect rgb(240, 255, 245)
        C->>M: 查询副本（控制）
        M-->>C: 副本地址与 lease
        C->>S: Client 发起 TE READ 请求
        S-->>C: READ 数据返回（硬件数据流）
        Note over C: Client 处理 CQE，检查结果与 lease
        Note over C,S: 数据阶段没有对应的对象级 GET 响应，健康走独立控制 Query
    end
```

前者由 Worker 提交 WR、处理 CQE，业务响应可携带健康摘要；后者通常由请求侧 Client 的实际 TE 执行者提交 READ、处理 CQE，数据阶段没有对应的对象级 Worker 响应。READ 的数据流是 Store → Client，但“本地”仍指提交 WR、接收 CQE 的一方，不随数据方向翻转。存在 TE 代理时以实际执行进程为准，再向业务调用方映射错误。

以上只描述远端内存 UB 主路径，本地缓存、memcpy、TCP、磁盘和 offload 分支按实际 opcode 判断。当前 Copy/Move 的内存数据阶段由源 Store WRITE 到目标；实际采用 READ 的其他拉取路径另按 READ 处理。

代码对照：[yuanrong Get 回写](../../../yuanrong-datasystem/src/datasystem/worker/object_cache/service/worker_query_and_get_impl.cpp)、[Supercache TransferRead/TransferWrite 与复制](../../mooncake-store/src/client_service.cpp)。

### 0.4 yuanrong 可靠性能力如何适配

| 参考能力 | Supercache 适配方式 |
| --- | --- |
| CQE 异常触发端口核验，不直接隔离 | CQE 决定当前操作失败及查询侧；本地 urma_user_ctl 返回事实决定后续隔离 |
| Client/Worker 本地监控和业务准入 | 每个四端口域一个 Monitor；同域 context 共享事实；进程级 NodeBusinessAdmission 聚合域 |
| Worker 在 GET 响应携带健康摘要 | Store 独立提供健康 Query，控制通道传播摘要；不依赖不存在的 GET 数据响应 |
| WorkerRouter/Client 健康过滤 | Master 避让新分配；请求侧 Client 过滤读副本和缓存 descriptor；TE 最终守卫 |
| Worker 间传输隔离 | 对 Copy/Move 等按实际 WRITE/READ 执行者处理，源/目标都检查准入 |
| 端口恢复查询 | 故障侧本地持续至 bad=0；远端恢复轮询确认可服务即停止，不把两者混为一个任务 |

### 0.5 本方案固定的底层契约

**CQE 处理当前操作，端口 Query 处理后续准入，两条链分开。端口健康不能把已经失败的读写改成成功。**

`4 LOC_ACCESS_ERR` 对 READ/WRITE 都只触发提交端本地端口核验；`9 ACK_TIMEOUT_ERR` 对 READ/WRITE 都触发提交端与对端各自查询本地 context，不能固定解释为远端故障。查询失败不等于全坏，双方端口都健康则不建立端口隔离，继续按连接/链路及请求错误处理。具体 SDK 状态映射在适配器中校验，不把裸数值扩展到其他 provider。

## 1. Story 整体设计

### 1.1 功能描述

形成“异常观测 → 四端口核验 → 节点业务隔离 → 健康传播 → 选点过滤 → 核验恢复”的闭环。一个域四口全坏先限制该域；全部配置的数据服务域确认不可用，才关闭节点新数据业务。当前失败操作的结果不等待健康查询，更不由查询改写。

#### 影响范围

覆盖 Store Client、承载 segment 的存储进程、Master、TransferSubmitter、TE 的 classic kunpeng_transport URMA 路径，以及 HA/CVM 分配链路。Mooncake 的存储节点通常由挂载 segment 的 Client 进程承载，不能直接套用 yuanrong Worker 类和 RPC。

#### 用户体验

部分端口故障时继续使用合法路径；节点隔离后，新 Put/Get、批量、异步和后台数据阶段快速失败或选择健康副本。控制查询、诊断和既有操作结算继续；节点、对象和副本记录保留，恢复后无需重新挂载即可重新参与业务，资源 fatal 除外。

### 1.2 功能目标与边界

#### 成功标准

- 本地与远端都以覆盖最新异常的有效核验作为端口事实，CQE/AE 和普通心跳不直接决定节点隔离。
- Store 入口、缓存快速路径、传输提交、Master 分配与跨 Master 转发使用同一身份和准入契约，不能通过切换策略绕过隔离。
- 已接纳操作保留硬件完成与缓冲责任，不把逻辑失败等同于 DMA 已停止。

#### 恢复标准

当前节点身份下，至少一个域经有效核验满足 `good > 0 && resource_online`，才可解除全节点隔离；其他域仍异常时进入 DEGRADED。远端消费者必须独立 Query 确认，摘要、PORT_ACTIVE 或业务成功只能触发核验。

#### 隔离含义

隔离的是该进程的新数据业务，不是物理主机成员关系。一个进程挂载的所有 segment 共用节点 gate；TCP、memcpy、staging、文件路径也执行该 gate。保留元数据、数据和副本，不自动 unmount、迁移或重建。

这是对 issue #28 的明确适配：沿用事实与准入分离的闭环，但按本项目已确定的边界关闭所有新数据策略，不照搬参考实现中的部分 TCP 回退行为。

#### 非目标边界

不新增 TCP 故障检测，不设计拥塞/容量评分，不扩展通用成员服务，不重做 Jetty 生命周期。硬件核验首期只覆盖 classic URMA，不宣称 TENT/generic RDMA 已有同等能力。必要 hook 或核验能力缺失时不能启用 enforce。

### 1.3 术语说明

| 术语 | 本文含义 |
| --- | --- |
| Path Observation | CQE/AE、超时等异常线索，包含实际 opcode、执行者、对端身份与 provider 原因；不是节点故障事实 |
| Port Health Fact | provider 查询成功、身份与结果合法、覆盖最新异常的端口快照 |
| Local Node Admission | `NodeBusinessAdmission` 聚合本进程服务域，唯一写入本地节点业务状态 |
| Peer Admission | 消费者以独立 Query 确认的远端准入状态，用于选点和提交前过滤 |
| Node Health Registry | 每个 Master/Client 的带身份健康视图，独立于成员状态和 segment 挂载状态 |
| 服务域 | provider 能确定边界的已配置数据端口集合及其资源状态；同一物理端口集合可被多个 context 共享 |
| Requester / Provider | READ 的发起方/数据提供方；不能套用“Worker 总是 WRITE 回写”的方向 |
| Writer / Receiver | WRITE 的执行方/目标方；PUT 常见为发起 Client 写入存储节点 |

## 2. 场景分析

### 2.0 问题上下文：AS-IS / TO-BE

| 方面 | AS-IS：当前实现 | TO-BE：本方案 |
| --- | --- | --- |
| 故障判断 | context active、post/poll 失败与 AE 局部处理 | 观测与核验分离，聚合配置域形成节点准入 |
| 恢复 | monitor 存在周期无条件置 active 的逻辑 | 同身份有效核验恢复，fatal 需资源重建 |
| 业务隔离 | 无贯通 Store/Master/TE 的节点 gate | 本地入口、远端选点、最终 post 分层守卫 |
| 控制传播 | 普通 Probe 只证明响应，缺少端口查询协议 | 版本化 Query、摘要、完整/增量视图 |
| 生命周期 | context 故障与 WR/任务责任分散 | 隔离只停止新提交，Jetty/operation owner 负责安全收敛 |

以下场景固定采用目标 UB 定义：E4 为 LOC_ACCESS_ERR，E9 为 ACK_TIMEOUT_ERR。READ/WRITE 的查询归属一致，底层数据方向不改变“本地”的定义；端口故障归属仍以各方查询结果为准。

### 2.1 UC-1 Client PUT：WRITE 收到 E4

Client 将当前 WR 标记失败，不报告写入成功，不自动重放可能已部分写入的 WRITE；同时异步触发自身四端口 Monitor。确认四口全坏才关闭该域，全部服务域不可用才关闭本地节点。端口健康时仍保留本次访问错误，检查地址、长度、权限和 segment 生命周期；不因此标坏 Store。

### 2.2 UC-2 Client GET：READ 收到 E4

本地仍是发起 READ 的 Client。当前 READ 失败，接收内容无效，不交给业务；staging READ 禁止执行成功路径的 H2D。只触发本端端口查询，端口健康不能证明本地注册已恢复。重试须检查注册、范围、lease，并等待关联 WR 安全收敛；本地隔离时不能靠换 Store 绕过。

### 2.3 UC-3 Client PUT：WRITE 收到 E9

Client 当前 WRITE 失败，但保留远端可能已写入的不确定性；不自动重放到原地址或其他目标。同时启动本地 Monitor，并通过控制 Query 触发 Store 自己查询本地 context。

谁确认四口全坏就关闭谁的域，各方再按完整服务域集合聚合节点状态。Client 确认 Store 节点隔离后排除它，Master 独立核验后停止新分配。双方有健康端口时不建立端口隔离；远端 RPC 失败也不证明端口全坏。旧 PutStart 不绕过新 gate，未 post 部分进入现有撤销流程，已 post 前缀保留原 owner。

### 2.4 UC-4 Client GET：READ 收到 E9

Client 将本次内容标为无效，同时核验自身与 Store。源 Store 确认隔离时过滤该副本；本地节点也隔离时停止新的 READ。旧操作及关联 WR 安全收敛后，才可在 lease/注册有效且本地准入允许时重新读取或换健康副本，不能让新旧 READ 并发写同一 buffer。

对象存在但全部副本不可服务时返回节点不可用，不返回对象不存在、不建立负缓存。双方端口健康仍不改变当前读取失败的结果。

### 2.5 UC-5 节点间后台任务与恢复

#### 复制、迁移和实际 READ 路径

| 业务与 opcode | 数据方向 / CQE 接收方 | E4 查询 | E9 查询 |
| --- | --- | --- | --- |
| Client PUT / WRITE | Client → Store / Client | Client 自身 | Client、Store 各查自身 |
| Client GET / READ | Store → Client / Client | Client 自身 | Client、Store 各查自身 |
| Store A Copy/Move / WRITE | A → B / A | A 自身 | A、B 各查自身 |
| Store A 实际拉取 / READ | B → A / A | A 自身 | A、B 各查自身 |

Copy/Move 的源隔离则停止新发送，目标隔离则停止新 WR；符合现有策略时才另选健康源。数据阶段成功并完成提交前不改变副本状态，目标写失败不得提前删除源副本。所有新任务、重试、窗口和 grant 检查双方准入，已 post 部分继续由原 owner 收敛。

#### 多域退化与恢复

一个域四口全坏而另一个域仍可用时保持 DEGRADED。只有全部配置域有效不可用才隔离节点，故障域不能从集合删除。

本地 Monitor 从 bad=4 降为 3/2/1 时立即解除端口隔离，但继续每秒查询至 bad=0；远端消费者经独立 Query 确认节点可服务后解除自己的端口隔离并停止该远端恢复轮询。旧身份、旧 ticket 和旧 CQE 不恢复新实例，端口恢复也不复活退役 Jetty 或清除资源 fatal。

## 3. 方案详细设计

### 3.1 当前分支实现分析

当前 UB 路径在 `UbTransport` 选择设备并提交任务，`UbWorkerPool` 监控 context，URMA endpoint 负责 post、CQE/AE 与连接资源。它们已有局部错误处理，但没有覆盖 Store 业务入口与 Master 分配的节点级闭环。

重点缺口是：monitor 无条件恢复 active；端口状态与资源状态未形成独立事实；普通 Probe 不返回可核验健康证据；上层选点、跨策略传输和 HA 接管缺少统一隔离围栏。具体文件见 3.7，本文提出的新组件和协议均为待实现。

参考 [PeerUbAdmission](../../../yuanrong-datasystem/src/datasystem/common/object_cache/peer_ub_admission.h)、[Worker 本地健康](../../../yuanrong-datasystem/src/datasystem/worker/object_cache/worker_self_port_health.cpp)、[健康同步](../../../yuanrong-datasystem/src/datasystem/cluster/ub_health/ub_health_lease_sync.cpp)、[Client 准入](../../../yuanrong-datasystem/src/datasystem/client/object_cache/transport/transport_layer.cpp) 与 [路由过滤](../../../yuanrong-datasystem/src/datasystem/client/object_cache/routing/ub_health_filter.cpp)。跨仓链接供本地对照，在线入口为 issue #28。参考源码存在随机化恢复与部分被动恢复证据；本文采用 issue 的独立 Query 契约与固定一秒目标周期。

### 3.2 总体设计

```mermaid
flowchart TD
    Event["CQE / AE 观测"] --> Adapter["UrmaPortStatusProvider"]
    Adapter --> Facts["UbHealthManager：域事实与路径准入"]
    Facts --> Gate["NodeBusinessAdmission：本地节点准入"]
    Gate --> Business["Store / TransferSubmitter"]
    Gate --> Sync["NodeHealthSync：摘要与独立 Query"]
    Sync --> Registry["NodeHealthRegistry：Master / Client"]
    Registry --> Select["分配与副本过滤"]
    Business --> TE["TE 通用节点准入 hook"]
    TE --> Post["节点 permit + 路径 permit + Jetty PostGate"]
    Post --> Owner["原 operation owner：完成与安全释放"]
```

Store 只消费通用节点摘要，不依赖 URMA 类型；TE 不依赖 Master/Client 类；Master 不直接调用 provider。端口事实、节点准入、资源生命周期各有唯一所有者，通过快照和通知协作。

#### 3.2.1 唯一隔离判据

节点身份为 `(client_id, transport_incarnation)`，不是 IP、host_id、HCA 名或 vector index。新实例必须通过 Mount/ReMount 的所有权绑定替换，健康响应不能改写 segment ownership。

初始化固定服务域集合并分配 `service_set_generation`。本方案每域预期四端口，需由配置与 provider 确认完整身份集合；多个 context 共享这四口时只建一个域 Monitor，查询使用该域有效的本地 context，生命周期受引用保护。provider 不能证明共享集合时不得误合并。失败、fatal 或暂不可用域仍保留在集合中；变更集合必须显式更新代际。空集合是 NOT_READY，不能利用空集合全称判断得到 all-bad。

端口 all-bad 必须满足：`valid && !verification_pending && expected_total == 4 && total == 4 && bad == 4`，且查询覆盖该域最新异常。域不可用来源为已确认端口 all-bad，或经适配器确认的资源 fatal；两种原因分别记录，不能把 fatal 伪装为端口坏。

节点进入 ISOLATED 的唯一升级条件是：**非空配置服务域集合中，每个域都有覆盖最新异常的有效不可用证据**。单次 CQE、查询失败、普通心跳丢失不能满足该条件。

#### 3.2.2 Admission 状态机

| 状态 | 条件与准入 |
| --- | --- |
| NOT_READY | 未建立服务域或身份绑定，不接纳新数据 |
| AVAILABLE | 所有域资源在线且有效端口全 GOOD，允许业务 |
| DEGRADED | 至少一个域确认可用，其他域/端口退化，只允许合法路径 |
| ISOLATED | 全域确认不可用，关闭节点新数据 |
| UNKNOWN | 证据不足或待核验；不能新增 all-bad 隔离，也不能解除已有关闭 |
| REVALIDATING | HA 接管后的消费者围栏，独立核验前不参与新分配 |
| STOPPING | 退出围栏，停止新业务，继续既有责任收敛 |

UNKNOWN 描述证据状态，不能覆盖已确认的 ISOLATED；已建立可用状态在新增不确定证据下不凭空变为 all-bad，路径是否可提交仍由资源和路径 gate 决定。NOT_READY/REVALIDATING 的关闭原因也不同于已确认故障。

恢复必须同身份、同服务集合代际，且核验覆盖最新异常。至少一个域 `good > 0 && resource_online` 时，ISOLATED 才转 DEGRADED/AVAILABLE。DEV_FATAL、JFC_ERR、EID_CHANGE 形成的资源关闭必须经过明确重新初始化，端口 GOOD 不够。

#### 3.2.3 本地端口健康监控

每个四端口域一个 Monitor，多个 context 的业务异常合并到同一实例。首次异常异步触发核验；同域 single-flight、每秒最多一次。CQE 线程不阻塞调用 urma_user_ctl。启动核验建立初始事实后，全 GOOD 时不持续轮询。

| 本次有效查询 | 端口隔离决定 | 本地后续查询 |
| --- | --- | --- |
| total=4、bad=4 | 建立/保持该域端口隔离 | 每秒继续 |
| total=4、bad=1～3 | 立即取消该域端口隔离，进入退化；资源 fatal 另判 | 每秒继续 |
| total=4、bad=0 | 取消端口隔离，资源在线则健康 | 停止周期查询，后续异常重新触发 |
| 失败、pending、集合不完整或结果非法 | 保留此前确认状态，不新增/解除端口隔离 | 每秒继续核验 |

预期四口却只返回三口时，即使三口全 GOOD 也不能作为恢复证据。端口键用 `(chip_id, die_id, port_idx)`，排序、判重并核对预期集合，再统计 bad。`health_epoch` 表示事实变化，`query_seq` 表示成功新查询，`observation_generation` 表示新异常，分别更新。

删除 monitor 无条件 active 恢复。PORT_DOWN/PORT_ACTIVE 及成功业务可触发核验，不直接改变隔离；资源 fatal 独立关闭路径，恢复需资源重建而非仅端口 GOOD。provider 查询失败不覆写最后确认事实，快照另标验证失败/pending。

#### 3.2.4 E4/E9 统一归因与当前操作处理

CQE 先按类型分类，再读取操作上下文。`FLUSH_ERR_DONE` 等特殊完成走 provider/Jetty 生命周期分支，不能像普通请求 CQE 一样解引用 user_ctx。普通 WR 才通过稳定完成记录提取 opcode、本端/对端、域与操作身份；旧 incarnation 只能结算原操作。

| CQE 状态 | 核验动作（READ/WRITE 一致） | 当前操作 |
| --- | --- | --- |
| `4 LOC_ACCESS_ERR` | 仅本端本地 Monitor | 失败，排查本地地址/长度/权限/注册；不直接标坏任何节点 |
| `9 ACK_TIMEOUT_ERR` | 本端 Monitor + 控制 RPC 触发对端 Monitor，各查询自身 | 失败，保留完成不确定性；不能固定归因为远端故障 |
| 其他错误 | 保留原内存/连接/Jetty 分类规则 | 不统一转成端口故障 |

WRITE 失败不得作为成功副本发布，也不自动重放可能已接受的写入；旧 WR 保持缓冲、注册和完成责任。READ 失败则本次 buffer 内容无效，不交付业务、不执行 staging 成功 H2D；重试前检查 lease/注册并完成旧 WR 收敛，不能与旧 READ 复用同一缓冲并发写入。

对 E9，双方查询各自决定端口隔离，查询健康不反转本次失败。双方有健康端口则继续连接/链路/超时处理；一方查询失败保持待核验，不把 RPC 不通当作四口全坏。适配器使用实际 SDK 状态枚举确认以上映射，上层不直接比较跨 provider 裸数值。

健康 Query 与请求完成链独立：CQE 失败可先返回逻辑失败，但物理资源释放仍须符合原 operation owner 的安全收敛条件。

#### 3.2.5 远端端口查询与恢复调度

复用 Probe action：`ub_port_health.v1` 查询域，`ub_node_health.v1` 查询节点。原 liveness Probe 保持兼容。handler 只读取快照并合并异步刷新，不在串行 listener 等待 provider。

请求带版本、`query_generation`、expected identity、服务集合代际和各域 `required_query_seq`；响应状态为 OK/PENDING/UNSUPPORTED/IDENTITY_MISMATCH。使用稳定新鲜性下界避免永远追赶：

1. 首次下界为零时，服务端返回当前各域 `query_seq + 1`，并触发合并刷新。
2. 调用方沿用该下界重试；只有达到下界、非 pending、快照年龄不超过一秒且覆盖最新异常时返回 OK。
3. 相同 ticket 不抬高下界、不增加异常代际；真实新异常使旧 ticket 失效，重新建立核验下界。集合或身份变化必须重建 ticket。

每 peer single-flight，每进程最多四个并发，一秒目标周期加固定 phase offset。使用有界公平队列，不建立全互连查询矩阵。一次 RPC 总 deadline 一秒，覆盖排队、解析、连接和收发；缓存控制地址。队列饱和记录延迟，不承诺任意规模都能一秒恢复。

对端 Query 只触发对端 Monitor，由对端进程调用自身 context 的 provider，发起方不能直接查询远端 context。E9 的本地刷新与远端 Query 独立调度，不能因一侧失败跳过另一侧。

远端恢复任务在独立 Query 确认至少一个域可服务后解除该消费者的端口隔离并停止轮询，不必等所有端口 GOOD；故障侧本地 Monitor 继续到 bad=0。停止的是恢复轮询，不停止摘要/视图同步；后续新异常或有效变化提示再触发核验。Query 失败、旧身份、非法字段、摘要过期或同步分区均不能解除隔离。控制服务仍可响应，原 Probe success 不代表端口健康。

#### 3.2.6 节点健康摘要传播

`NodeHealthSync` 复用现有控制通信边界，新增版本化 `ReportUbNodeHealthV1`。变化立即合并上报，一秒周期兜底；摘要带身份、集合代际、准入 epoch、状态/原因和域聚合信息，不广播完整 ports 数组。

被动摘要用于发现变化、发起独立 Query，不能直接建立或解除远端端口隔离。接收方只有在核验完成后更新确认视图。成员心跳与健康摘要独立，缺少上报不表示节点退出。

#### 3.2.7 Master/Client 全量健康视图

通过 `GetNodeHealthChangesV1` 首次分页获取完整视图，后续按 registry 版本获取有界增量；版本缺口重新全量。增量内容仍是核验线索，不能替代消费者独立确认。

Master 所有分配入口共用 eligibility：普通/批量 PUT、定向分配、后台目标和 InterMaster 分配。实际提交前复查准入代际，隔离不删除 allocator。Client 在现有副本策略前硬过滤，缓存 descriptor、换源和最终 transfer 再检查。

业务 GetReplicaList 过滤已确认隔离副本；管理查询展示完整记录。PutStart 不授予永久准入，PutEnd/abort 允许既有事务结算。对象存在但全部副本不可服务时返回可识别的节点不可用。

HA 快照/oplog 保存身份绑定、确认准入转换及 epoch，不保存每次端口查询。接管保留 ISOLATED，其余支持能力的 UB 节点进入 REVALIDATING，逐节点核验后开放分配，不让单个无响应节点阻塞全部服务。

CVM 节点向现有 Master 视图中的参与者同步，视图变化重新同步；转发方快速过滤，目的 Master 必须自主核验和提交复查。不开辟新成员协议。

### 3.3 Use Case 时序设计

图中 H 表示本地域 Monitor，A 表示本地 NodeBusinessAdmission，R 表示消费者自己的 NodeHealthRegistry。四口全坏先关闭该域；只有全部配置域确认不可用才关闭节点 gate。隔离生效于各消费者确认后的新准入，不代表全局瞬时撤销已发出的单边 DMA。

CQE 失败处理与健康核验异步进行，当前操作不能因后续端口健康而变为成功。Query 的 PENDING/稳定下界流程见 3.2.5；以下聚焦隔离建立后的业务行为。

#### 3.3.1 UC-1 PUT / WRITE E4：本地关闭后拒绝新业务

```mermaid
sequenceDiagram
    participant B as 业务调用方
    participant C as Client / TE
    participant O as 原 operation owner
    participant H as 本地域 Monitor
    participant A as 本地 Admission
    C->>O: WRITE E4，标记失败并保留在途责任
    C->>H: 异步触发本地四口查询
    H->>H: 本地 user_ctl 核验
    H->>A: 发布域事实，聚合全部配置域
    alt 全部配置域有效不可用
        A->>A: 关闭节点 gate，递增 admission epoch
        A-->>C: 通知停止未 post 阶段
        C->>C: 取消未提交工作，禁止新窗口和新 grant
        B->>C: 后续 Put / Upsert / Get 或缓存快速路径
        C->>A: 获取本地节点许可
        A-->>C: 拒绝 NODE_DATA_ISOLATED
        C-->>B: 返回隔离错误，不切 TCP / memcpy 绕过
    else 至少一个域可服务
        A-->>C: 节点不隔离，仅过滤不可用域
        Note over C,H: 当前 E4 仍是失败，检查注册、地址与权限
    else 查询失败或证据不足
        A->>A: 保留此前确认状态，不新建或解除隔离
    end
    O->>O: 已 post WR 按真实完成安全收敛
    C->>C: 按现有 PUT 失败路径撤销未成功分配
    Note over B,O: 控制结算继续，失败前缀不发布成功，也不自动重放 WRITE
```

如果该 Client 同时承载 segment，本地节点关闭覆盖其请求方与数据持有方角色；远端消费者仍须独立核验后更新自己的过滤视图。

#### 3.3.2 UC-2 GET / READ E4：不能用换源绕过本地隔离

```mermaid
sequenceDiagram
    participant B as 业务调用方
    participant C as Client / TE
    participant O as 原 operation owner
    participant H as 本地域 Monitor
    participant A as 本地 Admission
    C->>O: READ E4，结果无效，禁止成功 H2D
    C->>H: 只触发本地端口核验
    H->>A: 发布有效域事实并聚合
    alt 全部配置域不可用或本地已确认隔离
        A->>A: 关闭或保持节点 gate
        C->>A: 重读、换源或新窗口前申请许可
        A-->>C: 拒绝
        C-->>B: 读取失败，后续新数据返回隔离错误
        Note over C,A: 即使其他 Store 健康，也不能绕过本地 gate
    else 本地可服务且注册与 lease 有效
        O->>O: 原操作及关联 WR 安全收敛
        C->>A: 新 READ 重新获取许可
        A-->>C: 允许后续合法读取
        Note over C,O: 旧操作可能写入期间，不让新 READ 复用同一 buffer
    end
    O->>O: 保留实际 buffer / MR / batch 至可安全释放
    Note over B,H: 端口健康不改写当前失败结果，失败内容不交付业务
```

#### 3.3.3 UC-3 PUT / WRITE E9：两端分别隔离，Master 避让新分配

```mermaid
sequenceDiagram
    participant C as Client / TE
    participant A as Client 本地 Admission / Monitor
    participant S as Store Admission / Monitor
    participant R as Client Peer Registry
    participant M as Master / Registry
    C->>C: 当前 WRITE 失败，保留可能已写入，不自动重放
    par 本端核验
        C->>A: 本地刷新
        A->>A: 查询自身 context，聚合本地准入
    and 对端核验
        C->>S: 控制 Query 触发对端本地查询
        S->>S: 查询自身 context，聚合 Store 准入
        S-->>R: 返回同身份有效事实或 PENDING / 失败
    end
    opt Client 全部服务域确认不可用
        A->>A: 关闭本地节点 gate
        A-->>C: 拒绝后续数据、重试和新 grant
    end
    opt Store 全部服务域确认不可用
        S->>S: 关闭自身新数据 gate，保留控制服务
        R->>R: 独立 Query 确认后关闭该 Store 的远端准入
        C->>R: 检查缓存目标或已取得的 PutStart 结果
        R-->>C: 拒绝后续新提交，不沿旧描述符继续写
        S-->>M: 发送变化摘要，仅作核验线索
        M->>S: 独立 Query
        S-->>M: 返回有效不可用事实
        M->>M: Registry 确认隔离，过滤后续分配候选
        M->>M: 实际分配提交前再次复查身份与准入
    end
    Note over C,M: 未 post 部分走既有撤销，已 post 保留 owner；PutEnd / abort 控制结算继续
    Note over C,S: 双方有健康域不建立端口隔离；查询失败不等于四口全坏
```

隔离不删除节点、segment、对象或副本，不自动迁移。Client 与 Master 的确认存在传播间隔，双方必须各自执行最终准入；一方确认不能替另一方建立健康事实。

#### 3.3.4 UC-4 GET / READ E9：隔离读源后过滤缓存并换健康副本

```mermaid
sequenceDiagram
    participant B as 业务调用方
    participant C as Client / TE
    participant A as Client 本地 Admission / Monitor
    participant S as Store Admission / Monitor
    participant R as Client Peer Registry
    participant O as 原 operation owner
    C->>O: 当前 READ 失败，内容无效，保留旧 WR 责任
    par 本端核验
        C->>A: 本地 Query，更新自身准入
    and 对端核验
        C->>S: 独立 Query，Store 查询自身 context
        S-->>R: 同身份有效事实或待核验
    end
    opt Store 全域确认不可用
        S->>S: 关闭本地新数据 gate
        R->>R: 独立确认后过滤该读源及缓存 descriptor
    end
    alt Client 本地已隔离
        C->>A: 申请重读或换源许可
        A-->>C: 拒绝
        C-->>B: 本地隔离，禁止换源绕过
    else Client 可服务
        O->>O: 原操作及关联 WR 安全收敛
        C->>R: 按原副本策略过滤已隔离读源
        alt 有健康副本且 lease / 注册有效
            R-->>C: 返回合法候选
            C->>C: 取得节点、路径和 Jetty 许可后提交新 READ
        else 对象存在但无可服务副本
            C-->>B: NODE_DATA_UNAVAILABLE
            Note over B,R: 不返回 OBJECT_NOT_FOUND，不形成负缓存，不删除副本
        end
    end
    Note over C,S: Query 失败保持旧确认状态；端口健康也不使本次 READ 成功
```

#### 3.3.5 UC-5 后台任务与恢复：停止新阶段，独立核验后重新开放

后台 Copy/Move 需要在任务出队、重试和新传输阶段检查双方，不能只在任务创建时检查。

```mermaid
sequenceDiagram
    participant T as Copy / Move 执行器
    participant A as 源节点本地 Admission
    participant R as 目标 Peer Registry
    participant O as 原 operation owner
    T->>A: 出队或重试时检查源节点
    T->>R: 检查目标节点确认视图
    alt 源或目标已确认隔离
        T->>T: 停止未 post 阶段、窗口和新 grant
        T->>O: 保留已 post 部分，等待安全收敛
        T->>T: 通过既有控制流程上报失败或撤销
        Note over T,O: 目标失败不提前删除源副本，不因隔离释放 DMA 缓冲
    else 双方可服务
        T->>T: 最终提交前再次获取节点、路径和 Jetty 许可
        T->>T: 数据成功且提交完成后，才变更副本状态
    end
```

恢复只开放后续合法操作，不复活失败任务或退役资源。

```mermaid
sequenceDiagram
    participant H as 故障侧 Monitor
    participant A as 本地 Admission
    participant R as Master / Client Registry
    participant B as 业务分配或副本选择
    H->>H: 每秒查询，域 bad 从 4 降至 3
    H->>A: 有效事实覆盖最新异常
    alt 至少一个配置域可用且资源在线
        A->>A: 解除节点隔离，更新 epoch，进入 DEGRADED
        A-->>R: 恢复摘要，仅触发核验
        R->>H: 独立 Query，验证身份、集合与新鲜性
        H-->>R: 返回有效可服务事实
        R->>R: 解除消费者隔离，停止远端恢复轮询
        B->>R: 后续分配或副本选择
        R-->>B: 允许参与，实际提交仍复查准入
    else 仍无可用域或资源 fatal 未修复
        A->>A: 保持节点关闭，不因端口部分 GOOD 强行开放
    end
    H->>H: 本地继续每秒查至 bad=0 才停止周期轮询
    Note over H,R: Query 失败或旧身份保持关闭；旧 WR / Jetty 按原生命周期结算
```

### 3.4 核心类与数据结构

#### 3.4.1 类图

```mermaid
classDiagram
    class UrmaPortStatusProvider
    class UbHealthManager
    class NodeBusinessAdmission
    class NodeHealthRegistry
    class NodeHealthSync
    class OperationOwner
    UbHealthManager --> UrmaPortStatusProvider : 查询正式 ABI
    NodeBusinessAdmission --> UbHealthManager : 聚合域快照
    NodeHealthSync --> NodeBusinessAdmission : 发布本地摘要
    NodeHealthRegistry --> NodeHealthSync : 独立查询与同步
    OperationOwner --> NodeBusinessAdmission : 获取节点 permit
    OperationOwner --> UbHealthManager : 获取路径 permit
```

#### 3.4.2 C++ 数据结构建议

以下为字段契约草案，序列化采用现有机制，不把 URMA 类型暴露给 Store。

```cpp
struct NodeIdentity {
    ClientId client_id;                 // 沿用现有 client_id 类型
    Incarnation transport_incarnation;
};
struct UbDomainHealth {
    DomainIdentity identity;           // 含 domain/context incarnation
    bool resource_online;
    bool valid;
    bool verification_pending;
    uint32_t expected_total;           // 本方案固定为 4，另持有预期端口身份集合
    uint32_t total;
    uint32_t bad;
    uint64_t health_epoch;
    uint64_t query_seq;
    uint64_t observation_generation;
    uint64_t verified_observation_generation;
};
struct NodeHealthSnapshot {
    NodeIdentity identity;
    uint64_t service_set_generation;
    uint64_t admission_epoch;
    BusinessState state;
    AdmissionReason reason;
    std::vector<UbDomainHealth> domains; // 有界，完整 ports 留在本地
};
struct HealthQueryTicket {
    NodeIdentity expected_identity;
    uint64_t service_set_generation;
    uint64_t query_generation;
    std::vector<DomainQueryFloor> required_query_seq;
};
```

`NodePermit`、`PathPermit` 和 Jetty `PostGate` 分别持有稳定引用与关闭代际。许可获取是本地准入线性化点；关闭之前取得许可的有限提交属于既有准入。不能把跨进程同步描述为瞬时原子关闭。

#### 3.4.3 所有权和并发模型

| 对象 | 唯一写入者/责任 |
| --- | --- |
| provider 上下文与查询结果 | context adapter；查询生命周期受 context incarnation 约束 |
| 域事实、共享 Monitor 与路径 gate | `UbHealthManager`；每四端口域一个实例，只发布不可变快照 |
| 本地节点 gate | `NodeBusinessAdmission`；集合聚合与准入转换 |
| 远端确认视图 | `NodeHealthRegistry`；拒绝旧身份、旧 epoch 和过期 ticket |
| 查询/同步任务 | `NodeHealthSync`；有界、可取消，退出等待回调排空 |
| WR/注册/实际缓冲/batch | Jetty resource 与原 operation owner；健康模块不得代为完成或释放 |

先发布快照再通知。provider、RPC、allocator、endpoint 操作及 join 均在状态锁外执行，回调不能反向同步 Query。隔离取消未 post 工作并 Seal；不 RESET、不清零 depth、不制造 CQE。

MR 引用不等于持有实际用户内存。裸指针调用者须保持缓冲至安全完成，Store operation 须接入实际 buffer hold 并处理 BatchBusy。timeout 不是可释放许可；无法证明 DMA 收敛时保留 pending/quarantine，遵守 Jetty 文档。

### 3.5 接口适配

#### 3.5.1 URMA Provider 适配

`UrmaPortStatusProvider` 封装正式头文件与结构体。CMake 检测声明/结构体，运行时检查 opcode 支持；不能为通过编译构造私有 ABI 数字，也不能对 enum 使用无效的宏存在性判断。端口结果必须完整校验，不支持时返回 UNSUPPORTED，不能返回伪造健康。

对齐 [yuanrong UrmaPortStatusProvider](../../../yuanrong-datasystem/src/datasystem/common/rdma/urma_port_status_provider.cpp) 的正式调用：本地 `urma_context_t` 上调用 `urma_user_ctl()`，`in.opcode = BONDP_USER_CTL_QUERY_PORT_STATUS`，输出使用 `bondp_query_port_status_out_t`，以返回的 `out.len` 校验 port_count 对应长度和数组容量。逐项只接受 BONDP_PORT_STATUS_GOOD/BAD，按三元组排序判重后重算计数。Supercache 额外核对预期四口完整身份集合；失败或缺口不返回 bad=0，不覆写最后确认状态。

#### 3.5.2 控制协议适配（对应参考 Proto 接口）

Mooncake 沿用现有 Probe JSON/action 与 Store 内部 RPC 序列化，不为模仿参考引入一套 DataSystem Proto。

| 接口 | 最小语义 |
| --- | --- |
| `ub_port_health.v1` / `ub_node_health.v1` | 请求身份、集合代际、query generation、下界；响应状态、真实身份、域聚合与新鲜性 |
| `ReportUbNodeHealthV1` | 合并上报本地节点摘要，仅作为远端核验触发 |
| `GetNodeHealthChangesV1` | 分页完整视图/有界版本增量，含版本缺口语义 |
| DeviceDesc 可选 `transport_payload` | 发布不透明身份与能力，通用 metadata 不解释 UB 判据 |
| HandShakeDesc `payload` | 核对 transport/domain 身份，防止旧 target/descriptor 复用 |

限制域数、消息大小、handler 数与 peer 表容量；未识别版本、未知 action 和旧 Probe success 不得当作健康。全部调用共享总 deadline，而非每个阶段各给一秒。

#### 3.5.3 C++ 接口

拟提供以下最小职责接口，名称在实现时统一：

| 组件 | 接口职责 |
| --- | --- |
| `UbHealthManager` | `Observe`、`RequestRefresh`、`ReadSnapshot`、`TryAcquirePathPermit`、`Stop` |
| `NodeBusinessAdmission` | `UpdateDomainSnapshot`、`ReadSnapshot`、`TryAcquireNodePermit`、`Close` |
| `NodeHealthRegistry` | `OnSummaryHint`、`EnsureVerified`、`IsEligible`、`ReadConfirmedSnapshot` |
| TE 通用准入 hook | 提交/新 grant 前取得通用节点许可，不依赖 Store 类和 URMA 类型 |

Client 数据入口先检查本地 gate，TransferSubmitter 在任意数据策略前检查双方节点。UB 最终 post 同时获取节点、路径和 Jetty 许可；队列唤醒、重试及后续窗口不能复用失效决策。

#### 3.5.4 错误码与报错信息

建议新增稳定错误 `NODE_DATA_ISOLATED` 与 `NODE_DATA_UNAVAILABLE`，保持既有错误数值不变。分别表达已确认隔离与当前无可服务路径/副本，和容量忙、对象不存在区分。

错误附带节点身份、原因和 admission epoch，TE/Store 显式映射。对端错误码只是 Query 线索，不能直接作为硬件事实。PutEnd/abort 保持既有事务语义，不因当前隔离丢弃已完成结果。

### 3.6 性能、可靠性与兼容性

#### 3.6.1 热路径开销

前台读取不可变快照并获取轻量 permit，不做同步 provider 查询或网络 RPC。异常与周期查询合并；完整 ports 不进入高频广播。故障摘要可触发负载方案更新，但本模块不计算成本权重、不引入拥塞信号。

#### 3.6.2 可靠性约束

远端隔离/恢复都需独立核验，失败保持既有关闭事实。Master 最终分配、Client 缓存路径和最终提交重复检查代际；不能仅在 PutStart 或首次选点过滤。

单边传输没有目标逐 WR 的业务 RPC，目标软件 gate 不能撤销已经发出的 DMA。传播与核验有延迟，需观测；提供的是消费者确认后的新准入关闭和既有责任安全收敛。

#### 3.6.3 滚动兼容

采用 `off / observe / enforce`，默认 off。observe 仅记录拟决策；enforce 要求参与数据业务的 Client、节点、Master 都具备必要能力，混合旧版本只允许 observe。公开 SDK 方法签名不变，新增内部 RPC/action 版本化。

#### 3.6.4 可观测性

记录域查询触发/合并/失败、身份不匹配、隔离原因与 epoch、业务拒绝、候选过滤、查询排队、核验/传播/恢复延迟，以及在途/quarantine 责任。日志区分 observation、confirmed fact 和 admission，不能用同一个 active 混合解释。

### 3.7 运行接入点

| 位置 | 修改职责 |
| --- | --- |
| [ub_transport.cpp](../../mooncake-transfer-engine/src/transport/kunpeng_transport/ub_transport.cpp) | manager 所有权、域注册、设备硬过滤与最终许可 |
| [ub_context.cpp](../../mooncake-transfer-engine/src/transport/kunpeng_transport/ub_context.cpp) | 监控核验、取消无条件恢复、退出顺序 |
| [urma_endpoint.cpp](../../mooncake-transfer-engine/src/transport/kunpeng_transport/urma/urma_endpoint.cpp) | provider 适配、CQE/AE 结构化观测、保留完成责任 |
| [transport.h](../../mooncake-transfer-engine/include/transport/transport.h)、[scheduler_core.cpp](../../mooncake-transfer-engine/src/scheduler_core.cpp) | 可选通用准入 hook、新 grant 守卫 |
| [transfer_metadata.cpp](../../mooncake-transfer-engine/src/transfer_metadata.cpp)、[transfer_metadata_plugin.cpp](../../mooncake-transfer-engine/src/transfer_metadata_plugin.cpp) | action 分发、能力/身份字段、总 deadline |
| [client_service.cpp](../../mooncake-store/src/client_service.cpp)、[transfer_task.cpp](../../mooncake-store/src/transfer_task.cpp) | 本地业务 gate、跨策略双方检查、缓存/重试守卫、实际 buffer hold |
| [master_service.cpp](../../mooncake-store/src/master_service.cpp)、[allocation_strategy.h](../../mooncake-store/include/allocation_strategy.h) | registry、全分配入口 eligibility、提交复查 |
| [replica_selection.h](../../mooncake-store/include/replica_selection.h) | 现有读策略前健康硬过滤，不引入新评分 |
| [rpc_service.cpp](../../mooncake-store/src/rpc_service.cpp)、[master_client.cpp](../../mooncake-store/src/master_client.cpp) | 版本化摘要/视图控制 RPC |
| [inter_master_rpc.cpp](../../mooncake-store/src/cvm/inter_master_rpc.cpp)、[master_snapshot_codec.cpp](../../mooncake-store/src/ha/snapshot/master_snapshot_codec.cpp) | 目的侧过滤、确认状态持久化与接管核验 |

新增健康组件按 TE 事实层与 Store 准入/传播层就近放置，头文件不循环依赖。该清单是实施接入范围，不表示本次修改运行时代码。

### 3.8 Use Case DryRun

| 场景 | 健康核验 | 当前操作与后续行为 |
| --- | --- | --- |
| UC-1 PUT/WRITE E4 | 只查本端 | 当前失败不重放；本地确认全域不可用才隔离 |
| UC-2 GET/READ E4 | 只查本端 | 内容无效、无成功 H2D；注册与旧 WR 安全检查后才可重读 |
| UC-3 PUT/WRITE E9 | 两端各查自身 | 谁确认全坏隔离谁；双方有健康端口仍保留本次失败 |
| UC-4 GET/READ E9 | 两端各查自身 | 读源与本地分别过滤；安全收敛后重读，不误判对象不存在 |
| UC-5 后台/恢复 | WRITE/READ 同样按 4/9 触发；逐域聚合 | bad=4→3 解除端口隔离；远端停止恢复轮询，本地继续至 0 |

以上为设计推演，不是已执行测试报告。

## 4. 对外接口

### 4.1 SDK 接口

保持现有 Put/Upsert/Get、批量、范围和异步签名。补充节点不可用错误契约；所有新数据入口在策略选择前执行本地准入。管理完整副本查询、健康查询和既有事务结算不因数据隔离关闭。

### 4.2 内部控制接口（对应参考 Proto 接口）

内部健康协议见 3.5.2；不新增面向用户的健康 Proto SDK。对外可见副本查询区分业务可服务列表与管理完整记录，保留既有元数据含义。旧版本不能解释新增能力时按兼容模式处理。

### 4.3 配置接口

| 配置契约 | 初始值/行为 |
| --- | --- |
| UB 隔离模式 | off；可切 observe/enforce，enforce 前检查参与者能力 |
| 域查询间隔 | 一秒，四端口域共享 Monitor、single-flight；本地至 bad=0 停止 |
| 远端恢复轮询 | 一秒目标周期，独立确认节点可服务后停止；不停止健康同步 |
| 远端查询并发 | 每进程最多四个，peer single-flight，有界公平队列 |
| RPC 总 deadline | 一秒，覆盖所有阶段 |
| 摘要兜底周期 | 一秒，变化立即合并上报 |

以上为待落地配置契约，最终名称沿现有配置体系确定。集合、消息和队列容量需显式上限；不新增动态评分或资源池调参配置。

## 5. 约束

- 节点键和服务集合代际贯穿注册、握手、Query、摘要与完成记录；旧身份不能修改新状态。
- 不以成员退出、segment 删除或 allocator 移除实现隔离；不自动迁移/重建副本。
- 一域故障只限制路径；全域有效不可用才建立节点故障隔离。UNKNOWN 不等于健康，也不等于确认 all-bad。
- 普通 Ping、业务成功、被动摘要、TTL 到期均不解除已确认隔离；fatal 需资源重建。
- Store gate 覆盖 TCP/memcpy/staging/文件路径；TE 仍保留最终节点/路径/Jetty 守卫。
- 查询与控制面有界运行，不能在 CQE/listener 中同步等待；控制服务和安全清理继续。
- E4 只触发本端，E9 触发双方各查自身；本次失败与后续健康准入分离，不能用健康查询修正操作结果。
- 四口集合不完整不能作为恢复证据；远端恢复轮询确认可服务即停，本地监控持续至全 GOOD。
- 退出依次关闭新准入、注销并排空查询、停止未 post 生产者并 Seal、保持 CQE/AE/flush 收敛、完成 H2D/缓冲责任，最后释放 target/MR/队列/context。健康 Stop 不替代 DMA 排空。

## 6. Example

以下为语义示例，不是最终线协议；每个服务域的预期集合均为四口，不按 context 重复统计。

### 6.1 存储节点健康状态示例

```json
{
  "node": {"client_id": "example-node", "transport_incarnation": "inc-2"},
  "service_set_generation": 1,
  "admission_epoch": 8,
  "state": "ISOLATED",
  "reason": "ALL_SERVICE_DOMAINS_UNAVAILABLE",
  "domains": [
    {"id": "A", "resource_online": true, "valid": true, "pending": false, "expected_total": 4, "total": 4, "bad": 4, "query_seq": 12},
    {"id": "B", "resource_online": true, "valid": true, "pending": false, "expected_total": 4, "total": 4, "bad": 4, "query_seq": 15}
  ]
}
```

仅当两域结果同身份、同集合代际且都覆盖各自最新异常时，该状态成立；省略的异常代际与新鲜性字段在真实协议中必须校验。

### 6.2 部分故障但不隔离

A 域 `total=4,bad=4`；B 域 `total=4,bad=1` 且资源在线，两者均为有效新鲜结果。节点为 DEGRADED，新业务可走 B 的合法路径；不能把 A 从配置集合删除。B 的 GOOD 端口若伴随未修复 fatal，则不能据此恢复节点。

若唯一域从 bad=4 降到 bad=3，资源在线且核验有效，则立即解除端口隔离并进入 DEGRADED；本地仍每秒查至 bad=0，远端独立确认可服务后停止恢复轮询。预期四口只返回三口全 GOOD 时查询无效，不能解除隔离。

### 6.3 Client 本地快速拒绝

Client 本地已确认 ISOLATED，再调用 Put/Get 时直接返回 `NODE_DATA_ISOLATED`；不继续请求新目标，也不转 TCP/memcpy。此前操作的 PutEnd/abort、健康 Query 和必要清理仍能执行。

### 6.4 读源快速拒绝

Client 缓存副本 A/B，独立确认 A 隔离后先过滤 A，再按原策略选择 B。若 A/B 均不可服务，返回 `NODE_DATA_UNAVAILABLE`，不返回 OBJECT_NOT_FOUND；管理查询仍展示两个副本及原因。

## 7. 可信软件

### 7.1 安全性 Security

复用既有可信控制通信边界；Query/摘要校验身份、版本、长度、域数和代际。健康响应不能修改 segment 所有权，避免错误身份影响分配。本文不新增独立认证体系。

### 7.2 韧性 Resilience

查询失败、同步分区和队列饱和保留已确认关闭；任务有界且可取消。HA 接管恢复关闭围栏，逐节点重验；CVM 目的 Master 自主过滤，不能依赖转发方单点判断。

### 7.3 隐私性 Privacy

健康消息仅携带节点标识、聚合状态、原因和代际，不携带对象内容或用户缓冲地址。完整端口明细留本地；诊断日志遵守现有访问和脱敏规则。

### 7.4 可靠性 Reliability

异常、事实、准入分别建模；唯一写入者与不可变快照避免多个 active 相互覆盖。结果不能跨实例复用，重试不能重放可能已接受的 WRITE。故障隔离不会提前释放实际缓冲。

### 7.5 可用性 Availability

一域故障不升级为全节点隔离，健康域和健康副本继续服务。控制面独立运行；独立核验恢复后重新开放新业务，失败节点不阻塞其他节点的核验与分配。

### 7.6 安全 Safety

逻辑 timeout 不等于硬件停止。保留稳定 completion token、注册和实际 buffer owner，无法证明安全则 quarantine。测试替身不能证明真实 provider 的 flush、target 或 DMA 屏障，必须进行 UB 实机验证。

## 8. 自验用例

以下均为实施后的验收计划；本次仅修改设计文档，不宣称已通过运行时测试。

### 8.1 分支覆盖矩阵

| 分支/竞争 | 必须验证的结果 |
| --- | --- |
| 单端口坏、单域全坏、多域全坏、空集合 | 依次退化/局部关闭/节点隔离/NOT_READY，不误判 |
| pending、非法结果、重复端口、旧身份、旧集合、四口缺一 | 不新增错误隔离，不解除旧关闭 |
| 同域多 context、独立多域 | 同域只建一个 Monitor，独立域不误合并，故障域不消失 |
| 四种 WRITE/READ 场景分别注入 E4/E9 | E4 仅本端，E9 双方各查自身，不按数据方向翻转 |
| E9 两端健康/一端全坏/两端全坏/对端 RPC 失败 | 分别不隔离/隔离确认侧/隔离双方/待核验，不直接归责远端 |
| bad=4→3→0、失败穿插 | 3 解除端口隔离，远端确认后停轮询，本地至 0 才停；失败保留状态 |
| FLUSH_ERR_DONE 等特殊完成 | 分类后处理，不当作普通 Slice user_ctx 解引用 |
| 失败 READ、staging、重读同 buffer | 不交付无效内容、不执行成功 H2D，旧 WR 收敛后才重用 |
| Query floor 重试、新异常、并发重复请求 | 下界稳定，合并刷新，新异常使旧 ticket 失效 |
| PORT_ACTIVE、普通 Ping、成功业务、摘要恢复 | 只触发核验，不直接恢复 |
| 端口恢复与 fatal 并存 | fatal 未重建前不能开放该域 |
| Put/Get、批量/范围/异步、缓存及全部策略 | 同一节点 gate 无绕过，重试/新窗口再检查 |
| PutStart 后关闭、permit 与 post 竞争 | 有限既有准入按真实结果结算，未提交阶段拒绝 |
| partial post、迟到 CQE、timeout、staging、BatchBusy | 不重复完成、不重放模糊 WRITE、不提前释放缓冲 |
| Master 普通/批量/定向/后台/跨 Master 分配 | eligibility 共用，实际提交再查代际 |
| 全部副本不可服务 | 返回节点不可用，不删除元数据、不负缓存 |
| 控制分区、摘要丢失、队列饱和 | 保留确认隔离，控制可用，任务容量受限 |
| HA 接管、CVM 视图变化、重启、旧版本混部 | 围栏正确恢复，旧实例不生效，混部不启 enforce |
| 退出与查询/CQE 回调并发 | 回调排空后再销毁 context，DMA 责任不遗失 |

UT 注入 provider、clock 与 Query；集成测试覆盖 Client/Master/TE 和协议兼容；系统测试使用真实 UB 注入端口与资源故障，并验证真实 DMA 生命周期。mock 只验证状态和协议分支。

### 8.2 五个 Use Case 验收

| UC | 验收重点 |
| --- | --- |
| UC-1 | WRITE E4 只触发本地核验，不重放可能接受的 WRITE；端口健康也不改成功 |
| UC-2 | READ E4 只触发本地核验，失败内容无效，无成功 H2D，本地隔离不换源绕过 |
| UC-3 | WRITE E9 双方独立核验、按事实隔离，Master 过滤与 PUT 撤销/结算完整 |
| UC-4 | READ E9 双方核验，注册/lease 与旧 WR 安全后重读，所有源不可用仍保留对象语义 |
| UC-5 | Copy/Move 成功提交前不删源；多域正确聚合，本地与远端恢复轮询停止条件不同 |

性能验收同时记录无故障热路径开销、异常查询合并率、恢复/传播延迟和控制队列饱和行为；不以不可证明的全局一秒原子恢复作为验收条件。
