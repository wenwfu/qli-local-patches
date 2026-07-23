# qli-local-patches

A **kas local patch overlay** for the [QLI-mainline](https://github.com/qualcomm-linux)
(Qualcomm Linux, mainline) Yocto build. It carries boot-time / QuickBoot
development and instrumentation patches **without touching any upstream layer or
the upstream-generated buildconf lock**, so those stay pristine across nightly
syncs.

## How it works

Both the overlay declaration (`local-patches.yml`) and the patch **files** live
in this one directory. The overlay declares a url-less `local-patches` repo
(a plain directory — kas performs no VCS operations on it) and deep-merges the
patch entries onto the repos defined by the buildconf lock it includes:

```yaml
header:
  includes:
    - ../meta-qcom-buildconf/lock.yml   # file-relative: one level up
```

Because the include path is **file-relative** (`../meta-qcom-buildconf/lock.yml`),
this repo must be checked out **as `local-patches/` inside the QLI-mainline
workspace root**, alongside `meta-qcom-buildconf/`, `meta-qcom/`, `oe-core/`,
etc.:

```
QLI-mainline/                 # kas work dir (workspace root)
├── meta-qcom-buildconf/      # upstream: provides lock.yml
├── meta-qcom/  oe-core/ ...  # upstream layers
└── local-patches/            # <-- THIS repo
    ├── local-patches.yml
    ├── meta-qcom/*.patch
    ├── oe-core/*.patch
    └── ...
```

## Usage

Applied via the workspace `build.sh` two-step flow (needs an explicit `--lock`,
since the default-lock flow has no checkout step):

```sh
./build.sh <device> --lock --local-patches
# --local-patches (bare) defaults to local-patches/local-patches.yml
```

kas emits a harmless `Falling back to file-relative addressing` warning for the
include — expected.

## What's here

`local-patches.yml` is the source of truth for **which** patches are active;
entries are commented in/out there. Patch files are grouped by the upstream repo
they target:

| Dir | Targets | Contents |
|-----|---------|----------|
| `meta-audioreach/` | meta-audioreach | APM timeout + snd-card retry boot-time opt |
| `meta-qcom-distro/` | meta-qcom-distro | QuickBoot runtime manager + camera/audio/display/udev recipes → `qcom-multimedia-image` |
| `oe-core/` | oe-core | BOOTPROF systemd unit-load instrumentation, initramfs timing markers, early tracefs mount, block-only udev trigger |
| `meta-qcom/` | meta-qcom | boot-time ftrace cmdline, SELinux allow for tracefs markers, PCIe1 disable, build-only-proprietary-image dev speedup |
| `meta-updater/` | meta-updater | console baud bump, ostree prepare-root timing (currently disabled) |

Naming: `DEBUG-*` = instrumentation/measurement only; `DEV-*` = developer
convenience (not for product). See per-entry comments in `local-patches.yml` for
what each patch does and when to drop it.

> The upstream mariadb patch is intentionally **not** listed here — it ships in
> the buildconf lock itself; duplicating it would apply it twice and fail.
