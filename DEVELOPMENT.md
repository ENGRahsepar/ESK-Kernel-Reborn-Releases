# ESK Reborn Kernel — Developer Handoff Document

**Purpose:** This file is a complete context-transfer document. Give this file plus the
kernel source repository to any AI model or developer, and they can continue development
without any prior session knowledge.

- **Repo (kernel):** https://github.com/ENGRahsepar/android_kernel_xiaomi_mt6895
- **Branch:** `17.0` (Android 17 baseline; `16.2-rebase` remains for 16.x ROMs)
- **Push policy: every functional change AND every stable merge lands on BOTH
  `17.0` and `16.2-rebase`** (merge direction: `16.2-rebase` → `17.0`, then
  push both + verify each with `git log --oneline origin/<branch> -1`).
- **Repo (releases):** https://github.com/ENGRahsepar/ESK-Kernel-Reborn-Releases
- **Maintainer / owner:** ENGRahsepar (alirahsepar199@gmail.com), GitHub user `ENGRahsepar` (renamed from `AKALIorg`, Oct 2026; old URLs redirect)
- **Device:** Xiaomi POCO X4 GT (`xaga`), 8GB RAM — Dimensity **8100** (MT6895Z):
  4×Cortex-A78 @ 2.85GHz (CPU4-6 = Big cluster, CPU7 = **Prime, separate DVFS domain**)
  + 4×Cortex-A55 @ 2.0GHz (CPU0-3 = Little). DTS: `capacity-dmips-mhz` = 380 (little) /
  1024 (big+prime). THREE cpufreq performance-domains in `mt6895.dts` (0=CPU0-3, 1=CPU4-6, 2=CPU7).
- **OS base:** Google **android12-5.10** GKI common kernel (device ships with Android 12);
  ROM support target: Android 16/17 custom ROMs (A17 boot verified Sep 2026 on
   `Angxddeep/android_kernel_xiaomi_mt6895:seventeen` @ 5.10.264 — our 5.10.270
   is a strict superset; binderfs/ashmem present, LXC USER_NS via builder).
- **Current stable sublevel:** **5.10.272** (tracked; check kernel.org for newer).
- **Localversion convention:** `CONFIG_LOCALVERSION="-ESK-Reborn_V0.X"` in
  `arch/arm64/configs/vendor/xaga.config` — **bump per release** (V0.3.2 on both
  branches since Oct 2026 (was V0.3); the `17.0` V17.0 bump was reverted in `0edb2f37`).
  `uname -r` shows `5.10.272-android12-...-ESK-Reborn_V0.3.2/<git-sha12>`.

---

## 1. CRITICAL BUILD & TEST INFRASTRUCTURE

### 1.1 Builder
- Builds happen via a private builder at `~/esk_builder/` on the maintainer's PC
  (scripts: `build.sh`, `config.sh`, `build/compile.sh`, `build/setup.sh`).
- **Config merge order** (from `build/compile.sh`):
  `scripts/kconfig/merge_config.sh -m -r gki_defconfig vendor/xiaomi_mt6895.config vendor/xaga.config`
- **Builder patches** applied at setup time (in this order):
  1. SuSFS: `git clone gitlab.com:simonpunk/susfs4ksu -b gki-android12-5.10`, copies
     `kernel_patches/fs/*` + `include/*`, then `patch -p1 --fuzz=3 < 50_add_susfs_in_gki-android12-5.10.patch`
     — **fixup for `769e31f` statfs bug (see §14):** kernel `fs/statfs.c:12` now carries early
     `extern susfs_is_inode_sus_kstat`/`susfs_sus_kstat_spoof_vfs_statfs`/`susfs_get_non_sus_vfsmnt`
     (guarded by `CONFIG_KSU_SUSFS_*`) so a clean clone + patch already compiles; builder
     `build/setup.sh:apply_susfs()` keeps an idempotent `awk` injector as a safety net for
     older trees — upstream patch declares them after first use, triggering
     `-Werror=implicit-function-declaration` with clang 22.
  2. **LXC support:** `~/esk_builder/kernel_patches/lxc_support.patch` (adds SYSVIPC,
     POSIX_MQUEUE, namespaces, CGROUP_DEVICE, NAT netfilter bits to gki_defconfig —
     uses ANDROID_KABI_RESERVE(6/7/8) in `include/linux/sched.h`; ESK's BORE uses reserves 1-4,
     no conflict)
  3. `stock_config.patch` (only for stock builds)
  4. KernelSU variants: clones `Baka-SU/BakaSU` (branch main; renamed from ReSukiSU Oct 2026 — same `drivers/kernelsu` + `CONFIG_KSU`/`CONFIG_KSU_SUSFS` interface), enables `CONFIG_KSU`
  5. Variants: VNL (vanilla), KSU-SUSFS, KSU-SUSFS-LXC (LXC flag applies lxc_support.patch)
- **Toolchain:** Android clang (currently r596125-based, "clang 22.0.2") at `~/esk_builder/clang/`.
  Build flags: `LLVM=1 LLVM_IAS=1`, `ARCH=arm64`, out-dir = `~/esk_builder/work`,
  source copy = `~/esk_builder/kernel` (the builder syncs/pulls from GitHub; if manually syncing,
  copy every changed file into BOTH `~/esk_builder/kernel/` and `~/esk_builder/work/`).
- Outputs: `~/esk_builder/out/` → AnyKernel3 zips named
  `ESK-Reborn-<ver>-<variant>-<git-sha12>-AnyKernel3.zip` + `module.tar.xz` (608 modules).

### 1.2 Build verification protocol (ALWAYS do before committing kernel changes)
Single-file compile tests with the builder's clang against the prepared work tree:

```bash
K="/home/akali/Documents/Default Project/android_kernel_xiaomi_mt6895"
W=~/esk_builder/work
# copy changed files to ~/esk_builder/kernel/<path> ONLY (srctree) - NEVER
# copy sources into $W: a $W copy SHADOWS srctree for that file and breaks
# its quoted sibling includes (2026-09-14: services.c shadow without
# context.h broke the full build with a bogus 'context.h not found').
# If a build fails on a missing sibling header, check for $W shadows first:
#   for f in $(git diff HEAD --name-only); do rm -f ~/esk_builder/work/$f; done
cd $W && export PATH=~/esk_builder/clang/bin:$PATH
# config: merge fresh when needed
bash ~/esk_builder/kernel/scripts/kconfig/merge_config.sh -m .config <(fragment) 
make O=. ARCH=arm64 CC=clang HOSTCC=clang LLVM=1 LLVM_IAS=1 olddefconfig
make O=. ARCH=arm64 CC=clang HOSTCC=clang LLVM=1 LLVM_IAS=1 <path/to/file.o>
```
Notes:
- `O=` builds read sources from `~/esk_builder/kernel` (srctree), falling back
  to it whenever a file is absent under `work/`. A source file present in BOTH
  places compiles the `work/` copy — with ITS directory as the quoted-include
  base — so a partial sync (e.g. services.c without context.h) fails the build
  while srctree alone is fine. Rule: sources live ONLY in srctree; `work/` holds
  outputs (.o/.cmd/.config) plus build-GENERATED sources (.asn1.c/.mod.c) which
  must stay. 454 legacy .c + 3508 .h copies predate this rule in `work/` and are
  benign (full builds passed with them); do NOT bulk-delete, only remove shadows
  of files you are actively verifying.
- vdso prepare may fail on the host (bfd vs llvm emulation) — ignore, build single objects.
- NEVER trust `git push` — **always verify with `git log --oneline origin/<branch> -1`** after
  pushing. Pushes have silently failed before.

### 1.3 Debugging crashes on device (proven workflow)
- Cmdline contains `aee_aed.poffreason=AP_WDT` → `AP_WDT` = hardware watchdog reset
  (full system hang, NO kernel panic logged). `ramoops.mem_address=0x48090000`,
  console_size=0x40000 (wraps fast — the panic moment may be missing).
- Grab logs **immediately** (before another reboot):
```bash
adb shell su -c 'ls /sys/fs/pstore/'
adb shell su -c 'cat /sys/fs/pstore/console-ramoops-0' > ramoops-console.txt
adb shell su -c 'dmesg' > dmesg-now.txt
adb logcat -d > logcat-main.txt
adb logcat -d -b events > logcat-events.txt
```
- For recoverable stalls (20s freeze, no reboot): enable hung-task detection FIRST,
  then reproduce, then pull dmesg — this gives the exact lock + holder:
```bash
adb shell su -c 'sysctl -w kernel.hung_task_timeout_secs=10'
adb shell su -c 'sysctl -w kernel.hung_task_warnings=100'
# reproduce, then:
adb shell su -c 'dmesg' > hung_task_dump.txt
```
- Red flags in logs: `SPM system_bus didn't enter low power` every ~5s +
  `[RC] ratio ... dram:0% syspll:0%` = 100% busy livelock. `sched: RT throttling activated`
  = RT overload (symptom, usually not root cause). `audit_lost=<huge>` = SELinux denial
  flood (userspace noise, MI ROM).
