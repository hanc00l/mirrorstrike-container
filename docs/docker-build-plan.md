# Docker 镜像构建方案（mirrorstrike + sextant 统一梳理）

> 2026-11-09 起草。梳理 `/opt/mirrorstrike`、`/opt/sextant` 两仓库对容器环境的需求现状，
> 结合现有 `container/Dockerfile`（agent 完整版）与 `container/Dockerfile.tsec-hosted`
> （tsec 托管版），收敛为 **1 个公共 base + 3 个镜像模板**（新增 sextant 模板，
> DSH 以同构建工具链 + 源码挂载形态提供，不装本体）。

## 1. 需求现状

### 1.1 两个仓库的运行形态

| 仓库 | 角色 | 容器内需要什么 |
| --- | --- | --- |
| `/opt/mirrorstrike` | 渗透测试任务调度 + pi-agent 执行环境 | Kali 渗透工具面、pi-coding-agent、（tsec 模式另需 PG/Redis server + 调度器自启动） |
| `/opt/sextant` | DSH（DeepSeek Harness）之上的竞赛求解运行时（controller/scheduler/worker） | 与 agent 相同的渗透工具面（Worker 在容器内执行任务）+ Node ≥ 24.9 + **deepseek-harness `dsh-v0.2.2-alpha.3`**；不需要 redis-server / postgresql-server / pi-agent |

### 1.2 现有模板盘点

| 模板 | 用途 | 现状 |
| --- | --- | --- |
| `container/Dockerfile` | **agent 执行环境**（四场景工具面超集，运行时挂载 `/opt/tools`、`/opt/kb`） | 无 ENTRYPOINT，`CMD ["/bin/bash"]`，平台 Worker 以 `tail -f /dev/null` 常驻 + `docker exec` 拉起 Agent —— 形态正确，保留 |
| `container/Dockerfile.tsec-hosted` | **tsec 托管 all-in-one**（含 PG/Redis server、`COPY app/ bin/ kb/`、ENTRYPOINT=tini+entrypoint.sh 自启动题目调度器 guard.py） | 已满足"自启动调度器"要求，保留 |
| （缺失） | **sextant 模板** | 需新增：工具面与 agent 相同，去 server 端/去 pi；DSH `dsh-v0.2.2-alpha.3` 以"构建工具链 + 源码挂载 + bootstrap 引导"形态提供（§4.3，不装本体），纯安装环境 + volume 挂载 |

### 1.3 现有 Dockerfile 的问题

1. **两份 Dockerfile 大段重复**：apt 清单、nmap 修复、pinned 资产、Android 栈、JDK8、
   nuclei 配置等约 70% 内容复制粘贴，双份维护、易漂移（tsec-hosted 的 `psmisc`、
   `python3-setuptools`、apt 缓存挂载、curl retry 均未同步）。
2. **tsec-hosted 的 `COPY runtime-tools/deepstrike-kit / kb/ / app/ / bin/`** 把执行
   内容打进镜像——托管模式这是刻意的，但构建上下文必须由 `deploy/hosted/build-image.sh`
   在 mirrorstrike 仓库组装，与 agent 镜像的"零 GitHub 下载 + pinned 资产"约定耦合在同一
   文件里，阅读成本高。
3. **没有 sextant 模板**；sextant 与 agent 的差异（去 pi-agent、去 server 端、加 DSH）
   目前只能靠人工在 agent 镜像上二次改造。
4. **Node 版本冲突**：agent 模板断言 Node ≥ 22.19（pi 要求）；DSH/Sextant 要求
   **Node ≥ 24.9**（`sextant/docs/deployment.md`）。Kali rolling 的 `nodejs` apt 包版本
   会漂移，不能假定同时满足两者 —— **2026-11-09 裁决：三模板统一到 sextant 基线
   ≥24.9（pinned tarball），见 §4.2**。
5. capa-rules 仍是 `git clone --depth 1` 未固定哈希（唯一"零 GitHub 下载"例外，
   AGENTS.md 复核待办），三模板统一后应一次性收口。

## 2. 目标模板矩阵

统一收敛为 **1 个公共 base + 3 个模板**（共四个 Dockerfile），放在 `container/` 下；
base 的抽法见 §5：

| 文件 | 模板 | 自启动 | 进程模型 | 主要内容 |
| --- | --- | --- | --- | --- |
| `container/Dockerfile` | `mirrorstrike-agent`（现完整版改名语义） | ❌ 无 ENTRYPOINT | `docker run tail -f /dev/null` + `docker exec` | Kali 工具面超集 + pi-coding-agent 1.4.0 + semgrep + qemu 系 |
| `container/Dockerfile.tsec-hosted` | `mirrorstrike-tsec-hosted` | ✅ tini + entrypoint.sh 自启动 tsec 调度器 | all-in-one 常驻 | 同上工具面（去 qemu 系）+ PG/Redis server + `app/ bin/ kb/ deepstrike-kit` COPY |
| `container/Dockerfile.sextant` | `sextant-runtime`（**新增**） | ❌ 无 ENTRYPOINT | `docker run` 常驻壳 + `docker exec` 启动 DSH profile / Worker | Kali 渗透工具面（含 qemu 系，与 agent 同清单）+ Node 24（base 继承）；**不含** redis/postgresql server、pi-agent、semgrep |

