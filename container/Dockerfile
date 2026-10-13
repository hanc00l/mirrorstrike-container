# =============================================================================
# Dockerfile — mirrorstrike agent 执行容器（pi-only 精简版）
#
# 参考 Dockerfile.tsec-hosted 的显式最小清单重构，与旧完整版的关键差异：
#   - 弃用 kali-linux-headless 元包（含 metasploit 全家，旧镜像 16.8GB 主因），
#     改显式最小清单：渗透 + CTF + Pwn + 逆向（无 server 端组件）
#   - 无任何 GitHub 下载工具（katana/dalfox/cloudfox/kerbrute/trivy/ysoserial/
#     jwt_tool/jdwp-shellifier/pwndbg/nuclei-templates 全部取消）；nuclei/naabu
#     用 Kali 源版本。ysoserial/JNDIExploit/suo5/frp/fscan 等自研/敏感工具与
#     nuclei 模板均经运行时挂载提供（/opt/tools、/opt/kb，:ro）
#   - AI 执行器仅 pi-coding-agent（+ MCP SDK 解析兜底），无 claude-code/agent-browser
#   - 交互 java 仅 JDK8（SDKMAN zulu）；jadx/apktool 为 Kali apt 版（依赖链带入
#     JVM25（Kali default-jre，当前 openjdk-25）仅供二者运行；jadx 类文件为 Java 11+，
#     经 2b 节 /usr/local/bin wrapper 显式绑定 apt JVM，否则吃 sdkman JDK8 即崩；
#     交互默认 java 仍为 JDK8）；ghidra 等
#     JDK21 工具由 reverse 专用镜像（mirrorstrike deploy/reverse-static）承载，
#     agent 容器内以 radare2/gdb 兜底
#   - playwright 为默认能力：apt chromium + npm playwright/@playwright/mcp
#     （平台 mcp/mcp.json 以 --executable-path /usr/bin/chromium 接线，无头运行）
#   - 保留 kali 用户 + sudo 免密（docker_profile=security 高安全档以镜像默认用户运行）
#   - /opt/{tools,kb,workspace,deepstrike,traffic} 为运行时挂载点，仅建占位目录，
#     Dockerfile 内不可写入任何内容
#
# 注：不写 `# syntax=` 指令——该指令会让 buildx 去 Docker Hub 拉 frontend 镜像，
# 离线/受限环境常不可达；本文件只用到内置 frontend 支持的特性。
# =============================================================================
FROM --platform=amd64 kalilinux/kali-rolling:latest

# ---------- 版本号集中管理（改版本只动这里）----------
ARG JDK8=8.0.482.fx-zulu

ENV TZ=Asia/Shanghai \
    PYTHONUNBUFFERED=1 \
    PIP_BREAK_SYSTEM_PACKAGES=1 \
    PIP_NO_CACHE_DIR=1 \
    PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    PLAYWRIGHT_MCP_BROWSER=chromium \
    NUCLEI_TEMPLATES_DIR=/opt/kb/nuclei-templates

# ---------- 1. base（固定 apt 源到 kali.download，避免 mirrorbrain 重定向到失效 mirror 如 neusoft）----------
RUN echo "deb http://kali.download/kali kali-rolling main contrib non-free non-free-firmware" > /etc/apt/sources.list

