# AVBTOOL Bundle

English | [中文](README_zh.md)

Original project: https://github.com/AndroidBootloader/platform_external_avb

## Command tree

### avbtool

```
avbtool
├── generate_test_image           Generates a test image with a known pattern
├── version                       Prints the version of avbtool
├── extract_public_key            Extract a public key
├── make_vbmeta_image             Make a vbmeta image
├── add_hash_footer               Add hash and footer to an image
├── append_vbmeta_image           Append a vbmeta image to an image
├── add_hashtree_footer           Add hashtree and footer to an image
├── erase_footer                  Erase the footer from an image
├── zero_hashtree                 Zero out the hashtree and FEC data
├── extract_vbmeta_image          Extract vbmeta from an image with a footer
├── resize_image                  Resize an image with a footer
├── info_image                    Show information about vbmeta or a footer
├── verify_image                  Verify an image
├── print_partition_digests       Print partition digests
├── calculate_vbmeta_digest       Calculate a vbmeta digest
├── calculate_kernel_cmdline      Calculate a kernel cmdline
├── set_ab_metadata               Set A/B metadata
├── make_atx_certificate          Create an ATX certificate
├── make_atx_permanent_attributes Create ATX permanent attributes
├── make_atx_metadata             Create ATX metadata
└── make_atx_unlock_credential    Create an ATX unlock credential
```

### fec

```
fec
├── --print-fec-size SIZE --roots N   Print the FEC byte count for a SIZE-byte input
└── --encode [--roots N] INPUT OUTPUT Encode INPUT to OUTPUT (RS parity + 60-byte footer)
```

### openssl

```
openssl
├── Standard
│   ├── asn1parse  ca  ciphers  cmp  cms  configutl  crl  crl2pkcs7
│   ├── dgst  dhparam  dsa  dsaparam  ec  ech  ecparam  enc
│   ├── errstr  fipsinstall  gendsa  genpkey  genrsa  help  info  kdf
│   ├── list  mac  nseq  ocsp  passwd  pkcs12  pkcs7  pkcs8
│   ├── pkey  pkeyparam  pkeyutl  prime  rand  rehash  req  rsa
│   ├── rsautl  s_client  s_server  s_time  sess_id  skeyutl  smime
│   ├── speed  spkac  srp  storeutl  ts  verify  version  x509
├── Message digest
│   └── blake2b512  blake2s256  md4  md5  mdc2  rmd160  sha1  sha224
│       sha256  sha3-224  sha3-256  sha3-384  sha3-512  sha384  sha512
│       sha512-224  sha512-256  shake128  shake256  sm3
└── Cipher (via enc)
    └── aes-128/192/256-{cbc,ecb,cfb,ctr,ofb}  aria-*  base64  cast-*
        des-*  des-ede*  etc.
```

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

## License

- `avbtool.py`, `contrib/` and `LICENSE`: Apache 2.0, Copyright 2016, The Android Open Source Project.
- `fec/`: LGPL 2.1+ (AOSP external/fec, derived from Phil Karn's libFEC).
- `openssl/`: Apache 2.0 (OpenSSL Project).
- `fec_core/fec_cli.c`: Apache 2.0, this project.
