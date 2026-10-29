# =============================================================================
# Dockerfile — mirrorstrike agent 执行容器（完整版，四场景）
#
# 2026-10-16 重构：以 Dockerfile.tsec-hosted（托管测试镜像）工具面为基线做超集收敛——
# tsec-hosted 有的全包含，另保留完整版四场景刚需（代码审计 semgrep、跨架构 Pwn qemu 系）。
# 差异裁决（逐项对照见 AGENTS.md「预装工具清单」表）：
#   - 无 server 端组件：不装 postgresql/redis-server/tini（平台侧提供）；
#     数据库客户端完整直装：mysql=default-mysql-client、psql=postgresql-client、
#     redis-cli=redis-tools（tsec-hosted 由 server 包传递带入，此处显式装客户端包）
#   - 无 metasploit / hashcat / nikto / dirsearch；零预装字典（平台 dicts/ 承担）
#   - pinned 资产例外与 tsec-hosted 对齐：chisel/grpcurl/jadx/apktool/capa/Temurin JDK21
#     均为 URL+SHA256 锁定（升级须同步更新哈希），capa-rules 随镜像、禁联网更新
#   - /opt/tools、/opt/kb 为运行时挂载点（:ro），镜像内不可装物 → android 栈落
#     /opt/android、capa 落 /opt/capa（tsec-hosted 落 /opt/tools/*，系构建上下文差异）
#   - AI 执行器仅 pi-coding-agent（版本锁定 + 漂移断言）
#   - 交互 java 仅 JDK8（sdkman zulu，fx 变体安装后剔除 JavaFX）；jadx/apktool 经
#     wrapper 绑定 /opt/android/jre（jlink 裁剪 JRE21），与 JDK8 互不污染
#   - 保留 kali 用户 + sudo 免密；无 ENTRYPOINT（平台 Worker 覆盖 tail -f /dev/null）
#
# 注：不写 `# syntax=` 指令——该指令会让 buildx 去 Docker Hub 拉 frontend 镜像，
# 离线/受限环境常不可达；本文件只用到内置 frontend 支持的特性。
# =============================================================================
FROM --platform=amd64 kalilinux/kali-rolling:latest

# ---------- 版本号集中管理（改版本只动这里）----------
ARG JDK8=8.0.482.fx-zulu
# pi 执行器版本锁定：与 MS 扩展的 pi 1.1.x 基线对齐；裸装会随 npm latest 漂移
ARG PI_VERSION=1.4.0
# 浏览器层（chromium + playwright npm）默认开；--build-arg WITH_PLAYWRIGHT=0 关闭
ARG WITH_PLAYWRIGHT=1
# 云 CLI 族（awscli/trivy）：d 类红区题面拍板后再开
ARG WITH_CLOUD=0
# exploitdb（searchsploit，~60MB）默认开；--build-arg WITH_EXPLOITDB=0 关闭
ARG WITH_EXPLOITDB=1
# kubectl（~55MB，dict-base 清单硬承诺）默认开
ARG WITH_KUBECTL=1

# pinned 增补资产（URL+SHA256 构建期校验，与 Dockerfile.tsec-hosted 同源同值，升级须同步更新）
ARG CHISEL_URL=https://github.com/jpillora/chisel/releases/download/v1.10.1/chisel_1.10.1_linux_amd64.deb
ARG CHISEL_SHA256=0c7e4abb892faca44dbc9b7363af27e3859b68492e10771fea446fd5e8535e5d
ARG GRPCURL_URL=https://github.com/fullstorydev/grpcurl/releases/download/v1.9.3/grpcurl_1.9.3_linux_amd64.deb
ARG GRPCURL_SHA256=0710dc05a28f1fac2230fd438c1fc61b90e0431809372a65aea4b53a57947451

