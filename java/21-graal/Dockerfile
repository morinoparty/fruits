FROM --platform=$TARGETOS/$TARGETARCH ghcr.io/graalvm/jdk-community:21.0.2

LABEL author="Nikomaru" maintainer="nikomaru@nikomaru.dev"

LABEL org.opencontainers.image.source="https://github.com/Nlkomaru/yolks"

RUN microdnf install -y lsof curl ca-certificates openssl git tar sqlite fontconfig freetype tzdata iproute libstdc++ \
    && useradd -d /home/container -m container

USER container
ENV USER=container HOME=/home/container
WORKDIR /home/container

COPY ./../entrypoint.sh /entrypoint.sh
CMD [ "/bin/bash", "/entrypoint.sh" ]
