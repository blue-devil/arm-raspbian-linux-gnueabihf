# Raspbian (ARMv6) cross toolchain for Arch Linux ARM (aarch64 host)

PKGBUILDs for a toolchain that builds **32-bit ARM executables for the legacy
Raspbian Jessie image** (`2017-04-10-raspbian-jessie.img`, the one from Azeria's
ARM exploitation tutorials, run under QEMU) on an Arch Linux ARM machine.

This is **not** the AUR `arm-linux-gnueabihf-*` toolchain. Debian/AUR "armhf" means
ARMv7-A + VFPv3-D16 + Thumb-2 and a 2025 glibc; Raspbian's "armhf" is **ARMv6 + VFPv2,
ARM mode** (BCM2835, Pi 1/Zero) with glibc 2.19. Binaries from the AUR toolchain do
not run on the image.

| Image component (dpkg)  | Version          | This project                                      |
| ----------------------- | ---------------- | ------------------------------------------------- |
| gcc-4.9 / libstdc++6    | 4.9.2-10         | gcc 4.9.2 (`c,c++,lto`)                           |
| libc6                   | 2.19-18+deb8u7   | glibc 2.19                                        |
| binutils                | 2.25-5           | binutils 2.25                                     |
| kernel (image / qemu)   | 4.4.50 / 4.4.34  | linux-api-headers 4.4.34                          |
| gdb                     | 7.7.1            | gdb 17.2 (host tool, matches gdb-common)          |
| zlib1g                  | 1.2.8            | zlib 1.2.8                                        |
| libssl1.0.0             | 1.0.1t           | openssl 1.0.2u (same ABI, Debian symbol versions) |
| (none)                  |                  | openssl 1.1.1w (for software that needs 1.1)      |

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
```

Debug with QEMU user mode:

```bash
qemu-arm -g 1234 -L /usr/arm-raspbian-linux-gnueabihf ./hello &
arm-raspbian-linux-gnueabihf-gdb -q ./hello -ex 'target remote :1234' -ex 'break main' -ex continue
```

### Debug inside the image (system emulation)

Boot your related Raspbian image. In your boot script or options let it
forward host port `5022` to ssh and `1234` to gdbserver,
run `gdbserver :1234 ./hello` in the guest and `target remote localhost:1234`
from the cross gdb; `set sysroot` already points at our glibc, which has
symbols (the image's libc is stripped).

OpenSSL: `-I$SYSROOT/include/openssl-1.0 -L$SYSROOT/lib/openssl-1.0`
(or `-1.1`), `PKG_CONFIG_LIBDIR=$SYSROOT/lib/openssl-1.0/pkgconfig`.

## Building 2014 sources on a 2026 host

This is stupid but I did it!

GCC 4.9.2 needs one patch (`gcc/cp/cfns.h` gnu_inline) and
`-std=gnu11`/`-std=gnu++98 -fpermissive` host flags. glibc 2.19 needs
the LFS `libc_cv_*` cache variables and `make no_deps=t`
(GNU make >= 4.4 takes 25 minutes per subdirectory otherwise).

## Resources

* [AUR arm-linux-gnueabihf-* packages][01] (layout reference)
* [Azeria Labs: emulate Raspberry Pi with QEMU][02]
* [LFS 7.5: glibc 2.19 with a pass-1 GCC][03]

## Author

Blue DeviL // SCT

## Last Words

> An empty shell shines  
> The clock arm points to wasted time  
> Forever rots
>
> Blue DeviL // SCT
> 25/09/2026

## License

AGPLv3

[01]: https://aur.archlinux.org/packages?O=0&K=arm-linux-gnueabihf
[02]: https://azeria-labs.com/emulate-raspberry-pi-with-qemu/
[03]: https://www.linuxfromscratch.org/lfs/view/7.5/chapter05/glibc.html
