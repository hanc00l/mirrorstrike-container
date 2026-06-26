# syntax=docker/dockerfile:1.7
FROM --platform=amd64 kalilinux/kali-rolling:latest

# ---------- 版本号集中管理（改版本只动这里）----------
ARG KATANA_VERSION=1.5.0
ARG NUCLEI_VERSION=3.7.1
ARG DALFOX_VERSION=2.12.0
ARG CLOUDFOX_VERSION=2.0.1
ARG KERBRUTE_TAG=v1.0.3
ARG TRIVY_VERSION=0.71.2
ARG CHROME_VERSION=149.0.7827.54
ARG JDK21=21.0.11.fx-zulu
ARG JDK8=8.0.482.fx-zulu

ENV TZ=Asia/Shanghai \
    PYTHONUNBUFFERED=1 \
    PIP_BREAK_SYSTEM_PACKAGES=1 \
    PIP_NO_CACHE_DIR=1 \
    CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1 \
    PLAYWRIGHT_MCP_BROWSER=chromium

# ---------- 1. base（固定 apt 源到 kali.download，避免 mirrorbrain 重定向到失效 mirror 如 neusoft；root 装 headless 元包）----------
RUN echo "deb http://kali.download/kali kali-rolling main contrib non-free non-free-firmware" > /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y kali-linux-headless && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ---------- 2. 用户 ----------
RUN useradd -m -s /bin/bash kali && \
    echo 'kali ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# ---------- 3. apt 工具大合并（渗透 + CTF + Pwn + 审计）----------
