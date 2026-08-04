# mirrorstrike-container

基于 Kali Linux 的 MirrorStrike Docker 容器环境，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景，预装完整工具链与 AI 编码代理（Claude Code / pi-coding-agent）。

## 镜像变体

| 变体 | 标签 | 用途 | 大小（估算）|
|------|------|------|------------|
| **完整版** | `latest` | 四场景全覆盖（渗透 + CTF + 审计 + Pwn）| ~4-6 GB |
| **渗透版** | `latest-pentest` | 纯渗透测试 + AI 代理 + 浏览器自动化 | ~2-3 GB |

## 快速开始

```bash
# 完整版
docker pull ghcr.io/hanc00l/mirrorstrike-container:latest
docker run -it ghcr.io/hanc00l/mirrorstrike-container:latest

# 轻量渗透版
docker pull ghcr.io/hanc00l/mirrorstrike-container:latest-pentest
docker run -it ghcr.io/hanc00l/mirrorstrike-container:latest-pentest
```

## 渗透版 vs 完整版

渗透版（`-pentest`）去掉了 CTF、代码审计和 Pwn/逆向工具，保留完整的渗透测试和 AI 代理能力：

| 类别 | 完整版 | 渗透版 |
|------|--------|--------|
| 渗透工具（metasploit、nuclei、sqlmap 等）| ✅ | ✅ |
| Claude Code + pi-coding-agent + Playwright + agent-browser | ✅ | ✅ |
| JDK 8（ysoserial 等）| ✅ | ✅ |
| JDK 21 + Ghidra | ✅ | ❌ |
| pwndbg + GDB 调试链 | ✅ | ❌ |
| Pwn 工具（pwntools、ROPgadget、ropper 等）| ✅ | ❌ |
| CTF 工具（binwalk、steghide、z3-solver 等）| ✅ | ❌ |
| 审计工具（semgrep、trivy、gitleaks）| ✅ | ❌ |

## 运行时挂载点

容器预留以下路径，供运行时通过 `-v` 注入（镜像内不安装任何工具到这些目录）：

| 路径 | 用途 |
|---|---|
| `/opt/tools` | 第三方工具（用户挂载）|
| `/opt/workspace` | 工作目录（用户挂载）|
| `/opt/mirrorstrike/claude-code` | Claude Code 工作区 |
| `/opt/mirrorstrike/pi-agent` | pi 执行器共享层（只读；扩展、CLAUDE.md、models.json）|
| `/opt/mirrorstrike/pi-agent/agents` | pi 会话层（可写，仅会话文件）|
| `/opt/mirrorstrike/logs/agent` | Agent 日志 |

内置工具位于 `/usr/local/bin` 与 `/home/kali/tools`，不占用上述挂载点。

镜像默认 `WORKDIR` 为 `/opt/mirrorstrike`（中立位置）；Agent 进程的实际工作目录由
Worker 按执行器设置——claude 为 `/opt/mirrorstrike/claude-code`，pi 为
`/opt/mirrorstrike/pi-agent`。

## 完整版工具链

### 渗透测试
metasploit、netexec、pwncat、coercer、bloodyad、evil-winrm、impacket、sqlmap、hydra、proxychains4、nuclei（含模板）、katana、naabu、nikto、dirsearch、dalfox、cloudfox、kerbrute、enum4linux-ng、sshpass、chisel

### CTF
pwntools、jwt_tool、binwalk、foremost、steghide、pycryptodome、z3-solver、requests、beautifulsoup4

### 代码审计
semgrep、trivy、gitleaks、nuclei（DAST）

### Pwn 与逆向
gdb + pwndbg、ghidra（JDK 21）、gdb-multiarch、patchelf、qemu-user / qemu-user-static、ROPgadget、ropper、one_gadget、libc6-dbg

### AI 代理与浏览器
- [Claude Code](https://github.com/anthropics/claude-code)
- [pi-coding-agent](https://github.com/earendil-works/pi-mono)（`pi` CLI，锁定 0.83.0；Node >= 22.19，构建期有版本断言）
- Playwright（Chromium）
- agent-browser（渲染走 apt chromium，headless；Xvfb `:99` 由 entrypoint 自动启动）

pi 执行器复用 Claude Code 工作区的 `.claude/skills/*`（经 `--skill` 逐个挂载），MCP 工具经
`pi-agent/extensions/mirrorstrike-mcp.ts` 适配注册；镜像内置
`@modelcontextprotocol/sdk`，并在 `/opt/mirrorstrike/node_modules` 提供软链，供
`/opt/mirrorstrike/pi-agent`（运行时由宿主机挂载）下的扩展做 ESM 依赖解析兜底。

## JDK 策略（完整版）

- 默认 JDK **8**（Zulu，供 ysoserial 等老反序列化工具）— 由 sdkman 管理，`ENV JAVA_HOME/PATH` 已设，非交互 shell / agent 调用同样生效。
- Ghidra 12.x 需 JDK 21，经 `GHIDRA_JDK_HOME` 单独指向 JDK 21，与默认 JDK 8 解耦。

## 构建

```bash
# 完整版
docker build -t ghcr.io/hanc00l/mirrorstrike-container:latest container/

# 渗透版
docker build -t ghcr.io/hanc00l/mirrorstrike-container:latest-pentest -f container/Dockerfile.pentest container/
```

> CI（GitHub Actions，国外网络）可直接构建。**国内本地构建**较慢且 `auth.docker.io` 易被 DNS 污染，需配置代理并预拉基础镜像。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送两个镜像：
- `ghcr.io/hanc00l/mirrorstrike-container:latest`（完整版）
- `ghcr.io/hanc00l/mirrorstrike-container:latest-pentest`（渗透版）

## 参考

[Cairn](https://github.com/oritera/Cairn)

## 许可

本项目仅用于合法的安全测试和教育目的。
