# Aimbot FOV Implementation Plan

## Architecture

**General FOV (RANGE - "Range"):**
- Used in `GetAimingPlayer()` to find targets and set global state (`iTargetPlayer`, `iTargetBone`, `vecTargetBone`)
- Used for FOV circle rendering
- Used for smooth aim activation
- This is the **primary** targeting range

**Silent-Specific FOV (SILENT_RANGE - "Silent Range"):**
- Additional restriction **inside** silent aim hooks
- Silent aim will only trigger if target is within this smaller range
- Acts as a secondary check on top of the general FOV

**Global State (kept as-is):**
- `iTargetPlayer`, `iTargetBone`, `vecTargetBone` remain shared across all aim modes
- Set by `GetAimingPlayer()` using general FOV (RANGE)
- Used by: Silent aim, Smooth aim, Pro aim, Triggerbot

---

## Changes Needed in Aimbot.cpp

### 1. Render() - Line 32-33
**Current:**
```cpp
g_Config.g_Aimbot.iAimbotConfig[weapon][RANGE]
```

**No change needed** - Already uses RANGE constant correctly.

**Reason:** Use general Range (RANGE) for FOV circle rendering.

**Optional Enhancement:** Draw both circles (general Range + Silent Range) when both modes are active.

---

### 2. GetAimingPlayer() - Line 82
**Current:**
```cpp
if (g_Config.g_Aimbot.bAimbot && fCentreDistance >= (float)g_Config.g_Aimbot.iAimbotConfig[weapon][RANGE] * 1.5f)
    continue;
```

**No change needed** - Already uses RANGE constant correctly.

**Reason:** Use general Range (RANGE) for target finding.

---

### 3. hkFireInstantHit() (Silent Aim) - Lines 100-127

**Change 1 - Line 102 (Hitchance):**
```cpp
// Current:
rand() % 100 <= g_Config.g_Aimbot.iAimbotConfig[weapon][SILENT]

// Change to:
rand() % 100 <= g_Config.g_Aimbot.iAimbotConfig[weapon][SILENT_HIT]
```

**Change 2 - Add Silent Range Check (after line 114):**
Add validation that target bone is within Silent Range before applying silent aim:
```cpp
CVector vecBone;
Utils::getBonePosition(pPed, (ePedBones)pAimbot->iTargetBone, &vecBone);

// ADD THIS CHECK:
CVector vecBoneScreen;
Utils::CalcScreenCoors(&vecBone, &vecBoneScreen);
float fDistance = Math::vect2_dist(&pAimbot->vecCrosshair, &vecBoneScreen);
if (fDistance >= (float)g_Config.g_Aimbot.iAimbotConfig[weapon][SILENT_RANGE] * 1.5f)
{
    // Target outside Silent Range, skip silent aim
    Memory::memcpy_safe((void*)0x740B4E, "\x6A\x01\x6A\x01", 4);
    *reinterpret_cast<float*>(0x8D6114) = 5.f;
    return pAimbot->oFireInstantHit(this_, pFiringEntity, pOrigin, pMuzzle, pTargetEntity, pTarget, pVec, bCrossHairGun, bCreateGunFx);
}

pTarget = &vecBone;
// ... rest of silent aim logic
```

**Question:** Should the Silent Range check use `* 1.5f` multiplier or direct comparison?

---

### 4. hkAddBullet() (Silent Sniper) - Lines 129-146

**Change 1 - Line 131 (Hitchance):**
```cpp
// Current:
rand() % 100 <= g_Config.g_Aimbot.iAimbotConfig[34][SILENT]

// Change to:
rand() % 100 <= g_Config.g_Aimbot.iAimbotConfig[34][SILENT_HIT]
```

**Change 2 - Add Silent Range Check (after line 136):**
Similar to hkFireInstantHit, add distance validation:
```cpp
CVector vecBone;
Utils::getBonePosition(pPed, (ePedBones)pAimbot->iTargetBone, &vecBone);

// ADD THIS CHECK:
CVector vecBoneScreen;
Utils::CalcScreenCoors(&vecBone, &vecBoneScreen);
float fDistance = Math::vect2_dist(&pAimbot->vecCrosshair, &vecBoneScreen);
if (fDistance >= (float)g_Config.g_Aimbot.iAimbotConfig[34][SILENT_RANGE] * 1.5f)
{
    // Target outside Silent Range, skip
    Memory::memcpy_safe((void*)0x736212, "\x6A\x01\x6A\x01", 4);
    return pAimbot->oAddBullet(pCreator, weaponType, vecPosition, vecVelocity);
}

vecVelocity = vecBone - vecPosition;
// ... rest of silent sniper logic
```

---

### 5. SmoothAimbot() - Lines 156-225

**Change 1 - Line 213:**
```cpp
// Current:
float fSmoothX = fVecX / (g_Config.g_Aimbot.iAimbotConfig[byteWeapon][SMOOTH] * 2);

// Change to:
float fSmoothX = fVecX / (g_Config.g_Aimbot.iAimbotConfig[byteWeapon][SMOOTH_FACTOR] * 2);
```

**Change 2 - Line 221:**
```cpp
// Current:
float fSmoothZ = (atan2f(fDistZ, vecVector.fZ) - fZ - TheCamera.m_aCams[0].m_fVerticalAngle) / (g_Config.g_Aimbot.iAimbotConfig[byteWeapon][SMOOTH] * 2);

// Change to:
float fSmoothZ = (atan2f(fDistZ, vecVector.fZ) - fZ - TheCamera.m_aCams[0].m_fVerticalAngle) / (g_Config.g_Aimbot.iAimbotConfig[byteWeapon][SMOOTH_FACTOR] * 2);
```

**No FOV check needed:** Smooth aim already uses the general Range from `GetAimingPlayer()`.

---

## Open Questions

1. **Silent Range multiplier:** Should we use `* 1.5f` multiplier for Silent Range checks, or direct comparison with the configured value?

2. **FOV circle rendering:** Should we draw:
   - Only the general Range circle (current behavior)?
   - Both Range and Silent Range circles when both modes are active?
   - Same color or different colors for the two circles?

3. **Side effects:** Any concerns about the tracer line (line 35-36) or other rendering features with these changes?

---

## Commit Message

```
aimbot: use separate FOVs for silent/smooth aim

- Use RANGE as general FOV for target finding
- Add SILENT_RANGE check in silent aim hooks
- Update SMOOTH to SMOOTH_FACTOR, SILENT to SILENT_HIT
- Maintain shared target state (iTargetPlayer/iTargetBone)
```
