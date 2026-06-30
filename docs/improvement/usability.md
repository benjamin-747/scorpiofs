# 易用性改进计划

## 背景与评估范围

- 适用版本:0.2.2
- 评估对象:CLI 入口体验、HTTP API 一致性、文档可信度、日志可观测性、运行时健壮性、仓库卫生
- 评估方法:基于源码静态审查(主二进制、antares 二进制、daemon、lib 文档注释、script 目录、README)

---

## 一、现状问题

### 1.1 README Quick Start 曾简陋且有误(文档侧已修复)
- README 的 "How to Use" 已从旧的三步说明扩展为主配置、主二进制启动、Antares API、legacy API 和基础配置说明。
- 原先让用户改 `config.toml` 的 `mount_path` 是事实错误:主配置文件是 `scorpio.toml` 而非 `config.toml`(后者是运行时状态文件),配置键名是 `workspace` 而非 `mount_path`。README 现已改为 `scorpio.toml` / `workspace`。
- 原示例配置使用开发者本机绝对路径;README、`docs/perf_test.md` 与实际 `scorpio.toml` 现均已改为 `/tmp/scorpio-megadir/...` 通用路径。
- README 的 usage 块已补充实际存在的 `--http-addr`。

### 1.2 主二进制与 antares 二进制 CLI 割裂
- 主 `scorpio` 二进制仅暴露 `-c/--config-path` 与 `--http-addr` 两个参数(`src/main.rs`),挂载/卸载/列表全靠 `curl` 调 `/api/fs/mount`。
- 注意 `--http-addr`(默认 `0.0.0.0:2725`)在代码中确实存在,且 `README.md` 的 usage 示例已补正;后续 CLI 子命令重构仍需保持 README/help 同步。
- `antares` 二进制反而提供 `mount / umount / list / serve / http-mount` 子命令,体验完整。
- 两个二进制入口职责重叠、命令风格不一致,用户不知道该用哪个。

### 1.3 HTTP API 无鉴权

主服务所有路由(`/api/fs/*`、`/api/config`、`/antares/*`)均无认证或授权层,默认监听 `0.0.0.0:2725`。在共享网络中等同于开放挂载控制面。改进计划须在文档中标注风险,并在 P3+ 评估 localhost 默认绑定或 token 中间件;短期内 health 响应不得泄露完整 workspace/store 路径。

### 1.4 两套 HTTP API 并存

> **进度说明(文档侧已对齐):** 本节涉及的文档缺陷已在 Phase 0 修复,以下标注 **【已修复】**;**代码侧**的"两套 API 重叠"本身仍是待收敛项。

- **代码侧仍存在**:主服务端口暴露老接口 `/api/fs/mount`、`/api/fs/mpoint`、`/api/fs/unmount`,以及 `GET /api/fs/select/{request_id}`(`src/daemon/mod.rs`);**【已修复】** 后者现已在 `docs/api.md` §2 完整文档化(原"完全未提及"已不成立)。
- 同一进程又 nest 了 Antares 路由:`/antares/mounts`、`/antares/health`、`/antares/mounts/{id}/ready`、`/antares/mounts/{id}/cl` 等,功能与老接口重叠。**【已修复】路径前缀差异已文档化**:`docs/antares.md` 已新增"访问方式与 URL 前缀"章节,明确独立 `antares serve` 用根路径(`/mounts`、`/health`),主进程 nest 后为 `/antares/*`,并在每个端点同时给出两种写法。统一时以"主进程 `/antares/*` 为推荐"。
- `docs/api.md` 描述的 `GET/POST /api/config` 中,`POST` 是空操作(见配置文档),`GET` 返回的字段名(`mega_url`/`mount_path`)与配置键不一致。**【已修复(文档侧)】** `docs/api.md` 已补充 `mega_url`→`base_url`、`mount_path`→`workspace` 映射表,并标注 `POST` 为 non-functional;字段名不一致的**代码侧**修复仍待办。
- **【已修复】** 文档原先分散且失真(`docs/api.md` 含已禁用 git 接口残留结构、`docs/antares.md` 漏写 `/mounts/{id}/ready`)。现 `docs/api.md` 已删除 git 幽灵结构,`docs/antares.md` 已补 §4.2 `GET /mounts/{id}/ready`,且 `docs/api.md` 顶部已注明"推荐 Antares API"。**【待办】** 老 `/api/fs/*` 的 deprecation 标注仍需落地。

### 1.5 `POST /api/config` 响应语义不一致

`GET /api/config` 返回 `status: "Success"`,而 `POST /api/config` 返回 `status: "success"`(小写)且仅回显请求体、不持久化。**【文档侧已修复】** `docs/api.md` §6 已标注该接口 non-functional 并在 Note 中明确说明大小写差异为"current implementation behavior"。**遗留(代码侧):** 统一 `status` 大小写或标记 deprecated 仍需在代码落地;客户端按文档已不会误判 POST 生效。