- Known harmless: `mali_submit not set to RT prio` (Mali RT kthread bind fails under
  cpuset/thermal churn — pre-existing), `gpufreq_fix_target_oppidx: fail to fix STACK OPP
  index` (userspace booster procfs write rejected by GPUEB — thermal guard).

---

## 2. FEATURE INVENTORY (what is in the tree, commit references)

Base: `dd3b1030` = 5.10.266 vendor tree. Current HEAD sequence (all pushed to `16.2-rebase`):

| Commit | What |
|---|---|
| `f8e3dbe2` | ANDROID: imgsensor frame_sync 4K stack warning fix (first ESK commit) |
| `221f9dae`/`df46125d` | 5.10.267 + 5.10.268 stable merges |
| `03e2cd2b` | huge_memory `flags` build fix after 267 |
| `3680ab2b` | **NoMount** (maxsteeel/nomount, keyring-based) built-in at `fs/nomount/`, `CONFIG_NOMOUNT=y`; Droidspaces netfilter bits + `TMPFS_POSIX_ACL` + `WQ_POWER_EFFICIENT_DEFAULT` in `vendor/xiaomi_mt6895.config` |
| `2256037a` | **EEVDF + BORE backport** (from Templar-Kernel-GKI-5.10 commits `3e7145c6`,`bb8bac7d`,`e37771cf`,`8c0ad4c4`,`f65f0962`) — applied as patch files with `git apply`, some hunks needed manual application (`git apply --exclude=kernel/sched/features.h` + manual features.h hunk) |
| `7760bd3f` | **BBRplus** (UJX6N/bbrplus-5.10 + Templar tuning); added `bbrplus_tso_segs()` shim because 5.10 `tcp_congestion_ops` has `tso_segs(sk,mss)` not `min_tso_segs(sk)` |
| `3092c431` | **le9uo** working set protection (Templar `c0762abeb0` + `6476d4d5c8`) |
| `db442051` | **ESK governor** = Templar Vorpal v2.1 renamed (vorpal→esk, rfx_→esk_); `drivers/cpufreq/cpufreq_esk.c`; helpers `esk_get_util_gki510`/`esk_dl_bw_exceeded_gki510` appended to `kernel/sched/cpufreq_schedutil.c`, declared in `include/linux/sched/cpufreq.h` |
| `7f86a52f` | `CONFIG_BRIDGE_NETFILTER=y` |
| `9a8402c7` | ESK v2 daily retune (later partially softened — see history) |
| `b55531ca` | **5.10.269** stable merge (clean; brought upstream u64-to_ratio fix already) |
| `fe6ff006` | `ZRAM_WRITEBACK=y` (later REMOVED), battery bits |
| `5a64ca47` | **ZRAM_WRITEBACK removed** (UFS health), **KSM=y** added (off at boot) |
| `eacd4cc4` | localversion → `_V0.3` |
| `a0810861` | default governor → schedutil (ESK opt-in) after freeze reports |
| `ecfce530` | ESK skippability probe (fast-switch idle re-eval skip) |
| `1bf9ef41` | BBRv3 mobile tuning (pacing margin 2%, PROBE_RTT 140ms, probe base 1.9s; min_rtt window KEPT at 10s per Templar's later cellular fix `f94513b439`) |
| `87c22842` | **PELT half-life 16ms** (Templar's `pelt.c` runtime-switcher + `sched-pelt.h` tables pelt8/12/16/32 + Kconfig choice in `init/Kconfig`), `RCU_BOOST=y delay=15`, `IDLE_PAGE_TRACKING=y` |
| `91d0b1ff` | ESK irq_work/work_lock unconditional init |
| `47b9606d` | limits_changed 1ms gate + clamp-only fast path |
| `b1d029ca` | **CRITICAL FIX: two early-returns inside raw_spin_lock in `esk_update_shared()`** (clamp path + skippable path) — held `update_lock` with IRQs off → AP_WDT. Both now unlock before returning |
| `db15178e` | PELT `LOAD_AVG_PERIOD`/`LOAD_AVG_MAX` compile-time constants when fixed half-life selected; `LOAD_AVG_MAX` moved inside `#else` branch (Werror macro-redefined fix) |
| `527111b4` | **binder freeze fix**: `binder_install_single_page()` uses `mmap_write_lock_killable()` — cgroup-frozen app holding mmap_sem read during page fault blocked all binder txns into it (root cause of 20s game freezes + app kills) |
| `25176a53` | **le9uo ratios DISABLED (0/0/0)** — root cause of game-specific freezes: `clean_min_ratio=25%` (~1GB floor) made `shrink_folio_list()` keep_every_page under gaming memory pressure → direct reclaim zero-progress livelock. Sysctls remain runtime-tunable |
| `90bac8e9` | **EEVDF-functional sched_yield** (skip-buddy is a no-op under EEVDF; now refreshes slice/deadline for entitled entities) + **BORE reweight deadline refresh** (reweight_task_by_prio now recomputes slice/deadline; needs forward decls of `sched_slice`/`calc_delta_fair`) + removed dead `set_skip_buddy` |
| `25462d21` | **ESK three-tier classification + Prime-selective gaming boost**: prime = policy owning topmost CPU (`esk_policy_is_prime()`), NOT capacity alone; new sysfs `prime_gaming_floor_pct` (0-100, default 70) on prime tunables; big tier releases to idle floor at demand<40% in gaming (prime holds 25%) |
| `088c0f238` | **ESK governor upgraded to v2.2** (full port of Vorpal Linux6-Staging end state `f3efbafdbbc2`): see §3 for the feature delta; drops our skippability probe + pending-clamp machinery (bug class); keeps topmost-CPU prime classification + `prime_gaming_floor_pct` (default now 64); new `esk_setattr_sugov_gki510()` schedutil helper |
| `779b6e85` | **EEVDF rescale on reweight** (Templar `77672e88`, mainline `eab03c23`): new `reweight_eevdf()` preserves lag/slice across weight change; fixes `min_deadline` rbtree corruption / 1%-low drops |
| `659dc646` | **BORE weight fix + fork_atavistic clamp** (Templar `2a8e879f`): TASK_NEW no reweight, IDLE keeps `WEIGHT_IDLEPRIO`, reset walk uses `task_rq_lock`+`for_each_process_thread`, `fork_atavistic` extra2 → `SYSCTL_ZERO` |
| `9b9ad306` | **sched_yield EEVDF fix** (Templar `8a9affcf`): `deadline += one slice` replaces no-op skip buddy; supersedes our `90bac8e9` entitled-entity variant |
| `3f5a3f13` / `cf88c2d8` | **le9uo reclaim fix + default 5** (Templar `b21d737` + `10d579`): file-reclaim no longer blocked by `anon_below_min` on swapless, `clean_low_ratio` default 15→5 |
| `ed9ed6f3` | **media: mtk-aie CID guard** (XagaForge `f89055e3` + `23f18ca4`): `mtk_aie_53.c` `CHECK_SERVICE_0` guards KEPT; v4l2-ctrls core part REVERTED by `cde9d859` (camera-open panic, see §14) |
| `8f3defb9` | **mali IPA clock fix** (XagaForge `e1129586`): consistent clock for IPA timestamps — fixes GPU IPA util accounting |
| `cde9d859` | **camera panic fix: revert v4l2 per-frame alloc** — `v4l2-ctrls.c` back to no-op `request_complete` on control-less requests; AIE guards kept; `v4l2-ctrls.o` + `mtk_aie_53.o` compile clean (builder clang) |
| `870efaea` | **5.10.270 stable merge** (merged from `16.2-rebase`; 686 files, 20 conflicts — see §10 for resolutions; inotify half-merge later fixed by `a1ef0f5b`) |
| — | **A17 boot verified** via `Angxddeep/...:seventeen` @ 5.10.264 booting A17 on xaga — our 5.10.270 superset therefore A17-ready (no extra patch needed; see §16.1) |
| `d5843e1f` | **Bypass charging** (xaga, `mediatek/mtk_charger.c`): `bypass_charging` flag in `struct mtk_charger` + `USB_PROP_BYPASS_CHARGING` sysfs (`/sys/class/power_supply/usb/bypass_charging` RW 0/1, default 0, off at boot) + `cmd_discharging_get_flag()` export; `charger_check_status()` forces `charging=false` on either flag; `pd_cp_manager`/`pd_single_cp_manager`/`qc_cp_manager` SM_HOLD on either (covers SimpleBypassChargerMTK `current_cmd 0 1` path — pumps ran 9V/~4A straight through module bypass before this). On-device verified (CODM + 67W): BMS 0mA, VBUS 4.9V/650–940mA in, cap frozen. `plato/` untouched |
| `861ef240` | **Bypass v4 + PD type fix** (xaga): (a) `mtk_basic_charger.c:641` now gates FCC==0 CHG-off on `bypass_charging`/`cmd_discharging` so `do_algorithm` and `charger_check_status` agree — fixes 7000mA↔0 oscillation; (b) `mtk_charger_intf.c:get_charger_type()` now overrides BC12 CDP to `USB_PD` when `pd_verify_done` + `PE_READY_SNK*`, syncs `chr_type`/`usb_desc.type` so `/sys/class/power_supply/usb/type` reports `USB_PD` (was stale CDP while `pd_authentication=1`) |
| `a0a4da23` | **5.10.271 stable merge** (from `16.2-rebase`; 582 files, 589 upstream commits, 1 conflict: `net/xfrm/xfrm_user.c` kept GKI variant — discarded MTIMER_THRESH + SETDEFAULT/GETDEFAULT helpers, kept extack hardening + `xfrm_net_nlsk`; 3 files deleted upstream: sun8i-ce/ss-prng, asoc-ux500-msp.h) |
| `e455e426` | **271 build fix**: `xfrm_net_nlsk()` `sk` → `__maybe_unused` (lockdep condition compiles out without PROVE_LOCKING → clang 22 `-Werror=unused-variable`); helper kept (7 auto-merged call sites) |

### 2.1 Known-in-tree-but-inert features
- **NoMount**: dentry-op hooks only attach to dentries with registered rules; zero rules
  (default) = dormant. Userspace = NoMount metamodule (KSU/APatch) or `nm` CLI via keyring.
- **KSM**: `CONFIG_KSM=y` but `ksm/run=0` at boot; enable via `/sys/kernel/mm/ksm/run`.
- **Droidspaces**: config support only (in LXC variant via lxc_support.patch).
- **DriveDroid**: already supported by GKI (`USB_F_MASS_STORAGE=y` builtin).

### 2.2 What was evaluated and REJECTED (do not re-add)
- `sched_rt_runtime_us` bump to 980000: RT throttling was a symptom, not root cause;
  weakening the safety valve risks hard hangs. Runtime sysctl if ever needed.
- EAS energy-model hand-tuning: no in-tree table for MT6895 (MediaTek EAS supplies
  out-of-tree); hand-tuning without profiling is dangerous. Only safe step: distinct
  capacity-dmips-mhz for CPU7 in DTS (not done, not needed for governor-level work).
- Templar adios 3.2.2 / ssg tuning: our adios 3.2.0 already more battery-aggressive.
- NoMount v1.1.1 (Templar): genl-based, breaks keyring metamodule userspace compat.
- NTSYNC: driver absent in their tree too (dead defconfig option).
- BBR min_rtt window 7s: Templar reverted it themselves (cellular PROBE_RTT churn) — keep 10s.

---

## 3. ESK GOVERNOR — CURRENT STATE & NEXT STEPS

File: `drivers/cpufreq/cpufreq_esk.c` (~2270 lines) — **v2.2** as of `088c0f238f1e`
(see §3.1). Governor name: `esk`. Boot pr_info: `ESK Governor v2.2 by Templar Dev`.
Structural facts (MUST KNOW):
- **MT6895 uses `mediatek,cpufreq-hw` → `fast_switch_possible=true`** → ESK kthreads are
  NEVER created (`esk_kthread_create()` early-returns); **all commits run inline in the
  scheduler hook with IRQs disabled** via `cpufreq_driver_fast_switch()` (just a
  `writel_relaxed` to HW). Templar's Poco F5 (Qualcomm) uses the kthread path —
  their governor was never exercised on the inline path. ALL ESK bugs so far were
  inline-path bugs. v2.2's hook is **single-lock, zero early-returns inside it**
  (work carried out on flags) — the old §3 "every early-exit MUST unlock" rule is
  moot by construction, but AUDIT anyway after touching this file: any new early
  return between `raw_spin_lock_irqsave(&p->update_lock, ...)` and its unlock is a
  boot-unlock AP_WDT.
- sysfs: `gaming_mode` (prime policy), `prime_gaming_floor_pct` (NEW), `temp_mc`,
  `thermal_zone`, per-cluster `rate_limit_us`/`up/down_rate_limit_us`.
- Tier classification (25462d21): little = cap≤614; prime = cap≥1000 AND policy owns
  topmost CPU (`esk_policy_is_prime`); big = the rest. Two-tier SoCs: top policy=prime,
  big tier unused (preserves 2-cluster behavior).
- Governor must be enabled+default via `vendor/xiaomi_mt6895.config`
  (`CONFIG_CPU_FREQ_GOV_ESK=y`, `CONFIG_CPU_FREQ_DEFAULT_GOV_ESK=y`). **Currently
  default=schedutil, ESK=opt-in** (revert happened after freeze reports; crashes since
  fixed). Making ESK default again = pending user validation.
- ESK renames: vorpal→esk, rfx_→esk_ everywhere. Original author credit
  (Templar Dev / Steambot12) preserved in header. Upstream repo:
  https://github.com/Steambot12/Templar-Kernel-GKI-5.10 and
  https://github.com/Steambot12/Governor-Config (their tunables/logs repo).

### 3.1 DONE: Vorpal/ESK v2.2 port (was the major task — completed 088c0f238f1e)
Templar's final state is **Linux6-Staging head** (`f3efbafdbbc2`, Sep 4 2026), NOT
the Linux3-Staging snapshot this section was originally written from. It folds:
- **v2.2 upgrade** (`3061f2d`): gaming band (Prime 64/92, Big 58/90, Little
  45/66), EMA divisor 100 + step cap 32, headroom GAMING 2 / sat-to-max 98,
  adaptive warmup (300ms base / 600ms max / feeds the frame-boost ramp),
  hysteretic thermal cool-down (enter 80 / exit 85, steady floor 52, boost
  floor 72), floor gate deadband (25/35, `floor_gated`), sustained-lock
  REMOVED, frame-boost window 33ms + ramp 60ms with consumed-time accounting,
  daily Big/Prime caps (70/68 base, 80 boost, 80/80 sustained)
- **Linux4 fold** (`3c9d17ed`): is_top/is_prime split, **fceil folds
  policy->max via `max_seen` high-water** (MTK has no cpufreq_cooling —
  policy->max IS the throttle channel), sustained latch edges 80/68,
  cold-start gated on `esk_input_active()`, **gaming_mode user-owned** (PM
  suspend notifier removed), frame-boost window 33→25ms
- **Linux5** (`182ff823`): EMA advances its reference by consumed periods
  (sub-period remainder kept), `esk_elapsed()` clamps rq_clock skew across a
  shared policy (u64 wrap → ~584y → every window fired instantly)
- **Linux6** (`f3efbafd`): sustained caps == boost caps (background work no
  longer outranks interaction); irq_work/work_lock unconditional init (= our
  `91d0b1ff`, adopted upstream — they cite our fork's crash history)

What we kept from our fork (deliberate divergences):
- **Prime = policy owning topmost CPU** (`esk_policy_is_prime`). Templar's
  `ntiers()` counts distinct capacities — on MT6895 Big and Prime both report
  cap 1024, so their rule would move CPU7 into the BIG band and lose the
  PRIME band entirely.
- **`prime_gaming_floor_pct` sysfs** (now on the dynamic prime-cluster attr
  group alongside gaming_mode; default 64 = new macro value).
- GKI510 util helpers; new `esk_setattr_sugov_gki510()` in
  `kernel/sched/cpufreq_schedutil.c` + decl in `include/linux/sched/cpufreq.h`
  (SUGOV DL class for the DVFS worker; latent on MTK, correct elsewhere).

What we DROPPED from our fork (deliberately):
- skippability probe (`ecfce530`), limits 1ms gate + `pending_clamp` fast
  path (`47b9606d`), `b1d029ca`'s early-return unlock discipline (moot: v2.2
  hook is single-lock, zero returns inside it) — upstream's design is bounded
  by construction. Our tuning constant tweaks (a0810861-era) replaced by
  v2.2's measured-stable values.

Sysfs surface unchanged for users: `gaming_mode` + `prime_gaming_floor_pct`
still on policy7 only; rate limits on all three.

### 3.2 Known remaining perf items (from user testing + PUBG SF-latency data)
- PUBG SF captures: avg FPS stable (~89) but 1% low/min FPS and jank vary run-to-run;
  CPU7 oscillates 300↔2850MHz; fixes shipped: EEVDF yield, BORE reweight deadline,
  prime-selective boost. NOT yet verified on device — test next build.
- `surfaceflinger`/`RenderThread` RT prioritization: deliberately NOT implemented
  (no blanket RT — no watchdog safety in tree; existing uclamp/top-app path preferred).
  If pursued: SCHED_RR prio 1-5 max, runtime-quota safeguard, name-matched via cgroup
  attach, userspace wiring required.
- EAS energy model: only safe step = distinct `capacity-dmips-mhz` for CPU7 in DTS
  (currently 1024 same as big). Prerequisite for any EAS-level prime awareness.
  Do NOT hand-tune power tables.

---

## 4. RELEASE WORKFLOW (releases repo)

Repo: `ESK-Kernel-Reborn-Releases` (branch `main`), files: `README.md` (variants +
supported devices xaga/xagapro/xagain table + versioning), `INSTALL.md` (AOSP-recovery
Apply-update→Apply-from-ADB flow only — no TWRP exists for this device; no backup
section per user request).

Release naming: tag `0.X`, title `ESK Reborn 0.X [Beta N]`. Current: **0.3 Beta 2
(prerelease)** with asset `ESK-Reborn-5.10.269-KSU-SUSFS-LXC-25176a53e9b8-AnyKernel3.zip`.
Tag renames: use `gh release edit <tag> --tag <newtag>`; **watch for orphaned git tag
refs** (delete via `gh api repos/.../git/refs/tags/<name> -X DELETE`). Prerelease
cannot be marked `--latest`.

Release notes style: markdown, device table, base/kernel info, Builds table
(file | features | source commit), Changelog grouped (Stable base / Scheduler & CPU /
Crash fixes / Memory & UFS / Networking / Containers / Power), install link.
Add "variants coming soon" note when partial variants upload. Verify EVERY upload with
`gh release view <tag> --json assets` and verify the zip's Image version string:
`unzip -p zip Image.zst | zstd -d | strings | grep "Linux version"` — a stale zip was
almost shipped once; also verify the commit hash inside matches the intended build.

User workflow: user builds variants locally, drops zips in `/home/akali/ESK_Reborn/<ver>_ENGRahsepar/`
folders, asks for upload. **ALWAYS verify zip's Image version string matches the intended
commit BEFORE uploading.**

Commit message format (Android Common Kernel rules):
- `ANDROID:` for vendor/device-specific, `BACKPORT:` for stable/mainline backports
- Body: what/why, Link: to source repos, `Fixes:` tag when fixing a prior commit
- `Change-Id: I<40-hex>` (generate: `I$(echo <unique> | sha1sum | cut -c1-40)`)
- `Signed-off-by: akaliorg <alirahsepar199@gmail.com>`

---

## 5. DEVICE-SPECIFIC FACTS (learned the hard way)

- **UFS health is poor** (user report): ZRAM writeback must stay OFF (removed in
  `5a64ca47`); KSM added as flash-free alternative.
- **Battery is aged**: 1783 cycles; sags to 3.6-3.9V under load at ~20% SOC.
  Charging + gaming = thermal_level up to 15, charge FCC cut to 500mA.
  Game freezes originally correlated with charging+gaming.
- **Freeze/crash history (all root-caused & fixed):**
  1. Boot-unlock/warm-switch AP_WDT → ESK inline-path spinlock double-holds (`b1d029ca`)
  2. Limits livelock (thermal min/max churn × full walk) (`47b9606d`)
  3. Game freezes + app kills + soft reboot, both governors → **le9uo reclaim livelock**
     (`clean_min_ratio=25%` floor) (`25176a53`) + **binder/cgroup-freezer mmap_sem wedge**
     (`527111b4`) — found via hung_task dumps (`sysctl kernel.hung_task_timeout_secs=10`)
  4. First 0.3 build "worked" by luck — races are timing-dependent.
- **UI lags first reported on ESK v1** → v2 retune + PELT16 + yield/reweight fixes address it.
- MT6895 GPU = Mali-Valhall + **GED/gpufreq stack** (GPUEB firmware owns GPU DVFS via IPI;
  kbase devfreq uses MTK "dummy" governor). **Never put simple_ondemand on the kbase
  devfreq** — two controllers fight. GPU errors (`gpufreq_fix_target_oppidx fail`) come
  from userspace writing debug procfs; GPUEB rejects under thermal guard. Harmless-ish.
- MT6895 CPU freq = `mediatek,cpufreq-hw` with `fast_switch_possible=true` (3 domains).
- Phone used on OmniROM(A16-style) w/ BakaSU (renamed from ReSukiSU Oct 2026)+SUSFS; Roblox = the repro game.

---

## 6. TESTING CHECKLIST FOR EVERY NEW BUILD

1. `uname -r` shows expected version + commit sha
2. Boot → **unlock immediately** (historical crash trigger), repeat 3-4×
3. Heavy Roblox game **while charging** (thermal storm trigger), 10+ min
4. Background app switching while warm
5. Check `dmesg` for: RT throttling, SPM 0%-idle lines, audit flood rate, esk errors
6. If governor=esk: verify `gaming_mode` node exists at
   `/sys/devices/system/cpu/cpufreq/policy7/esk/gaming_mode` (prime policy),
   tune `prime_gaming_floor_pct`
7. sysctl sanity: `vm.anon_min_ratio` etc = 0; `kernel.sched_bore=1`;
   `net.ipv4.tcp_congestion_control=bbr` (bbrplus selectable)
8. 5+ hour daily use before promoting a prerelease to stable

---

## 7. UPSTREAM SOURCES TO WATCH

- **Steambot12/Templar-Kernel-GKI-5.10** (branches: Templar-LinuxStable = main,
  Linux1-7-Staging = newer governor work, Templar-MGLRU = unexplored, GoogleLTS-Staging,
  Templar-OSS, Templar-RC). **Governor-Config** repo = their defconfig + governor files + LOG.
- **maxsteeel/nomount** (dev branch; we use keyring version, v1.1.1 = genl — incompatible)
- **ravindu644/Droidspaces-OSS** (container runtime; requirements = our LXC config set)
- **simpunk/susfs4ksu** branch `gki-android12-5.10` (builder pulls it)
- **UJX6N/bbrplus-5.10** (bbrplus source)
- **kernel.org stable** (v5.10.x — rebase when new sublevel lands; procedure: fetch tag,
  `git merge --no-commit v5.10.26X`, resolve conflicts keeping vendor blocks, verify
  BORE/EEVDF KABI block in sched.h survives, commit `Update to Linux 5.10.26X`)
- **google/bbr** v3 branch (BBRv3 upstream; our in-tree bbr IS BBRv3 already,
  `BBR_VERSION=3`, from the GCE android12-5.10 tree)
- **MGLRU**: NOT in 5.10 tree; Templar has `Templar-MGLRU` branch — big future item,
  requires serious port work. Highest-impact remaining feature.

---

## 8. SESSION HANDOFF — DO THIS FIRST IN A NEW SESSION

1. Read this file fully.
2. `git -C <repo> log --oneline -30` and compare against §2 table — confirm no drift.
3. Check `git log origin/17.0 -1` AND `git log origin/16.2-rebase -1` match local (push verification habit — both branches).
4. Check kernel.org for new 5.10.x; check Templar branches for governor updates
   (v2.2 port is queued — §3.1).
5. Ask the user what variant they built and what they tested; pull fresh logs if issues.
6. Commit format: §4. Build/verify: §1.2. Debugging: §1.3.
7. **Communication style with user**: casual, direct, technical; user is hands-on
   (builds/tests locally, provides logs); explain root causes with evidence from THEIR
   logs; never push unverified fixes; always compile-test with their clang before push;
   always verify pushes AND uploaded zips.

---

## 9. DOCUMENTATION MAINTENANCE RULE (MANDATORY)

> **EVERY functional change to this kernel (new feature, fix, revert, config change,
> stable rebase, governor tuning) MUST be followed by an update to THIS file
> (`DEVELOPMENT.md`)** — appended to §10 (Full Change History) and, where relevant,
> to §2 (feature inventory), §3 (governor state), §5 (device facts) or §7 (upstream).
> The release-notes body in the releases repo must also be updated. This document is
> the single source of truth for session handoff; if it drifts from the tree, the next
> session starts from wrong assumptions. Same rule applies to the releases-repo
> `README.md`/`INSTALL.md` when user-facing behavior changes.

---

## 10. FULL CHANGE HISTORY (chronological, complete)

### Pre-ESK era (base tree)
- `0af7cf4f` / `6ca89a93` / `0223d21b` — MT6895 DTSI imports (xaga + plato), DTBO build support, `fenrir=true`
- `dd3b1030` — **Update to Linux 5.10.266** (= 0.2 release base)

### 0.2 → 0.3 development (current line, all on 16.2-rebase)
| Commit | Change |
|---|---|
| `f8e3dbe2` | imgsensor frame_sync_console: removed dead 4KB str_bufs — fixed the only build warning (stack frame 4144 > 4096 in `fsync_console_store`) |
| `221f9dae` | Merge v5.10.267 (58 commits; binder pin dedup, isotp rewrite, xfrm sk_dst_reset; huge_memory flags fix folded in) |
| `df46125d` | Merge v5.10.268 (inet frags: skb_gso_reset before reassembly) |
| `03e2cd2b` | huge_memory: `unsigned long flags = 0` in split_huge_page_to_list (vendor path uses local_irq_disable, not lru_lock) |
| `3680ab2b` | **NoMount VFS redirection integrated** (`fs/nomount/`, CONFIG_NOMOUNT=y; source maxsteeel/nomount dev branch, keyring API) + vendor fragment: NETFILTER_ADVANCED, IP_SET family, XT_TARGET_REJECT/LOG, MATCH_RECENT/COMMENT/HL/PKTTYPE/OWNER, TMPFS_POSIX_ACL, WQ_POWER_EFFICIENT_DEFAULT |
| `2256037a` | **EEVDF (6.6 backport) + BORE hybrid** — BORE base → EEVDF → BORE 6.6.3 tuning (Templar commits; features PLACE_LAG/PLACE_DEADLINE_INITIAL; avg_vruntime; SCHED_BORE sysctls incl. sched_burst_fork_atavistic default 0) |
| `7760bd3f` | **BBRplus** — tcp_bbrplus.c (UJX6N 5.10 port + Templar 3a4802809c stability/perf tuning) + `bbrplus_tso_segs()` shim for 5.10 tso_segs(sk,mss) API |
| `3092c431` | **le9uo** working-set protection (sysctls vm.anon_min_ratio / vm.clean_low_ratio / vm.clean_min_ratio; mm/Kconfig ANON_MIN_RATIO etc.) |
| `db442051` | **ESK governor** (Vorpal v2.1 renamed): dual-profile gaming/daily, tri-cluster bands, EMA smoothing, frame-risk detection, touch boost, thermal emergency net; helpers in cpufreq_schedutil.c (esk_get_util_gki510 via schedutil_cpu_util = uclamp/RT/DL/IRQ aware); CONFIG_CPU_FREQ_GOV_ESK + DEFAULT |
| `7f86a52f` | CONFIG_BRIDGE_NETFILTER=y |
| `9a8402c7` | ESK v2 daily retune (little cap 72/84, UI floor 40, burst floors 48/45, ramp delta 8, UI window 400ms, coldstart 300ms, rate gates) — later softened in a0810861 (coldstart 200, floors 45/42, delta 10, little rate 2000) |
| `b55531ca` | **5.10.269 stable merge** (clean; USB UAF fixes, ext4 fast-commit, kcov preempt rework, IPVS/xfrm hardening) |
| `fe6ff006` | ZRAM_WRITEBACK=y (later reverted) + WQ_POWER restructure |
| `5a64ca47` | **ZRAM_WRITEBACK removed** (UFS health), **CONFIG_KSM=y** (off at boot, /sys/kernel/mm/ksm/run) |
| `eacd4cc4` | LOCALVERSION → `-ESK-Reborn_V0.3` |
| `a0810861` | **Default governor → schedutil** (freeze reports; ESK opt-in); ESK boot-storm knobs softened |
| `ecfce530` | ESK skippability probe (idle re-eval skip) — **introduced skip-path lock bug (fixed in b1d029ca)** |
| `1bf9ef41` | BBRv3 mobile tuning: pacing margin 2%, PROBE_RTT 140ms, probe base 1.9s; min_rtt 10s kept (Templar cellular fix) |
| `87c22842` | **PELT half-life 16ms** (runtime-switchable pelt.c + tables; Kconfig choice) + RCU_BOOST/delay 15 + IDLE_PAGE_TRACKING |
| `91d0b1ff` | ESK irq_work/work_lock unconditional init |
| `47b9606d` | limits_changed 1ms gate + clamp fast path (**introduced clamp-path lock bug, fixed in b1d029ca**) |
| `b1d029ca` | **CRITICAL: two early-returns inside update_lock fixed** (clamp + skippable paths) — the AP_WDT boot-unlock/warm-switch crash root cause |
| `db15178e` | PELT LOAD_AVG_PERIOD/MAX compile-time constants (fixed half-life); LOAD_AVG_MAX scope Werror fix |
| `527111b4` | **binder freeze wedge fix**: binder_install_single_page → mmap_write_lock_killable (frozen app holding mmap_sem read blocked all binder txns 10-20s → game freezes, app kills) |
| `25176a53` | **le9uo ratios 0/0/0** (root cause of game-specific freezes: clean_min 25% floor → reclaim zero-progress livelock under gaming+charging pressure; sysctls runtime-tunable) |
| `90bac8e9` | **EEVDF-functional sched_yield** (skip-buddy removed; deadline refresh for entitled entities) + **BORE reweight_task_by_prio deadline refresh** (fwd decls sched_slice/calc_delta_fair) |
| `25462d21` | **ESK three-tier classification** (prime = topmost-CPU policy) + **prime_gaming_floor_pct sysfs** (default 70) + big-tier gaming release at demand<40% |
| `5e085648` / `650390c8` / `189032c2` | DEVELOPMENT.md (this file) |
| `088c0f23` | **ESK governor → v2.2** — full port of Vorpal Linux6-Staging end state (§3.1 has the complete feature delta + keep/drop decisions). Files: `drivers/cpufreq/cpufreq_esk.c` (wholesale replace + grafts), `kernel/sched/cpufreq_schedutil.c` (+`esk_setattr_sugov_gki510`), `include/linux/sched/cpufreq.h` (+decl). Compile-tested clean. **Needs on-device validation (gaming + daily) before any release** |
| `779b6e85` | **EEVDF rescale on reweight** (Templar 77672e88, mainline eab03c23) — see §2 |
| `659dc646` | **BORE weight fix + fork_atavistic clamp** (Templar 2a8e879f) — see §2 |
| `9b9ad306` | **sched_yield EEVDF fix** (Templar 8a9affcf) — replaces 90bac8e9 variant |
| `3f5a3f13` / `cf88c2d8` | **le9uo reclaim fix + default 5** (Templar `b21d737` + `10d579`): file-reclaim no longer blocked by `anon_below_min` on swapless, `clean_low_ratio` default 15→5 |
| `ed9ed6f3` | **media: mtk-aie CID guard** (XagaForge `f89055e3` + `23f18ca4`): `mtk_aie_53.c` guards kept, v4l2 core part later REVERTED by `cde9d859` (camera panic) |
| `8f3defb9` | **mali IPA clock fix** (XagaForge `e1129586`): consistent clock for IPA timestamps — fixes GPU IPA util accounting |
| `cde9d859` | **REVERT v4l2 core part of ed9ed6f3 (camera-open panic fix)** — `request_complete` no-op again on control-less requests; AIE guards kept; `v4l2-ctrls.o` + `mtk_aie_53.o` compile clean; see §14 |
| — | **builder infrastructure fix (2026-09-13): SuSFS 769e31f statfs build break** — `fs/statfs.c` `extern susfs_sus_kstat_spoof_vfs_statfs` declared after first use in `susfs_statfs_by_dentry()` → clang 22 `-Werror=implicit-function-declaration` + `make[2]: fs/statfs.o Error 1` → `__sub-make Error 2` on every KSU+SUSFS variant; root cause upstream `susfs4ksu` `769e31f` patch ordering. Fixed in `~/esk_builder/build/setup.sh:apply_susfs()` (awk injects early externs before `susfs_statfs_by_dentry`; idempotent guard) + patched live `~/esk_builder/kernel/fs/statfs.c`; verified `fs/statfs.o`, `fs/*`, `drivers/kernelsu/built-in.a` compile clean (§1.2, `DEVELOPMENT.md:1.1/14`) |
| `870efaea` | **5.10.270 stable merge** (from `16.2-rebase`) — 686 files / 792 upstream commits, 20 conflicted files. Took upstream: schedutil refactor core (sg_cpu util/max, void getters), platform driver-core reorg (old blocks deleted, GKI cast kept), xhci bounce-buffer fix (sysdev), irqdomain_info/instantiate (in KABI guard), KMAP_LOCAL (+DAMON kept), sunrpc threadless-pool fallback, inet_csk_prepare out-of-line, rproc_detach decl. Kept HEAD: GKI schedutil up/down variant (dropped uncompilable single-rate helper), no-busy-check, remoteproc core + adapted 2× RPROC_DELETED→deleting flag, nfsd/lockd/bpf-cgroup (dead/no-callers), fsnotify (4-part unit reverted for coherence), u_audio (UAF fix inapplicable to GKI layout). KABI reserves 1-4 intact, SUBLEVEL 270. Compile clean (builder clang): schedutil, esk, fair, memcontrol, vmscan, platform, xhci-ring, irqdomain, inet_connection_sock, v4l2-ctrls, remoteproc_core, cgroup, bbr, bbrplus |
| `a1ef0f5b` | **inotify half-merge fix** — restored GKI `inotify_update_existing_watch` (270 had taken 1/4 of upstream `27d172b60eec`, breaking the full build); §1.2 corrected to srctree-only sync after stale `work/` shadows caused a bogus selinux `context.h` failure; see §14 |
| `8ddce89a` | **inet_csk link fix** — upstream `5d5389b2c37c`'s `this_cpu_inc(*orphan_count)` cannot link on 5.10 (`struct percpu_counter *`, fallback symbol undefined); restored `percpu_counter_inc`, kept upstream TCP cb-flags clear |
| `d5843e1f` | **Bypass charging final** (xaga `mediatek/` only): history `470eaf2d` (v1 flag+sysfs+pump gates) → `1b01738` (v2 one-shot FCC, failed: do_algorithm overwrote → +843mA) → `bc1775f` (v3 persistent FCC clamp, REVERTED: MT6375 F_CC floors at 300mA in silicon `val_toreg`, trickle not idle) → `d5843e1` (v1 semantics + pumps gated on `bypass_charging` AND `cmd_discharging` via new export; `mtk_basic_charger:641` untouched). Root causes found on-device: (a) 67W pumps bypassed module `0 1` (PDM TUNE, 9V/~4A); (b) v1-on-CDP drain was power budget (575mA in vs gaming load), not a bug; (c) "slow charge" report was PC-USB source (CDP, no PD) — 67W gives 8.6V/3A+. Verified: `bypass_charging=1` and `current_cmd 0 1` both → BMS 0mA, `PDM_SM_HOLD`, VBUS ~4.9V. Compile: mtk_charger.o/mtk_basic_charger.o/pd_cp_manager.o |
| `861ef240` | **Bypass v4 + PD type fix** (xaga): (a) `mtk_basic_charger.c:641` now gates `FCC==0` CHG-off on `bypass_charging`/`cmd_discharging` so `do_algorithm` and `charger_check_status` agree — fixes `7000mA↔0` oscillation; (b) `mtk_charger_intf.c:get_charger_type()` overrides BC12 `CDP` to `USB_PD` when `pd_verify_done` + `PE_READY_SNK*`, syncs `chr_type`/`usb_desc.type` so `usb/type` reports `USB_PD` (was stale `CDP` while `pd_authentication=1`) |
| `ff8863c1` | **Bypass final design** (xaga, pre-build deep audit of `c79cc77`): loop forced CHG on + ICHG 0, but silicon floors F_CC at 300mA → trickle; nothing restored thermals on disable → FCC stuck 0. Final: CHG forced OFF in `do_algorithm` 641-gate (both flags, steady, no flap); `can_charging=true` so AICR recomputes live (plug-out leaves 100mA residue `plug_out:2802`); FCC clamp 0 stops PE/HV (`basic:531/580`); PDM HOLD leaves 5V/3A (no starvation); thermal FCC/AICR saved + re-clamped mid-bypass, restored on disable (`bypass_has_saved`); notify/vbat_mon preserved in-branch; plug-out clears `charge_full` (self-heals 100% edge). Compile: 4 objects clean |
| `a0a4da23` | **5.10.271 stable merge** (from `16.2-rebase`, merged into `17.0` as `7590bfa5`): 582 files / 589 upstream commits, 1 conflict (`net/xfrm/xfrm_user.c` — kept GKI variant, see §2). Upstream highlights: cpufreq `zalloc_cpumask` + rwsem init order, xfrm verify extack strings + `xfrm_net_nlsk` helper, Bluetooth L2CAP/RFCOMM + NFC llcp hardening. Compile clean (builder clang): xfrm_user, cpufreq, schedutil, esk, fair, memcontrol, vmscan, platform, xhci-ring, irqdomain, inet_connection_sock, v4l2-ctrls, remoteproc_core, cgroup, bbr, bbrplus |
| `e455e426` | **271 build fix** (`net/xfrm/xfrm_user.c`): `xfrm_net_nlsk()` `sk` → `__maybe_unused` — carried into `17.0` by `7590bfa5` (single merge; earlier two-merge layout superseded by sign-off rewrite) |
| — | **Builder: KernelSU source `ReSukiSU/ReSukiSU` → `Baka-SU/BakaSU`** (upstream rename Oct 2026; same `kernel/setup.sh <ref>` interface, same `drivers/kernelsu` symlink + `CONFIG_KSU`/`CONFIG_KSU_SUSFS` symbols, SUSFS menu intact — builder flow unchanged; `build/setup.sh` + builder README updated, §1.1/§11 synced). Next KSU-variant build exercises it live |
| `157418db` | **xaga: ARM64_SW_TTBR0_PAN** in `vendor/xaga.config` (A17 VINTF FCM level 7 requirement). Pushed by user direct to `17.0` (17.0-only, NOT on `16.2-rebase`) |
| `15e0e52d` | **mtk-mml video playback + in-kernel scaling fix** (Angaddeep Singh, via `angx` upstream): `-EBUSY` fallback to initial config on `dup_task`; new `mtk-mml-rsz-fw.c/h` (in-kernel resizer firmware calc, ported from Nothing MT6886 5.15). Pushed by user direct to `17.0` (17.0-only). Mml objects compile-verified (builder clang) |
| `2989dbe0` | **Merge origin/17.0 into 17.0** — takes the 2 commits above on top of 271 merge `7590bfa5`; clean auto-merge. Both branches pushed + verified Oct 2026 |

### Release history
| Release | Tag | Build commit | Notes |
|---|---|---|---|
| 0.1 | 0.1 | 5.10.266 (dd3b1030/f8e3dbe2 era) | First public; VNL/KSU-SUSFS/KSU-SUSFS-LXC |
| 0.2 | 0.2 | 5.10.268 (03e2cd2b) | 5.10.267/268 merges, huge_memory fix |
| 0.3 Beta 1 | 0.3 (renamed) | 5.10.268→269 era | **WITHDRAWN** — freezes (ESK deadlocks, later binder+le9uo) |
| 0.3 | 0.3 | `e4428edd3ad6` | 5.10.270, all crash fixes, ESK v2.2 opt-in, bypass charging; KSU-SUSFS-LXC + modules live, VNL/KSU-SUSFS pending |
| 0.3.1 | 0.3.1 | 5.10.271 (conntrack 271 fix era) | Localversion bump V0.3 → V0.3.1 on both branches; 5.10.272 merge prepared |
| **0.3.2 (current)** | 0.3.2 | 5.10.272 (3c27474e merge) | Localversion bump V0.3.1 → V0.3.2; tree source (5.10.272) newer than 0.3's 5.10.270 release base |
| 0.4 (planned) | — | — | Vorpal v2.2 port + ESK default + fps/battery tuning |
| 0.3.1/0.4 (planned) | — | — | Vorpal v2.2 port + ESK default + fps/battery tuning |

---

## 11. UPSTREAM LINKS (complete reference list)

### Kernel sources
- Kernel source repo: https://github.com/ENGRahsepar/android_kernel_xiaomi_mt6895 (branch 16.2-rebase)
- Releases repo: https://github.com/ENGRahsepar/ESK-Kernel-Reborn-Releases (branch main)
- kernel.org stable: https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git (tag v5.10.271 = current base; `git ls-remote ... "refs/tags/v5.10.*" | sort -V` to check newest)
- AOSP common: https://android.googlesource.com/kernel/common (branch android12-5.10 / android12-5.10-lts)
- Builder reference (closed, on user PC): `~/esk_builder/` (see §1.1)

### Feature sources (all vetted this project)
- Templar kernel (primary feature donor — scheduler/governor/net/mm): https://github.com/Steambot12/Templar-Kernel-GKI-5.10
  - Branches: `Templar-LinuxStable` (main), `Linux1/2/3-Staging` (Vorpal v2.2 lives here), `GoogleLTS-Staging`, `Templar-MGLRU` (future work), `Templar-NoBore`, `Templar-OSS`, `Templar-RC`, `LinuxStable-Pure`, `Linux-RC`
  - Key commits: EEVDF `8c0ad4c4`, BORE `3e7145c6`/`bb8bac7d`/`e37771cf`/`f65f0962`, Vorpal v2.1 `d89c1ed206` / v1.0 `2942ce8e12`, BBRplus `5f811acb4b`→`3a4802809c`, le9uo `c0762abeb0`+`6476d4d5c8`, NoMount v1.1.1 `3be84ff962`, Droidspaces IPC `ba729331bf`, BBRv3 tune `77964c8e3c` + min_rtt revert `f94513b439`, adios `2b0f833dd7`, ntsync `096c2a4d4e`
- Templar governor/config extras: https://github.com/Steambot12/Governor-Config (cpufreq_vorpal.c v2.0, cpufreq_reflex.c v7.7-BORE-UNIVERSAL-GAMING, cpufreq_schedutil.c, fair.c, prefer_silver.c, drm_vblank.c, LOG, GKI defconfig)
- NoMount: https://github.com/maxsteeel/nomount (we use dev-branch keyring version; kernel/README.md has integration methods; metamodule expects keyring API)
- Droidspaces: https://github.com/ravindu644/Droidspaces-OSS (Kernel-Configuration.md = GKI config guide; SUSFS "hide sus mounts for all processes" must be OFF for Droidspaces)
- BBRplus 5.10: https://github.com/UJX6N/bbrplus-5.10 (convert_official_linux-5.10.x_src_to_bbrplus.patch)
- google/bbr: https://github.com/google/bbr (v3 branch; in-tree bbr is already v3)
- SuSFS: https://gitlab.com/simonpunk/susfs4ksu (branch gki-android12-5.10; builder pulls it)
- BakaSU (renamed from ReSukiSU Oct 2026): https://github.com/Baka-SU/BakaSU (builder installs on KSU variants; old `ReSukiSU/ReSukiSU` URL redirects)
- KernelSU (base): https://github.com/tiann/KernelSU
- AIK/AnyKernel3: https://github.com/osm0sis/AnyKernel3 (builder's AK3 template)
- Linux upstream EAS/EEVDF reference: EEVDF merged in v6.6 (`5f50b5a2` series by Peter Zijlstra)
- hakavlad le9 patch (le9uo origin): https://github.com/hakavlad/le9-patch (le9ec_patches/le9ec-5.10.patch)
- BORE scheduler origin: https://github.com/firelzrd/bore-scheduler (BORE by firelzrd; Templar carries the working 5.10 GKI port; hamadmarri has related scheduler repos)

### Device reference
- Device: POCO X4 GT (xaga) / RB: Redmi Note 11T Pro (xaga) / Pro+ (xagapro) / K50i (xagain) — models 22041216G/C/UC/I
- SoC: MT6895Z (Dimensity 8100): 4×A78 2.85GHz + 4×A55 2.0GHz; GPU Mali-G610 MC6 (Valhall, ged/gpufreq/GPUEB stack); UFS (health-poor unit), MT6375 charger IC, goodix/fpc fingerprint
- DTS: `arch/arm64/boot/dts/vendor/mediatek/mt6895.dts` + `xaga*.dts(i)`; 3 cpufreq performance-domains (0=CPU0-3, 1=CPU4-6, 2=CPU7)

---

## 12. GLOSSARY / PROJECT CONVENTIONS

- **ESK** = the project name (kernel = ESK Reborn). Governor name `esk` (renamed from Vorpal). Symbols `esk_*` (was `rfx_*` — Reflex origin in later Templar versions).
- **AP_WDT** = MTK Application-Processor hardware watchdog reset (full hang; cmdline `aee_aed.poffreason=AP_WDT`).
- **AEE** = MTK Android Exception Engine (userspace-facing crash/reboot handler; "main process crashed" reports).
- **wdtk** = MTK watchdog kicker kthread (logs `[wdtk] kick watchdog` every ~15.5s; its absence in logs = system wedged).
- **GED / GPUEB** = MTK GPU Enhanced Driver + GPU Embedded-Controller firmware (owns GPU DVFS via IPI; gpufreq_v2.c wrapper).
- **lmkd/AEE soft reboot** = userspace-triggered reboot after stalls (distinct from AP_WDT).
- **fast-switch path** = cpufreq policies with `fast_switch_possible=true` (mediatek,cpufreq-hw) where governor commits run inline in the scheduler hook with rq->lock held and IRQs disabled. ALL ESK crashes were inline-path bugs.
- **Vendor fragment** = `arch/arm64/configs/vendor/xiaomi_mt6895.config` (+ `xaga.config` device file) merged over `gki_defconfig`.
- **KABI reserves** = `ANDROID_KABI_RESERVE(n)` padding in task_struct; BORE uses 1-4, lxc_support.patch uses 6-8. New task_struct fields MUST use a free reserve or be appended past them.
- Commit style: `ANDROID:` / `BACKPORT:` prefixes, Change-Id + Signed-off-by (see §4).
- Push + verify (`git log origin/branch -1`). Upload + verify zip Image version. Update DEVELOPMENT.md every change (§9).

---

## 13. COMPLETE CPU POLICY / SYSFS MAP (MT6895)

```
policy0  = CPU0-3 (Little, A55, cap 380, perf-domain 0)
policy4  = CPU4-6 (Big, A78, cap 1024, perf-domain 1)
policy7  = CPU7   (Prime, A78, cap 1024, perf-domain 2 — SEPARATE DVFS domain)
```

ESK sysfs (only exists after `echo esk > .../scaling_governor`):
```
/sys/devices/system/cpu/cpufreq/policy0/esk/  → rate_limit_us, up_rate_limit_us, down_rate_limit_us
/sys/devices/system/cpu/cpufreq/policy4/esk/  → same (big tier)
/sys/devices/system/cpu/cpufreq/policy7/esk/  → same + gaming_mode, prime_gaming_floor_pct,
                                                temp_mc, thermal_zone (prime tier hosts gaming)
```

Key runtime sysctls:
```
kernel.sched_bore                       = 1     (BORE on)
kernel.sched_burst_fork_atavistic       = 0     (fork inherits fresh)
kernel.sched_burst_smoothness_long/short, sched_burst_penalty_offset/scale
vm.anon_min_ratio / vm.clean_low_ratio / vm.clean_min_ratio = 0/0/0 (le9uo OFF)
net.ipv4.tcp_congestion_control         = bbr   (bbrplus selectable)
kernel.hung_task_timeout_secs           = 10 during debugging (default 120)
/sys/kernel/mm/ksm/run                  = 0 (KSM opt-in)
```

---

## 14. PROBLEM→FIX HISTORY MATRIX (symptom → mechanism → commit)

| Symptom | Root cause | Fix commit | Status |
|---|---|---|---|
| Build warning stack frame 4144 | Dead 4KB str_bufs in imgsensor frame_sync_console | `f8e3dbe2` | fixed |
| huge_memory build error (flags) | 5.10.267 merge + vendor irq-path divergence | `03e2cd2b` | fixed |
| AP_WDT boot-unlock crash | ESK inline-path: clamp+skip paths returned holding update_lock IRQs-off | `b1d029ca` (+gates `47b9606d`) | fixed |
| Warm app-switch AP_WDT | Same mechanism (thermal limit churn re-entry) | `b1d029ca` | fixed |
| Game freezes ~20s, display off, SystemUI crash, app kills, soft reboot (both governors, charging makes worse, heavy Roblox only) | **TWO independent bugs:** (a) le9uo `clean_min_ratio=25%` floor → reclaim zero-progress livelock; (b) binder `mmap_write_lock` uninterruptible while cgroup-frozen app holds mmap_sem read in page fault | `25176a53` + `527111b4` | fixed |
| UI lags (ESK v1 era) | CFS skip-buddy no-op under EEVDF + stale BORE reweight deadlines + PELT32 latency | `90bac8e9` + `87c22842` | fixed |
| CPU7 300↔2850MHz oscillation (PUBG capture) | No prime/big tier split; sustained boosts pinned CPU7 | `25462d21` | fixed, needs on-device validation |
| EMA time constant stretched / latch releases late (upstream Linux5) | EMA discarded sub-period remainders; reference advanced before callee decided | `088c0f23` (v2.2 port) | fixed in tree, needs validation |
| rq_clock skew fired every timing window instantly (upstream Linux5) | u64 elapsed-time subtraction across sibling CPUs of shared policy wraps | `088c0f23` (`esk_elapsed()` clamp) | fixed in tree, needs validation |
| fceil blind to vendor thermal HAL on MTK (upstream Linux4) | Only thermal_pressure was read; cpufreq_cooling never registers on MTK | `088c0f23` (`max_seen` + policy->max fold) | fixed in tree, needs validation |
| Lock-order inversion, deferred commit path (upstream) | work_lock held across cpufreq_driver_target() which takes policy->rwsem held by ->limits() | `088c0f23` (`__cpufreq_driver_target`) | latent on MTK (fast-switch), fixed for other drivers |
| Thermal poller busy-rearm with no temp source (upstream Linux4) | Poll re-armed forever with neither temp source configured | `088c0f23` | fixed in tree |
| ZRAM writeback wear (UFS health) | Writeback writes cold pages to flash | `5a64ca47` (removed) | fixed |
| le9uo originally shipped active | Ratios too aggressive for 8GB gaming | `25176a53` (0/0/0) | fixed |
| Camera open → instant reboot/panic (first seen after `ed9ed6f3`, 2026-09-13) | Upstream `c3bf5129` backport made `v4l2_ctrl_request_complete()` kzalloc+bind a handler for every control-less request — runs per preview frame in vb2 hot path on a vendor 5.10.269 tree whose `media_request` core predates it (XagaForge hit the same panic at `7b4c52fd`) | `cde9d859` (revert v4l2 core to no-op, keep AIE `CHECK_SERVICE_0` guards) | fixed in tree, NEEDS on-device camera validation |
| Full build break after 270 merge (2026-09-14, `Image modules` link stage): (a) `inotify_user.o` undefined `INOTIFY_MARK_FLAGS` + implicit `inotify_arg_to_flags` — the merge took 1 of 4 parts of upstream `27d172b60eec`; (b) bogus `security/selinux/ss/services.c:56 'context.h' not found` — a stale `work/` shadow copy of services.c without its sibling (repo was fine; O= builds prefer `work/` copies, see §1.2); (c) vmlinux link `ld.lld: undefined __bad_size_call_parameter` from `inet_csk_prepare_for_destroy_sock` — upstream `5d5389b2c37c` uses `this_cpu_inc(*orphan_count)` but on 5.10 orphan_count is `struct percpu_counter *` (this_cpu only does 1/2/4/8-byte scalars; fallback symbol is extern-but-never-defined) | (a) half-merged upstream unit; (b) verification sync copied 686 merge files into `work/`; (c) broken upstream backport line | `a1ef0f5b` (restore GKI inotify) + deleted all 686 `work/` shadows + `8ddce89a` (`percpu_counter_inc`, keeps upstream cb-flags clear) | fixed in tree, user to rebuild |
| `xfrm_user.o` compile break after 271 merge (2026-10-07, object stage): `net/xfrm/xfrm_user.c:40 unused variable 'sk'` under clang 22 `-Werror` — new upstream `xfrm_net_nlsk()` helper's `sk` is only consumed by the `rcu_dereference_protected` lockdep condition, compiled out without PROVE_LOCKING | New upstream helper vs GKI config (helper itself kept: 7 auto-merged call sites) | `e455e426` (`sk` → `__maybe_unused`) | fixed in tree, compile-verified |
| Full `device` build break on 271 (2026-10-07, user build log): `nf_conntrack.h:178 incompatible pointer types (atomic_t* → refcount_t*)` in 6 TUs (flow_dissector, cls_api, nfnetlink_queue, nf_conntrack_core/_standalone, icmp), cascading into `drivers/misc/mediatek` Error 2 — upstream 271 added `nf_ct_shared()` with `refcount_read()`, but this tree kept the GKI `atomic_t ::use` variant (an EARLIER stable did the refcount conversion; our prior merges kept HEAD) | Same vendor-divergence class as 270 inet_csk + 271 xfrm | `a0b90dba` (`refcount_read` → `atomic_read`, GKI struct kept) | fixed in tree, 6 objects compile-verified, user to rebuild |

**Last known 100% crash-free baseline: 0.2 (dd3b1030, 5.10.266).** 0.3 Beta 2 (25176a53)
fixes all known regressions but has less cumulative on-device hours than 0.2.

---

## 15. PROVEN PROCEDURES (how we do things here)

### 15.1 Porting a feature from Templar (or any donor kernel)
1. Find the donor commit(s): `https://api.github.com/repos/<owner>/<repo>/commits?sha=<branch>&path=<file>`
2. Download `.patch`: `curl -sL https://github.com/<owner>/<repo>/commit/<sha>.patch -o x.patch`
3. `git apply --check` → if fails, `git apply -3` → if fails, hand-port against the donor's
   FINAL file version (`raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>`)
4. Rename symbols if needed (vorpal→esk, rfx_→esk_) preserving upstream author credit
5. Adapt API deltas (5.10 vs donor base — e.g. tso_segs(sk,mss) vs min_tso_segs(sk))
6. Compile-test the .o with builder clang (§1.2)
7. Commit BACKPORT:/ANDROID: format + Link: tags + Change-Id + Signed-off-by
8. Push, **verify push**, update DEVELOPMENT.md

### 15.2 Stable-sublevel rebase procedure
1. `git fetch --no-tags stable tag v5.10.26X`
2. `git merge --no-commit v5.10.26X`
3. Conflict policy: vendor-diverged files keep HEAD; check each conflict against §2 map
4. **Verify BORE/EEVDF KABI block in include/linux/sched.h survives** (reserves 1-4)
5. Verify Makefile SUBLEVEL, commit `Update to Linux 5.10.26X`, push+verify
6. User rebuilds + tests before release

### 15.3 Uploading a release asset (exact steps)
1. `ls /home/akali/ESK_Reborn/*/` — locate user's zips
2. **VERIFY ZIP**: `unzip -p <zip> Image.zst | zstd -d | strings | grep "Linux version"`
   → must match intended commit sha
3. `gh release upload <tag> --clobber <zip> <module.tar.xz>`
4. `gh release view <tag> --json assets` — verify size+state=uploaded
5. Update notes: real filename in Builds table, banner update
6. `gh release edit <tag> --prerelease` (or remove) per user intent

### 15.4 Token/cost efficiency with the user
- User pays per token; avoid re-reading huge files; use grep/sed targeted extraction
- Logs: always grep for signatures (§1.3) before reading raw
- Don't re-clone repos that exist in /tmp — check first; /tmp is wiped on reboot
- Batch related shell work in single calls

---

## 16. CURRENT PROJECT STATUS & ROADMAP (as of this document)

**State: 0.3.2 (5.10.272, V0.3.2 localversion; PUSHED + verified Oct 2026 on both branches — tips: `git log --oneline origin/16.2-rebase -1` / `origin/17.0 -1`).** ESK v2.2,
- **Branch-divergence decision (Oct 2026, user-confirmed): `157418db` (PAN) + `15e0e52d` (mtk-mml) stay `17.0`-ONLY**, not mirrored to `16.2-rebase` (A17-flavored; `16.2-rebase` serves 16.x ROMs). Do NOT "fix" this divergence with a branch merge.
- **Upstream watch: `ESK-Project/android_kernel_xiaomi_mt6895:main` (Oct 3 2026)** carries a scheduler modernization we do NOT have: BORE 6.8.0 + 6.12-era EEVDF (delayed dequeue, run-to-parity, slice protection, lag placement, O(1) fastpath) on a 5.10.269 base. It does NOT have our ESK governor / NoMount / BBRplus. Port = dedicated project (fair.c rewrite zone), not a merge.
Templar stable fixes (EEVDF rescale, BORE weight, yield), le9uo/mali
fixes from XagaForge — on both branches. The XagaForge
v4l2 `request_complete` backport is REVERTED (camera-open panic, see §14);
A17 boot on xaga verified
via `Angxddeep/...:seventeen` @ 5.10.264 (our tree is superset, see §16.1).
GitHub owner renamed `AKALIorg` → `ENGRahsepar` (Oct 2026; remotes/docs updated, history NOT rewritten).

### 16.1 Android 17 readiness (checked Sep 2026)

* **Base:** `Angxddeep/...:seventeen` boots A17 on xaga at 5.10.264 with minimal
  vendor configs (no Polly, no UNAME_OVERRIDE). Our `16.2-rebase` is 5.10.270 +
  strict superset — version string spoof (`UNAME_OVERRIDE` → `5.10.226-...`)
  stays for Play Integrity and does not affect boot; A17 init does not require
  a GKI bump or selinux genfscon patch (already in 5.10.269 via `b55531ca`).
  No extra A17 init patch needed — the staging Mali IPA + media fixes above
  are the only device pieces that were missing vs XagaForge.
* **XagaForge sync:** `16.2` media (f89055+23f18ca) + `staging` Mali IPA
  (e11295) merged. Remaining XagaForge delta (~70 commits) is history we
  intentionally diverged from (Polly, etc.) — not needed for boot.
* **Binder/ashmem:** `BINDERFS=y`, `ASHMEM=y` present in `gki_defconfig`;
  builder LXC patch adds `USER_NS`/`PID_NS` at build time for Droidspaces.

### Roadmap (priority order)
1. **On-device validation of 17.0 HEAD** on A17 ROM — user builds
   KSU-SUSFS-LXC variant, runs §6 checklist + gaming session. Verify:
   **camera opens with zero reboots** (regression test for §14 `cde9d859`),
   `gaming_mode` + `prime_gaming_floor_pct` (default **64**) on policy7,
   CPU7 no longer oscillating 300↔2850, freq trace sane under throttle
   (`fceil/max_seen` path), media (AIE) no crash, GPU IPA sane
2. On-device validation of prime-selective boost + EEVDF yield (PUBG SF-capture
   targets: max FT <200ms, Big Jank −50%, run-over-run variance down)
3. ESK → default governor once user validates multi-day stability (flip
   `CONFIG_CPU_FREQ_DEFAULT_GOV_ESK=y` + localversion bump)
4. **MGLRU port** (Templar-MGLRU branch) — highest-impact remaining feature, large effort
5. Stable-sublevel tracking (5.10.26X when new lands, §15.2)
6. Optional: SurfaceFlinger RT scoping (needs ROM-side init.rc wiring, spec §1)
7. Optional: DTS CPU7 distinct capacity (EAS-level prime awareness prerequisite)

### Open questions / user-dependent
- Does the user's ROM auto-enable KSM? (`cat /sys/kernel/mm/ksm/run` under load)
- RT throttling recurrence under charging+gaming after binder fix? (if yes: consider
  sched_rt_runtime_us=980000 runtime sysctl — NOT baked in, see §2.2)
- Battery replacement candidate (1783 cycles) — hardware, not kernel

---

## 17. SESSION COST / EFFICIENCY NOTES

- This project's chat sessions are expensive for the user — maximize per-session output:
  plan with todo lists, batch tool calls, verify pushes/uploads inline, and END every
  session by updating DEVELOPMENT.md (§9) so the next session starts with zero re-discovery.
- Prefer downloading donor files raw over cloning donor repos (unless multi-file).
- Keep local clones: kernel (always), releases repo (/tmp/opencode/esk-releases),
  susfs (/tmp/opencode/susfs — re-clone when needed, /tmp wiped on reboot).
