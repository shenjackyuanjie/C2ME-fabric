# Minecraft 1.16.5 选择性回移计划

> 状态：设计与实施计划，尚未实施；难度和效果是源码评估，不是构建、兼容性或性能实测结果。
>
> 编写日期：2026-10-17。
>
> 目标仓库：`shenjackyuanjie/C2ME-fabric`。
>
> 目标分支：`backport/ver/1.16.5`，即早期的 1.16.5 backport 分支；不是另建一个名为 `backport/1165` 的分支。
>
> 文档创建前的分支基线：`d4ffccfc373cabbfd1453d8e833836cf438cca85`，上游提交 `Initial backport to 1.16.5`，2021-10-10。

## 1. 决策摘要

采用“保留 1.16.5 区块、票据和世界生成体系，选择性回移可分离的机制”的路线。优先目标是跑图和保存时的稳定性、资源控制以及维护体验，而不是追求当前 C2ME 全量功能对齐。

实施顺序：

1. 恢复可重复验证的旧版基线，补充配置、诊断与资源控制。
2. 改进 no-tick 加载调度、平滑区块发送、分摊自动保存。
3. 独立实施带有积压保护和明确完成语义的 I/O 流水线。
4. 根据 profiling 结果决定是否开展 POI 卸载、同步加载优先级、快速序列化等后续项目。

**I/O 后端重写与快速区块序列化器是两个项目。I/O 后端不以新 Chunk System 或快速序列化器为前提。** 第一版 I/O 优化保留旧版区块到 NBT 的语义，先改造 NBT 之后的编码、压缩、写出和完成通知。

本计划不要求移植整个 `c2me-rewrites-chunk-system`、FlowSched、DFC 或 OpenCL，也不把升级 Java 到当前上游要求作为默认前提。

## 2. 已确认的基线与参考范围

### 2.1 旧版已经存在的功能

基线的 `C2MEConfig`、I/O mixin 与序列化上下文已经包含以下能力，不能再次作为“新版新增功能”立项：[B01]、[B02]、[B03]。

| 旧版能力 | 本计划的处理方式 |
| --- | --- |
| 异步区块 I/O，`StorageIoWorker` 使用独立单线程执行器 | 保留可回退路径；新增的是流水线分工、积压保护和明确的完成/失败语义。 |
| `AsyncSerializationManager` 及异步序列化上下文 | 保留并审计其快照边界；存在上下文不代表其中所有可变对象都已经是不可变快照。 |
| no-tick view distance | 新项目是其加载控制器，不是从零增加 no-tick 功能；旧版配置默认关闭。 |
| 客户端视距上限修改 | 新项目是更合理的默认内存预算；旧版已经有视距上限配置。 |
| Gzip、Zlib、不压缩等区块压缩选择 | 不是新增；不顺便改变存档格式或默认压缩策略。 |
| 同步磁盘写入的系统属性开关 | 已存在 `com.ishland.c2me.chunkio.syncDiskWrites`；本次不将改变持久化策略包装成无代价优化。[B04] |

旧分支已有 Mixin 插件按配置控制 worldgen、chunk I/O 和 no-tick 等功能。新增开关应在现有体系中逐步接入，不需要为了少量功能先搬整个新版模块化结构。[B05]

### 2.2 Java 与构建约束

基线的 `fabric.mod.json` 要求 Java >=16，根构建脚本的源码/目标级别为 Java 16。[B06]、[B07]

实施默认以保留现有 Java 16 字节码目标为约束；构建 JDK 与实际运行 JDK 的组合在 M0 中验证。是否调整最低运行时必须独立记录，不能因复制上游代码而被动提高要求。不承诺 Java 8 兼容，也不把未经验证的较新 JDK 直接列为已支持环境。

回移时替换或隔离新 API，例如 `Thread.threadId()`、虚拟线程、`SequencedMap` 和外部函数/内存接口。这里的策略是保留功能意图，不是原样复制源码再批量改映射名。

### 2.3 固定上游快照

| 用途 | 提交 | 说明 |
| --- | --- | --- |
| 主要实现参考 | `4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15` | `dev/26.3.0` 在前期评估时读取的快照，2026-10-15。本文对“新版实现”的描述均以固定快照为准，不是对未来 HEAD 的承诺。 |
| 平滑发包历史实现 | `d0aa709244acea0d3a438aab7eaed2fc917ef5ce` | `preview/0.3.6-1.21.11` 的 0.3.6.0 快照；用作 EMA 平滑策略参考。[U06] |
| 调试显示补充参考 | `44ac42a0e75b1a1ac503e82fc952158af8ddecd6` | `dev/26.4.0` 中追加 holder 数量的实现，仅借鉴显示目标。[U03b] |

两个容易误读的地方：

