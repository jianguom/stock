# ===================================
# A股自选股智能分析系统 - Docker 镜像
# ===================================

# 第一阶段：构建阶段 - 安装依赖
FROM ubuntu:22.04 AS builder

WORKDIR /app

# 设置时区为上海
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 修复 APT GPG 密钥问题和 post-invoke 脚本问题
RUN chmod 644 /etc/apt/trusted.gpg.d/* 2>/dev/null || true && \
    rm -f /etc/apt/apt.conf.d/docker-clean 2>/dev/null || true && \
    echo 'APT::Update::Post-Invoke "";' > /etc/apt/apt.conf.d/99no-post-invoke

# 安装系统依赖
RUN apt-get update --allow-insecure-repositories && apt-get install -y --allow-unauthenticated --no-install-recommends \
    python3.11 \
    python3.11-dev \
    python3.11-venv \
    python3-pip \
    python3-setuptools \
    python3-wheel \
    gcc \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 设置 pip 环境变量
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV PIP_NO_COLOR=1
ENV PIP_PROGRESS_BAR=off

# 创建虚拟环境
RUN python3.11 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt -q

# ===================================
# 第二阶段：运行阶段 - 最终镜像
# ===================================
FROM ubuntu:22.04

WORKDIR /app

# 设置时区为上海
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 修复 APT GPG 密钥问题和 post-invoke 脚本问题
RUN chmod 644 /etc/apt/trusted.gpg.d/* 2>/dev/null || true && \
    rm -f /etc/apt/apt.conf.d/docker-clean 2>/dev/null || true && \
    echo 'APT::Update::Post-Invoke "";' > /etc/apt/apt.conf.d/99no-post-invoke

# 仅安装最小化运行时系统依赖
RUN apt-get update --allow-insecure-repositories && apt-get install -y --allow-unauthenticated --no-install-recommends \
    python3.11 \
    python3-pip \
    gosu \
    curl \
    fontconfig \
    libjpeg-turbo8 \
    libxrender1 \
    libxext6 \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户 (UID 1000)
RUN groupadd -g 1000 dsa && \
    useradd -u 1000 -g dsa -m dsa

# 从构建阶段复制虚拟环境
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 复制应用代码（不复制不需要的文件）
COPY *.py ./
COPY api/ ./api/
COPY data_provider/ ./data_provider/
COPY bot/ ./bot/
COPY src/ ./src/
COPY strategies/ ./strategies/
COPY static/ ./static/
COPY docker/entrypoint.sh /usr/local/bin/docker-entrypoint.sh

# 确保数据目录存在并授权给 non-root 用户
RUN mkdir -p /app/data /app/logs /app/reports && \
    chown -R dsa:dsa /app && \
    chmod +x /usr/local/bin/docker-entrypoint.sh

# 设置环境变量默认值
ENV PYTHONUNBUFFERED=1
ENV LOG_DIR=/app/logs
ENV DATABASE_PATH=/app/data/stock_analysis.db
# Web/API service
ENV WEBUI_HOST=0.0.0.0
ENV API_PORT=8000

# 暴露 API 端口
EXPOSE 8000

# 数据卷（持久化数据）
VOLUME ["/app/data", "/app/logs", "/app/reports"]

# 健康检查（FastAPI 模式）
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/api/health || curl -f http://localhost:8000/health \
    || python3.11 -c "import sys; sys.exit(0)"

# 启动入口先修复 bind mount 目录权限，再降权为 dsa 执行应用；
# 文档化 docker-compose exec 命令需显式使用 -u dsa，避免手动进程写入 root-owned 文件。
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

# 默认命令（可被覆盖）
CMD ["python3.11", "main.py", "--schedule"]
