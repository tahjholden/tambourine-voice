ARG BASE_IMAGE=nvidia/cuda:12.9.1-cudnn-runtime-ubuntu24.04
FROM ${BASE_IMAGE}

# System dependencies for OpenCV and audio processing.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgl1 \
    libglib2.0-0 \
    ffmpeg \
    && rm -rf /var/lib/apt/lists/*

# uv (and uvx) from the official distroless image.
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# uv tuning for containerized builds:
#   UV_LINK_MODE=copy        — cache and install target are on different filesystems
#                              under BuildKit, so hardlinks are not possible.
#   UV_COMPILE_BYTECODE=1    — precompile .pyc at install time for faster cold starts.
#   UV_PROJECT_ENVIRONMENT   — pin the venv path so cache mounts and PATH stay in sync.
ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1 \
    UV_PROJECT_ENVIRONMENT=/app/.venv \
    PATH="/app/.venv/bin:${PATH}"

WORKDIR /app

# Build with: docker build -f server/Dockerfile .  (from repo root)

# Layer 1: install dependencies only. pyproject.toml / uv.lock are bind-mounted
# (not COPYed) so this layer's cache key is purely "did the lockfile change".
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=server/pyproject.toml,target=pyproject.toml \
    --mount=type=bind,source=server/uv.lock,target=uv.lock \
    uv sync --locked --no-dev --no-install-project

# server/pyproject.toml has `readme = "../README.md"`, so the README must sit
# one directory above WORKDIR for the project install in the next step.
COPY README.md /README.md
COPY server/ /app/

# Layer 2: install the project itself now that source is present.
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

EXPOSE 8765

CMD ["uv", "run", "python", "main.py", "--host", "0.0.0.0"]