- 历史版本的 EMA 平滑发包和主要参考快照中调整 `desiredBatchSize` 倍率的实现不是同一功能。计划回移前者的平滑目标，不把后者误称为同一套平滑算法。[U06]、[U07]
- 必须同时检查源码、Mixin 配置、插件条件和 Gradle 模块装配。主要参考快照中 DFC/OpenCL 的 include 被注释，Java 噪声优化的两个 mixin 也已从注册列表移除；目录存在不等于功能正在生效。[U14]、[U15]

## 3. 目标、非目标与行为边界

### 3.1 目标

- 改善单机/联机跑图时的尾延迟和区块可见延迟，减少短时间集中交付区块给客户端造成的负载尖峰。
- 降低集中保存导致的主线程尖峰，限制待写、编码中和压缩中数据的资源占用。
- 在保留 1.16.5 存档语义的前提下，分离 I/O 管理与可并行的 CPU 工作。
- 提供足以定位“卡在哪个区块、阶段或队列”的诊断，允许逐项关闭新功能进行对照。

以上均是需要验证的目标。提高预生成吞吐不自动等于改善日常体验，较低工作线程优先级也可能牺牲吞吐。

### 3.2 非目标

- 不全量移植新版 Chunk System、票据状态机或世界生成管线。
- 不把世界 tick 并行化，不修改红石、实体、刷怪、计划刻或随机数序列来换取性能。
- 不默认增加超大视距，不将 no-tick 区块错误转为应 tick 区块。
- 不默认启用低分配序列化器，不引入新的必需存档格式。
- 不把日志更少、异常被吞掉、区块出错后重生成视为稳定性提升。
- 本文不新增配置 GUI、运行时热重载或通用异步区块 API；新开关默认按重启生效设计。

### 3.3 共同约束

每一项功能都应有独立提交、独立开关、基线回退和对应测试。测试未完成的新路径保持显式 opt-in；成熟后改变默认值应另提提交，并附测量与兼容性结果。

“关闭功能”不应遗留线程、任务、票据、区域文件句柄或跨世界引用。若未来支持运行中切换，需要单独实现状态排空；不能因为存在配置项就声称支持热切换。

## 4. 候选功能与优先级

难度为 1—5 级：1 是少量局部适配，2 是局部模块改动，3 涉及调度/生命周期，4 涉及重要子系统及大量正确性测试，5 接近重新实现重要基础设施。难度包含适配与验证，不是工时或上线日期承诺。

优先级：P0 为开工/发布前置条件，P1 为首批建议，P2 为后续重点项目，P3 为 profiling 或实际需求触发的候选。

| ID | 项目 | 优先级 | 难度 | 预期效果与适用场景 | 回移方式/边界 |
| --- | --- | --- | --- | --- | --- |
| BP-00 | 可构建、可运行、可重复测试的基线 | P0 | 2—3 | 后续收益、回归和错误归因有可信对照 | 修复必要构建/测试问题，不混入性能功能；详见 M0。 |
| BP-01 | 配置迁移、兼容性规则、启动参数覆盖 | P1 | 1—2 | 整合包排错、单功能对照更方便 | 借鉴 `ConfigSystem` 的版本谓词、覆盖和原因记录；不是热重载。[U01] |
| BP-02 | 线程并行度预算与可选 OS 优先级 | P1 | 2 | 减少小堆内存或单机环境下的过度竞争 | 在旧执行器上实现；参考核心数/堆内存/环境预算和线程优先级预设，不依赖 FlowSched。[U02] |
| BP-03 | 任务跟踪、Watchdog 与队列/holder 指标 | P1 | 1—2 | 定位加载卡死、长期积压和未释放状态；不直接提高 FPS | 接旧版 holder、执行器和 Watchdog，默认低开销。[U03]、[U03b] |
| BP-04 | 待写 NBT 与在途工作积压保护 | P1 | 2—3 | 生成/修改速度超过写出速度时控制内存增长 | 原版后端和未来新后端分别接入；条目阈值之外增加估算字节预算，不丢弃最新脏数据。[U04] |
| BP-05 | 客户端默认视距上限的内存预算 | P1，可顺带做 | 1 | 降低小堆配置误选过大视距的风险 | 只影响默认允许值；重新校准 1.16.5 每区块估算，不当作真实内存硬限制。[U05] |
| BP-06 | no-tick 近处优先、并发上限、完成补位 | P1 | 3 | 改善近处地形可见延迟，减少不必要排队 | 复用螺旋遍历/在途 future 控制思路；用旧票据和 future 接口替换新 Chunk System 接线。[U08]、[B12] |
| BP-07 | 平滑区块发送 | P1 | 3 | 减少区块成批抵达客户端造成的处理尖峰 | 借鉴历史 EMA 实现，在 1.16.5 构建专门的初始区块发送队列；不能延迟全部 `sendPacket`。[U06] |
| BP-08 | tick 间空闲时间分摊自动保存 | P1 | 3 | 降低集中保存尖峰 | 旧保存入口上增加公平队列、时间预算和防饥饿策略，保留强制保存/关闭语义。[U09]、[U10] |
| BP-09 | 存储管理与 NBT 编码/压缩分离的 I/O 流水线 | P2 | 3—4 | 读取、解码、编码或压缩是瓶颈时提高并行度 | 保留旧区块到 NBT 路径；新增后端独立于快速序列化器，必须通过 M3 安全门槛。[U11] |
| BP-10 | 同步加载需求的优先级提升 | P3 | 3—4 | 减少主线程等待的任务被后台工作拖住 | 连同依赖任务一起评估；先证明旧调度器中存在该类等待，不全搬新调度系统。[U12] |
| BP-11 | POI 缓存卸载 | P3 | 3—4 | 长时间探索/大地图场景可能降低驻留内存 | 先用旧版 heap/生命周期证明确有问题；保证保存、按需重载和查询正确。[U13] |
| BP-12 | 错误线程访问随机数的诊断 | P3，维护用途 | 2—3 | 定位不安全的模组调用 | 适配旧随机数类型；诊断不等于修复，不照搬会改随机序列的 fallback。[U16] |
| BP-13 | 超过 127 区块的扩展视距协商 | P3，需求触发 | 2—3 | 为极大视距请求提供协议通路 | 适配旧 payload/线程模型、服务端能力协商和上限校验；常规 32/64 视距不需要先做。[U05] |
| BP-14 | 低分配区块序列化器 | P3，研究项目 | 4—5 | 序列化 CPU/对象分配确为瓶颈时可能受益 | 按 1.16.5 数据格式与快照边界重新适配，默认关闭，不能以普通 I/O 优化名义并入。[U17]、[U18] |

