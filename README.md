# mirrorstrike-container

基于 Kali Linux 的 MirrorStrike Docker 容器环境，预装渗透测试工具集与 AI 编码代理。

## 快速开始

```bash
docker pull ghcr.io/hanc00l/mirrorstrike-container:latest
docker run -it ghcr.io/hanc00l/mirrorstrike-container:latest
```

## 构建

```bash
docker build -t ghcr.io/hanc00l/mirrorstrike-container:latest container/
```

## 包含的工具

### 渗透测试框架
- [Metasploit](https://www.metasploit.com/) — 漏洞利用框架
- [NetExec](https://github.com/Pennyw0rth/NetExec) — 网络服务利用工具包
- [pwncat](https://github.com/calebstewart/pwncat) — 高级反向 Shell 框架
- [Coercer](https://github.com/p0dalirius/Coercer) — 强制认证攻击
- [BloodyAD](https://github.com/CravateRouge/bloodyAD) — Active Directory 权限提升

### 扫描与发现
- [Nuclei](https://github.com/projectdiscovery/nuclei) — 基于模板的漏洞扫描器
- [Katana](https://github.com/projectdiscovery/katana) — Web 爬虫
- [Naabu](https://github.com/projectdiscovery/naabu) — 快速端口扫描
- [Nikto](https://github.com/sullo/nikto) — Web 服务器扫描
- [dirsearch](https://github.com/maurosoria/dirsearch) — Web 路径暴力破解

### 漏洞利用
- [ysoserial](https://github.com/frohoff/ysoserial) — Java 反序列化
- [jwt_tool](https://github.com/ticarpi/jwt_tool) — JWT 测试工具包
- [Dalfox](https://github.com/hahwul/dalfox) — XSS 扫描与利用
- [JDWP-shellifier](https://github.com/IOActive/jdwp-shellifier) — JDWP 漏洞利用

### AI 编码代理
- [Claude Code](https://github.com/anthropics/claude-code) — Anthropic 编码助手
- [OpenAI Codex](https://github.com/openai/codex) — OpenAI 编码助手
- [Pi Coding Agent](https://github.com/mariozechner/pi-coding-agent) — 轻量级编码代理

### 知识库
- PayloadsAllTheThings / InternalAllTheThings
- HackTricks / HackTricks Cloud
- 漏洞库: CVE-PoC, exphub, Awesome-POC, vulhub

## 自动化构建

GitHub Actions 在推送/PR 到 `main` 分支时自动构建并推送镜像至 GitHub Container Registry。


## 参考

[Cairn](https://github.com/oritera/Cairn)

## 许可

本项目仅用于合法的安全测试和教育目的。
