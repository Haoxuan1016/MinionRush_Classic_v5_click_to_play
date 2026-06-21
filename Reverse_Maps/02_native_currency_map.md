# 02 — Native currency engine map (libDespicableMe.so)

All addresses are virtual addresses in `libDespicableMe.so`. For `.text`,
**file_offset == vaddr**, so patch bytes go straight to the same file offset.

## Currency type enum

From the currency-table initializer `fcn.0046b44c` (which sets up the four names in order):

```
0 = bananas            ("bananas")
1 = tokens             ("tokens")        <-- premium currency, the target
2 = item_def_currency  ("item_def_currency" / puzzle pieces etc.)
3 = real_money         ("real_money")
```

Other relevant `.rodata` anchors:
- `0x46b834` "bananas", `0x46b844` "tokens", `0x46b854` "item_def_currency", `0x46b880` "real_money"
- `0x6d8cfc` "bananas_%d_tokens_%d_size_%d_bytes" (save serialization format)
- `0x28f40c` "NotEnoughTokens", `0x28f424` "TokenValueBelow" (spend-failure debug tags)
- `0x14c1d37` "coins", `0x14c1d3d` "shop_bananas", `0x14c1d4a` "shop_tokens"
- save files: "SaveFileGame.dat", "SaveFileCC.dat" (`0x14b7a28` area)

## Wallet object model

A "wallet" object holds **secured integers** (anti-tamper). Token amount lives at
`wallet + 0x20`, banana amount at `wallet + 0x10`.

### Wallet sub-struct selector — `fcn.007e9304(this, &type)`
```asm
ldr  r2, [r1]        ; type value (0 or 1...)
add  r1, r0, 8       ; default sub-object at this+8
cmp  r2, 1
addeq r1, r0, 0x1d0  ; type==1 -> this+0x1d0
mov  r0, r1
bx   lr
```
So the wallet base depends on currency type; then the per-currency secured int is at +0x10
(bananas) / +0x20 (tokens) within that base.

## Secured integer primitives (anti-tamper)

These are GENERIC (used for all currencies). They XOR/ROR-encode the value against global
keys and keep TWO encoded copies that must agree (tamper detection). Do not write raw values.

### `fcn.002d3d30` = `SecuredInt::Get(this, clampMode)`
- Reads two encoded copies + global keys at `0x14488cc..0x14488d8`, decodes, and compares them.
- If they mismatch (tamper), returns early.
- `clampMode` (r1): `==1` clamp to upper bound (movgt), else clamp to lower bound (movlt).
- Returns the decoded amount in r0.

### `fcn.002d3cb4` = `SecuredInt::Set(this, value)`
- Encodes `value` into the two copies using the global keys (ror/eor sequence around
  `0x1448924..0x144894c`).
- ~20 call sites total, ALL inside the SecuredInt class region (0x2d–0x32) and the wallet
  region (0x77e–0x782). This strongly implies secured ints are used ONLY for currencies,
  not score/timers — useful if you ever want a global secured-int approach.

## High-level wallet accessors (the clean edit points)

| addr | signature | notes |
|---|---|---|
| `0x780a20` | `int GetBananas(this)` | `ldr r0,[r0,8]; bl 0x7e9304; add r0,0x10; mov r1,1; b 0x2d3d30` |
| `0x780a3c` | `int GetTokens(this)`  | `ldr r0,[r0,8]; bl 0x7e9304; add r0,0x20; mov r1,1; b 0x2d3d30` |
| `0x780a58` | `bool CanAfford(this, type, price)` | dispatch by type; token branch at `0x780ab4` |
| `0x780b74` | `Wallet::SetCurrency(this, Currency*)` | type from `GetType` (0x6ec1d4); type1 → token write via 0x7e9304/+0x20/0x2d3cb4 |
| `0x780c20` | another SetCurrency variant (float→int conv path) | also writes via 0x2d3cb4 |
| `0x780cb8` | reset/zero currency variant | writes 0 via 0x2d3cb4 |
| `0x6eb700` | `Currency::GetAmount(desc)` = `add r0,0x60; mov r1,1; b 0x2d3d30` | desc+0x60 |
| `0x6ec1d4` | `Currency::GetType(desc)` = `ldr r0,[r0,0x24]; bx lr` | desc+0x24 |

### `CanAfford` token branch detail (`fcn.00780a58`)
```
0x780a64  cmp r2, 5          ; type switch
0x780a68  beq 0x780a9c       ; type 5 path
0x780a70  beq 0x780ab4 (after cmp r2,1)  ; <-- TOKEN branch (type==1)
...
0x780ab4: mov r5,0; str r5,[sp]; ldr r0,[r0,8]; bl 0x7e9304; add r0,0x20  ; read tokens
0x780acc: mov r1,1; bl 0x2d3d30   ; get amount
0x780ad4: cmp r0, r4; movge r5,1  ; r5 = (tokens >= price)
0x780adc: mov r0, r5; ... ; return r5
```
Forcing `mov r5,1; b 0x780adc` at `0x780ab4` makes token purchases always affordable.

## GetTokens caller map (who reads the balance — for "always rich" approach)

`bl 0x780a3c` call sites (from capstone scan): 0x23e0d4, 0x242e50, 0x294e0c, 0x4118b8,
0x411910, 0x412770, 0x41df04, 0x5674a8, 0x567fb4, 0x689da0, 0x689e6c, 0x6d8934, 0x70ffe0,
0x713f80, 0x7142f8, 0x728c34, 0x7298f4, 0x72b3b0, 0x72bc54, 0x72db68 (HUD/shop/logic).
Patching `GetTokens` itself covers all of them at once → why the shipped patch targets it.

## Transaction / spend path

- `fcn.0028f040` references `NotEnoughTokens` and `TokenValueBelow` — it is the transaction
  RESULT/notification handler (iterates results, fires "not enough" popups). It does its own
  inline balance check; it does NOT call `GetTokens(0x780a3c)`.
- IMPORTANT CAVEAT: because spend logic inlines its own check, "always rich" via GetTokens may
  not be enough for every deduction path. In practice the shipped build (GetTokens=999999 +
  CanAfford-always-true) was confirmed working by the user, but if a specific purchase still
  fails, the spend deduction at the `Wallet::SetCurrency` writers (0x780b74 / 0x780c20) is the
  next lever (e.g., make the token write a no-op, or clamp to a floor).

## Save serialization

- `fcn.006d875c` builds the save string using `"bananas_%d_tokens_%d_size_%d_bytes"` and reads
  the live counts via `GetBananas`/`GetTokens` getters before writing. So whatever the getters
  return is what gets persisted. (Relevant if you want the patched balance to survive saves.)
- Save files: internal storage `SaveFileGame.dat`, `SaveFileCC.dat` (the all-unlocked dump in
  `en/data 5.7.0/...` had everything unlocked; saves are INTERNAL, not external).
