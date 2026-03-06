# Building Prince of Persia Apple II

## Prerequisites

The included build tools (`Build/osx32/snap` and `Build/osx32/crackle`) were originally 32-bit i386 binaries that don't run on modern macOS (Catalina+). They have been replaced with 64-bit x86_64 builds compiled from the [snapNcrackle source](https://github.com/adamgreen/snapNcrackle). The originals are preserved as `snap_i386` and `crackle_i386`.

If you need to rebuild the tools from source (e.g. on Apple Silicon):

```bash
cd /tmp
git clone https://github.com/adamgreen/snapNcrackle.git
cd snapNcrackle
```

Add `-Wno-deprecated-declarations` to `CPPUTEST_WARNINGFLAGS` in `libcommon/makefile`, `libsnap/makefile`, and `libcrackle/makefile`. Also add `-Wno-varargs` to `libcrackle/makefile`. Then:

```bash
make clean all CFG=Release
cp snap/Release/snap /path/to/Prince-of-Persia-Apple-II/Build/osx32/snap
cp crackle/Release/crackle /path/to/Prince-of-Persia-Apple-II/Build/osx32/crackle
```

## Build

```bash
cd /Users/nick/Projects/Prince-of-Persia-Apple-II
make clean all
```

This produces three disk images:

| File | Format | Size |
|------|--------|------|
| `PrinceOfPersia_3.5.hdv` | 3.5" ProDOS block image | 800 KB |
| `PrinceOfPersia_5.25_SideA.nib` | 5.25" nibble image (Side A) | 228 KB |
| `PrinceOfPersia_5.25_SideB.nib` | 5.25" nibble image (Side B) | 228 KB |

Assembly warnings about `DO/IF` and `fin` directives in GRAFIX.S and SPECIALK.S are expected and harmless.

To build with release patches applied (disables cheats, updates version date):

```bash
make clean all RELEASE_PATCH=1
```

To see assembler/imager commands as they run:

```bash
make clean all VERBOSE=1
```

## Running in Virtual II

The game runs on **Virtual II** (`/Applications/Virtual ][.app`) using the 5.25" .nib disk images.

### Setup

1. Open Virtual II
2. Configure an **Apple IIe** machine
3. Ensure two 5.25" floppy drives are attached (Disk II in slots 6)
4. Insert `PrinceOfPersia_5.25_SideA.nib` into Drive 1
5. Boot the machine (power on or Ctrl+Reset)
6. When prompted to "Insert Prince of Persia Disk", swap Drive 1 to `PrinceOfPersia_5.25_SideB.nib`

The 3.5" `.hdv` image also works if you configure a UniDisk 3.5 drive instead.

### Controls

Press **Ctrl+K** for keyboard mode:

| Key | Action |
|-----|--------|
| J / L | Move left / right |
| I | Jump up / climb |
| K | Crouch / climb down |
| U / O | Jump diagonally left / right |
| Open-Apple | Action button (careful step, grab ledge, draw sword, strike) |

Press **Ctrl+J** to switch to joystick mode.

### Cheats (non-release builds only)

Type `POP` (no prompt, just type it during gameplay), then:

| Code | Effect |
|------|--------|
| SKIP | Advance to next level |
| TINA | Jump to final level |
| BOOST | Add 1 health point |
| R | Recharge health to max |
| Z | Reduce opponent health to 1 |
| ZAP | Kill opponent |
