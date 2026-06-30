# 配置改进计划

> 评估版本:0.2.2
> 评估范围:配置模型、文档一致性、环境变量覆盖、校验、多文件管理

## 一、背景与现状问题

ScorpioFS 的配置系统以 `scorpio.toml` 为入口,但配置模型弱类型、文档与实现脱节、存在静默副作用与空操作 API,导致"按文档配置会失败"的阻断性问题,且难以适配容器化部署所需的 env 覆盖能力。

具体问题如下:

### 1. 配置全为字符串,类型错误延迟到运行时

`config` 模块用 `HashMap<String, String>` 承载所有配置项。数值型(如 `load_dir_depth`、`fetch_file_thread`、`dicfuse_*_secs`)与布尔型(`dicfuse_readable`)在 toml 中均以字符串书写(如 `load_dir_depth = "3"`、`dicfuse_readable = "true"`),在访问器中靠运行时 `parse()` 转换。配置笔误(写成非数字、拼错布尔值)只有在运行到对应代码路径时才暴露,缺乏编译期/启动期 schema 校验。

### 2. 文档与实现不一致(阻断性)

> **进度说明(文档侧已对齐):** `docs/antares.md` 与 `docs/api.md` 已完成 Phase 0 文档对齐,本节多数原"阻断性"文档问题已修复。以下逐条标注 **【已修复】/【遗留】**,遗留项多为**代码侧**(而非文档侧)缺陷。

- **【已修复】** `docs/antares.md` 的"配置"章节曾声称 Antares 使用 `[antares]` 子表(`mount_root`/`upper_root`),与实现读取的扁平键 `antares_mount_root`/`antares_upper_root`/`antares_cl_root`/`antares_state_file` 不一致。现 `docs/antares.md` 已按实际实现书写扁平键,并明确标注"当前实现不使用 `[antares]` 子表"。**遗留(本计划方向):** [P0] 强类型配置仍以 `[antares]` 子表为目标格式,届时需同步更新 `docs/antares.md`,并保留扁平键只读兼容 ≥1 个 minor。
- **【部分修复】** `docs/api.md` 现已新增 `mega_url`→`base_url`、`mount_path`→`workspace` 的字段映射表,文档侧不再误导。**遗留(代码侧):** 代码中的 `ConfigRequest` 结构体本身仍声明为 `mega_url`/`mount_path`(`src/daemon/mod.rs`),与配置键 `base_url`/`workspace` 不一致。由于 `update_config_handler` 是空操作(见第 5 条),命名漂移目前被掩盖;若未来恢复该接口,需同时对齐 API 契约层与配置层,而非仅靠文档映射表。
- **【已修复】** `docs/api.md` "更新配置"章节末尾原有的格式错误 JSON 残留片段(`{ "status_code": 200, }`)已清理,代码块围栏配对正确。
- **【已修复】** `docs/api.md` 曾保留 `AddReq`/`GitStatus`/`GitStatusParams`/`CommitPayload`/`CommitResp`/`PushRequest`/`ResetReq` 等 git 数据结构(对应路由在 `src/daemon/mod.rs` 已被注释禁用,移至 `src/daemon/git.rs`)。现 `docs/api.md` 已删除这些"幽灵接口"数据结构,并改为明确说明 git 路由"存在于 `src/daemon/git.rs` 但默认未启用、此处不予文档化"。**遗留:** 若未来启用 `daemon::git::router()`,再补全 git 接口 URL/方法文档。
- **【已修复】** `README.md`(仓库根,不在 `docs/` 下)的示例 toml 已改为 `/tmp/scorpio-megadir/...` 通用路径,并说明主配置文件是 `scorpio.toml`、挂载键是 `workspace`、`config.toml` 是运行时状态文件。

### 3. `set_defaults` 静默改写用户配置文件

`init_config` 在首次运行(检测到 `workspace`/`store_path` 为空)时,会计算默认值并**直接写回用户的 `scorpio.toml`**(`fs::write(path, &toml)`)。这是一种意外的副作用:用户注释、格式、字段顺序会被破坏,且用户并不预期配置文件被改写。容器只读挂载配置文件时还会因此启动失败。

### 4. 双配置文件纠缠

系统同时维护两份配置:
- `scorpio.toml`:主配置(base_url、workspace、store_path 等)。
- `config.toml`:由 `ScorpioManager` 管理,记录 `works`(已挂载工作区状态),默认内容为 `works = []`。

`scorpio.toml` 中的 `config_file` 字段指向后者,但状态持久化路径**未统一**:

