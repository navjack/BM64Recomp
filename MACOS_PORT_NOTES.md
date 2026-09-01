# macOS port notes — BM64Recomp (Bomberman 64)

All three previously-recorded blockers are resolved. An earlier note claimed `lib/sf64` was unused
and that `BM64Syms` was empty upstream — **both were wrong**, see below.

## ROM

Base dump: `Bomberman 64 (U) [!].z64`, 8,388,608 bytes, md5 `093058ece14c8cc1a887b2087eb5cfe9`,
internal name `BOMBERMAN64`, cart code `NBME`. This matches `src/main/main.cpp:356`.

The build needs a **decompressed** variant (`bm64.decomp.us.z64`); the app itself uses the standard
compressed ROM at runtime.

## Blocker 1 — submodules were never checked out

`git submodule status` was misleading. Every submodule had a complete packfile under
`.git/modules/...` but **no index**: the clones succeeded and the checkout pass never ran, aborted
partway by the dead `lib/sf64` (below). `BM64Syms` looked empty for this reason, not because
anything was missing upstream — its blobs were already local.

Plain `--init` no-ops (git sees the correct SHA at HEAD). Force it, skipping the dead one:

```bash
git -c submodule.lib/sf64.update=none submodule update --init --recursive --force
```

This needs no network for the already-packed modules. Afterwards `BM64Syms/dump.toml` (5.9 MB) and
`data_dump.toml` (1.3 MB) are present, as are rt64's contrib tree and `lib/sse2neon`.

## Blocker 2 — lib/sf64 is REQUIRED, and upstream is dead

An earlier note claimed nothing referenced sf64. That is true of `CMakeLists.txt` but **false
overall**: `patches/Makefile:9` adds three include paths from it —
`../lib/sf64/include`, `../lib/sf64/include/libultra`, `../lib/sf64/src` — and `PatchesLib` is on
the critical path to the final binary (`CMakeLists.txt:126-130`, `139-150`, `110-113`, `356-357`).
`-nostdinc` is set, so there is no system fallback. Headers needed: `ultra64.h`,
`PR/ultratypes.h`, `PR/os_pi.h`, `PR/sched.h`, `libc/stdarg.h`, `libc/stddef.h`.

`https://github.com/sonicdcer/sf64.git` (the pinned URL) now 404s.

**A genuine mirror exists** — no header substitution or hand-written shims required.
`jnmartin84/sf64-dc`, a Dreamcast port that retained the upstream history, contains the exact
pinned commit `b870a09cf33e882463dfecedaf917a9505b79cec`. Verified via the GitHub API (HTTP 200,
where `sonicdcer/sf64` and case variants return 404, and `HarbourMasters/Starship` does not have
the commit).

`.gitmodules` here is repointed at **`https://github.com/navjack/sf64-dc.git`** — a personal fork
kept as insurance against the mirror disappearing too. Confirmed to carry the same pinned commit.

Fetch just that commit (no full clone needed):

```bash
rm -rf lib/sf64 && mkdir -p lib/sf64 && cd lib/sf64
git init -q . && git remote add origin https://github.com/navjack/sf64-dc.git
git fetch -q --depth 1 origin b870a09cf33e882463dfecedaf917a9505b79cec
git checkout -q FETCH_HEAD
```

All six headers resolve. Note `ultra64.h` is at `include/libultra/ultra64.h`, which is why the
Makefile's second include path (`include/libultra`) matters — it is **not** at `include/ultra64.h`.

## Blocker 3 — the decompressed ROM

Nothing in BM64Recomp produces it; CI copies a pre-made file from a private repo. Per `BUILDING.md`,
build the external decomp's standalone, dependency-free decompressor (no splat/IDO toolchain):

```bash
git clone --depth 1 https://github.com/Bomberhackers/bm64.git ../bm64-decomp
cd ../bm64-decomp
cp "/path/to/roms/Bomberman 64 (U) [!].z64" baserom.z64
gmake -C tools bm64decompress      # naming the target avoids building n64crc, which uses an obsolete -s flag
./tools/bm64decompress baserom.z64 baserom.decomp.z64
cp baserom.decomp.z64 ../bm64/bm64.decomp.us.z64
```

