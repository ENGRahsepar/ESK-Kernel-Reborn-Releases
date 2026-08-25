# ESK Kernel Reborn — Releases

Official release repository for the **ESK Reborn Kernel**.

- **SoC:** MediaTek Dimensity 8100 (`MT6895`)
- **Kernel:** Linux **5.10 LTS** (stable sublevel tracked)
- **Base:** Android Common Kernel **android12-5.10** + MediaTek/Xiaomi vendor code
- **Source:** [AKALIorg/android_kernel_xiaomi_mt6895](https://github.com/AKALIorg/android_kernel_xiaomi_mt6895)

## Supported Devices (xaga family)

| Device Name | Regional Codename | Model Number(s) | Notes |
|---|---|---|---|
| POCO X4 GT | `xaga` | 22041216G | Global variant |
| Redmi Note 11T Pro | `xaga` | 22041216C | China variant |
| Redmi Note 11T Pro+ | `xagapro` | 22041216UC | China variant (120W charging) |
| Redmi K50i | `xagain` | 22041216I | India variant |

## Variants

| Suffix | Root / Features |
|---|---|
| `KSU-SUSFS` | KernelSU + SUSFS |
| `KSU-SUSFS-LXC` | KernelSU + SUSFS + LXC containers |
| `VNL` | Vanilla (no root) |

Flash the `AnyKernel3.zip` matching your variant. See the [Releases](https://github.com/AKALIorg/ESK-Kernel-Reborn-Releases/releases) tab for downloads.