| 位置 | 写入路径 | 错误处理 |
|---|---|---|
| `src/main.rs` 读入 | `config::config_file()` | 正常 |
| `src/manager/mod.rs` `remove_workspace` | 硬编码 `"config.toml"` | 传播 `?` |
| `src/daemon/mod.rs` 临时挂载 | 硬编码 `"config.toml"` | `let _ =` 静默丢弃 |
| `src/manager/fetch.rs` ×3 | `config::config_file()` | `let _ =` 静默丢弃 |

一旦用户把 `config_file` 改为其他路径,`remove_workspace` 与临时挂载仍写默认文件,而 `fetch` 写自定义路径,造成**状态分裂**。此外 `set_defaults` 还会在 `config_file` 不存在时自动创建 `works=[]`,与"主配置只读"目标冲突。`config.toml`(运行时状态)已提交进仓库,`.gitignore` 未忽略。

### 5. `update_config_handler` 是空操作

`POST /api/config` 处理函数仅将请求体回显在响应中,**不真正持久化任何配置**,也不更新内存中的全局配置。**【文档侧已修复】** `docs/api.md` 已将该接口章节标题改为"Update Configuration (non-functional)",并明确说明"Does not persist or apply configuration changes"、"may be deprecated in a future release",不再误导用户。**遗留(代码/契约侧):** 是否真正实现热更新或在响应中显式返回 deprecation 标记,见下方 [P1]。

### 6. 无环境变量覆盖

容器化与 12-factor 部署需要通过环境变量(如 `SCORPIO_BASE_URL`、`SCORPIO_WORKSPACE`)覆盖配置文件值。当前配置系统完全不支持环境变量,所有配置必须来自 toml 文件,无法通过 `docker run -e` 或 k8s env 调整。

### 7. 校验薄弱

`validate` 函数仅检查必填字段是否存在且非空,不校验:
- `base_url`/`lfs_url` 是否为合法 URL 格式。
- `workspace`/`store_path`/`antares_*_root` 路径是否可写。
- `load_dir_depth`/`fetch_file_thread` 是否为正数(0 或负数会通过校验,运行时才出错)。
- `dicfuse_stat_mode`/`antares_dicfuse_stat_mode` 是否为合法枚举值(`fast`/`accurate`),非法值被静默回退为默认值。

### 8. README 配置示例与文档割裂

`README.md` 的"如何配置"章节已覆盖基础必需项并指向 `docs/antares.md` 的 Antares 配置,但仍不是完整配置 schema。`scorpio.toml` 实际包含 `dicfuse_*`、`antares_*` 等十余项调优参数;后续应通过 `scorpio.toml.example` 或生成文档提供完整可配置项。

### 9. 库消费者与双 `OnceLock` 约束

`config.rs` 刻意使用 `SCORPIO_CONFIG` 与 `DEFAULT_CONFIG` 两个 `OnceLock`:库集成方(orion 的 `warmup_dicfuse` 等)可能在 `init_config()` 之前调用访问器。若合并为单锁,早读会锁定默认值并导致后续 `init_config` 失败。配置重构**不得**破坏这一语义:访问器保持 `&'static str` 签名,`init_config` 成功后必须覆盖显式配置。

### 10. `DEFAULT_CONFIG` 懒初始化路径含 panic

`get_config()` 在 fallback 分支对目录创建失败和 `config_file` 创建失败使用 `panic!`,与主路径 `init_config` 返回 `Result` 不一致。测试或未调用 `init_config` 的代码路径可能意外崩溃。

## 二、改进方向

### [P0] 强类型配置模型

- 用 `serde` 派生的 `ScorpioConfig` 结构体替换 `HashMap<String, String>`,数值用 `usize`/`u64`,布尔用 `bool`,枚举(`DicfuseStatMode`)用 serde 枚举。
- 按职责拆分子表:`[server]`、`[dicfuse]`、`[antares]`,与文档对齐。
- 保留向后兼容:对旧扁平键做 deprecation 警告;迁移必须由显式命令触发,启动时不自动改写用户配置。
- 配置笔误在反序列化阶段即报错,带字段名与原因。

### [P0] 修复文档与实现对齐

> 多数文档侧子项已在 Phase 0 完成,以下标注 **【已完成】/【待办】**。

