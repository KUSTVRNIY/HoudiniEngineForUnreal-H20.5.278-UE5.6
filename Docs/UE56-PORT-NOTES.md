# UE 5.6 porting notes — Houdini Engine for Unreal (H20.5.278)

This document records **what** changed to make the upstream plugin compile and run on Unreal Engine
5.6, **why** each change was needed, and **how** to re-apply the same work to a newer version.

- Base: SideFX *Houdini Engine for Unreal* v2 sources distributed for **Houdini 20.5.278**
  (`HoudiniVersion = "20.5.278"`, HAPI headers 20.5.278 / HAPI 7.0). This is **not** a single public
  upstream tag: the GitHub tag `v2.2.0` targets Houdini 20.5.292, and this base predates the UE 5.5
  landscape-edit-layer API (the port adds the 5.5/5.6 branches).
- Port: **13 source files, +479 / −36 lines** (`Source/` only). Patch: [`UE56-PORT.diff`](UE56-PORT.diff).
  Descriptor edits are *not* part of the patch — see section 11.
- Target: **Unreal Engine 5.6**, built and used on **5.6.1**, Win64.

The port keeps `ENGINE_MAJOR_VERSION` / `ENGINE_MINOR_VERSION` guards around every API change, so the
5.4 and 5.5 code paths are preserved. That makes the patch a usable reference if you have to target
several engine versions from one tree.

---

## 1. Quick map of the changes

| File | +/− | Purpose |
|---|---|---|
| `Source/HoudiniEngine/Private/HoudiniEngineAttributes.cpp` | +20 | HAPI heightfield data accessors |
| `Source/HoudiniEngine/Private/HoudiniEngineAttributes.h` | +3 | declarations for the above |
| `Source/HoudiniEngine/Private/HoudiniEngineUtils.h` | +10 | `TSet<TObjectPtr<T>>` → `TSet<T*>` helper (declared only) |
| `Source/HoudiniEngine/Private/HoudiniInstanceTranslator.cpp` | +77 | recover zero scale / identity rotation from attributes |
| `Source/HoudiniEngine/Private/HoudiniLandscapeSplineTranslator.cpp` | +1/−1 | map value type `TObjectPtr` → raw pointer |
| `Source/HoudiniEngine/Private/UnrealDataTableTranslator.cpp` | +210 | StaticMesh socket export for DataTable inputs |
| `Source/HoudiniEngine/Private/UnrealLandscapeTranslator.cpp` | +58/−14 | landscape edit-layer API + mask commit + logic fixes |
| `Source/HoudiniEngine/Private/UnrealLevelInstanceTranslator.cpp` | −1 | unused local removed |
| `Source/HoudiniEngine/Private/UnrealSkeletalMeshTranslator.cpp` | +2 | physics includes (IWYU) |
| `Source/HoudiniEngineEditor/Private/HoudiniBakeLandscape.cpp` | +58/−8 | landscape bake API migration + output recording |
| `Source/HoudiniEngineEditor/Private/HoudiniToolsEditor.cpp` | +5/−3 | weak texture cache access |
| `Source/HoudiniEngineEditor/Private/HoudiniToolsEditor.h` | +4/−1 | `TMap<FString, TWeakObjectPtr<UTexture2D>>` |
| `Source/HoudiniEngineRuntime/Private/HoudiniEngineRuntime.cpp` | +31/−8 | safe component unregistration (shutdown) |

Totals: **+479 / −36** over 13 files (`git apply --numstat` on `UE56-PORT.diff`). Lines starting with
`+`/`−` that are pure re-indentation are counted too, so a few hunks contain no semantic change.

---

## 2. Landscape edit layers (the biggest API break in 5.6)

UE 5.6 replaced the way landscape edit layers are represented. In 5.4 a landscape layer was an
`FLandscapeLayer` with its own `Name`/`Guid`/`bLocked` fields; from 5.5 the layer became an
`ULandscapeEditLayerBase` object and the fields moved behind it, and 5.6 changed the container accessors
again and moved the edit-layer types into their own header.

### 2.1 Container access

```cpp
// 5.6
for (const FLandscapeLayer& Layer : Landscape->GetLayersConst())
// 5.5
for (const FLandscapeLayer& Layer : Landscape->GetLayers())
// <= 5.4
for (FLandscapeLayer& Layer : Landscape->LandscapeLayers)
```

`Landscape->GetLayerCount()` (≤5.5) is `Landscape->GetLayersConst().Num()` on 5.6.
`GetLayerConst(i)` replaces `GetLayer(i)`; the result is `const`, so anything that used the pointer to
mutate must go through the edit-layer object instead.