**Verification that matters:** output is **10,090,784 bytes = 0x99F920**, exactly the maximum ROM
extent declared in `BM64Syms/dump.toml` (8 MB base with overlays appended). A size that merely
"looks right" is not enough — check it against that extent.

*Ignore* `src/game/rom_decompression.cpp`. It is dead Star Fox 64 fork residue: it sanity-checks
12 MB and cart code `NFXE`, is declared in `include/zelda_game.h:11`, and is called from nowhere.
Do not try to use it to produce the decompressed ROM.

## Generate and build

```bash
D=lib/N64ModernRuntime/N64Recomp
cmake -S "$D" -B "$D/build" -DCMAKE_BUILD_TYPE=Release && cmake --build "$D/build" -j8
cp "$D/build/N64Recomp" "$D/build/RSPRecomp" .
./N64Recomp bm64.us.toml     # 74,447 functions -> 1,489 files in RecompiledFuncs/
./RSPRecomp aspMain.toml

cmake -B build -DCMAKE_BUILD_TYPE=Release -DCMAKE_OSX_DEPLOYMENT_TARGET=11.0 \
      -DPATCHES_C_COMPILER=/opt/homebrew/opt/llvm/bin/clang \
      -DPATCHES_LD=/opt/homebrew/bin/ld.lld .
cmake --build build -j8
```

Apple clang has **no MIPS target**; `patches/` requires Homebrew LLVM. Note CI generates code with a
*separately cloned* N64Recomp (`Mr-Wiseguy/N64Recomp` @ `98bf104b`) rather than the vendored copy —
if the generated sources ever fail to compile with errors that look like missing includes or
incomplete types, that version mismatch is the first suspect.

## Fixes applied

**`patches/Makefile`** — added `-Wno-error=incompatible-pointer-types`. `required_patches.c` has
numerous loose-C pointer conversions (`Gfx *` from `Gfx (*)[1536]`, `u8 (*)[]` to `u8 *`, …). Clang
16+ promotes these to errors; upstream CI uses clang-18 where they remain warnings. Demoted rather
than silenced, so the real mismatches stay visible.

**`CMakeLists.txt`** — added `add_compile_options("SHELL:-include stdlib.h")` to the existing Apple
block. rt64 forces `HLSLPP_SCALAR` on Apple Silicon and hlslpp's `scalar.h` calls `labs()` without
including `<stdlib.h>`. `SHELL:` prevents CMake de-duplicating the repeated `-include` token.

**`.github/macos/apple_bundle.cmake`** — the same three upstream bugs found in every sibling that
shares this file (banjo, bmhero, snowboard-kids-2):

1. *Icon dimensions* — the 512x512 source was copied to both `icon_512x512.png` and
   `icon_512x512@2x.png`; `iconutil` validates dimensions and the `@2x` entry must be 1024x1024.
   Now resized with `sips`.
2. *Missing output directory* — `iconutil` writes into `${CMAKE_BINARY_DIR}/resources/`, which
   nothing creates, and reports only a generic "Failed to generate ICNS" when it is absent.
3. *SDL3 not vendored* — `fixup_bundle` follows only link-time dependencies, but sdl2-compat
   `dlopen`s `libSDL3.dylib` at load time. Without it the app aborts in `dllinit` with
   "Failed loading SDL3 library" before `main()` runs.

These were applied **preemptively** here based on the sibling projects, rather than discovered one
build at a time.

## macOS support status

Mature. `src/main/support_apple.mm` already exists (68 lines, implementing everything
`include/zelda_support.h` declares), Apple branches are present throughout `CMakeLists.txt`
(3-5, 24-26, 208-210, 250-256, 301-312, 383-390), and there is a `macos-14` CI job. The three
`static_assert(false)` sites all have real `__APPLE__` branches above them and are unreachable on
Darwin. No `/proc/self/exe` anywhere.

`.github/macos/ld64` is a python wrapper requiring `macholib` — available here via Xcode's python3.
