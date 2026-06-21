# RE_information — Minion Rush (Despicable Me) 5.7.0 reverse-engineering map

This folder records everything learned while reversing the game so future agents can
make more edits **without re-doing the analysis from scratch**. It is a knowledge dump,
not a fresh task.

## Contents

- `01_overview.md` — target, toolchain, environment, file layout
- `02_native_currency_map.md` — the core: token/banana currency engine in `libDespicableMe.so`
- `03_applied_patches.md` — exact patches applied for the "always-rich tokens" build
- `04_workflow_and_scripts.md` — how to analyze / patch / repack / sign / install (reusable)
- `05_open_threads.md` — what is NOT yet done / known caveats / next levers

## TL;DR

- Game logic (HUD, shop, currency) is **100% native** in `lib/armeabi-v7a/libDespicableMe.so`
  (ARM 32-bit, 24 MB). Java layer only does surface/touch/audio/ads.
- The `.so` is **NOT obfuscated** — intact C++ class names, RTTI strings, and even original
  Windows source paths (`C:\DM2\UD44_Android_b\source\game\...`). This makes it very tractable.
- Currency is stored as **anti-tamper "secured integers"** (XOR/ROR encoded, dual-copy
  tamper check). Do NOT poke the raw stored bytes blindly; use the get/set helpers.
- The shipped working build patches `GetTokens()` to return 999999 and forces the token
  affordability check to always pass. Verified on device.

## Key addresses (file offset == vaddr for `.text`)

| vaddr | meaning |
|---|---|
| `0x780a20` | `GetBananas()` — `bl 0x7e9304; add r0,0x10; mov r1,0; b 0x2d3d30` |
| `0x780a3c` | `GetTokens()`  — `bl 0x7e9304; add r0,0x20; mov r1,1; b 0x2d3d30`  ← PATCHED |
| `0x780a58` | `CanAfford(type, price)` (token branch at `0x780ab4`) ← PATCHED |
| `0x780b74` | `Wallet::SetCurrency(Currency*)` (type-dispatched setter) |
| `0x7e9304` | wallet sub-struct selector (type 0 → +8, type 1 → +0x1d0) |
| `0x2d3d30` | `SecuredInt::Get(this, clampMode)` (decode + tamper check) |
| `0x2d3cb4` | `SecuredInt::Set(this, value)` (encode into 2 copies) |
| `0x6eb700` | `Currency::GetAmount()` (reads desc+0x60) |
| `0x6ec1d4` | `Currency::GetType()` (reads desc+0x24) |
| `0x28f040` | transaction RESULT handler (matches `NotEnoughTokens`/`TokenValueBelow`) |

See `02_native_currency_map.md` for full detail.