# ---------- 2. apt 工具大合并（渗透 + CTF + Pwn + 逆向，显式最小清单，无元包/无 metasploit）----------
# 清单基线取 Dockerfile.tsec-hosted；whatweb/nikto/dirsearch 明确不装
#（nmap+sqlmap+gobuster/ffuf/feroxbuster+nuclei 覆盖 web 面）。
# 相对 tsec 基线的增补均为平台承诺（skills/system SKILL.md「PATH 实测可用」清单）或 agent 刚需：
#   git / openssh-client / iputils-ping / tmux / strace — coding agent、ssh 类利用与
#     常驻隧道（tmux/nohup 为平台文档化的跨 Agent 存活手段）刚需
#   radare2 / yara — 逆向/pwn 容器内主力（ghidra 随 JDK21 一并移除，正式逆向管线走
#     reverse 专用镜像 run_reverse_tool），且 r2 为技能承诺
#   feroxbuster / coercer / gitleaks — 枚举纪律 / 域渗透 / 审计的技能硬承诺
#   hashid / fcrackzip — 密码小件（R1-T1，对齐 tsec-hosted）；default-mysql-client —
#     数据库利用客户端（R1 对齐）
#   jadx / apktool — Android 逆向（R1-T2 对齐，Kali apt 版；依赖链带入 JVM25（Kali default-jre，当前 openjdk-25）仅供
#     二者运行，交互默认 java 仍为第 7 节 sdkman JDK8——sdkman PATH 前置，
#     apt 的 update-alternatives 不影响 sdkman 路径）
#   chromium 栈 — playwright 浏览器；xvfb 供有头场景 xvfb-run 兜底
# 字典集为技能硬承诺（system 技能 + auth-brute 手册引用 /usr/share/seclists、
# /usr/share/wordlists/rockyou.txt.gz、/usr/share/dirb/wordlists），必须打包。
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates curl wget git sudo less vim-tiny file \
      procps iproute2 iputils-ping tmux \
      bsdextrautils npm jq yq \
      openssh-client sshpass netcat-openbsd socat tcpdump tshark dnsutils whois proxychains4 \
      nmap masscan nuclei naabu sqlmap gobuster ffuf feroxbuster \
      seclists dirb dnsrecon wordlists \
      hydra john hashid fcrackzip gitleaks \
      python3-impacket evil-winrm netexec smbclient enum4linux-ng bloodyad coercer \
      chisel-common-binaries default-mysql-client \
      gdb gdb-multiarch strace ltrace binutils xxd binwalk foremost steghide exiftool \
      radare2 yara jadx apktool \
      unzip p7zip-full unrar-free xz-utils patchelf \
      qemu-user qemu-user-static libc6-dbg libc6-dev \
      python3 python3-pip python3-venv python3-paramiko \
      gcc \
      ruby \
      chromium xvfb libgtk-3-0 libgbm1 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ---------- 2b. jadx JVM 绑定（apt 路线配套）----------
# jadx 1.5.6 类文件为 Java 11+（class 55），而交互默认 java 是第 7 节 sdkman JDK8
# （class 52）——PATH 命中即崩（UnsupportedClassVersionError）。与 tsec-hosted
# 「每工具显式 JAVA_HOME」同型：/usr/local/bin wrapper 把 jadx 固定到 apt 依赖带入的
# JVM25，默认 java 仍为 JDK8；apktool 2.7 类文件兼容 Java 8，无需绑定。
RUN set -eux; \
    jvm_home="$(dirname "$(dirname "$(readlink -f /usr/bin/java)")")"; \
    printf '%s\n' '#!/bin/sh' \
      '# jadx 类文件为 Java 11+：显式走 apt JVM（交互默认 java 仍为 sdkman JDK8）' \
      "export JAVA_HOME=${jvm_home}" \
      'export PATH="${JAVA_HOME}/bin:${PATH}"' \
      'exec /usr/bin/jadx "$@"' > /usr/local/bin/jadx; \
    chmod 0755 /usr/local/bin/jadx; \
    /usr/local/bin/jadx --version

# ---------- 3. pip / gem（Pwn + CTF 必备；审计类 semgrep 在第 5 层以 pip --user 安装）----------
# Python 工具优先取 Kali apt 发行版包（python3-pwntools=4.15 而非 pip 在 Python 3.14 下
# 回溯出的 2019 年 3.12 源码包；apt 依赖闭包自动带上 capstone 等）。
# ropper 不装：其依赖 filebytes 的 setup.py 使用 Python 3.12+ 移除的 ast.Str，
# 在本底座不可构建且 Kali 无 apt 包；ROPgadget + pwntools ROP 已覆盖 gadget 需求。
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      python3-pwntools python3-ropgadget python3-z3 \
      python3-pycryptodome python3-requests python3-bs4 && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    gem install one_gadget --no-document

# ---------- 4. 用户 + 目录占位 ----------
# 高安全档（docker_profile=security）以镜像默认用户运行，kali(uid 1000) + sudo 免密必须保留。
# /opt 各目录为运行时挂载点：tools/kb/workspace/deepstrike/traffic 由平台 -v 注入
#（kb/tools/deepstrike 只读，workspace/traffic/日志/任务目录见 worker isolatedWorkspaceVolumes），
# 镜像内不装任何工具；logs/ 仅建目录供日志卷挂载。
RUN useradd -m -s /bin/bash kali && \
    echo 'kali ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
    mkdir -p /opt/tools /opt/kb /opt/workspace /opt/deepstrike /opt/traffic \
             /opt/mirrorstrike/pi-agent/agents /opt/mirrorstrike/logs/agent \
             /home/kali/.config/nuclei && \
    chown -R kali:kali /opt/tools /opt/kb /opt/workspace /opt/deepstrike /opt/traffic \
                       /opt/mirrorstrike /home/kali/.config

