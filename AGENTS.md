# mirrorstrike-container

MirrorStrike Docker 容器环境，基于 Kali Linux 构建，面向 **渗透测试 / CTF / 代码审计 / Pwn** 四类任务场景：**AI 执行器仅 pi-coding-agent**、**无 metasploit**、**零 GitHub 下载工具**（两镜像仅允许 URL+SHA256 锁定的 pinned 资产例外）、**交互单 JDK8**（jadx/apktool 等需新 JVM 的工具经 wrapper 绑定独立 JVM 隔离），重量级自研/敏感工具与 nuclei 模板经运行时挂载（`/opt/tools`、`/opt/kb` 等）提供。

## 项目结构

```
.
├── container/
│   ├── Dockerfile                  # 完整版（四场景，tsec-hosted 工具面超集）构建文件
│   └── Dockerfile.tsec-hosted      # tsecbench 托管 all-in-one 唯一最终版（含 PG/Redis server 端）
├── .github/
│   └── workflows/
│       └── docker-build.yml     # 自动化构建与推送工作流（完整版）
├── AGENTS.md
└── README.md
```

## 镜像变体

- **完整版**（`container/Dockerfile`）：四场景显式最小清单。**2026-10-16 重构为 tsec-hosted 工具面超集**——tsec-hosted 有的全包含（pinned chisel/grpcurl/jadx/apktool/capa、exploitdb/kubectl/浏览器层门控、nmap 入口修复、playwright-cli wrapper、psmisc/python3-setuptools 收口等），另保留完整版刚需（代码审计 semgrep、跨架构 Pwn qemu-user/-static + libc6-dbg）。与 tsec-hosted 的差异面：
  - **无 server 端组件**：不装 postgresql/redis-server/tini；数据库客户端完整直装（mysql/psql/redis-cli = default-mysql-client/postgresql-client/redis-tools）
  - pinned 资产与 tsec-hosted 同源同值（URL+SHA256 锁定）；android 栈落 `/opt/android`、capa 落 `/opt/capa`（`/opt/tools` 为运行时 :ro 挂载点，镜像内不可装物，见「关键约定」）
  - ysoserial/JNDIExploit/frp/fscan/gsl5-cli/mdut-cli/suo5 等仍走 `/opt/tools` 运行时挂载（tsec-hosted 为构建期 COPY，工具面等价）
  - 2026-10-15 起同样**不预装任何字典**（seclists/dirb/rockyou 已移除）
- **tsec 托管版**（`container/Dockerfile.tsec-hosted`）：**唯一最终版**（2026-10-15 由原基线与 slim 精简变体合并收敛，slim 文件已删除）。要点：
  - 体积收敛：去 qemu-user/-static（~780MB，跨架构 PWN 仿真不做）、chisel-common-binaries → pinned 单架构 chisel deb（URL+SHA256 锁定）、JDK8 fx 变体安装后剔除 JavaFX（~100MB，反射链不触）
  - nmap 入口修复（2026-10-15）：`setcap` 削 file caps 的 `cap_net_admin`（exec 子集校验恒过）+ `/usr/local/bin/nmap` wrapper 非 root 强制 `--unprivileged`（security 档自动降级 connect 扫描，消除 "Operation not permitted"/QUITTING——round-3 251 次 / round-4 25 次报错收口）；仅托管模式需要，平台常规 capability 档（root+CAP ALL）nmap 全功能不受影响
  - **零预装字典**（2026-10-15 契约）：seclists（实测 1,861MB）/dirb/wordlists(rockyou) 全部移出 apt（dnsrecon 为 DNS 枚举工具本体，保留）；字典由 mirrorstrike 平台字典库 `dicts/`（应用树随 `app/` COPY 进镜像）承担，Worker 物化到任务工作区，skills 按任务相对路径引用保证一致性
  - 工具面对齐 dict-base AGENTS.md 预装清单：安卓链 adb/apksigner/zipalign/aapt、make/zip、python 科学库（numpy/sympy/Pillow/lxml/PyJWT/pymongo/psycopg2/pymysql，pg8000 走 pip）、bc/sqlite3/ripgrep/fd-find/php-cli/tesseract/wafw00f/whatweb/checksec/rlwrap/ncat/arping/ldap-utils、exploitdb（`WITH_EXPLOITDB=0` 可关）、grpcurl（pinned deb）、`playwright-cli` wrapper（固定系统 Chromium）
  - 门控：`WITH_KUBECTL=1`（kubectl，dict-base 硬承诺）/ `WITH_CLOUD=0`（awscli/trivy，d 类红区题面拍板后再开）/ `WITH_PLAYWRIGHT=1`（浏览器层默认开）
  - 排除项（缺失即预期）：qemu-user/-static、hashcat（无 GPU，john 覆盖）、ghidra、metasploit、semgrep、ropper、nikto、dirsearch、一切预装字典
  - 构建统一走 mirrorstrike `deploy/hosted/build-image.sh`（原 build-image-slim.sh 已删；导出 + **3GB 压缩门禁**在该脚本内强制）
  - 仅用于托管仿真环境

