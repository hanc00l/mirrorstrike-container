# mirrorstrike-container

MirrorStrike Docker 容器环境，基于 Kali Linux 构建，预装渗透测试工具集与 AI 编码代理。

## 项目结构

```
.
├── container/
│   └── Dockerfile          # Docker 镜像构建文件
├── .github/
│   └── workflows/
│       └── docker-build.yml # 自动化构建与推送工作流
├── CLAUDE.md
└── README.md
```

## 包含的主要工具

- **渗透测试框架**: metasploit, netexec, pwncat, coercer, bloodyad
- **扫描与发现**: nuclei (含模板), katana, naabu, nikto, dirsearch
- **漏洞利用**: ysoserial, jwt_tool, jdwp-shellifier, dalfox
- **凭据与认证**: kerbrute, enum4linux-ng, krb5-user, sshpass
- **AI 代理**: claude-code, codex, pi-coding-agent
- **浏览器自动化**: playwright (chromium)

## 自动化构建

推送或合并 PR 到 `main`/`master` 分支时，GitHub Actions 自动构建并推送镜像至 `ghcr.io/hanc00l/mirrorstrike-container`。

## 参考

[Cairn](https://github.com/oritera/Cairn)