# Android 逆向 pinned 资产（与 mirrorstrike deploy/reverse-static/tool-lock.json 一致）
ARG ANDROID_JDK_URL=https://github.com/adoptium/temurin21-binaries/releases/download/jdk-21.0.12.1%2B1/OpenJDK21U-jdk_x64_linux_hotspot_21.0.12.1_1.tar.gz
ARG ANDROID_JDK_SHA256=ce79869e1307ed8ee1e2baa86a412b1eb5b75d10a01006d788a6f968bcfaee94
ARG JADX_URL=https://github.com/skylot/jadx/releases/download/v1.5.6/jadx-1.5.6.zip
ARG JADX_SHA256=545ea2be9c242511bc145755cf4bda2485ade42966e096f8b4d3da2a230e8974
ARG APKTOOL_URL=https://github.com/iBotPeaches/Apktool/releases/download/v3.0.2/apktool_3.0.2.jar
ARG APKTOOL_SHA256=eee4669a704a14e0623407e6701b0b91887e61e1e4049cb7a82833e14ae8b5fd
ARG CAPA_URL=https://github.com/mandiant/capa/releases/download/v9.4.0/capa-v9.4.0-linux.zip
ARG CAPA_SHA256=07800a1d20a21eb18fc98716e2ae81b668e0c9a04defd588c8aa17ea3d3281e4

ENV TZ=Asia/Shanghai \
    PYTHONUNBUFFERED=1 \
    PIP_BREAK_SYSTEM_PACKAGES=1 \
    PIP_NO_CACHE_DIR=1 \
    PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    PLAYWRIGHT_MCP_BROWSER=chromium \
    NUCLEI_TEMPLATES_DIR=/opt/kb/nuclei-templates

# ---------- 1. base（固定 apt 源到 kali.download，避免 mirrorbrain 重定向到失效 mirror 如 neusoft）----------
RUN echo "deb http://kali.download/kali kali-rolling main contrib non-free non-free-firmware" > /etc/apt/sources.list

# ---------- 2. apt 工具大合并（渗透 + CTF + 审计 + Pwn + 逆向，显式最小清单）----------
# 清单 = Dockerfile.tsec-hosted 全量 ∪ 完整版四场景刚需。要点：
#   - 无 server 端组件（postgresql/redis-server/tini 不装）；数据库客户端完整直装：
#     mysql=default-mysql-client、psql=postgresql-client、redis-cli=redis-tools
#   - 无 metasploit/hashcat/nikto/dirsearch；chisel 走 §2b pinned deb（不用
#     chisel-common-binaries）；jadx/apktool 不走 apt 版（JVM25 依赖链），见 §4b
#   - 保留完整版 Pwn 承诺：qemu-user/-static、libc6-dbg（tsec-hosted 有意排除，此处保留）
#   - 复核待办收口：psmisc（killall/pstree）、python3-setuptools（pip 兜底链需要）
#   - 零预装字典（平台 dicts/ 承担）；dnsrecon 保留（DNS 枚举工具本体）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates curl wget git sudo less vim-tiny file \
      procps psmisc iproute2 iputils-ping iputils-arping tmux rlwrap libcap2-bin \
      bsdextrautils npm jq yq bc sqlite3 ripgrep fd-find php-cli python3-setuptools \
      openssh-client sshpass netcat-openbsd ncat socat tcpdump tshark dnsutils whois proxychains4 \
      nmap masscan nuclei naabu sqlmap gobuster ffuf feroxbuster \
      dnsrecon wafw00f whatweb \
      hydra john hashid fcrackzip gitleaks \
      python3-impacket evil-winrm netexec smbclient enum4linux-ng bloodyad coercer \
      ldap-utils \
      default-mysql-client postgresql-client redis-tools \
      gdb gdb-multiarch strace ltrace binutils xxd radare2 yara patchelf checksec make \
      gcc libc6-dev libc6-dbg qemu-user qemu-user-static \
      binwalk foremost steghide exiftool unzip zip p7zip-full unrar-free xz-utils \
      tesseract-ocr tesseract-ocr-chi-sim \
      adb apksigner zipalign aapt \
      python3 python3-pip python3-venv python3-paramiko \
      python3-numpy python3-sympy python3-pil python3-lxml \
      python3-jwt python3-pymongo python3-psycopg2 python3-pymysql \
      ruby && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# pg8000（dict-base 清单）无 apt 包：pip 单包补齐，失败不阻断（非核心依赖）
RUN pip3 install --no-cache-dir pg8000 || echo "[warn] pg8000 pip 安装失败，非核心依赖，继续构建"

