# Extlinux boot entries

pocketboot reads `extlinux/extlinux.conf` and `boot/extlinux/extlinux.conf`
on discovered boot filesystems. This does not expand partition discovery:
ESP/XBOOTLDR and supported nested `userdata` boot partitions are still the
candidates. Merely putting a config on another partition will not make it
discoverable.

The compatibility reference is lk2nd's [boot documentation][lk2nd-doc] and
[parser][lk2nd-parser] at `8b46487c4c76776c4f2f61468d44a74c69e6b9ea`.
This is not a complete Syslinux or U-Boot implementation.

## Supported directives

| Directive | Behavior |
| --- | --- |
| `label` | Start an entry. |
| `default` | Prefer the first exact, case-sensitive matching label; absent or unknown defaults select the first declared entry. |
| `linux`, `kernel` | Required kernel file. |
| `initrd` | Optional initramfs; comma-separated files are concatenated in order. |
| `fdt`, `devicetree` | Explicit DTB file. |
| `fdtdir`, `devicetreedir` | Select a DTB using the device identity, as described below. Takes precedence over `fdt` when both are specified. |
| `append` | Kernel command line, without variable expansion. `append -` clears earlier options. |
| `menu label` | Human-readable title in pocketboot's menu. |

Directive names are case-insensitive. Blank lines and full-line `#` comments
are ignored. Per-entry settings are not inherited from before the first label
or from other labels. Repeating a setting replaces its previous value, including
`append` and `initrd`. Unknown directives, including `timeout`, are ignored.

Unlike lk2nd, pocketboot presents all usable entries in its own menu, with the
preferred entry first and the others in config order. Missing kernels,
initrds, or requested DTBs exclude an entry with a diagnostic; other entries
remain available. An invalid default is not silently assigned to another label.

## Paths and DTBs

Paths starting with `/` are relative to the mounted boot filesystem, not the
running initramfs. Other paths are relative to the config directory. For example,
with `extlinux/extlinux.conf`, `/vmlinuz` and `../vmlinuz` both select a file
at the filesystem root, whereas `vmlinuz` selects `extlinux/vmlinuz`.
Absolute symlink targets are also boot-filesystem-relative. Symlinks and `..`
are resolved, but attempts to walk above the boot filesystem root are rejected.
Link expansion is bounded to reject loops.

For `fdtdir`, pocketboot shares BLS's filename search, using the packaged
`/chosen/pocketboot,mainline-compatible` property or, when absent, the running
DT's root `compatible` list. This differs from lk2nd's per-device DTB hint table.
Board-specific names are tried before generic SoC names. For Ferrari's
`xiaomi,ferrari`, `qcom,msm8939` identity, the search includes:

- `qcom/msm8939-xiaomi-ferrari.dtb` (Linux arm64 layout)
- `qcom-msm8939-xiaomi-ferrari.dtb` (vendor-prefixed flat layout)
- `msm8939-xiaomi-ferrari.dtb` (boot-deploy's flattened layout)

Failure to resolve a requested `fdt` or `fdtdir` rejects the entry. Only omitting
both retains pocketboot's existing live-DTB fallback. Unlike lk2nd, a DTB
directive is not mandatory.

**DT overlays are not implemented.** Entries with nonempty `fdtoverlays` or
`devicetree-overlay` are rejected rather than booting with a silently incomplete
device tree.

## Example

Illustrative config for a separate boot filesystem (adjust the filenames and
root argument to the actual installation):

```text
default postmarketOS
label postmarketOS
    menu label postmarketOS
    linux /vmlinuz
    initrd /initramfs
    fdt /dtbs/msm8939-xiaomi-ferrari.dtb
    append root=LABEL=pmOS_root quiet
```

The explicit `fdt` line can instead be `fdtdir /dtbs` when the packaged/live
identity is correct. Config discovery and payload resolution are not proof
of a successful kexec handoff; see [Ferrari bring-up](ferrari-bringup.md).

[lk2nd-doc]: https://github.com/msm8916-mainline/lk2nd/blob/8b46487c4c76776c4f2f61468d44a74c69e6b9ea/Documentation/boot.md
[lk2nd-parser]: https://github.com/msm8916-mainline/lk2nd/blob/8b46487c4c76776c4f2f61468d44a74c69e6b9ea/lk2nd/boot/extlinux.c
