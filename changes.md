# BB Instant APK Patch - Changes Made

## Objective
Hardcode the minimum wallet balance threshold to **₹10** (instead of the server-dictated ₹30), both for UI display and the actual purchase gate.

## Patch v2 (2026-07-27) — PaymentManager.smali

The initial patch (v1) only changed the UI state checks but the app still showed ₹30 and blocked purchases based on the server's `minBalanceAvailable` boolean. Patch v2 fixes this at the source — where the server response is consumed.

### 4. `com/bigbasket/bbinstant/core/payments/newiml/PaymentManager.smali`

#### 4a. Override `getMinbalance()` in balance state constructor (line ~1570)
- **Original:** `invoke-virtual {v8}, ...->getMinbalance()D` → `move-result-wide v15` (server value, e.g. 30.0)
- **Patched:** Added `const-wide v15, 0x4024000000000000L` after move-result (overrides to 10.0)
- **Effect:** All payment state objects store minbalance=10.0 regardless of server response

#### 4b. Force `isMinBalanceAvailable()` to true (line ~1575)
- **Original:** `invoke-virtual {v8}, ...->isMinBalanceAvailable()Z` → `move-result v17` (server boolean, false when balance < 30)
- **Patched:** Added `const/16 v17, 0x1` after move-result (forces true)
- **Effect:** Checkout always allowed — `LowBalance` exception path in `c.java` is unreachable

#### 4c. Override SharedPreferences store value (line ~1678)
- **Original:** `double-to-int v8, v8` (converts server's 30.0 → 30)
- **Patched:** Added `const/16 v8, 0xa` after double-to-int (overrides to 10)
- **Effect:** SharedPreferences `"min_balance_required"` always stores 10

#### 4d. Override fallback max loop value (line ~1958)
- **Original:** `invoke-virtual {v3}, ...->getMinbalance()D` → `move-result-wide v6` (server value)
- **Patched:** Added `const-wide v6, 0x4024000000000000L` after move-result (overrides to 10.0)
- **Effect:** Fallback minbalance calculation uses 10.0 instead of server's 30.0

### Java equivalent of patches:
```java
// In PaymentManager, where balance API response is processed:

// 4a + 4b: Constructor - was:
new b(type, balance, balanceDetails.getMinbalance(), balanceDetails.isMinBalanceAvailable(), ...);
// Now effectively:
new b(type, balance, 10.0, true, ...);

// 4c: SharedPreferences store - was:
int minbalance = (int) balanceDetails.getMinbalance(); // 30
// Now:
int minbalance = 10;

// 4d: Max loop - was:
dMax = Math.max(dMax, it.next().getMinbalance()); // 30.0
// Now:
dMax = Math.max(dMax, 10.0);
```

---

## Patch v1 (2026-07-24) — UI State Checks

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

## Smali Patch Detail (v1 — UI checks)

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

---

## Data Flow (After All Patches)

```
Server API (kwik24/v3/payment/balance)
  └─ returns: { balance: 25, minbalance: 30, minBalanceAvailable: false }
                                    │                        │
                     PATCHED → 10.0 (4a,4d)       PATCHED → true (4b)
                                    │                        │
              SharedPrefs ← 10 (4c) │                        │
                    │               │                        │
         UI checks (v1) ←──────────┘                        │
         "balance ≤ 10?"                                    │
                                                            ▼
                                              Checkout: c.java b() method
                                              bVar.f6175d = true → ALLOW
                                              (LowBalance exception unreachable)
```

## Build Pipeline

| Step | Tool | Command | Output |
|------|------|---------|--------|
| 1 | apktool 3.0.3 | `apktool d --no-res bb.apk` | `smali_nores/` (smali + raw resources) |
| 2 | manual | Patched 4 smali files (7 patch points) | Modified smali |
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
- The first build attempt failed because apktool 3.0.3's full resource rebuild dropped 1736 obfuscated resource files; fixed by injecting only the patched DEX into the original APK
- v1 patch alone was insufficient because the server's `minBalanceAvailable` boolean (not the integer) is what actually gates purchases in `c.java`
- `const/4` cannot address registers >v15; must use `const/16` for v17+ (fixed build error)
