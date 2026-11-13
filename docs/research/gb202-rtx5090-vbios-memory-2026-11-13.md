# BL-HW-5090-FW-01 — GB202/RTX 5090 firmware-only research

**Date:** 2026-11-13 · **Zone:** 1 · **State:** proposed research, no firmware collected or modified.

## Decision and scope

Research the firmware-level constraints on RTX 5090 (GB202) VRAM identification, memory strap/configuration, training profile, signed VBIOS and NVIDIA GSP initialization. **No soldering, board alteration, hardware test, ROM flashing, production GPU firmware loading or driver modification.**

REA is the read-only analysis **instrument**, not the canonical storage of experimental truth. The corresponding working record is in Noelab, currently represented by the private `noerix-dev/laborix` repo pending the Postgres/MCP rollout.

## Evidence at project start

1. Panchovix's open-source NVIDIA VBIOS Vix Editor exposes `nvbios memory ROM [--compare ROM2] [--raw]`. Its memory tables are explicitly **read-only**. The Blackwell board memory-training profile (object `0x89`) is not decoded. The project notes that the profile only takes effect by flashing the entire ROM. Thus the currently documented VBIOS override **does not demonstrate** memory-topology or capacity modification.
   - https://github.com/Panchovix/nvidia-vbios-vix-editor
   - https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/main/docs/changelog-details.md
2. Separate driver-mediated VBIOS override loaders exist for Windows and Linux. An override of voltage/power parameters is **not evidence** of a firmware-only VRAM unlock:
   - https://github.com/Panchovix/nvidia-windows-vbios-vix-loader
   - https://github.com/Panchovix/nvidia-linux-vbios-vix-loader
3. Reports of 64/96 GB RTX 5090 modifications claim firmware progress, but do not provide public reproducible binaries, hardware topology data or independent memory-validation results:
   - https://shiftdelete.net/en/rtx-5090-96gb
   - https://videocardz.com/newz/alleged-geforce-rtx-5090-with-96gb-listed-for-sale-in-china

**Question under investigation:** Which firmware and driver layers encode and enforce memory-device density, populated channels, clamshell configuration and training? Compare GB202 GeForce vs professional variant, but never confuse the presence of a capacity field with physical memory.

## REA investigation — read-only, evidence-first

1. Obtain lawful source materials: published source code, documentation and authorized **original** board VBIOS dumps; record board ID, image version, provenance, SHA-256. No unknown downloaded ROMs, proprietary images or executable artifacts committed to this repository.
2. Analyze the **source** of the editor and both loaders: distinguish fields exposed read-only from editable fields, byte ranges touched in override operations, and boundaries delegated to NVIDIA driver/GSP. Use code references and exact versions/commits.
3. With approved local binary-analysis tools through REA, inspect authorized images offline and compare the memory-configuration table, straps and object `0x89` without writing or running modified firmware.
4. Track each conclusion as **observed**, **derived**, **hypothesis** or **unknown**; document alternative explanations such as board straps or silicon fuses.
5. Produce a reproducible firmware-only feasibility report for the Noelab task and state whether a capacity change without physical memory is meaningful (**it is not**), whether the chip/board can initialize a different physically populated configuration (**unknown**), and what evidence would resolve it.

## Acceptance criteria

- Sources, dates, image/commit identities and SHA-256 when images are lawfully available.
- Table of firmware vs GSP/driver vs physical-board responsibilities, with confidence and evidence.
- Separate claims for 48 GB, 64 GB and 96 GB; firmware alone does not conjure additional GDDR7 chips.
- No flash, driver override, board modification or executable payload; do not assert success in the absence of reproducible validation.

**Linked working item:** `BL-HW-5090-FW-01` (Noelab; private repository `noerix-dev/laborix`, research branch of the same name).
