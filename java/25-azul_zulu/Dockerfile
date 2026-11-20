FROM --platform=$TARGETOS/$TARGETARCH ubuntu:26.04

LABEL author="nikomaru" maintainer="nikomaru@nikomaru.dev"

LABEL org.opencontainers.image.source="https://github.com/morinoparty/fruits"

# Azul Zulu 25.38.201 (JDK 25.0.5.1, fixes CVE-2026-114938). Installed from the tarball
# so that every image shares the same Ubuntu base; azul/zulu-openjdk is built on Ubuntu 22.04.
ARG JDK_URL=https://cdn.azul.com/zulu/bin/zulu25.38.201-ca-jdk25.0.5.1-linux_x64.tar.gz
ARG JDK_SHA256=fc7a3d9396b443134666b9cf7cd946b532dc1781f5350c725849ac50d16a7198

# Match the locale of the official eclipse-temurin / azul/zulu-openjdk images
ENV LANG=en_US.UTF-8 LANGUAGE=en_US:en LC_ALL=en_US.UTF-8

RUN apt-get update -y \
    && apt-get install -y locales lsof curl ca-certificates openssl git tar sqlite3 fontconfig libfreetype6 tzdata iproute2 libstdc++6 \
    && echo "en_US.UTF-8 UTF-8" > /etc/locale.gen \
    && locale-gen en_US.UTF-8 \
    && rm -rf /var/lib/apt/lists/* \
    && useradd -d /home/container -m container

# Verify the tarball against the vendor's published checksum before unpacking
RUN curl -fsSL -o /tmp/jdk.tar.gz "${JDK_URL}" \
    && echo "${JDK_SHA256}  /tmp/jdk.tar.gz" | sha256sum -c - \
    && mkdir -p /opt/java/jdk \
    && tar -xzf /tmp/jdk.tar.gz -C /opt/java/jdk --strip-components=1 \
    && rm /tmp/jdk.tar.gz

ENV JAVA_HOME=/opt/java/jdk
ENV PATH=$JAVA_HOME/bin:$PATH

USER container
ENV USER=container HOME=/home/container
WORKDIR /home/container

COPY ./../entrypoint.sh /entrypoint.sh
CMD [ "/bin/bash", "/entrypoint.sh" ]