三者共同的**设计原则**（sextant 与 agent 模板严格遵循）：

- **只构建安装环境**：镜像内只装解释器/二进制工具/运行时依赖；执行的文件、知识库
  （kb）、第三方自研工具（deepstrike-kit 等）、任务工作区一律经 volume 挂载动态加载。
- **进程经 `docker run` / `docker exec` 启动**：容器只是"带全套渗透工具的壳"，
  生命周期由平台 Worker / 调度方管理。
- **零 GitHub 下载**：仅允许 URL+SHA256 锁定的 pinned 资产（chisel/grpcurl/jadx/
  apktool/capa/Temurin JDK21/Node 发行包），capa-rules 例外待固定 commit 后收口。

## 3. 工具面与组件差异矩阵

以 agent 模板为基准（= 现 `Dockerfile`，tsec-hosted 工具面超集 + 四场景刚需）：

| 组件 | agent | tsec-hosted | sextant |
| --- | --- | --- | --- |
| Kali 渗透/CTF/Pwn/逆向 apt 工具面 | ✅ 超集 | ✅（去 qemu 系/libc6-dbg/semgrep） | ✅ **与 agent 完全同清单**（含 qemu 系，保持任务执行能力一致） |
| pinned 资产（chisel/grpcurl/jadx/apktool/capa/JDK21） | ✅ | ✅ | ✅ 同源同值，三处共享一份 ARG 定义 |
| postgresql / redis-server / tini | ❌ | ✅ | ❌（明确排除） |
| pi-coding-agent + @modelcontextprotocol/sdk | ✅ 1.4.0 锁定 | ✅ | ❌（明确排除） |
| playwright/chromium 浏览器层 | ✅（门控） | ✅ | ✅（门控，`WITH_PLAYWRIGHT` 同款） |
| semgrep | ✅ | ❌ | ❌（非代码审计执行环境） |
| JDK8（sdkman zulu，剔 JavaFX） | ✅ | ✅ | ✅（任务场景如 Java 反序列化仍需要） |
| Node.js | **Node 24 pinned（统一版）**，构建期断言 ≥ 24.9（同时覆盖 pi 的 ≥22.19 与 DSH/Sextant 的 ≥24.9），见 §4.2 | 同 base 统一版 | 同 base 统一版（sextant 模板直接继承，不再额外装） |
| deepseek-harness 本体 | ❌ | ❌ | ❌ **不安装**——镜像仅装构建工具链（Node24/pnpm/编译链 + `sextant-bootstrap` 引导脚本），DSH/Sextant 源码与构建产物经 volume 挂载（§4.3） |
| corepack / pnpm | ✅（base 统一预装 `pnpm@12.16.1`） | ✅ 同 base | ✅（DSH 同构建与 sextant 构建/校验需要） |
| `/opt/tools` `/opt/kb` `/opt/workspace` `/opt/traffic` | 运行时挂载占位 | 构建期 COPY（tools/kb）+ 运行时注入（workspace/traffic） | 运行时挂载占位（同 agent 形态） |
| ENTRYPOINT | 无，`CMD ["/bin/bash"]` | tini + entrypoint.sh | 无，`CMD ["/bin/bash"]` |
| 运行用户 | kali(uid 1000) + sudo 免密 | root | kali(uid 1000) + sudo 免密 |

## 4. sextant 模板设计（`container/Dockerfile.sextant`）

### 4.1 构建思路（与 agent 模板一致）

只装环境，不装业务。镜像内**不 COPY** sextant 源码、DSH 业务 bundle、知识库、
挑战数据；这些全部经 volume 挂载：

```text
/opt/sextant              ← sextant 仓库 checkout（rw，构建产物 lib/ 也可现场重建）
/opt/deepseek-harness     ← DSH 源码 checkout（DSH_SOURCE_DIR，同构建工作区）
/opt/sextant-runtime      ← $DSH_HOME：sessions/、profiles/、凭证、工作区（volume）
/opt/workspace /opt/traffic /opt/tools /opt/kb  ← 任务面挂载点（同 agent 形态）
```

进程通过 `docker exec` 启动，例如：

```bash
# 控制台 profile（同构建 sextant-web）
docker exec sextant-node bash -lc \
  'cd /opt/deepseek-harness && DSH_HOME=/opt/sextant-runtime \
   SEXTANT_MATCH_ID=<id> node --import tsx/esm apps/cli/src/bin.ts \
   --profile sextant-web --port 3081 --no-open'

# process 模式 worker（sextant-worker profile）
docker exec sextant-node bash -lc \
  'cd /opt/deepseek-harness && DSH_HOME=/opt/sextant-runtime \
   dsh --profile sextant-worker "<prompt>"'
```

无 ENTRYPOINT、`WORKDIR /opt/sextant`、`CMD ["/bin/bash"]`，与 agent 模板形态对齐。
示例中的 `dsh` 是 bootstrap 生成的 launcher（§4.3 步骤 6）；如需指到其它 checkout，
用 `SEXTANT_DSH_EXECUTABLE` 覆盖。

### 4.2 Node 版本统一（2026-11-09 补充裁决：三模板统一到 sextant 基线）

**裁决：全部模板统一使用 sextant 基线的 Node ≥ 24.9**，不再维持"pi 用 apt nodejs
（≥22.19）、sextant 另装 Node24"的双轨。Kali apt 的 `nodejs` 版本不可控、无法承诺
≥24.9，因此统一采用 **pinned Node tarball**（延续 URL+SHA256 约定）安装在 base 镜像
`/opt/node24`，三变体直接继承：

