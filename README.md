# Raspbian ARMHF Toolchain for Arch Linux ARM (Aarch64 Host)

**TODO** Compile stage1

There is a populer Raspbian Image which can be emulate by QEMU. That Raspbian
image is which @azeria used in her blog. Here are the versions list of
important packages in that raspbian image

Version list

```txt
Linux Kernel v4.4.34
Binutils v2.25
libc v2.19
GCC v4.9.2
gdb v7.7.1
openssl v1.0.1
zlib v1.2.8
```

Compilation and Installation order:

1. arm-linux-gnueabihf-raspbian-linux-api-headers
2. arm-linux-gnueabihf-raspbian-binutils

Build and install above two packages. Before building gcc, we need some
**intermediary** packages to build gcc.

1. arm-linux-gnueabihf-raspbian-gcc-stage1
2. arm-linux-gnueabihf-raspbian-glibc-headers -> needs stage1

Build and install above two packages. Now we can build stage2. After
installing gcc-stage2 we can now build `glibc`. But while installing
gcc-stage2, gcc-stage1 must be uninstalled.

1. arm-linux-gnueabihf-raspbian-gcc-stage2

Before installing x86_64-linux-gnu-glibc you should uninstall
`x86_64-linux-gnu-glibc-headers`.

1. arm-linux-gnueabihf-raspbian-glibc -> needs gcc-stage2 to build
2. arm-linux-gnueabihf-raspbian-gcc -> remove gcc-stage2 before installing
3. arm-linux-gnueabihf-raspbian-gdb

## Optional

I need openssl-1.1 for some packages. To build openssl we first need zlib.
Build and install zlib, then build and install openssl-1.1

1. arm-linux-gnueabihf-raspbian-zlib
2. arm-linux-gnueabihf-raspbian-openssl-1.1

## Run ARMHF Binaries on Arch Linux using QEMU

## Resources

* [Archlinux AUR - arm-linux-gnueabihf-* packages][01]
* [Archlinux AUR - arm-linux-gnueabihf-binutils v2.25.1][02]

## Author

Blue DeviL // SCT

## License

AGPLv3

[01]: https://aur.archlinux.org/packages?O=0&K=arm-linux-gnueabihf
[02]: https://aur.archlinux.org/cgit/aur.git/tree/PKGBUILD?h=arm-linux-gnueabihf-binutils&id=46c4b550f223eeb61a707a090d6079e8c4f549e0
