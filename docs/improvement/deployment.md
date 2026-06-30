# 部署改进计划

> 评估版本:0.2.2
> 评估范围:打包、容器化、系统服务、CI/CD 发布、系统依赖、权限

## 一、背景与现状问题

当前 ScorpioFS 的部署路径仅支持"克隆源码 → 改配置 → `cargo build --release` → 手动起进程",缺乏任何面向生产或快速体验的打包与部署产物,新用户上手门槛高,运维侧也无标准化方式纳管。

具体问题如下:

### 1. 无打包与容器化产物

仓库中不存在 `Dockerfile`、`docker-compose`、systemd unit、`.deb`/`.rpm` 打包脚本、`Makefile`、`flake.nix`、`build.rs` 等任何打包或部署相关文件。用户必须具备 Rust 工具链才能运行,无法以"下载一个产物即用"的方式部署。

### 2. CI 无发布流水线

`.github/workflows/` 下仅有 `build.yml`、`ci.yml`、`clippy.yml`、`fmt.yml`、`test.yml`,全部只做编译、测试与 lint,没有针对 tag 的 release 流水线。`Cargo.toml` 已声明 `repository`、`keywords`、`categories`,且 crates.io 上已存在 `scorpiofs` 包,但仓库中没有任何自动发布二进制或发布到 crates.io 的自动化。用户只能自行编译,拿不到官方构建产物。

### 3. 系统依赖文档零散且不完整

`docs/develop.md` 仅 10 行,要求用户自行 `apt install libfuse-dev librust-openssl-dev`、修改 `/etc/fuse.conf` 启用 `user_allow_other`、在 `.cargo/config` 配置 `runner = 'sudo -E'`。这些步骤散落在文档中,无自动化安装脚本,容易遗漏。依赖 FUSE 内核模块与用户态库,但文档未说明如何确认内核已加载 fuse、如何处理 `/dev/fuse` 权限。

### 4. `init.bash` 名不副实

仓库根目录的 `init.bash` 仅 8 行,内容是创建本地测试目录(`./dictest`、`./lower/a..d`、`./upper/e`、`./workerdir`、`./true_temp`),与"init"语义不符,既不是系统初始化也不是服务初始化,容易误导用户以为是部署脚本。

### 5. 需要 root/sudo 但缺乏指引

FUSE 挂载需要特权或 `/etc/fuse.conf` 中的 `user_allow_other`。当前仅在 `docs/develop.md` 提到一句 `runner = 'sudo -E'`,没有提供 setuid 封装、Linux capabilities(如 `CAP_SYS_ADMIN`)配置、`/dev/fuse` 设备权限等生产环境常用做法的指引。容器化场景下如何以非 root 运行也未说明。

### 6. 端口规划缺失(非纯硬编码)

主二进制默认监听 `0.0.0.0:2725`(`--http-addr` 可覆盖),antares 二进制默认 `0.0.0.0:2726`(`--bind` 可覆盖)。即两者均可经 CLI 改绑,但默认值仍写死、且配置文件层不能覆盖(无 env、无 toml 端口项),README 与 API 文档已说明 CLI 可覆盖,但仍缺统一的端口规划文档;容器端口映射、反向代理、多实例部署时容易冲突。改进时应让端口同样进入"CLI > env > 配置文件 > 默认值"的统一覆盖链。

### 7. HTTP 服务无鉴权且默认全网监听

主进程 HTTP 路由(`/api/fs/*`、`/api/config`、`/antares/*`)均无认证中间件,默认绑定 `0.0.0.0`。在公网或共享网络部署时,任何可达客户端均可触发挂载/卸载。改进计划须在部署文档中明确防火墙/反向代理要求,并评估后续 token 或 mTLS 方案(P3+)。

### 8. `rfuse3` 已启用 `unprivileged` feature

`Cargo.toml` 中 `rfuse3` 依赖包含 `unprivileged` feature,与文档仅强调 `CAP_SYS_ADMIN`/`sudo` 的叙述不完全对齐。部署指引应补充:在支持的环境下优先尝试非特权挂载,失败再回退 capability/setuid 方案。

## 二、改进方向

### [P0] 提供 Dockerfile 与 docker-compose

- 采用多阶段构建:构建阶段基于 `rust:slim` 编译 release 二进制,运行阶段优先基于 `debian:slim`;确认 FUSE 动态依赖可满足前不优先使用 distroless。
- 运行镜像需处理 `/dev/fuse` 设备挂载与 `CAP_SYS_ADMIN` 能力。
- 提供 `docker-compose.yml`,内含 scorpiofs 服务与所依赖的 mega server,支持 `docker compose up` 一键拉起完整可用环境。
- 通过环境变量覆盖配置(与"配置改进"中的 env 覆盖项联动),避免把开发者路径写进镜像。
- **硬依赖**:env 覆盖(Phase 1)未落地前,不得将具体路径/URL 烘焙进镜像默认值。
- CI smoke 可复用现有 `testcontainers` dev-dependency 做 HTTP health 与最小挂载验证。

