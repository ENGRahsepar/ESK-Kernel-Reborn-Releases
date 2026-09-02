# ESK Kernel Reborn — Releases

Official release repository for the **ESK Reborn Kernel**.

- **SoC:** MediaTek Dimensity 8100 (`MT6895`)
- **Kernel:** Linux **5.10 LTS** (stable sublevel tracked)
- **Base:** Google **android12-5.10** common kernel (the devices' stock base) + MediaTek/Xiaomi vendor code
- **ROM support:** Android **16 / 17** custom ROMs *(not tested)*
- **Source:** [AKALIorg/android_kernel_xiaomi_mt6895](https://github.com/AKALIorg/android_kernel_xiaomi_mt6895)

## Supported Devices (xaga family)

| Device Name | Regional Codename | Model Number(s) | Notes |
|---|---|---|---|
| POCO X4 GT | `xaga` | 22041216G | Global variant |
| Redmi Note 11T Pro | `xaga` | 22041216C | China variant |
| Redmi Note 11T Pro+ | `xagapro` | 22041216UC | China variant (120W charging) |
| Redmi K50i | `xagain` | 22041216I | India variant |

## Versioning

`uname -r` carries the release tag, e.g. `5.10.269-android12-...-ESK-Reborn_V0.3`. The suffix in `CONFIG_LOCALVERSION` (xaga.config) is bumped for every release (`V0.1`, `V0.2`, ...).

## Variants

| Suffix | Root / Features |
|---|---|
| `KSU-SUSFS` | **ReSukiSU** (KernelSU) + SUSFS **v2.2.0** |
| `KSU-SUSFS-LXC` | **ReSukiSU** (KernelSU) + SUSFS **v2.2.0** + LXC containers |
| `VNL` | Vanilla (no root) |

All KernelSU-based builds use **ReSukiSU** with **SUSFS 2.2.0**. Flash the `AnyKernel3.zip` matching your variant — see the [Installation Guide](INSTALL.md) (custom recovery + `adb sideload`). Downloads in the [Releases](https://github.com/AKALIorg/ESK-Kernel-Reborn-Releases/releases) tab.
