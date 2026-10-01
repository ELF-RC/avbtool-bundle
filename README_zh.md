# AVBTOOL Bundle

[English](README.md) | 中文

原项目：https://github.com/AndroidBootloader/platform_external_avb

## 命令树

### avbtool

```
avbtool
├── generate_test_image           生成已知图案的测试镜像
├── version                       打印 avbtool 版本
├── extract_public_key            提取公钥
├── make_vbmeta_image             生成 vbmeta 镜像
├── add_hash_footer               为镜像添加哈希和 footer
├── append_vbmeta_image           将 vbmeta 镜像追加到镜像
├── add_hashtree_footer           为镜像添加哈希树和 footer
├── erase_footer                  擦除镜像 footer
├── zero_hashtree                 清零哈希树和 FEC 数据
├── extract_vbmeta_image          从带 footer 的镜像提取 vbmeta
├── resize_image                  调整带 footer 镜像的尺寸
├── info_image                    查看 vbmeta 或 footer 信息
├── verify_image                  校验镜像
├── print_partition_digests       打印分区摘要
├── calculate_vbmeta_digest       计算 vbmeta 摘要
├── calculate_kernel_cmdline      计算内核 cmdline
├── set_ab_metadata               设置 A/B 元数据
├── make_atx_certificate          创建 ATX 证书
├── make_atx_permanent_attributes 创建 ATX 永久属性
├── make_atx_metadata             创建 ATX 元数据
└── make_atx_unlock_credential    创建 ATX 解锁凭证
```

### fec

```
fec
├── --print-fec-size SIZE --roots N   打印 SIZE 字节输入对应的 FEC 字节数
└── --encode [--roots N] INPUT OUTPUT 将 INPUT 编码到 OUTPUT（RS 奇偶 + 60 字节 footer）
```

### openssl

```
openssl
├── 标准命令
│   ├── asn1parse  ca  ciphers  cmp  cms  configutl  crl  crl2pkcs7
│   ├── dgst  dhparam  dsa  dsaparam  ec  ech  ecparam  enc
│   ├── errstr  fipsinstall  gendsa  genpkey  genrsa  help  info  kdf
│   ├── list  mac  nseq  ocsp  passwd  pkcs12  pkcs7  pkcs8
│   ├── pkey  pkeyparam  pkeyutl  prime  rand  rehash  req  rsa
│   ├── rsautl  s_client  s_server  s_time  sess_id  skeyutl  smime
│   ├── speed  spkac  srp  storeutl  ts  verify  version  x509
├── 消息摘要
│   └── blake2b512  blake2s256  md4  md5  mdc2  rmd160  sha1  sha224
│       sha256  sha3-224  sha3-256  sha3-384  sha3-512  sha384  sha512
│       sha512-224  sha512-256  shake128  shake256  sm3
└── 密码算法（经 enc）
    └── aes-128/192/256-{cbc,ecb,cfb,ctr,ofb}  aria-*  base64  cast-*
        des-*  des-ede*  等
```

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

## 许可证

- `avbtool.py`、`contrib/` 与 `LICENSE`：Apache 2.0，Copyright 2016, The Android Open Source Project。
- `fec/`：LGPL 2.1+（AOSP external/fec，源自 Phil Karn 的 libFEC）。
- `openssl/`：Apache 2.0（OpenSSL Project）。
- `fec_core/fec_cli.c`：Apache 2.0，本项目。
