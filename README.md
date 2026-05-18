# tokenizers — Windows ARM64

Native **Windows ARM64** builds of [`Anush008/tokenizers`](https://github.com/Anush008/tokenizers)
(the HuggingFace `tokenizers` napi-rs binding) published as the
`@anush008/tokenizers-win32-arm64-msvc` platform package — the one the upstream
author does not ship, which otherwise blocks native ARM64 Electron/Node apps
that depend on `@anush008/tokenizers` (e.g. via `fastembed`).

Consumed by [`superset-windows-arm64`](https://github.com/khairm/superset-windows-arm64).

## How it works

1. A GitHub Actions workflow runs nightly (also `workflow_dispatch`).
2. **Change-gated:** it determines the target version — by default the
   `@anush008/tokenizers` version `fastembed` pins (the consumer that matters;
   e.g. `fastembed@2.1.0` → `^0.0.0` → `0.0.0`), or an explicit
   `workflow_dispatch` input — and **only builds if that version is not already
   released here** — no rebuilds when nothing changed.
3. On a native `windows-11-arm` runner it builds upstream `tokenizers`
   **unmodified** with its own pinned `@napi-rs/cli` for the native host triple
   `aarch64-pc-windows-msvc` (Rust stable + VS 2022) — see [`PATCHES.md`](PATCHES.md).
4. It verifies the `.node` is ARM64 and loads under Node, then publishes a
   **per-version Release** (tag = the tokenizers version) with
   `tokenizers-win32-arm64-msvc.tar.gz` + a `.sha256`.

Because napi-rs addons are **N-API** (ABI-stable across Node/Electron
versions), a single ARM64 build works for any Electron — there are no per-ABI
variants to track (unlike V8-ABI native modules).

## Consuming it

Download the Release tagged with the version you need:

```bash
gh release download <version> --repo khairm/tokenizers-windows-arm64 \
  -p 'tokenizers-win32-arm64-msvc.tar.gz' -p 'tokenizers-win32-arm64-msvc.tar.gz.sha256'
sha256sum -c tokenizers-win32-arm64-msvc.tar.gz.sha256
mkdir -p out && tar -xzf tokenizers-win32-arm64-msvc.tar.gz -C out   # -> out/tokenizers.win32-arm64-msvc.node + out/package.json
```

Drop `out/` in as `node_modules/@anush008/tokenizers-win32-arm64-msvc` so
`@anush008/tokenizers`'s platform resolution finds it.

If a version you need has not been built yet, trigger it:

```bash
gh workflow run "Nightly tokenizers Windows ARM64 Build" \
  --repo khairm/tokenizers-windows-arm64 -f tokenizers_version=<version>
```

## Requirements / CI

Runs on the free `windows-11-arm` GitHub-hosted runner (public repos). No
secrets required — the build is deterministic (no AI patch step; upstream is
built unmodified with its own pinned toolchain).

## Attribution

Upstream: [Anush008/tokenizers](https://github.com/Anush008/tokenizers) (MIT) —
multi-arch napi-rs bindings for HuggingFace `tokenizers`. This repo only
compiles the missing Windows ARM64 target; not affiliated with the upstream
author.