## 任务场景与工具

- **渗透测试**: nmap, masscan, nuclei, naabu, sqlmap, gobuster, ffuf, feroxbuster, netexec, coercer, bloodyad, enum4linux-ng, evil-winrm, impacket, smbclient, hydra, john, proxychains4, sshpass, chisel, whatweb, wafw00f, searchsploit（exploitdb）, grpcurl, 数据库客户端（mysql/psql/redis-cli）, kubectl（fscan、gsl5-cli、mdut-cli、ysoserial、JNDIExploit、suo5、frp 等经 `/opt/tools` 挂载提供）
- **CTF**: pwntools, ROPgadget, one_gadget, pycryptodome, z3-solver, binwalk, foremost, steghide, exiftool, hashid, fcrackzip
- **代码审计**: semgrep（`~/.local/bin`）, gitleaks, nuclei
- **Pwn / 逆向**: gdb + gdb-multiarch, strace, radare2, yara, patchelf, qemu-user / qemu-user-static, ROPgadget, one_gadget, libc6-dbg, jadx, apktool, capa（两镜像同 pinned jadx 1.5.6 / apktool 3.0.2 / capa v9.4.0 + jlink 裁剪 JRE21；完整版落 `/opt/android`、`/opt/capa`，tsec-hosted 落 `/opt/tools/android`、`/opt/tools/capa`；ghidra 等正式逆向管线由 mirrorstrike `deploy/reverse-static` 专用镜像经 `run_reverse_tool` 承载）
- **AI 代理**: 仅 `@earendil-works/pi-coding-agent`（+ `@modelcontextprotocol/sdk` 解析兜底）
- **浏览器自动化**: playwright + @playwright/mcp + `playwright-cli` wrapper（两镜像同：浏览器走 apt chromium 固定 `/usr/bin/chromium`，无头；平台 `mcp/mcp.json` 以 `--executable-path` 接线；有头场景用 `xvfb-run` 兜底）

## 预装工具清单（两镜像对照）

> 2026-10-15 按 dict-base《预装工具》清单 + round-4 复盘 §10 逐项核对整理；2026-10-16 完整版重构为 tsec-hosted 工具面超集，下表为重构后对照。`✅`=预装，`❌`=不预装（有意排除与原因见下节），括注差异形态。两镜像均**零预装字典**（平台 `dicts/` 承担）。

