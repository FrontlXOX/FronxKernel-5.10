# FronxKernel 5.10 — Port Workspace

Working area for porting **FronxKernel** (Xiaomi everpal, MT6833P) from
Linux 4.14 to Linux 5.10.

- **Donor:** `FrontlXOX/kernel_xiaomi_gold` (`gold-s-oss`, 5.10.168 — Xiaomi OSS drop, same MT6833 family)
- **Production kernel:** `FrontlXOX/android_kernel_xiaomi_mt6833` (`lineage-24.0`, FronxKernel 1.0)
- **Plan:** Phase 1 = stock 5.10 boot on everpal, then Phase 2 = forward-port features one at a time

Start with **[AGENTS.md](AGENTS.md)** — it has the full plan, the per-item
4.14 to 5.10 port map, and the rules for working here. Plan before code.