```dockerfile
ARG NODE24_URL=https://nodejs.org/dist/v24.9.0/node-v24.9.0-linux-x64.tar.xz
ARG NODE24_SHA256=<锁定哈希>
RUN curl -fsSL -o /tmp/node24.tar.xz "${NODE24_URL}" && \
    echo "${NODE24_SHA256}  /tmp/node24.tar.xz" | sha256sum -c - && \
    mkdir -p /opt/node24 && \
    tar -xJf /tmp/node24.tar.xz -C /opt/node24 --strip-components=1 && \
    rm /tmp/node24.tar.xz && \
    ln -sf /opt/node24/bin/node /usr/local/bin/node && \
    ln -sf /opt/node24/bin/npm  /usr/local/bin/npm && \
    ln -sf /opt/node24/bin/npx  /usr/local/bin/npx
ENV PATH=/opt/node24/bin:${PATH}
# 构建期版本门（对全部模板生效；单行写法——引号内反斜杠换行会传入字面量导致 JS 语法错）
RUN node -e 'const v=process.versions.node.split(".").map(Number);if(v[0]<24||(v[0]===24&&v[1]<9)){console.error("[fatal] 统一 Node 基线要求 >= 24.9");process.exit(1)}'
```

- pi-coding-agent 要求 ≥22.19，被 ≥24.9 覆盖；agent/tsec-hosted 变体的版本断言改为
  ≥24.9 同一门（原 22.19 断言删除，改注释指向统一基线）。
- corepack + pnpm 下沉进 base：`corepack enable`（生成 `/opt/node24/bin` 下 shim）+
  `corepack prepare pnpm@12.16.1 --activate`（Sextant packageManager）与
  `corepack prepare pnpm@11.28.5`（DSH workspace 的 packageManager，见
  `sextant/docs/dsh-integration.md`）**两个版本都预置**——corepack 在各自目录按
  `packageManager` 字段自动切换；预置是为了避开运行期首次 `pnpm` 触发的联网下载
  （离线比赛机适用）。体积约 +20MB，换取三镜像 Node/pnpm 工具链完全一致。
- 升级 Node 只改 `NODE24_URL` + SHA256；若 Kali apt `nodejs` 未来稳定 ≥24.9，可评估
  回退 apt 形态，但 pinned tarball 仍是防漂移的默认形态。

> sextant 变体层**不再有 Node 安装步骤**，仅继承 base；§4.3 的 DSH 同构建对 Node
> 版本的要求即镜像全局要求。base 内 `npm i -g corepack@0.36.0`（Node 自带 0.34
> 不兼容 pnpm 12，见 §10 复核记录 #12）。

### 4.3 deepseek-harness：镜像只装构建工具链，源码与产物挂载（2026-11-09 裁决）

按 `sextant/docs/deployment.md`，Sextant 与 DSH 是**同构建**关系：DSH 以源码 checkout
（`DSH_SOURCE_DIR`）参与构建，Sextant 的 `packages/sextant/*` 链接进 DSH workspace，
事件 catalog 校验与 profile 生成都在该源码树上完成。镜像内装一个脱离源码的全局
`@deepseek-ai/dsh` npm 包无法承载同构建校验链。

**裁决：sextant 模板不安装 deepseek-harness 本体**（不预装源码、不预装 npm 包），
镜像只承担"构建工具链 + 运行时依赖"的安装环境，完全对齐 §1 的设计原则——执行的
文件（DSH/Sextant 源码与构建产物）经 volume 挂载动态加载：

- **镜像内预装**：Node 24 pinned（§4.2）、corepack + 双版本 pnpm（12.16.1 / 11.28.5，
  §4.2）、native addon 编译链（`gcc`/`make`/`python3`——base 工具面已含）、
  **`musl-tools`（landlock-run 静态构建必需）**、git；以及首次引导脚本
  `/usr/local/bin/sextant-bootstrap`（下述）。
- **native 构建的两个硬前提**（2026-11-09 复核 `native/system/scripts/build.ts` +
  `native/system/README.md`）：
  1. **Node 头文件**：官方 Node tarball 自带 `include/node/*.h`——pinned 安装必须保留
     `include/`，否则 `build.ts` 找不到 Node-API 头文件直接抛错；
  2. **完整构建需要 musl-gcc**：README 明示 "Linux requires musl-gcc for a complete
     build"；`--host-addon-only` 只产 `system.node`（flock），**跳过** `landlock-run`
     （static musl 二进制）。
- **为什么必须产 `landlock-run`**：DSH 沙箱 provider 在 Linux 的 runner 顺序是
  `['bwrap', 'landlock']`（`packages/sandbox/sandbox-local/src/index.ts`）；两者都不可用
  时 `SandboxUnavailableError`（**fail-closed**），`workspace-write` 模式无法启动。
  镜像默认不装 bubblewrap（容器内 bwrap 依赖 user namespace，Docker 默认 seccomp 常
  阻断），因此**以 landlock 为容器内沙箱路径**：sextant 层装 `musl-tools`，bootstrap
  执行完整 `node native/system/scripts/build.ts`（**不带** `--host-addon-only`），产出
  `bin/landlock-run` + `bin/glibc/system.node`；验收时跑
  `<DSH>/native/system/packages/linux-x64/bin/landlock-run --probe`，期望
  `landlock: fully enforced`（老内核 `partially enforced` 也可用）。
  > 若目标内核 <5.13 无 Landlock，则改装 `bubblewrap` 并确认 Docker 放行 userns，
  > 作为后备路径（写进运行手册，非默认）。