# ---------- 2b. pinned 单架构工具：chisel + grpcurl（替换 chisel-common-binaries，-~110MB）----------
RUN set -eux; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors -o /tmp/chisel.deb "${CHISEL_URL}"; \
    echo "${CHISEL_SHA256}  /tmp/chisel.deb" | sha256sum -c -; \
    apt-get install -y --no-install-recommends /tmp/chisel.deb; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors -o /tmp/grpcurl.deb "${GRPCURL_URL}"; \
    echo "${GRPCURL_SHA256}  /tmp/grpcurl.deb" | sha256sum -c -; \
    apt-get install -y --no-install-recommends /tmp/grpcurl.deb; \
    rm -f /tmp/chisel.deb /tmp/grpcurl.deb; \
    chisel --version; grpcurl --version

# ---------- 2b+. 增补兼容软链：Debian 系 fd-find 二进制名为 fdfind，补 fd 别名 ----------
RUN command -v fdfind >/dev/null 2>&1 && ln -sf "$(command -v fdfind)" /usr/local/bin/fd; \
    command -v rg >/dev/null 2>&1; command -v fd >/dev/null 2>&1

# ---------- 2b*. nmap 入口修复（与 tsec-hosted 行为对齐；两档皆稳）----------
# Kali 打包的 nmap 真实二进制带 file caps（含 cap_net_admin），非 root 无 effective
# CAP_NET_RAW 且 cap_net_admin 超出 Docker 默认 bounding set → exec EPERM；Kali wrapper
# 又对非 root 传 --privileged，关闭了 nmap 的自动降级。修复：
#   - setcap 削掉 cap_net_admin（保留 raw+bind_service），exec 子集校验恒通过；
#   - /usr/local/bin 覆盖入口（PATH 优先于 /usr/bin）：root 沿用原逻辑（平台常规
#     capability 档 root+CAP ALL，file caps 削减不影响全功能）；非 root 强制
#     --unprivileged 自动降级 connect 扫描，-sS 给出明确的 requires-root 语义。
RUN setcap cap_net_raw,cap_net_bind_service=eip /usr/lib/nmap/nmap && \
    printf '%s\n' '#!/usr/bin/env sh' 'set -e' \
      '# nmap 入口：root 沿用原逻辑；非 root 强制 --unprivileged 自动降级 connect 扫描' \
      'if [ "$(id -u)" -eq 0 ]; then exec /usr/lib/nmap/nmap "$@"; fi' \
      'exec /usr/lib/nmap/nmap --unprivileged "$@"' > /usr/local/bin/nmap && \
    chmod 0755 /usr/local/bin/nmap && \
    nmap --version | head -1