## 5. 实施里程碑

### M0：恢复基线，禁止把“已有测试目录”等同于“测试已可用”

关联：BP-00。所有运行时功能的前置条件。

基线 `settings.gradle` 已包含 `tests:test-mod`、`tests:vanilla-test` 和 `tests:c2me-test`，但 `tests:world-diff` 的 include 被注释。[B08]

进一步阅读脚本发现：[B09]、[B10]、[B11]。

- `tests/build.gradle` 为共享测试环境加入 LazyDFU 和 Starlight 依赖，因此名为 `vanilla-test` 的项目不能未经核对就作为“没有其他优化 mod 的纯原版”对照。
- 两个专用服务器运行任务都带有“测试运行目录不存在才执行”的 `onlyIf` 条件。重复调用 Gradle 成功不等于测试又执行了一次。
- C2ME 测试运行配置带有强制启用 worldgen、减少锁半径的系统属性和 incubator vector 参数，不能把这个配置的结果直接等同于用户默认配置。
- 常规 `runServer` 等任务在共享测试脚本中被禁用，应该核对专用任务，而不是凭常规任务名称编造验证命令。

实施任务：

- 核对旧 Loom、Gradle wrapper、映射、依赖仓库和包含的 `libs` 项目；只修复使基线可构建所必需的问题。
- 根构建使用 `git describe` 生成版本后缀，验证克隆历史/tag 是否足够；不要把缺少 Git 元数据的问题与 Java 编译错误混为一谈。[B07]
- 明确构建 JDK、字节码目标、客户端/专用服务端运行 JDK 和所需 JVM 参数。
- 使每次测试创建独立的临时运行目录，或者显式控制测试工作目录复用；不能隐式跳过，也不能通过清理命令删除用户真实存档。
- 优先复用现有测试 mod、预生成与世界差异工具；对未装配或尚不可用部分先恢复，不另造一个只能测平均吞吐的替代测试项目。
- 建立 A/B/C 对照：A 为 Minecraft 1.16.5 + 必要测试支架，B 为相同环境 + 旧 C2ME 基线，C 为 B + 本次单一功能。兼容性模组作为另一组明确固定的环境重复测试。
- 登记旧分支已存在的崩溃、结果差异、泄漏或保存缺陷。影响后续安全性的基线缺陷先独立修复，不能被新功能的性能结果掩盖。

候选验证入口，须在修复上述配置后确认实际执行；本次文档提交没有运行这些命令：

```sh
./gradlew tasks --all
./gradlew :assemble
./gradlew :tests:vanilla-test:runTestVanillaServer
./gradlew :tests:c2me-test:runTestC2MEServer
```

验收：生成可启动的开发/发行制品；两个测试运行任务可重复实际执行；退出和重新加载通过基础检查；保存配置、mod 列表、种子、路线、原始日志和性能基线。`world-diff` 的恢复状态单独记录，不能随一次 `build` 成功标记为已通过。

### M1：配置、诊断、资源预算和第一层积压保护

关联：BP-01、BP-02、BP-03、BP-04，可并入 BP-05。

