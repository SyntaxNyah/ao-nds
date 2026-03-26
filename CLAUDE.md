# ao-nds

## Project Overview

ao-nds is a Nintendo DS port of Attorney Online, the multiplayer roleplaying client based on the Ace Attorney games. It is built using BlocksDS (the modern successor to devkitARM/libnds) and supports WiFi connectivity, courtroom gameplay, character animations, audio playback, and the full Attorney Online protocol over TCP and WebSocket. Created by Headshotnoby, with additional code by stonedDiscord and UI design by Samevi.

## Architecture

- **Tech Stack:** C (gnu17) and C++ (gnu++17), BlocksDS SDK, dswifi, grit (for graphics conversion). The primary Makefile delegates to BlocksDS's `rom_arm9arm7` build system. `Makefile.dkarm` is a legacy devkitARM build file kept for reference but no longer used in the main build.
- **Structure:** Dual-CPU Nintendo DS architecture — ARM7 handles audio/WiFi firmware, ARM9 runs the main application. Headers and the libadx interface shared between CPUs live in `common/`. Game assets (sprites, backgrounds, audio, character data) are stored on the SD card under `fat/data/ao-nds/`.
- **Key Components:** The ARM9 application contains the full AO client: network sockets, courtroom rendering (backgrounds, character sprites, chatbox, shouts, WTCE animations), a complete UI system with many screens, font rendering via stb_truetype (fixed-point variant), INI/JSON config parsing, and WebSocket support via mongoose.

## Development Setup

### Prerequisites

