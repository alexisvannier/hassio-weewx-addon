ARG BUILD_FROM=alpine:3.17.3
ARG PYTHON_VERSION=3.11.2
ARG WEEWX_UID=421
ARG WEEWX_VERSION=4.10.2
ARG WEEWX_HOME="/home/weewx"
# hadolint ignore=DL3006
FROM ${BUILD_FROM}

# Environment variables
ENV \
    CARGO_NET_GIT_FETCH_WITH_CLI=true \
    HOME="/root" \
    LANG="C.UTF-8" \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_FIND_LINKS=https://wheels.home-assistant.io/musllinux/ \
    PIP_NO_CACHE_DIR=1 \
    PIP_PREFER_BINARY=1 \
    PS1="$(whoami)@$(hostname):$(pwd)$ " \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_CMD_WAIT_FOR_SERVICES=1 \
    YARN_HTTP_TIMEOUT=1000000 \
    TERM="xterm-256color" 

# Set shell
SHELL ["/bin/ash", "-o", "pipefail", "-c"]

# Install base system
ARG BUILD_ARCH=amd64
ARG BASHIO_VERSION="v0.14.3"
ARG S6_OVERLAY_VERSION="3.1.5.0"
ARG TEMPIO_VERSION="2021.09.0"
RUN \
    set -o pipefail \
    \
    && apk add --no-cache --virtual .build-dependencies \
        tar=1.34-r2 \
        xz=5.2.9-r0 \
    \
    && apk add --no-cache \
        libcrypto3=3.0.8-r4 \
        libssl3=3.0.8-r4 \
        musl-utils=1.2.3-r4 \
        musl=1.2.3-r4 \
    \
    && apk add --no-cache \
        bash=5.2.15-r0 \
        curl=8.0.1-r0 \
        jq=1.6-r2 \
        tzdata=2023c-r0 \
    \
    && S6_ARCH="${BUILD_ARCH}" \
    && if [ "${BUILD_ARCH}" = "i386" ]; then S6_ARCH="i686"; \
    elif [ "${BUILD_ARCH}" = "amd64" ]; then S6_ARCH="x86_64"; \
    elif [ "${BUILD_ARCH}" = "armv7" ]; then S6_ARCH="arm"; fi \
    \
    && curl -L -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz" \
        | tar -C / -Jxpf - \
    \
    && curl -L -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz" \
        | tar -C / -Jxpf - \
    \
    && curl -L -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-symlinks-noarch.tar.xz" \
        | tar -C / -Jxpf - \
    \
    && curl -L -s "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-symlinks-arch.tar.xz" \
        | tar -C / -Jxpf - \
    \
    && curl -J -L -o /tmp/bashio.tar.gz \
        "https://github.com/hassio-addons/bashio/archive/${BASHIO_VERSION}.tar.gz" \
    && mkdir /tmp/bashio \
    && tar zxvf \
        /tmp/bashio.tar.gz \
        --strip 1 -C /tmp/bashio \
    \
    && mv /tmp/bashio/lib /usr/lib/bashio \
    && ln -s /usr/lib/bashio/bashio /usr/bin/bashio \
    \
    && curl -L -s -o /usr/bin/tempio \
        "https://github.com/home-assistant/tempio/releases/download/${TEMPIO_VERSION}/tempio_${BUILD_ARCH}" \
    && chmod a+x /usr/bin/tempio \
    \
    && apk del --no-cache --purge .build-dependencies \
    && rm -f -r \
        /tmp/*

# Copy root filesystem
COPY rootfs /

# Copy s6-overlay adjustments
COPY s6-overlay /package/admin/s6-overlay-${S6_OVERLAY_VERSION}/

# Entrypoint & CMD
ENTRYPOINT ["/init"]

# Build arugments
ARG BUILD_DATE
ARG BUILD_REF
ARG BUILD_VERSION
ARG BUILD_REPOSITORY

# Labels
LABEL \
    io.hass.name="Addon base for ${BUILD_ARCH}" \
    io.hass.description="Home Assistant Community Add-on: ${BUILD_ARCH} base image" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="base" \
    io.hass.version=${BUILD_VERSION} \
    io.hass.base.version=${BUILD_VERSION} \
    io.hass.base.name="alpine" \
    io.hass.base.image="hassioaddons/base" \
    maintainer="Franck Nijhof <frenck@addons.community>" \
    org.opencontainers.image.title="Addon base for ${BUILD_ARCH}" \
    org.opencontainers.image.description="Home Assistant Community Add-on: ${BUILD_ARCH} Base image" \
    org.opencontainers.image.vendor="Home Assistant Community Add-ons" \
    org.opencontainers.image.authors="Franck Nijhof <frenck@addons.community>" \
    org.opencontainers.image.licenses="MIT" \
    org.opencontainers.image.url="https://addons.community" \
    org.opencontainers.image.source="https://github.com/${BUILD_REPOSITORY}" \
    org.opencontainers.image.documentation="https://github.com/${BUILD_REPOSITORY}/blob/main/README.md" \
    org.opencontainers.image.created=${BUILD_DATE} \
    org.opencontainers.image.revision=${BUILD_REF} \
    org.opencontainers.image.version=${BUILD_VERSION}


# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM tonistiigi/xx AS xx

FROM --platform=$BUILDPLATFORM python:${PYTHON_VERSION} as build-stage

ARG WEEWX_VERSION
ARG ARCHIVE="weewx-${WEEWX_VERSION}.tar.gz"

COPY --from=xx / /
RUN apt-get update && apt-get install -y clang lld
ARG TARGETPLATFORM
RUN xx-apt install -y libc6-dev

# RUN apk --no-cache add cargo gcc libffi-dev make musl-dev openssl-dev python3-dev tar
RUN apt-get install -y wget

WORKDIR /tmp
RUN \
  --mount=type=cache,mode=0777,target=/var/cache/apt \
  --mount=type=cache,mode=0777,target=/root/.cache/pip <<EOF
apt-get update
python -m pip install --upgrade pip
pip install --upgrade virtualenv
virtualenv /opt/venv
EOF

COPY src/hashes README.md requirements.txt setup.py ./
COPY src/_version.py ./src/_version.py

# Download sources and verify hashes
RUN wget -O "${ARCHIVE}" "https://weewx.com/downloads/released_versions/${ARCHIVE}"
RUN wget -O weewx-kl.zip https://github.com/matthewwall/weewx-klimalogg/archive/master.zip
RUN wget -O weewx-mqtt.zip https://github.com/matthewwall/weewx-mqtt/archive/master.zip
RUN wget -O weewx-interceptor.zip https://github.com/matthewwall/weewx-interceptor/archive/master.zip
RUN sha256sum -c < hashes

# WeeWX setup
RUN tar --extract --gunzip --directory /root --strip-components=1 --file "${ARCHIVE}"

# Python setup
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache --requirement requirements.txt

WORKDIR /root

RUN bin/wee_extension --install /tmp/weewx-kl.zip
RUN bin/wee_extension --install /tmp/weewx-mqtt.zip
RUN bin/wee_extension --install /tmp/weewx-interceptor.zip
COPY src/entrypoint.sh src/_version.py ./

FROM python:${PYTHON_VERSION}-slim-bullseye as final-stage

ARG TARGETPLATFORM
ARG WEEWX_HOME
ARG WEEWX_UID

# For a list of pre-defined annotation keys and value types see:
# https://github.com/opencontainers/image-spec/blob/master/annotations.md
# Note: Additional labels are added by the build workflow.
LABEL org.opencontainers.image.authors="markf+github@geekpad.com"
LABEL org.opencontainers.image.authors="AlxV"
LABEL com.weewx.version=${WEEWX_VERSION}

RUN addgroup --system --gid ${WEEWX_UID} weewx \
  && adduser --system --uid ${WEEWX_UID} --ingroup weewx weewx

RUN apt-get update && apt-get install -y libusb-1.0-0 gosu busybox-syslogd tzdata

WORKDIR ${WEEWX_HOME}

COPY --from=build-stage /opt/venv /opt/venv
COPY --from=build-stage /root ${WEEWX_HOME}

RUN mkdir /data && \
  cp weewx.conf /data && \
  chown -R weewx:weewx ${WEEWX_HOME}

VOLUME ["/data"]

ENV PATH="/opt/venv/bin:$PATH"
ENTRYPOINT ["./entrypoint.sh"]
CMD ["/data/weewx.conf"]
