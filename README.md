# mirrorstrike-container

基于 Kali Linux 的 MirrorStrike Docker 容器环境，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景，预装完整工具链与 AI 编码代理（Claude Code）。

## 快速开始

```bash
docker pull ghcr.io/hanc00l/mirrorstrike-container:latest
docker run -it ghcr.io/hanc00l/mirrorstrike-container:latest
```

## 运行时挂载点

容器预留以下路径，供运行时通过 `-v` 注入（镜像内不安装任何工具到这些目录）：

| 路径 | 用途 |
|---|---|
| `/opt/tools` | 第三方工具（用户挂载）|
| `/opt/workspace` | 工作目录（用户挂载）|
| `/opt/mirrorstrike/claude-code` | Claude Code 工作区（默认 WORKDIR）|
| `/opt/mirrorstrike/logs/agent` | Agent 日志 |

内置工具位于 `/usr/local/bin` 与 `/home/kali/tools`，不占用上述挂载点。

## 四场景工具链

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
- Playwright（Chromium）
- agent-browser + Chrome for Testing（headless；Xvfb `:99` 由 entrypoint 自动启动）

## JDK 策略

- 默认 JDK **8**（Zulu，供 ysoserial 等老反序列化工具）— 由 sdkman 管理，`ENV JAVA_HOME/PATH` 已设，非交互 shell / agent 调用同样生效。
- Ghidra 12.x 需 JDK 21，经 `GHIDRA_JDK_HOME` 单独指向 JDK 21，与默认 JDK 8 解耦。

## 构建

```bash
docker build -t ghcr.io/hanc00l/mirrorstrike-container:latest container/
```

> CI（GitHub Actions，国外网络）可直接构建。**国内本地构建**较慢且 `auth.docker.io` 易被 DNS 污染，需配置代理并预拉基础镜像。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像至 `ghcr.io/hanc00l/mirrorstrike-container`。

## 参考

[Cairn](https://github.com/oritera/Cairn)

## 许可

本项目仅用于合法的安全测试和教育目的。
