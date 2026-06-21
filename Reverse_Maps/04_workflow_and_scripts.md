# 04 — Reusable workflow & scripts

All scripts live in `build/re/`. Paths assume the Windows host used in this project.

## Tool paths

```
adb     C:\Users\Haoxuan\platform-tools\adb.exe
r2      C:\Users\Haoxuan\Desktop\Projects\Rev_3_mn\build\re\r2\radare2-6.1.6-w64\bin\radare2.exe
signer  C:\Users\Haoxuan\Desktop\Projects\Rev_2_conduct\tools\uber-apk-signer-1.3.0.jar
apktool C:\Users\Haoxuan\Desktop\Projects\Rev_2_conduct\tools\apktool_3.0.2.jar
java    (Adoptium JDK 17 on PATH)
python  (3.14 on PATH) + pyelftools, capstone, keystone-engine
```

Target `.so`: `build/dmhm/lib/armeabi-v7a/libDespicableMe.so`
Base APK (offline, fresh, pre-patch): `build/MinionRush-5.7.0-classic-offline-fresh.apk`

## Scripts in build/re/

| script | purpose |
|---|---|
| `recon.py` | dump ELF sections, exported funcs matching keywords, and string anchors in .text/.rodata. Fast (~1s). Start here to find new anchors. |
| `xref.py <hexaddr> ...` | **capstone linear-sweep xref scanner.** Finds DATA xrefs (adr/ldr-literal/add-pc to target) and CALL xrefs (bl/blx/b to target) across all of .text. ~130s for full .text, no r2 analysis needed. Resilient to undecodable words (skips 4 bytes and continues). This was the workhorse for resolving xrefs that r2 missed. |
| `patch.py` | keystone-assemble patches at given vaddr (PC-relative correct) + capstone verify + write `libDespicableMe.patched.so`. Edit the `patches` list to add more. |
| `repack_so.py <base_apk> <out_apk> <entry_name> <repl_file>` | **fast raw-zip swap** of a single entry (e.g. the .so) into a copy of the base APK, preserving all other entries byte-for-byte. ~1s vs minutes for apktool. Removes data-descriptor flag, rewrites local+central headers. |

### Typical r2 one-off (when you need real CFG/decompile)

radare2 with `aaa` takes ~3.5 min on this 24MB binary. Do NOT use `Ps` (project save hangs).
Run a script file instead:
```
& $r2 -q -i script.r2 libDespicableMe.so > out.txt 2>&1
```
Inside script.r2 start with:
```
e bin.relocs.apply=true
e scr.color=0
aaa            ; only if you need xrefs/CFG; skip for pure pdf/pd at a known addr
pdf @ 0xADDR   ; decompile-ish function disasm
axt @ 0xADDR   ; xrefs to addr (requires aaa)
pd N @ 0xADDR  ; linear N instructions
```
For quick single-function disasm at a KNOWN address you can skip `aaa` and just `af @ addr; pdf @ addr` (fast).

## End-to-end rebuild recipe (currency patch)

```powershell
cd C:\Users\Haoxuan\Desktop\Projects\Rev_3_mn\build
# 1. edit re\patch.py patches[], then:
python re\patch.py
# 2. swap patched .so into base APK (raw, ~1s):
python re\repack_so.py "MinionRush-5.7.0-classic-offline-fresh.apk" `
    "re\MR_tokenhack_unsigned.apk" "lib/armeabi-v7a/libDespicableMe.so" `
    "re\libDespicableMe.patched.so"
# 3. sanity check zip:
python -c "import zipfile;z=zipfile.ZipFile(r're\MR_tokenhack_unsigned.apk');print(z.testzip())"
# 4. zipalign + sign (debug key, built-in zipalign):
java -jar C:\Users\Haoxuan\Desktop\Projects\Rev_2_conduct\tools\uber-apk-signer-1.3.0.jar -a re\MR_tokenhack_unsigned.apk
#    -> produces re\MR_tokenhack_unsigned-aligned-debugSigned.apk
```

## Install / test on device

```powershell
$adb="C:\Users\Haoxuan\platform-tools\adb.exe"
& $adb devices -l
# signature-mismatch (debug key vs prior) requires uninstall first:
& $adb uninstall com.gameloft.android.ANMP.GloftDMHM
& $adb install -r re\MR_tokenhack_unsigned-aligned-debugSigned.apk
# launch:
& $adb shell monkey -p com.gameloft.android.ANMP.GloftDMHM -c android.intent.category.LAUNCHER 1
# screenshot (downscale before viewing; raw png is large):
& $adb exec-out screencap -p > shot.png
python -c "from PIL import Image;im=Image.open('shot.png');im.thumbnail((540,1200));im.save('shot_small.png')"
```

Notes:
- adb server sometimes drops the device after long CPU-heavy analysis runs; `adb kill-server`
  then `adb start-server` and replug.
- First launch extracts ~536 MB from `assets/gamedata.zip`; give it time before testing.

## Re-signing with a NON-debug key (to update in place / match an existing install)

If you need the patched APK to update an existing install without uninstall, sign with the
same keystore the prior build used (the project had an AOSP testkey converted to PKCS12,
alias `androidkey`, pass `android`). uber-apk-signer supports `--ks <p12> --ksAlias ... --ksPass ... --ksKeyPass ...`.
(The testkey p12 was under `build/testkey/` during the project; regenerate if missing —
AOSP testkey is public.)
