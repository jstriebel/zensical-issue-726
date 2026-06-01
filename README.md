# zensical/zensical#726 — reproduction

Minimal reproduction for [zensical/zensical#726](https://github.com/zensical/zensical/issues/726),
fixed in [zensical/zensical#727](https://github.com/zensical/zensical/pull/727).

The [CI action](../../actions) builds a patched zensical 0.0.43, runs it against
this project, and asserts that `site/index.html` is generated — the assertion
fails because the bug causes an empty `site/`.

## Root cause

`Watcher::new` in `crates/zensical/src/watcher.rs` registers the docs directory
with the file agent **last**, after config and theme directories.  The agent
scans directories asynchronously; by the time the docs directory is scanned the
scheduler has already processed the config file, gone empty, and exited — so no
markdown files are ever inserted, the barrier never fires, and no HTML is
rendered.

Fix: watch docs **first** so markdown files enter the scheduler before
config/theme files, keeping the build loop alive long enough for the barrier to
collect all pages.

## How the reproduction works

`race.patch` inserts a `thread::sleep(2s)` in 0.0.43's `Watcher::new`, right
before `agent.watch(docs_dir)`.  The sleep gives the scheduler time to drain the
config and theme events and exit before docs files are registered — making the
race that occurs naturally on fast multi-core machines deterministic on any
runner.

## Reproduce locally

```sh
uvx zensical==0.0.43 build
```

On a fast (≥8-core) machine this fails reliably.  On a slower machine apply
the patch manually:

```sh
git clone https://github.com/zensical/zensical --branch v0.0.43 zensical-src
patch -p1 -d zensical-src < race.patch
cargo build --release --manifest-path zensical-src/Cargo.toml
./zensical-src/target/release/zensical build
# site/ is empty — bug reproduced
```
