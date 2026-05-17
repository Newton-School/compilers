# compilers — Newton School fork

Base Docker image that ships every language toolchain Judge0 executes
student submissions in. Built downstream as `judge0/compilers:1.4.0` (the
upstream artifact) and `newtonschool/judge0-newton-compiler:0.33` (Newton's
modernised + trimmed artifact, which is what `Newton-School/judge0` actually
consumes).

## What's in the image

GCC 9.5 (C/C++), Java 21, Kotlin 2.3.21, Scala 3.8.3, Python 3.13 + 3.12 ML,
Ruby 3.3.6, Node 22 + TypeScript, Go 1.23, Rust 1.83, R 4.5, Bash 5.2,
NASM (amd64-only), FreeBASIC (amd64-only), SQLite, MARS, nand2tetris,
Icarus Verilog 13.0, Mono 6.12 + .NET 7 + .NET 8 (C# lanes), isolate v2.

Plain text (judge0 id 43) needs no toolchain — handled judge0-side.

**Operational notes:**

- **GCC pinned to 9.5.0 only.** GCC 14 was archived in 0.28; ids 3003/3004 in
  judge0 (`Newton-School/judge0`) live in `archived.rb`. The lenient flags on
  ids 50/54 keep legacy student code compiling on the 9.x diagnostic surface.
- **`gfortran` is not in the image.** GCC builds use `--enable-languages=c,c++`
  only, since Fortran (id 59) was archived in 0.27.
- **nand2tetris-web-ide is pinned to a commit SHA** (`NAND2TETRIS_REF`). The
  repo is Newton-owned and public — any accidental push to its default branch
  would otherwise change image behaviour on the next rebuild. Bumping the pin
  is intentional.
- **Kotlin/Scala live in Tier 4** alongside the JDK. `kotlinc` / `kotlin` /
  `scalac` / `scala` are symlinked into `/usr/local/bin`; versioned install
  dirs at `/usr/local/kotlin-2.3.21` and `/usr/local/scala-3.8.3`.
- **Icarus Verilog lives in Tier 11** (added in 0.29, below the pip ML
  layer) — pure C/C++ source build, no per-arch branching. Backs judge0
  language id 3005. Build deps (autoconf/gperf/flex/bison) are installed
  and purged in the same RUN.
- **C# / .NET lives in Tier 12** (added in 0.30, retuned in 0.32, W^X
  workaround added in 0.33). Mono 6.12.0.122 is a from-source build into
  `/usr/local/mono-<ver>` (legacy `.NET Framework 4.7`-era compat); .NET
  7.0.400 (hiring courses target net7.0) and .NET 8.0.302 SDKs install
  side-by-side into `/usr/local/dotnet-sdk` via the official
  `dotnet-install.sh`, with `DOTNET_ROOT` set and
  `DOTNET_MULTILEVEL_LOOKUP=0`. Telemetry/first-run noise suppressed via
  `DOTNET_NOLOGO=1`, `DOTNET_CLI_TELEMETRY_OPTOUT=1`,
  `DOTNET_SKIP_FIRST_TIME_EXPERIENCE=1` in the same ENV. Per-test SDK
  selection is via a `global.json` (`rollForward: disable`). `mono`,
  `mcs`, and `dotnet` are symlinked into `/usr/local/bin`.
