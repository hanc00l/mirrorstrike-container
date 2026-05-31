FROM --platform=amd64 kalilinux/kali-rolling:latest

RUN echo "deb http://mirrors.ustc.edu.cn/kali kali-rolling main contrib non-free non-free-firmware" > /etc/apt/sources.list && \
    rm -f /etc/apt/sources.list.d/*.sources && \
    apt -o Acquire::ForceIPv4=true update && \
    apt -o Acquire::ForceIPv4=true -y install kali-linux-headless && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash kali && \
    echo "kali ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
USER kali

RUN sudo apt -o Acquire::ForceIPv4=true update && \
    sudo apt -o Acquire::ForceIPv4=true install -y bsdextrautils npm jq iputils-ping sshpass ncat rlwrap \
    bloodyad coercer enum4linux-ng pwncat chisel-common-binaries \
    dirsearch yq krb5-user gitleaks naabu nikto netexec adb && \
    sudo apt clean && \
    sudo rm -rf /var/lib/apt/lists/* && \
    pip3 install pwntools pymongo --break-system-packages && \
    sudo npm install -g @playwright/cli@latest && \
    cd /tmp && playwright-cli install

RUN cd /tmp && \
    (for i in 1 2 3 4 5; do wget -q --timeout=60 "https://github.com/projectdiscovery/katana/releases/download/v1.5.0/katana_1.5.0_linux_amd64.zip" -O katana.zip && exit 0; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do wget -q --timeout=60 "https://github.com/projectdiscovery/nuclei/releases/download/v3.7.1/nuclei_3.7.1_linux_amd64.zip" -O nuclei.zip && exit 0; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do wget -q --timeout=60 "https://github.com/hahwul/dalfox/releases/download/v2.12.0/dalfox-linux-amd64.tar.gz" -O dalfox.tar.gz && exit 0; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do wget -q --timeout=60 "https://github.com/BishopFox/cloudfox/releases/download/v2.0.1/cloudfox-linux-amd64.zip" -O cloudfox.zip && exit 0; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do wget -q --timeout=60 "https://github.com/ropnop/kerbrute/releases/download/v1.0.3/kerbrute_linux_amd64" -O kerbrute && exit 0; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth 1 "https://github.com/projectdiscovery/nuclei-templates.git" /home/kali/.local/nuclei-templates && exit 0; rm -rf /home/kali/.local/nuclei-templates; sleep 5; done; exit 1) && \
    unzip -q katana.zip katana && sudo mv katana /usr/local/bin/ && \
    unzip -q nuclei.zip nuclei && sudo mv nuclei /usr/local/bin/ && \
    tar -xzf dalfox.tar.gz && sudo mv dalfox-linux-amd64 /usr/local/bin/dalfox && \
    unzip -q cloudfox.zip cloudfox/cloudfox && sudo mv cloudfox/cloudfox /usr/local/bin/ && \
    sudo cp kerbrute /usr/local/bin/ && \
    nuclei -ut && \
    sudo rm -rf /tmp/*

RUN mkdir -p /home/kali/knowledges && cd /home/kali/knowledges && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/swisskyrepo/PayloadsAllTheThings.git && exit 0; rm -rf PayloadsAllTheThings; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/swisskyrepo/InternalAllTheThings.git && exit 0; rm -rf InternalAllTheThings; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/HackTricks-wiki/hacktricks.git && exit 0; rm -rf hacktricks; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/HackTricks-wiki/hacktricks-cloud.git && exit 0; rm -rf hacktricks-cloud; sleep 5; done; exit 1)

RUN mkdir -p /home/kali/tools && cd /home/kali/tools && \
    (for i in 1 2 3 4 5; do wget -q --timeout=60 "https://github.com/frohoff/ysoserial/releases/download/v0.0.6/ysoserial-all.jar" -O ysoserial.jar && exit 0; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/ticarpi/jwt_tool.git && exit 0; rm -rf jwt_tool; sleep 5; done; exit 1) && \
    cd jwt_tool && pip3 install -r requirements.txt --break-system-packages && cd .. && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/IOActive/jdwp-shellifier.git && exit 0; rm -rf jdwp-shellifier; sleep 5; done; exit 1)

RUN mkdir -p /home/kali/pocs && cd /home/kali/pocs && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/iSee857/CVE-PoC.git && exit 0; rm -rf CVE-PoC; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/zhzyker/exphub.git && exit 0; rm -rf exphub; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/passwa11/2023Hvv_.git && exit 0; rm -rf 2023Hvv_; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/Threekiii/Awesome-POC.git && exit 0; rm -rf Awesome-POC; sleep 5; done; exit 1) && \
    (for i in 1 2 3 4 5; do git clone --depth=1 https://github.com/vulhub/vulhub.git && exit 0; rm -rf vulhub; sleep 5; done; exit 1)
    
RUN (for i in 1 2 3 4 5; do sudo npm install -g @openai/codex && exit 0; sleep 5; done; exit 1)
RUN (for i in 1 2 3 4 5; do sudo npm install -g @anthropic-ai/claude-code && exit 0; sleep 5; done; exit 1)
RUN (for i in 1 2 3 4 5; do sudo npm install -g @mariozechner/pi-coding-agent && exit 0; sleep 5; done; exit 1)

RUN sudo chmod +x /usr/local/bin/kerbrute
RUN mkdir -p /home/kali/.config/nuclei && \
    echo >> /home/kali/.config/nuclei/config.yaml && \
    echo "disable-update-check: true" >> /home/kali/.config/nuclei/config.yaml && \
    echo "update-template-dir: /home/kali/.local/nuclei-templates" >> /home/kali/.config/nuclei/config.yaml && \
    rm -rf /home/kali/nuclei-templates
RUN sudo rm -rf /usr/lib/llvm-*/build && \
    sudo chmod o+rx /usr/lib/mysql/plugin/auth_pam_tool_dir /etc/ssl/private
