# ESK Reborn Kernel — Installation Guide

Applies to all supported devices: **POCO X4 GT (`xaga`)** · **Redmi Note 11T Pro (`xaga`)** · **Redmi Note 11T Pro+ (`xagapro`)** · **Redmi K50i (`xagain`)**

> ⚠️ Unlocking the bootloader and flashing custom software carries risk. Always keep a full backup before proceeding. You do this at your own risk.

---

## 1. Requirements

- **Unlocked bootloader** (Xiaomi unlock via Mi Unlock tool)
- PC with **ADB + Fastboot** (platform-tools) installed
- USB debugging enabled (**Settings → Developer options**)
- Battery above **50%**
- The correct `AnyKernel3.zip` from the [Releases](https://github.com/AKALIorg/ESK-Kernel-Reborn-Releases/releases) tab

### Pick your build

| File suffix | Root / Features |
|---|---|
| `-KSU-SUSFS-` | **ReSukiSU** root + SUSFS 2.2.0 |
| `-KSU-SUSFS-LXC-` | Same + LXC container support |
| `-VNL-` | Vanilla, no root |

---

## 2. Before you flash

1. Boot the device and connect it to the PC, then verify ADB works:
   ```
   adb devices
   ```
2. Make a **backup of your current boot image** so you can restore later:
   ```
   adb shell su -c "dd if=/dev/block/by-name/boot of=/sdcard/boot_backup.img"
   adb pull /sdcard/boot_backup.img
   ```
   *(If you don't have root yet, back up `boot` and `vendor_boot` from your custom recovery instead.)*

---

## 3. Method A — Custom Recovery (TWRP / OrangeFox / ROM recovery)

1. Reboot to recovery:
   ```
   adb reboot recovery
   ```
2. *(Recommended)* Take a backup of **Boot** and **Vendor Boot** partitions.
3. Go to **Install** (or **Apply update**) and select the downloaded `AnyKernel3.zip`.
4. Swipe/confirm to flash.
5. Reboot to system:
   ```
   adb reboot system
   ```

---

## 4. Method B — ADB Sideload (no touch needed)

1. Reboot to recovery:
   ```
   adb reboot recovery
   ```
2. In recovery open **Apply update → Apply update from ADB** *(on MIUI/HyperOS recovery: **Menu → Connect with MIAssistant** is NOT sideload — use a custom recovery or ROM recovery instead)*.
3. On the PC confirm the device is in sideload mode:
   ```
   adb devices
   ```
   It should list the device with state `sideload`.
4. Start the sideload (use your exact file name):
   ```
   adb sideload ESK-Reborn-5.10.266-KSU-SUSFS-f8e3dbe2abb9-AnyKernel3.zip
   ```
5. Wait until it reports `Total xfer` / success, then reboot:
   ```
   adb reboot system
   ```

> 💡 If the transfer stops around ~47% on some recoveries, that is normal — check the message on the phone screen before assuming failure.

---

## 5. After installing

- Install the **ReSukiSU Manager** APK to control root access (KSU builds only).
- **SUSFS 2.2.0** is built into the kernel — no extra flashing needed. Configure hiding inside ReSukiSU Manager.
- `-LXC` build: run containers via any LXC-capable manager app (e.g., Termux + LXC tooling).
- If a ROM update replaces the kernel, simply re-flash the matching AnyKernel3 zip afterwards.

---

## 6. Restoring stock / rolling back

Flash your backed-up boot image:

```
adb reboot bootloader
fastboot flash boot boot_backup.img
fastboot reboot
```

Or re-flash your ROM's full package (its kernel will replace this one).

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Bootloop after flash | Wipe **Cache/Dalvik**, re-flash; if still stuck restore `boot_backup.img` via fastboot |
| Root missing after OTA | ROM replaced the kernel — re-flash the AnyKernel3 zip |
| Wrong variant flashed | Just flash the intended zip on top, no wipe needed |
