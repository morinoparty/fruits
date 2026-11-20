FROM --platform=$TARGETOS/$TARGETARCH ubuntu:26.04

LABEL author="nikomaru" maintainer="nikomaru@nikomaru.dev"

LABEL org.opencontainers.image.source="https://github.com/morinoparty/fruits"

# GraalVM Community 25.0.5.0.3 (graal-25.5.5.0.3). No GraalVM CE build with the
# CVE-2026-114938 fix existed when this was written, so this image is still affected.
# GraalVM is not offered by the eggs; bump this once a fixed release is published.
ARG JDK_URL=https://github.com/graalvm/graalvm-ce-builds/releases/download/graal-25.5.5.0.3/graalvm-community-jdk-25i5-25.0.5.0.3_linux-x64_bin.tar.gz
ARG JDK_SHA256=c6438dee6c3ade2220d40b7ccdcad24b512d30e233e61bb4fc70519522de34ef

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