旧版接入点：`common/config/C2MEConfig.java`、`mixin/C2MEMixinPlugin.java`、`common/GlobalExecutors.java`、`common/threading/chunkio/`，以及客户端配置与专用服务器诊断入口。[B01]、[B02]、[B05]

配置和资源控制要求：

- 保留旧键兼容映射，迁移前备份原配置，记录使用的是默认值、用户值、系统属性覆盖还是兼容性禁用值。
- 配置解析失败应报告具体问题；不要机械复制上游的重置策略，静默覆盖用户旧配置。
- 工作线程数在启动时根据可用处理器、可选物理核心信息、堆上限和客户端/服务端环境计算，始终保留用户显式覆盖；不是运行中动态伸缩线程池。
- OS 线程优先级通过可选适配层应用到真正执行任务的线程。未支持平台、权限失败或库加载失败时回退 Java 策略，并避免循环刷日志。
- 不把功耗限制或最低优先级设为所有机器统一默认值；比较吞吐和主线程/渲染线程尾延迟再决定。
- 诊断按区块坐标、维度、目标阶段、等待原因和经过时间记录；任务作用域必须在异常时也能退出，并清理已经结束的线程记录。

积压保护要求：

- 先给旧 `StorageIoWorker.results` 路径加可配置的软/硬阈值与写出调度；后续新后端拥有自己的计数和预算，不能假定这个 mixin 自动覆盖它。
- 使用“条目数 + 可估算字节 + 在途任务数”联合观察；NBT 估算字节不是整个 JVM 的精确硬上限。
- 软阈值用于更积极排空/限制新工作，硬阈值必须协调生产者背压。不得以丢掉脏 NBT 的方式控制内存，也不得让 I/O 线程等待只能由它自己推进的 future。
- 报告达到阈值时的队列大小、估算字节、最老任务年龄和写出速度。I/O 失败时进入可观察错误状态，而不是持续接受无限工作。

验收：覆盖低核心数、小堆、配置异常、无 native 能力、关闭世界和故障路径；验证错误配置不会破坏旧配置，队列在压力解除后可以排空，诊断开销可测且可禁用。

### M2-A：no-tick 加载控制器

关联：BP-06。依赖 M0/M1，不依赖新 I/O 后端或新 Chunk System。

旧接入点是 `PlayerNoTickDistanceMap` 及相关 no-tick mixin；参考新 `PlayerNoTickLoader` 的螺旋遍历、在途 future 集合与完成补位机制。[B12]、[U08]

实现要求：

- 把“每 tick 提交多少票据”和“系统中总共有多少正在加载”分开。近处优先提交，完成后补位，维持明确的在途上限。
- 多玩家/来源公平轮转；同区块的共享需求计数正确，不能一名玩家离开就取消另一名玩家仍需的区块。
- 玩家移动、传送、切换维度、退出、调整视距后，清理本控制器不再需要的票据；使用会话/代次校验阻止过期回调重新加票据。
- 只撤销本控制器拥有的需求。不能取消原版、其他玩家或其他 mod 持有的共享加载 future。
- 所有要求主线程执行的旧版票据/世界操作仍在主线程执行，不因控制器放到后台就越过线程边界。
- 保持旧版 no-tick 的 tick 隔离语义；区块已生成、可发送、应进行方块 tick、应进行实体 tick 必须明确区分。

验收：螺旋遍历无重复/遗漏，包含负坐标和边界半径；多玩家共享与连续跨维度无泄漏；成功、失败、取消后的补位计数正确；队列排空后近/远区块可见性符合需求，红石/实体不会在 no-tick 区域被意外激活。

### M2-B：平滑区块发送

关联：BP-07。加载节流与发送节流各有开关，可独立对照；二者控制的是不同阶段。

历史实现根据待发送数量维护指数移动平均，再用于目标批量。该实现依赖新版 `ChunkDataSender`，回移应重写 1.16.5 的接入层，而不是只改类名。[U06]

实现要求：

- 每玩家维护已具备发送条件的初始区块队列，采用有界预算和可调平滑策略，兼顾近处优先与长期公平。
- 延迟交付区块时，保证初始区块、光照、方块实体、实体跟踪和后续更新的顺序一致。可以选择接近实际发送时构造快照，或对快照之后的更新建立明确缓冲/排序机制。
- 不缓存一个已经过期的初始快照再覆盖较新的更新；不对所有网络包统一延迟。
- 客户端实际未收到区块前，不能错误推进“已发送”状态；玩家离开范围后不得又发出过期区块。
- 同时限制队列占用和发送工作量；有条件时评估字节/CPU 成本，而不是假设每个区块成本相同。
- 提供低延迟/关闭模式。平滑通常以部分加载延迟换取负载稳定，不能只报告更漂亮的帧时间而隐瞒可见延迟。

