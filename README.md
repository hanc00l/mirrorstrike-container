# mirrorstrike-container

基于 Kali Linux 的 MirrorStrike Docker 容器环境，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景：**AI 执行器仅 pi-coding-agent**、**无 metasploit**、**零 GitHub 下载工具**、**单 JDK8**，重量级工具与 nuclei 模板经运行时挂载提供。

## 镜像变体

| 变体 | 标签 | 用途 | 大小（估算）|
|------|------|------|------------|
| **完整版** | `latest` | 四场景显式最小清单（渗透 + CTF + 审计 + Pwn）| ~6.5 GB（未压缩，seclists 字典为主）|
| **tsec 托管版** | - | tsecbench 托管 all-in-one（内置 PG/Redis），仅用于托管仿真环境 | ~7.4 GB 未压缩 / ~2.77 GB 压缩（<3GB 预算）|
| **tsec 托管精简版** | - | tsec-hosted 精简+增补变体：去 qemu-user/static（跨架构仿真不做）与全架构 chisel（换 pinned 单架构 deb）、JDK8 剔除 JavaFX；按 venom 会话缺口统计增补 bc/sqlite3/rg+fd/php-cli/tesseract-ocr/wafw00f/whatweb/checksec/exploitdb/searchsploit/grpcurl 等；云 CLI 族经 `--build-arg WITH_CLOUD=0` 默认关。构建编排同基线，仅 `-f` 换本文件 | 未压缩 ~6.5 GB（估算，较基线省 ~1GB，未实测）|

## 快速开始

```bash
# 完整版
docker pull ghcr.io/hanc00l/mirrorstrike-container:latest
docker run -it ghcr.io/hanc00l/mirrorstrike-container:latest
```

## 设计要点（完整版）

| 要点 | 说明 |
|------|------|
| 显式最小 apt 清单 | 无元包、无 metasploit，按四场景显式列出工具 |
| 零 GitHub 下载 | 构建期无任何 GitHub release / git clone 依赖；nuclei/naabu 用 Kali 源版本 |
| AI 执行器 pi-only | 仅 `@earendil-works/pi-coding-agent`（+ MCP SDK 解析兜底）|
| 单 JDK8 | SDKMAN Zulu；ghidra 等 JDK21 工具由 reverse 专用镜像承载 |
| playwright 默认内置 | apt chromium + npm playwright/@playwright/mcp，经平台 `mcp/mcp.json`（`--executable-path /usr/bin/chromium`）接线 |
| sudo 免密 | 运行用户 kali(uid 1000)，适配平台高安全档（镜像默认用户 + sudo 提权）|
| 无 server 端组件 | 不装 postgresql/redis-server/tini；PG/Redis 由平台侧提供 |
| /opt 全挂载 | `tools`/`kb`/`workspace`/`deepstrike`/`traffic` 仅建占位目录，运行时 `-v` 注入 |

## 运行时挂载点

容器预留以下路径，供运行时通过 `-v` 注入（镜像内不安装任何工具到这些目录）：

| 路径 | 用途 |
|---|---|
| `/opt/tools` | 第三方工具（fscan/gsl5-cli/mdut-cli/ysoserial/JNDIExploit/suo5/frp 等，只读挂载）|
| `/opt/kb` | 知识库与 nuclei 模板（`/opt/kb/nuclei-templates`，只读挂载）|
| `/opt/workspace` | 项目工作区（projects/<pid>，平台按任务目录细粒度挂载）|
| `/opt/deepstrike` | deepstrike-kit MCP（入口文件存在才挂载，存在性门控）|
| `/opt/traffic` | 流量存档 |
| `/opt/mirrorstrike/pi-agent` | pi 执行器共享层（只读；扩展、CLAUDE.md、models.json）|
| `/opt/mirrorstrike/pi-agent/agents` | pi 会话层（可写，仅会话文件）|
| `/opt/mirrorstrike/logs/agent` | Agent 日志 |

镜像默认 `WORKDIR` 为 `/opt/mirrorstrike`（中立位置）；Agent 进程的实际工作目录由
Worker 设置为 `/opt/mirrorstrike/pi-agent`。镜像无 ENTRYPOINT——平台 Worker 统一以
`--entrypoint tail -f /dev/null` 常驻容器并经 `docker exec` 拉起 Agent。

## 完整版工具链

### 渗透测试
nmap、masscan、nuclei（模板走 `/opt/kb/nuclei-templates` 挂载）、naabu、sqlmap、gobuster、ffuf、feroxbuster、netexec、coercer、bloodyad、enum4linux-ng、evil-winrm、impacket、smbclient、hydra、john、proxychains4、sshpass、chisel；fscan、gsl5-cli、mdut-cli、ysoserial、JNDIExploit、suo5、frp 经 `/opt/tools` 挂载提供

### CTF
pwntools、ROPgadget、one_gadget、pycryptodome、z3-solver、binwalk、foremost、steghide、exiftool、requests、beautifulsoup4

### 代码审计
semgrep（`~/.local/bin`）、gitleaks、nuclei（DAST）

### Pwn 与逆向
gdb + gdb-multiarch、strace、radare2、yara、patchelf、qemu-user / qemu-user-static、ROPgadget、one_gadget、libc6-dbg；ghidra/jadx/apktool/capa 等正式逆向管线由 mirrorstrike `deploy/reverse-static` 专用镜像经 `run_reverse_tool` 承载

### AI 代理与浏览器
- [pi-coding-agent](https://github.com/earendil-works/pi-mono)（`pi` CLI；Node >= 22.19，构建期有版本断言）
- Playwright + @playwright/mcp（浏览器走 apt chromium，无头；有头场景用 `xvfb-run` 兜底）

pi 扩展经 `pi-agent/extensions/mirrorstrike-mcp.ts` 注册 MCP 工具；镜像内置
`@modelcontextprotocol/sdk`，并在 `/opt/mirrorstrike/node_modules` 提供软链，供
`/opt/mirrorstrike/pi-agent`（运行时由宿主机挂载）下的扩展做 ESM 依赖解析兜底。

## JDK 策略（完整版）

- 仅 JDK **8**（Zulu，供 ysoserial 等老反序列化工具）— 由 sdkman 管理，`ENV JAVA_HOME/PATH` 已设，非交互 shell / agent 调用同样生效。
- 单 JDK 是刻意的：反序列化利用链对新 JDK 兼容差；ghidra 等需要 JDK 21 的工具不在本镜像。

## 构建

```bash
# 完整版
docker build -t ghcr.io/hanc00l/mirrorstrike-container:latest container/
```

> CI（GitHub Actions，国外网络）可直接构建。**国内本地构建**较慢且 `auth.docker.io` 易被 DNS 污染，需配置代理并预拉基础镜像，例如：
> `docker build --build-arg HTTP_PROXY=http://192.168.52.1:7890 --build-arg HTTPS_PROXY=http://192.168.52.1:7890 ...`（socks5 同端口亦可用）。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像：
- `ghcr.io/hanc00l/mirrorstrike-container:latest`（完整版）

## 参考

[Cairn](https://github.com/oritera/Cairn)

## 许可

本项目仅用于合法的安全测试和教育目的。
