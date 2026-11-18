# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

FROM docker.io/bitnami/minideb:bookworm

ARG DOWNLOADS_URL="https://downloads.bitnami.com/files/stacksmith"
ARG JAVA_EXTRA_SECURITY_DIR="/bitnami/java/extra-security"
ARG LOGSTASH_PLUGINS
ARG TARGETARCH

LABEL org.opencontainers.image.base.name="docker.io/bitnami/minideb:bookworm" \
      org.opencontainers.image.created="2026-09-16T00:00:00Z" \
      org.opencontainers.image.description="Application packaged by Lyearn, based on Broadcom, Inc. Bitnami container. Bitnami's upstream 9.5/debian-12 recipe, with one deviation: the java component is NOT taken from stacksmith (which stops at java-21.0.12-10-0, vulnerable to CVE-2026-114938) - the JDK is fetched from Amazon Corretto 21.0.13.1.2, which carries the fix. The logstash-9.5.4-0 and yq components are still the stacksmith ones." \
      org.opencontainers.image.documentation="https://github.com/bitnami/containers/tree/main/bitnami/logstash/README.md" \
      org.opencontainers.image.ref.name="9.5.4-debian-12-r1" \
      org.opencontainers.image.source="https://github.com/bitnami/containers/tree/main/bitnami/logstash" \
      org.opencontainers.image.title="logstash" \
      org.opencontainers.image.vendor="Lyearn" \
      org.opencontainers.image.version="9.5.4"

ENV HOME="/" \
    OS_ARCH="${TARGETARCH:-amd64}" \
    OS_FLAVOUR="debian-12" \
    OS_NAME="linux"

COPY prebuildfs /
SHELL ["/bin/bash", "-o", "errexit", "-o", "nounset", "-o", "pipefail", "-c"]
# Install required system packages and dependencies
RUN install_packages ca-certificates curl procps zlib1g
RUN --mount=type=secret,id=downloads_url,env=SECRET_DOWNLOADS_URL \
    DOWNLOADS_URL=${SECRET_DOWNLOADS_URL:-${DOWNLOADS_URL}} ; \
    mkdir -p /tmp/bitnami/pkg/cache/ ; cd /tmp/bitnami/pkg/cache/ || exit 1 ; \
    COMPONENTS=( \
      "yq-4.53.6-2-linux-${OS_ARCH}-debian-12" \
      "logstash-9.5.4-0-linux-${OS_ARCH}-debian-12" \
    ) ; \
    for COMPONENT in "${COMPONENTS[@]}"; do \
      if [ ! -f "${COMPONENT}.tar.gz" ]; then \
        curl -SsLf "${DOWNLOADS_URL}/${COMPONENT}.tar.gz" -O ; \
      fi ; \
      sha256sum -c "/opt/bitnami/checksums/${COMPONENT}.tar.gz.sha256" ; \
      tar -zxf "${COMPONENT}.tar.gz" -C /opt/bitnami --strip-components=2 --no-same-owner ; \
      rm -rf "${COMPONENT}".tar.gz ; \
    done ; \
    rm -rf /opt/bitnami/checksums ;

# CVE-2026-114938 (Java4Shell) - JDK sourced from Corretto, not stacksmith.
#
# The vulnerable AVX-512 Base64 decode intrinsic affects OpenJDK >= 17.0.6; the
# fix for the 21 line is 21.0.13.1. As of 2026-11-18 the Bitnami stacksmith
# mirror publishes no java component past java-21.0.12-10-0 (verified by HEAD
# check across 21.0.13-*/21.0.14-*/21.0.15-*), and Temurin's newest GA 21 build
# is still jdk-21.0.13+9 - neither carries the patch. Amazon Corretto has
# shipped it as 21.0.13.1.2, so the JDK comes from Corretto directly, the same
# fetch-from-vendor approach already used by the elasticsearch images here.
#
# Corretto publishes only a GPG .sig (its .sha256 URL returns 403), so the
# digest below was computed locally from the downloaded artifact and pinned.
# Revert to a stacksmith "java-21.0.13.x" component once one is published.
ARG CORRETTO_VERSION="21.0.13.1.2"
ARG CORRETTO_SHA256_AMD64="2aaa2a9a43143b20ff0b28c8d0da27407b97dc40b160d9a2f60a26965815e948"
ARG CORRETTO_SHA256_ARM64="55751c47c7e3be3b33af43ccdcf3a6f4814d308390b7400477594d421bd16a53"
RUN case "${OS_ARCH}" in \
      amd64) CORRETTO_ARCH="x64" ; CORRETTO_SHA256="${CORRETTO_SHA256_AMD64}" ;; \
      arm64) CORRETTO_ARCH="aarch64" ; CORRETTO_SHA256="${CORRETTO_SHA256_ARM64}" ;; \
      *) echo "No pinned Corretto digest for OS_ARCH=${OS_ARCH} - add one before building this arch." >&2 ; exit 1 ;; \
    esac ; \
    cd /tmp || exit 1 ; \
    curl -SsLf -o corretto.tar.gz \
      "https://corretto.aws/downloads/resources/${CORRETTO_VERSION}/amazon-corretto-${CORRETTO_VERSION}-linux-${CORRETTO_ARCH}.tar.gz" ; \
    echo "${CORRETTO_SHA256}  corretto.tar.gz" | sha256sum -c - ; \
    rm -rf /opt/bitnami/java ; mkdir -p /opt/bitnami/java ; \
    tar -zxf corretto.tar.gz -C /opt/bitnami/java --strip-components=1 --no-same-owner ; \
    rm -f corretto.tar.gz ; \
    /opt/bitnami/java/bin/java -version
RUN apt-get update && apt-get upgrade -y && \
    apt-get clean && rm -rf /var/lib/apt/lists /var/cache/apt/archives
RUN chmod g+rwX /opt/bitnami
RUN find / -perm /6000 -type f -exec chmod a-s {} \; || true
RUN uninstall_packages curl

COPY rootfs /
RUN /opt/bitnami/scripts/java/postunpack.sh
RUN /opt/bitnami/scripts/logstash/postunpack.sh
ENV APP_VERSION="9.5.4" \
    BITNAMI_APP_NAME="logstash" \
    IMAGE_REVISION="1" \
    JAVA_HOME="/opt/bitnami/java" \
    PATH="/opt/bitnami/common/bin:/opt/bitnami/java/bin:/opt/bitnami/logstash/bin:$PATH"

EXPOSE 8080

WORKDIR /opt/bitnami/logstash
USER 1001
ENTRYPOINT [ "/opt/bitnami/scripts/logstash/entrypoint.sh" ]
CMD [ "/opt/bitnami/scripts/logstash/run.sh" ]