### [P0] 提供 systemd unit

- 提供 `scorpiofs.service` 与可选的 `scorpiofs-antares.service` 单元文件。
- 包含 `ExecStartPre` 创建运行目录(workspace / store_path / antares 各层目录)。
- 通过 `AmbientCapabilities=CAP_SYS_ADMIN` 授予挂载能力,避免以 root 长期运行。
- 配置 `Restart=on-failure`、`RestartSec=5s`、`LimitNOFILE`、`StartLimitBurst`/`StartLimitIntervalSec`、日志走 journald。
- `TimeoutStopSec=45` 或更高(代码内 daemon join 20s + Antares 清理 15s + 卸载余量);`KillMode=mixed`。
- 提供 `Type=simple` 配合 `/health` 就绪探针;`Type=notify` 仅在代码实现 `sd_notify` 后启用。
- `ExecStopPost` 可选执行 `fusermount -u` 清理残留挂载点(带 `-l` 懒卸载说明)。

### [P0] 提供 install.sh 安装脚本

- 检测并安装系统依赖(apt/dnf/pacman 分支):`libfuse-dev`/`fuse`、`openssl` 开发库。
- 检测 `/etc/fuse.conf` 是否启用 `user_allow_other`;默认只提示,自动修改必须通过显式参数开启。
- 从 GitHub Releases 下载对应平台预编译二进制到 `/usr/local/bin`(与下方 CI release 联动)。
- 生成示例配置 `/etc/scorpiofs/scorpio.toml`(使用通用路径,不含开发者本机路径)。
- 支持 `--version`、`--prefix`、`--uninstall` 参数。

### [P1] CI release 工作流

- 新增 `.github/workflows/release.yml`,由推送 `v*` tag 触发。
- 使用 `cargo-dist` 或 `cross` 在 `x86_64-unknown-linux-gnu` 与 `aarch64-unknown-linux-musl` 上构建静态/动态二进制。
- 产出物包括:裸二进制 tarball、`.deb`、`.rpm`、SHA256 校验和。
- 自动创建 GitHub Release 并上传二进制产物;`cargo publish` 需使用受保护环境或人工审批,不与普通 tag push 无门禁绑定。
- 与 `install.sh` 形成闭环:脚本从 Release 拉取对应平台产物。

### [P2] capabilities / setuid 部署指引

- 在 `docs/improvement/` 或部署文档中补充:如何用 `setcap cap_sys_admin+ep` 给二进制赋能、如何在容器中以非 root 运行、`/dev/fuse` 设备权限与 group 配置、`user_allow_other` 的安全影响。
- 说明内核 overlayfs 与本项目用户态 OverlayFs(libfuse-fs)的区别,避免运维误用 `mount -t overlay`。

### [P2] 改造或移除 `init.bash`

- 若保留,重命名为语义清晰的名称(如 `scripts/mktestdirs.sh`)并移至 `script/` 目录。
- 若删除,在文档中说明其历史用途。
- 不应让用户误以为它是部署初始化脚本。

## 三、验收标准

- `docker compose up` 可在干净环境拉起 scorpiofs + mega server,挂载点可读写,HTTP API 可访问。
- `systemctl start scorpiofs` 启动后,`systemctl status` 显示 active,日志进入 journald;kill 后自动重启。
- `curl -fsSL https://.../install.sh | sh` 可在全新 Ubuntu/Debian 机器上完成依赖安装、二进制下载、配置生成。
- 推送受保护的 `v0.3.0` tag 后,CI 自动产出 GitHub Release(含二进制、deb/rpm 如已支持、checksums);crates.io 发布需人工审批或独立受保护步骤。
- 文档中不再出现开发者本机路径作为部署示例。


## 四、计划复核与补充约束

