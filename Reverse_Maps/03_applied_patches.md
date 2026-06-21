# 03 — Applied patches (the shipped "always-rich tokens" build)

These are the EXACT byte patches in the working final APK
(`MinionRush-5.7.0-offline-tokens.apk`). Patched file:
`build/re/libDespicableMe.patched.so`.

Patcher script: `build/re/patch.py` (keystone assemble + capstone verify).
`.text` file_offset == vaddr, so offsets below are both.

## Patch 1 — `GetTokens()` always returns 999999

- **vaddr / offset:** `0x780a3c`
- **Original (12 bytes):**
  ```
  ldr  r0, [r0, 8]      ; 080090e5
  bl   0x7e9304         ; (relative)
  add  r0, r0, 0x20     ; 200080e2
  mov  r1, 1            ; 0110a0e3
  ... (tail b 0x2d3d30 at 0x780a54)
  ```
- **Patched (12 bytes), bytes = `3f0204e3 0f0040e3 1eff2fe1`:**
  ```
  movw r0, #0x423f      ; 3f0204e3
  movt r0, #0x000f      ; 0f0040e3   -> r0 = 0x000F423F = 999999
  bx   lr               ; 1eff2fe1
  ```
- Effect: every reader of the token balance (HUD, shop, gameplay logic — 20 call sites) sees
  999,999 tokens.

## Patch 2 — token affordability check always passes

- **vaddr / offset:** `0x780ab4` (the type==1 / token branch inside `CanAfford` `fcn.00780a58`)
- **Patched (8 bytes), bytes = `0150a0e3 070000ea`:**
  ```
  mov r5, #1            ; 0150a0e3
  b   0x780adc          ; 070000ea   -> skip the real balance read/compare
  ```
- Effect: `CanAfford(type=token, price)` always returns true, so the shop lets you buy
  token-priced items regardless of the (decoded) stored balance.

## What was deliberately NOT patched

- The secured-int encode/decode (`0x2d3cb4` / `0x2d3d30`) was left untouched — safer than
  fighting the dual-copy tamper check.
- The inline spend deduction in the transaction path (`0x28f040` region / `SetCurrency`
  writers) was left untouched. User confirmed the two patches above are sufficient in practice.

## Verification status

- Each patch verified by re-disassembly (capstone) at build time — see patch.py output.
- Repacked APK passed `zipfile.testzip()` (no corrupt entries); patched `.so` present and
  DEFLATE-compressed; AndroidManifest + gamedata.zip intact.
- uber-apk-signer: zipalign + sign OK, signatures v1/v2/v3 verified (debug key).
- **Installed and confirmed working in-game by the user on device `ff882f5a`.**

## Final artifact

- `MinionRush-5.7.0-offline-tokens.apk` (600.6 MB, 629,766,683 bytes)
- SHA-256: `13E145FC75D1C3A8A8A375B914E08A366BCDEA4325A14D8CB20C4BF593043942`
- Signed with debug key (CN=Android Debug). Different key than the original/all-unlocked
  builds, so a signature-mismatched prior install must be uninstalled first.