- **挂载卷提供**：
  - `/opt/deepseek-harness`（rw）— DSH 源码 checkout，由调用方 checkout 到
    **pinned ref `dsh-v0.2.2-alpha.3`** 并核对 commit SHA；
  - `/opt/sextant`（rw）— sextant 仓库 checkout；
  - `/opt/sextant-runtime`（rw）— `$DSH_HOME`（sessions/profiles/凭证）。
- **首次引导（重活只做一次，产物留在卷上）**：`sextant-bootstrap` 幂等脚本按
  `deployment.md §2.3` 顺序执行，已完成的步骤跳过：
  0. **版本前置校验（fail-closed）**：校验 `/opt/deepseek-harness` 的 ref/commit 与
     `DSH_EXPECTED_REF`（及可选的 `DSH_EXPECTED_COMMIT`）一致、`/opt/sextant` checkout
     存在；不符直接退出，不进入构建。
  1. `cd /opt/deepseek-harness && corepack pnpm install`
  2. `node native/system/scripts/build.ts`（**完整构建**：产 `landlock-run` 静态 musl +
     `system.node`；native 二进制与内核/glibc 绑定，须在目标容器内构建——镜像与宿主
     同为 amd64 Linux）。deployment.md 的 `--host-addon-only` 只够源码测试，缺
     `landlock-run` 会令 workspace-write 沙箱 fail-closed，故此处修正。
  3. `pnpm run build:lib:host && build:lib:client && build:web`
     （`NODE_OPTIONS=--max-old-space-size=6144` 防 OOM）
  4. `cd /opt/sextant && pnpm install --frozen-lockfile && pnpm run build:dsh-integration`
     （同构建：`packages/sextant/*` 链接进 DSH workspace；幂等标记记录 sextant/dsh
     双 commit，任一变化即重跑同步，防止沿用旧产物/旧 catalog）
  5. `pnpm run verify:dsh-build`（12/12 PASS 门禁）+
     `pnpm run setup:dsh-web-profile`（生成 sextant-web / sextant-worker profile）。
     **bootstrap 全程统一 `export DSH_HOME="$DSH_HOME_DIR"`**（2026-11-09 复核 P1 修复：
     setup 脚本 `dshHome()` 是 DSH_HOME 优先、否则 `$HOME/.dsh`；只覆盖 HOME 会把
     profile 落到 `.dsh/` 子层，与运行期 `resolveDshHome()` 的 `$DSH_HOME` 差一层
     `.dsh` → profile not found）。bootstrap 后断言
     `$DSH_HOME/profiles/{sextant-web,sextant-worker}` 存在。
  6. **安装 `dsh` launcher**：sextant 的 process Worker 以 `SEXTANT_DSH_EXECUTABLE`
     （默认 `dsh`，见 `packages/dsh-plugin/src/dsh/wiring.ts`）执行
     `dsh --profile sextant-worker`。DSH 源码包的 bin 是
     `apps/cli/lib/bin.js`（`apps/cli/package.json` 的 `bin.dsh`，由 `build:lib:host`
     产出），因此 bootstrap 生成 `/usr/local/bin/dsh` 包装脚本
     （`exec /opt/node24/bin/node /opt/deepseek-harness/apps/cli/lib/bin.js "$@"`），
     并断言 `dsh --version` 输出 `0.2.2-alpha.3`。该 launcher 落在镜像层
     `/usr/local/bin`，由 kali 经 sudo 写入。
  7. 输出版本清单（DSH commit、Sextant commit）写入
     `$DSH_HOME/sextant-build-manifest.json`，供验收核对。
- **镜像侧门禁**（CI 构建期仍可执行）：用一个临时构建 stage 挂载 pinned DSH/Sextant
  源码跑一遍 bootstrap 全链路，exit 0 才发布——保证镜像内的工具链与 pinned 基线
  兼容，但不把产物留进镜像。
- 运行期进程仍经 `docker exec` 启动（§4.1 示例不变）；模型 key 经
  `DEEPSEEK_API_KEY` 环境变量注入，不进镜像。

**代价与边界**（写明，避免误用）：

- 首次 `docker run` + bootstrap 约 10–30 分钟（pnpm install 依赖树 + 全仓 typecheck），
  且需要 ≥6GB 内存配额；产物持久在 volume 上，后续容器直接复用，秒级就绪。
- 换 DSH ref/commit 后必须清卷上旧产物（`lib/`、`*.tsbuildinfo`，见 deployment.md
  §2.2 实证教训）再重跑 bootstrap。
- 镜像与挂载源码的版本由调用方负责对齐（镜像只锁工具链版本；DSH ref/commit 校验在
  bootstrap 步骤 0 以 `DSH_EXPECTED_REF`/`DSH_EXPECTED_COMMIT` fail-closed 校验）。