验收：传送、断连、切维度、光照变化和方块修改期间无缺块/旧块覆盖/实体顺序错误；比较客户端 P95/P99 帧时间、实际可见延迟、积压和网络发送量。

### M2-C：分摊自动保存

关联：BP-08。参考上游在 `runOneTask` 无普通任务时尝试保存、限制搜索数和并发保存数的做法；旧版没有相同的所有字段与保存入口，需按实际生命周期实现。[U09]、[U10]

实现要求：

- 主线程维护脏区块的公平保存候选队列，在 tick 间有预算时执行必要的快照/提交工作。单次搜索、主线程耗时与在途保存数分别有上限。
- 快照之后的再次修改必须重新标脏，完成较旧的保存不能清掉更新一代的脏状态。
- 防止“服务器持续满载，所以永远不保存”：设置保存年龄监测、最低推进策略和明确报警。I/O 不可用时不能承诺仍能满足持久化时限。
- 保留 `save-off`、`save-all`/flush、卸载、关闭世界的明确语义。周期性玩家和世界元数据保存也要审计，不能只处理区块后误删原服务器的其他保存工作。
- 不在 I/O 队列已超预算时无限提交新快照；不为腾空队列而降低数据安全性。

验收：持续负载、频繁修改、长时间驻留区块、保存禁用/恢复和立即退出均通过；保存后重新加载的方块、实体、方块实体、计划刻和元数据没有新增差异；记录保存尾延迟与最老脏数据年龄。

### M3：独立 I/O 流水线

关联：BP-09，依赖 M0/M1 和存储正确性测试；不以 BP-14 为前置条件。M2 可以提供更多真实压力场景，但不是接口依赖。

建议职责划分：

```text
旧版区块/世界保存入口
    -> 旧版语义下的安全快照或已完成的 NBT
    -> 有界 CPU 工作：NBT 编码 / 压缩
    -> 单所有者存储调度：版本检查 / RegionFile 写入
    -> 明确的写入完成、flush 完成与错误传播
```

读取反向拆分：由受控存储路径取得对应输入，后台执行适合并行的解压/NBT 处理，再按旧版线程要求安装到世界。流、文件与缓冲区的所有权及关闭时机必须清楚；不能在关闭 RegionFile 后仍让后台任务使用无效资源。

`C2MEStorageThread` 是参考，不是可无审计复制的正确性模板。它包含按位置合并待写项、后台编码/压缩及写出前匹配当前 future 的逻辑，但写入错误路径中也保留了日志处理和 `TODO error retry`。[U11]

必须满足的存储契约：

1. **输入所有权**：不能默认 `NbtCompound` 是不可变/线程安全对象。明确谁可以继续修改、何时复制、什么时候可以转移给后台。
2. **同位置顺序**：每个区块的写入/删除具有单调代次。旧任务晚完成不得覆盖较新的已受理数据；删除与后续写入同样受此规则约束。
3. **读后写一致性**：读请求应看到按队列顺序已经受理的相应数据，而不是因为尚未落盘就返回旧磁盘内容。精确语义在适配旧接口时记录并测试。
4. **完成层级**：内部区分受理完成、实际写入完成、显式持久化/flush 完成。旧接口应保持已验证的语义或使用明确适配器，不能把受理 future 当作已经安全落盘。
5. **写入失败**：不在实际写入成功前不可恢复地移除最新缓存/脏状态。不吞异常后向上层报告成功；有限重试或保留待处理数据必须有预算、可观察状态和退出路径。
6. **flush 屏障**：覆盖调用时已受理的工作（或取代它们的更新版本），等待相应编码、压缩、写入结束，按所选持久化策略完成同步；失败必须可见。
7. **关闭流程**：进入关闭状态后停止接受不允许的新工作，排空既有工作或明确失败，最后关闭存储资源和工作线程。不得靠 daemon 线程让进程退出来假装保存完成。
8. **全路径背压**：预算覆盖排队 NBT、编码中对象、未压缩/压缩缓冲区及写入队列；不能只限制其中一个 Map。不要在同一个有界执行器里阻塞等待依赖任务，造成全部 worker 被等待者占满。
9. **故障不得变重生成**：读盘错误与区块不存在必须区分；损坏或失败不能静默返回“空区块”并覆盖原数据。
10. **回退可读**：初期保持旧版支持的存档结构和压缩方式，关闭新后端后仍能读取并继续保存同一存档。

建议先定义与 Minecraft 弱耦合的存储调度/版本/预算核心，再做旧版 `StorageIoWorker` 适配。本文描述的是拟议职责，不表示仓库已经存在这些新接口。若需要内部 `accepted`/`written`/`flush` 信号，应在实现提交中明确定义，而非直接改变所有旧调用点的期望。

验收：通过第 7 节的存储故障注入与往返测试；分别测已生成地形加载、连续保存和关闭耗时；确认收益来自流水线，而不是关闭持久化或省略保存。

