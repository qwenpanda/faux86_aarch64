# faux86 — AArch64 (64-bit) port for Raspberry Pi 3A+

This is [jhhoward/Faux86](https://github.com/jhhoward/Faux86) (the original, **not**
ArnoldUK's remake) patched to cross-compile against
[rsta2/circle](https://github.com/rsta2/circle) in 64-bit AArch64 mode for a
Raspberry Pi 3A+, producing a `kernel8.img`.

The CPU core (`src/faux86/*.cpp`) is untouched — it was already
architecture-agnostic. Only the Circle glue layer and build config needed
changes. All changes below compile clean with `-Wall -Werror` and link with
zero undefined symbols.

## 1. Toolchain — get this right first

**Do not use `gcc-aarch64-linux-gnu` from apt.** It's a hosted (glibc)
toolchain. It compiles and links almost everything, then fails at the final
link step because Circle's `Rules.mk` pulls in `libm.a`, and the hosted
glibc `libm.a` needs OS-level plumbing (`errno`, `__stack_chk_guard`,
`__getauxval`) that doesn't exist on bare metal. You'll see undefined
references to those three symbols plus a linker assertion/segfault.

Use a real bare-metal `aarch64-none-elf-` toolchain instead (xPack's GCC,
via npm — reachable even from sandboxes with restricted egress):

```bash
mkdir -p toolchain && cd toolchain
npm pack @xpack-dev-tools/aarch64-none-elf-gcc@13.3.1-1.1.2
tar xzf xpack-dev-tools-aarch64-none-elf-gcc-13.3.1-1.1.2.tgz
# read package/package.json -> xpack.binaries.platforms.linux-x64.fileName + sha256
curl -L -o toolchain.tar.gz "https://github.com/xpack-dev-tools/aarch64-none-elf-gcc-xpack/releases/download/v13.3.1-1.1/xpack-aarch64-none-elf-gcc-13.3.1-1.1-linux-x64.tar.gz"
echo "f76dc6d105f054fcb3f2a39ecf206d99101dc87931a5b9227fe886cb9478b667  toolchain.tar.gz" | sha256sum -c -
tar xzf toolchain.tar.gz
# make the aarch64-none-elf-* binaries available on your PATH, e.g.:
export PATH="$(pwd)/xpack-aarch64-none-elf-gcc-13.3.1-1.1/bin:$PATH"
```

Verify: `aarch64-none-elf-gcc -print-file-name=libm.a` should resolve to a
path **inside the toolchain's own sysroot** (newlib), not a system path.

## 2. Build Circle

```bash
git clone --depth 1 https://github.com/rsta2/circle.git
cd circle
./configure -r 3 -p aarch64-none-elf-   # RASPPI=3, AARCH=64
./makeall
make -C addon/SDCard
make -C addon/vc4/sound
make -C addon/vc4/vchiq
make -C addon/linux
make -C addon/fatfs
make -C addon/Properties
```

All of the above should build with zero errors. `circle/` is **not**
included in this repo — clone it as a sibling directory (`../circle`
relative to `pi/`, per `pi/Makefile`'s `CIRCLEHOME`).

## 3. Build faux86

```bash
cd pi
make
```

Produces `kernel8.elf` / `kernel8.img`. Verify:

```bash
aarch64-none-elf-readelf -h kernel8.elf | grep -E "Class|Machine"   # expect ELF64, AArch64
aarch64-none-elf-nm -u kernel8.elf                                   # expect empty (zero undefined syms)
```

## 4. What was changed vs. upstream jhhoward/Faux86

| File | Change | Why |
|---|---|---|
| `src/faux86/Types.h` | Added `#include <stddef.h>`, deleted the manual `typedef unsigned int size_t;` | On AArch64, real `size_t` is 8 bytes; the hand-rolled 4-byte typedef conflicts. `Types.h` never included `<stddef.h>`, so the typedef was filling a real gap — the fix is the include, not an `#ifdef`. |
| `src/faux86/*.cpp` / `.h` (16 include sites across the whole tree, not just the Pi build) | Every `#include "x.h"` rewritten to the real on-disk casing (`Ram.h`, `Audio.h`, `Config.h`, `CPU.h`, `Ports.h`, `Types.h`, `Video.h`) | faux86 was developed on a case-insensitive filesystem; various files `#include`d these with the wrong case. Silent on Windows/macOS, hard-fails on Linux. Fixed at the include site rather than via same-directory symlinks, so the repo has one real file per header and no symlink indirection to break on non-POSIX checkouts. |
| `pi/CircleHostInterface.cpp` | `(uint8_t*) frameBuffer->GetBuffer()` → `(uint8_t*)(uintptr) frameBuffer->GetBuffer()` (both call sites) | `CBcmFrameBuffer::GetBuffer()` returns a 32-bit GPU bus address (`u32`). On AArch64 casting straight to a pointer truncates/warns. `uintptr` is Circle's own architecture-aware type for exactly this (see `addon/Spectrum/SpectrumScreen.cpp`). |
| `pi/CircleHostInterface.{cpp,h}` | `mouseStatusHandler` gained a 4th param `int nWheelMove` | Circle's `TMouseStatusHandler` typedef gained this parameter upstream; the old 3-param signature no longer matches. |
| `pi/PWMSound.h` | `#include <circle/pwmsoundbasedevice.h>` → `#include <circle/sound/pwmsoundbasedevice.h>` | Header moved in current Circle. |
| `pi/Makefile` | Added `$(CIRCLEHOME)/lib/sound/libsound.a` to `LIBS` | Was missing entirely — nothing to do with 64-bit, just a gap in the original Makefile; without it, linking fails with undefined `CSoundBaseDevice`/`CPWMSoundBaseDevice` references. |

Not needed for this codebase (called out in case they resurface after an
upstream sync):
- `CMouseDevice::Setup()` signature change (`Setup(w,h)` → `Setup(CDisplay*)`) — this glue code never calls `Setup()` on the mouse.
- Hardcoded 32-bit compiler flags (`-marm`, `-mfpu`, etc.) — that's an ArnoldUK/Faux86-remake issue; the original Makefile has none, it trusts Circle's `Rules.mk`.

## 5. SD card contents

```
bootcode.bin, start.elf, fixup.dat   # from circle/boot/, or fetch individually:
                                      # https://raw.githubusercontent.com/raspberrypi/firmware/<commit>/boot/<file>
config.txt                           # = circle/boot/config64.txt (arm_64bit=1, [pi3] kernel=kernel8.img)
kernel8.img                          # built above
pcxtbios.bin, videorom.bin, asciivga.dat, dosboot.img   # from data/ — exact filenames pi/kernel.cpp opens, nothing else
```

Passing the readelf/nm checks above means a structurally valid, fully-linked
64-bit kernel image — **not** a guarantee it boots on real hardware. This
has been built and verified structurally but not yet tested on a physical
Pi 3A+.
