# Cross-build toolchain for FF5M / AD5M (zMod)

How to build `guppyscreen` for the Flashforge Adventurer 5M and AD5X running zMod. The binary must run in the zMod chroot, where it starts as `/root/guppyscreen/guppyscreen` (see `S80guppyscreen`).

## Target

- Allwinner R528, ARMv7, hard float (armhf).
- Kernel 5.4.61.
- zMod chroot: glibc 2.40, libstdc++ from GCC 13, `libts.so.0`.
- The binary zMod ships (`z_ff5m/.shell/root/guppyscreen`) was built with Buildroot GCC 13.3. It requires at most `GLIBC_2.38` and `GLIBCXX_3.4.32`, uses **32-bit `time_t`** and 64-bit file offsets.

## Toolchain

The Ubuntu 24.04 cross toolchain: GCC 13.3, glibc 2.39. The packages are extracted into a directory, here called `<xtools>`, with no system install and no root access needed. This machine uses `/home/xeye/workplace/guppy-xtools`.

### Packages

C/C++ compiler and runtime. `apt-get download <pkg>` works on an Ubuntu 24.04 host:

| Package | Version |
|---|---|
| `binutils-arm-linux-gnueabihf` | 2.42-4ubuntu2.10 |
| `cpp-13-arm-linux-gnueabihf` | 13.3.0-6ubuntu2~24.04.1cross1 |
| `gcc-13-arm-linux-gnueabihf` | 13.3.0-6ubuntu2~24.04.1cross1 |
| `gcc-13-arm-linux-gnueabihf-base` | 13.3.0-6ubuntu2~24.04.1cross1 |
| `g++-13-arm-linux-gnueabihf` | 13.3.0-6ubuntu2~24.04.1cross1 |
| `libc6-armhf-cross` | 2.39-0ubuntu8cross1 |
| `libc6-dev-armhf-cross` | 2.39-0ubuntu8cross1 |
| `libgcc-13-dev-armhf-cross` | 13.3.0-6ubuntu2~24.04.1cross1 |
| `libgcc-s1-armhf-cross` | 14.2.0-4ubuntu2~24.04.1cross1 |
| `libatomic1-armhf-cross` | 14.2.0-4ubuntu2~24.04.1cross1 |
| `libstdc++-13-dev-armhf-cross` | 13.3.0-6ubuntu2~24.04.1cross1 |
| `libstdc++6-armhf-cross` | 14.2.0-4ubuntu2~24.04.1cross1 |
| `linux-libc-dev-armhf-cross` | 6.8.0-25.25cross1 |

tslib (armhf) comes from `http://ports.ubuntu.com/ubuntu-ports/pool/universe/t/tslib/`:

| Package | Version |
|---|---|
| `libts0t64` | 1.22-1.1build2 |
| `libts-dev` | 1.22-1 |

```sh
cd <xtools>/debs && for p in libts0t64 libts-dev; do
  f=$(curl -s http://ports.ubuntu.com/ubuntu-ports/pool/universe/t/tslib/ | grep -oE "${p}_[^\"]*_armhf\.deb" | sort -uV | tail -1)
  curl -sO http://ports.ubuntu.com/ubuntu-ports/pool/universe/t/tslib/$f
done
```

The Ubuntu `libts` is only used for the header and for linking. At runtime the binary loads the chroot's own `libts.so.0`.

### Setup

```sh
X=<xtools>
mkdir -p $X/root
for d in $X/debs/*.deb; do dpkg-deb -x "$d" $X/root; done

# the gcc-13 / g++-13 packages only ship versioned names
cd $X/root/usr/bin
for t in gcc g++ cpp gcc-ar gcc-nm gcc-ranlib; do
  [ -e arm-linux-gnueabihf-$t ] || ln -s arm-linux-gnueabihf-$t-13 arm-linux-gnueabihf-$t
done
```

### Compiler wrappers (required)

`<xtools>/wrap/arm-linux-gnueabihf-gcc` (and the same for `-g++`):

```sh
#!/bin/sh
# 32-bit time_t to match zMod's Buildroot libs (libts ABI); keep 64-bit file offsets
exec <xtools>/root/usr/bin/arm-linux-gnueabihf-gcc --sysroot=<xtools>/root -U_TIME_BITS -D_FILE_OFFSET_BITS=64 "$@"
```

Symlink the binutils next to them so that `$(CROSS_COMPILE)ar`, `strip` and the others resolve:

```sh
cd $X/wrap
for b in ar as ld nm strip ranlib objcopy objdump readelf size; do
  ln -sf $X/root/usr/bin/arm-linux-gnueabihf-$b .
done
```

Why each flag:

- **`-U_TIME_BITS`**: Ubuntu's armhf glibc headers default to 64-bit `time_t`. tslib's `struct ts_sample` contains a `struct timeval`, so a 64-bit build passes a struct of the wrong size to the chroot's 32-bit `libts`, and `ts_read` writes past it. The shipped binary imports plain `gettimeofday` and `clock_gettime`, not the `__*_time64` versions, which confirms it uses 32-bit `time_t`.
- **`-D_FILE_OFFSET_BITS=64`**: Ubuntu turns on 64-bit file offsets only together with 64-bit time. With `-U_TIME_BITS` alone, `off_t` drops to 4 bytes. Buildroot uses 64-bit offsets, and the shipped binary imports `stat64`, `lstat64` and `fstat64`.
- **`--sysroot`**: inside the wrapper, so the top-level Makefile, libhv's Makefile and spdlog's cmake all use the same sysroot and flags.

