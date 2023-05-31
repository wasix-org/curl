## Building for WASIX

This document has been tested on Linux (Debian distrib, but any should work). It should work on macSO without changes. If you are on Windows, try to look as WSL to get a Linux environment.

To build CURL for WASIX, you need recent version of LLVM/Clang with Wasm32 support (for example version 15 from the OS), the [wasix-lib](https://github.com/wasix-org/wasix-libc) sysroot32, and a WASIX build of [libopenssl](https://github.com/wasix-org/openssl). An optionnal WASI(X) build of zlib can also be used.
OpenSSL can be a bit tricky to build on WASIX so I suggest you use the forked repo linked in the description, and follow the guide to build the WASIX libs. Libz is fairly trivial to build and will not be detailled here. Untouched sources from the official repo should build without error.

CURL can be build with CMake of configure. I'll detail the CMake path only.

First, a cross toolchain file for WASIX needs to be created. I create `~/croos_toolchain.cmake` file with the following content:
```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR wasm32)

set(CMAKE_SYSROOT /PATH/TO/sysroot32)
set(triple wasm32-wasi)

set(tools /usr)
set(CMAKE_C_COMPILER ${tools}/bin/clang-15)
set(CMAKE_CXX_COMPILER ${tools}/bin/clang++-15)
set(CMAKE_C_COMPILER_TARGET ${triple})
set(CMAKE_CXX_COMPILER_TARGET ${triple})

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

SET(CMAKE_C_FLAGS "-matomics -mbulk-memory -mmutable-globals -pthread -mthread-model posix -ftls-model=local-exec -fno-trapping-math -D_WASI_EMULATED_MMAN -D_WASI_EMULATED_SIGNAL -D_WASI_EMULATED_PROCESS_CLOCKS" CACHE STRING "" FORCE)
SET(CMAKE_EXE_LINKER_FLAGS "-Wl,--shared-memory -Wl,--max-memory=4294967296 -Wl,--import-memory -Wl,--export-dynamic -Wl,--export=__heap_base -Wl,--export==__stack_pointer -Wl,--export=__data_end -Wl,--export=__wasm_init_tls -Wl,--export=__wasm_signal -Wl,--export=__tls_size -Wl,--export=__tls_align -Wl,--export=__tls_base" CACHE STRING "" FORCE)
```

The path to `wasix-libc` needs to be adapted of course.

The cmake project can now be generated. From the curl repo source, create a build folder and `cd` inside it:
```bash
mkdir build
cd build
```
Then configure the project (you need `cmake` also, install it from your OS repo)
```bash
cmake --toolchain ~/cross_compile_wasix.cmake .. -DOPENSSL_INCLUDE_DIR=/PATH/TO/openssl/include -DOPENSSL_CRYPTO_LIBRARY=/PATH/TO/openssl/libcrypto.a -DOPENSSL_SSL_LIBRARY=/PATH/TO/openssl/libssl.a -DZLIB_LIBRARY=/PATH/TO/zlib/libz.a -DZLIB_INCLUDE_DIR=/PATH/TO/zlib/ -DBUILD_SHARED_LIBS=NO -DBUILD_TESTING=NO -DCMAKE_BUILD_TYPE=Release
```
You need to adapt the PATHS to openssl repo source and zlib zepo source (or omit both ZLIB_XXX variable if you don't have a wasi(x) zlib)

Aftert that, the project should have been generated, and curl can be build with a simple `make` or `make -j4` to build it faster.
It will generate a `src/curl` binary, that is a wasm file. You can rename it to `curl.wasm` and use it (don't forget to use `--net` if you are using Wasmer runtime to run it)
