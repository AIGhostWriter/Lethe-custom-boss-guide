# Lethe Custom Boss Guide

A guide to creating custom boss encounter mods for **Limbus Company** using the **Lethe** framework.

| Chapter | Content |
|---------|---------|
| [Chapter 1](chapter1_EN.md) | Finding boss data in dumpedData |
| [Chapter 2](chapter2_EN.md) | Building the full encounter mod |
| [Chapter 3](chapter3_EN.md) | Advanced Lua scripting |
| [Chapter 4](chapter4_EN.md) | Complete sample: Soji Abi (소지아비) |

---

## Chapter 4 — What's Inside

Chapter 4 is a full end-to-end walkthrough of a real boss mod built from scratch. It documents every design decision, file, error, and fix as they actually happened during development.

| Section | Content |
|---|---|
| Planning Stage | Boss concept, two-keyword mechanic design, 3-cycle pattern philosophy, ID scheme |
| Data Extraction | Finding appearance strings, passive IDs, and skill motion names in dumpedData |
| Full Folder Tree | All 12 files across 7 directories, with encounter.json and subchapterui.json breakdowns |
| Passives | Boss vs. identity passive format differences; using base-game passives; custom Lua trigger passive |
| Skills — Deep Dive | `GiveBuffOnUse stack:0`, `GiveBuffBeforeAttackIfOnesideAttack`, `TagetNumAdderOnLoseDuel`, SuperCoin + Unbreakable, `CriticalDmgUpByJsonValue`, 9-coin Vibration chain |
| Advanced Lua | **SojiMotion.lua** (per-coin direct frame functions), **SojiBarrage.lua** (random frame on SuperCoin), **SojiPattern.lua** (3-cycle pattern with `skillslotreplace` + passive trigger) |
| Common Mistakes | `OnCoinToss` + SuperCoin bug, `skillslotreplace` silent failure, locale coindescs mismatch |