- **【已完成】** 统一 `docs/antares.md` 的配置章节与实际读取方式:现已书写扁平键并标注不使用 `[antares]` 子表。**【待办】** 引入 [P0] 强类型结构时,目标格式切换为 `[antares]` 子表,需同步更新该章节并保留扁平键兼容。
- **【已完成】** `docs/antares.md` 的 `--mount-root`/`--upper-root`/`--cl-root`/`--state-file` CLI 覆盖参数已核对为 `src/bin/antares.rs` 真实实现,文档与实现一致。
- **【已完成,代码侧待办】** `docs/api.md` 已新增 `mega_url`→`base_url`、`mount_path`→`workspace` 映射表。**代码侧仍需**:改 API 契约字段名或在 handler 内做显式映射(`ConfigRequest` 结构体仍用 `mega_url`/`mount_path`)。
- **【已完成】** `docs/api.md` 中格式错误的 JSON 残留片段与配对错误的代码块围栏已修复。
- **【已完成】** `docs/api.md` 中已禁用 git 路由的"幽灵接口"数据结构已删除,并改为说明其默认未启用、不予文档化。若未来启用 `daemon::git::router()` 再补全 URL/方法文档。
- **【已完成】** `docs/api.md` 已补充真实端点 `GET /api/fs/select/{request_id}` 的文档。
- **【已完成】** 清理 `README.md`(仓库根)示例中的开发者本机路径,改用 `/tmp/scorpio-megadir/...` 通用路径。
- **【部分完成】** README 已补基础配置说明并指向 `docs/antares.md`;完整配置项参考仍待 `scorpio.toml.example` 或生成文档落地。

### [P0] 环境变量覆盖

- 引入 `figment` 或 `config` crate,建立优先级:`CLI 参数 > 环境变量 > 配置文件 > 默认值`。
- 环境变量命名规范:`SCORPIO_` 前缀 + 字段名大写下划线,如 `SCORPIO_BASE_URL`、`SCORPIO_DICFUSE_LOAD_DIR_DEPTH`。
- 支持子表前缀,如 `SCORPIO_ANTARES_UPPER_ROOT`。
- 文档中列出所有可覆盖的环境变量。

### [P1] 移除静默改写用户配置

- `init_config` 首次运行时只创建运行所需目录,不回写 `scorpio.toml`。
- 若需提示用户缺少配置,打印建议命令(如 `scorpio config init`)。
- 必要时写入独立的 `scorpio.generated.toml` 并明确提示,不污染用户原始配置。

### [P1] 修复 `update_config_handler` 空操作

- 方案一:真正实现持久化(写回 toml + 热更新内存配置)。
- 方案二:标记为 deprecated,响应中明确说明该接口不生效,并从 `docs/api.md` 移除或标注。
- 推荐方案二,配置变更应走配置文件 + 重启,避免运行时热改引发状态不一致。

### [P1] 修复状态文件路径与错误处理

- **所有** `to_toml` 调用统一使用 `config::config_file()`(含 `manager/mod.rs`、`daemon/mod.rs`、`manager/fetch.rs`)。
- 禁止 `let _ = ...to_toml(...)`;写失败必须返回 `Err` 并记 `tracing::error!`,HTTP handler 返回 5xx,CLI 返回非零退出码。
- 状态写入采用临时文件 + `rename` 原子替换,避免半写损坏。
- 统一 `ScorpioManager` 构造:读/写路径一致(`main.rs` 已用 `config_file()` 读入,写路径缺陷被默认值相同掩盖)。

### [P2] 提供 `scorpio config` 子命令

- `scorpio config init`:交互式生成 `scorpio.toml` 模板。
- `scorpio config validate`:离线校验配置文件,报告缺失字段、类型错误、路径不可写等。
- `scorpio config show`:打印当前生效配置(合并文件 + env + 默认值后的最终值)。

### [P2] 增强校验

- URL 字段:校验 scheme + host。
- 路径字段:轻量校验父目录存在;可写性由 `config validate --deep` 或 `doctor` 执行,启动时最多警告。
- 数值字段:校验下界(`load_dir_depth >= 1`、`fetch_file_thread >= 1`)。
- 枚举字段:非法值报错而非静默回退。
- 端口字段(若引入):校验 1-65535。

## 三、验收标准

- 按 `docs/antares.md` 配置章节书写的 `scorpio.toml` 可直接启动,无字段名/类型/结构错误。
- `SCORPIO_BASE_URL=... SCORPIO_WORKSPACE=/tmp/ws scorpio` 可用环境变量覆盖文件配置。
- 启动时若 `load_dir_depth = "abc"`,立即报错并退出,错误信息指明字段名与原因,而非延迟到运行时。
- `scorpio config validate` 对一份故意写错的配置报告所有问题(缺失、类型、范围、URL 格式)。
- `scorpio.toml` 在启动后内容与字节顺序保持不变(未被静默改写)。
- `POST /api/config` 的行为与 `docs/api.md` 描述一致;在未设计热更新安全模型前明确标注 deprecated。
- `config_file` 指向非默认路径时,挂载/卸载状态正确写入该路径。


