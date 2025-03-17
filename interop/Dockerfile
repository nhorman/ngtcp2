FROM --platform=$BUILDPLATFORM tonistiigi/xx:1.5.0 AS xx

FROM --platform=$BUILDPLATFORM ubuntu:24.04 AS build

COPY --from=xx --link / /

ARG TARGETPLATFORM

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive \
        apt-get install -y --no-install-recommends \
        git clang-18 make autoconf automake autotools-dev libtool dpkg-dev \
        pkg-config ca-certificates && \
    xx-info debian-arch && \
    DEBIAN_FRONTEND=noninteractive \
        xx-apt-get install -y --no-install-recommends \
        gcc-14 g++-14 libc6-dev binutils \
        libev-dev libjemalloc-dev && \
    ln -s /usr/bin/clang-18 /usr/bin/clang && \
    ln -s /usr/bin/clang++-18 /usr/bin/clang++

#RUN if [ "$TARGETPLATFORM" = "linux/amd64" ]; then export WSSLFLAGS="--enable-aesni"; fi && \
#    if [ "$TARGETPLATFORM" = "linux/arm64" ]; then export WSSLFLAGS="--enable-armasm"; fi && \
#    git clone --depth 1 -b v5.7.4-stable https://github.com/wolfSSL/wolfssl && \
#    cd wolfssl && autoreconf -i && \
#    ./configure --disable-dependency-tracking --enable-static --enable-all \
#        --enable-harden --enable-keylog-export --disable-ech \
#        --enable-ticket-nonce-malloc $WSSLFLAGS \
#        --host=$(xx-clang --print-target-triple) \
#        CC=$(xx-info)-gcc-14 \
#        CXX=$(xx-info)-g++-14 && \
#    make -j$(nproc) && make install && cd .. && rm -rf wolfssl

RUN git clone --depth 1 https://github.com/openssl/openssl && \
    cd openssl && \
    CC=clang ./Configure enable-sslkeylog enable-fips disable-docs --prefix=/usr --openssldir=/etc/pki/tls && \
    make -j 4 && make install && \
    rm -rf /openssl

RUN git clone --recursive --shallow-submodules --depth 1 https://github.com/ngtcp2/nghttp3 && \
    cd nghttp3 && \
    autoreconf -i && \
    ./configure --disable-dependency-tracking --enable-lib-only \
        --host=$(xx-clang --print-target-triple) \
        CC=$(xx-info)-gcc-14 \
        CXX=$(xx-info)-g++-14 \
        LDFLAGS="-fsanitize=address,undefined -fno-sanitize-recover=undefined" \
        CPPFLAGS="-fsanitize=address,undefined -fno-sanitize-recover=undefined -g3" && \
    make -j$(nproc) && make install && cd .. && rm -rf nghttp3

ARG NGTCP2_BRANCH=openssl-interop

RUN git clone --recursive --shallow-submodules --depth 1 -b $NGTCP2_BRANCH https://github.com/nhorman/ngtcp2 && \
    cd ngtcp2 && \
    autoreconf -i && \
    ./configure \
        --host=$(xx-clang --print-target-triple) \
        CC=$(xx-info)-gcc-14 \
        CXX=$(xx-info)-g++-14 \
        LIBTOOL_LDFLAGS="-static-libtool-libs" \
        LDFLAGS="-fsanitize=address,undefined -fno-sanitize-recover=undefined -static-libgcc -static-libstdc++ -static-libasan -static-libubsan" \
        CPPFLAGS="-fsanitize=address,undefined -fno-sanitize-recover=undefined -g3" \
        LIBEV_LIBS="-l:libev.a" \
        JEMALLOC_LIBS="-l:libjemalloc.a" \
        PKG_CONFIG_PATH=/usr/lib64/pkgconfig:/usr/lib/pkgconfig \
        --disable-dependency-tracking \
        --with-openssl && \
    make -j$(nproc) && \
    cp examples/opensslclient examples/opensslserver \
        examples/h09opensslclient examples/h09opensslserver /usr/local/bin && \
    rm -rf ngtcp2

FROM martenseemann/quic-network-simulator-endpoint:latest

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        media-types

COPY --from=build --link /usr/local/bin/opensslclient /usr/local/bin/opensslserver \
    /usr/local/bin/h09opensslclient /usr/local/bin/h09opensslserver /usr/local/bin/


COPY --from=build --link /usr/lib64/libcrypto.so /usr/lib64/libcrypto.so.3 \
   /usr/lib64/libssl.so /usr/lib64/libssl.so.3 /usr/lib64
RUN mkdir /usr/lib64/ossl-modules
RUN mkdir /usr/lib64/engines-3
COPY --from=build --link /usr/lib64/ossl-modules/ /usr/lib64/ossl-modules
COPY --from=build --link /usr/lib64/engines-3/ /usr/lib64/engines-3

COPY --link --chmod=0775 run_endpoint.sh .

ENTRYPOINT [ "./run_endpoint.sh" ]