| 维度 | 复核结论 | 调整要求 |
|---|---|---|
| 合理性 | 打包、容器、systemd、release 是部署体验的必要闭环。 | 保留方向,但部署产物依赖 env 覆盖和 health 先落地。 |
| 可行性 | Docker 和 systemd 可先做最小可用;跨架构 deb/rpm 和 crates.io 自动发布复杂度更高。 | P0 交付最小部署路径,P1 再做完整 release。 |
| 完整性 | 原计划缺少升级、卸载、回滚、FUSE 版本差异和供应链校验。 | 补充 dry-run、uninstall、checksum、回滚和兼容矩阵。 |
| 安全性 | `CAP_SYS_ADMIN`、`curl | sh`、自动修改 `/etc/fuse.conf` 和自动发布都有高风险。 | 默认显式确认,支持 dry-run,发布使用受保护环境和最小 secret 权限。 |
| 功能正确性与接口兼容性 | Docker/systemd 必须匹配最终 CLI/config 契约。 | 不依赖即将废弃的旧参数;旧入口保留兼容期。 |
| 数据流与控制流 | 原计划未定义启动前置检查顺序。 | 固定为安装 -> 校验 -> 建目录 -> 启动 -> health -> deep doctor。 |
| 性能与效率 | 多阶段 Docker 和 CI cache 可提升构建效率。 | 运行镜像只保留必要动态库;release 使用 cargo registry/cache。 |
| 可靠性与容错性 | systemd restart 不等于挂载清理;但代码已具备 SIGTERM/SIGINT 优雅退出与 Antares 关闭清理(15s 超时),systemd 配置应与之协同而非冲突。 | `TimeoutStopSec` 需 ≥ 代码内关闭超时(主进程 daemon join 20s + Antares 清理 15s),避免 systemd 在清理完成前 SIGKILL;增加 ExecStop、StartLimit、残留挂载处理(`fusermount -u`)和升级回滚说明。 |
| 兼容性与互操作性 | FUSE、OpenSSL、glibc/musl、发行版包名存在差异。 | 首批明确支持 Ubuntu/Debian x86_64 + libfuse3;注明 libfuse2 差异;其他平台 best effort。 |
| 可扩展性与可维护性 | 部署脚本易腐化。 | Dockerfile、install.sh、unit 文件必须进入 CI 语法或 smoke 校验。 |
| 合规性与标准符合性 | 需遵循 FHS、systemd hardening、供应链发布安全。 | 使用 `/etc/scorpiofs`、`/var/lib/scorpiofs`、`/run/scorpiofs`,发布 checksum/SBOM 可选。 |

### 安全边界修正

- `install.sh` 不应默认静默修改 `/etc/fuse.conf`;自动启用 `user_allow_other` 必须通过显式参数触发。
- 文档可以提供 `curl | sh` 快速路径,但同时必须给出下载、校验 SHA256、再执行的安全路径。
- 容器示例必须明确 `/dev/fuse`、capability 和宿主机内核模块要求,不能承诺所有 Docker 环境都能无权限运行 FUSE。
- `AmbientCapabilities=CAP_SYS_ADMIN` 和 `setcap cap_sys_admin+ep` 都属于高权限方案,文档必须解释风险和替代方案。
- crates.io 发布不应与普通 tag push 完全自动绑定;需要 protected environment 或人工审批。

### 推荐部署控制流

```text
下载/安装二进制
  -> 校验 SHA256 和版本
  -> 生成或读取 /etc/scorpiofs/scorpio.toml
  -> scorpio config validate
  -> 创建 /var/lib/scorpiofs 和 /run/scorpiofs
  -> systemd/docker 启动服务
  -> /health 轻量检查
  -> doctor/readiness 执行深度依赖检查
```

故障处理要求:

- 任一步失败都停止后续动作并输出可操作错误。
- 升级时保留旧二进制备份,新版本 health 失败可回滚。
- 卸载默认不删除用户数据目录;删除数据必须显式确认。
- `Type=notify` 只有在代码实际支持 sd_notify 后启用;否则保持 `Type=simple`。

### 部署验收补充

- `install.sh --dry-run` 能展示将执行的包管理器、下载、写文件和 systemd 操作。
- Release 产物带版本、commit、目标平台和 SHA256;校验失败时拒绝安装。
- systemd unit 使用专用用户,并限制可写路径到配置的运行目录;若某项 hardening 与 FUSE 冲突,需在注释中说明。
- Docker smoke test 至少验证 HTTP health 和一次最小挂载/读写流程。

### 多实例与互操作补充

- 多实例部署时 `workspace`、`store_path`、`antares_state_file` 必须实例隔离;共享 state 文件会导致挂载元数据冲突。
- 反向代理(Nginx/Traefik)仅需暴露 HTTP 端口;FUSE 挂载在宿主机/容器内完成,不可经 HTTP 代理。
- 与 k8s 集成时:liveness 用 `GET /health`;readiness 用 `GET /ready` 或 `GET /antares/mounts/{id}/ready`;挂载操作不适合作为探针逻辑。

### 发布供应链补充

- Release tarball 命名:`scorpiofs-{version}-{target}.tar.gz`,内含 `scorpio`、`antares`、`LICENSE`、`README`。
- 可选 SBOM:`cargo cyclonedx` 或 `cargo auditable`;最低要求 SHA256 checksums 文件。
- `cargo publish` 与 GitHub Release 解耦:二进制发布自动化,crate 发布人工审批。