RUN sudo apt-get update && \
    sudo apt-get install -y \
      bsdextrautils npm jq iputils-ping sshpass ncat rlwrap \
      bloodyad coercer enum4linux-ng pwncat chisel-common-binaries \
      dirsearch yq krb5-user gitleaks naabu nikto netexec adb \
      gdb ghidra strace file gdb-multiarch patchelf qemu-user qemu-user-static \
      binwalk foremost steghide libc6-dbg libc6-dev \
      ruby ruby-dev \
      sqlmap hydra proxychains4 evil-winrm python3-impacket \
      xvfb libgtk-3-0 libgbm1 && \
    sudo apt-get clean && \
    sudo rm -rf /var/lib/apt/lists/*

# ---------- 4. 目录（/opt/tools、/opt/workspace 为运行时挂载点，仅占位；内置工具归 /home/kali/tools）----------
RUN sudo mkdir -p /opt/tools /opt/workspace \
                 /opt/mirrorstrike/claude-code /opt/mirrorstrike/logs/agent \
                 /home/kali/tools /home/kali/workspace && \
    sudo chown -R kali:kali /opt/tools /opt/workspace /opt/mirrorstrike \
                              /home/kali/tools /home/kali/workspace

USER kali

# ---------- 5. pip 工具合并（已删 angr；含审计 / CTF / Pwn）----------
RUN pip3 install --user \
      pwntools pymongo pycryptodome requests beautifulsoup4 \
      semgrep ROPgadget ropper z3-solver && \
    sudo gem install one_gadget --no-document

# pip --user 工具（semgrep / ROPgadget / ropper 等）落在 ~/.local/bin，加入 PATH 供 agent 调用
ENV PATH=/home/kali/.local/bin:${PATH}

# ---------- 6. GitHub release 二进制 + nuclei 模板 ----------
RUN mkdir -p /tmp/dl && cd /tmp/dl && \
    wget -q "https://github.com/projectdiscovery/katana/releases/download/v${KATANA_VERSION}/katana_${KATANA_VERSION}_linux_amd64.zip" -O katana.zip && \
    wget -q "https://github.com/projectdiscovery/nuclei/releases/download/v${NUCLEI_VERSION}/nuclei_${NUCLEI_VERSION}_linux_amd64.zip" -O nuclei.zip && \
    wget -q "https://github.com/hahwul/dalfox/releases/download/v${DALFOX_VERSION}/dalfox-linux-amd64.tar.gz" -O dalfox.tar.gz && \
    wget -q "https://github.com/BishopFox/cloudfox/releases/download/v${CLOUDFOX_VERSION}/cloudfox-linux-amd64.zip" -O cloudfox.zip && \
    wget -q "https://github.com/ropnop/kerbrute/releases/download/${KERBRUTE_TAG}/kerbrute_linux_amd64" -O kerbrute && \
    wget -q "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz" -O trivy.tar.gz && \
    unzip -q katana.zip katana && \
    unzip -q nuclei.zip nuclei && \
    tar -xzf dalfox.tar.gz && \
    unzip -q cloudfox.zip cloudfox/cloudfox && \
    tar -xzf trivy.tar.gz && \
    chmod +x katana nuclei dalfox-linux-amd64 cloudfox/cloudfox kerbrute trivy && \
    sudo mv katana nuclei cloudfox/cloudfox kerbrute trivy /usr/local/bin/ && \
    sudo mv dalfox-linux-amd64 /usr/local/bin/dalfox && \
    git clone --depth 1 https://github.com/projectdiscovery/nuclei-templates.git /home/kali/.local/nuclei-templates && \
    sudo rm -rf /tmp/*

# ---------- 7. nuclei 配置（heredoc；不再重复 -ut）----------
RUN mkdir -p /home/kali/.config/nuclei && \
    cat > /home/kali/.config/nuclei/config.yaml <<'EOF'
disable-update-check: true
update-template-dir: /home/kali/.local/nuclei-templates
EOF

# ---------- 8. Java / 反序列化工具（内置源码工具归 /home/kali/tools，不碰挂载点 /opt/tools）----------
RUN cd /home/kali/tools && \
    wget -q "https://github.com/frohoff/ysoserial/releases/download/v0.0.6/ysoserial-all.jar" -O ysoserial.jar && \
    git clone --depth 1 https://github.com/ticarpi/jwt_tool.git && \
    pip3 install --user -r jwt_tool/requirements.txt && \
    git clone --depth 1 https://github.com/IOActive/jdwp-shellifier.git

# ---------- 9. AI 代理 + Playwright（--with-deps 自动按发行版装浏览器系统依赖，规避 libasound2/t64 包名问题）----------
RUN sudo npm install -g @anthropic-ai/claude-code playwright && \
    playwright install --with-deps chromium

# ---------- 10. SDKMAN JDK 8 / 21，默认 JDK8（修复 sdk 在子 shell 失效；单引号 + ${JDK*} 文本替换避免 $HOME 被误展开）----------
RUN curl -s "https://get.sdkman.io" | bash && \
    bash -lc 'source "$HOME/.sdkman/bin/sdkman-init.sh" && \
              sdk install java ${JDK21} && \
              sdk install java ${JDK8} && \
              sdk default java ${JDK8}' && \
    rm -rf "$HOME/.sdkman/archives"/* "$HOME/.sdkman/tmp"/*
# 默认 JDK8（sdkman current → 8.0.482）：设 ENV 让非交互 shell / agent 调用也走 JDK8（sdkman 仅在交互 ~/.bashrc 生效）
ENV JAVA_HOME=/home/kali/.sdkman/candidates/java/current
ENV PATH=/home/kali/.sdkman/candidates/java/current/bin:${PATH}
# Ghidra(12.1.1) 需 JDK21；默认仍 JDK8，Ghidra 单独走 JDK21
ENV GHIDRA_JDK_HOME=/home/kali/.sdkman/candidates/java/21.0.11.fx-zulu

# ---------- 11. pwndbg（替换 GEF；固定路径，setup.sh 会把 source $PWD/gdbinit.py 写进 ~/.gdbinit）----------
RUN git clone --depth 1 https://github.com/pwndbg/pwndbg.git /home/kali/tools/pwndbg && \
    bash -c 'cd /home/kali/tools/pwndbg && ./setup.sh'

# ---------- 12. agent-browser + Chrome for Testing（修正环境变量名：CHROME_PATH → EXECUTABLE_PATH）----------
RUN sudo npm install -g agent-browser && \
    sudo mkdir -p /opt/chrome && cd /opt/chrome && \
    sudo wget -q --no-proxy --timeout=60 --tries=3 "https://registry.npmmirror.com/-/binary/chrome-for-testing/${CHROME_VERSION}/linux64/chrome-linux64.zip" && \
    sudo unzip -q chrome-linux64.zip && sudo rm chrome-linux64.zip && \
    sudo chmod -R 755 /opt/chrome
ENV AGENT_BROWSER_EXECUTABLE_PATH=/opt/chrome/chrome-linux64/chrome \
    DISPLAY=:99

# ---------- 13. agent-browser skill（非标准 CLI，失败不阻断构建）----------
RUN npx skills add vercel-labs/agent-browser --skill agent-browser -g -y \
    || echo "[warn] agent-browser skill 不可用/失败，已跳过（不影响 agent-browser 本体）"

# ---------- 14. shell 提示 + 默认工作区 git 配置 ----------
RUN printf '%s\n' 'PS1="${PS1%\\n*} \[\033[0;36m\]\D{%y-%m-%d} - \t\[\033[;32m\]\n${PS1##*\\n}"' >> /home/kali/.bashrc && \
    echo 'set auto-load safe-path /' >> /home/kali/.gdbinit && \
    git -C /home/kali/workspace init && \
    git -C /home/kali/workspace config user.email "agent@mirrorstrike" && \
    git -C /home/kali/workspace config user.name "mirrorstrike-agent"

# ---------- 15. 清理 + entrypoint（常驻 Xvfb，解决 DISPLAY=:99 无服务）----------
RUN sudo rm -rf /usr/lib/llvm-*/build 2>/dev/null || true
RUN sudo tee /usr/local/bin/docker-entrypoint.sh > /dev/null <<'EOF'
#!/bin/sh
if command -v Xvfb >/dev/null 2>&1 && [ -n "$DISPLAY" ]; then
  Xvfb "$DISPLAY" -screen 0 1280x800x24 -ac >/tmp/xvfb.log 2>&1 &
fi
exec "$@"
EOF
RUN sudo chmod +x /usr/local/bin/docker-entrypoint.sh

WORKDIR /opt/mirrorstrike/claude-code
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
CMD ["/bin/bash"]
