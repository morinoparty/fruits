FROM --platform=$TARGETOS/$TARGETARCH ubuntu:22.04

LABEL author="Nikomaru" maintainer="nikomaru@nikomaru.dev"
LABEL org.opencontainers.image.source="https://github.com/Nlkomaru/yolks"
# Install necessary dependencies
RUN apt-get update && \
    apt-get install -y wget tar

WORKDIR /opt
RUN wget https://github.com/graalvm/graalvm-ce-builds/releases/download/jdk-21.0.2/graalvm-community-jdk-21.0.2_linux-x64_bin.tar.gz && \
    tar -xvf graalvm-community-jdk-21.0.2_linux-x64_bin.tar.gz && \
    rm graalvm-community-jdk-21.0.2_linux-x64_bin.tar.gz && \
    ls

ENV JAVA_HOME=/opt/graalvm-community-jdk-21.0.2
ENV PATH=$JAVA_HOME/bin:$PATH

RUN apt-get update -y \
    && apt-get install -y lsof curl ca-certificates openssl git tar sqlite3 fontconfig libfreetype6 tzdata iproute2 libstdc++6 \
    && useradd -d /home/container -m container

USER container
ENV USER=container HOME=/home/container
WORKDIR /home/container

COPY ./../entrypoint.sh /entrypoint.sh
CMD [ "/bin/bash", "/entrypoint.sh" ]