### M4：证据驱动的后续项目

- **BP-10**：先采集主线程同步等待的任务链，再设计依赖优先级提升和防饥饿；若瓶颈在磁盘或主线程安装阶段，不因看到优先级类就移植。[U12]
- **BP-11**：先证明 1.16.5 POI 驻留导致可观的长期增长；审计保存、预加载引用和重新加载，再接旧版卸载入口。不能只照搬移除 section 缓存的循环。[U13]
- **BP-12**：默认定位为开发诊断，避免宽松模式改变随机来源后宣称世界生成完全不变。[U16]
- **BP-13**：仅在实际需要超大视距时开展，限制服务端接受的请求并验证未知/恶意 payload；不要求旧客户端必须安装新协议支持。[U05]
- **BP-14**：先证明旧版区块序列化分配/CPU 已成为主要热点。保留 1.16.5 的实体、方块实体、计划刻、结构、光照、palette 和 DataVersion 语义；不能复制新版 `SerializedChunk` 路径就当成旧格式兼容。上游配置默认关闭且声明 Architectury 不兼容，这是该快照的上游规则，不是已验证的 1.16.5 兼容性结论。[U17]、[U18]

## 6. 暂不立项的方向

| 方向 | 决策 | 理由 |
| --- | --- | --- |
| 全量新版 Chunk System / FlowSched | 不属于本计划 | 新状态体系包含读盘、延迟、生成、可访问/发送、方块 tick 和实体 tick 等阶段；移植范围远超 QoL 与局部 I/O 优化。[U19] |
| 当前整套 native 噪声加速 | 不纳入首批 | 当前接线依赖新采样区域/缓冲区和内存接口，必须重接旧生成代码并验证数值/平台；可复用数学内核不等于模块可直接回移。[U20] |
| DFC、含水层等强绑定新版生成结构的优化 | 不按模块整体回移 | 先验证旧版是否存在对应结构和热点；不为承载优化反向移植新生成体系。主要快照中 DFC 未装配。[U14] |
| OpenCL 模块 | 排除直接代码回移 | 主要快照中未装配，且该目录在仓库许可证中单独列为 All Rights Reserved，不属于默认 MIT 授权部分。[U14]、[U21] |
| 仅凭 `opts` / `fixes` 目录名称复制的优化和修复 | 不接受 | 要有旧版对应问题、实际注入路径和测试。上游在某新版停用某优化，也不能反过来证明它在 1.16.5 一定无益。[U15] |

回移 MIT 部分时保留相应版权和许可文本，并在提交或文档中记录原提交。第三方代码和依赖分别检查来源，不把整个上游仓库笼统当成同一许可证。[U21]

## 7. 验证与发布门槛

### 7.1 正确性测试矩阵

优先复用旧测试支架；队列、版本和预算核心可加入不启动 Minecraft 的确定性单元测试，用可控 executor、时钟、输入流及假存储注入失败。

| 测试组 | 必须覆盖的场景 | 通过标准 |
| --- | --- | --- |
| 基线与差分 | A/B/C 对照、固定种子和操作顺序、同场景多次运行 | 区分旧 C2ME 已有问题与本次回归；不得以宽泛忽略规则隐藏新差异。 |
| I/O 顺序 | 同区块 A/B 写入，A 编码晚于 B；写后立即读；写/删除/再写 | 最终内容与受理顺序、代次和读语义一致，旧数据不覆盖新数据。 |
| I/O 故障 | 编码异常、压缩异常、读/写 IOException、模拟空间不足、关闭时失败 | future 以明确错误结束，不静默成功、不转成不存在、不无限重试；数据保留/重试策略可验证。 |
| flush/退出 | 大量在途写入时 flush、立即退出、重开世界、多次建关世界 | 屏障覆盖正确，无遗漏、资源泄漏或依赖死锁；失败在关闭结果中可见。 |
| 预算/背压 | 慢存储、大区块、小堆、持续生成、反复覆盖同一位置 | 计数无泄漏/下溢，压力解除后可恢复；最新脏数据不丢弃，不把无界队列换到别处。 |
| no-tick 生命周期 | 多玩家共享区块、快速移动、传送、换维度、退出 | 本控制器票据正确回收，旧回调不恢复需求，不破坏其他持有者，不改变 tick 范围。 |
| 发送一致性 | 延迟发包期间改方块/光照、实体跟踪、断连和切维度 | 初始数据与更新顺序正确，客户端不收到过期区块或永久缺失必要更新。 |
| 自动保存 | 长期满载、长驻脏区块、反复修改、save-off/on、强制保存 | 没有新增保存饥饿或脏标记误清，区块/玩家/世界元数据均按策略保留。 |
| 存档往返 | 方块、实体、方块实体、计划刻、结构、POI、光照及模组扩展数据 | 使用复制存档重复保存/重载，无新增语义差异；关闭功能仍可读写。 |
| 线程和模组 | 并行生成开/关、no-tick 开/关、目标 Lithium/Starlight/Fabric API 组合 | 固定实际版本单独记录；不能因模组名字出现在旧脚本里就标记兼容。 |