## 四、计划复核与补充约束

| 维度 | 复核结论 | 调整要求 |
|---|---|---|
| 合理性 | 强类型配置、env 覆盖和文档对齐是正确方向。 | P0 先修文档与加载契约,再迁移 schema。 |
| 可行性 | 直接替换 `HashMap<String, String>` 会牵动访问器、测试和调用方。 | 先增加兼容解析层,保留现有 `config::base_url()` 等访问器。 |
| 完整性 | 原计划缺少旧扁平键到新子表的迁移窗口。 | 旧扁平键和新子表至少共存一个 minor 版本,旧格式只警告不自动改写。 |
| 安全性 | 真正实现 `POST /api/config` 热更新会扩大配置注入和状态不一致风险。 | 推荐废弃该接口;若未来恢复,必须先有鉴权、字段白名单、审计和回滚。 |
| 功能正确性与接口兼容性 | `mega_url`/`base_url`、扁平键/子表都是用户可见契约。 | 文档先描述当前真实行为;新契约通过兼容层迁移,不能一次性删除旧字段。 |
| 数据流与控制流 | 原计划未定义配置合并顺序和状态写入边界。 | 固定优先级 **`CLI > 环境变量 > 配置文件 > 默认值`**,合并后统一校验;运行时状态只写 `config_file` 指向路径。 |
| 性能与效率 | 启动期深度检查目录可写或远端连通性可能变慢。 | 启动只做轻量校验;深度 I/O 和网络检查放到 `scorpio config validate --deep` 或 `doctor`。 |
| 可靠性与容错性 | 静默写主配置和硬编码状态路径会造成状态漂移。 | 主配置只读;状态文件使用临时文件 + rename 原子写。 |
| 兼容性与互操作性 | 新 schema 要同时适配裸机、容器、systemd 和测试环境。 | 默认路径使用 FHS/容器友好路径,测试通过临时目录覆盖。 |
| 可扩展性与可维护性 | 强类型 schema 有利于长期维护。 | 配置结构、示例、env 列表和校验测试应共源或同步生成。 |
| 合规性与标准符合性 | env 覆盖符合 12-factor。 | `config show` 和日志必须为未来可能出现的 token/secret/password 字段预留脱敏规则。 |

### 推荐配置加载数据流

```text
CLI 参数选择配置文件路径
  -> 解析 toml(旧扁平键 + 新子表兼容)
  -> 合并默认值(最低优先级)
  -> 合并 SCORPIO_* 环境变量
  -> 合并 CLI 覆盖项(最高优先级)
  -> 统一校验(轻量,启动路径)
  -> 构造只读 RuntimeConfig(进程内不可变)
  -> 服务/CLI/manager 只从 RuntimeConfig 读取
  -> mount 状态只写入 config_file 指向的 state store
```

**端口覆盖链(与部署/易用性联动):** `scorpio serve --http-addr` > `SCORPIO_HTTP_ADDR` > `[server].http_addr`(待引入) > 默认 `0.0.0.0:2725`。

关键约束:

- 主配置是输入,运行时状态是输出,两者不能混写。
- 配置加载失败必须在服务监听端口和执行挂载前返回错误。
- 运行时 state 写失败时,挂载/卸载 API 必须返回错误并记录结构化日志,不得假装成功。
- `scorpio config show` 默认脱敏未来可能出现的敏感字段;`validate --deep` 才做路径可写和远端连通性检查。

### 兼容与迁移要求

- 旧扁平键继续可读,新 `[server]`、`[dicfuse]`、`[antares]` 子表作为目标格式进入模板。
- 同一字段同时出现在旧键和新子表时,必须定义确定性优先级并输出警告。
- 不在启动时自动重写用户配置;需要迁移时提供显式 `scorpio config migrate --write`。
- `POST /api/config` 在未设计热更新安全模型前应标记为 deprecated,文档不得承诺其会生效。

### 安全补充

- HTTP API 当前**无鉴权**;即使废弃 `POST /api/config`,`GET /api/config` 仍暴露 `mega_url`/`mount_path`/`store_path`。`config show` 与 API 响应须脱敏绝对路径(可保留末段或哈希)。
- `init_config` 二次调用返回 `"Configuration already initialized"`;多配置文件热切换不在 v0.3 范围,文档应明确单进程单配置。