### 2.2 Layer name / GUID / lock / visibility

| Concept | 5.6 | 5.5 | ≤5.4 |
|---|---|---|---|
| name | `Layer.EditLayer->GetName()` | `Layer.Name` | `Layer.Name` |
| GUID | `Layer.EditLayer->GetGuid()` | `Layer.Guid` | `Layer.Guid` |
| lock | `Layer.EditLayer->SetLocked(bool, bool)` | `Landscape->SetLayerLocked(idx, bool)` | `Layer.bLocked` |
| visibility | `Landscape->GetEditLayer(idx)->SetVisible(bool, bool)` | `Landscape->SetLayerVisibility(idx, bool)` | `Landscape->SetLayerVisibility(idx, bool)` |
| splines layer | `Landscape->FindEditLayerOfType(ULandscapeEditLayerSplines::StaticClass())` | `FindLayerOfType(...)` | `GetLandscapeSplinesReservedLayer()` |

Because `FLandscapeLayer` is now `const` in the iteration, `HoudiniBakeLandscape.cpp` stores
`const FLandscapeLayer*` and computes the GUID once into `BakedLayerGuid`, which is then passed to
`ClearLayer`, `ClearPaintLayer` and `FScopedSetLandscapeEditingLayer`.

### 2.3 Header

`#include "LandscapeEditLayer.h"` is required from 5.5 onward (`ULandscapeEditLayerSplines`,
`SetLocked`/`SetVisible` declarations). It is guarded by `ENGINE_MINOR_VERSION >= 5`.

### 2.4 Mask heightfield must always be committed

`SendTargetLayersToHoudini()` only created the mask when merged paint layers were exported. Without a
committed mask the heightfield volume info was not initialised, and landscape exports could end up with
empty or invalid layers. The port always creates and commits a default mask:

```cpp
if (Options.bExportMergedPaintLayers)
    bSuccess &= SendCombinedTargetLayersToHoudini(...);
else
{
    InitDefaultHeightfieldMask(HeightFieldVolumeInfo, MaskId);
    ApplyAttributesToHeightfieldNode(MaskId, PartId, LandscapeProxy);
    HOUDINI_CHECK_ERROR_RETURN(FHoudiniEngineUtils::HapiCommitGeo(MaskId), false);
}
```

### 2.5 Other landscape fixes

- `CreateInputNodeForLandscapeObject`: the full-landscape and selection-only calls were put into explicit
  `{ }` branches (cosmetic) and the selected-component collection is passed through a
  `TSet<ULandscapeComponent*>` conversion the callee expects.
- `CreateHeightfieldFromLandscape`: the `return false;` after a failed `SendTargetLayersToHoudini` is only
  **re-indented** in the same hunk — the guard already existed upstream, this is not a behaviour fix.
- No whitespace noise: the port does not add or remove BOMs or trailing blank lines in this file. (The
  original working copy had a UTF-8 BOM and an extra EOF blank line; both were removed before the patch
  in `UE56-PORT.diff` was generated, so they do not appear in it.)

---

## 3. UObject lifetime / GC and shutdown assertions

UE 5.6 tightened `TObjectPtr` / `TWeakObjectPtr` behaviour. Two shutdown crashes appeared:

### 3.1 `TArray<TWeakObjectPtr<…>>::Find()`

`FHoudiniEngineRuntime::UnRegisterHoudiniComponent(UHoudiniAssetComponent*)` used
`RegisteredHoudiniComponents.Find(HAC)`. Comparing a `TWeakObjectPtr` can resolve through
`GUObjectArray.IndexToObject(INDEX_NONE)` once components have been removed, and the resulting
`Assertion failed: Index >= 0` kills the editor on exit.

The port walks the array manually and compares raw pointers:

```cpp
int32 FoundIdx = -1;
for (int32 nIdx = 0; nIdx < RegisteredHoudiniComponents.Num(); nIdx++)
{
    const TWeakObjectPtr<UHoudiniAssetComponent> Ptr = RegisteredHoudiniComponents[nIdx];
    if (!Ptr.IsValid(true, true))
        continue;

    UHoudiniAssetComponent* CurrentHAC = Ptr.GetEvenIfUnreachable();
    if (CurrentHAC && CurrentHAC == HAC) { FoundIdx = nIdx; break; }
}
```

