# syntax=docker/dockerfile:1
# check=error=true

# Latest version: https://hub.docker.com/r/hashicorp/packer/tags?page=1&name=light
FROM hashicorp/packer:light-1.14.3

RUN apk add --no-cache --upgrade --no-progress \
        docker-cli=28.3.3-r5 \
    && packer plugins install github.com/hashicorp/docker \
    && packer plugins install github.com/hashicorp/ansible \
    && packer plugins install github.com/UpCloudLtd/upcloud

WORKDIR /workspace