### 1.6 lib.rs 文档示例无法编译
- `lib.rs` 顶层 doc comment 多处使用 `scorpio::util::config`、`scorpio::antares` 等路径,但 crate 名实为 `scorpiofs`(`Cargo.toml` 声明)。
- 所有示例均标注 `rust,ignore` 跳过 doctest,库用户文档可信度受损。

### 1.7 日志不一致
- `main.rs` 混用 `println!` / `eprintln!` / `print!`,且 `print!("server running...")` 无换行。
- `antares.rs` 使用 `tracing` + `tracing_subscriber`。
- 无统一日志级别配置项,无文件输出/轮转支持,`RUST_LOG` 行为在不同入口下不一致。

### 1.8 启动 banner 噪声
- 主二进制每次启动打印 ASCII art banner(`src/main.rs` 中 `println!` 输出 5 行 logo),在 systemd/journald 与容器日志中是纯噪声,干扰结构化日志采集。
- 该 `println!` 位于 `Args::parse()` **之前**,因此 `scorpio --help` / `--version` 也会先打印 banner,污染机器可解析的输出;清理时应将 banner 移到参数解析之后,并受 TTY/`--quiet` 控制。

### 1.9 主端口缺少根级健康检查
- 严格说,主端口(2725)并非完全没有健康检查:Antares 路由被 nest 后,`GET /antares/health` 在主端口可用,且已存在按挂载粒度的 `GET /antares/mounts/{id}/ready`。
- 但主端口没有约定俗成的**根级 `GET /health`**,探针需要知道 `/antares/health` 这一非标准路径;老 `/api/fs/*` 这一侧没有任何 liveness 端点。
- 因此问题应表述为"缺少统一、稳定、根级的 liveness/readiness 入口",而非"无健康检查"。设计 `/health` 时应复用既有 Antares 健康逻辑,避免再造一套语义。

### 1.10 生产路径大量 unwrap/panic
- HTTP daemon 启动(`src/daemon/mod.rs` 中 `TcpListener::bind(bind_addr).await.unwrap()`)与卸载流程存在 `unwrap()`;`src/main.rs` 中 `ScorpioManager::from_toml(...).unwrap()` 也会在状态文件损坏时直接 panic。
- 卸载路径 `find_path(req.inode.unwrap()).await.unwrap()`(`src/daemon/mod.rs`)**双重 unwrap**:既假设请求一定带 `inode`,又假设查找一定成功,任一不成立即 crash 服务线程。
- 启动失败或卸载异常会直接 crash,而非返回错误并优雅退出,影响服务可用性与排障。
- 反例(已做对的部分):`src/main.rs` 与 `src/daemon/mod.rs` 已实现 SIGINT/SIGTERM 优雅关闭、HTTP 先停再卸载主挂载、Antares 关闭清理带 15s 超时、daemon join 带 20s 超时。unwrap 清理应在这套既有优雅退出框架内推进,而非另起炉灶。

### 1.11 script 目录是临时脚手架
- `run.sh` / `run.rs` / `run_1000_files.sh` 为含中文注释的临时压测脚本,未集成进 `cargo bench` 或任务 runner。
- `perf_test.md` 原引用开发者本机路径,现已改为 `/tmp/scorpio-megadir/...`;脚本仍属于临时压测资产,尚未集成进 `cargo bench` 或任务 runner。

### 1.12 无 shell completion / man page
- 无 bash/zsh/fish 补全生成,无 man page,Linux 发行版打包与日常使用体验欠缺。

### 1.13 运行时状态文件入库与写入不一致

`config.toml`(`works=[]`)为运行时状态,已提交进仓库。更严重的是状态写入路径不统一:见配置文档中的持久化路径表——`fetch.rs` 用 `config_file()` 但静默丢弃错误,`daemon/mod.rs`/`manager/mod.rs` 仍硬编码 `"config.toml"`。
- `config.toml`(`works=[]`)实为运行时状态文件,却被提交进仓库;`.gitignore` 仅忽略 `test_config.toml`,未忽略 `config.toml`。
- 多人/多机共用仓库时易产生冲突与脏工作区。

---

## 二、改进方向