> 备选形态（未采纳，留档）：构建期把 pinned DSH 源码 + 产物 COPY 进镜像
> （镜像 +1.5–2GB，但免首次引导）；或运行期只装 `@deepseek-ai/dsh@0.2.2-alpha.3`
> npm 包（放弃同构建校验链）。均与"纯安装环境 + 挂载动态加载"原则冲突，仅在
> 禁止运行期构建的场景（如离线比赛机）作为降级方案启用。

### 4.4 Dockerfile 骨架

```dockerfile
# 构建上下文：container/（与现有 CI 一致：context: container, file: container/Dockerfile）
FROM ghcr.io/hanc00l/mirrorstrike-container:base
# —— base 已含：apt 工具面、pinned 资产、Android 栈、JDK8、Node24/pnpm、kali 用户、
# nmap 修复、nuclei 离线配置、/opt/{tools,kb,workspace,traffic} 占位（§5）——
# （§3 差异矩阵：sextant 层不装 postgresql/redis-server/tini/pi-coding-agent/semgrep）
# —— qemu 系 + libc6-dbg：与 agent 同承诺（§3），base 不含，本层补 ——
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends qemu-user qemu-user-static libc6-dbg musl-tools && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
# —— 首次引导脚本：幂等，按 §4.3 步骤 0–7 执行，产物落挂载卷/ /usr/local/bin 不进镜像源码 ——
ARG DSH_EXPECTED_REF=dsh-v0.2.2-alpha.3
ENV DSH_EXPECTED_REF=${DSH_EXPECTED_REF}
COPY sextant-bootstrap /usr/local/bin/sextant-bootstrap
# —— 目录占位（均为运行时挂载点，镜像内不装物；kali 用户由 base 创建）：——
RUN mkdir -p /opt/sextant /opt/deepseek-harness /opt/sextant-runtime && \
    chown -R kali:kali /opt/sextant /opt/deepseek-harness /opt/sextant-runtime
# —— VOLUME 占位（2026-11-09 补充）：声明卷挂载点，`docker inspect` 可自描述挂载契约，
# 运行时 -v/--mount 覆盖同名挂载点；未显式挂载时 Docker 自动落匿名卷（数据不随容器
# 删除而丢，但也不受管理）——生产启动命令必须显式 -v，VOLUME 仅作契约声明与兜底。
# 任务面四挂载点（tools/kb :ro 语义在 docker run -v 侧声明，VOLUME 指令无法表达只读）
# 与 agent 模板保持同一组挂载点；deepstrike-kit 等经 tools 卷提供，镜像内不 COPY。
# —— VOLUME 占位（复核修订）：仅任务面四挂载点 + 运行态卷；源码目录不声明（缺挂载
# 由 bootstrap 步骤 0 fail-closed 报 "checkout 缺失"，不落匿名卷）：——
VOLUME ["/opt/tools", "/opt/kb", "/opt/workspace", "/opt/traffic", "/opt/sextant-runtime"]
USER kali
WORKDIR /opt/sextant
CMD ["/bin/bash"]
```

## 5. 三模板去重：共享层方案

避免继续三份粘贴，采用 **`FROM` 基础镜像两级构建**（不引入 BuildKit `--build-context`
的离线兼容性风险）：

```text
container/
├── Dockerfile              # agent：FROM mirrorstrike-base，追加 qemu 系+libc6-dbg / semgrep / pi
├── Dockerfile.tsec-hosted  # tsec：FROM mirrorstrike-base，追加 PG/Redis/tini + COPY app/kb/kit + ENTRYPOINT
├── Dockerfile.sextant      # sextant：FROM mirrorstrike-base，追加 qemu 系+libc6-dbg / sextant-bootstrap
└── Dockerfile.base         # base：公共 apt 渗透工具面（不含 qemu 系）+ pinned 资产 + Android 栈 + JDK8 + Node24/pnpm + kali 用户 + nmap 修复 + nuclei 配置
```

- **base 镜像**包含三者共同的最大交集：§3 矩阵中"三列全 ✅"的条目（apt 工具面全量——
  qemu 系因 tsec-hosted 排除而**不**进 base、pinned 资产、Android 逆向栈、JDK8、
  浏览器层、kali 用户、nmap 入口修复、nuclei 离线配置、`/opt/*` 占位目录），以及
  **2026-11-09 统一的 Node 24 pinned + corepack/pnpm 工具链**（§4.2）。
- **agent** = base + qemu 系 + libc6-dbg + semgrep + pi-coding-agent + `node_modules`
  软链（无 ENTRYPOINT）；pi 的 Node 断言随统一基线改为 ≥24.9。
- **tsec-hosted** = base + PG/Redis server + tini + `COPY app/ bin/ kb/ deepstrike-kit`
  + ENTRYPOINT（保持现 build-image.sh 组装上下文的发布形态）。
- **sextant** = base + qemu 系 + libc6-dbg + `sextant-bootstrap` 引导脚本
  （Node/编译链已由 base 提供，无额外 Node 安装步骤）。
- 版本 ARG 收敛：pinned URL/SHA256（chisel/grpcurl/jadx/apktool/capa/JDK21/Node24）
  **只存在于 `Dockerfile.base`**，变体不再复制；`PI_VERSION` 仅 agent / tsec-hosted
  两处同步（后续可再抽 `versions.env` 由 build 脚本 `--build-arg` 注入，进一步消除）。