USER kali

# ---------- 5. semgrep（代码审计 SAST 兜底，技能承诺 ~/.local/bin/semgrep）----------
RUN pip3 install --user semgrep

# pip --user 工具落在 ~/.local/bin，加入 PATH 供 agent 调用
ENV PATH=/home/kali/.local/bin:${PATH}

# ---------- 6. AI 执行器：pi-coding-agent + playwright 浏览器 ----------
# pi 扩展 extensions/mirrorstrike-mcp.ts 裸导入 @modelcontextprotocol/sdk，全局包为解析兜底；
# 运行时 /opt/mirrorstrike/pi-agent 由宿主机只读挂载（可能不含 node_modules），
# 镜像侧提供 /opt/mirrorstrike/node_modules 软链（指向全局包目录）作为 ESM 解析兜底。
# playwright 走 apt chromium：PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 不下载浏览器，
# 平台 mcp/mcp.json 以 --executable-path /usr/bin/chromium 接线，无头运行。
RUN sudo npm install -g \
      @earendil-works/pi-coding-agent \
      @modelcontextprotocol/sdk \
      playwright \
      @playwright/mcp && \
    sudo ln -sfn "$(npm root -g)" /opt/mirrorstrike/node_modules

# Node 版本门（pi-coding-agent 要求 >= 22.19）+ pi CLI 冒烟
RUN node -e 'const [maj, min] = process.versions.node.split(".").map(Number); if (maj < 22 || (maj === 22 && min < 19)) { console.error("[fatal] pi-coding-agent 需要 Node >= 22.19，当前 " + process.versions.node); process.exit(1); }' && \
    pi --version >/dev/null || { echo "[fatal] pi 不可用"; exit 1; }

# ---------- 7. SDKMAN JDK8（单 JDK：反序列化利用链对新 JDK 兼容差；单引号 + ${JDK8} 文本替换避免 $HOME 被误展开）----------
RUN curl -s "https://get.sdkman.io" | bash && \
    bash -lc 'source "$HOME/.sdkman/bin/sdkman-init.sh" && \
              sdk install java ${JDK8} && \
              sdk default java ${JDK8}' && \
    rm -rf "$HOME/.sdkman/archives"/* "$HOME/.sdkman/tmp"/*
# 默认 JDK8：设 ENV 让非交互 shell / agent 调用也走 JDK8（sdkman 仅在交互 ~/.bashrc 生效）
ENV JAVA_HOME=/home/kali/.sdkman/candidates/java/current
ENV PATH=/home/kali/.sdkman/candidates/java/current/bin:${PATH}

# ---------- 8. nuclei 离线模板指向挂载路径（禁联网更新；模板由 /opt/kb 运行时挂载）----------
RUN printf '%s\n' 'disable-update-check: true' 'update-template-dir: /opt/kb/nuclei-templates' \
      > /home/kali/.config/nuclei/config.yaml

# ---------- 9. shell 提示 + gdb 自动加载 + git 身份 + 登录 shell 的 java PATH ----------
# Kali 的 ~/.profile 不 source ~/.bashrc，登录 shell（bash -l）会被 /etc/profile 重写 PATH
# 丢掉 sdkman java——补 source sdkman-init，使 bash -lc 下 java 同样可用
RUN printf '%s\n' 'PS1="${PS1%\\n*} \[\033[0;36m\]\D{%y-%m-%d} - \t\[\033[;32m\]\n${PS1##*\\n}"' >> /home/kali/.bashrc && \
    echo 'set auto-load safe-path /' >> /home/kali/.gdbinit && \
    echo 'source "$HOME/.sdkman/bin/sdkman-init.sh"' >> /home/kali/.profile && \
    git config --global user.email "agent@mirrorstrike" && \
    git config --global user.name "mirrorstrike-agent"

# 默认工作目录保持中立：Agent 进程的实际 cwd 由 wrapper 设置（pi → /opt/mirrorstrike/pi-agent），
# WORKDIR 仅影响 docker run/exec 的默认 shell 位置。
# 无 ENTRYPOINT：平台 Worker 统一以 --entrypoint tail -f /dev/null 常驻容器，
# Agent 进程经 docker exec 拉起；有头浏览器等 GUI 需求用 xvfb-run 手动起虚拟显示。
WORKDIR /opt/mirrorstrike
CMD ["/bin/bash"]
