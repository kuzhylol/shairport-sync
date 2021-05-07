FROM alpine:3.12 AS builder-base
# Version 3.12 is required for now since libplist has breaking changes in v3.13 and above which are not yet standard
# in debian/ubuntu.

# General Build System:
RUN apk -U add \
        git \
        build-base \
        autoconf \
        automake \
        libtool \
        dbus \
        su-exec \
        alsa-lib-dev \
        libdaemon-dev \
        popt-dev \
        mbedtls-dev \
        soxr-dev \
        avahi-dev \
        libconfig-dev \
        libsndfile-dev \
        mosquitto-dev \
        xmltoman \
        openssh-client \
        libsodium-dev \
        ffmpeg-dev \
        xxd \
        libressl-dev \
        openssl-dev \
        libplist-dev

# ALAC Build System:
FROM builder-base AS builder-alac

RUN git clone https://github.com/mikebrady/alac
WORKDIR /alac
RUN autoreconf -fi
RUN ./configure
RUN make
RUN make install

# Shairport Sync Build System:
FROM builder-base AS builder-sps

# This may be modified by the Github Action Workflow.
ARG SHAIRPORT_SYNC_BRANCH=development

COPY --from=builder-alac /usr/local/lib/libalac.* /usr/local/lib/
COPY --from=builder-alac /usr/local/lib/pkgconfig/alac.pc /usr/local/lib/pkgconfig/alac.pc
COPY --from=builder-alac /usr/local/include /usr/local/include

# TODO: Remove when no longer pulling from private repos.
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

RUN --mount=type=ssh,id=github git clone git@github.com:charlesomer/nqptp.git
WORKDIR /nqptp
# Use development branch for now.
RUN git checkout development
RUN autoreconf -fi
RUN ./configure
RUN make
RUN make install
WORKDIR /

RUN --mount=type=ssh,id=github git clone git@github.com:aillwee/shairport-sync.git
WORKDIR /shairport-sync
RUN git checkout "$SHAIRPORT_SYNC_BRANCH"
RUN autoreconf -fi
RUN git submodule init
RUN git submodule update
# RUN ./configure \
#         --with-alsa \
#         --with-dummy \
#         --with-pipe \
#         --with-stdout \
#         --with-avahi \
#         --with-ssl=mbedtls \
#         --with-soxr \
#         --sysconfdir=/etc \
#         --with-dbus-interface \
#         --with-mpris-interface \
#         --with-mqtt-client \
#         --with-apple-alac \
#         --with-convolution \
#         --with-airplay-2
# RUN make -j $(nproc)
RUN CFLAGS="-O0 -g" CXXFLAGS="-O0 -g" ./configure --sysconfdir=/etc --with-metadata --with-dummy --with-pipe \
        --with-alsa --with-soxr --with-avahi --with-ssl=openssl --with-dbus-interface --with-stdout \
        --with-mpris-interface --with-mqtt-client --with-apple-alac --with-convolution --with-airplay-2
RUN make -j
RUN make install

# Shairport Sync Runtime System:
FROM alpine:3.12

RUN apk add \
        alsa-lib \
        dbus \
        popt \
        glib \
        mbedtls \
        soxr \
        avahi \
        libconfig \
        libsndfile \
        mosquitto-libs \
        su-exec \
        libuuid \
        ffmpeg-dev \
        ffmpeg-libs \
        libsodium-dev \
        libplist-dev

RUN rm -rf  /lib/apk/db/*

COPY --from=builder-alac /usr/local/lib/libalac.* /usr/local/lib/
COPY --from=builder-sps /etc/shairport-sync* /etc/
COPY --from=builder-sps /etc/dbus-1/system.d/shairport-sync-dbus.conf /etc/dbus-1/system.d/
COPY --from=builder-sps /etc/dbus-1/system.d/shairport-sync-mpris.conf /etc/dbus-1/system.d/
COPY --from=builder-sps /usr/local/bin/shairport-sync /usr/local/bin/shairport-sync
COPY --from=builder-sps /usr/local/bin/nqptp /usr/local/bin/nqptp

# Create non-root user for running the container -- running as the user 'shairport-sync' also allows
# Shairport Sync to provide the D-Bus and MPRIS interfaces within the container

RUN addgroup shairport-sync 
RUN adduser -D shairport-sync -G shairport-sync

# Add the shairport-sync user to the pre-existing audio group, which has ID 29, for access to the ALSA stuff
RUN addgroup -g 29 docker_audio && addgroup shairport-sync docker_audio && addgroup shairport-sync audio

COPY ./docker/shairport-sync.conf /etc/shairport-sync.conf
COPY ./docker/start.sh /

RUN chmod +x ./start.sh

ENTRYPOINT [ "/start.sh" ]