> 若不想引入 base 镜像发布链路（多一次 push/依赖），退化方案是保留三份自足 Dockerfile、
> 仅共享 `versions.env`。默认推荐 base 方案：改动收敛、CI 构建时间显著下降
> （apt 工具面层三镜像共享缓存）。

## 6. CI / 构建 / 门禁

- `Dockerfile.base` 先行构建并推送 `ghcr.io/hanc00l/mirrorstrike-container:base`；
  本仓库 GitHub Actions 串联 **base → 并行 agent / sextant**（tsec-hosted 不在本仓库
  CI——见下条）。镜像命名从现 `mirrorstrike-container:latest` 平移为
  `mirrorstrike-agent`，需在 README/平台侧引用处同步（过渡期可双 tag `latest` 兼容）。
  **注意 tsec-hosted 的构建入口在 mirrorstrike 仓库**：`deploy/hosted/build-image.sh`
  负责组装 `runtime-tools/ kb/ app/ bin/` 上下文，其 `FROM` 需同步改为 base 镜像，
  并在离线构建时预拉 `:base`（base 镜像变更时需手动触发该脚本重建 tsec-hosted，
  本仓库 CI 不串联它）。
- sextant 镜像的门禁：
  1. 构建期：Node ≥ 24.9 断言、pnpm/corepack 版本断言、`musl-gcc` 可用性断言。
  2. CI 发布前：临时 stage 挂载 pinned DSH/Sextant 源码跑 `sextant-bootstrap`
     全链路（§4.3），exit 0 + `verify:dsh-build` 12/12 PASS +
     `landlock-run --probe` 为 `fully/partially enforced` + `dsh --version` 与
     pinned 版本一致，才发布（产物不进镜像）。
  3. 运行期首次引导：`verify:worker-profile`（不调模型/平台的启停冒烟）。
- 体积门禁：沿用 `deploy/hosted/build-image.sh` 的 3GB 压缩门禁思路。sextant 镜像
  **不含** DSH 依赖树与构建产物（均在卷上），预计与 agent 完整版同量级；首次实测后
  定独立阈值。
- tag 方案：
  - `mirrorstrike-agent:<date>` / `latest`
  - `mirrorstrike-tsec-hosted:<date>`
  - `sextant-runtime:<date>` + `DSH 版本` 双 tag（如 `sextant-runtime:dsh0.2.2-alpha.3`），
    便于按 DSH 版本锁定比赛环境。

## 7. 运行规范（volume 挂载契约）

| 镜像 | 挂载点 | 读写 | 内容 |
| --- | --- | --- | --- |
| agent / sextant | `/opt/tools` | ro | deepstrike-kit 等自研工具 |
| agent / sextant | `/opt/kb` | ro | 知识库 + nuclei-templates |
| agent / sextant | `/opt/workspace`、`/opt/traffic` | rw | 任务工作区 / 流量 |
| sextant | `/opt/sextant` | rw | sextant 仓库 checkout（构建产物在容器内 bootstrap 生成后留存） |
| sextant | `/opt/deepseek-harness` | rw | DSH 源码 checkout（pinned `dsh-v0.2.2-alpha.3` + commit 校验；`lib/`、native addon 等构建产物留在本卷） |
| sextant | `/opt/sextant-runtime` | rw | `$DSH_HOME`：sessions/profiles/凭证/工作区（状态卷） |
| tsec-hosted | 无需外部工具/kb 挂载 | — | all-in-one，`/opt/workspace`、`/opt/traffic` 运行时注入 |

> **VOLUME 声明（2026-11-09 复核修订）**：sextant 模板仅对任务面四挂载点 +
> `/opt/sextant-runtime` 写 `VOLUME`；**两个源码目录（`/opt/sextant`、
> `/opt/deepseek-harness`）刻意不声明**——缺挂载时 bootstrap 步骤 0 立即
> fail-closed 报 "checkout 缺失"，而非静默落匿名卷把产物写进容器层。注意两点：
> ① VOLUME 无法表达只读，`/opt/tools`、`/opt/kb` 的 `:ro` 仍由 `docker run -v` 侧
> 声明；② `/opt/sextant-runtime` 未显式 `-v` 时落匿名卷（兜底），生产启动必须显式
> 挂载，否则每次重建都触发全量 bootstrap。
> **预引导卷固化 SOP**：`docker commit` **不包含挂载卷内容**，不可用；离线预引导应
> 跑完 bootstrap 后导出卷 tar（`docker run --rm -v <vol>:/data -v $PWD:/out alpine
> tar -C /data -czf /out/runtime.tgz .`），或构建衍生镜像 COPY 产物（+1.5–2GB，仅
> 离线降级）。浅克隆的 DSH checkout 须先 `git fetch --tags`，否则 bootstrap 步骤 0
> 的 ref 校验 fail。

注意：sextant 模板中 `/opt/kb` 供 Worker 执行任务时读知识库；nuclei 模板路径**统一为
`NUCLEI_TEMPLATES_DIR=/opt/kb/kb-resource/nuclei-templates`**（与 tsec-hosted 一致，
2026-11-09 复核统一），禁联网更新。agent 完整版现值 `/opt/kb/nuclei-templates` 属漂移项，
随 §8 步骤 2 抽 base 时一并改正（`ENV NUCLEI_TEMPLATES_DIR` + `~/.config/nuclei/config.yaml`
两处同步）。**复核结论（2026-11-09）**：该路径与 mirrorstrike 平台侧既有 KB 目录契约
（`{kb_dir}/kb-resource` + `kb-index`，`internal/config/config.go` 固定常量；kb 语料
`BasePath: nuclei-templates`；`query_intent` 测试与 hosted entrypoint 自检均按
kb-resource 断言）完全一致，无运维侧目录重构需求；唯一前置是节点 kb 经 `cmd/kb dist`
组装（`kb_gate.go` 既有 gate 条件）。验证时在容器内跑 `nuclei -tl` 确认模板加载数非零。

