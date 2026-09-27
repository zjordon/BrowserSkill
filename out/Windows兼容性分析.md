# BrowserSkill - Windows 兼容性分析

> 生成时间: 2026-09-27 · 分析版本: v0.3.1 (commit 8cbcc49) · 分析环境: Windows 11 x64

## 总体评估

| 指标 | 结果 |
|------|------|
| 总体评级 | ✅ **完全兼容（一等公民支持）** |
| 高危问题 | 0 |
| 中危问题 | 2（均为设计上的行为差异，非缺陷） |
| 低危问题 | 6 |
| 已正确适配项 | 20+（覆盖进程模型、IPC、信号、路径、安装、更新、CI） |

### 评级说明

Windows x64 是官方发布目标平台之一（`release-cli.yml` 产出 `x86_64-pc-windows-msvc` zip）。该项目对 Windows 的支持**不是条件编译凑合**，而是为 daemon 进程模型、IPC、自更新三大核心机制各写了完整的原生实现，配有 3 个 Windows 专用集成测试文件 + 3 个 Windows CI job。历史上多个 Windows 特有 bug（#75 特殊用户名管道、#180 句柄继承）已修复并有回归测试。

## 1. 构建系统兼容性

### 1.1 package.json Scripts

| Script | 命令 | 问题 | 严重程度 |
|--------|------|------|----------|
| `eval:browser:check` | `node ... && node --test ...` | `&&` 在 cmd 下可用，glob `*.test.mjs` 由 `node --test` 自身展开，不依赖 shell | ✅ 无问题 |
| 其余脚本 | `cargo run` / `pnpm -r` / `pnpm --filter` | 均为跨平台调用 | ✅ 无问题 |

未发现仅 bash 语法的 scripts；也没有 `rm -rf`、`cp -r` 类 Unix 命令。

### 1.2 安装与脚本入口

| 文件 | 说明 |
|------|------|
| `install.ps1` | PowerShell 5.1 兼容的 Windows 原生安装器（详见 §4） |
| `install.sh` | `#!/bin/sh`，POSIX 系统专用，与 ps1 内容对等 |
| `scripts/*.mjs` | Node 跨平台脚本；`check-dsh-package.mjs` 已处理 Windows 的 `npm.cmd` 与路径空格 |

结论：**不存在"只有 .sh 版本"的功能入口**，双版本安装器内容对等。

### 1.3 构建工具链

| 工具 | Windows 支持 | 说明 |
|------|-------------|------|
| Cargo（edition 2024, rust 1.85） | ✅ | MSVC 目标发布 |
| pnpm 10 workspace | ✅ | `pnpm-workspace.yaml` 的 `allowBuilds`（esbuild、spawn-sync）均已放行 |
| WXT 0.20 / Vite / Tailwind v4 | ✅ | 标准跨平台前端链 |
| tsdown / lightningcss | ✅ | DSH 插件构建 |
| Biome / stylelint / Vitest | ✅ | 纯跨平台 |
| GitHub Actions | ✅ | `ci.yml` 含 3 个 windows-latest job；`release-cli.yml` 产出 Windows 包；仅 `release-dsh-plugin.yml` 与 `release-extension.yml` 是 Linux-only（产物平台无关，无影响） |

## 2. 源代码兼容性

### 2.1 平台依赖隔离

- `crates/bsk-cli/Cargo.toml:65-67`：`nix` / `libc` 严格限定 `[target.'cfg(unix)'.dependencies]`；
- `:69-77`：Windows 侧 `windows-sys 0.59`，features 精确覆盖 `Win32_System_JobObjects`、`Win32_System_Threading`、`Win32_System_Pipes` 等；
- `nix` 的全部调用点（`daemon/lockfile.rs:89-98`、`daemon/start.rs:1101-1121`）均在 `#[cfg(unix)]` 内，Windows 不参与编译。

### 2.2 daemon 后台进程创建（Windows 深度定制，超出常规水准）

