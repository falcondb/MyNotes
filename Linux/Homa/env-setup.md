## Homa kernel module
### Linux kernel
Upgrade the kernel version to v5.18
Tried to compile the Homa module on v5.4, some functions are missing in early versions


## Homa gRPC
### Build environment setup
The build script (Makefile) in gRPC (as May 2023) needs customization. It also depends on gRPC

#### gRPC setup
I referred to [gRPC Quick start](https://grpc.io/docs/languages/cpp/quickstart/)

```
### Change the .local
export MY_INSTALL_DIR=$HOME/.local
mkdir -p $MY_INSTALL_DIR
export PATH="$MY_INSTALL_DIR/bin:$PATH"


git clone --recurse-submodules -b v1.54.0 --depth 1 --shallow-submodules https://github.com/grpc/grpc

cd grpc
mkdir -p cmake/build
pushd cmake/build
cmake -DgRPC_INSTALL=ON \
      -DgRPC_BUILD_TESTS=OFF \
      -DCMAKE_INSTALL_PREFIX=$MY_INSTALL_DIR \
      ../..
make -j 4
make install
popd

```

##### Run a helloworld
```
cd examples/cpp/helloworld
mkdir -p cmake/build
pushd cmake/build
cmake -DCMAKE_PREFIX_PATH=$MY_INSTALL_DIR ../..
make -j 4
```
