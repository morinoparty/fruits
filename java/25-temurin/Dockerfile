FROM --platform=$TARGETOS/$TARGETARCH ubuntu:26.04

LABEL author="nikomaru" maintainer="nikomaru@nikomaru.dev"

LABEL org.opencontainers.image.source="https://github.com/morinoparty/fruits"

# Eclipse Temurin 25.0.5.1+2 from Adoptium. Installed from the tarball because the
# eclipse-temurin Docker images did not yet ship the CVE-2026-114938 fix (25.0.5.1+).
ARG JDK_URL=https://github.com/adoptium/temurin25-binaries/releases/download/jdk-25.0.5.1%2B2/OpenJDK25U-jdk_x64_linux_hotspot_25.0.5.1_2.tar.gz
ARG JDK_SHA256=760f91b22c3efc9bd1b64ff950598231be82928cc807e30691a0e7a2995a62c0

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