| 用途 | 工具或命令 | Dockerfile（完整版） | Dockerfile.tsec-hosted（tsec 托管版） |
| --- | --- | --- | --- |
| 端口与服务发现 | `nmap`、`masscan`、`naabu`、`arping` | ✅ | ✅ |
| Web 扫描与目录爆破 | `nuclei`、`sqlmap`、`gobuster`、`ffuf`、`feroxbuster` | ✅ | ✅ |
| 域与 Windows 枚举、利用 | `netexec`、`enum4linux-ng`、`bloodyad`、`coercer`、`impacket`、`evil-winrm`、`smbclient`、`ldapsearch` | ✅ | ✅ |
| 口令与凭据 | `hydra`、`john`、`hashid`、`fcrackzip`、`gitleaks` | ✅ | ✅ |
| 连接与隧道 | `nc`、`ncat`、`socat`、`sshpass`、`proxychains4`、`chisel`、`rlwrap`、`tmux`、`openssh-client` | ✅（chisel=pinned 单架构 amd64 deb，URL+SHA256 锁定） | ✅（同左） |
| 集群运维 | `kubectl` | ✅（`WITH_KUBECTL=1` 默认开） | ✅（同左） |
| Pwn 与二进制分析 | `gdb`、`gdb-multiarch`、`radare2`、`binutils`、`xxd`、`ltrace`、`strace`、`patchelf`、`qemu-user`/`qemu-user-static`、`gcc`、`make`、`checksec`、`libc6-dbg`、`libc6-dev` | ✅（qemu 系 + libc6-dbg 保留——完整版跨架构 Pwn 承诺） | ✅（qemu-user/-static、`libc6-dbg` ❌ 有意排除；`libc6-dev` 已显式安装（gcc 头文件，2026-10-15 补）；`make`/`checksec` ✅） |
| 逆向（Android / 恶意样本） | `jadx`、`apktool`、`adb`、`apksigner`、`aapt`/`aapt2`、`zipalign`、`capa`、`yara` | ✅（jadx 1.5.6 / apktool 3.0.2 pinned + jlink 裁剪 JRE21 落 `/opt/android`；capa v9.4.0 落 `/opt/capa`） | ✅（同版 pinned，落 `/opt/tools/android`、`/opt/tools/capa`） |
| Java 反序列化与 JNDI | `ysoserial`、`jndi-injection-exploit` | 独立 `/opt/tools/ysoserial` 可选，不属于 kit | 不随当前 ZIP/hosted 分发；JNDIExploit 仍由 kit 提供 |
| 取证与隐写 | `binwalk`、`foremost`、`steghide`、`exiftool`、`unzip`、`zip`、`7z`、`unrar-free`、`xz` | ✅ | ✅ |
| 抓包与 DNS 枚举 | `tshark`、`tcpdump`、`dnsutils`、`dnsrecon`、`whois` | ✅ | ✅ |
| 文件搜索与数据处理 | `rg`、`fd`、`jq`、`yq`、`file`、`bc`、`sqlite3` | ✅ | ✅ |
| OCR 与 WAF 指纹 | `tesseract`(+chi-sim)、`wafw00f`、`whatweb` | ✅ | ✅ |
| CVE 检索与 gRPC 探测 | `searchsploit`（exploitdb）、`grpcurl` | ✅（`WITH_EXPLOITDB=1` 默认开；grpcurl pinned deb） | ✅（同左） |
| 浏览器自动化 | `chromium`、`xvfb`、`playwright`、`@playwright/mcp`、`playwright-cli` | ✅（`WITH_PLAYWRIGHT=1` 默认开；playwright-cli wrapper 固定 `/usr/bin/chromium`） | ✅（同左） |
| 代码审计 SAST | `semgrep` | ✅（pip `--user`，`~/.local/bin`） | ❌（有意排除） |
| 语言与常用库 | Python 3（`pwntools`、`ROPgadget`、`z3`、`pycryptodome`、`requests`、`bs4`、`paramiko`；cryptography 随 paramiko 传递带入）、Node.js + npm、Ruby、Java JDK 8（sdkman zulu） | ✅ | ✅（JDK8 装 fx 变体后剔除 JavaFX，~省 100MB） |
| Python 扩展库 | `numpy`、`sympy`、`Pillow`、`lxml`、`PyJWT`、`pymongo`、`psycopg2`、`pymysql`、`pg8000`、`php-cli` | ✅（pg8000 无 apt 包走 pip 兜底） | ✅（同左） |
| 数据库客户端 | `mysql`、`psql`、`redis-cli` | ✅（postgresql-client/redis-tools 显式直装，仍无 server 端组件） | ✅（随 postgresql/redis-server 服务端依赖带入） |
| 服务端与常驻 | `postgresql`、`redis-server`、`tini` | ❌（平台侧提供；无 ENTRYPOINT，Worker 覆盖 `tail -f /dev/null`） | ✅（all-in-one 托管，ENTRYPOINT=tini） |
| 基础命令行 | `git`、`curl`、`wget`、`sudo`、`less`、`vim-tiny`、`psmisc` | ✅ | ✅（`psmisc` ❌，待同步） |

### 有意排除（缺失即预期）

