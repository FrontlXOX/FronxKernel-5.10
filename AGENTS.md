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

Key findings from the 2026-09-23 tree audit (both trees on the build
machine — `android_kernel_xiaomi_mt6833#lineage-24.0` vs
`kernel_xiaomi_gold#gold-s-oss`):
- Same display panel on both sides
  (`nt35595_fhd_dsi_cmd_truly_nt50358_drv`) — display should carry over.
- Same MTK touch framework; the 5.10 side is a superset. everpal uses the
  NT36672C 1080x2400 variant — point the board file at it (verify the
  panel resolution matches everpal hardware, not gold's).
- MT6360 PMU: 5.10 uses the refactored new-generation drivers
  (`MFD_MT6360` / `CHARGER_MT6360` / `REGULATOR_MT6360` / `LEDS_MT6360`,
  mostly `=m`). Map the old 4.14 symbols to these — do not copy old
  drivers.
- Camera sensor sets are disjoint (only `ov16a1q` overlaps, and naming
  variants differ) — carry everpal's 5-sensor list; verify each imgsensor
  driver exists in the 5.10 tree before wiring.

Checklist, in order (from the audit):

1. **Board file shell:** ✅ done 2026-09-23 — `everpal-510.dts` +
   `everpal-510/cust.dtsi` stub on the 5.10 `mt6833.dts` base; everpal
   deltas applied (`mt6360_typec` intr pio9, `&chosen` verbatim);
   display GPIOs verified identical both sides (no pinmux adaptation);
   dtc-clean (23 KB blob, `lcmname` + `intr_gpio_num=0x9` verified).
   New branch `everpal-5.10` in the `kernel_xiaomi_gold` fork;
   `gold-s-oss` stays a pristine donor. `everpal-510.dtb` wired into
   `mediatek/Makefile` (the donor's own k510 file was never wired).
2. **Defconfig** (`arch/arm64/configs/`): ✅ done 2026-09-23 —
   `everpal_510_defconfig` (donor + 6 lines). All audit MT6360/audio/
   touch mappings were already present. Fixed 2 silent-drops: the
   `CONFIG_SOUND`/`_SND`/`_SND_SOC` parent stack (entire audio dropped)
   and `DMABUF_HEAPS` + `DEFERRED_FREE`/`PAGE_POOL` parents. ION
   decision: dmabuf heaps, no ION. Note: this donor builds monolithic
   (`# CONFIG_MODULES` unset) — zero module-load-order hazards for
   bringup; revisit modularization later.
3. **Battery auth:** ✅ done 2026-09-23 — ported 4.14
   `drivers/misc/maxim/` verbatim (3,881 lines across 11 files:
   ds28e16.c, onewire_gpio.c bit-bang, SHA384/ucl) instead of aliasing
   onto `ds28e30.c` — different chips, different command maps; aliasing
   risked silent battery-auth misbehavior. DTS: onewire pinctrl (GPIO53
   active/sleep) + `onewire_gpio` + `maxim_ds28e16` nodes, verified in
   blob. Defconfig `BATT_VERIFY_BY_DS28E16=y` + `ONEWIRE_GPIO=y` resolve.
   Caveat: 4.14-era driver needs a compile check at first full build
   (API drift unknown). Known issue (2026-09-24): `ONEWIRE_GPIO` is now
   defined twice — ported `drivers/misc/maxim/Kconfig` (bool) vs
   `drivers/power/supply/battery_secrete/Kconfig` (tristate); resolves
   to our `=y`, harmless, unify later.
4. **Touch:** ✅ done 2026-09-23 — neither 5.10 candidate dtsi fits
   (verified, not guessed: wrong chip / wrong bus+GPIO). Created
   `cust_mt6833_everpal_touch_nt36672c.dtsi` from 4.14 content (SPI1
   pins 27-30, reset 13, irq 14, 9.6 MHz, MP tables verbatim); no
   Makefile/Kconfig changes needed (TOUCH_LISTS mechanism). Defconfig:
   added `INPUT_TOUCHSCREEN=y` (third silent-drop catch).
5. **Camera:** ✅ done 2026-09-24 — 4 sensor drivers ported from 4.14
   (imx355×2, ov50c40, s5kjn1); mechanism decoded (not as assumed):
   `CUSTOM_KERNEL_IMGSENSOR` string → `FILTER_DRV` → `-D` flags guard
   `sensor_list.c` (no per-sensor Kconfig); all 5 sensor Makefiles
   rewritten to the 5.10 `imgsensor_isp6s-objs` pattern. ov16a1q resolved
   by silicon ID (everpal ofilm/qtech = 0x1641/0x1642 = existing
   AAC/SUNNY drivers — mapped by name, no port; stub dirs keep the `-D`
   machinery honest). Added IDs/DRVNAMEs to `kd_imgsensor.h`, 6 list
   entries, everpal CUSTOM string, `cust_mt6833_everpal_camera.dtsi`
   (everpal GPIOs/eeproms) + missing `wl2866d.dtsi` + `wl2866d.c`
   regulator port. Two more silent-drop kills: `CONFIG_REGULATOR` itself
   was off, and the SYSTEM heap needed its helper symbols.
6. **NFC** (`i2c3`/`st21nfc`): ✅ done 2026-09-23 — adapted, not copied:
   5.10's st21nfc binds `st,st21nfc` + `irq-gpios`/`reset-gpios`, while
   4.14's `mediatek,nfc-gpio-v2` bare-number style is incompatible. New
   node: i2c3@400kHz, addr 0x08, RST pio92, IRQ pio5 EDGE_RISING
   (verified default in both drivers' code). Bus label `i2c3` confirmed
   in 5.10 base.
7. **Fingerprint** (`spi5`/`fpc1542`): ✅ done 2026-09-23 — `spi5` label
   confirmed in 5.10 base; ported `fpc1542/` (driver + Kconfig +
   Makefile); TEE includes repointed
   `drivers/misc/mediatek/teei/` → `drivers/tee/teei/` (both exist in
   5.10; TEE surface is 3 symbols, `uuid_fp` present both sides — low
   risk, compile-time proof pending). DTS: pinctrl (RST pio17) +
   `spi5`/`fpc_spi@0` + EINT pio18; 5.10 base lacked the
   `fpsensor_fp_eint` stub — defined locally (driver probe fails
   `-EINVAL` without it). Defconfig `MTK_FINGERPRINT_SUPPORT` +
   `INPUT_FINGERPRINT_FPC1542` resolve.
8. **Haptics** (`i2c9`/`aw8697`): full port last — driver + node +
   calibration verbatim. Biggest single item; **not needed for boot.**
9. **Audio:** ✅ done 2026-09-24 — no 4.14-style EINT binding needed.
   5.10's `mt6359p-accdet` takes IRQs via `platform_get_irq()` from its
   own DT node, already present as a PMIC child in `mt6359p.dtsi`
   (included by base) with everpal-identical calibration values.
   Nothing to wire — audio probing is a non-issue for bringup.
10. **Boot test gates:** `sys.boot_completed=1` → dumpsys gpu →
    verifydevice.py. Scheduler work (WALT/BORE/EAS) is explicitly
    deferred — not needed for stock boot.

Then: build `Image.gz` + `dtbo.img` + modules, pack into the existing
boot image (magiskboot / AIK), flash, boot. **No full ROM build required.**
Success criteria: boots, display works, touch works, basic I/O works.

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
| 2 | BORE big-task rotation (default on) | `kernel/sched/eas_plus.c` (MTK EAS+) | n/a — stock `kernel/sched/` only (no `eas_plus.c`, no WALT, no BORE) | No MTK EAS in the 5.10 donor, so this is **not** a default-value tweak anymore — it becomes a scheduler-backport task, deferred past stock boot (see §5 item 10). |
| 3 | Mali DVFS period 100ms → 50ms | `drivers/misc/mediatek/gpu/gpu_mali/mali_valhall/mali-r32p1/.../mali_kbase_config_defaults.h` | `drivers/gpu/mediatek/gpu_mali/mali_valhall/mali-r32p1/drivers/gpu/arm/midgard/mali_kbase_config_defaults.h` (confirmed) | Same file, new path. Direct port. |
| 4 | accdet plugout debounce → 100ms | `drivers/misc/mediatek/accdet/mt6359/accdet.c` | `sound/soc/codecs/mt6359p-accdet.c` (confirmed) | Same logic, new location (MTK moved accdet under sound/soc in 5.10). Note PMIC is now `mt6359p`. Open: whether 5.10 needs the `ACCDET_EINT_IRQ` binding (§10). |
| 5 | DTS board tweaks | `arch/arm64/boot/dts/mediatek/evergo.dts`, `k6833v1_64.dts` | `arch/arm64/boot/dts/mediatek/mt6833.dts` (+ board file to create/adapt) | **Main work item.** evergo-only nodes to carry: `fpsensor_fp_eint` (Goodix EINT, pio 18), `i2c3`/`st21nfc@8` (NFC), `i2c9`/`aw8697_haptic@5a` (full calibration), `maxim_ds28e16`, `onewire_gpio` (GPIO53), `spi5`/`fpc_spi@0`. See §5 checklist. |
| 6 | ThermalMgmt v2 | Kernel thermal tunables: DVFSRC, down_rate, migration params, Mali `13000000` clock paths, CCI mode lock | MTK thermal framework in 5.10 | Re-tune against 5.10 sysfs paths; verify each path still exists before assuming. |
| 7 | Fronx branding | `Branding.patch` at kernel root (13 lines) | n/a | Trivial. Apply last. |
| 8 | `build.sh` | Kernel root (outputs zip+img to EverpalTweaks `out/`, PI-X repack, auto-revert) | n/a | Adapt paths for the 5.10 tree layout (§7). Build logic is kernel-version-agnostic. |
| 9 | MT6360 PMU (charger/regulator/LED) | `MFD_MT6360_PMU`, `MT6360_PMU_CHARGER`, `MT6360_PMU_FLED`, `MT6360_PMIC`, `MT6360_LDO` (`=y`) | `MFD_MT6360`, `CHARGER_MT6360`, `REGULATOR_MT6360`, `LEDS_MT6360`, `AUXADC`, `TCPC_MT6360` (`=m`) | Map old→new symbols; sources exist (`drivers/mfd/mt6360-core.c`, `drivers/power/supply/mt6360_charger.c`, `drivers/regulator/mt6360-regulator.c`, `drivers/leds/leds-mt6360.c`). Do not copy old drivers. |
| 10 | Battery auth (DS28E16 + onewire) | `drivers/misc/maxim/` + `onewire_gpio` (`xiaomi,onewire_gpio`, GPIO53) | `drivers/power/supply/battery_secrete/ds28e30.c` (ds28e30-only match) + `drivers/w1/masters/w1-gpio.c` | Extend `ds28e30.c` of-match with `maxim,ds28e16`, or port `drivers/misc/maxim/`; adapt the `auth_battery.dtsi` pattern to GPIO53 + everpal pinctrl. |
| 11 | Fingerprint (FPC1542, SPI) | `fpc1542/mtk_spi.c` + `spi5`/`fpc_spi@0` node + `fpsensor_fp_eint` | n/a — only generic `FPC_FINGERPRINT` (`fpc1022_tee.c`, TEE-based) | Port driver + Kconfig/Makefile + pinctrl + both DTS nodes. |
| 12 | Haptics (AW8697) | `INPUT_AW8697_HAPTIC` + `i2c9`/`aw8697_haptic@5a` node (full calibration: vib tables, wf_0..wf_9) | n/a — zero `*aw8697*` in tree | Full port (driver + node + calibration verbatim). Deferred — not needed for boot. Biggest single item. |
| 13 | Camera sensors | `cust_evergo_camera.dtsi`: imx355, ov16a1qofilm, ov16a1qqtech, ov50c40ofilm, s5kjn1sunny | `cust_mt6833_gold_camera.dtsi`: ov08d10, ov16a1q, ov50d40, ov64b40, s5khm6 | Disjoint sets. Carry everpal's list; verify each imgsensor driver exists in 5.10 before wiring. |
| 14 | Touchscreen variant | `cust_mt6833_touch_nt36672c_1080x2400.dtsi` | NT36672C drivers + `cust_mt6833_touch_1080x2400.dtsi` present | Point board file at everpal's 1080x2400 variant; verify panel resolution. |
| 15 | NFC (ST21NFC) | `i2c3` + `st21nfc@8` (`mediatek,nfc-gpio-v2`) | i2c buses present in 5.10 `mt6833.dts` | Enable bus + port child; verify the `mediatek,nfc-gpio-v2` driver exists in 5.10. |

## 7. Known 4.14 → 5.10 path moves (MTK reorganization)

MediaTek reorganized the driver tree in 5.10. When a 4.14 path is missing,
check these new locations first. **Extend this list as you discover more.**

- `drivers/misc/mediatek/gpu/` → `drivers/gpu/mediatek/` (confirmed:
  `mali_kbase_config_defaults.h` under
  `drivers/gpu/mediatek/gpu_mali/mali_valhall/mali-r32p1/drivers/gpu/arm/midgard/`)
- `drivers/misc/mediatek/accdet/` → `sound/soc/codecs/` (confirmed:
  `mt6359p-accdet.c`; PMIC renamed `mt6359` → `mt6359p`)
- MT6360 PMU stack refactored: `MFD_MT6360_PMU` / `MT6360_PMU_CHARGER` /
  `MT6360_PMU_FLED` / `MT6360_PMIC` / `MT6360_LDO` → `MFD_MT6360` /
  `CHARGER_MT6360` / `REGULATOR_MT6360` / `LEDS_MT6360` (+ `AUXADC`,
  `TCPC_MT6360`), mostly `=m`
- Audio machine driver modularized + renamed: `SND_SOC_MT6833_MT6359`
  → `SND_SOC_MT6833` + `SND_SOC_MT6833_MT6359P`
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
9. **Defconfig silent-drop check (standing rule).** The donor defconfig
   sets leaf symbols whose parent menus are off — they silently vanish
   from the resolved config. Every defconfig edit must be validated:
   `make O=/tmp/<tmp> ARCH=arm64 <defconfig>` into a temp dir, grep the
   resolved `.config` values, delete the temp dir. 30 seconds, catches
   the whole bug class. (Caught twice on 2026-09-23: the SOUND stack and
   DMABUF_HEAPS parents.)

## 9. Build & test

- **Defconfig start:** `k6833pv1_64_k510_defconfig` → `everpal_510_defconfig`
- **Toolchain:** clang-r416183b (AOSP prebuilt, per gold
  `build.config.common`: `LLVM=1 LLVM_IAS=1`,
  `CROSS_COMPILE=aarch64-linux-gnu-`) at
  `/root/EverpalTweaks/build/toolchains/clang-r416183b/`. Decision
  2026-09-24: genuine r416183b for the first build (eliminates the
  toolchain as a variable when bringup failures land); ZyC Clang 22
  (`/root/EverpalTweaks/build/toolchains/ZyC-clang-22.0.0/`) for
  iteration after boot. System aarch64-linux-gnu-gcc 15.2 provides the
  cross-prefix binutils.
- **Toolchain quirk (2026-09-23):** host dtc 1.7.2 rejects explicit
  `fragment@N` blocks (proven with minimal repro) — new root nodes in
  overlays must use the `&{/}` override style.
- **Test method:** build `Image.gz` + `dtbo.img` (+ modules), pack into the
  existing everpal boot image with magiskboot/AIK, flash, boot. No full ROM
  build is needed to validate the kernel.
- **What "done" looks like per phase:** §5 success criteria. Phase 1 is not
  done until display + touch + basic I/O work on real hardware.

## 10. Open questions (for Shovit / his developers)

- ACCDET EINT binding: does 5.10 `mt6359p-accdet` need the
  `ACCDET_EINT_IRQ` / `ACCDET_SUPPORT_EINT0` binding, or a different IRQ
  binding? (4.14 defconfig had them; 5.10 lacks them.)
- ION vs dmabuf: decide per the donor defconfig (§5 checklist item 2).
- `&keypad` / `&mtk_leds` in the 5.10 board file: do they claim GPIOs
  that everpal uses? Keep if harmless, drop on conflict.
- Panel resolution: confirm 1080x2400 is the everpal panel (not gold's).
- imgsensor drivers: verify imx355, ov16a1qofilm, ov16a1qqtech,
  ov50c40ofilm, s5kjn1sunny all exist under 5.10
  `drivers/misc/mediatek/imgsensor/` before wiring the camera list.
- `mediatek,nfc-gpio-v2` driver: present in the 5.10 tree?
- `spi5` label: present in 5.10 `mt6833.dts`?
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
- **2026-09-23 (evening):** Full tree audit on the build machine (4.14
  `lineage-24.0` vs gold `gold-s-oss`; donor confirmed at `bd6f35a`).
  Findings folded into §5–§7 and §10: same panel, same touch framework,
  MT6360 new-gen drivers present in 5.10; the real port work is the
  fingerprint driver, AW8697 (deferred past boot), DS28E16/onewire
  compat, the camera sensor list, and NFC verification. BORE is not a
  default-tweak in 5.10 (no MTK EAS in the donor) — it becomes a
  scheduler-backport task. Nothing structural blocks a stock boot.
- **2026-09-23 (night):** Phase 1 items 1–2 complete, reviewed, approved:
  `everpal-510.dts` board shell (+ `cust.dtsi` stub, dtc-clean) and
  `everpal_510_defconfig` (donor + 6 lines; fixed silent-dropped SOUND
  and DMABUF_HEAPS parents; ION-vs-dmabuf → dmabuf heaps; donor is
  monolithic — no module-load-order hazards for bringup). Work committed
  on new `everpal-5.10` branch in the `kernel_xiaomi_gold` fork;
  `gold-s-oss` stays pristine. Standing rule added: temp-dir
  resolved-grep check on every defconfig edit.
- **2026-09-23 (night):** Phase 1 items 3–4 complete, reviewed, approved:
  battery auth via verbatim port of 4.14 `drivers/misc/maxim/` (11
  files; GPIO53 onewire pinctrl + DTS nodes; defconfig symbols resolve;
  compile check deferred to first full build) and everpal-specific
  NT36672C touch dtsi (SPI1, reset 13; third silent-drop catch:
  `INPUT_TOUCHSCREEN`). dtbs build passes (68 KB blob, 13 new nodes
  verified). Committed on `everpal-5.10`.
- **2026-09-24 (early):** Phase 1 items 6–7 complete, reviewed, approved:
  NFC via `st,st21nfc` binding adaptation (i2c3, RST pio92, IRQ pio5) and
  fpc1542 fingerprint port (TEE includes repointed; `fpsensor_fp_eint`
  stub defined locally after finding 5.10 base lacks it). dtbs passes
  (77 KB blob, all six new compatibles verified). Committed as 4dd4657
  on `everpal-5.10`. Item 5 (camera) unblocked as a sensor-driver port:
  imx355/ov50c40/s5kjn1/ov16a1qqtech from 4.14, ov16a1q ofilm adapted in
  place. Item 9 (audio ACCDET binding) still open.
- **2026-09-24 (early):** Phase 1 board adaptation COMPLETE (items 1–7,
  9; item 8 haptics deferred past boot per plan). Camera: 4 sensor
  drivers ported, imgsensor `-D` flag mechanism decoded, ov16a1q mapped
  by silicon ID (no port), wl2866d regulator ported; two more
  silent-drop kills (`CONFIG_REGULATOR`, SYSTEM heap helpers). Audio:
  no EINT binding needed (`platform_get_irq`, node already in
  `mt6359p.dtsi`). dtbs passes (77 KB blob, all peripheral compatibles
  + 29 camera nodes verified). Committed on `everpal-5.10`. Next: first
  full kernel build (item 10a/10b).
- **2026-09-24 (early):** Toolchain gate: gold tree expects clang-r416183b
  (absent from disk); ZyC Clang 22 + system gcc-15 cross present. Decision:
  fetch genuine r416183b for the first build, ZyC-22 for iteration after
  boot. Fetch succeeded (Clang 12.0.5 verified). First build attempt failed
  in 28s on the `modules` goal — donor builds monolithic, goal invalid;
  retry with `Image.gz dtbs`. Defconfig cruft noted (not blocking):
  `SND_SOC_MT6789_MT6366`/`SND_SOC_MT6885_MT6359P` enabled for other SoCs.
- **2026-09-24 (early):** Second build attempt failed at 77s: vendor block
  at `crypto/Makefile:190-200` (N17/HQ-293392) builds `ecdsa_generic.o`
  from `ecdsasignature.asn1.[co]` with no kbuild generation rule;
  `CONFIG_CRYPTO_ECDSA` is pulled in via selects, not present in our
  defconfig. Diagnosing the select chain before fixing.
- **2026-09-24 (early):** ECDSA vendor block resolved: `crypto/Makefile`
  N17/HQ-293392 builds ecdh+ecdsa unconditionally with no Kconfig guard;
  the asn1 generation pattern exists but needs `CONFIG_ASN1`, which
  neither our defconfig nor gold's resolves — the donor is broken the
  same way (proven, not our regression). Fix: commented out the 4 ecdsa
  lines + `obj-y ecdsa_generic.o` with the proof in a comment; ecdh half
  untouched. Safe: zero asymmetric-key consumers in this config
  (no X.509/PKCS7/IMA/EVM/MODULE_SIG) and zero "ecdsa" references in
  crypto/security/net. Rebuild running.
- **2026-09-24 (early):** Rebuild #2 failed at 90s past ECDSA: `-Werror`
  on dead declarations in our ported `wl2866d.c` (5 unused locals/label —
  this tree builds with `CONFIG_WERROR=y`) and a pre-existing unused
  function `set_idac_trim_val` in donor `mt6338.c`. Fix approved: delete
  the dead lines in our port; for the donor file, remove only if verified
  truly unreferenced, else report first.
