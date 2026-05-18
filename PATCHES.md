# Windows ARM64 build notes — tokenizers

> **No upstream source patch is required.** Building
> [`Anush008/tokenizers`](https://github.com/Anush008/tokenizers) for Windows
> ARM64 needs **zero modifications** to upstream source. The nightly builds
> upstream unmodified with its own pinned `@napi-rs/cli` and assembles the
> platform package deterministically. This file documents the recipe (the
> "patch" is purely a build/packaging procedure, applied by the workflow).

## The recipe (per `.github/workflows/nightly-build.yml`)

1. **Runner:** `windows-11-arm` (native ARM64; host triple
   `aarch64-pc-windows-msvc`). Has Rust-capable VS 2022 + libclang.
2. **Toolchain:** `rustup` **stable** (the repo has no `rust-toolchain.toml`
   pin); add target `aarch64-pc-windows-msvc` (== host). Set `LIBCLANG_PATH`
   defensively (only matters if a transitive HF-tokenizers dep uses bindgen).
3. **Install:** upstream uses **yarn berry** (`packageManager: yarn@3.x`,
   `.yarnrc.yml`). `corepack enable` then `yarn install --immutable` (falls
   back to a plain `yarn install`).
4. **Build:** `yarn build` (= `napi build --platform --release`, using the
   repo's own pinned `@napi-rs/cli` ^2.x).
   - **Do NOT pass `--target`.** The runner host *is*
     `aarch64-pc-windows-msvc`; a native host build with `--platform` emits
     `tokenizers.win32-arm64-msvc.node` at project root. Passing `--target` on
     a native host can trigger napi-rs cross-compile detection that inserts a
     linker prefix and breaks the link.
5. **Package:** copy `tokenizers.win32-arm64-msvc.node`; emit a `package.json`
   mirroring the published `@anush008/tokenizers-win32-x64-msvc` but with
   `cpu:["arm64"]` and the arm64 `main`. Prove it loads under Node
   (`require()` → `ADDON LOADED OK`), then tar + sha256 and publish a
   per-version Release (tag = the tokenizers version).

## Why N-API means one build covers every Electron

`@anush008/tokenizers` is a **napi-rs** addon → **N-API** (Node-API), which is
ABI-stable across Node/Electron major versions. So unlike V8/NAN-ABI modules
(e.g. `better-sqlite3`, which needs a distinct `electron-v<ABI>` prebuilt per
Electron major), a **single** `win32-arm64-msvc` build loads under any
Electron. That is also why upstream ships only per-*platform* packages with no
`electron-v*` / `node-v*` ABI variants — and why proving the load under the
runner's Node 20 is sufficient proof for the Electron 40 consumer.

## Why this package is needed

`@anush008/tokenizers` `optionalDependencies` cover `win32-x64-msvc`, the
linux x64/arm64 (gnu/musl) targets and `darwin-universal`, but **no
`@anush008/tokenizers-win32-arm64-msvc` at any version**. Its `index.js`
resolves `@anush008/tokenizers-${platform}-${arch}-msvc` with no native
fallback, so native ARM64 Electron/Node apps depending on it (e.g. via
`fastembed` for embeddings) crash on Windows ARM64. This repo fills exactly
that gap.

## If upstream changes break the recipe

`Anush008/tokenizers` is built unmodified, so upstream churn rarely matters. If
a future version fails to build for `aarch64-pc-windows-msvc`, add the minimal
build fix here and document it in this file (keeping the build deterministic).