**核心文件**: `crates/bsk-cli/src/windows_process.rs`、`crates/bsk-cli/src/daemon/start/windows.rs`

标准库 `std::process::Command` 无法阻止其他可继承句柄泄漏给子进程（issue #180）。本项目手工 `CreateProcessW`：

| 机制 | 实现 | 证据 |
|------|------|------|
| 显式句柄继承白名单 | `PROC_THREAD_ATTRIBUTE_HANDLE_LIST` + `STARTUPINFOEXW`，只继承 duplicate 出的 3 个 stdio 句柄 | `windows_process.rs:118-150` |
| stdio 隔离 | stdio 指向 NUL 文件 | `start/windows.rs:37-41` |
| 分离启动 | `DETACHED_PROCESS \| CREATE_NEW_PROCESS_GROUP \| CREATE_BREAKAWAY_FROM_JOB \| CREATE_SUSPENDED` | `start/windows.rs:42-49` |
| Job 逃逸验证 | 先挂起创建 → `IsProcessInJob` 确认已逃出宿主 Job → 才 `ResumeThread`；失败则杀挂起子进程并输出 DETACH_HINT 指引 | `windows_process.rs:68-80, 225-232`；`start/windows.rs:50-55` |
| 命令行转义 | 按 CRT 规则手工处理反斜杠/引号，UTF-16 路径，不经 shell | `start/windows.rs:61-87` |
| 诊断 | daemon 启动日志记录 `in_job` 状态 | `start.rs:326-332` |

### 2.3 Unix API 的 Windows 替代实现（完整对照表）

| Unix 机制 | Windows 替代 | 证据 |
|-----------|--------------|------|
| UDS 服务端 | `tokio::net::windows::named_pipe`（first_pipe_instance、交接前预建下一实例防 NotFound） | `daemon/ipc.rs:1502-1617` |
| UDS 客户端 | `ClientOptions` + `ERROR_PIPE_BUSY` 50ms 重试（5s 预算） | `ipc_client.rs:172-195` |
| `peer_cred()` 取对端 PID | `GetNamedPipeServerProcessId` | `ipc_client.rs:198-206` |
| SIGTERM/SIGKILL 两段式停止 | `OpenProcess(PROCESS_TERMINATE)` + `TerminateProcess`（一步到位，见风险 1） | `daemon/start.rs:1125-1138` |
| `kill(pid,0)` 存活探测 | `OpenProcess(PROCESS_SYNCHRONIZE)` + `WaitForSingleObject`（正确处理 exit code 259 STILL_ACTIVE） | `daemon/lockfile.rs:101-119` |
| SIGINT/SIGTERM 优雅关停监听 | `tokio::signal::ctrl_c()` | `daemon/start.rs:921-937` |
| 子进程 Ctrl-C 取消 | **stdin EOF + `BSK_CANCEL_ON_STDIN_CLOSE=1`** 触发 CLI 取消 RPC | `cli/business_rpc.rs:197-228` |
| `setsid` + /dev/null detach | 父进程提供 NUL 句柄，detach_stdio 空操作 | `start.rs:995-1000` |
| chmod 0600/0700（约 15 处） | 无操作（依赖用户目录默认 ACL，见风险 3） | `paths.rs:67-74` 等 |
| POSIX rename 覆盖 | `MoveFileExW(MOVEFILE_REPLACE_EXISTING\|MOVEFILE_WRITE_THROUGH)` | `cli/atomic_output.rs:20-42` |
| 原子替换运行中 exe | 目标 exe 旁 staged `.cmd` 脚本（cmd.exe 等待解锁→替换→重启 daemon） | `cli/update.rs:747-757`、`cli/update/windows.rs:14-58` |
| 目录 fsync | 跳过（DSH 插件侧同样处理） | `packages/.../start-journal.ts:143-150` |

### 2.4 路径处理

