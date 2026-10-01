# AVBTOOL Mod

English | [中文](README_zh.md)

Original project: https://github.com/AndroidBootloader/platform_external_avb

AVBTOOL Mod packages the upstream `avbtool.py` (platform_external_avb) together with the external tools it shells out to, into one self-contained release artifact. `avbtool.py` is based on the upstream source; the only intentional deviation is the optional `--threads N` flag on `add_hashtree_footer` (default `N=1`, preserving original behaviour). All other default values and command surface are unchanged.

## What the release contains

The build workflow compiles three binaries on each architecture and ships them in one tarball:

```text
bin/
├── avbtool    # upstream avbtool.py, frozen (PyInstaller or Nuitka)
├── fec        # standalone CLI implementing the AOSP libfec RS-8 protocol
└── openssl    # OpenSSL 4.2.0-dev CLI + static libcrypto
```

`avbtool` invokes `fec` and `openssl` by name at runtime, so all three must be on `PATH` (e.g. extract the tarball and add its `bin/` to `PATH`). `fec` and `openssl` are statically linked — no shared-library dependencies at run time.

## How the pieces fit together

- `avbtool.py` — upstream source with one intentional deviation: `add_hashtree_footer` accepts `--threads N` (default 1, serial; N > 1 uses multiprocessing to bypass the GIL for large images). Output is byte-identical to the serial path regardless of N. All other behaviour, defaults, and command surface are unchanged.
- `fec/` — vendored AOSP `external/fec` source (Phil Karn libFEC). `fec_core/fec_cli.c` is a small standalone wrapper around its RS-8 char codec (`encode_rs_char.c`, `init_rs_char.c`, `fec.c`) that implements the `fec --print-fec-size` / `fec --encode` protocol avbtool expects, including the 60-byte packed `struct fec_header` footer (magic `0xfecfecfe`) with a SHA-256 digest of the raw parity.
- `openssl/` — vendored OpenSSL 4.2.0-dev source, built with `no-asm no-shared -no-docs` so the packaged `openssl` CLI and `libcrypto.a` are self-contained.
- `contrib/` — upstream Linux kernel patches for dm-verity/AVB (reference material only; not used by the build).

## Build (GitHub Actions)

The workflow builds on Ubuntu 22.04 for amd64 and arm64, with two avbtool builders per architecture (4 jobs total):

| Matrix | Runner | avbtool builder |
|--------|--------|-----------------|
| amd64 / arm64 | ubuntu-22.04(-arm) | PyInstaller `--onefile` |
| amd64 / arm64 | ubuntu-22.04(-arm) | Nuitka `--standalone --onefile --static-libpython=yes` |

Steps: configure + build + install OpenSSL from the vendored tree → compile `fec` against the static `libcrypto` → freeze `avbtool.py` → assemble `bin/{avbtool,fec,openssl}` into `avbtool-mod-<version>-<arch>-<builder>-<YYYYMMDD>.tar.gz` (plus a `-logs` artifact). Triggers: `workflow_dispatch` and push to `main`.

## Command surface

The command set is identical to upstream `avbtool 1.2.0`; run `./avbtool --help` for the full tree. Highlights relevant to signing:

- `add_hash_footer` — sign small partitions (boot/recovery/dtbo).
- `add_hashtree_footer` — sign large partitions (system/vendor) with dm-verity hashtree; `--fec_num_roots N` generates FEC data via the bundled `fec` tool (OpenMP-accelerated, deterministic regardless of thread count); `--calc_max_image_size` prints the largest image that fits a given `--partition_size`; `--threads N` (default 1, serial) runs level-0 block hashing in N worker processes for large images — output is byte-identical to the serial path.
- `resize_image` — re-fit an already-signed image to a new partition size.
- `verify_image` / `info_image` / `print_partition_digests` — inspect and validate.
- `make_vbmeta_image`, `append_vbmeta_image`, `extract_vbmeta_image`, `extract_public_key`, `erase_footer`, `zero_hashtree` — footer/vbmeta manipulation.
- `make_atx_certificate` / `make_atx_permanent_attributes` / `make_atx_metadata` / `make_atx_unlock_credential` — Android ATX certificate signing.

## License

- `avbtool.py`, `contrib/` and `LICENSE`: Apache 2.0, Copyright 2016, The Android Open Source Project.
- `fec/`: LGPL 2.1+ (AOSP external/fec, derived from Phil Karn's libFEC).
- `openssl/`: Apache 2.0 (OpenSSL Project).
- `fec_core/fec_cli.c`: Apache 2.0, this project.
