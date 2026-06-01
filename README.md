# zensical/zensical#726 — reproduction

[![Reproduce](https://github.com/jstriebel/zensical-issue-726/actions/workflows/repro.yml/badge.svg)](https://github.com/jstriebel/zensical-issue-726/actions/workflows/repro.yml)

Minimal reproduction for [zensical/zensical#726](https://github.com/zensical/zensical/issues/726),
fixed in [zensical/zensical#727](https://github.com/zensical/zensical/pull/727).

## Root cause

`Watcher::new` registers the docs directory with the file agent last; the
scheduler can drain and exit before any docs files arrive, so no HTML is
rendered.  See [#726](https://github.com/zensical/zensical/issues/726) for details.

## Reproduce locally

This is a timing race — running `uvx zensical==0.0.43 build` fails consistently
on the reporter's machine.  `race.patch` surfaces it on machines where the race
doesn't trigger naturally (fewer cores, different architecture) by inserting a
`thread::sleep(2s)` in `Watcher::new` right before `agent.watch(docs_dir)`,
giving the scheduler time to drain and exit before docs files are registered.

```sh
uvx zensical==0.0.43 build
```

To reproduce with the patch:

```sh
# Note: python/zensical/templates/ is not committed to git (it is a build
# artifact).  Install the PyPI wheel for its theme assets, then swap in the
# patched .so built from source.
git clone https://github.com/zensical/zensical --branch v0.0.43 zensical-src
patch -p1 -d zensical-src < race.patch
cd zensical-src && maturin build --release --out dist && cd ..
python3 -m venv zenv && zenv/bin/pip install "zensical==0.0.43"
so=$(unzip -l zensical-src/dist/zensical-*.whl | tail -n +4 | head -n -2 \
  | awk '{print $NF}' | grep '\.so$')
site_dir=$(zenv/bin/python3 -c "import site; print(site.getsitepackages()[0])")
unzip -p zensical-src/dist/zensical-*.whl "$so" > "$site_dir/$so"
zenv/bin/zensical build
# RuntimeError: Watcher disconnected  (or silent empty site/) — bug reproduced
```
