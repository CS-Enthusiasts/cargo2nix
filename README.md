# cargo2nix

nitul1

[![Build Status][build-badge]][build-url]

[build-badge]: https://circleci.com/gh/tenx-tech/cargo2nix.svg?style=shield
[build-url]: https://circleci.com/gh/tenx-tech/cargo2nix

[Nixify](https://nixos.org/nix) your Rust projects today with `cargo2nix`,
bringing you reproducible builds and better caching.

This repository hosts two components:

- A [Nixpkgs](https://github.com/NixOS/nixpkgs) overlay, located at the `/overlay`
  directory, providing utilities to build and test your Cargo workspace.
  
- A utility written in Rust to generate version pins of crate dependencies.
  
Together, these components will take an existing `Cargo.lock` and delegate the
process of fetching and compiling your dependencies (generated by Cargo) using
the deterministic Nix package manager.

## Install

This project assumes that the [Nix package manager](https://nixos.org/nix) is
already installed on your machine. Run the command below to install `cargo2nix`:

```bash
nix-env -iA package -f https://github.com/tenx-tech/cargo2nix/tarball/master
```

## How to use this for your Rust projects
nitul2


### As a build system

The basic process of converting an existing Cargo project to `cargo2nix` boils
down to the following steps:

1. Generate a `Cargo.nix` file by running `cargo2nix -f` at the root of your
   Cargo workspace.
2. Create a `default.nix` file which imports Nixpkgs with the [cargo2nix] and
   [nixpkgs-mozilla] overlays and builds your project using the `Cargo.nix` file
   from earlier.
3. Run `nix-build` to compile and/or test your project.

[nixpkgs-mozilla]: https://github.com/mozilla/nixpkgs-mozilla#rust-overlay
[cargo2nix]: ./overlay

Check out our series of [example projects](./examples) which showcase how to use
`cargo2nix` in detail.

### Optional declarative development shell

You can optionally use any of these crate derivations as your `nix-shell`
development shell. The advantage of this shell is that in this environment users
can develop their crates and be sure that their crates builds in the same way
that `cargo2nix` overlay will build them.

To do this, run `nix-shell -A 'rustPkgs.<registry>.<crate>."x.y.z"' default.nix`.
For instance, the following command being invoked in this repository root drops
you into such a development shell.

nitul3

```bash
# When a crate is not associated with any registry, such as when building locally,
# the registry is "unknown" as shown below.
nix-shell -A 'rustPkgs.unknown.cargo2nix."0.8.3"' default.nix
```

You will need to bootstrap some environment in this declarative development
shell first.

```bash
runHook configureCargo  # This overrides your .cargo folder, e.g. for setting cross-compilers
runHook setBuildEnv     # This sets up linker flags for the `rustc` invocations
```

You will need to override your `Cargo.toml` and `Cargo.lock` in this shell,
so make sure that you have them backed up beforehand.

```bash
runHook overrideCargoManifest
```

Now you can use your favorite editor in this environment. To run the build
command used by `cargo2nix`, use

```bash
runHook runCargo
```

## Common issues

1. Many `crates.io` public crates may not build using the current Rust compiler,
   unless a lint cap is put on these crates. For instance, `cargo2nix` caps all
   lints to `warn` by default.

2. Nix 2.1.3 ships with a broken `builtins.fromTOML` function which is unable to
   parse lines of TOML that look like this:

   ```toml
   [target.'cfg(target_os = "linux")'.dependencies.rscam]
   ```

   If Nix fails to parse your project's `Cargo.toml` manifest with an error
   similar to the one below, please upgrade to a newer version of Nix. Versions
   2.3.1 and newer are not affected by this bug. If upgrading is not an option,
   removing the inner whitespace from the problematic keys should work around
   this issue.

   ```text
   error: while parsing a TOML string at /nix/store/.../overlay/mkcrate.nix:31:14: Bare key 'cfg(target_os = "linux")' cannot contain whitespace at line 45
   ```

3. Git dependencies and crates from alternative Cargo registries rely on
   `builtins.fetchGit` to support fetching from private Git repositories. This
   means that such dependencies cannot be evaluated with `restrict-eval`
   applied.

   Also, if your Git dependency is tied to a Git branch, e.g. `master`, and you
   would like to force it to update on upstream changes, you should append
   `--option tarball-ttl 0` to your `nix-build` command.

## Design

This Nixpkgs overlay builds your Rust crates and binaries by first pulling the
dependencies apart, building them individually as separate Nix derivations and
linking them together. This is achieved by passing custom linker flags to the
`cargo` invocations and the underlying `rustc` and `rustdoc` invocations.

In addition, this overlay takes cross-compilation into account and build the
crates onto the correct host platform configurations with the correct
platform-dependent feature flags specified in the Cargo manifests and build-time
dependencies.

## Credits

The design for the Nix overlay is inspired by the excellent work done by James
Kay, which is described [here][blog-1] and [here][blog-2]. His source is
available [here][mkRustCrate]. This work would have been impossible without
these fantastic write-ups. Special thanks to James Kay!

[blog-1]: https://www.hadean.com/blog/managing-rust-dependencies-with-nix-part-i
[blog-2]: https://www.hadean.com/blog/managing-rust-dependencies-with-nix-part-ii
[mkRustCrate]: https://github.com/Twey/mkRustCrate

## License

`cargo2nix` is free and open source software distributed under the terms of the
[MIT License](./LICENSE).
