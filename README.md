# mirrorstrike-container

基于 Kali Linux 的 MirrorStrike Docker 容器环境，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景：**AI 执行器仅 pi-coding-agent**、**无 metasploit**、**零 GitHub 下载工具**（两镜像仅允许 URL+SHA256 锁定的 pinned 资产例外）、**交互单 JDK8**（jadx/apktool 等需新 JVM 的工具经 wrapper 绑定独立 JVM 隔离），重量级工具与 nuclei 模板经运行时挂载提供。

## 镜像变体

| 变体 | 标签 | 用途 | 大小（估算）|
|------|------|------|------------|
| **完整版** | `latest` | 四场景显式最小清单（渗透 + CTF + 审计 + Pwn）；**2026-10-16 起为 tsec-hosted 工具面超集**（无 server 端组件、数据库客户端完整直装）| ~5 GB 未压缩（2026-10-16 工具面收敛后估算，未实测）|
| **tsec 托管版** | - | tsecbench 托管 all-in-one（内置 PG/Redis server 端），仅用于托管仿真环境。**唯一最终版**：2026-10-15 由原基线与 slim 合并收敛（slim 文件已删除）；含 nmap 入口修复（setcap 削 `cap_net_admin` + 非 root 强制 `--unprivileged`），逐项对照、体积收敛史与排除项见 [AGENTS.md](AGENTS.md) | ~4.9 GB 未压缩（估算，未实测）/ 压缩 <3 GB（build-image.sh 3GB 门禁强制）|

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
| 零预装字典 | seclists/dirb/wordlists(rockyou) 不随镜像，由平台 `dicts/` 能力同步承担（两镜像同）|
| 零 GitHub 下载 | 构建期仅允许 URL+SHA256 锁定的 pinned 资产（chisel/grpcurl/jadx/apktool/capa，与 tsec-hosted 同源同值）；nuclei/naabu 用 Kali 源版本 |
| AI 执行器 pi-only | 仅 `@earendil-works/pi-coding-agent`（**版本锁定 1.4.0** + 漂移断言；+ MCP SDK 解析兜底）|
| 单 JDK8 | SDKMAN Zulu（交互/Agent 面唯一 JDK，安装后剔除 JavaFX）；jadx/apktool 经 wrapper 绑定 `/opt/android/jre`（jlink 裁剪 JRE21，pinned 1.5.6/3.0.2）运行，ghidra 等 JDK21 工具由 reverse 专用镜像承载 |
| playwright 可选内置 | `WITH_PLAYWRIGHT=1` 默认开：apt chromium + npm playwright/@playwright/mcp + `playwright-cli` wrapper（固定 `/usr/bin/chromium`），经平台 `mcp/mcp.json` 接线 |
| sudo 免密 | 运行用户 kali(uid 1000)，适配平台高安全档（镜像默认用户 + sudo 提权）|
| 无 server 端组件 | 不装 postgresql/redis-server/tini；数据库客户端完整直装（mysql/psql/redis-cli），PG/Redis 服务端由平台侧提供 |
| /opt 全挂载 | `tools`/`kb`/`workspace`/`traffic` 仅建占位目录，运行时 `-v` 注入 |

## 运行时挂载点

容器预留以下路径，供运行时通过 `-v` 注入（镜像内不安装任何工具到这些目录）：

| 路径 | 用途 |
|---|---|
| `/opt/tools` | 第一方工具发布根（`deepstrike-kit`、`infor-gather`，只读挂载）|
| `/opt/kb` | 知识库与 nuclei 模板（`/opt/kb/nuclei-templates`，只读挂载）|
| `/opt/workspace` | 项目工作区（projects/<pid>，平台按任务目录细粒度挂载）|
| `/opt/traffic` | 流量存档 |
| `/opt/mirrorstrike/pi-agent` | pi 执行器共享层（只读；扩展、CLAUDE.md、models.json）|
| `/opt/mirrorstrike/pi-agent/agents` | pi 会话层（可写，仅会话文件）|
| `/opt/mirrorstrike/logs/agent` | Agent 日志 |

镜像默认 `WORKDIR` 为 `/opt/mirrorstrike`（中立位置）；Agent 进程的实际工作目录由
Worker 设置为 `/opt/mirrorstrike/pi-agent`。镜像无 ENTRYPOINT——平台 Worker 统一以
`--entrypoint tail -f /dev/null` 常驻容器并经 `docker exec` 拉起 Agent。

## 完整版工具链

### 渗透测试
nmap、masscan、nuclei（模板走 `/opt/kb/nuclei-templates` 挂载）、naabu、sqlmap、gobuster、ffuf、feroxbuster、netexec、coercer、bloodyad、enum4linux-ng、evil-winrm、impacket、smbclient、hydra、john、proxychains4、sshpass、chisel、whatweb、wafw00f、searchsploit（exploitdb）、grpcurl、kubectl、数据库客户端（mysql/psql/redis-cli）；fscan、gsl5-cli、mdut-cli、JNDIExploit、suo5、frp 经 `/opt/tools/deepstrike-kit` 挂载提供；独立 ysoserial 如节点保留，则只走 `/opt/tools/ysoserial` bash 路线

### CTF
pwntools、ROPgadget、one_gadget、pycryptodome、z3-solver、binwalk、foremost、steghide、exiftool、hashid、fcrackzip、requests、beautifulsoup4

### 代码审计
semgrep（`~/.local/bin`）、gitleaks、nuclei（DAST）

### Pwn 与逆向
gdb + gdb-multiarch、strace、radare2、yara、patchelf、qemu-user / qemu-user-static、ROPgadget、one_gadget、libc6-dbg、jadx 1.5.6 / apktool 3.0.2（pinned + jlink 裁剪 JRE21，落 `/opt/android`）、capa v9.4.0 + 离线规则（落 `/opt/capa`）、adb/apksigner/zipalign/aapt；ghidra 等正式逆向管线由 mirrorstrike `deploy/reverse-static` 专用镜像经 `run_reverse_tool` 承载

### AI 代理与浏览器
- [pi-coding-agent](https://github.com/earendil-works/pi-mono)（`pi` CLI；Node >= 22.19，构建期有版本断言）
- Playwright + @playwright/mcp（浏览器走 apt chromium，无头；有头场景用 `xvfb-run` 兜底）

pi 扩展经 `pi-agent/extensions/mirrorstrike-mcp.ts` 注册 MCP 工具；镜像内置
`@modelcontextprotocol/sdk`，并在 `/opt/mirrorstrike/node_modules` 提供软链，供
`/opt/mirrorstrike/pi-agent`（运行时由宿主机挂载）下的扩展做 ESM 依赖解析兜底。

## JDK 策略（完整版）

- 仅 JDK **8**（Zulu，供 ysoserial 等老反序列化工具）— 由 sdkman 管理，`ENV JAVA_HOME/PATH` 已设，非交互 shell / agent 调用同样生效；fx 变体安装后剔除 JavaFX 产物（反射链不触，省 ~100MB）。
- 单 JDK 是刻意的：反序列化利用链对新 JDK 兼容差；jadx/apktool 经 wrapper 绑定 `/opt/android/jre`（jlink 裁剪 JRE21）运行，与 JDK8 互不污染；ghidra 等需要 JDK 21 的工具不在本镜像。

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
