# SoCcentric WebConfig

An **on-device, browser-based configuration web UI** for the Vela
Yocto/embedded-Linux platform (Qualcomm Snapdragon QCS6490). One small Rust daemon serves a vanilla-JS
frontend over TLS; board-specific data lives in a single board-manifest file
(`board-manifests/vela.json`).

> The full design specification lives in [`docs/webconfig-spec.md`](docs/webconfig-spec.md).

## What it is

A field engineer or operator points a browser at the device
(`https://<device>:8443`) and configures it: system identity, networking,
services, updates, monitoring, logs, alerts, security, and I/O — all behind
authenticated, role-based access with a commit/confirm rollback safety net for
networking changes.

- **One daemon**, `current_thread` tokio runtime, rustls (no OpenSSL), no
  database (JSON + ring files). Target: <15 MB RAM, 3-6 MB stripped binary.
- **Vanilla JS frontend**, no build step, no node at build time, <500 KB total.
- **Schema-driven rendering**: adding a setting is a schema file — zero UI code.
- **Flagship networking rollback**: network changes are provisional; reconnect
  and confirm within a window or they auto-roll-back.
- **Real + mock system interactions**: RAUC/zbus talk to real backends on-device
  and to mocks on the fake-sysroot test harness (no hardware to test).

## Repository layout

```
Cargo.toml                  # workspace, pinned deps, exact release profile
crates/
  webconfig-engine/         # config store, appliers, schema, manifest, alert/monitoring
  webconfigd/               # axum HTTP daemon (TLS, auth, CSRF, 9 section APIs, SSE)
frontend/                   # vanilla JS: index.html, styles.css, schema-form.js, pages.js, app.js
schemas/                    # *.schema.json — schema-driven rendering + validation
board-manifests/            # vela.json (board-specific data) + generic.yaml fallback
meta-webconfig/             # Yocto layer: daemon, static assets, schemas, manifest, hardened unit
docs/                       # spec, API reference, schema format, board-manifest spec, planning/audits
Makefile                    # build | release | test | lint | fmt | vendor | cross | run | clean
```

## Build & test

The repo-root Makefile is the single entry point (per SoCcentric conventions):

```bash
make build    # debug build (host target)
make release  # exact spec release profile (opt-level=z, lto, panic=abort, stripped)
make test     # engine + HTTP integration tests (fake sysroot, no hardware)
make lint     # clippy -D warnings + rustfmt --check
make vendor   # regenerate the offline crate tarball for reproducible builds
make cross    # build all three cross-targets: x86_64, aarch64, armv7
make run      # run the daemon against a tempdir fake sysroot (dev)
```

Validation gate (used by the git workflows and CI): `make build && make test && make lint`.

## Cross-targets

`x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`,
`armv7-unknown-linux-gnueabihf`. Vendored crates + `CARGO_NET_OFFLINE=1` make
builds fully offline and reproducible.

## Deploying (Yocto)

Add the layer and packages to your image:

```bitbake
IMAGE_INSTALL:append = " webconfigd webconfig-static webconfig-schemas webconfig-manifest"
```

Map your BSP `MACHINE` to its board manifest in
`meta-webconfig/recipes-core/webconfig/webconfig-manifest.bbappend`.
See [`meta-webconfig/README.md`](meta-webconfig/README.md).

## Documentation

- [Design specification](docs/webconfig-spec.md)
- [REST API reference](docs/api-reference.md)
- [Schema format](docs/schema-format.md)
- [Board-manifest format](docs/board-manifest.md)
- [Production-readiness audit](docs/planning/MasterPlan.md) and its [acceptance checklist](docs/planning/Checkpoint.md)

## Project Summary

The configuration service is designed as a small device-local control plane,
not as a replacement for the fleet API. The Rust daemon owns HTTP concerns,
authentication, request validation, event delivery, and backend coordination;
the engine owns configuration state, schemas, manifests, and application
logic. The frontend is vanilla JavaScript and the `schemas/` directory drives
form rendering and validation. `meta-webconfig/` packages the daemon, assets,
schemas, manifest, and hardened service unit for Yocto deployment. The Makefile
also documents release, lint, formatting, vendoring, cross-target, and fake
sysroot workflows. These local mocks are valuable for deterministic tests, but
they cannot establish that real RAUC, zbus, systemd, networking, or board I/O
behaves correctly. The feature table therefore reports the implemented seams
and calls out runtime, failure-recovery, and hardware gaps explicitly.

Operators should distinguish a fake-sysroot run from a safe change on a
networked field unit. Credentials, certificates, service users, permissions,
and manifest selection all affect the deployed result.

The service boundary includes both read-only operational views and mutating
configuration actions. Validation should cover malformed schema values,
unauthorized requests, stale jobs, interrupted event streams, failed systemd
operations, and persistence across restart. Network changes deserve special
attention because a device can become unreachable before a confirmation is
received. The fake-sysroot path is useful for exercising deterministic engine
logic and HTTP behavior, while the Yocto layer demonstrates intended image
packaging. Neither substitutes for checking certificates, permissions,
firewall exposure, RAUC slots, zbus services, and recovery behavior on the
actual board. The implementation status below is therefore deliberately more
specific than a general claim that the configuration product is complete.

## Feature Status

The Rust workspace implements an on-device configuration service. Its daemon
contains API modules for system, network, services, updates, monitoring, logs,
alerts, security, and I/O, plus authentication, CSRF, TLS, SSE, jobs, state,
systemd, and bootstrap support. `board-manifests/vela.json` and the generic
manifest provide board data. `crates/webconfigd/src/tests.rs` provides local
test evidence, while real backend, hardware, rollback, and production behavior
still require runtime confirmation.

| # | Name / Description | Implemented? | Test Coverage? | Manual Test |
|---:|---|---|---|---|
| 1 | Rust workspace and webconfig daemon | Implemented | `make test` / Rust tests documented; results not verified | [ ] Manual verification required |
| 2 | System, network, service, update, monitoring, log, alert, security, and I/O APIs | Implemented | Partial in-source coverage | [ ] Manual verification required |
| 3 | Authentication, role access, and CSRF protection | Implemented | Tests present; depth unknown | [ ] Manual verification required |
| 4 | TLS server configuration | Implemented | Deployment coverage unknown | [ ] Manual verification required |
| 5 | SSE event and job state handling | Implemented | In-source tests present; edge cases unknown | [ ] Manual verification required |
| 6 | Schema-driven frontend and JSON schemas | Implemented | Schema/runtime coverage unknown | [ ] Manual verification required |
| 7 | Vela and generic board manifests | Implemented | Manifest validation coverage unknown | [ ] Manual verification required |
| 8 | Engine bridge and persistent configuration | Partial | Tests present; persistence depth unknown | [ ] Manual verification required |
| 9 | Real RAUC/zbus/systemd backends | Partial | Mock/runtime coverage not established | [ ] Manual verification required |
| 10 | Network commit/confirm rollback recovery | Unknown | No verified failure-injection result found | [ ] Manual verification required |

Run `make build`, `make test`, and `make lint` from this repository; use
`make run` for the fake-sysroot development path.

## License

CLOSED (SoCcentric internal IV&V tooling).