**两镜像共同**：
- **一切预装字典**（seclists/dirb/wordlists(rockyou)/PasswordDic）：2026-10-15 契约，由平台 `dicts/` 能力同步、Worker 物化到任务工作区；`dnsrecon` 的字典部分随包移除，DNS 枚举工具本体保留。
- **metasploit / hashcat / nikto / dirsearch**：显式最小清单；web 面由 nmap+sqlmap+gobuster/ffuf/feroxbuster+nuclei 覆盖，口令由 john 覆盖（无 GPU）。
- **ghidra / pwndbg / katana / dalfox / cloudfox / kerbrute / nuclei-templates 等**：零 GitHub 下载约定（pinned 资产例外除外，见「关键约定」）；ghidra 类 JDK21 逆向管线走 `deploy/reverse-static` 专用镜像 `run_reverse_tool`。
- **ropper**：依赖 filebytes 的 setup.py 使用 Python 3.12+ 移除的 `ast.Str`，本底座不可构建且 Kali 无 apt 包；ROPgadget + pwntools ROP 已覆盖。
- **docker CLI / docker.sock**：agent 本就运行在任务容器内（round-4 实测 49 次无效探测，靠环境事实注入收敛）。

**完整版独有不装**：
- **postgresql / redis-server / tini**：无 server 端组件原则，PG/Redis 由平台侧提供；数据库客户端完整直装（psql=postgresql-client、redis-cli=redis-tools 显式安装，不再随 server 依赖带入）。
- **nikto / dirsearch**：web 面覆盖足够（whatweb 已随 2026-10-16 收敛补齐）。

**tsec-hosted 独有不装**（2026-10-15 收敛裁决，复核确认为有意排除）：
- **qemu-user / qemu-user-static**（~780MB）：跨架构 PWN/仿真不做；gdb-multiarch/ROPgadget/r2 静态分析链不受影响。**完整版保留 qemu 系 + libc6-dbg**（跨架构 Pwn 承诺）。
- **chisel-common-binaries**（~121MB 全架构）：换 pinned 单架构 amd64 deb（两镜像同）。
- **libc6-dbg**：仅 qemu 跨架构调试场景需要，随 qemu 一并移除（完整版保留）。
- **semgrep**：tsec 托管场景不含代码审计管线（完整版仍提供）。
- **JavaFX 产物**：JDK8 fx 变体安装后剔除（~100MB），反序列化反射链不触（2026-10-16 起两镜像同）。

### tsec-hosted 增补史（2026-10-16 起完整版已全量同步）

依据 `docs/缺什么工具与缺什么内容-2026-10-14.md` + round-4 §10 实测 + dict-base 清单对齐。2026-10-16 完整版重构时以下增补全部并入（唯一形态差异：ysoserial/JNDIExploit/frp/fscan 等在完整版仍走 `/opt/tools` 运行时挂载，tsec-hosted 为镜像 COPY）：
- 高频缺失件：`bc`（73 次缺失，全场最高）/ `sqlite3` / `rg`+`fd` / `php-cli` / `tesseract-ocr`(+chi-sim) / `wafw00f` / `whatweb` / `checksec` / `rlwrap` / `ncat` / `iputils-arping` / `ldap-utils`
- 安卓链：`adb` / `apksigner` / `zipalign` / `aapt`（round-4 §10 坐实的唯一真缺失）
- 门控层：`exploitdb`（`WITH_EXPLOITDB=1` 默认开）、`grpcurl`（pinned deb）、`kubectl`（`WITH_KUBECTL=1` 默认开）、`awscli`/`trivy`（`WITH_CLOUD=0` 红区待拍板）
- 构建配套：`make` / `zip`、python 科学/协议库（numpy/sympy/Pillow/lxml/PyJWT/pymongo/psycopg2/pymysql，pg8000 走 pip 兜底）
- 入口统一：`playwright-cli` wrapper（固定系统 Chromium，对齐 dict-base 用法）；jadx/apktool 改 pinned GitHub 版 + jlink 裁剪 JRE21；渗透工具统一由 `deepstrike-kit.zip` 发布，hosted 在安装侧过滤后 COPY 到 `/opt/tools/deepstrike-kit`

### 复核待办

- ✅ `psmisc`（killall/pstree，~1MB）：2026-10-16 已落入完整版（tsec-hosted 待同步）。
- ✅ `python3-setuptools`：2026-10-16 已落入完整版（python3-pip 已不传递带入，dict-base 原镜像为显式安装；tsec-hosted 待同步）。
- tsec-hosted 当前不携带独立 `ysoserial`；`JNDIExploit` 由 `/opt/tools/deepstrike-kit` 提供。普通 Worker 如保留独立 `/opt/tools/ysoserial`，只走 bash 路线，均不提供 `/usr/local/bin` wrapper。