## 8. 实施步骤

1. **固定版本基线**：确定 Node 24 pinned 版本 + SHA256、DSH pinned ref
   `dsh-v0.2.2-alpha.3` 对应 commit SHA（写入镜像 `DSH_EXPECTED_REF`/校验逻辑）；
   capa-rules 改 pinned commit + 构建期校验（收口唯一例外）。
2. **抽出 `Dockerfile.base`**：从现 `Dockerfile` 剥离公共层，先保证 agent 镜像
   `FROM base` 后功能与现状等价（逐节 diff + 冒烟）。**同时收口已确认漂移项**：
   `NUCLEI_TEMPLATES_DIR` 统一为 `/opt/kb/kb-resource/nuclei-templates`
   （agent 版现值 `/opt/kb/nuclei-templates`，与 tsec-hosted 不一致；改 `ENV` 与
   `~/.config/nuclei/config.yaml` 两处。已复核：与 mirrorstrike 平台 KB 目录契约
   `{kb_dir}/kb-resource` 一致，无运维侧目录重构需求——已落地于 container/Dockerfile）。
3. **改造 tsec-hosted**：`FROM base`，补齐其待同步项（psmisc、python3-setuptools、
   apt cache mount、curl retry），保持 ENTRYPOINT/COPY 形态与 build-image.sh 不变。
4. **新增 `Dockerfile.sextant` + `sextant-bootstrap` 引导脚本**（构建上下文
   `container/`，故两文件落 `container/Dockerfile.sextant`、`container/sextant-bootstrap`）：
   按 §4 落地（镜像只装工具链），本地构建 + §6 门禁验证（CI stage 挂 pinned 源码跑全链路）。
5. **CI 更新**：`docker-build.yml` 改为 base→三变体串联，补 sextant tag 推送；
   mirrorstrike 的 `deploy/hosted/build-image.sh` 同步改 `FROM :base`（§6 注意）。
6. **文档同步**：AGENTS.md 模板矩阵、README 构建命令、`/opt/sextant/docs/deployment.md`
   增加"容器内运行"一节（§4.1 的 exec 命令）。

## 9. 风险与待拍板项

| 项 | 说明 | 建议 |
| --- | --- | --- |
| DSH 交付形态 | **已裁决**：镜像只装构建工具链，DSH/Sextant 源码与产物挂载，运行期 `sextant-bootstrap` 幂等引导（§4.3） | 离线/禁运行期构建场景再降级为"产物 COPY 进镜像"或 npm 包形态 |
| 首次引导成本 | pnpm install + 全仓 typecheck 约 10–30 分钟、≥6GB 内存；产物在卷上，后续秒级就绪 | `$DSH_HOME` 与两个源码卷必须持久化，勿用匿名卷 |
| Node 版本统一（2026-11-09 裁决） | ~~双轨~~ 已收敛：三模板统一 sextant 基线 Node ≥24.9 pinned（§4.2）；pi（≥22.19）被覆盖 | 升级 Node 只改 base 的 `NODE24_URL`+SHA256；注意 pi 在 Node 24 下的运行回归冒烟 |
| 版本对齐责任 | 镜像只锁工具链；DSH/Sextant 的 pinned commit 由调用方 checkout + bootstrap fail-closed 校验（`DSH_EXPECTED_REF`） | 实施时把 DSH pinned ref 写入镜像 ENV 默认值，换版改 ARG |
| 镜像体积 | sextant 不再含 DSH 依赖树与构建产物，体积回落到 agent 版量级 | 沿用 3GB 门禁同档；首次构建实测后定档 |
| 沙箱 runner（2026-11-09 复核新增） | Linux runner = bwrap→landlock，缺两者则 workspace-write fail-closed；镜像默认无 bwrap，`landlock-run` 需 musl-gcc 完整构建 | sextant 层装 `musl-tools`，bootstrap 跑完整 `build.ts` 并以 `--probe` 门禁；内核 <5.13 才退回 bwrap+userns |

## 10. 复核记录（2026-11-09）

> 第二轮复核（同日）追加修正 6 处：① 开篇/§1.2 仍写"三个镜像模板/装 DSH 本体"，
> 与 base+3 变体及"不装 DSH 本体"裁决不一致；② §4.3 预装清单只写 pnpm 12.16.1，
> 与 §4.2 双版本（12.16.1+11.28.5）预置矛盾；③ §4.2 版本门代码块引号内用反斜杠
> 换行——单引号会传字面量导致 JS 语法错，改单行；④ §5 版本 ARG 仍称"三个文件同步"，
> 改为"pinned 集中 base、PI_VERSION 仅 agent/tsec 两处"；⑤ §6 CI 先说串联
> tsec-hosted 又说其不在本仓库 CI，改为 base→agent/sextant 串联 + tsec-hosted
> 手动触发；⑥ §6 补镜像改名（mirrorstrike-container→mirrorstrike-agent）的引用
> 同步与双 tag 过渡提示。

