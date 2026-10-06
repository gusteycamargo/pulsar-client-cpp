ARG NODE_IMAGE=node:24.21-alpine

FROM ${NODE_IMAGE} AS cpp

ARG CPP_REPOSITORY=https://github.com/gusteycamargo/pulsar-client-cpp.git
ARG CPP_REF=9666a9b
ARG BUILD_JOBS=4

RUN apk add --no-cache build-base linux-headers bash curl curl-dev cmake perl \
      git ninja zip unzip pkgconf python3 openssl-dev

RUN git clone "$CPP_REPOSITORY" /src \
 && git -C /src checkout "$CPP_REF" \
 && git -C /src submodule update --init vcpkg

WORKDIR /src

ENV VCPKG_FORCE_SYSTEM_BINARIES=1

RUN sed -i 's/-DCMAKE_BUILD_TYPE=Release/-DVCPKG_LIBCURL_DLSYM=OFF -DCMAKE_BUILD_TYPE=Release/' \
      vcpkg/scripts/bootstrap.sh

RUN cmake -B build -DINTEGRATE_VCPKG=ON -DCMAKE_BUILD_TYPE=Release \
      -DBUILD_TESTS=OFF -DBUILD_PERF_TOOLS=OFF \
      -DBUILD_DYNAMIC_LIB=OFF -DBUILD_STATIC_LIB=ON \
      -DCMAKE_INSTALL_PREFIX=/opt/pulsar-cpp \
 && cmake --build build -j"$BUILD_JOBS" \
 && build-support/merge_archives_vcpkg.sh "$PWD/build" \
 && cmake --install build \
 && cp build/libpulsarwithdeps.a /opt/pulsar-cpp/lib/

FROM ${NODE_IMAGE} AS addon

ARG PULSAR_CLIENT_NODE_VERSION=1.17.0

RUN apk add --no-cache python3 make g++

WORKDIR /addon

RUN npm pack "pulsar-client@${PULSAR_CLIENT_NODE_VERSION}" \
 && tar xzf pulsar-client-*.tgz --strip-components=1 \
 && rm pulsar-client-*.tgz

COPY --from=cpp /opt/pulsar-cpp pkg/linux/pulsar-cpp

RUN npm install --ignore-scripts --no-audit --no-fund \
 && npx node-pre-gyp rebuild

FROM scratch

COPY --from=addon /addon/lib/binding/pulsar.node /pulsar.node
