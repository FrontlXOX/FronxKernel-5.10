# AGENTS.md — FronxKernel 5.10 Port Workspace

> **Read this entire file before doing anything.** Then inspect the trees,
> write a step-by-step plan, and get Shovit's approval before changing any
> code. Planning first is mandatory, not optional.

## 1. What this is

This repo is the working area for porting **FronxKernel** — Shovit Dutta
(FrontlXOX)'s custom kernel for the Xiaomi **everpal** — from Linux **4.14**
to Linux **5.10**.

This repo does **not** hold kernel source. It holds the plan, the patch
inventory, porting notes, and helper scripts. The actual kernel work happens
in `FrontlXOX/android_kernel_xiaomi_mt6833`. Treat this repo as the mission
brief: learn it, then act.

## 2. Device & SoC

- **Device:** Xiaomi POCO M4 Pro 5G / Redmi Note 11S 5G, codename `everpal`
  (board name `evergo`)
- **SoC:** MediaTek MT6833P (Dimensity 810), Mali-G57 MC2, MT6359P PMIC
- **Target OS:** Android 16 / LineageOS 23.x
- **Current production kernel:** Linux 4.14.357-Aqua — "FronxKernel 1.0"
  (branch `lineage-24.0` of `android_kernel_xiaomi_mt6833`)

## 3. Repo landscape — the full picture

| Repo | Role | Branch | Notes |
|------|------|--------|-------|
| `FrontlXOX/EverpalTweaks` | Meta repo, 9 submodules | `main` | `src/trees/kernel-5.10` points at the 5.10 donor (see §4) |
| `FrontlXOX/android_kernel_xiaomi_mt6833` | Production 4.14 kernel (FronxKernel 1.0) | `lineage-24.0` | One-shot patch system: `Branding.patch`, `ResukiSU-SusFS.patch`, applied by `build.sh` |
| `FrontlXOX/kernel_xiaomi_gold` | **5.10 donor** (fork) | `gold-s-oss` | Xiaomi OSS drop, 5.10.168, MT6833 — the base for this port |
| This repo (`FronxKernel-5.10`) | Port workspace | `main` | Plan, inventory, notes, scripts — you are here |

Upstream donor source: `linastorvaldz/kernel_xiaomi_gold`.
Donor device: **gold** = Redmi Note 13 5G, MT6833 (Dimensity 6080),
ships HyperOS 3.0 / Android 15.

## 4. Why this donor

The previous 5.10 reference (`MillenniumOSS/kernel_millennium_mt6789-common`)
was a **different SoC** (MT6789) — porting from it meant porting across chips.
The gold donor is the **same MT6833 family** as everpal, so the job shrinks
from "port across SoCs" to "adapt board-level differences."

What the donor already provides:
- `arch/arm64/boot/dts/mediatek/mt6833.dts` — SoC device tree
- `arch/arm64/configs/k6833pv1_64_k510_defconfig` — MT6833 + 5.10 defconfig
- `arch/arm64/configs/gold_defconfig` — gold board defconfig
- mali_valhall r25p0 / r30p0 / r32p0 / **r32p1** GPU drivers
- `sound/soc/codecs/mt6359p-accdet.c` — headset detection driver

## 5. The plan — two phases, strict order

### Phase 1: Stock 5.10 boot on everpal

Goal: a clean 5.10 kernel that boots on everpal hardware. **No root, no
tweaks, no scheduler changes.**

1. Base the work on `gold-s-oss` as-is.
2. **Defconfig:** start from `k6833pv1_64_k510_defconfig`, add everpal board
   options. Do NOT copy the 4.14 defconfig over.
3. **Device tree:** adapt everpal board specifics onto the gold base —
   display panel, touchscreen controller, fingerprint, camera sensors,
   and any `evergo.dts` / `k6833v1_64.dts` tweaks from the 4.14 tree.
   This is the main work item of the whole port.
4. Build `Image.gz` + `dtbo.img` + modules. Test-boot by packing into the
   existing boot image (magiskboot / AIK) — **no full ROM build required.**
5. Success criteria: boots, display works, touch works, basic I/O works.

### Phase 2: Forward-port FronxKernel features, one at a time

Each feature lands individually and gets a test-boot before the next goes
in, so breakage is always attributable. Suggested order (easiest first):

1. accdet debounce fix → 2. Mali DVFS tuning → 3. BORE default →
   4. ThermalMgmt v2 → 5. ReSukiSU + SUSFS → 6. Branding

## 6. Customization inventory — 4.14 → 5.10 map

Verified 2026-09-23 against the gold tree. Keep this table current as the
port progresses.