# ---------- 2c. 可选层：exploitdb（searchsploit）/ 云 CLI 族 / kubectl ----------
RUN set -eux; \
    if [ "$WITH_EXPLOITDB" = "1" ]; then \
      apt-get update && apt-get install -y --no-install-recommends exploitdb && \
      apt-get clean && rm -rf /var/lib/apt/lists/*; \
    fi; \
    if [ "$WITH_CLOUD" = "1" ]; then \
      apt-get update && apt-get install -y --no-install-recommends awscli trivy && \
      apt-get clean && rm -rf /var/lib/apt/lists/*; \
    fi; \
    if [ "$WITH_KUBECTL" = "1" ]; then \
      apt-get update && apt-get install -y --no-install-recommends kubectl && \
      apt-get clean && rm -rf /var/lib/apt/lists/*; \
    fi; \
    command -v kubectl || echo "[warn] kubectl 未启用（WITH_KUBECTL=0）"; \
    kubectl version --client=true 2>/dev/null | head -1 || true

# ---------- 3. pip / gem（Pwn + CTF 必备）----------
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
# /opt/tools、/opt/kb、/opt/deepstrike 为运行时挂载点（只读），镜像内不装任何工具；
# fscan/gsl5-cli/mdut-cli/ysoserial/JNDIExploit/suo5/frp 等经挂载提供（tsec-hosted 为
# 构建期 COPY，本镜像系运行时挂载形态，工具面等价）。
RUN useradd -m -s /bin/bash kali && \
    echo 'kali ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers && \
    mkdir -p /opt/tools /opt/kb /opt/workspace /opt/deepstrike /opt/traffic \
             /opt/mirrorstrike/pi-agent/agents /opt/mirrorstrike/logs/agent \
             /home/kali/.config/nuclei && \
    chown -R kali:kali /opt/tools /opt/kb /opt/workspace /opt/deepstrike /opt/traffic \
                       /opt/mirrorstrike /home/kali/.config

# ---------- 4b. Android 逆向层：jadx + apktool + jlink 裁剪 JRE（pinned，与 tsec-hosted 同版）----------
# jadx/apktool 要求 JDK>=11；默认 JDK8（反射链刚需）不动，wrapper 显式指向
# /opt/android/jre（注意：不能落 /opt/tools——那是运行时 :ro 挂载点，会被遮蔽），
# 双 JAVA_HOME 互不污染。JRE 由 jlink 按 jdeps 依赖裁剪（失败回退固定模块集）。
RUN set -eux; \
    mkdir -p /tmp/android-build /opt/android; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors -o /tmp/jdk21.tar.gz "${ANDROID_JDK_URL}"; \
    echo "${ANDROID_JDK_SHA256}  /tmp/jdk21.tar.gz" | sha256sum -c -; \
    tar -xzf /tmp/jdk21.tar.gz -C /tmp/android-build; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors -o /tmp/jadx.zip "${JADX_URL}"; \
    echo "${JADX_SHA256}  /tmp/jadx.zip" | sha256sum -c -; \
    unzip -q /tmp/jadx.zip -d /opt/android/jadx; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors -o /opt/android/apktool.jar "${APKTOOL_URL}"; \
    echo "${APKTOOL_SHA256}  /opt/android/apktool.jar" | sha256sum -c -; \
    jdk21_dir="$(find /tmp/android-build -maxdepth 1 -type d -name 'jdk-21*' | head -1)"; \
    mods="$("${jdk21_dir}/bin/jdeps" --print-module-deps --ignore-missing-deps \
      /opt/android/jadx/lib/*.jar /opt/android/apktool.jar 2>/dev/null || true)"; \
    if [ -z "${mods}" ]; then \
      mods="java.base,java.logging,java.xml,java.naming,java.desktop,java.management,java.sql,java.net.http,jdk.unsupported,jdk.crypto.ec,jdk.zipfs"; \
    fi; \
    echo "android JRE modules: ${mods}"; \
    "${jdk21_dir}/bin/jlink" --add-modules "${mods}" --strip-debug --no-header-files \
      --compress zip-6 --output /opt/android/jre; \
    rm -rf /tmp/jdk21.tar.gz /tmp/jadx.zip /tmp/android-build; \
    printf '%s\n' '#!/bin/sh' \
      '# android 工具显式指向裁剪 JRE21（默认 java 仍为 JDK8，互不污染）' \
      'export JAVA_HOME=/opt/android/jre' \
      'export PATH="/opt/android/jre/bin:${PATH}"' \
      'exec /opt/android/jadx/bin/jadx "$@"' > /usr/local/bin/jadx; \
    printf '%s\n' '#!/bin/sh' \
      'export JAVA_HOME=/opt/android/jre' \
      'exec /opt/android/jre/bin/java -jar /opt/android/apktool.jar "$@"' > /usr/local/bin/apktool; \
    chmod 0755 /usr/local/bin/jadx /usr/local/bin/apktool; \
    /usr/local/bin/jadx --version; \
    /usr/local/bin/apktool --version

# ---------- 4c. capa：恶意软件能力识别单二进制 + 离线规则（pinned，与 tsec-hosted 同版）----------
# 规则随镜像（禁联网更新，对齐 nuclei 模板处理）；capa 默认在二进制同级 rules/ 找规则。
# 落 /opt/capa（/opt/tools 为运行时挂载点不可装物，见 §4b 注）。
RUN set -eux; \
    mkdir -p /opt/capa; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors -o /tmp/capa.zip "${CAPA_URL}"; \
    echo "${CAPA_SHA256}  /tmp/capa.zip" | sha256sum -c -; \
    unzip -q /tmp/capa.zip -d /tmp/capa-unpack; \
    capa_bin="$(find /tmp/capa-unpack -type f -name capa | head -1)"; \
    install -m 0755 "${capa_bin}" /opt/capa/capa; \
    ln -sf /opt/capa/capa /usr/local/bin/capa; \
    for i in 1 2 3; do \
      if git clone --depth 1 https://github.com/mandiant/capa-rules.git /opt/capa/rules; then break; fi; \
      rm -rf /opt/capa/rules; sleep 5; \
    done; \
    test -d /opt/capa/rules/.git; \
    rm -rf /opt/capa/rules/.git /tmp/capa.zip /tmp/capa-unpack; \
    capa --version

USER kali

# ---------- 5. semgrep（完整版代码审计场景承诺，技能面 ~/.local/bin/semgrep）----------
RUN pip3 install --user semgrep

# pip --user 工具落在 ~/.local/bin，加入 PATH 供 agent 调用
ENV PATH=/home/kali/.local/bin:${PATH}

# ---------- 6. AI 执行器：pi-coding-agent（版本锁定 + 漂移断言）----------
# @modelcontextprotocol/sdk 为 pi 扩展裸导入的解析兜底；/opt/mirrorstrike/node_modules
# 软链到全局包目录，供运行时只读挂载的 pi-agent（可能不含 node_modules）做 ESM 解析。
RUN sudo npm install -g \
      @earendil-works/pi-coding-agent@${PI_VERSION} \
      @modelcontextprotocol/sdk && \
    sudo ln -sfn "$(npm root -g)" /opt/mirrorstrike/node_modules

# Node 版本门（pi-coding-agent 要求 >= 22.19）+ pi CLI 冒烟
RUN node -e 'const [maj, min] = process.versions.node.split(".").map(Number); if (maj < 22 || (maj === 22 && min < 19)) { console.error("[fatal] pi-coding-agent 需要 Node >= 22.19，当前 " + process.versions.node); process.exit(1); }' && \
    pi --version >/dev/null || { echo "[fatal] pi 不可用"; exit 1; }
# 版本锁定不是自证的：断言实际安装版本等于 PI_VERSION（fail-closed）
RUN test "$(pi --version)" = "${PI_VERSION}" || { echo "[fatal] pi 版本漂移: $(pi --version) != ${PI_VERSION}"; exit 1; }

# ---------- 6b. 可选浏览器层（默认开；--build-arg WITH_PLAYWRIGHT=0 关闭）----------
# playwright 走 apt chromium（PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 不下载浏览器），
# 平台 mcp/mcp.json 以 --executable-path /usr/bin/chromium 接线，无头运行。
RUN if [ "$WITH_PLAYWRIGHT" = "1" ]; then \
      sudo apt-get update && \
      sudo apt-get install -y --no-install-recommends chromium xvfb libgtk-3-0 libgbm1 && \
      sudo apt-get clean && sudo rm -rf /var/lib/apt/lists/* && \
      sudo npm install -g playwright @playwright/mcp; \
      # playwright-cli wrapper：固定系统 Chromium 免下载，浏览器自动化统一入口
      printf '%s\n' '#!/bin/sh' \
        '# 浏览器自动化统一入口（dict-base AGENTS.md 对齐）：系统 Chromium，免下载浏览器' \
        'export PLAYWRIGHT_MCP_EXECUTABLE_PATH=/usr/bin/chromium' \
        'exec playwright "$@"' > /tmp/playwright-cli; \
      chmod 0755 /tmp/playwright-cli && sudo mv /tmp/playwright-cli /usr/local/bin/playwright-cli; \
    fi; \
    playwright-cli --version >/dev/null 2>&1 || echo "[warn] 浏览器层未启用（WITH_PLAYWRIGHT=0），playwright-cli 缺席"

# ---------- 7. SDKMAN JDK8（单 JDK：反序列化利用链对新 JDK 兼容差；fx 变体安装后剔除 JavaFX）----------
# zulu8 在 SDKMAN 仅有 fx 整合变体；反射利用链不触 JavaFX，剔除不影响（省 ~100MB）。
# curl 落盘 + fail-fast：管道模式（curl | bash）会静默吞掉失败导致 sdkman-init 缺失。
RUN set -eux; \
    curl -fsSL --retry 5 --retry-delay 3 --retry-all-errors \
      "https://get.sdkman.io" -o /tmp/sdkman-install.sh; \
    bash /tmp/sdkman-install.sh; \
    bash -lc 'source "$HOME/.sdkman/bin/sdkman-init.sh" && \
              sdk install java ${JDK8} && \
              sdk default java ${JDK8}'; \
    find "$HOME/.sdkman/candidates/java/current" \( -iname "*jfx*" -o -iname "javafx*" \) \
      \( -name "*.jar" -o -name "*.so" -o -name "*.diz" \) -delete && \
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
