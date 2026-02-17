FROM ghcr.io/actions/actions-runner:latest

ARG MELANGE_VERSION=0.41.1
ARG APKO_VERSION=1.1.6
ARG TARGETARCH=amd64

USER root

RUN apt-get update && apt-get install -y \
    bubblewrap \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN curl -sL "https://github.com/chainguard-dev/melange/releases/download/v${MELANGE_VERSION}/melange_${MELANGE_VERSION}_linux_${TARGETARCH}.tar.gz" \
    | tar xz --strip-components=1 -C /usr/local/bin "melange_${MELANGE_VERSION}_linux_${TARGETARCH}/melange"

RUN curl -sL "https://github.com/chainguard-dev/apko/releases/download/v${APKO_VERSION}/apko_${APKO_VERSION}_linux_${TARGETARCH}.tar.gz" \
    | tar xz --strip-components=1 -C /usr/local/bin "apko_${APKO_VERSION}_linux_${TARGETARCH}/apko"

USER runner
