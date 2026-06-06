# syntax=docker/dockerfile:1
# check=error=true


# Latest version: https://hub.docker.com/r/hashicorp/packer/tags?name=light
FROM hashicorp/packer:light-1.15.4 AS packer

# Latest version: https://github.com/astral-sh/uv/releases
FROM ghcr.io/astral-sh/uv:0.11.19 AS uv

# Latest version: https://hub.docker.com/_/python/tags?name=3.14.5-alpine
FROM python:3.14.5-alpine3.23

ENV PACKER_CONFIG_DIR=/root/.config/packer
ENV PACKER_PLUGIN_PATH=${PACKER_CONFIG_DIR}/plugins

COPY --from=packer /bin/packer /bin/packer
COPY --from=uv /uv /usr/local/bin/uv

WORKDIR /workspace

COPY packer/pyproject.toml packer/uv.lock ./

# hadolint ignore=DL3018
RUN apk add --no-cache --no-progress \
        bash \
        docker-cli \
        openssh-client-default \
        sshpass \
        rsync \
        git \
    && uv sync --frozen --no-dev \
    && rm pyproject.toml uv.lock \
    && packer plugins install github.com/hashicorp/docker \
    && packer plugins install github.com/hashicorp/ansible \
    && packer plugins install github.com/UpCloudLtd/upcloud

ENV PATH="/workspace/.venv/bin:$PATH"
