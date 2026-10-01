# AVBTOOL Mod

[English](README.md) | 中文

原项目：https://github.com/AndroidBootloader/platform_external_avb

AVBTOOL Mod 把上游 `avbtool.py`（platform_external_avb）和它运行时依赖的外部工具一起打包成一个自包含的发布产物。`avbtool.py` 基于上游源码，唯一有意偏离是 `add_hashtree_footer` 上可选的 `--threads N` 参数（默认 `N=1`，保持原行为），其余默认参数和命令结构不变。

## 产物内容

构建工作流在每个架构上编译三个二进制文件，打进同一个压缩包：

```text
bin/
├── avbtool    # 上游 avbtool.py 冻结产物（PyInstaller 或 Nuitka）
├── fec        # 独立 CLI，实现 AOSP libfec 的 RS-8 协议
└── openssl    # OpenSSL 4.2.0-dev CLI + 静态 libcrypto
```

`avbtool` 运行时按名字调用 `fec` 和 `openssl`，因此三者必须在 `PATH` 上（例如解压后把 `bin/` 加入 `PATH`）。`fec` 与 `openssl` 为静态链接，运行时无动态库依赖。

## 各组件说明

- `avbtool.py` — 上游源码，仅一处有意偏离：`add_hashtree_footer` 新增 `--threads N`（默认 1 为串行；N > 1 时用 multiprocessing 绕过 GIL，适用于大镜像）。输出与串行路径逐字节一致，N 不影响结果。其余行为、默认参数和命令结构不变。
- `fec/` — vendored 的 AOSP `external/fec` 源码（Phil Karn libFEC）。`fec_core/fec_cli.c` 是基于其 RS-8 char 编解码器（`encode_rs_char.c`、`init_rs_char.c`、`fec.c`）的小封装，实现 avbtool 期望的 `fec --print-fec-size` / `fec --encode` 协议，包括 60 字节 packed `struct fec_header` 尾部（magic `0xfecfecfe`）及原始奇偶数据的 SHA-256 摘要。
- `openssl/` — vendored 的 OpenSSL 4.2.0-dev 源码，以 `no-asm no-shared -no-docs` 构建，使产物中的 `openssl` CLI 与 `libcrypto.a` 自包含。
- `contrib/` — 上游的 dm-verity/AVB Linux 内核补丁（仅作参考材料，构建不使用）。

## 构建（GitHub Actions）

工作流在 Ubuntu 22.04 上为 amd64 和 arm64 构建，每种架构使用两种 avbtool 构建器（共 4 个 job）：

| 矩阵 | Runner | avbtool 构建器 |
|------|--------|----------------|
| amd64 / arm64 | ubuntu-22.04(-arm) | PyInstaller `--onefile` |
| amd64 / arm64 | ubuntu-22.04(-arm) | Nuitka `--standalone --onefile --static-libpython=yes` |

流程：配置 + 编译 + 安装 vendored OpenSSL → 用静态 `libcrypto` 编译 `fec` → 冻结 `avbtool.py` → 组装 `bin/{avbtool,fec,openssl}` 为单层 `avbtool-mod-<版本号>-<指令集>-<构建方式>-<YYYYMMDD>.zip`（`bin/` 位于 zip 根目录，不再套 tar 层；另附 `-logs` 产物）。触发方式：`workflow_dispatch` 与 main / edge 分支 push。

## 命令结构

命令集与上游 `avbtool 1.2.0` 完全一致，完整树见 `./avbtool --help`。与签名相关的重点：

- `add_hash_footer` — 小分区（boot/recovery/dtbo）签名。
- `add_hashtree_footer` — 大分区（system/vendor）dm-verity hashtree 签名；`--fec_num_roots N` 调用内置 `fec` 生成 FEC 数据（OpenMP 加速，输出与线程数无关）；`--calc_max_image_size` 打印给定 `--partition_size` 下可容纳的最大镜像；`--threads N`（默认 1，串行）在大镜像时用 N 个进程并行做 level-0 块哈希，输出与串行路径逐字节一致。
- `resize_image` — 已签名镜像按新分区大小重新适配。
- `verify_image` / `info_image` / `print_partition_digests` — 校验与查看。
- `make_vbmeta_image`、`append_vbmeta_image`、`extract_vbmeta_image`、`extract_public_key`、`erase_footer`、`zero_hashtree` — footer/vbmeta 操作。
- `make_atx_certificate` / `make_atx_permanent_attributes` / `make_atx_metadata` / `make_atx_unlock_credential` — Android ATX 证书签名。

## 许可证

- `avbtool.py`、`contrib/` 与 `LICENSE`：Apache 2.0，Copyright 2016, The Android Open Source Project。
- `fec/`：LGPL 2.1+（AOSP external/fec，源自 Phil Karn 的 libFEC）。
- `openssl/`：Apache 2.0（OpenSSL Project）。
- `fec_core/fec_cli.c`：Apache 2.0，本项目。
