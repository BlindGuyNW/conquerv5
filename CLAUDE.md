# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout and licensing boundary

This repo holds two copies of Conquer v5, a 1987-era Unix ncurses strategy game:

- `original/` — the original USENET distribution under its restrictive license. **Never modify anything here.** It is a historical archive (see `LICENSES/LicenseRef-Original-Restrictive.txt`).
- `gpl-release/` — the GPLv3+ relicensed tree. **All development happens here.**
- `packaging/`, `scripts/`, `.github/workflows/ci.yml` — packaging and CI, GPLv3+.

Every new or modified file in `gpl-release/`, `packaging/`, or `scripts/` needs an SPDX header (`// SPDX-License-Identifier: GPL-3.0-or-later` plus the copyright block used by neighboring files). `REUSE.toml` at the root declares path-based annotations; `reuse lint` must stay clean.

## Build

All build commands run from `gpl-release/`. The two configuration templates must be copied before the first build:

```bash
cd gpl-release
cp Makefile.top Makefile              # top-level Makefile is generated, not tracked
make Makefiles                        # expands %%TOKEN%% templates into per-dir Makefiles; also generates Include/header.h
make build                            # builds Src (conquer, conqrun), Auxil (conqsort), Docs
make install                          # installs to $PREFIX
```

`Include/header.h` is generated from `Include/header.h.dist` by `make Makefiles`/`make build` (an awk pass that substitutes `LOGIN`). Edit `header.h.dist`, not `header.h` — the latter is gitignored and regenerated.

The build is a template-expansion system, not autotools: `Makefile.top` computes every variable (platform, compiler, flags, libs, paths) and `make Makefiles` seds them into `Include/Makefile.inc` → `Include/Makefile`, `Src/Makefile.src` → `Src/Makefile`, `Auxil/Makefile.aux` → `Auxil/Makefile`, `Docs/Makefile.dcm` → `Docs/Makefile`. **After changing anything in `Makefile.top` you must re-run `make Makefiles`** or the sub-Makefiles keep the old values.

Useful targets and knobs:

```bash
make info                     # print detected platform, compiler, flags, libs
make clean                    # remove .o and editor temp files
make clobber                  # also remove generated Makefiles, header.h, binaries
make BUILD=release build      # -O2 -DNDEBUG, stripped binaries
cd Docs && make docs          # regenerate .doc (nroff) and .ps (groff) from .nr sources
```

`BUILD` defaults to `debug` (`-g -O0`), and when `$(CC) --version` identifies as gcc or clang the debug build additionally enables `-fsanitize=address -fsanitize=undefined`. Platform, libc (musl/Alpine), ncurses vs curses, and termcap availability are all autodetected via `uname -s` and file probes — do not hardcode `LIBS` or `SYSFLG`; extend the detection blocks instead.

Prerequisites on Debian/Ubuntu: `build-essential libncurses-dev libcrypt-dev` (plus `groff ghostscript` for docs).

Build artifacts (objects, binaries, generated `Makefile`s, `Include/header.h`, `Docs/*.doc`, `Docs/*.ps`) are covered by `gpl-release/.gitignore`. The default install `PREFIX` for a non-root user is `$HOME/conquerv5`, which is this repo's root, so `make install` drops `bin/` and `share/` into the checkout; those are ignored by the root `.gitignore`.

## Tests

There is no unit test suite. Verification is what CI does (`.github/workflows/ci.yml`):

1. Build from a clean checkout, confirm `gpl-release/Src/conquer` and `gpl-release/Src/conqrun` exist.
2. Smoke test: `./gpl-release/Src/conquer -h` and `./gpl-release/Src/conqrun -h`.
3. Package builds (both require Docker):
   ```bash
   scripts/build-melange.sh    # Alpine APK  -> packages/
   scripts/build-debian.sh     # Debian DEB  -> packages/debian/
   scripts/test-alpine.sh      # install/inspect the built APK in a container
   scripts/test-debian.sh      # install/inspect the built DEB in a container
   ```

A commit message containing `[release]` on master triggers the release job.

## Architecture

Two executables share one object pool, distinguished by a **one-letter suffix on every source and header filename**:

