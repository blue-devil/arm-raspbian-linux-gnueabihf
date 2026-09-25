# Raspbian (ARMv6) cross toolchain for Arch Linux ARM (aarch64 host)

PKGBUILDs for a toolchain that builds **32-bit ARM executables for the legacy
Raspbian Jessie image** (`2017-04-10-raspbian-jessie.img`, the one from Azeria's
ARM exploitation tutorials, run under QEMU) on an Arch Linux ARM machine.

This is **not** the AUR `arm-linux-gnueabihf-*` toolchain. Debian/AUR "armhf" means
ARMv7-A + VFPv3-D16 + Thumb-2 and a 2025 glibc; Raspbian's "armhf" is **ARMv6 + VFPv2,
ARM mode** (BCM2835, Pi 1/Zero) with glibc 2.19. Binaries from the AUR toolchain do
not run on the image. Everything here is period-correct and mirrors what the image
was built with, verified by reading the image itself (see `PLAN.md` and
`scripts/raspbian_image_inventory.py`):

| Image component (dpkg)          | Version           | This project                            |
|---------------------------------|-------------------|-----------------------------------------|
| gcc-4.9 / libstdc++6            | 4.9.2-10          | gcc 4.9.2 (`c,c++,lto`)                 |
| libc6                           | 2.19-18+deb8u7    | glibc 2.19                              |
| binutils                        | 2.25-5            | binutils 2.25                           |
| kernel (image / qemu)           | 4.4.50 / 4.4.34   | linux-api-headers 4.4.34                |
| gdb                             | 7.7.1             | gdb 17.2 (host tool, matches gdb-common)|
| zlib1g                          | 1.2.8             | zlib 1.2.8                              |
| libssl1.0.0                     | 1.0.1t            | openssl 1.0.2u (same ABI, Debian symbol versions) |
| (none)                          |                   | openssl 1.1.1w (for software that needs 1.1) |

## Layout

* One name for everything, the GNU triplet `arm-raspbian-linux-gnueabihf`: package
  names and folders (`arm-raspbian-linux-gnueabihf-*`), the commands
  (`arm-raspbian-linux-gnueabihf-{gcc,g++,as,ld,gdb,ldd,...}`), the sysroot
  (`/usr/arm-raspbian-linux-gnueabihf/{include,lib}`) and GCC's internals
  (`/usr/lib/gcc/arm-raspbian-linux-gnueabihf/4.9.2/`).
* Why a vendor field: `arm-linux-gnueabihf-raspbian` is not a valid GNU triplet
  (`cpu-vendor-os`), and plain `arm-linux-gnueabihf` is Debian's ARMv7 armhf, which
  the AUR packages already use. `arm-raspbian-linux-gnueabihf` is what crosstool-NG
  style toolchains do (`arm-rpi-linux-gnueabihf`); GCC/binutils/glibc key their
  behaviour on `arm` + `linux-gnueabihf` only, so nothing else changes.
* Autotools/CMake cross builds simply use `--host=arm-raspbian-linux-gnueabihf`;
  every `<host>-tool` exists. Nothing collides with the AUR `arm-linux-gnueabihf-*`
  packages.
* Compiler defaults: `-march=armv6 -mfpu=vfp -mfloat-abi=hard`, dynamic linker
  `/lib/ld-linux-armhf.so.3`, `--hash-style=both`.

## Build and install order

Every step needs the previous one installed (`sudo pacman -U <pkg>`). Use
`MAKEFLAGS=-j$(nproc)` unless your `makepkg.conf` sets it.

1. `arm-raspbian-linux-gnueabihf-linux-api-headers`
2. `arm-raspbian-linux-gnueabihf-binutils`
3. `arm-raspbian-linux-gnueabihf-gcc-stage1` (C only, no libc)
4. `arm-raspbian-linux-gnueabihf-glibc` (built with stage1)
5. `arm-raspbian-linux-gnueabihf-gcc` (replaces stage1 automatically)
6. `arm-raspbian-linux-gnueabihf-gdb`

Part 2 (libraries in the sysroot):

1. `arm-raspbian-linux-gnueabihf-zlib`
2. `arm-raspbian-linux-gnueabihf-openssl-1.0`
3. `arm-raspbian-linux-gnueabihf-openssl-1.1`

