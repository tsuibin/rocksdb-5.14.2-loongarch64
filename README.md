# RocksDB 5.14.2 on loongarch64

## 构建

    patch -p1 < doris-thirdparty/patches/rocksdb-5.14.2.patch   # Doris 自带补丁，可直接应用
    CXXFLAGS="-include cstdint -Wno-deprecated-copy -Wno-stringop-truncation -Wno-pessimizing-move \
              -Wno-range-loop-construct -Wno-shadow -Wno-unused-but-set-variable \
              -Wno-maybe-uninitialized -Wno-class-memaccess -Wno-array-parameter \
              -Wno-narrowing -Wno-dangling-reference" \
    LDFLAGS="-static-libstdc++ -static-libgcc" \
    PORTABLE=1 make USE_RTTI=1 -j4 static_lib

耗时约 14 分钟（-j4），`librocksdb.a` 约 866MB（带调试符号）。

## GCC 16 需要额外补的一个开关

Doris 官方参数里 `-Wno-unused-but-set-variable` **只在 clang 分支添加**（`build-thirdparty.sh:157` 附近）。
用 GCC 16 编译时会撞：

    third-party/fbson/FbsonJsonParser.h:617: error: 变量‘integ’被设定但未被使用 [-Werror=unused-but-set-variable]

所以 gcc 分支也要加 `-Wno-unused-but-set-variable`。另外 `-Wno-stringop-truncation`（`util/status.cc:28`）官方 gcc 分支已有。

## 注意

- 必须用 `PORTABLE=1`（否则会按本地 CPU 生成指令）。
- RocksDB 5.14 很老，`-include cstdint` 是必需的。

## 更新（2026-11-04）

Olap 阶段发现 Doris 的 `build_rocksdb()` **只拷了 `librocksdb.a`，头文件要另外手动拷**：

    mkdir -p $PREFIX/include && cp -r include/rocksdb $PREFIX/include/

漏了会报 `rocksdb/iterator.h：没有那个文件或目录`。
最终 `doris_be` 全量编译 + 链接成功（0 undefined symbol），rocksdb 是其中最大的静态库之一。