| # | 复核发现 | 处置 |
| --- | --- | --- |
| 1 | §2 与 §3 对 sextant 是否含 qemu 系自相矛盾 | 统一为「含 qemu 系，与 agent 同清单」（§2/§3/§5） |
| 2 | Node24 下沉 base 后，§5 skeleton 仍写 `FROM kali`、sextant 层仍「追加 Node24/DSH」 | §4.4 改 `FROM :base`，与 §5 对齐 |
| 3 | §5 层分解未体现 qemu 系归属（base 不含、agent 与 sextant 各补） | 明确 base / agent / sextant / tsec 四层内容，逐一列清 |
| 4 | §6 体积预估仍按「含 DSH 依赖树 +1.5–2GB」，与「不装 DSH」冲突 | 改为与 agent 同量级 |
| 5 | §7 表格被 VOLUME 说明块从中间截断 | 说明块移到表格后 |
| 6 | `DSH_EXPECTED_REF` / `DSH_EXPECTED_COMMIT` 命名与步骤号（步骤 0 未列出）不一致 | 补步骤 0（版本前置校验），统一两个 env 名 |
| 7 | `--host-addon-only` 跳过 `landlock-run` → workspace-write 沙箱不可用（fail-closed） | 改为完整 `build.ts` + 装 `musl-tools`，加 `--probe` 门禁 |
| 8 | 未给 `dsh` 可执行入口；sextant worker 进程用 `SEXTANT_DSH_EXECUTABLE`（默认 PATH 上的 `dsh`） | bootstrap 安装 `apps/cli/lib/bin.js` launcher（步骤 6） |
| 9 | COPY 路径与构建上下文（`context: container`）不一致 | skeleton 改 `COPY sextant-bootstrap`，实施步骤写明文件落位 |
| 10 | tsec-hosted 构建入口在 mirrorstrike 仓库，`build-image.sh` 未纳入 CI 依赖说明 | §6 补充 `FROM :base` 同步要求 |
| 11 | **构建实测**（2026-11-09）：`corepack enable` 经 `/usr/local/bin/corepack` 符号链接发现 corepack 时，pnpm shim 落 `/usr/local/bin` 且内部相对 require → `MODULE_NOT_FOUND` | Dockerfile.base 改 `corepack enable --install-directory /opt/node24/bin`（shim 与真实 corepack 同目录），并设 `COREPACK_ENABLE_DOWNLOAD_PROMPT=0` |
| 12 | **构建实测**（2026-11-09）：Node 24.9 自带 corepack 0.34 不兼容 pnpm ≥12 新 bin 布局（`bin/pnpm` 无 `.cjs` 后缀 → `Cannot find module '.../bin/pnpm.cjs'`） | 显式 `npm i -g corepack@0.36.0`（pnpm 12 走原生二进制下载，实测通过）；§4.2 Node24 安装步骤同步补该升级 |
| 13 | **外评采纳**（`/opt/sextant/docs/container-image-review.md` P1）：bootstrap 用 `HOME=` 而 setup 脚本 `dshHome()` 是 DSH_HOME 优先 → profile 落 `.dsh/` 子层，运行期 `resolveDshHome()` 找不到（已对照源码核实） | bootstrap 步骤 5 起统一 `export DSH_HOME`，并断言 `$DSH_HOME/profiles/` 存在；§4.3 步骤 5 同步 |
| 14 | **外评采纳**（同上 P2）：`.dsh-integration-done` 标记不随 checkout 变化失效 | 标记记录 sextant/dsh 双 commit，不一致即重跑同步 |
| 15 | **外评采纳**（同上 P2）：`docker commit` 不含 VOLUME 内容，"预引导打包"须改为导出卷 tar / 衍生镜像 | §7 VOLUME 说明补 SOP；源码目录移出 VOLUME（缺挂载由 bootstrap 步骤 0 fail-closed 兜底），pnpm install 加 `--frozen-lockfile`、补 `git fetch --tags` 说明 |
| 16 | **构建落地实测**（2026-11-09，base→agent→sextant 三连全通过）：① corepack 的 lastKnownGood 默认 per-user（`~/.cache`），运行期 kali 首跑 pnpm 现场下载 latest 不受 `--activate` 钉住 → `COREPACK_HOME=/opt/corepack-home` 烘入双版本 + `chmod a+rX`；② 登录 shell 被 `/etc/profile` 重置 PATH → `/etc/profile.d/node24.sh` 补回；③ base 默认用户为 kali，变体层 `mkdir /opt/*` 须 sudo；④ 构建上下文必须 `container/` | 已全部落入 `Dockerfile.base` / `Dockerfile.sextant`；**实测验收**：agent `pi=1.4.0`/Node 24.9.0/JDK8/12 工具全过、登录 shell `pnpm=12.16.1` 零下载；sextant `musl-gcc`/bootstrap 可执行/无 pi/无 semgrep/VOLUME=5 点/`DSH_EXPECTED_REF=dsh-v0.2.2-alpha.3`+commit `05b1cae8`。镜像体积：base 4.99GB / agent 6.04GB / sextant 5.49GB（tag `base-variant`，正式发布按 §6 tag 方案） |
