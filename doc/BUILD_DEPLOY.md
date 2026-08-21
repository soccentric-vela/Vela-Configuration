# Vela-Configuration Build and Deployment

## Purpose and scope

This leaf is the Rust WebConfig workspace: `webconfigd` serves the on-device
configuration UI and `webconfig-engine` contains backend logic. The root
`Makefile` wraps Cargo, pinned by `rust-toolchain.toml`, and the `meta-webconfig`
layer packages the daemon, static assets, schemas, and board manifest into a
Yocto image.

## 1. Prerequisites

- Rustup and the pinned Rust channel from `rust-toolchain.toml`; network access
  is needed by `make toolchain` unless the toolchain is already installed.
- GNU Make and Cargo. `make toolchain` also installs rustfmt, clippy, and the
  x86_64, aarch64, and armv7 targets.
- Yocto/KAS tooling and the matching Vela-OS build environment for image
  integration; no Node.js build is required for the vanilla frontend.

## 2. Setup

Run `make toolchain` once to install the exact channel, components, and targets.
The Makefile selects `$HOME/.cargo/bin/cargo` when present. For offline source
regeneration, run `make vendor` after dependency changes and commit the
generated `vendor/` tree as required by the repository policy.

## 3. Build and compile

- `make build` performs a workspace debug build.
- `make release` uses the checked-in size-focused profile: `opt-level=z`, LTO,
  one codegen unit, abort-on-panic, and symbol stripping.
- `make cross` builds release artifacts for all three configured targets.
- `make run-mock` runs against a temporary fake sysroot with mock backends and
  no hardware. `make run` uses the real host sysroot and live backends.

## 4. Tests and lint

`make test` runs all workspace targets and integration tests against the fake
sysroot. `make lint` runs clippy with `-D warnings` and `cargo fmt --check`.
`make fmt` formats in place. `make check` is the combined `build`, `test`, and
`lint` gate.

## 5. Packaging and artifacts

`make release-artifacts` invokes `tools/release-artifacts.sh` to package source
and vendor tarballs and print SHA-256 checksums; `make vendor` must have run
first. Yocto artifacts are produced by the `meta-webconfig` recipes, including
the `webconfigd`, static, schemas, and manifest packages.

## 6. Deployment, install, and run

For Yocto deployment, add `webconfigd webconfig-static webconfig-schemas
webconfig-manifest` to `IMAGE_INSTALL:append` and map the BSP machine to the
manifest in `meta-webconfig/recipes-core/webconfig/webconfig-manifest.bbappend`.
For local development use `make run-mock`; use `make run` only when live
`/proc`, `/sys`, systemd, RAUC, zbus, and the real data directory are intended.
The default development port is `8443`; override it with `WEBCONFIG_PORT=PORT`.

## 7. Cleanup

`make clean` removes Cargo `target/`. Remove the local `data/` directory
manually when a real-host run must return to first-boot state. `make stop`
terminates the process listening on `WEBCONFIG_PORT`.

## 8. Hardware, cloud, and manual verification caveats

Mock tests do not validate TLS certificates, permissions, systemd or RAUC
behavior, zbus services, network commit/confirm rollback, or the selected board
manifest. `make run` can affect the host because it uses real backends. Yocto
integration, credentials, certificate provisioning, firewall exposure, and
boot-time behavior require a target image and device-level verification.