`IsValid(true, true)` passes `bEvenIfPendingKill = true` (objects already pending destruction still count)
and `bThreadsafeTest = true` — the 5.6 signature is `IsValid(bool bEvenIfPendingKill, bool bThreadsafeTest)`;
`GetEvenIfUnreachable()` then returns the object even after it has been marked unreachable. Both arguments
are deliberate: during shutdown objects are unreachable / pending-kill but still alive enough to compare.
The method also bails out early when the registry is empty, and `UnRegisterHoudiniComponent(const int32&)`
now validates the index and guards `GetAssetId() >= 0` before queueing node deletion.

### 3.2 Tool icon cache

`FHoudiniToolsEditor::CachedTextures` was `TMap<FString, UTexture2D*>`. At module shutdown the textures
can already be gone from the UObject array, so `IsValid(Texture)` was not enough and `RemoveFromRoot()`
asserted. It is now:

```cpp
TMap<FString, TWeakObjectPtr<UTexture2D>> CachedTextures;
```

with `.Get()` on reads and `IsValid(CachedTexture) && CachedTexture->IsValidLowLevelFast()` before
`RemoveFromRoot()`.

### 3.3 TObjectPtr set conversion helper

`HoudiniEngineUtils.h` gained a tiny template for `TSet<TObjectPtr<T>>` → `TSet<T*>` conversion:

```cpp
template <typename T>
static TSet<T*> RemoveObjectPtr(const TSet<TObjectPtr<T>>& In);
```

The helper is declared and defined in the header but has **no call sites** in this tree yet — it is
available for code that has to hand a `TSet<T*>` to a pre-5.6 API. Keep it or drop it; nothing depends
on it.

---

## 4. Landscape spline output map type

`UHoudiniLandscapeSplineTargetLayerOutput::GetLayerOutputs()` returns a map keyed by edit-layer name.
Upstream code bound it as `TMap<FName, TObjectPtr<UHoudiniLandscapeSplineTargetLayerOutput>>&`, which no
longer matches the property type on 5.6 (`TObjectPtr` vs raw pointer mismatch in the UPROPERTY map).
The port binds it as a raw-pointer map:

```cpp
TMap<FName, UHoudiniLandscapeSplineTargetLayerOutput*>& LayerOutputs = InOutputObject.GetLayerOutputs();
```

---

## 5. HAPI heightfield data access

`FHoudiniHapiAccessor` had no heightfield reader, which the landscape mask commit needs. Two methods were
added (`HoudiniEngineAttributes.h/.cpp`):

```cpp
bool GetHeightFieldData(TArray<float>& Values, int IndexCount);
bool GetHeightFieldDataViaSession(const HAPI_Session* Session, const HAPI_AttributeInfo& AttributeInfo,
                                  float* Results, int IndexStart, int IndexCount) const;
```

They wrap `FHoudiniApi::GetHeightFieldData` and use `SetNumUninitialized` for the destination array. The
`ViaSession` variant exists so callers that already hold a session (and run outside the game thread) do
not have to re-fetch it. `H_SCOPED_FUNCTION_TIMER()` keeps the call visible in traces.

---

## 6. Instance transform recovery (custom fix, not upstream)

**Problem.** For attribute instancers emitted by multi-part HDAs (e.g. walls + roofs), HAPI could report
`zero` scale or an identity rotation for one part's instances while the point attributes held correct
values. Result: invisible or unrotated instances in the UE level.

**Fix.** After `HapiGetInstanceTransforms` has translated the HAPI transforms, it now reads the point
attributes and repairs only broken entries:

- `scale` (`HAPI_UNREAL_ATTRIB_INSTANCE_SCALE`, float3, point) — replaces a nearly-zero scale,
  swapping Y/Z for Houdini's Y-up convention the same way `TranslateHapiTransform` does.
- `orient` (float4, point) — replaces an identity rotation using
  `FQuat(x, z, y, -w)`, matching `TranslateHapiTransform`'s W inversion, then normalises. Entries whose
  quaternion is degenerate or identity are skipped.

Both passes are guarded by attribute-count checks (`count == OutInstancerUnrealTransforms.Num()`) and log
a warning with the recovered count and `Geo`/`Part` ids, so the situation is diagnosable from the log
instead of being a silent visual bug.

---

## 7. DataTable socket export ("props MVP", custom feature)

`FUnrealDataTableTranslator::CreateInputNodeForDataTable` now exports StaticMesh sockets of the first
mesh-typed column as point attributes, letting HDAs place props on sockets per row.

- Column discovery: first `FSoftObjectProperty` / `FObjectProperty` whose `PropertyClass` derives from
  `UStaticMesh`.
- Row mesh resolution: soft object (synchronous load) → object property → `DataTableUtils::GetPropertyValueAsString`
  + `StaticLoadObject` fallback.