Expected type sizes: `time_t` = 4, `off_t` = 8, `struct timeval` = 8.

### env.sh

```sh
# source this before building guppyscreen for FF5M (zMod chroot, glibc 2.40)
export PATH=<xtools>/wrap:$PATH
export LD_LIBRARY_PATH=<xtools>/root/usr/lib/x86_64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
export CROSS_COMPILE=<xtools>/wrap/arm-linux-gnueabihf-
export SYSROOT=<xtools>/root
```

`LD_LIBRARY_PATH` is required: the extracted binutils can't find their own `libbfd`, `libopcodes` and `libctf` otherwise.

## Source preparation

```sh
git submodule update --init --depth 1 --jobs 4
(cd lv_drivers && git apply ../patches/0001-lv_driver_fb_ioctls_flashforge_ad5m.patch)  # AD5X: ..._ad5x.patch
(cd spdlog     && git apply ../patches/0002-spdlog_fmt_initializer_list.patch)
(cd lvgl       && git apply ../patches/0003-lvgl-dpi-text-scale.patch)
(cd libhv      && git apply ../patches/0004-fix-cppflags-override.patch)
(cd libhv      && git apply ../patches/0005-libhv-websocket-release-big-message-buffer.patch)
(cd lvgl       && git apply ../patches/0006-lvgl-draw-indexed-8bit-decoded-images.patch)
(cd lvgl       && git apply ../patches/0007-lvgl-draw-indexed-8bit-images-whole.patch)
```

A shallow clone works: GitHub serves the pinned commits directly. The clones take about 113 MB in `.git/modules`, mostly `lvgl`.

## Build

```sh
. <xtools>/env.sh
V="CROSS_COMPILE=$CROSS_COMPILE FF5M=1 SYSROOT=$SYSROOT GUPPY_SMALL_SCREEN=1 GUPPYSCREEN_VERSION=0.0.30.1-zmod+$(git rev-parse --short HEAD)"

make libhv.a     $V      # first time, or after `make cleanall`
make libspdlog.a $V
make -j$(nproc)  $V      # guppyscreen -> build/bin/guppyscreen
```

`make build $V` runs all three steps, cleaning first.

**`GUPPY_SMALL_SCREEN=1` is required to match zMod.** The binary zMod ships logs `DPI: 90` at startup, and only `GUPPY_SMALL_SCREEN` sets `LV_DPI_DEF` to 90 (`lv_conf.h:104`); the default is 130. DPI scales LVGL's default sizes and padding across the whole UI. Under `FF5M` it has no other effect: `ASSET_DIR` stays `material`. Because it changes `lv_conf.h`, switching it on or off needs a full rebuild of the objects: `rm -rf build/obj`. Avoid `make clean`, which also deletes `libhv.a` and `libspdlog.a`.

zMod's binary reports `Guppy Screen Version: 0.0.30.1-zmod`. Append something like the commit hash so the log shows which build is running.

The build prints many `-Wstringop-overflow` / `-Warray-bounds` warnings from the nlohmann JSON headers. They're harmless.

## Checks

Use the host's `readelf`; it reads ARM binaries.

```sh
B=build/bin/guppyscreen
readelf -V $B | grep -o 'GLIBC_2\.[0-9]*'     | sort -uV | tail -1   # <= GLIBC_2.38 (chroot has 2.40)
readelf -V $B | grep -o 'GLIBCXX_3\.4\.[0-9]*' | sort -uV | tail -1   # <= GLIBCXX_3.4.32 (GCC 13)
readelf -d $B | grep NEEDED          # libts.so.0 libm libstdc++ libgcc_s libc ld-linux-armhf
readelf --dyn-syms -W $B | grep time64
```

The `time64` check should list exactly two symbols, `__stat64_time64` and `__lstat64_time64`. They come from Ubuntu's static `libstdc++fs.a` (`std::experimental::filesystem`), are used only inside it, and glibc 2.40 provides them. If the guppyscreen objects themselves reference `*time64` symbols, the wrappers were bypassed. Find the culprit with `nm -A build/obj/src/*.o | grep ' U .*time64'`.

Result of the first build (2026-11-12), compared with the shipped binary:

| | This build | zMod shipped |
|---|---|---|
| glibc / libstdc++ | 2.38 / 3.4.32 | 2.38 / 3.4.32 |
| `libatomic` loaded | no | yes |
| code and read-only data | 3,115,724 B | 3,889,453 B |
| BSS | 1,051,896 B | 1,051,896 B |

The shipped binary has more code because it was built from a different commit.

## Install on the printer

Same procedure as `mjpg-streamer/BUILD_AD5M.md`:
- The printer has no sftp server, so `scp` fails. Copy with `ssh … 'cat > file'`.
- Upload under a temporary name, then `mv` it into place.
- Back up the current binary first.
- Check that the printer isn't printing.

Restart with `/etc/init.d/S80guppyscreen restart` inside the chroot. The log is in `/opt/config/mod_data/log/guppyscreen.log`.

Not yet done: this build has not been run on a printer.