- `[P0]` **统一 CLI 子命令**:把 `antares` 的 `mount/umount/list/serve/http-mount` 模式合并进主 `scorpio` 二进制,采用 clap subcommand;主二进制不再是"只能 serve"。保留 `antares` 作为兼容别名至少一个 minor 版本。
- `[P0]` **统一日志**:全代码路径走 `tracing`,`main.rs` 初始化 `tracing_subscriber::fmt().with_env_filter(...)`,支持配置项 `log_level`;移除 `println!`/`eprintln!`/无换行 `print!`。
- `[P1]` **废弃/合并老 API**:明确 `/api/fs/*` 为 deprecated,文档统一指向 `/antares/mounts`(或后续统一的 `/api/v1/mounts`);在主端口也暴露 `/antares/*`(当前已 nest),并在文档中标注迁移路径。四类历史**文档**债务已在 Phase 0 清偿:(a) **【已完成】** 文档化 `GET /api/fs/select/{request_id}`;(b) **【已完成】** 从 `docs/api.md` 删除已禁用的 git 路由结构(启用 git 单列后续里程碑);(c) **【已完成】** 统一 antares 端点前缀口径——`docs/antares.md` 新增"访问方式与 URL 前缀"章节,主进程 `/antares/*` 与独立 `serve` 根路径双写;(d) **【已完成】** 补齐 `GET /mounts/{mount_id}/ready` 文档。**仍待办(代码侧):** 给 `/api/fs/*` 与 `POST /api/config` 实际打 deprecation 标记(响应头/日志)并统一 `status` 字段大小写。
- `[P1]` **shell completion**:用 `clap_complete` 生成 bash/zsh/fish 补全,随 release 分发;提供 `scorpio completions <shell>` 子命令。
- `[P1]` **主端口加根级 `/health`**:在 `2725` 根路径暴露轻量 `GET /health`(复用既有 Antares 健康逻辑),返回服务状态与可选挂载数量,不做远端或深度 FUSE 检查;保留 `/antares/health` 兼容。readiness 可复用已存在的 `/antares/mounts/{id}/ready` 或新增全局 `/ready`。
- `[P1]` **修 lib.rs doc 示例**:更正 crate 名(`scorpiofs`)或显式 `extern crate scorpiofs as scorpio;`,移除 `ignore` 启用 doctest,保证 `cargo test --doc` 通过。
- `[P2]` **替换生产路径 unwrap**:将 daemon 启动、卸载等路径上的 `unwrap()`/`panic!` 改为 `Result` 传播与优雅退出。
- `[P2]` **`scorpio doctor` 子命令**:检查 fuse 内核模块、`/etc/fuse.conf`、目录权限、与 mega server 连通性、配置有效性,输出诊断报告。
- `[P2]` **`config.toml` 出库**:将 `config.toml` 加入 `.gitignore`,仓库仅保留 `scorpio.toml.example` 模板;运行时按需生成。
- `[P3]` **perf 脚本集成**:将 `script/` 压测脚本改造为 `cargo bench` 用例或独立 `xtask`,移除开发者本机路径。
- `[P3]` **banner 清理**:banner 仅在交互式 TTY 输出,或通过 `--quiet` 关闭,避免污染 systemd/journald 日志;并将其移到 `Args::parse()` 之后,使 `--help`/`--version` 输出干净。

---

## 三、验收标准

- `scorpio serve / mount / umount / list` 统一入口可用,与原 `antares` 子命令行为一致。
- `curl http://<host>:2725/health` 返回轻量 JSON 健康状态,且不泄露敏感本地路径。
- `scorpio completions bash` 生成可用补全脚本。
- `cargo test --doc` 全绿,lib.rs 示例可编译。
- 启动/卸载失败时输出结构化错误日志并带非零退出码,而非 panic。
- `git status` 在正常运行后不出现 `config.toml` 改动。
- `scorpio doctor` 能定位常见部署问题(fuse 缺失、权限不足、mega 不可达)。


## 四、计划复核与补充约束

| 维度 | 复核结论 | 调整要求 |
|---|---|---|
| 合理性 | 统一 CLI/API、日志和健康检查能明显降低使用成本。 | 保留主线,但先定义推荐入口和兼容期。 |
| 可行性 | clap 子命令和 tracing 迁移可分步完成;API 合并需要兼容策略。 | P0 做统一入口框架和日志,P1 再收敛旧 API。 |
| 完整性 | 原计划缺少 API 迁移窗口、health 语义、错误码/退出码约定。 | 补充 deprecation、liveness/readiness、结构化错误要求。 |
| 安全性 | CLI/API 合并可能暴露更多挂载操作;当前**无鉴权**且默认 `0.0.0.0`。 | health 不返回完整路径;部署文档要求防火墙;P3+ 评估 token/mTLS;管理操作不得扩大未授权面。 |
| 功能正确性与接口兼容性 | 删除 `antares` 或旧 `/api/fs/*` 会破坏脚本。 | `antares` 二进制和旧 endpoint 至少保留一个 minor 版本。 |
| 数据流与控制流 | 原计划未定义 CLI 到 HTTP/manager 的调用边界。 | CLI 应复用 service/manager 层,不要通过本地 HTTP 自调用实现核心操作。 |
| 性能与效率 | 日志和 health 若过度采集会增加热路径开销。 | 默认 info 级别;health 只读内存状态,深度检查交给 `doctor`。 |
| 可靠性与容错性 | unwrap 清理方向正确,但需要统一错误类型。 | CLI 返回稳定非零退出码,HTTP 返回结构化错误 JSON。 |
| 兼容性与互操作性 | completion、man page 有利于包管理器集成。 | completion 随 release 分发,man page 可后续由 clap_mangen 生成。 |
| 可扩展性与可维护性 | 统一命令树和 API 文档能降低漂移。 | CLI help、README 示例和 API 文档应加入测试或生成流程。 |
| 合规性与标准符合性 | 日志进入 journald/stdout 符合服务化习惯。 | banner 只在交互式 TTY 输出,遵循常见 Unix 退出码语义。 |