世界/NBT 比较采用严格且有文档的归一化规则，例如不依赖 Compound 键顺序；哪些字段可忽略必须逐项说明。不能忽略实体、计划刻或未知模组字段来“让测试通过”。

发生存档损坏、丢失实体/计划刻、读失败被当作不存在、写失败静默成功、主线程依赖死锁，或者新增无法解释的生成/游戏行为变化时，停止发布该功能。先定位是基线缺陷还是新增问题，再修复或回退。

### 7.2 性能测量

场景至少包括：新地形生成、已生成地形快速移动、持续修改后的自动保存、多人分散加载、退出与重新加载。使用相同来源的世界副本，防止前一组测试改变后一组的输入。

记录 CPU/OS/JDK、堆与 JVM 参数、mod 版本和配置、视距、种子、路线、缓存冷热状态。每个场景至少重复三次并保留原始结果；开发阶段先明确测量噪声，不预设一个没有依据的收益百分比。

| 指标 | 用途 |
| --- | --- |
| 客户端帧时间 P95/P99、服务端 MSPT P95/P99 | 判断日常卡顿改善，而不是只看平均 TPS/FPS。 |
| 请求到区块实际可见的延迟、近处区块完成顺序 | 防止平滑或降优先级仅仅把工作无限推迟。 |
| 区块吞吐、读写/编码/压缩分阶段耗时 | 识别瓶颈与吞吐代价。 |
| 对象分配率、GC 暂停、峰值/稳态堆占用 | 检查优化是否以不可接受的内存换速度。 |
| 待写条目/估算字节、在途数、最老任务/脏数据年龄 | 验证背压、饥饿和恢复能力。 |
| flush 与关闭耗时、文件句柄和残留线程 | 防止运行期间更快但退出卡死或资源泄漏。 |

测得无益或显著恶化另一项关键指标时，保留 opt-in、调整策略或放弃；不能仅以预生成吞吐单项变好判定成功。

## 8. 提交与状态管理

功能提交按 BP 编号组织，建议提交信息采用 `feat(backport): ...`、`fix(backport): ...`、`test(backport): ...`；基线修复、配置默认值调整和新优化尽量分开，避免后续无法二分。

每项实现提交/评审记录至少包含：上游固定提交与文件、旧版接入点、复用/重写边界、开关与回退行为、测试场景与实际结果、兼容性限制。为拟议功能创建的配置键要注明是 fork 新增，不能伪装成当前分支已支持的上游键。

当前里程碑状态：

- [ ] M0：构建与可重复测试基线通过。
- [ ] M1：配置、诊断、资源预算和旧 I/O 路径积压保护通过。
- [ ] M2-A：no-tick 控制器通过。
- [ ] M2-B：平滑发送与客户端一致性通过。
- [ ] M2-C：分摊保存和保存防饥饿通过。
- [ ] M3：新 I/O 流水线及故障注入通过。
- [ ] M4：为选中的后续项目提供 profiling 或实际需求证据，再单独立项。

**本次仅提交计划文档，不修改运行时代码、配置、构建脚本或测试；上述复选框不得据此勾选。**

## 9. 源码参考索引

以下为固定提交的源码链接。事实描述以这些快照为依据；优先级、改造方案和验收标准是本计划的工程决策。实现时仍须核对被回移提交之后的修复历史，避免只复制最初引入功能的补丁。

### 基线

[B01]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/java/com/ishland/c2me/common/config/C2MEConfig.java
[B02]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/java/com/ishland/c2me/mixin/threading/chunkio/MixinStorageIoWorker.java
[B03]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/java/com/ishland/c2me/common/threading/chunkio/AsyncSerializationManager.java
[B04]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/java/com/ishland/c2me/mixin/optimization/chunkio/hide_sync_disk_writes_behind_flag/MixinRegionBasedStorage.java
[B05]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/java/com/ishland/c2me/mixin/C2MEMixinPlugin.java
[B06]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/resources/fabric.mod.json
[B07]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/build.gradle
[B08]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/settings.gradle
[B09]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/tests/build.gradle
[B10]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/tests/c2me-test/build.gradle
[B11]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/tests/vanilla-test/build.gradle
[B12]: https://github.com/shenjackyuanjie/C2ME-fabric/blob/d4ffccfc373cabbfd1453d8e833836cf438cca85/src/main/java/com/ishland/c2me/common/notickvd/PlayerNoTickDistanceMap.java

