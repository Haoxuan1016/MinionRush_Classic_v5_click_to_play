# 01 — Overview: target, toolchain, environment

## Target

- **Game:** Minion Rush / Despicable Me, version **5.7.0** (classic offline build; online
  servers are dead, so the game is run fully offline / airplane mode).
- **Package:** `com.gameloft.android.ANMP.GloftDMHM`
- **Main activity:** `com.gameloft.android.ANMP.GloftDMHM.MainActivity`
- **Original APK:** `en/com.gameloft.android.ANMP.GloftDMHM-32f538763ad1f0a4d1f747d388193f32.apk`
- **ABI:** `armeabi-v7a` only (ARM 32-bit). No arm64 lib present.

## Native libraries (`build/dmhm/lib/armeabi-v7a/`)

| file | size | role |
|---|---|---|
| `libDespicableMe.so` | 24.3 MB | **the entire game engine + all gameplay/currency/shop logic** |
| `libgenerator.so` | 0.38 MB | helper |
| `libcrashlytics.so` | 0.20 MB | crash reporting |

`libDespicableMe.so` is the only one that matters for currency/gameplay edits.

## Java/smali layer (essentially a thin shell)

The JNI bridge is tiny — all native methods are in
`smali/com/gameloft/android/ANMP/GloftDMHM/PackageUtils/JNIBridge.smali`:
`NativeInit, NativeOnTouch, NativeSurfaceChanged, NativeOnPause/Resume, NativeKeyAction,
NativeSendKeyboardData, NativeSetReward, NativeAudioFocusChanged, NativeUserMusicIsPlaying,
NativeVolumeAction, SetConnectionType, SetBatteryInfo, SetUserLocation, ...`

- **`NativeSetReward(I,String,String)` is a DEAD STUB** in this build: it marshals JNI args and
  calls an internal function that is just `bx lr` (no-op) with 0 callers. Do not use it to grant
  currency — it does nothing. (This was a confirmed dead-end earlier in the project.)
- Java-layer currency keyword hits are all 3rd-party ad SDKs (vungle/ironsource/fabric), NOT game
  currency. Confirms currency is native-only.

## ELF section layout of `libDespicableMe.so`

```
.dynsym       0x000001f0
.dynstr       0x0000c120
.rel.dyn      0x00034108
.rel.plt      0x000da588
.text         0x000e2210  size 0x01346d14   (file offset == vaddr)
.rodata       0x014b5e40  size 0x00201fe0
.data.rel.ro  0x016b9620
.init_array   0x0171ba34
.got          0x0171c57c
.data         0x01720000
.bss          0x01734550
```

IMPORTANT: For `.text`, **file offset == virtual address** (sh_addr == sh_offset = 0xe2210),
so patching is trivial: file_offset = vaddr. (Verified.)

- Defined dynsym funcs: 1782, objects: 737. Internal gameplay functions are **stripped of names**
  (so radare2 calls them `fcn.XXXXXX`), but C++ RTTI/typeinfo strings and source paths survive in
  `.rodata`, which is how everything was located.

## Build host toolchain (what's installed / used)

- **Python 3.14** (`python` on PATH) + `pip`
- `pyelftools` 0.33, `capstone` 5.0.7, `keystone-engine` (all pip-installed during project)
- **radare2 6.1.6** (downloaded to `build/re/r2/radare2-6.1.6-w64/bin/radare2.exe`)
- **Java 17** (Adoptium) — used to run jars
- **apktool 3.0.2** jar + **uber-apk-signer 1.3.0** jar at
  `C:\Users\Haoxuan\Desktop\Projects\Rev_2_conduct\tools\` (sibling project)
- **adb** at `C:\Users\Haoxuan\platform-tools\adb.exe`
- zipalign: bundled inside uber-apk-signer (BUILT_IN)

## Device

- Xiaomi, model `2509FPN0BC` / `popsicle`, serial `ff882f5a`, non-rooted.
- Game installs and runs offline. First launch extracts bundled `assets/gamedata.zip`
  (~536 MB) into external files dir, then plays fully offline.

## Environment gotchas (Windows / PowerShell)

- Shell is **PowerShell**, not bash. Use `& "C:\full\path.exe" args`. Do NOT use bare `cmd`.
- If a Shell command returns "no exit status"/empty, retry with full permissions (sandbox off).
- adb to device-paths from Git Bash mangles paths (`/data/local/tmp` → Windows path). Use PowerShell.
- radare2's `Ps` (project save) HANGS on this 24 MB binary — do NOT save r2 projects; just
  re-run `aaa` each session (≈3.5 min) or use the capstone scripts (faster, no full analysis).
