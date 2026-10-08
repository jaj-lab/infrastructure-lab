FROM debian:13-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ansible \
        openssh-client \
        git \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*