- Socket enumeration: `UStaticMesh::Sockets`, with a `TObjectIterator<UStaticMeshSocket>` fallback for
  editor builds where the array is empty, de-duplicated via `AddUnique`.
- Attributes added with `FHoudiniApi::AddAttribute` + `FHoudiniHapiAccessor::SetAttributeData`:

| Attribute | Storage | Tuple | Content |
|---|---|---|---|
| `unreal_socket_count` | int | 1 | per-row socket count |
| `unreal_socket_<i>_name` | string | 1 | socket name |
| `unreal_socket_<i>_pos` | float | 3 | `RelativeLocation` (cm, mesh local) |
| `unreal_socket_<i>_rot` | float | 3 | `RelativeRotation` as pitch/yaw/roll degrees |

`<i>` runs up to the maximum socket count across rows; missing sockets are zero/empty-filled so every
attribute has `count == NumRows`. Logging uses the `[props MVP]` prefix (row name, mesh, socket count,
resolution failures) to make bad rows obvious.

This is an MVP by design: one mesh column, editor-side sockets only, re-exported on input cook.

---

## 8. Includes and small compile fixes

- `UnrealSkeletalMeshTranslator.cpp`: explicit `#include "PhysicsEngine/SkeletalBodySetup.h"` and
  `"PhysicsEngine/AggregateGeom.h"`. 5.6 no longer pulls them in transitively through
  `PhysicsAsset.h`, so `USkeletalBodySetup`/`FKAggregateGeom` became incomplete types.
- `UnrealLevelInstanceTranslator.cpp`: removed an unused `FVector3f Position;` — shadowing/`-Werror`
  clean builds reject it.
- `UnrealDataTableTranslator.cpp`: explicit `#include "UObject/UObjectIterator.h"`, `"Engine/StaticMesh.h"`
  and `"Engine/StaticMeshSocket.h"` for the socket export. `HoudiniEngineAttributes.cpp` gained **no** new
  includes — its change is the two heightfield accessor methods only.

---

## 9. How to re-apply to a future engine version

1. Diff upstream against the ported tree to see the shape of the work:
   the patch in `UE56-PORT.diff` uses `a/upstream/…` (the pristine 20.5.278 base sources) and
   `b/ported/…` (this repository's `Source/` tree).
2. Start from the **new** upstream sources, not from this tree, and re-apply section by section with the
   `ENGINE_MINOR_VERSION` guards updated for the new target.
3. Expect the landscape edit-layer block (section 2) to need the most attention again — that API has
   changed in three consecutive engine versions (5.4 → 5.5 → 5.6).
4. Keep the UObject-lifetime fixes (section 3): they are engine-version-independent robustness fixes.
5. Build with the exact Houdini version whose HAPI headers you drop in, then run a smoke test:
   import HDA → cook → parameter change → bake, plus landscape export if you use it.

---

## 10. Verification status of this port

These notes describe a port that was compiled and used in a real project; **no CI pipeline, build log or
test log is stored in this repository**, so treat the statements below as author-side observations:

- The source tree compiles for `Win64 / Development Editor` with UE 5.6.1 and Houdini 20.5.278 (the
  build output was used in the project the port was made for; build artifacts are not committed).
- In that project the plugin loads, the *Houdini Engine* menu appears, sessions start against Houdini
  20.5.278, and HDA cook / parameter-change / bake workflows run.
- The landscape edit-layer paths compile against the 5.6 API and the mask/bake fixes are in place; they
  were exercised less than the mesh/instance workflows, so treat them as the most likely area for a
  remaining porting defect.
- Nothing in the port was validated on UE 5.5, 5.7, Linux/macOS, or in a packaged/Shipping build.

---

## 11. Plugin descriptor changes (not part of the patch)

`HoudiniEngine.uplugin` lives outside `Source/`, so it is **not** included in `UE56-PORT.diff`. Its
changes are:

| Field | This repository | Why |
|---|---|---|
| `Version` | `20050278` | unchanged — Houdini build number, matches the HAPI headers |
| `VersionName` | `2.2.0 - H20.5.278 (UE5.6 port)` | marks the port explicitly in *Edit → Plugins* (SideFX's naming scheme for this build is `2.0 - H20.5.<build>`) |
| `Installed` | `false` | source distribution: Unreal must compile the modules in the target project; upstream ships it as `true` together with prebuilt binaries |
| `CreatedBy`, URLs, modules | unchanged | SideFX attribution and the three-module layout are untouched |

If you re-base this port on a newer upstream drop, re-apply these three fields by hand after applying
`UE56-PORT.diff`.
