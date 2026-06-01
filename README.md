# zensical/zensical#726 — reproduction

Minimal reproduction for [zensical/zensical#726](https://github.com/zensical/zensical/issues/726),
fixed in [zensical/zensical#727](https://github.com/zensical/zensical/pull/727).

The [CI action](../../actions) runs `zensical build` and asserts `site/index.html` is generated — it fails on the buggy version.

## Reproduce locally

```sh
uvx zensical==0.0.43 build
```