- **`DOTNET_EnableWriteXorExecute=0` is intentional** (added in 0.33).
  .NET 6+ on Linux uses a W^X double-mapped JIT code allocator that
  calls `memfd_create` + `ftruncate` to a reservation derived from host
  RAM. On large-RAM hosts (EC2 prod) the reservation exceeds isolate's
  `RLIMIT_FSIZE` (capped at ~1-2 GiB by Judge0's `MAX_MAX_FILE_SIZE`),
  causing `ftruncate` to fail with EFBIG → SIGXFSZ → dotnet dies during
  runtime init, before any user-visible output. Diagnostic fingerprint:
  `exit 153` (= 128 + 25) with zero stdout/stderr on compile. Disabling
  W^X uses a single RWX mapping instead — fine inside isolate which
  already provides the security boundary. Judge0's `IsolateJob` must
  propagate this env via `-E DOTNET_EnableWriteXorExecute` (isolate
  strips env by default).

See `git log` for the 0.26 → 0.30 trim/revive chronology if you need to know
when a particular toolchain came or went.

## Repo layout you will care about

```
.
├── Dockerfile                          # upstream judge0/compilers:1.4.0 layout, frozen
├── NewtonDockerFiles/
│   ├── NewtonDockerfile-v1             # 0.25 production: layered on judge0/compilers:1.4.0
│   └── NewtonDockerfile-v2             # 0.26+ production: standalone on bookworm — USE THIS
├── bin/
│   ├── newton-test                     # 40+ language smoke test (must pass before tagging)
│   ├── run-tests / debug-tests         # upstream tests, mostly unused
├── extra/                              # upstream "extra" flavour, ignored
├── slim/                               # upstream "slim" flavour, ignored
└── tests/<lang>/                       # upstream per-language tests; we don't run these
```

## What's in NewtonDockerfile-v2

- Base: `buildpack-deps:bookworm` (Debian 12). The pre-0.26 image was on
  `judge0/buildpack-deps:buster-2019-12-28` and required a sources.list
  rewrite to `archive.debian.org` to do anything.
- isolate v2 (sandbox) — cgroup v2 capable. judge0's `docker-entrypoint.sh`
  substitutes for `isolate-cg-keeper` (the systemd service that normally
  populates `/run/isolate/cgroup`), so the Rails app runs isolate in
  cgroup-v2 mode — RSS-based memory, `cpu.stat`-based time. Falls back
  silently to rlimit mode on cgroup-v1 hosts.
- One latest-stable per language family. **GCC pinned to 9.5.0 only**
  (modern GCC 14 path dropped in 0.28); ids 48/49/50/52/53/54 still serve
  the GCC 9.x ABI/diagnostic surface. ids 3003/3004 (GCC 14 C/C++) are
  archived on the judge0 side.
- Drops: Python 2.7, VB.Net. (Mono was dropped in 0.27 and re-added in
  0.30 for hiring-course C# coverage. 0.32 swapped the side-by-side SDKs
  from .NET 8 + .NET 10 to .NET 7 + .NET 8 — net7.0 is the hiring-course
  target. .NET 7 is out of Microsoft support since May 2024; pin is
  intentional and locked via `global.json` rollForward: disable.)
- Multi-arch (amd64 + arm64) via `ARG TARGETARCH` branching. arm64 falls
  back to bookworm `apt sbcl` and `apt fpc` (no upstream binaries) and
  uses LDC instead of DMD.

### Layer ordering rule

Tier 0 (foundation, frozen) at the top, tier 10 (Python ML pip install,
volatile) at the bottom. Editing the Python ML package list invalidates
ONE layer; bumping a language version cascades to everything below it.
**Append new languages at the bottom; don't insert in the middle.**

**Per-language ENV pins, not a mega-ENV.** Each language's version ENV
sits directly above its install RUN. Bumping one language invalidates
that ENV + RUN + everything below — never above. Don't consolidate pins
into a shared block at the top: a single bump there rebuilds the whole
image (45+ min lost to this on the JDK pin in 0.26 — don't recreate it).

## Build commands

```bash
# arm64 native (Mac dev) — ~2-2.5 hrs from scratch
docker buildx build --platform linux/arm64 \
  -f NewtonDockerFiles/NewtonDockerfile-v2 \
  -t newtonschool/judge0-newton-compiler:0.33-arm64 \
  --load .

# amd64 (EC2 / prod) — ~45-75 min on a c6i.4xlarge
docker buildx build --platform linux/amd64 \
  -f NewtonDockerFiles/NewtonDockerfile-v2 \
  -t newtonschool/judge0-newton-compiler:0.33 \
  --load .
```

Disk: needs ≥100 GB host volume; buildkit cache balloons to 30-50 GB
during a fresh build. **Run inside `tmux`/`screen`** on EC2 to survive SSH
drops.

## Smoke test

```bash
docker run --rm \
  -v "$PWD/bin:/work/bin:ro" \
  newtonschool/judge0-newton-compiler:0.33-arm64 \
  bash /work/bin/newton-test
```

Expected (post 0.33: 3 C# lanes — Mono legacy, .NET 7, .NET 8):
22 PASS / 0 FAIL / 2 SKIP on arm64 (NASM and FreeBASIC are amd64-only
upstream and skip on arm64). 24 PASS / 0 FAIL / 0 SKIP on amd64.

If this isn't green, DON'T tag/push.

## Common pitfalls

1. **`/bin/sh` is dash, not bash.** RUN steps use POSIX shell. Bash-only
   constructs like `${var//pat/repl}` fail with `Bad substitution`. Use
   `tr` or compute the variant ahead of time as a separate `ENV` line.
2. **`cd <dir> && rm -rf <dir> && next-cmd` removes the cwd**, then the
   next command's open-cwd lookup fails ("rb_sysopen", "getcwd() failed").
   Always `cd /` (or `cd ..`) before `rm -rf`. Affects pip, Rscript,
   anything that opens cwd at startup.
3. **GCC 9.5.0, not 9.2.0.** GCC 9.2 was the upstream version but doesn't
   build cleanly against bookworm's modern glibc. 9.5.0 is the last 9.x
   patch and builds fine. Add `--disable-libsanitizer` to the configure
   for the same reason.
4. **GHC arm64 has no deb12 build.** Use `aarch64-deb11-linux` for arm64,
   `x86_64-deb12-linux` for amd64. The deb11 binary needs a libtinfo5
   shim because bookworm only ships libtinfo6.
5. **MARS jar URL uses major_minor only.** The release tag is `v.4.5.1`
   but the asset is `Mars4_5.jar` (no trailing `_1`). Don't try to compute
   the filename from the version string.
6. **Bash mirror at `ftpmirror.gnu.org/bash/` returns 403.** Use
   `ftp.gnu.org/gnu/bash/` directly.
7. **SBCL has no upstream arm64 Linux binary.** arm64 falls back to
   bookworm `apt sbcl` (2.2.x); amd64 keeps the SourceForge binary
   (2.4.10).
8. **DMD has no arm64 Linux binary.** arm64 uses LDC; amd64 uses DMD. The
   arm64 case adds a `linux/bin64/dmd` symlink so consumers can use a
   single path.
9. **nand2tetris-web-ide pins `node: "20.9.0"` exactly** in package.json.
   We're on Node 22 in 0.26+. Use `npm install --force` to bypass the
   engine pin.
10. **Adding to the top ENV block invalidates EVERYTHING below.** During
    initial 0.26 work, adding `JDK_FILE_VERSION` near the JDK pin caused
    a fresh GCC 14 rebuild (45+ min lost). For values needed only by one
    RUN, compute them inline in that RUN.

## Image is published as

- Docker Hub: `newtonschool/judge0-newton-compiler`
- Current tag: **`0.33`** (amd64 produced on EC2). Earlier published: 0.25, 0.26, 0.27, 0.28, 0.29, 0.30, 0.31, 0.32.

## Production deploys

The compilers image is the **base image only** — judge0 builds on top.
Don't deploy this image directly anywhere; deploy `newtonschool/newton-judge0:<tag>`
from `Newton-School/judge0` instead.

## Working notes

- `docker-compose.dev.yml` doesn't live here; it lives in
  `Newton-School/judge0`. This repo is build-only.
- The upstream `judge0/compilers` repo is the original source; rebasing on
  it is generally NOT desired any more (Newton has diverged with the
  v2 single-Dockerfile rewrite). For curiosity, see
  https://github.com/judge0/compilers.
