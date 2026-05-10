# ==============================================================================
# 多阶段构建的生产 Dockerfile
# Stage 1 (builder)：装所有依赖到指定目录
# Stage 2 (runtime)：只复制装好的依赖和代码，不带编译工具
# ==============================================================================

# ========== Stage 1：构建阶段 ==========
FROM python:3.10-slim AS builder

ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /build

# 装编译工具（某些 Python 包装时需要 gcc）
# 这些工具只存在于 builder 阶段，不会进入最终镜像
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# 关键：用 --target 把 pip 装的包放到固定目录
# 之后可以用 COPY --from 把这个目录拷到 runtime 阶段
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/install -r requirements.txt


# ========== Stage 2：运行阶段 ==========
FROM python:3.10-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# runtime 阶段只装运行时必需的系统依赖
# 注意没有 build-essential
RUN apt-get update && apt-get install -y --no-install-recommends \
    libglib2.0-0 \
    libsm6 \
    libxext6 \
    libxrender-dev \
    libgomp1 \
    libgl1 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# --from=builder 表示从 builder 阶段复制
# 把装好的 Python 包复制到标准 site-packages 位置
COPY --from=builder /install /usr/local/lib/python3.10/site-packages

# 再复制源码
COPY src/ ./src/
COPY tests/ ./tests/
COPY examples/ ./examples/

ENV PYTHONPATH=/app

# 健康检查：每 30 秒验证一次 OpenCV 能否正常导入
# 容器状态会显示 healthy/unhealthy
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD python -c "import cv2; import numpy" || exit 1

CMD ["pytest", "tests/", "-v"]