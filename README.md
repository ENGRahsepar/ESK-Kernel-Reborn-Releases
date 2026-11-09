# ESK Kernel Reborn — Releases

Official release repository for the **ESK Reborn Kernel**.

- **SoC:** MediaTek Dimensity 8100 (`MT6895`)
- **Kernel:** Linux **5.10 LTS** (stable sublevel tracked)
- **Base:** Google **android12-5.10** common kernel (the devices' stock base) + MediaTek/Xiaomi vendor code
- **ROM support:** Android **16 / 17** custom ROMs *(not tested)*
- **Source:** [ENGRahsepar/android_kernel_xiaomi_mt6895](https://github.com/ENGRahsepar/android_kernel_xiaomi_mt6895)

## Supported Devices (xaga family)

| Device Name | Regional Codename | Model Number(s) | Notes |
|---|---|---|---|
| POCO X4 GT | `xaga` | 22041216G | Global variant |
| Redmi Note 11T Pro | `xaga` | 22041216C | China variant |
| Redmi Note 11T Pro+ | `xagapro` | 22041216UC | China variant (120W charging) |
| Redmi K50i | `xagain` | 22041216I | India variant |

## Versioning

`uname -r` carries the release tag, e.g. `5.10.274-android12-...-ESK-Reborn_V0.3.3`. The suffix in `CONFIG_LOCALVERSION` (xaga.config) is bumped for every release (`V0.1`, `V0.2`, ...).

## Variants

| Suffix | Root / Features |
|---|---|
| `KSU-SUSFS` | **BakaSU** (KernelSU) + SUSFS **v2.3.0** |
| `KSU-SUSFS-LXC` | **BakaSU** (KernelSU) + SUSFS **v2.3.0** + LXC containers / Droidspaces |
| `VNL` | Vanilla (no root) |

All KernelSU-based builds use **BakaSU v4.2.0-rc3** (`sync-susfs` branch) with **SUSFS v2.3.0** pinned at the verified pairing commit. Flash the `AnyKernel3.zip` matching your variant — see the [Installation Guide](INSTALL.md) (Apply update → Apply from ADB; no TWRP on this device). Downloads in the [Releases](https://github.com/ENGRahsepar/ESK-Kernel-Reborn-Releases/releases) tab.

## Kernel features (0.3+)

- **Bypass charging** (xaga) — VBUS powers the system, battery idles ~0mA. Root shell:
> ```
> su -c "echo 1 > /sys/class/power_supply/usb/bypass_charging"  # on
> su -c "echo 0 > /sys/class/power_supply/usb/bypass_charging"  # off
> ```
> Needs charger plugged (67W-class recommended so VBUS covers gaming load); resets on reboot; compatible with the SimpleBypassChargerMTK module.
- **ESK governor** (opt-in) — `echo esk > /sys/devices/system/cpu/cpufreq/policy{0,4,7}/scaling_governor`; `gaming_mode` + `prime_gaming_floor_pct` tunables on policy7. Default governor stays **schedutil** until multi-day stability is confirmed.
- **TEO cpuidle governor** (0.3.3+) — upstream TEO fix set wired into the defconfig, prioritized over the other idle governors.
- **NoMount v2.0.0** (0.3.3+) — patchless dcache hijack, Bloom-filter child indexing, UID isolation, RCU-safe rule lookups.
- **Droidspaces / LXC containers** (0.3.3+) — all required GKI namespace/cgroup/netfilter configs enabled, with LXC kABI patches in-tree (`sched.h` reserves 6/7/8) so the GKI ABI is not shifted.

## Release history

| Release | Tag | Kernel base | Notes |
|---|---|---|---|
| 0.1 | `0.1` | 5.10.266 | First public; VNL / KSU-SUSFS / KSU-SUSFS-LXC |
| 0.2 | `0.2` | 5.10.268 | 5.10.267/268 merges, huge_memory fix |
| 0.3 | `0.3` | 5.10.270 | 5.10.269 merge, all crash fixes, ESK v2.2 opt-in, bypass charging |
| 0.3.2 | `0.3.2` | 5.10.272 | BakaSU rename + susfs pairing, 271/272 stable, conntrack fix |
| **0.3.3 (pre-release)** | `0.3.3` | **5.10.274** | **NoMount v2.0.0, Droidspaces/LXC, TEO upstream fixes, cpufreq-hw thermal pressure, RCU NOCB, DTBO support, LZ4 fixes** |
