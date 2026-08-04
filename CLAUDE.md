# mirrorstrike-container

MirrorStrike Docker 容器环境，基于 Kali Linux 构建，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景，预装完整工具链与 AI 编码代理（Claude Code / pi-coding-agent）。

## 项目结构

```
.
├── container/
│   ├── Dockerfile            # 完整版（四场景）Docker 镜像构建文件
│   └── Dockerfile.pentest    # 轻量渗透测试版 Docker 镜像构建文件
├── .github/
│   └── workflows/
│       └── docker-build.yml  # 自动化构建与推送工作流（完整版 + 渗透版）
├── CLAUDE.md
└── README.md
```

## 镜像变体

- **完整版**（`container/Dockerfile`）：四场景全覆盖，~4-6 GB
- **渗透版**（`container/Dockerfile.pentest`）：纯渗透测试 + AI 代理 + 浏览器，~2-3 GB；去掉 Ghidra、pwndbg、pwntools、CTF/审计工具和 JDK 21

## 任务场景与工具

- **渗透测试**: metasploit, netexec, pwncat, coercer, bloodyad, evil-winrm, impacket, sqlmap, hydra, proxychains4, nuclei(含模板), katana, naabu, nikto, dirsearch, dalfox, cloudfox, kerbrute, enum4linux-ng, sshpass, chisel
- **CTF**: pwntools, jwt_tool, binwalk, foremost, steghide, pycryptodome, z3-solver
- **代码审计**: semgrep, trivy, gitleaks, nuclei
- **Pwn / 逆向**: gdb + **pwndbg**, ghidra(JDK21), gdb-multiarch, patchelf, qemu-user / qemu-user-static, ROPgadget, ropper, one_gadget, libc6-dbg
- **AI 代理**: claude-code / pi-coding-agent
- **浏览器自动化**: playwright (chromium), agent-browser（渲染走 apt chromium；entrypoint 自动启 Xvfb `:99`）

## 关键约定（修改 Dockerfile 必读）

- **运行时挂载点**: `/opt/tools`、`/opt/workspace` 由用户运行时 `-v` 挂载，**Dockerfile 中不可往里装任何工具**；内置源码工具（ysoserial / jwt_tool / jdwp-shellifier / pwndbg）统一放 `/home/kali/tools`。
- **JDK 策略**: 默认 JDK 8（sdkman `current`，已用 `ENV JAVA_HOME/PATH` 让非交互 shell 生效）；Ghidra 12.x 需 JDK 21，经 `GHIDRA_JDK_HOME` 单独指向，**改默认 JDK 会破坏 Ghidra**。
- **pwndbg**: 必须以运行用户（kali）身份安装、固定路径 `/home/kali/tools/pwndbg`（setup.sh 会写 `~/.gdbinit`）；勿以 root 安装（否则配置落到 `/root`，kali 加载不到）。
- **apt 源**: 第 1 步已固定到 `kali.download`，避免默认 `http.kali.org` 重定向到失效 mirror（neusoft）；CI 与本地都受益。
- **pi 执行器依赖**: `@earendil-works/pi-coding-agent`（锁定版本 ARG）与 `@modelcontextprotocol/sdk` 随镜像全局安装；`/opt/mirrorstrike/node_modules` 软链到 `$(npm root -g)`，供 pi 扩展（运行时位于宿主机挂载的 `/opt/mirrorstrike/pi-agent`，可能不含 node_modules）做 ESM 裸导入解析。**Node 必须 >= 22.19**，构建期有版本断言，升级 pi 主版本时需同步复核。
- **pi 挂载点**: `/opt/mirrorstrike/pi-agent` 为只读共享层（扩展/CLAUDE.md/models.json），`/opt/mirrorstrike/pi-agent/agents` 为可写会话层；Dockerfile 仅建占位目录，不可向其中安装工具。
- **WORKDIR 中立**: 镜像默认 `WORKDIR /opt/mirrorstrike`，仅影响 `docker run/exec` 的默认 shell 位置；Agent 进程 cwd 由 wrapper 分别设置为 claude-code / pi-agent，修改 WORKDIR 不影响执行器。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像至 `ghcr.io/hanc00l/mirrorstrike-container`：
- `latest`（完整版）
- `latest-pentest`（渗透版）

## 参考

[Cairn](https://github.com/oritera/Cairn)