- **无硬编码 `/` 拼接**：对 `crates/bsk-cli/src` 全量搜索 `.join("/")` 无命中；路径全部 `PathBuf::join`（`daemon/paths.rs:92-129`、`skill_install/harness.rs:81-99`）。唯一例外 `build.rs:20` 将嵌入资源 `\` 归一化为 `/`（有意的资源命名规范）；
- `~` 展开统一走 `dirs::home_dir()`，可被 `BSK_HOME` 覆盖；
- 环境变量：`USERNAME`（管道名哈希 `paths.rs:143`）、`LOCALAPPDATA`（Hermes harness 目录 `harness.rs:331-340`）、`SystemRoot`（定位 cmd.exe `update/windows.rs:21-23`）；**无注册表读写**；
- 上传文件名同时拒绝 `/` 和 `\`（`daemon/file_transfer.rs:452`）；
- symlink 防护用 `fs::symlink_metadata`（Windows 也能识别 symlink/junction）。

### 2.5 数据目录与 IPC 端点

| 项 | Windows 值 |
|----|-----------|
| 数据目录 | `C:\Users\<用户>\.bsk`（或 `BSK_HOME`） |
| IPC 端点 | `\\.\pipe\bsk-daemon-<16位hex>`（USERNAME+BSK_HOME 的哈希）——**刻意 hash-only 以规避特殊用户名导致 NPFS 异常**（issue #75），有单测锁定字符集 `paths.rs:229-246` |
| 扩展 ↔ daemon | TCP WebSocket `ws://127.0.0.1:52800`（平台无关）；CLI 不探测浏览器可执行文件路径（扩展主动连入），规避了 Windows 注册表/Program Files 探测问题 |

### 2.6 TS 侧平台分支

| 位置 | 内容 |
|------|------|
| `apps/extension/src/long-screenshot/source.ts:23-31` | **Windows 专属渲染 workaround**：Agent 截图走 CDP 源 + checkFreshness |
| `apps/extension/src/tools/screenshot-full-page.ts:183-215, 252` | Windows 上未激活标签页停止产出合成帧 → 用 32px `Page.startScreencast` 保活渲染器 + 陈旧帧校验 |
| `packages/dsh-plugin-browserskill/src/runner.ts:103-121` | Windows 上 Node SIGINT 直接杀子进程 → 改 stdin EOF 取消通道；杀宽限 15s（Unix 3s） |
| `runner.ts:140-145` | Windows spawn 加 `windowsHide: true` |
| 扩展全局 | 无 `navigator.platform` / `process.platform` 误用，用 `chrome.runtime.getPlatformInfo()` |

对应测试均有 Windows 分支覆盖（`runner.test.ts:402`、`runner-exit.test.ts:24-176`）。

## 3. CI/CD 覆盖

`.github/workflows/ci.yml` 中的 Windows job：

| Job | 内容 |
|-----|------|
| `windows-daemon`（:43-94） | 命名管道并发/断连、审计存储、**受限 Job 内拒绝启动回归**、进程存活检测、skill 安装迁移、自更新、取消进程（`windows_parent_cancel`）、**安装器在 Windows PowerShell 5.1 与 PowerShell 7 各跑一遍** |
| `windows-daemon-lifecycle`（:96-115） | `scripts/test-windows-daemon.ps1`：用 WMI `Win32_Process.Create` 在 runner Job 之外启动独立测试宿主（自证 `IsProcessInJob == false`），验证 daemon 逃逸 Job、启动器生命周期、自动更新替换重启 |
| `windows-dsh-package`（:158-184） | 构建 DSH 包，故意把 TEMP/TMP 设为含空格和 `&` 的路径（`"bsk package & cache"`）验证资源路径处理 |

发布矩阵（`release-cli.yml:75-112`）：`x86_64-pc-windows-msvc` → zip。

Windows 专用集成测试：`crates/bsk-cli/tests/windows_daemon_start.rs`（14 用例：Job 逃逸/存活、管道 EOF、并发启动者共享 daemon、自动启动复用、`BSK_AUTO_START=0`）、`windows_update.rs`（替换运行中 exe）、`windows_parent_cancel.rs`。