- `*G.c` / `*G.h` — the player-facing curses UI, linked into `conquer`. Entry point `Src/mainG.c`.
- `*A.c` / `*A.h` — the administrative/world-update program, linked into `conqrun`. Entry point `Src/mainA.c`.
- `*X.c` / `*X.h` — shared engine code, linked into **both** binaries.

`Src/Makefile.src` encodes this: `conquer = GOBJS + XOBJS + jointG.o`, `conqrun = AOBJS + XOBJS + jointA.o`. The `jointG.c`/`jointA.c` pair is the seam — each provides program-specific definitions of symbols the shared `X` code references, so shared code can call into the UI or into the batch updater without knowing which binary it is in. When adding a shared function that must behave differently in the two programs, add it to both joint files rather than `#ifdef`-ing inside `X` code, and register the new source in the `GFILS`/`AFILS`/`XFILS` lists in `Makefile.top` (then re-run `make Makefiles`).

Header layering mirrors this: `dataX.h` (shared globals and constants, included by everything) ← `dataG.h` (UI globals, includes `dataX.h` + `keybindG.h`) and `dataA.h` (admin globals, includes `dataX.h`). `header.h` holds all compile-time game configuration (`ABSMAXNTN`, `COMPRESS`, `MANY_UNITS`, `HUGE_MAP`, `SPOOLDIR`, `OWNER`, `LOGIN`, editor/mailer paths). `paramX.h` defines the `PARM_0`…`PARM_5` macros used in every function definition — a K&R/ANSI portability shim that is still pervasive; match it when editing existing functions. The `fileX.h`/`fileG.h`/`fileA.h` headers are prototype indexes (originally cextract-generated) and must be kept in sync when a function's signature changes.

Two paths are baked in at compile time by `Src/Makefile.src` rules rather than by `header.h`: `customX.c` gets `-DDEFAULTDIR` (game data dir, `$PREFIX/share/conquerv5`) and `-DEXEDIR`; `miscA.c` gets `-DCONQ_SORT` (path to the `Auxil` sort helper). `Src/customX.c` resolves the effective data directory at runtime (`init_datadir`), letting `-d DIR` select alternate worlds.

`Auxil/` builds `conqsort`, a standalone helper invoked by the admin program; `Docs/` builds an `ezconv` filter that turns `*.nr` nroff sources into the `.doc` files and the in-game help text.

## Running the game

Binaries install setuid (`chmod 4751`) with a shared game data directory, so a normal dev loop uses a non-root `PREFIX` (defaults to `$HOME/conquerv5` when not root):

```bash
conqrun -m            # create a new world (reads the `nations` file for NPCs)
conqrun -a            # add a player interactively
conqrun -x            # process one turn (the cron-driven update)
conqrun -T            # maintenance mode: block logins
conqrun -E            # read+rewrite datafiles (v4 -> v5 conversion)
conqrun -d DIR        # operate on an alternate game directory
conquer               # play; `conquer -n god` for admin powers
```

`conqrun` option parsing lives in `Src/mainA.c` (`getopt` string `"ACEIQTZncmaxo:r:d:"`), `conquer`'s in `Src/mainG.c`.

## Code conventions

This is 1980s C. Match the surrounding style rather than modernizing opportunistically: `PARM_n` function definitions, K&R-ish brace and comment style, `/* ... */` comments, tab indentation in Makefiles, typedef'd struct pairs (`FOO_STRUCT, *FOO_PTR`). Fixes for modern toolchains (missing prototypes, implicit declarations, feature-test macros) belong in the build flags or headers where possible, as in commit `05ae34b`.

## Goals of this fork

This is a personal fork (BlindGuyNW/conquerv5) for experimenting with the game, not a preservation project. The upstream repo's goal was a behavior-preserving relicensing; this fork's goal is a good **solo 4X experience**. Changes to game mechanics, NPC behavior, events, win conditions, and the UI are welcome when they make solo play better, and compatibility with the original rules is not a constraint. The `original/` tree stays untouched as the reference for what the game did before any changes. Nothing here is expected to be sent upstream, so there is no need to keep diffs minimal or upstream-friendly.
