# =============================================================================
# Dockerfile — mirrorstrike agent 执行容器（pi-only 精简版）
#
# 四场景（渗透/CTF/代码审计/Pwn）显式最小清单，弃用 kali-linux-headless 元包
#（无 metasploit）。关键约定详见 AGENTS.md，要点：
#   - 零 GitHub 下载工具；ysoserial/JNDIExploit 等敏感工具与 nuclei 模板经
#     运行时挂载提供（/opt/tools、/opt/kb，:ro），挂载点仅建占位目录
#   - AI 执行器仅 pi-coding-agent（+ MCP SDK 解析兜底）
#   - 交互 java 仅 JDK8（sdkman zulu）；jadx 经 2b 节 wrapper 绑定 apt JVM
#   - 零预装字典（平台 dicts/ 承担）；保留 kali 用户 + sudo 免密
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
# 清单基线取 Dockerfile.tsec-hosted，差异项为平台承诺或 agent 刚需（逐项对照见
# AGENTS.md「预装工具清单」表）。零预装字典：seclists/dirb/wordlists 已移除，
# 由平台 dicts/ 能力承担；dnsrecon 保留（DNS 枚举工具本体）。
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates curl wget git sudo less vim-tiny file \
      procps iproute2 iputils-ping tmux \
      bsdextrautils npm jq yq \
      openssh-client sshpass netcat-openbsd socat tcpdump tshark dnsutils whois proxychains4 \
      nmap masscan nuclei naabu sqlmap gobuster ffuf feroxbuster \
      dnsrecon \
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

# ---------- 2b. jadx JVM 绑定 ----------
# jadx 类文件为 Java 11+，交互默认 java 是第 7 节 sdkman JDK8（PATH 命中即崩
# UnsupportedClassVersionError）：wrapper 把 jadx 固定到 apt 依赖带入的 JVM
#（Kali default-jre），默认 java 仍为 JDK8；apktool 2.7 兼容 Java 8，无需绑定。
RUN set -eux; \
    jvm_home="$(dirname "$(dirname "$(readlink -f /usr/bin/java)")")"; \
    printf '%s\n' '#!/bin/sh' \
      '# jadx 类文件为 Java 11+：显式走 apt JVM（交互默认 java 仍为 sdkman JDK8）' \
      "export JAVA_HOME=${jvm_home}" \
      'export PATH="${JAVA_HOME}/bin:${PATH}"' \
      'exec /usr/bin/jadx "$@"' > /usr/local/bin/jadx; \
    chmod 0755 /usr/local/bin/jadx; \
    /usr/local/bin/jadx --version

# ---------- 3. pip / gem（Pwn + CTF 必备；semgrep 在第 5 层 pip --user 安装）----------
# Python 工具优先取 Kali apt 发行版包（避免 pip 在新 Python 下回溯老版本源码包；
# apt 依赖闭包自动带上 capstone 等）。
# ropper 不装：依赖 filebytes 用到 Python 3.12+ 已移除的 ast.Str，本底座不可构建；
# ROPgadget + pwntools ROP 已覆盖。
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      python3-pwntools python3-ropgadget python3-z3 \
      python3-pycryptodome python3-requests python3-bs4 && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    gem install one_gadget --no-document

# ---------- 4. 用户 + 目录占位 ----------
# kali(uid 1000) + sudo 免密：高安全档（docker_profile=security）以镜像默认用户运行。
# /opt 各目录为运行时挂载点（tools/kb/deepstrike 只读），镜像内不装任何工具。
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
# @modelcontextprotocol/sdk 为 pi 扩展裸导入的解析兜底；/opt/mirrorstrike/node_modules
# 软链到全局包目录，供运行时只读挂载的 pi-agent（可能不含 node_modules）做 ESM 解析。
# playwright 走 apt chromium（PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 不下载浏览器），
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

# ---------- 7. SDKMAN JDK8（单 JDK：反序列化利用链对新 JDK 兼容差）----------
# curl 落盘 + fail-fast：管道模式（curl | bash）会静默吞掉失败导致 sdkman-init 缺失。
RUN set -eux; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors \
      "https://get.sdkman.io" -o /tmp/sdkman-install.sh; \
    bash /tmp/sdkman-install.sh; \
    bash -lc 'source "$HOME/.sdkman/bin/sdkman-init.sh" && \
              sdk install java ${JDK8} && \
              sdk default java ${JDK8}'; \
    rm -rf "$HOME/.sdkman/archives"/* "$HOME/.sdkman/tmp"/* /tmp/sdkman-install.sh
# 默认 JDK8：设 ENV 让非交互 shell / agent 调用也走 JDK8（sdkman 仅在交互 ~/.bashrc 生效）
ENV JAVA_HOME=/home/kali/.sdkman/candidates/java/current
ENV PATH=/home/kali/.sdkman/candidates/java/current/bin:${PATH}

# ---------- 8. nuclei 离线模板指向挂载路径（禁联网更新；模板由 /opt/kb 运行时挂载）----------
RUN printf '%s\n' 'disable-update-check: true' 'update-template-dir: /opt/kb/nuclei-templates' \
      > /home/kali/.config/nuclei/config.yaml

# ---------- 9. shell 提示 + gdb 自动加载 + git 身份 + 登录 shell 的 java PATH ----------
# 登录 shell（bash -l）被 /etc/profile 重写 PATH 会丢 sdkman java——.profile 补 source。
RUN printf '%s\n' 'PS1="${PS1%\\n*} \[\033[0;36m\]\D{%y-%m-%d} - \t\[\033[;32m\]\n${PS1##*\\n}"' >> /home/kali/.bashrc && \
    echo 'set auto-load safe-path /' >> /home/kali/.gdbinit && \
    echo 'source "$HOME/.sdkman/bin/sdkman-init.sh"' >> /home/kali/.profile && \
    git config --global user.email "agent@mirrorstrike" && \
    git config --global user.name "mirrorstrike-agent"

# 无 ENTRYPOINT：平台 Worker 以 --entrypoint tail -f /dev/null 常驻容器、docker exec
# 拉起 Agent；WORKDIR 仅影响默认 shell 位置，Agent cwd 由 wrapper 设置为 pi-agent。
WORKDIR /opt/mirrorstrike
CMD ["/bin/bash"]
