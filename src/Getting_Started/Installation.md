# Installation of MP-SPDZ
This tutorial will show you how to install MP-SPDZ on your system from source and precompiled binaries.
We also cover advanced configuration options that can be enabled during compilation.

## Source Installation
The source installation requires that some dependencies have a quite recent version.
Therefore, we assume that you use at least Ubuntu 24.04 for the sake of this tutorial.
If you are using any other distribution you have to make sure that the packages that you install meet the mentioned versions.

In a first step we are going to install all the necessary packages

```
apt-get update && apt-get install -y --no-install-recommends \
                automake \
                build-essential \
                ca-certificates \
                clang \
		        cmake \
                gdb \
                git \
                libboost-dev \
                libboost-filesystem-dev \
                libboost-iostreams-dev \
                libboost-thread-dev \
                libclang-dev \
                libgmp-dev \
                libntl-dev \
                libsodium-dev \
                libssl-dev \
                libtool \
                openssl \
                python3 \
                valgrind \
                vim 
```

with some needing to meet the following criteria in order to work properly.

```
- Clang 10 or later
- Cmake 3.15 or later
- Boost 1.75 or later
- GMP with C++ support
- Python 3.5 or later
```

After installing the build requirements we are going to build MP-SPDZ using the latest version which currently is `0.4.0`.

```
git clone https://github.com/data61/MP-SPDZ.git
cd MP-SPDZ
git checkout v0.4.0
make -j $(nproc)
```

### Configuration
As MP-SPDZ includes a variety of protocols a user might not need to build everything that MP-SPDZ is offering.
For this purpose the compilation can be limited to a single protocol and additional configuration options can be used.

For example some protocols do not implement an offline phase or the user might want to benchmark protocols with a fake offline phase.
This can be achieved through adding `MY_CFLAGS += -DINSECURE` to either `CONFIG` or `CONFIG.mine`.
Other advanced configuration options are for the usage of homomorphic encryption using `GF(2^40)` which can be done through setting `USE_NTL = 1` or the usage of KOS through `USE_KOS = 1` and `SECURE = -DINSECURE`.

Whenever a change to the configuration files is done `make clean` needs to be run before compiling again with the new changes.

Single protocols like `mascot` can be compiled using `make mascot`.
This can be used in cases where one is only interested in a small subset of protocols.

## Binary Installation
We first retrieve the latest version from the Github release page.
In our case this is `v0.4.0` and we can use `wget https://github.com/data61/MP-SPDZ/releases/download/v0.4.0/mp-spdz-0.4.0.tar.xz` for this.
Afterwards the archive needs to be extracted using `tar xf mp-spdz-0.4.0.tar.xz`.

From the top level folder execute `Scripts/tldr.sh` in order to place the binaries at their correct location.
Afterwards you have a ready to use installation.