## 关键约定（修改 Dockerfile 必读）

- **运行时挂载点**: `/opt/tools`、`/opt/kb`、`/opt/workspace`、`/opt/traffic` 由平台运行时 `-v` 挂载（kb/tools 只读），不再挂载 `/opt/deepstrike`。完整版的这些目录仅建占位目录并 chown 给 kali，不可往里装工具；tsec-hosted 例外是构建期 COPY 经 catalog 过滤的同一 `deepstrike-kit.zip` 安装内容，不复制旧项目或散落工具目录。
- **零 GitHub 下载（pinned 资产例外两镜像同）**: 除 URL+SHA256 锁定的 pinned 资产（chisel/grpcurl/jadx/apktool/capa/Temurin JDK21、capa-rules，升级须同步更新哈希）外，构建期禁止任何 GitHub release / git clone 依赖（katana/dalfox/cloudfox/kerbrute/trivy/ysoserial/jwt_tool/jdwp-shellifier/pwndbg/nuclei-templates 均不在镜像内）；nuclei 模板指向挂载路径 `/opt/kb/nuclei-templates`（`NUCLEI_TEMPLATES_DIR` + `~/.config/nuclei/config.yaml`，禁联网更新），capa 规则随镜像同禁联网更新（完整版 `/opt/capa/rules`、tsec-hosted `/opt/tools/capa/rules`）。
- **JDK 策略**: 交互 java 仅 JDK 8（sdkman `current`，已用 `ENV JAVA_HOME/PATH` 让非交互 shell 生效）。单 JDK 是刻意的：反序列化利用链对新 JDK 兼容差。jadx/apktool 两镜像同为 pinned 1.5.6 / 3.0.2 + jlink 裁剪 JRE21，wrapper 显式指向（完整版 `/opt/android/jre`、tsec-hosted `/opt/tools/android/jre`——`/opt/tools` 在完整版为运行时 :ro 挂载点，故另择 `/opt/android`），与 JDK8 互不污染；Kali apt 版 jadx/apktool 及其传递带入的 JVM25（default-jre）依赖链已在完整版移除。ghidra 等 JDK21 工具不在本镜像（走 reverse-static 专用镜像）。
- **sudo 免密**: 运行用户 kali(uid 1000) 带 `NOPASSWD:ALL` sudo——平台高安全档（`docker_profile=security`）以镜像默认用户运行，低权限工具链依赖 sudo 提权。
- **无 server 端组件**: 不装 postgresql/redis-server/tini 等；PG/Redis 由平台侧提供，容器常驻由 Worker `docker run --init --entrypoint tail -f /dev/null` 实现。
- **无 ENTRYPOINT**: 镜像只留 `CMD ["/bin/bash"]`；平台 Worker 统一覆盖 entrypoint 常驻容器、经 `docker exec` 拉起 Agent。
- **apt 源**: 第 1 步已固定到 `kali.download`，避免默认 `http.kali.org` 重定向到失效 mirror（neusoft）；CI 与本地都受益。
- **pi 执行器依赖**: `@earendil-works/pi-coding-agent`（**版本锁定 `PI_VERSION=1.1.0`**，构建期断言实际安装版本防 npm latest 漂移）与 `@modelcontextprotocol/sdk` 随镜像全局安装；`/opt/mirrorstrike/node_modules` 软链到 `$(npm root -g)`，供 pi 扩展（运行时位于宿主机挂载的 `/opt/mirrorstrike/pi-agent`，可能不含 node_modules）做 ESM 裸导入解析。**Node 必须 >= 22.19**，构建期有版本断言，升级 pi 主版本时需同步复核。
- **pi 挂载点**: `/opt/mirrorstrike/pi-agent` 为只读共享层（扩展/CLAUDE.md/models.json），`/opt/mirrorstrike/pi-agent/agents` 为可写会话层；Dockerfile 仅建占位目录，不可向其中安装工具。
- **WORKDIR 中立**: 镜像默认 `WORKDIR /opt/mirrorstrike`，仅影响 `docker run/exec` 的默认 shell 位置；Agent 进程 cwd 由 wrapper 设置为 pi-agent，修改 WORKDIR 不影响执行器。

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像至 `ghcr.io/hanc00l/mirrorstrike-container`：
- `latest`（完整版）

## 参考

[Cairn](https://github.com/oritera/Cairn)