RUN printf '%s\n' 'PS1="${PS1%\\n*} \[\033[0;36m\]\D{%y-%m-%d} - \t\[\033[;32m\]\n${PS1##*\\n}"' >> /home/kali/.bashrc && \
    mkdir -p /home/kali/workspace && cd /home/kali/workspace && git init
ENV TZ=Asia/Shanghai
ENV PLAYWRIGHT_MCP_BROWSER=chromium
ENV CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
ENV PYTHONUNBUFFERED=1

# 安装jdk8
RUN curl -s "https://get.sdkman.io" | bash && \
    bash -c "source $HOME/.sdkman/bin/sdkman-init.sh && sdk install java 8.0.482.fx-zulu" && \
    rm -rf $HOME/.sdkman/archives/* $HOME/.sdkman/tmp/*
ENV JAVA_HOME=/home/kali/.sdkman/candidates/java/current
ENV PATH=/home/kali/.sdkman/candidates/java/current/bin:$PATH

# agent-browser
RUN sudo apt update && sudo apt install -y xvfb libgtk-3-0 libgbm1 && sudo apt clean
RUN sudo npm install -g agent-browser && \
    sudo mkdir -p /opt/chrome && cd /opt/chrome && \
    sudo wget -q "https://registry.npmmirror.com/-/binary/chrome-for-testing/149.0.7827.54/linux64/chrome-linux64.zip" && \
    sudo unzip -q chrome-linux64.zip && sudo rm chrome-linux64.zip
ENV AGENT_BROWSER_CHROME_PATH=/opt/chrome/chrome-linux64/chrome
ENV DISPLAY=:99
RUN echo 'alias agent-browser="xvfb-run agent-browser"' >> /home/kali/.bashrc
RUN npx skills add vercel-labs/agent-browser --skill agent-browser -g -y

RUN sudo mkdir -p /opt/mirrorstrike/claude-code /opt/mirrorstrike/logs/agent /opt/workspace /opt/tools && \
    sudo chown -R kali:kali /opt/mirrorstrike /opt/workspace /opt/tools
WORKDIR /opt/mirrorstrike/claude-code