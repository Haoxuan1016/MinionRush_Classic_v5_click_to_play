# 05 — Open threads, caveats, and next levers

What is done, what is NOT, and where to look next if more edits are requested.

## Done & verified

- Located full token/banana currency engine in `libDespicableMe.so` (see 02).
- Shipped "always-rich tokens" patch (GetTokens=999999 + CanAfford-always-true). Confirmed
  working in-game by the user.
- One-click self-contained offline APK pipeline (bundled gamedata.zip auto-extract).

## NOT done / known caveats

1. **"+1000 per transaction" literal behavior was NOT implemented.** The user's earlier wording
   was "every token gain/consumption results in +1000". The shipped build instead does
   "always show 999999 + always affordable", which the user accepted as working. If the literal
   +1000-per-event behavior is wanted, you must patch the WRITE path (`SetCurrency` writers
   0x780b74 / 0x780c20 → make them add 1000 to the decoded value before `SecuredInt::Set`,
   or hook the gain/spend events). This needs care with the secured-int encode (0x2d3cb4).

2. **Spend deduction path not neutralized.** `GetTokens` patch makes the balance LOOK huge, but
   the actual stored secured-int could still be decremented by inline spend logic
   (transaction handler `0x28f040` region). In practice it didn't matter (display + CanAfford
   were enough), but if some purchase still fails or the real balance ever drains, neutralize
   the token write in the `SetCurrency` writers, or floor the decoded value in
   `SecuredInt::Get` for the token sub-struct.

3. **Save persistence of the fake balance:** `GetTokens` returning 999999 means saves
   (which call the getters, see 02 / `fcn.006d875c`) will serialize a large number. Not
   tested across full save/reload cycles for side effects.

4. **The shop "No Internet" / IAP item display problem was never solved** (separate, earlier
   goal). Store offline display attempts all failed: internal data seeding incl. full 534M
   dlcs, testkey signature match, connectivity bypass (hasConnectivity/CheckConnectionType→1).
   The currency patch route was chosen specifically to BYPASS the broken store. If revisiting
   the store, note `NativeSetReward` is a dead stub (see 01).

5. **Debug-key signing.** Final APK is debug-signed. Cannot update a differently-signed prior
   install (must uninstall). See 04 for re-signing with the project testkey.

## High-value next levers (if asked to edit further)

- **Make tokens truly unlimited & persistent:** patch the token write in `Wallet::SetCurrency`
  (`0x780b74`): after computing the new value, force it to a large constant before `bl 0x2d3cb4`,
  OR make the call a no-op so tokens never decrease. Combine with existing GetTokens patch.
- **Bananas too:** identical structure — `GetBananas` at `0x780a20` (offset +0x10, type via
  selector). Same patch shape as tokens.
- **Other secured values:** `SecuredInt::Set` (0x2d3cb4) has ~20 callers all in currency code;
  enumerate with `python xref.py 0x2d3cb4` to find every currency mutation site.
- **Find any new string-anchored feature:** add keywords to `recon.py` KEYWORDS, run it, then
  `python xref.py <stringaddr>` to find who uses it, then r2 `pdf` the function.

## Important reusable facts (don't re-derive)

- `.text` file_offset == vaddr (patch directly).
- Binary is unobfuscated; class/RTTI names + source paths intact in `.rodata`.
- Currency = secured dual-copy XOR/ROR ints; use get/set helpers, don't poke raw bytes.
- Token type id = 1; banana = 0. Token secured int at wallet-sub(+0x1d0 base)+0x20.
- radare2 `Ps` project save hangs — never use it; re-run `aaa` or use capstone scripts.
- repack via `repack_so.py` (raw zip swap, ~1s) instead of apktool full rebuild (slow, 562MB
  gamedata.zip).