### 推荐 CLI 命令树

```text
scorpio serve [--config-path <path>] [--http-addr <addr>]
scorpio mount <path>
scorpio umount <path|id>
scorpio list
scorpio config init|validate|show
scorpio doctor
scorpio completions <bash|zsh|fish>
```

兼容要求:

- 当前 `scorpio -c ... --http-addr ...` 作为 `serve` 的兼容写法保留,输出 deprecation 提示但不立即移除。
- `antares` 二进制保留为兼容别名或薄包装,至少一个 minor 版本内不删除。
- CLI 挂载/卸载直接调用共享 service/manager 层,不要依赖本地 HTTP 端口已启动。

### API 与健康检查语义

- 明确推荐 API 为 `/antares/mounts` 或后续统一后的 `/api/v1/mounts`;`/api/fs/*` 保留兼容并输出 deprecation 信息。
- 为旧 API 到新 API 建立字段映射表,避免 `path`、`mountpoint`、`request_id` 等字段语义漂移。

| 旧 API (`/api/fs/*`) | 新 API (`/antares/*`) | 备注 |
|---|---|---|
| `POST /api/fs/mount` `{path, cl?}` | `POST /antares/mounts` `{path, cl?, job_id?}` | 新 API 支持 job 幂等 |
| `GET /api/fs/mpoint` | `GET /antares/mounts` | 字段结构不同,需迁移说明 |
| `POST /api/fs/unmount` `{path\|inode}` | `DELETE /antares/mounts/{id}` 或 `by-job` | 旧 API 用 POST |
| `GET /api/fs/select/{request_id}` | 无直接等价 | 需文档化或提供 `/antares` 侧替代 |
| `GET/POST /api/config` | 无 | POST deprecated;配置走文件+重启 |

- `GET /health` 作为 liveness:只检查进程、配置已加载、router 可响应,不访问远端 mega,不执行 FUSE 深度检查;实现应委托既有 `AntaresDaemon::healthcheck` 逻辑。
- readiness 分层:`GET /ready`(全局,待新增)或 `GET /antares/mounts/{mount_id}/ready`(已实现,按挂载粒度);深度检查用 `scorpio doctor`。
- health 响应可包含 `status`、`version`、`uptime`、`mount_count`,但不返回完整 workspace/store_path。

### 错误处理与日志要求

- daemon 启动、监听端口、挂载/卸载、路径查找、状态读写等生产路径改为 `Result` 传播。
- CLI 错误输出到 stderr,返回稳定非零退出码;HTTP 错误返回 JSON,包含错误码、message 和 request context。
- 测试代码中的 `unwrap()` 可保留,生产模块逐步清理。
- 日志等级优先级:`CLI --log-level > SCORPIO_LOG/RUST_LOG > config log_level > info`。
- 默认输出到 stdout/stderr,交给 journald/docker 收集;文件输出和轮转不阻塞 P0。
- banner 仅在交互式 TTY 且非 `--quiet` 时输出。

### 文档与仓库卫生验收补充

- `cargo test --doc` 进入 CI;需要 FUSE/网络/特权的示例使用 `no_run` 而不是无条件 `ignore`。
- `scorpio completions bash` 在 CI 中至少验证命令可生成不 panic。
- 正常运行后 `config.toml` 等运行时状态文件不造成仓库脏状态。
- README Quick Start 必须覆盖推荐 CLI、推荐 API、health 和配置模板链接。

### 退出码约定(补充)

| 场景 | 退出码 |
|---|---|
| 成功 | 0 |
| 配置错误 | 2 |
| 挂载/卸载失败 | 3 |
| 端口绑定失败 | 4 |
| 内部错误 | 1 |

CLI 与脚本应依赖上述稳定码,而非解析 stderr 文本。
