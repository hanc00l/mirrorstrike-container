# =============================================================================
# Dockerfile — mirrorstrike agent 执行容器（2026-11-09 重构为 base 变体层）
#
# FROM mirrorstrike-base（公共工具面，见 Dockerfile.base），本层只追加 agent 专属：
#   - qemu-user/-static + libc6-dbg（跨架构 Pwn 承诺；base 因 tsec-hosted 排除而不含）
#   - semgrep（代码审计场景，pip --user）
#   - pi-coding-agent（版本锁定 + 漂移断言；Node 断言由 base 统一门 ≥24.9 承担）
# 无 ENTRYPOINT：平台 Worker 以 --entrypoint tail -f /dev/null 常驻、docker exec 拉起。
# =============================================================================
# FROM 可覆盖：CI 传本次构建的 base tag（同仓 PR 为 :base-pr-<num>），本地默认 :base
ARG BASE_IMAGE=ghcr.io/hanc00l/mirrorstrike-container:base
FROM ${BASE_IMAGE}

# pi 执行器版本锁定：裸装会随 npm latest 漂移
ARG PI_VERSION=1.4.0

# ---------- 1. qemu 系 + libc6-dbg（跨架构 Pwn：base 不含，本层补）----------
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    sudo apt-get update -o Acquire::Retries=10 -o Acquire::http::Timeout=60 && \
    sudo apt-get install -y --no-install-recommends \
      -o Acquire::Retries=10 -o Acquire::http::Timeout=60 \
      qemu-user qemu-user-static libc6-dbg && \
    sudo apt-get clean && sudo rm -rf /var/lib/apt/lists/*

# ---------- 2. semgrep（完整版代码审计场景承诺，技能面 ~/.local/bin/semgrep）----------
RUN pip3 install --user semgrep
# pip --user 工具落在 ~/.local/bin，加入 PATH 供 agent 调用
ENV PATH=/home/kali/.local/bin:${PATH}

# ---------- 3. AI 执行器：pi-coding-agent（版本锁定 + 漂移断言）----------
# @modelcontextprotocol/sdk 为 pi 扩展裸导入的解析兜底；/opt/mirrorstrike/node_modules
# 软链到全局包目录，供运行时只读挂载的 pi-agent（可能不含 node_modules）做 ESM 解析。
# 必须用 /opt/node24/bin/npm 的绝对路径：sudo 的 secure_path 不含 /usr/local/bin，
# 裸 `sudo npm` 会命中 apt 版并把包装到 /usr/local/lib/node_modules，与下面的
# /opt/node24/lib/node_modules 软链失配（软链指向空目录 → 扩展 ESM 导入 ERR_MODULE_NOT_FOUND）。
RUN sudo env PATH=/opt/node24/bin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
      /opt/node24/bin/npm install -g \
      @earendil-works/pi-coding-agent@${PI_VERSION} \
      @modelcontextprotocol/sdk && \
    sudo ln -sfn /opt/node24/lib/node_modules /opt/mirrorstrike/node_modules && \
    sudo ln -sfn /opt/node24/bin/pi /usr/local/bin/pi && \
    test -f /opt/mirrorstrike/node_modules/@earendil-works/pi-coding-agent/package.json && \
    test -d /opt/mirrorstrike/node_modules/@modelcontextprotocol/sdk
# 版本锁定不是自证的：断言实际安装版本等于 PI_VERSION（fail-closed）
# Node 版本门已由 base 统一承担（≥24.9，覆盖 pi ≥22.19）
RUN pi --version >/dev/null || { echo "[fatal] pi 不可用"; exit 1; } && \
    test "$(pi --version)" = "${PI_VERSION}" || { echo "[fatal] pi 版本漂移: $(pi --version) != ${PI_VERSION}"; exit 1; }

# 无 ENTRYPOINT：平台 Worker 以 --entrypoint tail -f /dev/null 常驻容器、docker exec
# 拉起 Agent；WORKDIR 仅影响默认 shell 位置，Agent cwd 由 wrapper 设置为 pi-agent。
WORKDIR /opt/mirrorstrike
CMD ["/bin/bash"]
