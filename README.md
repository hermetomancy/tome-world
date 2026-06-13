# Grimoire World Tome

`world` is the userland tome: software built *on* the core toolchain. Core stays minimal —
exactly what bootstraps grimoire and its managed build environment — and everything else
lives here. Add both:

```sh
grm tome add https://github.com/grimoire-of-glass/tome-core --ref main
grm tome add https://github.com/grimoire-of-glass/tome-world --ref main
```

## Packages

- `zoxide`, `starship`, `atuin`, `rustormy` — Rust CLIs (build dep: `rust`); the shell
  hooks print their `eval "$(... init zsh)"` lines as post-install notes.
- `gnupg` — with its support chain `libgpg-error`, `libgcrypt`, `libassuan`, `libksba`,
  `npth`, and `pinentry` (curses/tty flavors; gpg-agent's pinentry path is baked to the
  managed pinentry's store prefix).
- `zlib`, `bzip2`, `ncurses` — managed C libraries. Only libc/the platform SDK is an
  assumable host floor, so anything else a build links must come from the store. All
  three are static-only; zlib and bzip2 are therefore build deps for their consumers,
  while ncurses is also a *runtime* dep of anything linking it (the compiled-in default
  terminfo path is a store path GC must keep alive).
- `zstd` — the Zstandard compressor (the `zstd` CLI and libzstd), built without the
  gzip/xz/lz4 passthrough backends so it links no host compression libraries; the native
  `.zst` format is unaffected.
- `m4`, `autoconf`, `automake`, `gettext` — the autotools chain, for runes built from git tags that
  ship no generated `configure` (perl comes from the host ambient floor, the documented
  stage-2 debt).
- `git` — built on the managed transport stack rather than the host's: `curl` (HTTPS,
  linking core's `openssl` + static zlib) and `ca-certificates` (the Mozilla CA bundle,
  baked as curl's default CA path so HTTPS verifies without the macOS keychain). git links
  libcurl for transport and the static zlib, takes iconv from libc/the SDK, and turns off
  everything else (`NO_OPENSSL`/`NO_EXPAT`/`NO_GETTEXT`/`NO_TCLTK`/`NO_PERL`/`NO_PYTHON`);
  man pages come from the upstream prebuilt tarball. `curl` is a build + runtime dep (shared
  libcurl, baked store paths, pulling core's openssl in turn); `ca-certificates` is a
  fixed-output data package. (`openssl` lives in core — rust links it too.)
- `openssh` — the SSH client/key tools, for git's ssh transport (`git clone git@…`). Built on
  core's `openssl` (libcrypto, shared → runtime dep) and static `zlib` (build dep), with
  PAM/libedit/Kerberos/FIDO all off. It is a standalone package, not a git dep: git execs
  whatever `ssh` is on PATH, so installing `openssh` into the same generation is enough.
- `hello` — GNU Hello, the package-authoring exemplar.

Cross-tome deps resolve like any other: build deps reference core's `rust`, `gmake`,
`gsed`, `openssl`, and `build-env`. Conventions (implementation-named utilities, capability bins,
split packages) are documented in
[rune-authoring.md](https://github.com/grimoire-of-glass/grimoire/blob/main/docs/rune-authoring.md).

## Building

```sh
grm tome build --all --path .
```

`dist/` is git-ignored; publish its contents to the host `tome.rn` advertises and sign the
index (`minisign -S -m dist/index.nuon`).
