FROM ubuntu:focal AS logdevice-builder

RUN apt-get update && \
    DEBIAN_FRONTEND="noninteractive" apt-get install --no-install-recommends -y \
    curl cmake wget git g++-10 clang-10 make ninja-build lld \
    python3-dev python3 python3-pip\
    libboost-dev libboost-python-dev libboost-thread-dev libboost-regex-dev libboost-filesystem-dev \
    libboost-chrono-dev libboost-system-dev libboost-program-options-dev libboost-context-dev libevent-dev \
    zlib1g-dev libbz2-dev liblz4-dev libzookeeper-mt-dev libsqlite3-dev libssl-dev libunwind-dev \
    libdwarf-dev libelf-dev libsnappy-dev \ 
    libgoogle-glog-dev liblzma-dev libdouble-conversion-dev libicu-dev \
    libgflags-dev libsparsehash-dev libtbb-dev libnuma-dev libiberty-dev libsodium-dev libzstd-dev \
    autoconf autoconf-archive automake \
    pkg-config libtool bison flex \
    libkrb5-dev libcurl4-openssl-dev libjemalloc-dev && \
    update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 20 && \
    update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-10 20 && \
    update-alternatives --install /usr/bin/clang clang /usr/bin/clang-10 20 && \
    update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-10 20 && \
    rm -rf /var/lib/apt/lists/*

ARG PARALLEL
ENV CC=clang-10
ENV CXX=clang++-10

WORKDIR /deps

# jemalloc
ENV LIB_JEMALLOC_VSN=5.2.1
RUN curl --silent --show-error -kfL -o jemalloc.tar.gz "https://github.com/jemalloc/jemalloc/archive/refs/tags/${LIB_JEMALLOC_VSN}.tar.gz" && \
    mkdir -p jemalloc && \
    tar -zxf jemalloc.tar.gz -C jemalloc --strip-components 1 && \
    rm -f jemalloc.tar.gz
RUN cd /deps/jemalloc && ./autogen.sh --disable-initial-exec-tls --prefix=/usr && \
    make -j $(nproc) && \
    make install

# libfmt
ENV LIB_FMT_VSN=6.2.1
RUN curl --silent --show-error -kfL -o fmt.tar.gz "https://github.com/fmtlib/fmt/archive/refs/tags/${LIB_FMT_VSN}.tar.gz" && \
    mkdir -p fmt && \
    tar -zxf fmt.tar.gz -C fmt --strip-components 1 && \
    rm -f fmt.tar.gz
RUN cd fmt && \
    cmake -DCMAKE_BUILD_TYPE=Release \
          -DCMAKE_CXX_STANDARD=17 \
          -DCMAKE_POSITION_INDEPENDENT_CODE=True \
          -DFMT_TEST=OFF . && \
    make -j $(nproc) && make install

# libfolly
ENV LIB_FOLLY_VSN=v2021.11.15.00
RUN curl --silent --show-error -kfL -o folly.tar.gz "https://github.com/facebook/folly/archive/refs/tags/${LIB_FOLLY_VSN}.tar.gz" && \
    mkdir -p folly && \
    tar -zxf folly.tar.gz -C folly --strip-components 1 && \
    rm -f folly.tar.gz
RUN cd /folly && \
    cmake -DCMAKE_BUILD_TYPE=Release \
          -DCMAKE_CXX_STANDARD=17 \
          -DCMAKE_POSITION_INDEPENDENT_CODE=True \
          -DBUILD_SHARED_LIBS=ON \
          -DPYTHON_EXTENSIONS=OFF \
          # Folly doesn't build on clang without this flag \
          # ((https://github.com/facebook/folly/issues/976)) \
          -DFOLLY_USE_JEMALLOC=OFF \
          . && \
    make -j $(nproc) && make install

# libfizz
ENV LIB_FIZZ_VSN=v2021.11.15.00
RUN curl --silent --show-error -kfL -o fizz.tar.gz "https://github.com/facebookincubator/fizz/archive/refs/tags/${LIB_FIZZ_VSN}.tar.gz" && \
    mkdir -p fizz && \
    tar -zxf fizz.tar.gz -C fizz --strip-components 1 && \
    rm -f fizz.tar.gz
RUN cd fizz && \
    cmake -DCMAKE_BUILD_TYPE=Release \
          -DCMAKE_CXX_STANDARD=17 \
          -DCMAKE_POSITION_INDEPENDENT_CODE=True \
          -DBUILD_SHARED_LIBS=ON \
          -DBUILD_TESTS=OFF \
          -DBUILD_EXAMPLES=OFF \
          ./fizz && \
    make -j $(nproc) && make install && \
    cp build/fbcode_builder/CMake/FindSodium.cmake /usr/local/lib/cmake/FindSodium.cmake

# libwangle
ENV LIB_WANGLE_VSN=v2021.11.15.00
RUN curl --silent --show-error -kfL -o wangle.tar.gz "https://github.com/facebook/wangle/archive/refs/tags/${LIB_WANGLE_VSN}.tar.gz" && \
    mkdir -p wangle && \
    tar -zxf wangle.tar.gz -C wangle --strip-components 1 && \
    rm -f wangle.tar.gz
RUN cd wangle && \
    cmake -DCMAKE_BUILD_TYPE=Release \
          -DCMAKE_CXX_STANDARD=17 \
          -DCMAKE_POSITION_INDEPENDENT_CODE=True \
          -DBUILD_SHARED_LIBS=ON \
          -DBUILD_TESTS=OFF \
          ./wangle && \
    make -j $(nproc) && \
    make install

# fbthrift
#
# FIXME: cmake can not find python-fix
# - https://github.com/facebook/fbthrift/issues/376
# - https://github.com/facebook/fbthrift/issues/415
ENV LIB_FBTHRIFT_VSN=v2021.11.15.00
RUN curl --silent --show-error -kfL -o fbthrift.tar.gz "https://github.com/facebook/fbthrift/archive/refs/tags/${LIB_FBTHRIFT_VSN}.tar.gz" && \
    mkdir -p fbthrift && \
    tar -zxf fbthrift.tar.gz -C fbthrift --strip-components 1 && \
    rm -f fbthrift.tar.gz
RUN python3 /deps/fbthrift/build/fbcode_builder/getdeps.py build \
      --allow-system-packages python-six --install-dir /tmp/six && \
    mv /tmp/six/lib/cmake/python-six /usr/local/lib/cmake/ && \
    mv /tmp/six/lib/fb-py-libs /usr/local/lib/ && \
    rm -rf /tmp/*
# doesn't build on clang with c++17
# e.g. https://github.com/pybind/pybind11/issues/1818
RUN export CC=gcc-10 && \
    export CXX=g++-10 && \
    cd fbthrift && \
    cmake -DCMAKE_BUILD_TYPE=Release \
          -Dthriftpy3=OFF \
          -DCMAKE_CXX_STANDARD=17 \
          -DCMAKE_POSITION_INDEPENDENT_CODE=True \
          -DBUILD_SHARED_LIBS=ON \
          -DCMAKE_MODULE_PATH=/usr/local/lib/cmake/ \
          -Denable_tests=OFF \
          . && \
    make -j $(nproc) && make install

ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
WORKDIR /build

# FROM logdevice-builder AS logdevice-client

