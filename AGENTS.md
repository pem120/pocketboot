# AGENTS.md

LinuxBoot-style bootloader: builds an Android boot image containing a minimal
Linux kernel plus a Rust `/init` that kexecs a real OS. Cargo workspace members:
root package `pocketboot` (the `/init`), `pocketpreboot` (freestanding shim),
and `xtask` (the only build driver; no Makefile/justfile).

## Commands

Required Rust targets: `aarch64-unknown-linux-musl`, `aarch64-unknown-none`,
`armv7-unknown-linux-musleabihf` (configured in `.cargo/config.toml`, all use
`rust-lld`).

- `cargo xtask build [VENDOR/DEVICE]` — kernel + boot.img; with no device it
  lists configured devices. Alias expands to `cargo run --package xtask --`.
- Other subcommands: `kernel`, `bootimg`, `cpio`, `preboot`, `busybox`,
  `kernel-src`, `qemu`, `ci-matrix`.
- `cargo fmt --all -- --check` is the only lint/format check (no clippy config).
- Verification set (from `docs/msm8916-validation.md`):
  - `cargo test --offline --workspace --features pocketpreboot/soc-msm8916`
  - `CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_RUNNER=qemu-aarch64 cargo test --offline -p pocketboot --target aarch64-unknown-linux-musl kexec::`
  - `python3 -m unittest discover -s tools/tests -v` — the ramoops tests
    compile a host codec from the fetched kernel tree, so run
    `cargo xtask kernel-src qcom/msm8916-samsung-a5u-eur` first.
  - `bash pocketpreboot/tests/qemu/run.sh` — requires `qemu-system-aarch64` and
    `aarch64-linux-gnu-{as,ld,objcopy}`; not part of cargo test.

## Architecture and config

- Device ID is a kernel DTB path, `vendor/stem`, without `.dts`/`.dtb` suffix.
  The SoC is the stem prefix before the first `-` (`msm8916-samsung-a5u-eur`
  → `msm8916`).
- Config layers merge in order: `configs/pocketboot.toml` →
  `configs/soc/<vendor>/<soc>.toml` → `configs/device/<vendor>/<stem>.toml`.
  All structs use `deny_unknown_fields`, so unknown keys fail the build.
  `[kconfig]` keys are bare symbols (no `CONFIG_` prefix); `[kconfig.<feature>]`
  tables apply only when that feature is enabled in the device's `[features]`.
- `[kernel-source]` pins `remote` + `sha`; `patches` are workspace-relative and
  applied as one `git apply` stream. `kernel-src` refuses dirty trees, and
  changing/removing an already-applied series requires a fresh tree (delete
  `target/kernel/src/<tree>`). A top-level `kernel/` directory, if present, is a
  shared reference git repo used to create worktrees — not build output.
- Outputs live under `target/`: `target/kernel/<vendor>/<stem>/` (Image,
  `.config`, `boot.img`, plus `pocketboot.dtb` when a DT overlay or DTBO mask
  applies), `target/cpio/`, `target/preboot/`, `target/busybox/`.
- Adding a device config requires adding its ID to
  `all_checked_in_device_configs_load` in `xtask/src/commands/config.rs`; several
  devices also have invariant tests in that file that must keep passing.
- `pocketpreboot` is position-independent (`aarch64-unknown-none`); SoC/device
  selection is via cargo features (`soc-msm8916`, `device-msm8916-samsung-a5u-eur`,
  ...) that xtask adds automatically. Read `pocketpreboot/README.md` and
  `patches/kernel/msm8916/README.md` before touching the spin-table/CPU-parking
  code, linker script, or QEMU tests.
- DT overlays: `configs/dt-overlays/<vendor>/<stem>.dtso`; DTBO-mask manifests:
  `configs/dtbo-masks/...`. Mask generation and fail-closed FDT validation live
  in `xtask/src/commands/kernel.rs`.
- Toolchain overrides via env: `CROSS_COMPILE`, `LLVM`, `MAKE`, `OBJCOPY`,
  `BUSYBOX_CC`/`BUSYBOX_CFLAGS`/`BUSYBOX_CROSS_COMPILE`, `QEMU`. Setting
  `CCACHE_DIR` makes xtask write compiler wrappers into the kernel out dir; it
  deliberately skips distro ccache shims to avoid recursion.
- CI runs in a GHCR image built from `.github/Dockerfile` (tagged by file
  hash) with the matrix emitted by `cargo xtask ci-matrix`. PRs from outside the
  repo need the `ci-ok` label or CI refuses to run.
- `docs/attic/` is archived evidence, not current behavior. `.gitattributes`
  marks `*.patch` as `-whitespace`; don't reformat patches.