There is no `gcc-stage2` and no `glibc-headers` package: glibc 2.19 builds directly
with the headers-less stage1 compiler (LFS 7.5 did the same).

```bash
for p in linux-api-headers binutils gcc-stage1 glibc gcc gdb zlib openssl-1.0 openssl-1.1; do
  ( cd arm-raspbian-linux-gnueabihf-$p && MAKEFLAGS=-j$(nproc) makepkg -sf && sudo pacman -U --noconfirm *.pkg.tar.xz )
done
```

## Use

```bash
arm-raspbian-linux-gnueabihf-gcc -O2 -o hello hello.c
arm-raspbian-linux-gnueabihf-readelf -A hello | grep -E 'CPU_arch|FP_arch|VFP_args'
qemu-arm -L /usr/arm-raspbian-linux-gnueabihf ./hello           # run on the host
arm-raspbian-linux-gnueabihf-ldd hello                          # cross ldd
scripts/verify_toolchain.sh                                     # full self-test
```

Debug with QEMU user mode:

```bash
qemu-arm -g 1234 -L /usr/arm-raspbian-linux-gnueabihf ./hello &
arm-raspbian-linux-gnueabihf-gdb -q ./hello -ex 'target remote :1234' -ex 'break main' -ex continue
```

Debug inside the image (system emulation): boot it with the launch scripts next to
the image (`/mnt/psf/BOLUT/QMU/ARM/RASPBIAN/EL/run-raspbian-{linux,macos}.sh`, which
forward host port 5022 to ssh and 1234 to gdbserver), run `gdbserver :1234 ./hello`
in the guest and `target remote localhost:1234` from the cross gdb; `set sysroot`
already points at our glibc, which has symbols (the image's libc is stripped).

OpenSSL: `-I$SYSROOT/include/openssl-1.0 -L$SYSROOT/lib/openssl-1.0` (or `-1.1`),
`PKG_CONFIG_LIBDIR=$SYSROOT/lib/openssl-1.0/pkgconfig`.

## Verified

`scripts/verify_toolchain.sh` passes on the build host: C and C++ samples are
ARMv6/VFPv2/hard-float with interpreter `/lib/ld-linux-armhf.so.3` and ABI note
2.6.32, need at most `GLIBC_2.4`, `GLIBCXX_3.4.9`, `CXXABI_1.3` (image limits: glibc
2.19, libstdc++ 6.0.20 = `GLIBCXX_3.4.20`/`CXXABI_1.3.8`), run under `qemu-arm`, the
image's own `armageddon` and `pwndr3` samples run against the sysroot (the latter
needs `libcrypto.so.1.0.0@OPENSSL_1.0.0`), and the cross gdb remote-debugs a binary
through QEMU's gdbstub. Running inside the image itself (`qemu-system-arm`) is the
one step left to the user's QEMU setup.

## Building 2014 sources on a 2026 host

GCC 4.9.2 needs one patch (`gcc/cp/cfns.h` gnu_inline) and `-std=gnu11`/`-std=gnu++98
-fpermissive` host flags; glibc 2.19 needs the LFS `libc_cv_*` cache variables and
`make no_deps=t` (GNU make >= 4.4 takes 25 minutes per subdirectory otherwise). Each
package README explains its own quirk.

## Scripts

* `scripts/raspbian_image_inventory.py`: prints the image's identity and package
  versions straight from the `.img` (debugfs on the ext4 partition, no mount).
* `scripts/verify_toolchain.sh`: compiles C/C++ samples, checks ELF attributes and
  glibc symbol versions against the image, runs them under `qemu-arm`.

## Resources

* [AUR arm-linux-gnueabihf-* packages](https://aur.archlinux.org/packages?O=0&K=arm-linux-gnueabihf) (layout reference)
* [Azeria Labs: emulate Raspberry Pi with QEMU](https://azeria-labs.com/emulate-raspberry-pi-with-qemu/)
* [LFS 7.5: glibc 2.19 with a pass-1 GCC](https://www.linuxfromscratch.org/lfs/view/7.5/chapter05/glibc.html)

## Author

Blue DeviL // SCT

## License

AGPLv3
