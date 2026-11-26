# Zelda 3 for PlayStation 2

![Platform: PlayStation 2](https://img.shields.io/badge/platform-PlayStation%202-003791)
![License: MIT](https://img.shields.io/badge/license-MIT-green)
![Status: experimental](https://img.shields.io/badge/status-experimental-orange)

A native **PlayStation 2** port of *The Legend of Zelda: A Link to the Past*, built on top of [snesrev/zelda3](https://github.com/snesrev/zelda3), a reverse-engineered C reimplementation of the game. This is **not** an emulator: the game logic runs as native C code on the PS2's Emotion Engine (EE), with a PS2-specific platform layer for video (gsKit), input (libpad), audio (AUDSRV) and memory-card saves.

> **No game data is included.** You must supply your own, legally obtained copy of the US ROM and extract the assets yourself. See [Legal notice](#legal-notice).

> **Experimental and AI-assisted.** Most of the PS2-specific code was written with the help of AI tools. Please read [AI-assisted development](#ai-assisted-development) and [Known limitations](#known-limitations) before relying on it.

## Table of contents

- [Features](#features)
- [Legal notice](#legal-notice)
- [Quick start](#quick-start)
- [Getting the game assets](#getting-the-game-assets)
- [Building](#building)
- [Running](#running)
- [Controls](#controls)
- [Port menu](#port-menu)
- [Video modes](#video-modes)
- [Saves and memory cards](#saves-and-memory-cards)
- [Audio](#audio)
- [Repository layout](#repository-layout)
- [Testing](#testing)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [AI-assisted development](#ai-assisted-development)
- [Credits](#credits)
- [License](#license)

## Features

- **Native game logic.** The complete game, from start to finish, running as C code (no SNES emulation of the CPU).
- **Native PS2 input** through libpad: ports 1 and 2, D-pad and left analog stick, adjustable dead zone, button swaps.
- **gsKit video, NTSC output:** 480i (default), 240p and 480p (704×480 progressive, with a confirmation screen); 4:3 or 1:1-pixel screen format; optional soft filter; colour saturation; flash reduction; FPS counter.
- **Partial GS acceleration of the PPU.** Tiles, sprites and colour math are drawn by the Graphics Synthesizer whenever possible; scaling, filtering and menu drawing also use the GS. Mode 7, windows and mosaic still use the EE path.
- **Fixed 60 FPS presentation** with frameskip permanently off.
- **Native audio** through AUDSRV, using a bundled and patched IOP module (`audsrv.irx`) that is embedded in the ELF, so no extra file is needed.
- **Memory-card saves:** checksummed, alternating banks for game progress and preferences, plus a save icon.
- **Port menu** (Start + Select) with video, audio, controls and system pages. *The menu text is currently in Portuguese* (names are translated below).
- **ISO authoring script** that builds and verifies a bootable disc image.
- **Host-side test suite** (unit tests, replay regression harness, GS PPU tests) that runs on a PC without a PS2.

## Legal notice

This repository contains **source code only**. It does not contain, and must never contain, the original game's ROM, graphics, music, text, or any file extracted from them (this includes `zelda3_assets.dat`).

*The Legend of Zelda* and *A Link to the Past* are trademarks and copyrighted works of Nintendo. This project is not affiliated with, endorsed by or sponsored by Nintendo. Use it only with a copy of the game that you own.

## Quick start

```bash
# 1. Install the ps2dev toolchain (see "Building" for details) and set PS2DEV, PS2SDK and GSKIT.
# 2. Put your US ROM in the repository root as zelda3.sfc, then extract the assets:
pip install -r requirements.txt
python3 assets/restool.py --extract-from-rom      # creates zelda3_assets.dat

# 3. Build the ELF:
make -f Makefile.ps2                              # creates zelda3.elf

# 4. Copy zelda3.elf and zelda3_assets.dat to the same folder and launch the ELF on your PS2 / PCSX2.
```

## Getting the game assets

The game needs a file called `zelda3_assets.dat`, which is generated from your own ROM. The ROM itself is not needed afterwards.

**Supported ROM:** *Legend of Zelda, The - A Link to the Past (USA)*, SHA-1 `6D4F10A8B10E10DBE624CB23CF03B88BB8252973`. A 512-byte SMC header is detected and ignored. Translations, hacks and expanded ROMs are **not** supported.

**Option A: command line (Python 3)**

```bash
pip install -r requirements.txt                   # Pillow and PyYAML
cp /path/to/your/rom.sfc zelda3.sfc               # repository root
python3 assets/restool.py --extract-from-rom      # writes zelda3_assets.dat (about 680 KB)
```

**Option B: browser converter** (if `tools/conversor-assets-zelda3.html` is included in this repository)

Open the file in a desktop browser, choose your US ROM and download `zelda3_assets.dat`. It runs the same Python scripts locally through [Pyodide](https://pyodide.org/) (WebAssembly), so the ROM never leaves your browser. An internet connection is needed the first time to download Pyodide (about 10 MB).

## Building

### Requirements

| Component | Notes |
| --- | --- |
| [ps2dev](https://github.com/ps2dev/ps2dev) toolchain | EE compiler (`mips64r5900el-ps2-elf-gcc`), IOP compiler (`mipsel-ps2-irx-gcc`), PS2SDK, gsKit and the PS2SDK ports (SDL2, `ps2_drivers`) |
| `make` | GNU make |
| Python 3 + `Pillow`, `PyYAML` | only for extracting the assets |
| `pycdlib` | only for building an ISO |

Last verified with the official ps2dev release tarball (GCC 15.2.0).

### 1. Install the toolchain

Any complete ps2dev installation works. One way on Linux (x86-64):

```bash
curl -L -o ps2dev.tar.gz \
  https://github.com/ps2dev/ps2dev/releases/latest/download/ps2dev-ubuntu-latest.tar.gz
sudo tar -xzf ps2dev.tar.gz -C /usr/local         # creates /usr/local/ps2dev
```

The official `ps2dev/ps2dev` Docker image or a toolchain built from source are alternatives. Then set up the environment:

```bash
export PS2DEV=/usr/local/ps2dev
export PS2SDK=$PS2DEV/ps2sdk
export GSKIT=$PS2DEV/gsKit
export PATH=$PATH:$PS2DEV/bin:$PS2DEV/ee/bin:$PS2DEV/iop/bin:$PS2DEV/dvp/bin:$PS2SDK/bin
```

### 2. Compile

```bash
git clone https://github.com/<your-user>/<your-repo>.git
cd <your-repo>
make -f Makefile.ps2 -j"$(nproc)"                 # produces zelda3.elf
```

- The ELF is stripped by default (about 3 MB). For debug symbols: `make -f Makefile.ps2 DBGINFO="-gdwarf-2 -gz"`.
- `make -f Makefile.ps2 clean` removes the objects and the IOP module build.
- The AUDSRV IOP module is prebuilt (`ps2/iop/audsrv/audsrv.irx`) and embedded in the ELF through `ps2/audsrv_embed.S`. It is rebuilt automatically only when its sources change (this needs the IOP compiler and `iopfixup` from the full toolchain).

### 3. Build an ISO (optional)

```bash
pip install pycdlib
python3 ps2/make_iso.py zelda3.elf zelda3_assets.dat zelda3.iso
# or: ps2/make_iso.sh zelda3.elf zelda3_assets.dat zelda3.iso
```

The script creates an ISO 9660 disc with `SYSTEM.CNF`, `ZELDA3.ELF` and `Z3ASSETS.DAT`, and verifies every resulting disc entry. The disc is read-only, so a memory card is required to keep your progress.

## Running

Put `zelda3.elf` and `zelda3_assets.dat` in the same folder and launch the ELF with any PS2 homebrew loader (uLaunchELF, OPL, a USB stick, `ps2client`, ...), or with PCSX2 / ARMSX2. The game looks for `zelda3_assets.dat` in this order:

1. the folder the ELF was started from,
2. `cdrom0:\` and `cdfs:/` (disc),
3. the current directory,
4. `host:` (PCSX2 / ARMSX2: the folder containing the ELF),
5. `mass:/zelda3/` and `mass:/` (USB).

The output is **NTSC** (interlaced by default). The game starts straight away; the port menu is opened during play. The PS2 build does not read `zelda3.ini`; settings come from the port menu and are stored on the memory card.

## Controls

| PS2 | Action |
| --- | --- |
| D-pad / left analog stick | Move |
| ✕ Cross | Sword (SNES B) |
| ○ Circle | Action / run (SNES A) |
| □ Square | Item (SNES Y) |
| △ Triangle | Map (SNES X) |
| Start | Inventory / pause |
| Select | The game's own save menu |
| L1 / R1 | SNES L / R (optional item switching, see menu) |
| **Start + Select** | **Open the port menu** |

Port 1 has priority; port 2 is used if port 1 has no controller, and the active controller stays selected until it is unplugged. Cross/Circle and Square/Triangle can be swapped and the dead zone adjusted in the menu.

## Port menu

Open it during play with **Start + Select**. Navigate with the D-pad or analog stick, change values with left/right, confirm with ✕, cancel with ○. Release all buttons and centre the stick after leaving the menu so the game does not receive the confirmation press.

| Page | Options (original Portuguese label in parentheses) |
| --- | --- |
| **Video** (VIDEO) | Video mode (MODO DE VIDEO): 480i / 240p / 480p · Show FPS (MOSTRAR FPS) · Screen format (FORMATO DA TELA): 4:3 or 1:1 pixels · Soft filter (FILTRO SUAVE) · Saturation (SATURACAO) 0–200 % · Reduce flashes (REDUZIR CLAROES) |
| **Audio** (AUDIO) | Volume (VOLUME) 0–100 % · Stereo sound (SOM ESTEREO) · Low-health alert (ALERTA DE VIDA) |
| **Controls** (CONTROLES) | Analog stick (ANALOGICO) · Dead zone (ZONA MORTA) · Swap ✕/○ (TROCAR X / CIRCULO) · Swap □/△ (TROCAR QUAD / TRI) · Items on L1/R1 (ITENS EM L1 / R1) |
| **System** (SISTEMA) | Save now (SALVAR AGORA) · Restore defaults (RESTAURAR PADRAO) · Discard changes (DESCARTAR ALTERACOES) |
| **Quit** (SAIR DO JOGO) | Back (VOLTAR) · Quit game (SAIR DO JOGO), unsaved progress is lost |

"Resume game" (CONTINUAR JOGO) applies and saves the options. Saving preferences does **not** save game progress: use the game's own save command for that.

## Video modes

| Mode | Description |
| --- | --- |
| **480i** (default) | NTSC interlaced |
| **240p** | 15 kHz progressive, one SNES scanline per screen line |
| **480p** | 704×480 progressive (component/VGA cable). Selecting it shows a confirmation screen: press ✕ within 10 seconds to keep it, otherwise the previous mode returns automatically. The memory card only stores 480p after you confirm it. |

In 480p the GS still draws tiles and sprites, but lines using colour math and the GS soft filter fall back to the EE path.

Presentation is fixed at **60 FPS** (about 59.94 on NTSC): one drawing pass per game/audio update. **Frameskip is permanently off**, so no drawing pass is dropped to catch up. If the console cannot keep up, the game runs slower instead. Neither setting has a menu entry; older preference files are corrected automatically on load.

## Saves and memory cards

Data is stored in `mc0:/ZELDA3-LTTP/` or `mc1:/ZELDA3-LTTP/` (the menu shows which one is used).

| File | Content |
| --- | --- |
| `sram.a`, `sram.b` | Game progress (alternating copies with generation counter and checksum) |
| `zelda3.ini`, `zelda3.alt.ini` | Preferences (alternating copies, checksummed) |
| `sram.dat` | Legacy save, imported when no valid new bank exists and kept untouched |
| `zelda3.icn`, `icon.sys` | Memory-card icon |

- If one copy is incomplete (for example, after a power loss), the other valid copy is loaded.
- A card that already holds game progress takes priority over a card that only holds preferences. The chosen card stays fixed for the whole session.
- If the selected card is removed, writing fails; the port never silently switches to another card. The menu lets you apply settings without saving.
- With no card at startup, it tries a writable `saves/` folder next to the game data.
- **Back up the whole folder before migrating saves.** To import another `sram.dat` later, back up the new banks and remove `sram.a` / `sram.b` before starting the game.

## Audio

Audio uses the native AUDSRV server. The bundled copy in `ps2/iop/audsrv` is derived from the PS2SDK sources, with these changes: the PCM ring starts empty, it emits silence (without consuming unwritten data) when starved instead of replaying old samples, and the EE clock matches the resampler's effective 341-input / 512-output sample ratio. It targets the game's 16-bit, 32 kHz mono/stereo PCM stream; other AUDSRV formats are not validated. See `ps2/iop/audsrv/README.md` for the full list of changes and licences.

## Repository layout

| Path | Purpose |
| --- | --- |
| `src/`, `snes/` | The zelda3 reimplementation and the PPU/DSP code (upstream, with PS2 patches) |
| `assets/` | Asset extraction and packing scripts (`restool.py` and modules) |
| `ps2/ps2_video.c`, `ps2_gsppu.c`, `ps2_indexed.c` | gsKit presentation and the GS-accelerated PPU path |
| `ps2/ps2_pad.c` | libpad input |
| `ps2/ps2_audio.c` | AUDSRV PCM output |
| `ps2/ps2_compat.c` | File access, data-directory discovery, memory-card saves |
| `ps2/ps2_menu.c`, `ps2_font.c`, `ps2_settings.c` | Port menu, its font and the settings file |
| `ps2/iop/audsrv/` | Bundled and patched AUDSRV IOP module |
| `ps2/make_iso.py` | ISO builder and verifier |
| `ps2/host-test/`, `ps2/gsppu-test/` | Host-side tests |
| `third_party/` | Opus decoder (stripped), stb, GL loader |
| `Makefile.ps2` | PS2 build |
| `PS2.md`, `PS2.pt-BR.md`, `ps2/LEIA-ME.txt`, `ps2/VALIDACAO.md` | Additional notes (English / Portuguese) and validation records |

## Testing

The PS2 build compiles with ps2dev, and the host-side tests run on a normal PC (gcc, Python 3; the replay harness also needs `libsdl2-dev`):

```bash
# Unit tests: settings, menu, files / memory card, frame pacing, audio, resampler
ASAN_OPTIONS=detect_leaks=0 sh ps2/host-test/run_unit_tests.sh
```

The replay harness (`ps2/host-test/run_replays.sh`) plays the reference replays from `saves/ref` with SDL's dummy drivers and reports misaligned memory accesses; see `ps2/host-test/README.md`. The R5900 raises an exception on misaligned loads and stores, so any 16/32/64-bit access that may hit an odd address must use the unaligned types from `src/types.h` (`uint16u`, `int16u`, `uint32u`, `uint64u`).

## Known limitations

- **Experimental.** The code builds and passes the host-side tests. Timing, audio sync and GS presentation on real hardware and across emulators are still being evaluated: please report what you see, with your setup.
- **Partial hardware acceleration.** Part of the PPU (composition, effects, Mode 7) and the SNES DSP/SPC synthesis still run on the EE.
- **Fixed 60 FPS with frameskip off:** when the EE cannot keep up, the game slows down.
- **NTSC output only.**
- **English dialogue only.** The upstream language packs cannot be selected on PS2 because the INI file is not read.
- **The port menu is in Portuguese.** Translations of the menu strings are welcome.
- **Only the unmodified US ROM** is supported for asset extraction.
- **No log file.** The port does not write logs; console output (for example `printf`) is only visible on the EE console / emulator log.
- The soft filter disables palette expansion while keeping GS filtering.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Returns to the loader or shows nothing | `zelda3_assets.dat` must be next to the ELF (or in one of the locations listed in [Running](#running)) and must come from the US ROM |
| Game runs in slow motion | The console cannot sustain 60 FPS. Try 240p or 480i, turn the soft filter off, avoid 480p |
| Progress is lost | Use the game's save command (Select), and make sure a memory card is present. Discs are read-only |
| Menu does not open | Press Start and Select together during play |
| Preferences do not stick | Use "Save now" or "Resume game", and check the memory-card folder shown in the menu |

## Contributing

Issues and pull requests are welcome.

- Never submit ROM data, extracted assets or other copyrighted game files.
- Keep the R5900 alignment rule in mind (see [Testing](#testing)).
- Run `ps2/host-test/run_unit_tests.sh` and make sure `make -f Makefile.ps2` still builds before opening a pull request.
- If you used an AI tool to write or substantially edit your contribution, say so in the pull request description, and review and test the code yourself.

## AI-assisted development

This port was developed with substantial help from AI coding assistants, mainly **Claude** (by [Anthropic](https://www.anthropic.com/)). In the interest of transparency:

- **What the AI helped with:** writing and refactoring the PS2 platform layer in `ps2/` (gsKit video and the GS-accelerated PPU path, libpad input, AUDSRV audio and the patched IOP module, memory-card saves, the settings file and the port menu), the PS2 patches to the upstream sources, the host-side tests, the build and ISO tooling, the asset-conversion web page, and the documentation, including this README and the Portuguese translation of the notes.
- **What humans did:** the repository owner defined the goals and features, reviewed the changes, built the project, ran it and reported problems. All decisions about what ships were made by a human.
- **What the AI did not do:** the zelda3 reimplementation itself (`src/`, `snes/`, `assets/`) is the work of its original authors and contributors (see [Credits](#credits)). No game assets were generated or altered by AI; all graphics, music and text come from the user's own ROM.
- **Caveats:** AI-written code can contain subtle bugs and has not been independently audited. Real-hardware validation is limited, and some behaviour has only been verified by host-side tests and mocked drivers. Treat this project as experimental, read the code before trusting it with anything you care about (including save files, so back them up), and report problems.

## Credits

- **[snesrev](https://github.com/snesrev) and contributors:** [zelda3](https://github.com/snesrev/zelda3), the reverse-engineered C reimplementation this port is based on.
- **[elzo_d](https://github.com/elzo-d):** [LakeSnes](https://github.com/elzo-d/LakeSnes), the origin of the PPU and DSP code.
- **Spannerisms** and the other authors of the Zelda 3 disassemblies and documentation that named countless functions and variables.
- **[ps2dev](https://github.com/ps2dev) community:** [PS2SDK](https://github.com/ps2dev/ps2sdk) (including the AUDSRV sources the bundled IOP module derives from), [gsKit](https://github.com/ps2dev/gsKit), the PS2SDK ports (SDL2, `ps2_drivers`) and the toolchain.
- **Metroid Zero Mission DX (PS2):** the 480p video-mode setup follows its approach.
- **[Opus](https://opus-codec.org/)** by CSIRO, Xiph.Org Foundation, Jean-Marc Valin and Gregory Maxwell (stripped decoder in `third_party/`), used by the upstream MSU-1 audio code.
- **[stb](https://github.com/nothings/stb)** by Sean Barrett (`third_party/stb`).
- **[Pyodide](https://pyodide.org/)** (MPL-2.0), used by the optional browser asset converter.
- **Nintendo**, creators of *The Legend of Zelda: A Link to the Past*. All rights to the original game belong to them.
- **AI assistance:** Claude by Anthropic (see [AI-assisted development](#ai-assisted-development)).
- **PS2 port:** maintained by ([gogeta456<gogeta456>](https://github.com/gogeta456)).

## License

The source code is released under the **MIT License**. See [`LICENSE.txt`](LICENSE.txt) (Copyright (c) 2022 snesrev, Copyright (c) 2021 elzo_d). Third-party components keep their own licences: Opus (BSD-3-Clause, included in `LICENSE.txt`), and the bundled AUDSRV module (AFL 2.0 and LGPL 2.0, see `ps2/iop/audsrv/`).

The game's ROM, graphics, music and text are **not** covered by this license and are not distributed here.
