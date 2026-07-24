# BB Instant APK Patch - Changes Made

## Objective
Hardcode the minimum wallet balance threshold to **10** (instead of reading it dynamically from SharedPreferences).

## Files Modified (Smali)

### 1. `com/bigbasket/bbinstant/ui/discoverability/DiscoveryViewModel.smali`
- **Location:** Line ~206 (method `d`)
- **Original:** Reads `"min_balance_required"` from SharedPreferences via `getInt()` with default `Integer.MIN_VALUE`
- **Patched:** Replaced with `const/16 v1, 0xa` (hardcoded value 10)
- **Effect:** During machine discovery, if wallet balance ≤ 10, triggers low-balance flow

### 2. `com/bigbasket/bbinstant/ui/discoverability/DiscoverActivity.smali`
- **Location:** Line ~228 (method `h`)
- **Original:** Same SharedPreferences `getInt("min_balance_required", MIN_VALUE)` pattern
- **Patched:** Replaced with `const/16 v1, 0xa` (hardcoded value 10)
- **Effect:** Discovery screen returns state `2` (low balance) if wallet balance ≤ 10

### 3. `com/bigbasket/bbinstant/ui/machine/s.smali`
- **Location:** Line ~504 (method `h`)
- **Original:** Same SharedPreferences `getInt("min_balance_required", MIN_VALUE)` pattern
- **Patched:** Replaced with `const/16 v1, 0xa` (hardcoded value 10)
- **Effect:** Machine screen returns state `2` (low balance) if wallet balance ≤ 10

## Smali Patch Detail

**Before (all 3 files):**
```smali
invoke-virtual {v1}, Lcom/bigbasket/bbinstant/core/persistance/a;->l()Landroid/content/SharedPreferences;
move-result-object v1
const/high16 v2, -0x80000000
const-string v3, "min_balance_required"
invoke-interface {v1, v3, v2}, Landroid/content/SharedPreferences;->getInt(Ljava/lang/String;I)I
move-result v1
if-gt v0, v1, :cond_2
```

**After (all 3 files):**
```smali
# Patched: hardcoded min_balance = 10
const/16 v1, 0xa
if-gt v0, v1, :cond_2
```

## Build Pipeline

| Step | Tool | Command | Output |
|------|------|---------|--------|
| 1 | apktool 3.0.3 | `apktool d --no-res bb.apk` | `smali_nores/` (smali + raw resources) |
| 2 | manual | Patched 3 smali files | Modified smali |
| 3 | apktool 3.0.3 | `apktool b smali_nores/` | `bb_nores_built.apk` (new classes.dex) |
| 4 | PowerShell | Copy original APK, swap classes.dex | `bb_final.apk` |
| 5 | PowerShell | Remove META-INF/ (old signatures) | Unsigned APK |
| 6 | zipalign | `zipalign -f 4` | `bb_final_aligned.apk` |
| 7 | apksigner | `apksigner sign --ks debug.keystore` | Signed APK (v1+v2+v3) |

## Final Output

- **File:** `C:\Users\42773\Downloads\bbhack\bb_final_aligned.apk`
- **Size:** ~17.2 MB
- **Signature:** v1 (JAR) ✓, v2 ✓, v3 ✓
- **Integrity:** All 1971 original entries preserved, only classes.dex differs

## Notes

- Signed with a **debug keystore** — must uninstall original app before installing
- The `min_balance_required` value was previously fetched from the server via `kwik24/v3/payment/balance` API and stored in SharedPreferences by `PaymentManager.java`
- The first attempt failed because apktool 3.0.3's full resource rebuild dropped 1736 obfuscated resource files; fixed by injecting only the patched DEX into the original APK
