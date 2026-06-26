# mirrorstrike-container

MirrorStrike Docker 容器环境，基于 Kali Linux 构建，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景，预装完整工具链与 AI 编码代理（Claude Code）。

## 项目结构

```
.
├── container/
│   └── Dockerfile          # Docker 镜像构建文件
├── .github/
│   └── workflows/
│       └── docker-build.yml # 自动化构建与推送工作流
├── CLAUDE.md
└── README.md
```

## 任务场景与工具

- **渗透测试**: metasploit, netexec, pwncat, coercer, bloodyad, evil-winrm, impacket, sqlmap, hydra, proxychains4, nuclei(含模板), katana, naabu, nikto, dirsearch, dalfox, cloudfox, kerbrute, enum4linux-ng, sshpass, chisel
- **CTF**: pwntools, jwt_tool, binwalk, foremost, steghide, pycryptodome, z3-solver
- **代码审计**: semgrep, trivy, gitleaks, nuclei
- **Pwn / 逆向**: gdb + **pwndbg**, ghidra(JDK21), gdb-multiarch, patchelf, qemu-user / qemu-user-static, ROPgadget, ropper, one_gadget, libc6-dbg
- **AI 代理**: claude-code（codex / pi-coding-agent 已移除）
- **浏览器自动化**: playwright (chromium), agent-browser + Chrome for Testing（entrypoint 自动启 Xvfb `:99`）

## 关键约定（修改 Dockerfile 必读）

- **运行时挂载点**: `/opt/tools`、`/opt/workspace` 由用户运行时 `-v` 挂载，**Dockerfile 中不可往里装任何工具**；内置源码工具（ysoserial / jwt_tool / jdwp-shellifier / pwndbg）统一放 `/home/kali/tools`。
- **JDK 策略**: 默认 JDK 8（sdkman `current`，已用 `ENV JAVA_HOME/PATH` 让非交互 shell 生效）；Ghidra 12.x 需 JDK 21，经 `GHIDRA_JDK_HOME` 单独指向，**改默认 JDK 会破坏 Ghidra**。
- **pwndbg**: 必须以运行用户（kali）身份安装、固定路径 `/home/kali/tools/pwndbg`（setup.sh 会写 `~/.gdbinit`）；勿以 root 安装（否则配置落到 `/root`，kali 加载不到）。
- **apt 源**: 第 1 步已固定到 `kali.download`，避免默认 `http.kali.org` 重定向到失效 mirror（neusoft）；CI 与本地都受益。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像至 `ghcr.io/hanc00l/mirrorstrike-container`。

## 参考

[Cairn](https://github.com/oritera/Cairn)