| # | Feature | 4.14 location | 5.10 gold location | Port notes |
|---|---------|---------------|--------------------|------------|
| 1 | ReSukiSU v4.2.0-rc3 + SUSFS v2.3.0 | `ResukiSU-SusFS.patch` at kernel root (4752 lines, applied one-shot by `build.sh`) | n/a — regenerate | Both support 5.10. Regenerate the combined patch for 5.10 the same way the 4.14 one was made. Do not hand-port 4752 lines. |
| 2 | BORE big-task rotation (default on) | `kernel/sched/eas_plus.c` (MTK EAS+) | 5.10 MTK EAS (reworked upstream) | This is a **default-value tweak**, not a scheduler port. Find the equivalent tunable in 5.10's EAS code. |
| 3 | Mali DVFS period 100ms → 50ms | `drivers/misc/mediatek/gpu/gpu_mali/mali_valhall/mali-r32p1/.../mali_kbase_config_defaults.h` | `drivers/gpu/mediatek/gpu_mali/mali_valhall/mali-r32p1/.../mali_kbase_config_defaults.h` | Same file, new path. Direct port. |
| 4 | accdet plugout debounce → 100ms | `drivers/misc/mediatek/accdet/mt6359/accdet.c` | `sound/soc/codecs/mt6359p-accdet.c` | Same logic, new location (MTK moved accdet under sound/soc in 5.10). Note PMIC is now `mt6359p`. |
| 5 | DTS board tweaks | `arch/arm64/boot/dts/mediatek/evergo.dts`, `k6833v1_64.dts` | `arch/arm64/boot/dts/mediatek/mt6833.dts` (+ board file to create/adapt) | **Main work item.** Carry everpal board specifics onto the 5.10 base. |
| 6 | ThermalMgmt v2 | Kernel thermal tunables: DVFSRC, down_rate, migration params, Mali `13000000` clock paths, CCI mode lock | MTK thermal framework in 5.10 | Re-tune against 5.10 sysfs paths; verify each path still exists before assuming. |
| 7 | Fronx branding | `Branding.patch` at kernel root (13 lines) | n/a | Trivial. Apply last. |
| 8 | `build.sh` | Kernel root (outputs zip+img to EverpalTweaks `out/`, PI-X repack, auto-revert) | n/a | Adapt paths for the 5.10 tree layout (§7). Build logic is kernel-version-agnostic. |

## 7. Known 4.14 → 5.10 path moves (MTK reorganization)

MediaTek reorganized the driver tree in 5.10. When a 4.14 path is missing,
check these new locations first. **Extend this list as you discover more.**

- `drivers/misc/mediatek/gpu/` → `drivers/gpu/mediatek/`
- `drivers/misc/mediatek/accdet/` → `sound/soc/codecs/`
- `drivers/misc/mediatek/` (other subdirs) → check `drivers/` top level and
  `drivers/soc/mediatek/` before concluding a driver is gone

## 8. Rules for the agent working here

1. **Plan before code.** Read this file fully, inspect both trees
   (`android_kernel_xiaomi_mt6833#lineage-24.0` and
   `kernel_xiaomi_gold#gold-s-oss`), write a step plan, get Shovit's
   approval. No drive-by commits.
2. **Kernel-only scope.** Do not touch `device/`, `hardware/`, `vendor/`,
   or sepolicy trees. Those belong to full ROM builds, which are out of
   scope. The kernel tree is self-contained for this port.
3. **One feature per change (Phase 2).** Each port lands separately with a
   test-boot, so regressions are attributable.
4. **Never break the Phase 1 boot.** If a Phase 2 port breaks boot, revert
   it immediately and report — do not stack more changes on top.
5. **Keep the one-shot patch discipline.** New changes should be
   expressible as discrete patch files where practical, matching the
   FronxKernel 1.0 convention (`Branding.patch`, `ResukiSU-SusFS.patch`).
6. **Document as you go.** Update the inventory table (§6) and the path-move
   list (§7) in this file when you learn something. This file is the shared
   brain — keep it accurate.
7. **Commit message convention** (matches the kernel repo):
   `EMOJI [TAG]: description` — e.g. `🔊 [FIX]: ...`, `⚡ [PERF]: ...`,
   `🦋 [FEAT]: ...`, `🛠️ [FIX]: ...`, `📚 [DOCS]: ...`, `🦑 [CHORE]: ...`.
8. **Build machine access:** SSH via Tailscale to the WSL build host when
   available (`ssh -F ~/workspace/ssh_wsl/ssh_config wsl`). If SSH is down,
   say so and wait — do not improvise a different build environment.

## 9. Build & test

- **Defconfig start:** `k6833pv1_64_k510_defconfig`
- **Toolchain:** as per the gold tree / existing `build.sh` — verify before
  the first build, do not assume.
- **Test method:** build `Image.gz` + `dtbo.img` (+ modules), pack into the
  existing everpal boot image with magiskboot/AIK, flash, boot. No full ROM
  build is needed to validate the kernel.
- **What "done" looks like per phase:** §5 success criteria. Phase 1 is not
  done until display + touch + basic I/O work on real hardware.

## 10. Open questions (for Shovit / his developers)

- Board diff gold vs everpal: exact display panel, touchscreen controller,
  fingerprint reader, camera sensor models. Needed for §5 Phase 1 step 3.
- ReSukiSU + SUSFS 5.10 patch generation: confirm the exact procedure used
  for the 4.14 combined patch so it can be repeated for 5.10.
- ThermalMgmt v2: confirm where its tunables live (kernel driver vs
  userspace scripts) so the 5.10 sysfs paths can be mapped.
- Branch skew note: EverpalTweaks submodules span lineage-23.0 / 23.2 /
  24.0 — alignment can wait until a full ROM build is in scope.

## 11. History

- **2026-09-23:** Workspace created. 5.10 donor swapped from
  `MillenniumOSS/kernel_millennium_mt6789-common` (wrong SoC) to
  `FrontlXOX/kernel_xiaomi_gold` (gold-s-oss, 5.10.168, MT6833).
  Per-item port feasibility verified against the gold tree — all
  FronxKernel 1.0 customizations have a 5.10 path; board DTS adaptation
  is the main work item.
