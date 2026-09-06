# Maintained Fork

This is a maintained public fork of **[stremio-native/stream-server](https://github.com/stremio-native/stream-server)**.

Upstream: https://github.com/stremio-native/stream-server
Fork: this repository (`CybeRxNinja/stream-server`)

Goals of this fork:

- Keep `Cargo.lock` / Rust dependencies updated (`cargo update`).
- Fix backend-specific build breakage (e.g. `librqbit`-only build).
- Ship **static** Linux binaries + a **pure-Rust librqbit** binary so users never hit
  `libtorrent-rasterbar.so.2.0` runtime errors.
- Provide one-click GitHub Releases with downloadable install files
  (portable binary, `.deb`, `AppImage`, Arch `.pkg.tar.zst`, Windows `.exe`/`.msi`).

## Why `libtorrent-rasterbar.so.2.0: cannot open shared object file`?

That error means your `stream-server` binary was **dynamically linked** against
the distro's shared libtorrent 2.0 (`libtorrent-rasterbar.so.2.0`) and at runtime
the loader can't find that exact SONAME. Typical causes:

1. You built with `cargo build --release --features libtorrent` **without**
   `LIBTORRENT_STATIC=1`, so cargo linked the system shared lib
   (e.g. Ubuntu's `libtorrent-rasterbar-dev` 2.0.x or an older Arch package).
2. You later upgraded the OS / libtorrent to 2.1.x (which provides
   `libtorrent-rasterbar.so.2.1`, not `.so.2.0`), breaking the old binary's ABI.
3. You copied a dynamically-linked binary to another machine without installing
   `libtorrent-rasterbar` there.
4. You downloaded a portable binary that wasn't actually static.

Check with:

```bash
ldd ./stream-server | grep -i torrent
# bad (dynamic):  libtorrent-rasterbar.so.2.0 => not found
# good (static or librqbit):  no output
```

## Fixes (pick one)

### Option A — download a fixed release from this fork (recommended)

- **No libtorrent needed at all:** download `stream-server-linux-amd64-librqbit`
  from the latest GitHub Release. Pure Rust, zero native torrent dep.
  ```bash
  chmod +x stream-server-linux-amd64-librqbit
  ./stream-server-linux-amd64-librqbit
  ```
- **Full features (libtorrent, but static):** download `stream-server-linux-amd64`,
  `stream-server-linux-amd64.deb`, `stream-server-linux-amd64.AppImage`, or
  `stream-server-arch-x86_64.pkg.tar.zst`. These are built in CI with
  `LIBTORRENT_STATIC=1` + libtorrent 2.1.1 from source (`BUILD_SHARED_LIBS=OFF`),
  so `ldd` shows no `libtorrent-rasterbar.so` dependency.

Releases page: `https://github.com/<this-fork>/releases/latest`
(`<this-fork>` = the fork owner/repo you are reading now).

### Option B — install distro libtorrent and keep the old binary working

Arch:

```bash
sudo pacman -S libtorrent-rasterbar boost-libs openssl
# If your binary wants .so.2.0 but Arch now ships 2.1.x, prefer Option A
# (static/librqbit) instead of downgrading, or rebuild from source:
sudo pacman -S base-devel cmake rust boost pkg-config openssl clang gtk3 libayatana-appindicator
LIBTORRENT_STATIC=1 PKG_CONFIG_PATH=/usr/local/lib/pkgconfig cargo build --release --features libtorrent --no-default-features
```

Ubuntu/Debian:

```bash
sudo apt update
sudo apt install libtorrent-rasterbar-dev libboost-all-dev
# then re-run, or better: rebuild static via:
bash .github/scripts/install-libtorrent.sh
LIBTORRENT_STATIC=1 PKG_CONFIG_PATH=/usr/local/lib/pkgconfig cargo build --release --features libtorrent --no-default-features
```

### Option C — build the pure-Rust backend yourself (no C++)

```bash
cargo build --release -p server --no-default-features --features librqbit
./target/release/server
```

## How to cut a release from this fork

### Quick release (fast, Linux librqbit only, ~10 min)

Best when you just need a working Linux binary *now*.

- Via UI: **Actions → “Fork Quick Release (librqbit, no libtorrent)” → Run workflow →**
  enter `version` e.g. `v0.1.8-fork.1` → Run.
- Via tag:
  ```bash
  git tag v0.1.8-fork.1 && git push origin v0.1.8-fork.1
  # or: git tag fork-v1 && git push origin fork-v1
  ```

Result: a GitHub Release with `stream-server-linux-amd64-librqbit` + `SHA256SUMS.txt`.

### Full release (all platforms, ~40-60 min)

Builds Windows `.exe`/`.msi`, Linux portable/`.deb`/`AppImage` (both libtorrent-static
*and* librqbit), and Arch `.pkg.tar.zst`.

- Via UI: **Actions → “Release Build” → Run workflow →** optionally enter `version`
  e.g. `v0.1.9-fork.1` → Run.
- Via tag:
  ```bash
  git tag v0.1.9-fork.1 && git push origin v0.1.9-fork.1
  ```

Then download from **Releases** (or from the run's Artifacts while it builds).

## Maintenance notes (for fork owner)

- Sync upstream: `git fetch upstream && git merge upstream/master` (resolve, test, push).
- Update Rust deps: `cargo update` (this fork already advanced ~246 crates, e.g.
  `tokio 1.52→1.53`, `cxx 1.0.194→1.0.200`, `quick-xml 0.39→0.41`, etc.).
- Fixed in this fork vs upstream `master` (`f585ab6`):
  - `enginefs/src/lib.rs`: missing `EngineCacheConfig` import broke
    `--no-default-features --features librqbit` builds (CI only tested libtorrent).
  - `scripts/generate_release_notes.py`: added librqbit asset labels.
  - `.github/workflows/release.yml`: manual `version` input, tag resolution,
    dual libtorrent-static + librqbit Linux builds with `ldd` guards,
    dynamic Arch `pkgver`/`url`, release includes librqbit binaries.
  - Added `.github/workflows/fork-quick-release.yml` for fast librqbit-only releases.
- libtorrent stays pinned at **2.1.1** (matches Arch `extra` 2.1.1 and
  `vcpkg.json` baseline `84bab45d...`). Bump only together in
  `vcpkg.json`, `.github/scripts/install-libtorrent.sh`, and `bindings/libtorrent-sys/build.rs`
  (`atleast_version`).
