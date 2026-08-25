# ESK Reborn Kernel — Installation Guide

Applies to all supported devices: **POCO X4 GT (`xaga`)** · **Redmi Note 11T Pro (`xaga`)** · **Redmi Note 11T Pro+ (`xagapro`)** · **Redmi K50i (`xagain`)**

> ⚠️ Unlocking the bootloader and flashing custom software carries risk. You do this at your own risk.

---

## 1. Requirements

- **Unlocked bootloader** (Xiaomi unlock via Mi Unlock tool)
- PC with **ADB** (platform-tools) installed
- USB debugging enabled (**Settings → Developer options**)
- Battery above **50%**
- **AOSP recovery** (ships with your custom ROM) — there is no TWRP/OrangeFox for these devices
- The correct `AnyKernel3.zip` from the [Releases](https://github.com/AKALIorg/ESK-Kernel-Reborn-Releases/releases) tab

### Pick your build

| File suffix | Root / Features |
|---|---|
| `-KSU-SUSFS-` | **ReSukiSU** root + SUSFS 2.2.0 |
| `-KSU-SUSFS-LXC-` | Same + LXC container support |
| `-VNL-` | Vanilla, no root |

---

## 2. Installation — Apply update from ADB

1. Connect the device to the PC and verify ADB works:
   ```
   adb devices
   ```
2. Reboot to recovery:
   ```
   adb reboot recovery
   ```
3. In the AOSP recovery menu select **Apply update** → **Apply from ADB**.
4. On the PC confirm the device is in sideload mode:
   ```
   adb devices
   ```
   It should list the device with state `sideload`.
5. Start the sideload (use your exact file name):
   ```
   adb sideload ESK-Reborn-5.10.266-KSU-SUSFS-f8e3dbe2abb9-AnyKernel3.zip
   ```
6. Wait until it reports `Total xfer` / success, then reboot:
   ```
   adb reboot system
   ```

> 💡 If the transfer stops around ~47% on some recoveries, that is normal — check the message on the phone screen before assuming failure.

---

## 3. After installing

- Install the **ReSukiSU Manager** APK to control root access (KSU builds only).
- **SUSFS 2.2.0** is built into the kernel — no extra flashing needed. Configure hiding inside ReSukiSU Manager.
- `-LXC` build: run containers via any LXC-capable manager app (e.g., Termux + LXC tooling).
- If a ROM update replaces the kernel, simply re-flash the matching AnyKernel3 zip afterwards.

---

## 4. Rolling back

Re-flash your ROM's full package (its kernel will replace this one), or flash any other kernel the same way via **Apply update → Apply from ADB**.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Bootloop after flash | Re-flash the correct zip; if still stuck, re-flash your ROM package |
| Root missing after OTA | ROM replaced the kernel — re-flash the AnyKernel3 zip |
| Wrong variant flashed | Just flash the intended zip on top, no wipe needed |
