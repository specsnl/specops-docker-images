# syntax=docker/dockerfile:1
# check=error=true

# Latest version: https://hub.docker.com/r/hashicorp/packer/tags?page=1&name=light
FROM hashicorp/packer:light-1.15.0

RUN apk add --no-cache --upgrade --no-progress \
        docker-cli=29.1.3-r3 \
    && packer plugins install github.com/hashicorp/docker \
    && packer plugins install github.com/hashicorp/ansible \
    && packer plugins install github.com/UpCloudLtd/upcloud

WORKDIR /workspace