- [Wonderful Toolchain](https://wonderful.asie.pl/) with BlocksDS installed. On Windows the default path expected by the Makefiles is `C:/msys64/opt/wonderful/`.
- BlocksDS core at `BLOCKSDS` (default: `C:/msys64/opt/wonderful/thirdparty/blocksds/core` on Windows, `/opt/blocksds/core` on Linux/macOS).
- BlocksDS external libs at `BLOCKSDSEXT` (Windows only, default: `C:/msys64/opt/wonderful/thirdparty/blocksds/external`).
- dswifi library (included in BlocksDS).
- `grit` graphics conversion tool (included in BlocksDS at `$(BLOCKSDS)/tools/grit/grit`).
- `ndstool` for packing the final `.nds` ROM (provided by BlocksDS).
- Python 3 with FreeImage bindings (for the `converter/` tooling).

### Build Commands

```bash
# Build the full ROM (arm7 + arm9 -> ao-nds.nds)
make

# Clean all build artifacts
make clean
```

The build system:
1. Compiles `arm7/` with `Makefile.arm7` (links `-ldswifi7 -lnds7`).
2. Compiles `arm9/` with `Makefile.arm9` (links `-ldswifi9 -lnds9`), also converting all PNGs in `arm9/gfx/` via `grit` into binary image files placed at `fat/data/ao-nds/ui/*.img.bin`.
3. Packs both ELFs into `ao-nds.nds` using `ndstool` with `icon.png` and the title "Attorney Online DS / Headshotnoby".

The project can also be opened and built in **Code::Blocks** using `ao-nds.cbp` with the `blocksds` compiler profile configured.

**Compiler flags (ARM9):**
- C: `-std=gnu17 -O2 -fomit-frame-pointer -ffast-math`
- C++: `-std=gnu++17 -O2 -fomit-frame-pointer -ffast-math -fno-rtti`
- Defines: `-DLZ77_STREAM`
- Warnings: `-Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers`

## Project Structure Details

- **arm7/**: ARM7 coprocessor firmware. Contains `main.c` plus `adx/adx_7.c` (ARM7 side of the ADX audio streaming library). Links against `dswifi7` and `nds7`.

- **arm9/**: ARM9 main application. The bulk of the codebase.
  - `arm9/source/` — all C/C++ source files:
    - `main.cpp` — entry point
    - `engine.cpp` — core game/rendering engine
    - `global.cpp` — global state
    - `animStream.cpp` — animated sprite streaming
    - `fonts.cpp` — font rendering (stb_truetype fixed-point)
    - `colors.cpp`, `content.cpp`, `cfgFile.cpp`, `settings.cpp`
    - `mem.c` — memory utilities
    - `wav_nds.c` — WAV audio playback (dr_wav)
    - `adx/adx_9.c` — ARM9 side of ADX music streaming
    - `courtroom/` — background, character, chatbox, evidence, shout, WTCE animations
    - `sockets/` — `aotcpsocket.cpp` (raw TCP), `aowebsocket.cpp` (WebSocket via mongoose)
    - `ui/` — all UI screens: main menu, server list, direct connect, WiFi connect, court, settings, character select, music list, area list, court record, evidence, judge controls, OOC chat, IC chat log, moderator dialog, and more
    - `websocket/mongoose.c` — mongoose WebSocket/HTTP library
    - `wifikb/wifikb.cpp` — on-screen WiFi keyboard
  - `arm9/gfx/` — source PNG + grit config files for all UI graphics. Converted to `.img.bin` at build time and output to `fat/data/ao-nds/ui/`.
  - `arm9/include/` — all headers; also vendored: `rapidjson/`, `stb_truetype.h`, `stb_truetype_fixed.h`, `dr_wav.h`, `mongoose.h`, `utf8/`, `mini/ini.h`.

- **common/**: Shared code between ARM7 and ARM9. Contains `libadx.h`, the shared header for inter-CPU audio streaming.

- **converter/**: Python-based asset conversion tooling (`main.py`, `chatbox-converter.py`, `conversion.py`, `images.py`). Includes a bundled Windows grit binary and FreeImage DLLs.

- **fat/**: SD card filesystem content. `fat/data/ao-nds/` must be placed on the DS's microSD card. The `ui/` subfolder is populated at build time by the grit conversion step.

## OAM / Background Layer Layout

From `oam and bg notes.txt` (reverse-engineered from Apollo Justice via DeSmuME):

- **OAM (sprites):** slots 0-23 used for text rendering; slots 30-127 for bench and character sprites.
- **BG layers (sub screen):** BG 1 = chatbox; BG 3 = court background.

## Asset Conversion for SD Card

Assets from a standard Attorney Online installation must be converted before being placed on the SD card (`fat/data/ao-nds/`).

**Sound effects** — convert to 32000 Hz, mono WAV:
```bash
ffmpeg -i "input.wav"  -ar 32000 -ac 1 -b:a 96k "output.wav"
ffmpeg -i "input.opus" -ar 32000 -ac 1 -b:a 96k "output.wav"
```

**Music** — convert to 22050 Hz, stereo MP3 with blank title metadata (title tag must be padded with spaces so the DS ADX player does not misread it):
```bash
ffmpeg -i "input.mp3"  -ar 22050 -ac 2 -b:a 96k -metadata title="                           " -y "output.mp3"
ffmpeg -i "input.ogg"  -ar 22050 -ac 2 -b:a 96k -metadata title="                           " -y "output.mp3"
ffmpeg -i "input.opus" -ar 22050 -ac 2 -b:a 96k -metadata title="                           " -y "output.mp3"
```

Background images, sprite sheets, and evidence images are handled by the Python `converter/` tooling.

## Libraries and Vendored Dependencies

| Library | Purpose |
|---|---|
| BlocksDS / libnds | DS hardware abstraction, build system |
| dswifi | DS WiFi networking |
| libadx-nds | ADX format music streaming on DS hardware |
| dr_wav | WAV file decoding |
| stb_truetype (fixed-point variant) | Font rasterisation without FPU (DS ARM9 has no hardware FPU) |
| rapidjson | JSON parsing (public server list) |
| mINI | INI config file parsing |
| mongoose | WebSocket client support |
| utfcpp | UTF-8/16/32 string handling |

**Note on stb_truetype:** The project uses a modified version that replaces floating-point operations with fixed-point math to compensate for the DS ARM9's lack of a hardware FPU.

**Note on music metadata:** Music ID3 title tags must be padded with spaces to avoid a parsing issue in the ADX player.
