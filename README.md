1.name: "Mohamma"

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]
  workflow_dispatch:

2.jobs:
  tests:
    name: ๐งช Unit Tests
    runs-on: ubuntu-22.04
    permissions:
      actions: read
      contents: read
      security-events: write

    steps:
    - name: ๐งฐ Checkout
      uses: actions/checkout@v3
      with:
        submodules: recursive

    - name: ๐ Setup ccache
      uses: hendrikmuhs/ccache-action@v1.2
      with:
        key: ${{ runner.os }}-${{ secrets.CACHE_VERSION }}-build-${{ github.run_id }}
        restore-keys: ${{ runner.os }}-${{ secrets.CACHE_VERSION }}-build
        max-size: 50M
        

    - name: ๐ Restore CMakeCache
      uses: actions/cache@v3
      with:
        path: |
          build/CMakeCache.txt
        key: ${{ runner.os }}-${{ secrets.CACHE_VERSION }}-build-${{ hashFiles('**/CMakeLists.txt') }}
        
    - name: โฌ๏ธ Install dependencies
      run: |
        sudo apt update
        sudo bash dist/get_deps_debian.sh
    - name: ๐ ๏ธ Build
      run: |
        mkdir -p build
        cd build
        CC=gcc-12 CXX=g++-12 cmake                                                                      \
          -DCMAKE_BUILD_TYPE=Debug                                                                      \
          -DCMAKE_INSTALL_PREFIX="$PWD/install"                                                         \
          -DCMAKE_C_COMPILER_LAUNCHER=ccache                                                            \
          -DCMAKE_CXX_COMPILER_LAUNCHER=ccache                                                          \
          -DCMAKE_C_FLAGS="-fuse-ld=lld -fsanitize=address,leak,undefined -fno-sanitize-recover=all"    \
          -DCMAKE_CXX_FLAGS="-fuse-ld=lld -fsanitize=address,leak,undefined -fno-sanitize-recover=all"  \
          -DIMHEX_OFFLINE_BUILD=ON                                                                      \
          ..
        make -j4 unit_tests install
    - name: ๐งช Perform Unit Tests
      run: |
        cd build
        ctest --output-on-failure