## 4. 安装器（install.ps1）评估

PS 5.1 兼容，质量高：

- 安装到 `$HOME\.local\bin\bsk.exe`；从 GitHub Releases 下载并按 `version.json` 校验 SHA256（`install.ps1:187-221`）；
- 架构检测含 WOW64（`PROCESSOR_ARCHITEW6432`，:40-65）；x64/ARM64（ARM64 无资产时明确报错 :183-185）；
- **staged 替换**：临时文件 → `& $Source daemon stop` → `[System.IO.File]::Replace`（:125-152），失败不破坏旧二进制；
- PATH 三重处理：用户 PATH（持久、去重置顶 :71-84）+ 会话 PATH（:86-91）+ **Git Bash `~/.bashrc`**（含 Windows→Unix 路径转换 :95-119）；
- 绕开 PS 5.1 通配符/编码坑（:206-209, 224-226）；装完自检 `bsk --version` 并警告 PATH 冲突（:248-255）；
- 有独立测试 `scripts/install-windows.test.ps1`（AST 解析隔离测试，覆盖架构矩阵与 PATH 合并语义）。

## 5. 潜在风险 / 已知问题清单

| # | 严重程度 | 问题 | 详情与证据 |
|---|----------|------|-----------|
| 1 | 【中】 | **`bsk daemon stop` 在 Windows 上无优雅关闭** | `daemon/start.rs:1107-1110` 中 `send_term` 在 Windows 直接等于 TerminateProcess；Unix 是 SIGTERM→5s→验身份→SIGKILL 两段式。运行中会话/传输没有排空机会。缓解：所有落盘均为 tmp+rename 原子写、审计追加带尾部换行校验（`daemon/audit.rs:177-186`）、`wait_for_stopped` 清理残留元数据、PID 复用防护（`probe.rs:42-53`） |
| 2 | 【中】 | **Job Object 受限环境拒绝启动 daemon（fail-closed 设计）** | 宿主（CI runner、部分 agent 沙箱）若把进程放入未设 `JOB_OBJECT_LIMIT_BREAKAWAY_OK` 的 Job，`CREATE_BREAKAWAY_FROM_JOB` 失败 → daemon 无法后台启动，提示 `--foreground` 或 `BSK_AUTO_START=0`（`start/windows.rs:17-20, 50-55`）。有回归测试锁定该行为（`windows_daemon_start.rs:392`）。**某些 Windows 宿主环境开箱即用会失败**，需按提示操作 |
| 3 | 【低-中】 | **文件权限未在 Windows 上收紧（安全 parity 差异）** | 全部 `chmod 0600/0700` 为 `#[cfg(unix)]`（`daemon/info.rs:90-95`、`remote/authorization.rs:123-127`、`file_transfer.rs:417-433`、`audit.rs:154-175` 等）。Windows 下 `daemon.json`、远程授权 token、传输暂存、审计文件依赖用户目录继承 ACL（默认每用户私有，通常可接受）但未显式设置；命名管道未传自定义 SECURITY_ATTRIBUTES（`daemon/ipc.rs:1521-1533`），依赖默认 DACL + hash 管道名隔离。属差异而非确认漏洞 |
| 4 | 【低】 | 自更新依赖 cmd.exe + 安装目录可写 | 更新助手是 exe 旁 `.cmd` 脚本（`update/windows.rs:21-33`）。装到 Program Files 等不可写目录时自动更新失败；默认 `~\.local\bin` 可写，规避了此问题；已处理 exe 占用等待/重试上限/失败保留旧二进制 |
| 5 | 【低】 | Windows ARM64 无官方包 | README 仅声明 Windows x64；`install.ps1:52` 拒绝非 x64/ARM64；发布矩阵无 ARM64 资产 → `install.ps1:183-185` 报错。ARM64 用户需源码编译 |
| 6 | 【低】 | 部分集成测试仅 Unix | `foreground_smoke.rs:47-49`（SIGTERM）、`daemon_discovery.rs`（UDS）、`bundle_tests.rs:594-630`（symlink）等 `#[cfg(unix)]` 门控；Windows 由 3 个专用测试文件 + CI job 补偿，但 `auto_spawn`、`idle_exit` 等在 Windows 上不跑 |
| 7 | 【低】 | `disown_daemon` 行为差异 | Unix 后台线程 wait() 回收僵尸（`start.rs:210-215`）；Windows 仅 drop（:217-220）——Windows 无僵尸进程概念，正确，但 CLI↔daemon 句柄关系不同（由 DETACHED_PROCESS + 显式句柄列表保证独立） |
| 8 | 【提示】 | Git Bash PATH 仅写 `~/.bashrc` | `install.ps1:95-119` 不写 `.profile`/`.zshrc`；从登录 shell 启动 bash 的用户可能需手动 source；安装结尾有会话级 PATH 自检警告（:252-260） |
| 9 | 【提示】 | 管道名派生依赖 `USERNAME` 稳定 | `paths.rs:142-148`：USERNAME 异常（服务账户/容器）时回退 `"default"`，多用户隔离弱化；单用户桌面场景无影响 |