| 引用 | 阅读入口 |
| --- | --- |
| [B01] | 旧配置与功能默认值。 |
| [B02]、[B03] | 旧异步 I/O 与序列化上下文。 |
| [B04]、[B05] | 同步写入策略与 Mixin 开关。 |
| [B06]、[B07] | 运行时要求、编译与版本生成。 |
| [B08]—[B11] | 测试装配、共享依赖、专用运行任务与跳过条件。 |
| [B12] | 旧 no-tick 票据控制器。 |

### 上游参考

[U01]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-base/src/main/java/com/ishland/c2me/base/common/config/ConfigSystem.java
[U02]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-base/src/main/java/com/ishland/c2me/base/ModuleEntryPoint.java
[U03]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-base/src/main/java/com/ishland/c2me/base/mixin/report/MixinDedicatedServerWatchdog.java
[U03b]: https://github.com/RelativityMC/C2ME-fabric/blob/44ac42a0e75b1a1ac503e82fc952158af8ddecd6/c2me-rewrites-chunk-system/src/main/java/com/ishland/c2me/rewrites/chunksystem/mixin/client/MixinChunkSourceStatsDebugHudEntry.java
[U04]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-opts-chunkio/src/main/java/com/ishland/c2me/opts/chunkio/mixin/limit_nbt_cache/MixinStorageIoWorker.java
[U05]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-client-uncapvd/src/main/java/com/ishland/c2me/client/uncapvd/common/Config.java
[U06]: https://github.com/RelativityMC/C2ME-fabric/blob/d0aa709244acea0d3a438aab7eaed2fc917ef5ce/c2me-notickvd/src/main/java/com/ishland/c2me/notickvd/mixin/smooth_sending_rate/MixinChunkDataSender.java
[U07]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-notickvd/src/main/java/com/ishland/c2me/notickvd/mixin/MixinChunkDataSender.java
[U08]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-notickvd/src/main/java/com/ishland/c2me/notickvd/common/PlayerNoTickLoader.java
[U09]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-opts-scheduling/src/main/java/com/ishland/c2me/opts/scheduling/mixin/idle_tasks/autosave/enhanced_autosave/MixinMinecraftServer.java
[U10]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-opts-scheduling/src/main/java/com/ishland/c2me/opts/scheduling/mixin/idle_tasks/autosave/enhanced_autosave/MixinThreadedAnvilChunkStorage.java
[U11]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-rewrites-chunkio/src/main/java/com/ishland/c2me/rewrites/chunkio/common/C2MEStorageThread.java
[U12]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-base/src/main/java/com/ishland/c2me/base/common/scheduler/SchedulingManager.java
[U13]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-rewrites-chunk-system/src/main/java/com/ishland/c2me/rewrites/chunksystem/mixin/MixinSerializingRegionBasedStorage.java
[U14]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/settings.gradle
[U15]: https://github.com/RelativityMC/C2ME-fabric/commit/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15
[U16]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-fixes-worldgen-threading-issues/src/main/java/com/ishland/c2me/fixes/worldgen/threading_issues/common/CheckedThreadLocalRandom.java
[U17]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-rewrites-chunk-serializer/src/main/java/com/ishland/c2me/rewrites/chunk_serializer/ModuleEntryPoint.java
[U18]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-rewrites-chunk-serializer/src/main/java/com/ishland/c2me/rewrites/chunk_serializer/mixin/MixinThreadedAnvilChunkStorage.java
[U19]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-rewrites-chunk-system/src/main/java/com/ishland/c2me/rewrites/chunksystem/common/NewChunkStatus.java
[U20]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/c2me-opts-natives-math/src/main/java/com/ishland/c2me/opts/natives_math/mixin/MixinPerlinNoiseSampler.java
[U21]: https://github.com/RelativityMC/C2ME-fabric/blob/4d97f4d41cb90efa8937d8f2fe137e10f2fd8b15/LICENSE.md

| 引用 | 阅读入口 |
| --- | --- |
| [U01]—[U05] | 配置、资源预算、诊断、待写阈值和视距默认值。 |
| [U06]、[U07] | 历史平滑发包与较新倍率实现的区别。 |
| [U08] | 新 no-tick 控制策略及对新 Chunk System 的直接依赖。 |
| [U09]、[U10] | 分摊自动保存的服务器和区块管理接入点。 |
| [U11] | I/O 流水线、写入合并、过期检查及错误路径。 |
| [U12]、[U13] | 同步加载优先级和 POI 卸载。 |
| [U14]、[U15] | 实际模块装配与停用的 Java 噪声 mixin。 |
| [U16] | 随机数线程诊断及 fallback 的行为差异。 |
| [U17]、[U18] | 实验性快速序列化器的默认状态及保存路径。 |
| [U19]、[U20] | 不计划整体回移的新区块系统和 native 采样接线。 |
| [U21] | MIT 与 OpenCL 保留权利目录的授权边界。 |
