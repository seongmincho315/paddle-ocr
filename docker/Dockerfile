# syntax=docker/dockerfile:1.7

########################################
# 0) 공통 ARG
########################################
ARG PY_VER=3.10
ARG DEBIAN_DIST=bookworm
ARG APP_DIR=/app
# Paddle GPU wheel 있는 곳 (변수화)
ARG PADDLE_EXTRA_INDEX_URL="https://www.paddlepaddle.org.cn/packages/stable/cu126/"

########################################
# 1) Base (OS + uv + supervisor)
########################################
FROM python:${PY_VER}-slim-${DEBIAN_DIST} AS base

ARG APP_DIR
ENV APP_DIR=${APP_DIR}

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        tini \
        supervisor \
        curl \
        ca-certificates \
        unzip \
        libgl1 \
        libglib2.0-0 \
        libgomp1 \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

RUN mkdir -p ${APP_DIR}
WORKDIR ${APP_DIR}

########################################
# 2) deps (pyproject/uv.lock → venv)
########################################
FROM base AS deps

ARG APP_DIR
ARG PADDLE_EXTRA_INDEX_URL
WORKDIR ${APP_DIR}

COPY pyproject.toml uv.lock README.md ${APP_DIR}/

RUN uv venv ${APP_DIR}/.venv
ENV VIRTUAL_ENV=${APP_DIR}/.venv
ENV PATH="${VIRTUAL_ENV}/bin:${PATH}"

# paddlepaddle-gpu는 PyPI가 아니라 전용 인덱스에만 있어서 extra-index로 같이 봐야 함.
# --frozen을 안 쓰는 이유: 인덱스 우선순위가 고정돼 있지 않으면 파일 크기/버전이 안 맞을 수 있어
# unsafe-best-match로 두 인덱스를 다 뒤지게 함 (레퍼런스: doc_parser genon/serving/paddle).
RUN --mount=type=cache,target=/root/.cache/uv \
    UV_EXTRA_INDEX_URL="${PADDLE_EXTRA_INDEX_URL}" \
    uv sync --index-strategy unsafe-best-match --python ${VIRTUAL_ENV}/bin/python

########################################
# 3) models (OCR 모델 가중치를 이미지에 구워둠)
########################################
FROM deps AS models

ARG APP_DIR
ENV MODEL_DIR=/models

COPY resources/paddleocr_model.zip /tmp/paddleocr_model.zip

RUN mkdir -p ${MODEL_DIR} && \
    unzip -q -o /tmp/paddleocr_model.zip -d ${MODEL_DIR} && \
    rm -f /tmp/paddleocr_model.zip

########################################
# 4) runtime (최종)
########################################
FROM base AS runtime

ARG APP_DIR
ENV VIRTUAL_ENV=${APP_DIR}/.venv
ENV PATH="${VIRTUAL_ENV}/bin:${PATH}"

WORKDIR ${APP_DIR}

COPY --from=deps ${VIRTUAL_ENV} ${VIRTUAL_ENV}
COPY --from=models /models /models
COPY config ${APP_DIR}/config
COPY docker ${APP_DIR}/docker
COPY etc ${APP_DIR}/etc

RUN ln -sf ${APP_DIR}/config/supervisord.conf /etc/supervisor/conf.d/supervisord.conf && \
    chmod +x ${APP_DIR}/etc/*.sh

EXPOSE 8080

CMD ["supervisord", "-n"]