## 6. 结论与建议

**结论**：BrowserSkill 在 Windows 上是经过深度工程化的一等支持——进程模型（CreateProcessW 显式句柄继承、Job 逃逸验证）、IPC（hash 命名管道 + 对端 PID 验证）、自更新（cmd helper 替换运行中 exe）均为原生实现，且有 14+ 用例的 Windows 专用集成测试与 3 个 CI job 兜底。历史上 #75、#180 等 Windows 特有 bug 均已闭环。

**对本机（Windows 11 x64）使用者的建议**：

1. 用 `install.ps1` 安装后重开终端（或重启 agent）使 PATH 生效；Git Bash 用户注意 `.bashrc` 已被写入但登录 shell 需 source；
2. 在 CI runner / 受限沙箱内运行若遇到 daemon 拒绝启动，属 Job 限制的 fail-closed 设计：改用宿主托管 daemon（固定 `BSK_HOME` + `BSK_AUTO_START=0`）；
3. 长截图在 Windows 上有专属渲染路径（未激活 tab 用 `Page.startScreencast` 保活），行为与 macOS/Linux 等价但机制不同，遇到截图异常时这是首先怀疑点；
4. `bsk daemon stop` 在 Windows 上是硬终止——有进行中传输时会话无排空，但暂存会在下次 daemon 启动时回收，无泄漏风险；
5. 对安全敏感部署（多用户共用机器），注意 `~/.bsk` 下敏感文件（远程授权 token 等）未显式收紧 ACL，可自行对目录设置更严格的访问控制。

## 附：Windows 相关关键文件索引

| 主题 | 路径 |
|------|------|
| 原生进程创建封装 | `crates/bsk-cli/src/windows_process.rs` |
| daemon Windows 启动 | `crates/bsk-cli/src/daemon/start/windows.rs` |
| IPC 双实现 | `crates/bsk-cli/src/daemon/ipc.rs`（unix :1373 / windows :1502） |
| 数据目录/管道名 | `crates/bsk-cli/src/daemon/paths.rs` |
| Windows 自更新 | `crates/bsk-cli/src/cli/update.rs` + `update/windows.rs` |
| stdin 取消通道 | `crates/bsk-cli/src/cli/business_rpc.rs` + `packages/dsh-plugin-browserskill/src/runner.ts` |
| 扩展渲染 workaround | `apps/extension/src/long-screenshot/source.ts`、`src/tools/screenshot-full-page.ts` |
| 安装器与测试 | `install.ps1`、`scripts/install-windows.test.ps1`、`scripts/test-windows-daemon.ps1` |
| Windows 专用测试 | `crates/bsk-cli/tests/windows_daemon_start.rs`、`windows_update.rs`、`windows_parent_cancel.rs` |
