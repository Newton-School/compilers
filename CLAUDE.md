# compilers — Newton School fork

Base Docker image that ships every language toolchain Judge0 executes
student submissions in. Built downstream as `judge0/compilers:1.4.0` (the
upstream artifact) and `newtonschool/judge0-newton-compiler:0.28+` (Newton's
modernised + trimmed artifact, which is what `Newton-School/judge0` actually
consumes).

## What changed in 0.28 (vs 0.27)

Two-part diff:

1. **Dropped GCC 14.2.0** — only GCC 9.5.0 retained for C/C++. Production
   submissions never adopted the GCC 14 ids; the second toolchain build
   was paying ~45 min of compile time and ~1 GB of image for nothing.
   `GCC_VERSION` env pin and the Tier 1 build are gone; ids 3003/3004 in
   judge0 (`Newton-School/judge0`) move to `archived.rb` in the same
   release.
2. **Re-added Kotlin 2.3.21 and Scala 3.8.3** — both were trimmed in 0.27
   but are needed again for re-introduced course tracks. Sit in Tier 4
   alongside the JDK. `kotlinc`/`kotlin`/`scalac`/`scala` symlinked into
   `/usr/local/bin`; versioned install dirs at `/usr/local/kotlin-2.3.21`
   and `/usr/local/scala-3.8.3`.
3. **Pinned nand2tetris-web-ide to a specific commit SHA**
   (`NAND2TETRIS_REF`). The repo is public and Newton-owned, so any
   accidental push to its default branch would otherwise change image
   behavior on the next rebuild. Bumping the pin is now an intentional
   action, not a side effect.

Plain text (judge0 id 43) needs no toolchain — handled purely on the
judge0 side.

## What changed in 0.27 (vs 0.26)

Aggressive trim driven by prod usage data: the languages that students
weren't actually using were ~6-7 GB of the image. Dropped toolchains:

- **Clang/LLVM** (C/C++/Obj-C apt path) — Clang ids 75/76/79
- **.NET 8 SDK + dotnet-script** — C# (51) and F# (87)
- **Haskell GHC** (~3 GB by itself) — id 61
- **Swift** — id 83
- **Erlang/OTP + Elixir** — ids 58, 57
- **OCaml** — id 65
- **Octave** — id 66
- **Free Pascal (FPC)** — id 67
- **GnuCOBOL** — id 77
- **GNU Prolog** — id 69
- **SBCL** (Common Lisp) — id 55
- **DMD/LDC** (D) — id 56
- **Lua** — id 64
- **PHP** — id 68
- **Kotlin** — id 78
- **Scala 3** — id 81
- **Groovy** — id 88
- **Clojure** — id 86

Also: `gfortran` removed from base apt and `--enable-languages=c,c++` (no
fortran) on both GCC builds, since Fortran (id 59) was archived.

What's still in (after 0.28): GCC 9.5 (C/C++), Java 21, Kotlin 2.3.21,
Scala 3.8.3, Python 3.13 + 3.12 ML, Ruby 3.3.6, Node 22 + TypeScript,
Go 1.23, Rust 1.83, R 4.5, Bash 5.2, NASM, FreeBASIC, SQLite, MARS,
nand2tetris, isolate v2.

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
- isolate v2 (sandbox) — cgroup v2 capable. The judge0 Rails app currently
  runs it in non-cgroup mode (per-process rlimits) because in-container
  cgroup-v2 delegation needs systemd, which the container doesn't run.
- One latest-stable per language family. **GCC pinned to 9.5.0 only**
  (modern GCC 14 path dropped in 0.28); ids 48/49/50/52/53/54 still serve
  the GCC 9.x ABI/diagnostic surface. ids 3003/3004 (GCC 14 C/C++) are
  archived on the judge0 side.
- Drops: Python 2.7, Mono, VB.Net.
- Multi-arch (amd64 + arm64) via `ARG TARGETARCH` branching. arm64 falls
  back to bookworm `apt sbcl` and `apt fpc` (no upstream binaries) and
  uses LDC instead of DMD.

### Layer ordering rule

Tier 0 (foundation, frozen) at the top, tier 10 (Python ML pip install,
volatile) at the bottom. Editing the Python ML package list invalidates
ONE layer; bumping a language version cascades to everything below it.
**Append new languages at the bottom; don't insert in the middle.**

**0.28+ change: per-language version pins.** Each language's version
ENV now sits directly above its install RUN, instead of all pins
sharing one mega-ENV at the top of the file. Bumping one language only
invalidates that ENV layer + the RUN below it + everything further
down. Adding a new language at the bottom invalidates nothing above.
Pre-0.28, the top mega-ENV meant any single bump rebuilt the whole
image (45+ min wasted on the JDK_FILE_VERSION incident in 0.26 was the
proximate cause of this restructure).

## Build commands

```bash
# arm64 native (Mac dev) — ~2-2.5 hrs from scratch
docker buildx build --platform linux/arm64 \
  -f NewtonDockerFiles/NewtonDockerfile-v2 \
  -t newtonschool/judge0-newton-compiler:0.28-arm64 \
  --load .

# amd64 (EC2 / prod) — ~45-75 min on a c6i.4xlarge
docker buildx build --platform linux/amd64 \
  -f NewtonDockerFiles/NewtonDockerfile-v2 \
  -t newtonschool/judge0-newton-compiler:0.28 \
  --load .
```

Disk: needs ≥100 GB host volume; buildkit cache balloons to 30-50 GB
during a fresh build. **Run inside `tmux`/`screen`** on EC2 to survive SSH
drops.

## Smoke test

```bash
docker run --rm \
  -v "$PWD/bin:/work/bin:ro" \
  newtonschool/judge0-newton-compiler:0.28-arm64 \
  bash /work/bin/newton-test
```

Expected (post 0.28: GCC 14 dropped, Kotlin + Scala added → net +1):
18 PASS / 0 FAIL / 2 SKIP on arm64 (NASM and FreeBASIC are amd64-only
upstream and skip on arm64). 20 PASS / 0 FAIL / 0 SKIP on amd64.

If this isn't green, DON'T tag/push.

## Common pitfalls (encountered while building 0.26 — don't repeat)

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
6. **`ftpmirror.gnu.org` returns transient 403s from inside buildkit.**
   Bash hit this in 0.26; GCC 9.5 hit the same in 0.28 (a build that had
   been working previously failed on rebuild because the auto-redirecting
   mirror sent buildkit to a rate-limited downstream). Always use
   `ftp.gnu.org/gnu/<project>/` directly for any GNU source — applies to
   bash, gcc, gnucobol, octave, etc.
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
- Tags currently published: `0.25` (legacy), `0.26` (Phase 2 modernised),
  `0.27` (Phase 3 trimmed), `0.28` (Phase 4 — GCC 14 dropped, Kotlin/Scala
  re-added; current target)
- Phase-4 production tag: `0.28` (amd64) — produced on EC2

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
