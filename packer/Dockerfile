# syntax=docker/dockerfile:1
# check=error=true

# Latest version: https://hub.docker.com/r/hashicorp/packer/tags?page=1&name=light
FROM hashicorp/packer:light-1.15.4

ENV PACKER_CONFIG_DIR=/root/.config/packer
ENV PACKER_PLUGIN_PATH=${PACKER_CONFIG_DIR}/plugins

WORKDIR /workspace

RUN apk add --no-cache --upgrade --no-progress \
        docker-cli=29.5.2-r0 \
        ansible=13.0.0-r0 \
        openssh-client-default=10.2_p1-r0 \
        sshpass=1.10-r0 \
        rsync=3.4.3-r0 \
        git=2.52.0-r0 \
    && packer plugins install github.com/hashicorp/docker \
    && packer plugins install github.com/hashicorp/ansible \
    && packer plugins install github.com/UpCloudLtd/upcloud
