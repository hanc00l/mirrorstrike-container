# mirrorstrike-container

MirrorStrike Docker 容器环境，基于 Kali Linux 构建，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景：**AI 执行器仅 pi-coding-agent**、**无 metasploit**、**零 GitHub 下载工具**、**单 JDK8**，重量级自研/敏感工具与 nuclei 模板经运行时挂载（`/opt/tools`、`/opt/kb` 等）提供。

## 项目结构

```
.
├── container/
│   ├── Dockerfile               # 完整版（四场景，pi-only）构建文件
│   └── Dockerfile.tsec-hosted   # tsecbench 托管 all-in-one 单镜像（含 PG/Redis server 端）
├── .github/
│   └── workflows/
│       └── docker-build.yml     # 自动化构建与推送工作流（完整版）
├── AGENTS.md
└── README.md
```

## 镜像变体

- **完整版**（`container/Dockerfile`）：四场景显式最小清单，~6.5 GB（未压缩，seclists 字典 ~1.9GB 为主）
- **tsec 托管版**（`container/Dockerfile.tsec-hosted`）：tsecbench 托管模式 all-in-one（内置 PG/Redis，直接 root 运行），含 chromium 默认层与 agent 刚需工具（git/tmux/r2/yara/feroxbuster/coercer/gitleaks/mysql 客户端），压缩后 ~2.6GB（<3GB 预算），仅用于托管仿真环境

## 任务场景与工具

- **渗透测试**: nmap, masscan, nuclei, naabu, sqlmap, gobuster, ffuf, feroxbuster, netexec, coercer, bloodyad, enum4linux-ng, evil-winrm, impacket, smbclient, hydra, john, proxychains4, sshpass, chisel（fscan、gsl5-cli、mdut-cli、ysoserial、JNDIExploit、suo5、frp 等经 `/opt/tools` 挂载提供）
- **CTF**: pwntools, ROPgadget, one_gadget, pycryptodome, z3-solver, binwalk, foremost, steghide, exiftool
- **代码审计**: semgrep（`~/.local/bin`）, gitleaks, nuclei
- **Pwn / 逆向**: gdb + gdb-multiarch, strace, radare2, yara, patchelf, qemu-user / qemu-user-static, ROPgadget, one_gadget, libc6-dbg（ghidra 等正式逆向管线由 mirrorstrike `deploy/reverse-static` 专用镜像经 `run_reverse_tool` 承载）
- **AI 代理**: 仅 `@earendil-works/pi-coding-agent`（+ `@modelcontextprotocol/sdk` 解析兜底）
- **浏览器自动化**: playwright + @playwright/mcp（浏览器走 apt chromium，无头；平台 `mcp/mcp.json` 以 `--executable-path /usr/bin/chromium` 接线；有头场景用 `xvfb-run` 兜底）

## 关键约定（修改 Dockerfile 必读）

- **运行时挂载点**: `/opt/tools`、`/opt/kb`、`/opt/workspace`、`/opt/deepstrike`、`/opt/traffic` 由平台运行时 `-v` 挂载（kb/tools/deepstrike 只读），**Dockerfile 中不可往里装任何工具**，仅建占位目录并 chown 给 kali。
- **零 GitHub 下载**: 镜像构建期禁止任何 GitHub release / git clone 依赖（katana/dalfox/cloudfox/kerbrute/trivy/ysoserial/jwt_tool/jdwp-shellifier/pwndbg/nuclei-templates 均不在镜像内）；nuclei 模板指向挂载路径 `/opt/kb/nuclei-templates`（`NUCLEI_TEMPLATES_DIR` + `~/.config/nuclei/config.yaml`，禁联网更新）。
- **JDK 策略**: 仅 JDK 8（sdkman `current`，已用 `ENV JAVA_HOME/PATH` 让非交互 shell 生效）。单 JDK 是刻意的：反序列化利用链对新 JDK 兼容差；ghidra 等 JDK21 工具不在本镜像。
- **sudo 免密**: 运行用户 kali(uid 1000) 带 `NOPASSWD:ALL` sudo——平台高安全档（`docker_profile=security`）以镜像默认用户运行，低权限工具链依赖 sudo 提权。
- **无 server 端组件**: 不装 postgresql/redis-server/tini 等；PG/Redis 由平台侧提供，容器常驻由 Worker `docker run --init --entrypoint tail -f /dev/null` 实现。
- **无 ENTRYPOINT**: 镜像只留 `CMD ["/bin/bash"]`；平台 Worker 统一覆盖 entrypoint 常驻容器、经 `docker exec` 拉起 Agent。
- **apt 源**: 第 1 步已固定到 `kali.download`，避免默认 `http.kali.org` 重定向到失效 mirror（neusoft）；CI 与本地都受益。
- **pi 执行器依赖**: `@earendil-works/pi-coding-agent` 与 `@modelcontextprotocol/sdk` 随镜像全局安装；`/opt/mirrorstrike/node_modules` 软链到 `$(npm root -g)`，供 pi 扩展（运行时位于宿主机挂载的 `/opt/mirrorstrike/pi-agent`，可能不含 node_modules）做 ESM 裸导入解析。**Node 必须 >= 22.19**，构建期有版本断言，升级 pi 主版本时需同步复核。
- **pi 挂载点**: `/opt/mirrorstrike/pi-agent` 为只读共享层（扩展/CLAUDE.md/models.json），`/opt/mirrorstrike/pi-agent/agents` 为可写会话层；Dockerfile 仅建占位目录，不可向其中安装工具。
- **WORKDIR 中立**: 镜像默认 `WORKDIR /opt/mirrorstrike`，仅影响 `docker run/exec` 的默认 shell 位置；Agent 进程 cwd 由 wrapper 设置为 pi-agent，修改 WORKDIR 不影响执行器。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像至 `ghcr.io/hanc00l/mirrorstrike-container`：
- `latest`（完整版）

## 参考

[Cairn](https://github.com/oritera/Cairn)
