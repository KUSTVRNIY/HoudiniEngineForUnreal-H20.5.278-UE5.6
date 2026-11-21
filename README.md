<p align="center">
  <img src="Resources/Icon128.png" width="160" alt="Houdini Engine for Unreal">
</p>

# Houdini Engine for Unreal — H20.5.278 port for Unreal Engine 5.6

![Houdini](https://img.shields.io/badge/Houdini-20.5.278-orange)
![Unreal Engine](https://img.shields.io/badge/Unreal%20Engine-5.6-blue)
![Platform](https://img.shields.io/badge/platform-Win64-lightgrey)
![Status](https://img.shields.io/badge/status-working-brightgreen)

> ### ⚠️ READ THIS FIRST — THIS PLUGIN IS VERSION-LOCKED
>
> **It is built and tested for exactly these two versions; only this combination is supported:**
>
> | Component | Required version |
> |---|---|
> | **Houdini / Houdini Engine** | **20.5.278** (HAPI 7.0) |
> | **Unreal Engine** | **5.6** — the port was built and used on **5.6.1** |
> | **Platform** | **Windows 64-bit** |
>
> The Houdini version is **not a recommendation**, it is compiled into the plugin:
> `Source/HoudiniEngine/HoudiniEngine.Build.cs` hard-codes `20.5.278`, the HAPI headers in
> `Source/HoudiniEngine/Public/HAPI/` are generated for 20.5.278, and the runtime refuses a session
> whose Houdini Engine major/minor version does not match.
> If you have a different Houdini build or a different Unreal version, use the official SideFX build
> for your versions instead (see [section 15](#15-updating-the-plugin-to-other-versions)).

This repository is a **working UE 5.6 port** of the SideFX *Houdini Engine for Unreal v2* plugin, based
on the v2 sources distributed for **Houdini 20.5.278**. It is the full plugin folder — drop it into your
project's `Plugins/` directory; the editor builds the three modules once on first launch (Visual Studio
2022 required for that one step, see [section 8](#8-building-from-source)).

It is **unofficial** and **not affiliated with or endorsed by Side Effects Software Inc.**
The plugin code is SideFX's; this fork only adds the UE 5.6 compatibility work and a few fixes
described in [section 9](#9-what-was-changed-for-ue-56-this-fork) and
[section 10](#10-custom-fixes-and-features-added-in-this-fork).

---

## Table of contents

1. [What this plugin does](#1-what-this-plugin-does)
2. [Why the strict version lock](#2-why-the-strict-version-lock)
3. [Requirements](#3-requirements)
4. [Installation (drop-in)](#4-installation-drop-in)
5. [Configuring the Houdini location](#5-configuring-the-houdini-location)
6. [Verifying it works](#6-verifying-it-works)
7. [Quick start: HDA workflow](#7-quick-start-hda-workflow)
8. [Building from source](#8-building-from-source)
9. [What was changed for UE 5.6 (this fork)](#9-what-was-changed-for-ue-56-this-fork)
10. [Custom fixes and features added in this fork](#10-custom-fixes-and-features-added-in-this-fork)
11. [Feature overview](#11-feature-overview)
12. [Troubleshooting](#12-troubleshooting)
13. [Known limitations](#13-known-limitations)
14. [Repository layout](#14-repository-layout)
15. [Updating the plugin to other versions](#15-updating-the-plugin-to-other-versions)
16. [Licence and credits](#16-licence-and-credits)

---

## 1. What this plugin does

Houdini Engine for Unreal brings Houdini's procedural workflow into Unreal Engine through
**Houdini Digital Assets (HDAs)**. You import an `.hda`/`.otl` into the Content Browser, drag it into
the level, and the plugin cooks it through a Houdini Engine session and turns the result into Unreal
assets — static meshes, instances, landscapes, splines, geometry collections, data tables and more —
without baking by hand.

In practice:

- You expose parameters on an HDA, tweak them in the **Details** panel, and geometry updates live in
  the editor.
- Unreal assets can be fed back into Houdini as **inputs** (static meshes, actors, landscapes,
  splines, data tables, world/level instances, skeletal meshes, geometry collections).
- The result can be **baked** into normal Unreal assets when you are done.
- PDG (Houdini's procedural dependency graph) and **Session Sync** with a live Houdini session are
  supported, as in upstream.

This build makes all of that work on **UE 5.6**, where a number of landscape/edit-layer and
UObject-lifetime APIs changed and the upstream sources this fork is based on no longer compile.

---

## 2. Why the strict version lock

Four separate locks force Houdini 20.5.278 and UE 5.6 — this is why the versions in the table above
are non-negotiable:

1. **Houdini build lock — build time.** `Source/HoudiniEngine/HoudiniEngine.Build.cs` contains
   `string HoudiniVersion = "20.5.278";`. It resolves the Houdini install directory from the Windows
   registry (`Side Effects Software\Houdini Engine 20.5.278`, then `...\Houdini 20.5.278`), falls back
   to `C:/Program Files/Side Effects Software/Houdini 20.5.278`, and bakes the resulting path into the
   module (`HOUDINI_ENGINE_HFS_PATH_DEFINE`). With no matching install the build still **succeeds**: it
   only logs
   `Houdini Engine : Houdini 20.5.278 could not be found. Houdini Engine will not be available in this build.`
   and leaves the HFS define empty, so HAPI cannot be loaded at runtime and the plugin never initialises.
2. **HAPI lock — compile time.** `Source/HoudiniEngine/Public/HAPI/HAPI_Version.h` is generated for
   Houdini **20.5.278**, Houdini Engine (HAPI) version **7.0**, API version 0. Library calls are
   compiled against that exact HAPI.
3. **HAPI lock — run time.** When a session starts, `FHoudiniEngine::InitializeHAPISession`
   (`Source/HoudiniEngine/Private/HoudiniEngine.cpp`) queries the running Houdini Engine for its HAPI
   major/minor/API version (`HAPI_ENVINT_VERSION_HOUDINI_ENGINE_*`). On a major/minor mismatch it logs
   `Starting up the Houdini Engine module failed: built and running versions do not match.` together
   with the defined-vs-running numbers, and aborts; an API-only mismatch is a warning. This compares the
   *HAPI engine* version (7.0 / api 0), not the Houdini build number.
4. **Engine lock — source level.** The port targets UE 5.6 APIs (landscape edit layers, `TObjectPtr`
   handling, IWYU includes). On 5.7 further source edits are required; the preserved 5.5 branch has not
   been compiled or tested.

Mixing versions is the most common failure mode. Other Houdini builds (e.g. 20.5.370) can still pass
the runtime HAPI check because they share HAPI 7.0 / api 0, but this plugin is compiled against the
**20.5.278** headers and has only been validated on 20.5.278 — treat anything else as unsupported. Only
UE 5.6 is validated for this source; 5.5 and 5.7 would need further work. Keep both versions exactly as
listed.

---

## 3. Requirements

| Item | Requirement |
|---|---|
| OS | Windows 10 / 11, 64-bit |
| Unreal Engine | **5.6** (built and used on **5.6.1**) |
| Houdini | **Houdini 20.5.278** *or* **Houdini Engine 20.5.278** installed locally |
| Houdini licence | a valid Houdini licence that allows Houdini Engine sessions (Indie / Core / FX / Engine). Houdini Indie sessions are non-commercial only |
| Build tools | **Visual Studio 2022** with the *Game development with C++* workload and a Windows SDK compatible with UE 5.6 — this repository ships **source only**, so the modules are compiled once on first launch |
| Disk | ~15 MB for the plugin folder; the first build adds `Binaries/` + `Intermediate/` (a few hundred MB, regenerable) |

The plugin resolves the Houdini installation automatically. It does **not** bundle Houdini — HAPI is
loaded from your Houdini installation at runtime.

---

## 4. Installation (drop-in)

### 4.1 Put the plugin into the project

1. Download — either the **release archive**
   [`HoudiniEngine-H20.5.278-UE5.6.zip`](https://github.com/KUSTVRNIY/HoudiniEngineForUnreal-H20.5.278-UE5.6/releases/latest)
   (already contains the `HoudiniEngine/` folder, nothing to rename) or the repository itself via
   **Code → Download ZIP**.
2. Extract it. You get a folder containing `HoudiniEngine.uplugin`, `Source/`, `Content/`, `Resources/`, …
3. When using the repository ZIP, rename the extracted folder (e.g.
   `HoudiniEngineForUnreal-H20.5.278-UE5.6-main`) to **`HoudiniEngine`** — the folder name must match
   the `.uplugin` name.
4. Copy it into your project:

```
YourProject/
  YourProject.uproject
  Plugins/
    HoudiniEngine/
      HoudiniEngine.uplugin
      Config/
      Content/
      Resources/
      Source/
      README.md
      LICENSE.md
      NOTICE.md
```

> For Unreal **5**, project plugins only — do not put it inside the engine's `Engine/Plugins`.
> If your project has no `Plugins` folder, create it next to the `.uproject`.

`Binaries/` and `Intermediate/` are **not** part of the repository — Unreal creates them during the
first build (see [section 4.3](#43-building-the-modules-on-first-launch) and
[section 8](#8-building-from-source)).

### 4.2 Enable the plugin

The descriptor has `"EnabledByDefault": true`, so a fresh editor launch picks it up automatically.
If you want to be explicit, enable it in `YourProject.uproject`:

```json
{
	"FileVersion": 3,
	"EngineAssociation": "5.6",
	"Plugins": [
		{ "Name": "HoudiniEngine", "Enabled": true }
	]
}
```

or use **Edit → Plugins**, search for **Houdini Engine**, tick it and restart the editor when asked.
The plugin loads in the editor; the `HoudiniEngineRuntime` module keeps runtime types available.

### 4.3 Building the modules on first launch

This repository contains the plugin **source only** (no `Binaries/`). The three modules compile to
`Binaries/Win64/UnrealEditor-HoudiniEngine*.dll` when the project is first opened:

| Module | Type | What it is |
|---|---|---|
| `HoudiniEngine` | Editor | editor-side translators, cooking |
| `HoudiniEngineEditor` | Editor | menus, details panels, baking UI |
| `HoudiniEngineRuntime` | Runtime | assets, components, settings |

What to expect:

1. Install **Visual Studio 2022** with the *Game development with C++* workload and a Windows SDK
   compatible with UE 5.6 (the *Desktop development with C++* workload plus the SDK component is enough).
2. Open your project. Unreal detects the enabled plugin without modules and asks to build them →
   **Yes**. The first build of ~204k lines of plugin C++ (excluding the bundled HAPI headers) takes a few
   minutes on a normal desktop.
3. Subsequent launches reuse the compiled modules in `Plugins/HoudiniEngine/Binaries/`. You only build
   again after changing the plugin source, upgrading the engine, or after a clean.

The descriptor ships with `"Installed": false` so Unreal treats it as a project plugin that must be
built locally; that is also why no rebuild prompt appears for a plugin marked as installed. If you want
to debug the plugin, build the `Development Editor` configuration and keep the generated `.pdb` files
locally (they are not committed — symbols are large and machine-specific).

---

## 5. Configuring the Houdini location

### 5.1 Automatic detection (normal case)

If Houdini 20.5.278 is installed with the default installer, nothing needs configuring. The search
order is:

1. Registry: `HKEY_LOCAL_MACHINE\SOFTWARE\Side Effects Software\Houdini Engine 20.5.278` → `InstallPath`
2. Registry (32-bit view): `...\WOW6432Node\Side Effects Software\Houdini Engine 20.5.278`
3. `C:\Program Files\Side Effects Software\Houdini Engine 20.5.278`
4. Registry: `...\Side Effects Software\Houdini 20.5.278` → `InstallPath`
5. Registry (32-bit view): `...\WOW6432Node\Side Effects Software\Houdini 20.5.278`
6. `C:\Program Files\Side Effects Software\Houdini 20.5.278`
7. Fallback: the registry's `ActiveEngineVersion`, if its major/minor match 20.5

### 5.2 Custom location (non-default install path)

The custom location is **not** the Houdini install root — it must be the folder that directly contains
the HAPI library `libHAPIL.dll`, which on a normal Windows install is the **`bin`** subfolder:

**Project Settings → Plugins → Houdini Engine → Houdini Location**:

- tick **Use custom Houdini location (requires restart)**,
- set **Custom Houdini location** to
  `C:/Program Files/Side Effects Software/Houdini 20.5.278/bin` (the folder containing `libHAPIL.dll`),
- restart the editor (the setting is `ConfigRestartRequired`).

> If the folder does not contain `libHAPIL.dll`, the plugin shows a message box
> *"libHAPIL.dll was not found in &lt;path&gt;" / "Invalid Custom Location Specified, resetting."* and
> clears the setting — a path pointing at the install root is rejected with that message, not silently
> accepted.
> The check and the load are both a plain `&lt;path&gt;/libHAPIL.dll`, no `bin` is appended.

The equivalent `DefaultEngine.ini` entry (project `Config/` folder):

```ini
[/Script/HoudiniEngineRuntime.HoudiniRuntimeSettings]
bUseCustomHoudiniLocation=True
CustomHoudiniLocation=(Path="C:/Program Files/Side Effects Software/Houdini 20.5.278/bin")
```

Notes:

- The same panel lets you pick the **Houdini Executable** used for Session Sync, a **Custom Houdini
  Home Location** (what `$HOME` resolves to inside Houdini), `.env` files, and the OTL/DSO search paths.
- **Session Type** (Project Settings → Plugins → Houdini Engine → Session) selects how Houdini Engine is
  reached: *Named pipe* (default), *TCP socket*, *Shared Memory Buffer*, *In Process* (experimental) or
  *None* (no session — nothing cooks).

---

## 6. Verifying it works

1. Open the project. The main menu bar gains a new **Houdini Engine** entry (in the level editor it sits
   next to **Edit** / **Window**) with the sections *Session*, *PDG*, *Plugin*, *File*,
   *Help And Support* and *Actions*.
2. **Edit → Plugins → Houdini Engine**: the version string must read
   **`2.2.0 - H20.5.278 (UE5.6 port)`**, with version `20050278`.
3. **Window → Output Log**, filter on `Houdini`: session start-up logs through `LogHoudiniEngine`. A HAPI
   mismatch prints
   `Starting up the Houdini Engine module failed: built and running versions do not match.` followed by
   the defined-vs-running numbers — that means the Houdini being run is not HAPI 7.0 (Houdini 20.5.x).
   A healthy start-up shows no such error.
4. Functional test without any of your own content: `Content/Examples/hda/pig_head_subdivider_v01.hda`
   — drag the `.hda` file straight into a Content Browser folder (HDA import is drag-and-drop; there is
   no separate *Import HDA* menu command), then drag the resulting asset into the level. A subdivided
   pig head should cook. Changing a parameter in **Details** re-cooks it.
5. **Houdini Engine → Create Session** (in the *Session* section; also *Connect*, *Stop*, *Restart* and
   *Open Houdini Session Sync…*) to confirm a Houdini Engine session starts and the licence is accepted.

---

## 7. Quick start: HDA workflow

1. **Import** — drag an `.hda` / `.otl` file into the Content Browser. The plugin creates a
   `HoudiniAsset` asset (plus generated assets as needed).
2. **Place** — drag the asset into the viewport / level. This spawns a *Houdini Asset Actor* with a
   *Houdini Asset Component*.
3. **Cook** — geometry appears after the cook completes (cooking runs on a background thread). Use
   **Houdini Engine → Recook All / Recook Selection** (and *Rebuild All / Rebuild Selection* for a full
   rebuild); the **Show Cook Logs** button in the *Houdini Engine* category of the component's
   **Details** panel opens the cook log window.
4. **Parameters** — select the actor; HDA parameters appear in **Details** grouped as in Houdini.
   Changing one triggers a recook.
5. **Inputs** — on the component's *Inputs* tab you can feed Unreal content into the HDA: static
   meshes, actors from the level, landscapes, splines, data tables, world/level instances, skeletal
   meshes, geometry collections. Inputs can be *world* or *asset* based.
6. **Bake** — when you like the result, bake it to plain Unreal assets (static mesh, actors, landscape
   layers, …). The menu entries are **Bake And Replace All Houdini Assets** and **Bake And Replace
   Selection** (*Actions* section); per-output bake buttons and the bake name live in the component's
   *Outputs* details.
7. **Session Sync** — **Houdini Engine → Open Houdini Session Sync…** starts Houdini with Session Sync
   and syncs viewport/parameters both ways; the *Sync Viewport* submenu controls the direction.

Bundled examples worth opening: `Content/Examples/` (`hda`, `Chaos`, `Cpp`, `Python`, `Maps`,
`EUA`/`EUW`) and `Content/NodeSync/` (`KineFXToUnreal.hda`, `UnrealToKineFX.hda`).

---

## 8. Building from source

This is required on first install (the repository ships **source only**) and whenever you change the
plugin code or the engine version.

**Prerequisites:** UE 5.6 (launcher or source build) + Visual Studio 2022 with *Game development with
C++* and a Windows SDK compatible with UE 5.6; Houdini 20.5.278 installed (the build resolves the Houdini
HFS path and bakes it into the modules).

### Option A — let Unreal build the missing modules (simplest)

1. Place the plugin as in [section 4.1](#41-put-the-plugin-into-the-project).
2. Open the project. Unreal detects the enabled plugin without modules and offers to build them →
   **Yes**. (The descriptor is `"Installed": false`, so this prompt is expected.)
3. If it does not prompt and only reports a load failure, delete `Plugins/HoudiniEngine/Binaries/` and
   `Plugins/HoudiniEngine/Intermediate/`, reopen the project, and make sure the VS 2022 C++ workload and
   the Windows SDK are installed.

### Option B — build from the command line

```bat
"C:\Program Files\Epic Games\UE_5.6\Engine\Build\BatchFiles\Build.bat" YourProjectEditor Win64 Development ^
    -Project="D:\Projects\YourProject\YourProject.uproject" -WaitMutex -FromMsBuild
```

The three plugin modules are built as part of your project's editor target. A successful build logs:

```
Houdini Engine : Found Houdini in C:/Program Files/Side Effects Software/Houdini 20.5.278
```

If instead it logs `Houdini Engine : Houdini 20.5.278 could not be found. Houdini Engine will not be
available in this build.`, compilation still **succeeds**, but HFS is left empty in the built modules and
the plugin cannot load HAPI at runtime. Install 20.5.278, or override `HoudiniVersion` / `HFSPath` in
`Source/HoudiniEngine/HoudiniEngine.Build.cs` and rebuild.

### Option C — Visual Studio

Right-click `YourProject.uproject` → **Generate Visual Studio project files**, open the `.sln`, select
the `Development Editor | Win64` configuration, build. Output DLLs land in
`Plugins/HoudiniEngine/Binaries/Win64/`.

---

## 9. What was changed for UE 5.6 (this fork)

Base: the SideFX *Houdini Engine for Unreal* v2 sources distributed for **Houdini 20.5.278**
(`HoudiniVersion = "20.5.278"` in `Build.cs`, HAPI headers generated for 20.5.278, HAPI 7.0 / api 0).
This base is **not identical to a public upstream tag**: the upstream GitHub tag `v2.2.0` targets Houdini
20.5.292, and this base predates the UE 5.5 landscape-edit-layer API (the port adds the 5.5/5.6 branches).

Source diff against that base: **13 files, +479 / −36 lines**
([`Docs/UE56-PORT.diff`](Docs/UE56-PORT.diff)). The patch covers `Source/` only — the plugin-descriptor
edits (version marker, `Installed: false`) are listed in [`NOTICE.md`](NOTICE.md) and
[`Docs/UE56-PORT-NOTES.md`](Docs/UE56-PORT-NOTES.md), not in the diff. Per-change rationale:
[`Docs/UE56-PORT-NOTES.md`](Docs/UE56-PORT-NOTES.md).

| Area | Files | What changed |
|---|---|---|
| Landscape edit layers | `UnrealLandscapeTranslator.cpp`, `HoudiniBakeLandscape.cpp` | Migrated to the UE 5.6 landscape edit-layer API: `GetLayersConst()`, `FLandscapeLayer::EditLayer->GetName()/GetGuid()`, `SetLocked()`, `SetVisible()`, `FindEditLayerOfType()`, `LandscapeEditLayer.h`. Version guards `ENGINE_MINOR_VERSION >= 5 / >= 6` keep 5.4/5.5 paths intact. |
| Landscape heightfield mask | `UnrealLandscapeTranslator.cpp` | The mask heightfield is now always created and committed, also when *merged paint layers* export is off — HAPI's volume info does not work correctly without a committed mask. |
| Landscape bake | `HoudiniBakeLandscape.cpp` | Baked landscape path is written back to the output object; layer lock/visibility/clear calls use the 5.6 edit-layer API; splines reserved layer resolved via `FindEditLayerOfType()`. |
| Landscape spline output | `HoudiniLandscapeSplineTranslator.cpp` | `TMap` value type fixed (`TObjectPtr<…>` → `…*`) to match the UPROPERTY map type on 5.6. |
| UObject lifetime / GC | `HoudiniEngineRuntime.cpp`, `HoudiniToolsEditor.h/.cpp`, `HoudiniEngineUtils.h` | Shutdown crashes fixed: manual raw-pointer search instead of `TArray<TWeakObjectPtr>::Find()` in `UnRegisterHoudiniComponent()`, `GetEvenIfUnreachable()`, `IsValid(true, true)`; tool icon cache switched to `TWeakObjectPtr` with `IsValidLowLevelFast()`; helper for `TSet<TObjectPtr<T>>` conversion (declared; not called anywhere yet). |
| Instance outputs | `HoudiniInstanceTranslator.cpp` | recovers per-instance **scale** and **rotation** from the Houdini point attributes when HAPI reports zero scale / identity rotation (multi-part attribute instancers). |
| DataTable inputs | `UnrealDataTableTranslator.cpp` | exports **StaticMesh sockets** as point attributes so HDAs can place props on sockets ("props MVP"). |
| Heightfield access | `HoudiniEngineAttributes.h/.cpp` | added `GetHeightFieldData()` / `GetHeightFieldDataViaSession()` accessors. |
| Includes (IWYU) | `UnrealSkeletalMeshTranslator.cpp` | explicit `PhysicsEngine/SkeletalBodySetup.h`, `PhysicsEngine/AggregateGeom.h`. |
| Cleanup | `UnrealLevelInstanceTranslator.cpp` | removed an unused local that 5.6 compiles more strictly about. |
| Descriptor | `HoudiniEngine.uplugin` | version marked `20050278`, `VersionName` = `2.2.0 - H20.5.278 (UE5.6 port)`, `Installed: false` for source distribution. **Descriptor edit — not part of the source diff.** |

The diff adds no new files and deletes/renames none, so a future upstream drop can be re-merged by
applying `Docs/UE56-PORT.diff` to the `Source/` tree; the descriptor fields listed above then have to be
reapplied by hand.

---

## 10. Custom fixes and features added in this fork

### 10.1 Instance transform recovery (scale + orient)

**Symptom it fixes:** for attribute instancers that output several parts (e.g. an HDA emitting *walls*
plus *roof pieces*), HAPI could report a **zero scale** or an **identity rotation** for one part while
the points carried valid `scale` / `orient` values. In Unreal those instances were invisible or
unrotated.

**Behaviour:** `FHoudiniInstanceTranslator::HapiGetInstanceTransforms` now checks the point attributes
after translating HAPI transforms and repairs only the broken entries:

| Attribute | Type | Meaning |
|---|---|---|
| `scale` | float3, point | uniform/XYZ instance scale (Houdini Y-up; mapped to Unreal Z-up on import) |
| `orient` | float4, point | quaternion `(x, y, z, w)` rotation (W inverted on import, same convention as `TranslateHapiTransform`) |

- Only entries that are *nearly zero scale* or *identity rotation* are touched — valid transforms are
  left alone.
- Repairs are logged as warnings with the recovered count plus `Geo`/`Part` ids, so a silent visual bug
  becomes a visible message:

```
[HoudiniInstanceTranslator] HapiGetInstanceTransforms: recovered 12 zero instance scales from the 'scale' attribute (Geo 21, Part 3).
```

### 10.2 StaticMesh socket export for DataTable inputs ("props MVP")

**What it does:** when a DataTable is used as an HDA input, the plugin now also exports the sockets of
the **first mesh-typed column** as point attributes. An HDA can then place props/prefabs on those
sockets per row — e.g. a "wall kit" table where each row is a wall mesh and you want lamp/bracket
positions from the mesh's sockets.

Attribute contract (point attributes on the input geometry, one point per DataTable row):

| Attribute | Storage / tuple | Meaning |
|---|---|---|
| `unreal_socket_count` | int, 1 | number of sockets on that row's mesh (`0` if none) |
| `unreal_socket_<i>_name` | string, 1 | socket name |
| `unreal_socket_<i>_pos` | float, 3 | socket location in mesh-local space, cm (`RelativeLocation`) |
| `unreal_socket_<i>_rot` | float, 3 | socket rotation in degrees — `(pitch, yaw, roll)` (`RelativeRotation`) |

Where `<i>` runs `0 … max_socket_count−1`; rows with fewer sockets get zero-filled / empty values.
Which column counts as the "mesh column": the first `FSoftObjectProperty` or `FObjectProperty` whose
class derives from `UStaticMesh`. Meshes are resolved through the soft/object property, with a
string-path `StaticLoadObject` fallback. Diagnostics are printed with the `[props MVP]` prefix in the
output log.

**Limits:** MVP implementation — one mesh column only, sockets read from the mesh asset (with a
`TObjectIterator` fallback if the array is empty in some editor builds), no per-row socket subsets, and
the data is exported at input-cook time, so changing sockets requires a recook.

### 10.3 Shutdown crash fixes

Editor shutdown could trigger `Assertion failed: Index >= 0` from `RemoveFromRoot()` on already-collected
textures, and from `TWeakObjectPtr` comparisons over `GUObjectArray` after components had been removed.
The tool-icon cache is now weak (`TWeakObjectPtr<UTexture2D>`) and the component registry is searched by
raw pointer with `GetEvenIfUnreachable()`. Both paths now fail soft instead of asserting.

### 10.4 Landscape bake output recording

`MoveCookedToBakedLandscapes` now writes the baked landscape path into the output object
(`BakedOutputObject.Landscape`, set where the baked landscape actor is created). This is bookkeeping in
the baked-output state; today the only reader is the landscape-spline bake path, which uses it to locate
the baked landscape.

---

## 11. Feature overview

Inherited from upstream v2 (all of it is in this build):

| Feature | Notes |
|---|---|
| HDA import / cook / bake | `.hda`, `.otl`; background cooking; per-parameter recook |
| Inputs | static mesh, actor, landscape (+ per edit layer), spline, data table (**+ sockets**), world / level instance, skeletal mesh, geometry collection |
| Outputs | static meshes, ISM/HISM instancing, landscapes, landscape splines, geometry collections (Chaos), data tables, curves, attributes |
| PDG | PDG asset link, work items, dirty/cook, per-work-item outputs |
| Session Sync | open HIP from Unreal, bidirectional parameter/viewport sync |
| Node Sync | `Content/NodeSync/KineFXToUnreal.hda`, `UnrealToKineFX.hda` |
| Houdini Tools | `Content/Tools/`: `he_sop_boolean`, `he_sop_curve_instancer`, `he_sop_polyreduce`, `rock_generator` |
| Examples | `Content/Examples/`: HDA, Chaos, C++, Python (15 scripts), maps, EUA/EUW API examples |
| Python API | `Content/Python/HoudiniEngineV2` (`asyncprocessor`) |
| Public C++ API | `HoudiniPublicAPI*` classes for scripting the plugin from code |

---

## 12. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No **Houdini Engine** menu after launch | Plugin not enabled, or modules failed to load | Check **Edit → Plugins** → *Houdini Engine* enabled + restart; check Output Log for `LogHoudiniEngine` / module load errors |
| Log: `Starting up the Houdini Engine module failed: built and running versions do not match.` | The Houdini build in use is not HAPI 7.0 / Houdini 20.5 | Install **Houdini 20.5.278** (or Houdini Engine 20.5.278) and point **Custom Houdini location** at it |
| Unreal reports missing / out-of-date modules on first launch | Expected for a source-only plugin, or the engine build changed | Accept the rebuild prompt; if there is no prompt, delete `Binaries/` + `Intermediate/` and reopen the project (VS 2022 + Windows SDK required, [section 8](#8-building-from-source)) |
| Build logs `Houdini Engine : Houdini 20.5.278 could not be found. Houdini Engine will not be available in this build.` | Houdini 20.5.278 not installed / registry not visible — the build still succeeds | Install it, or override `HoudiniVersion`/`HFSPath` in `HoudiniEngine.Build.cs` and rebuild |
| Session does not start, licence errors | Houdini licence / licence server not available | Start the SideFX licence server, check the licence in Houdini itself, then retry **Create Session** |
| Cook fails immediately on every asset | Session type misconfigured, or Houdini blocked by firewall | Try *Named pipe* or *Shared memory* session type; check **Show Cook Logs** |
| Instances invisible / not rotated | HAPI reported zero scale / identity rotation; pre-fix builds do not recover it | Build from this source (see 10.1) and look for the `recovered … instance scales/rotations` warning in the log |
| Editor crashes on exit with `Assertion failed: Index >= 0` | Shutdown bug present upstream; fixed in this fork | Build the modules from this source (see 10.3) — an older copy of the DLLs still has the bug |
| Landscape bake produces nothing / wrong layer | Wrong landscape bake type, or the target edit layer does not exist | In the component's *Outputs* details set the landscape bake-type dropdown (**To Current Level** / **To Image** / **To New World**) and the **Edit Layer** name field; the layer must exist in the target landscape |
| Static meshes from an HDA are all double-sided / wrong material | Plugin defaults | **Project Settings → Plugins → Houdini Engine → Generated Static Mesh Settings** (`Double Sided Geometry`, `Simple Collision Physical Material`, lightmap options) |
| `Intermediate/` explodes to many GB | Normal cook/build cache | Never commit it; delete to reclaim space, it is regenerated |

When reporting a problem, include: exact Houdini build, exact UE build (e.g. `5.6.1`), whether the
modules were built from this source or you use some prebuilt copy, and the matching lines from
**Output Log** (`LogHoudiniEngine`, `[props MVP]`, `HoudiniInstanceTranslator`).

---

## 13. Known limitations

- **Version-locked** to Houdini 20.5.278 + UE 5.6 (Win64). Nothing else is supported — see
  [section 2](#2-why-the-strict-version-lock).
- **Source only — no prebuilt binaries are shipped.** The first launch requires a one-time C++ build, so
  Visual Studio 2022 + a Windows SDK are mandatory; without them the plugin will not load.
- **No Shipping / packaged-build validation** for this port. It is configured as an editor plugin (the
  `HoudiniEngine` and `HoudiniEngineEditor` modules are editor modules); cooking in a packaged game was
  not tested.
- **`.pdb` symbols are not included** — build locally if you want to debug the plugin.
- **Landscape 5.6 paths** (edit layers export, spline layers, bake to edit layers) are ported and build,
  but were exercised less than the mesh/instance workflows — treat them as the most likely place to find
  a porting bug.
- **DataTable socket export is an MVP** (one mesh column, editor-side sockets only, see 10.2).
- **Instance transform recovery** fires only for zero-scale / identity-rotation entries; it deliberately
  does not second-guess transforms that HAPI reported.
- **Not affiliated with SideFX.** Upstream plugin bugs belong to SideFX; UE 5.6 port bugs belong in this
  repository's issue tracker. SideFX's own documentation — <https://www.sidefx.com/docs/unreal/> — still
  applies to the plugin's behaviour.

---

## 14. Repository layout

```
HoudiniEngine/                       <- drop this folder into YourProject/Plugins/
  HoudiniEngine.uplugin              plugin descriptor (3 modules, EnabledByDefault, Installed=false)
  README.md                          this document
  LICENSE.md                         SideFX licence (upstream, unchanged)
  NOTICE.md                          port attribution / modification notice
  Config/BaseHoudiniEngine.ini       core redirects for deprecated enums
  Content/
    Examples/                        HDA, Chaos, C++, Python, maps, EUA/EUW examples
    Icons/  Textures/                plugin UI art
    NodeSync/                        KineFXToUnreal.hda, UnrealToKineFX.hda
    Python/HoudiniEngineV2/          Python API package
    Test/hda/                        small test HDAs (TestBox, TestParams, InputEcho, …)
    Tools/                           Houdini Tools packages (boolean, curve instancer, polyreduce, rock generator)
  Resources/                         icons and banners
  Source/
    HoudiniEngine/                   core module (HAPI, translators, cooking)
    HoudiniEngineEditor/             editor module (menus, details, bake UI)
    HoudiniEngineRuntime/            runtime module (assets, components, settings)
  Docs/
    UE56-PORT.diff                   complete Source/ port patch against the 20.5.278 base
                                     (descriptor edits are not included)
    UE56-PORT-NOTES.md               per-file porting notes and re-apply instructions
  Binaries/  Intermediate/           NOT in the repo — created by the first build
```

**Not in the repository on purpose:** `Binaries/` and `Intermediate/` (build output — compiled modules
also embed the builder's local paths and engine build id, so they are built per machine and per engine
build) and `.pdb` symbols.

---

## 15. Updating the plugin to other versions

If you need Houdini 21 / UE 5.7 later, this is the checklist (the same work that was done here for
5.6 — use [`Docs/UE56-PORT.diff`](Docs/UE56-PORT.diff) as the template):

1. **HAPI headers** — copy the new `HAPI` headers from your Houdini installation
   (`$HFS/toolkit/include/HAPI`) over `Source/HoudiniEngine/Public/HAPI/`.
2. **Build.cs version** — set `string HoudiniVersion = "<new Houdini version>"` in
   `Source/HoudiniEngine/HoudiniEngine.Build.cs`; make sure the registry/default paths for that version
   exist (the code builds them from the string).
3. **Descriptor** — update `Version`/`VersionName` in `HoudiniEngine.uplugin`.
4. **Compile** — rebuild and fix engine API breaks. Expect landscape edit-layer and UObject-lifetime
   code to be the first things to break; the 5.6 patch shows the exact calls that changed.
5. **Smoke test** — import a simple HDA, cook, change a parameter, bake; then test landscape export and
   DataTable inputs if you use them.

Do not mix a new HAPI with modules built against the old headers: the plugin's runtime HAPI check
aborts on a major/minor mismatch (an API-only mismatch only warns).

---

## 16. Licence and credits

- **Plugin code, assets and documentation:** © Side Effects Software Inc. Redistribution and use are
  permitted under the SideFX licence in [`LICENSE.md`](LICENSE.md) — the copyright notice must be kept,
  and the SideFX/Side Effects names may not be used to endorse or promote derived products. This
  repository keeps the licence and notices intact.
- **This fork (UE 5.6 port, fixes, DataTable socket export, documentation):** rly_ghost.
  See [`NOTICE.md`](NOTICE.md) for what was modified.
- **Unofficial.** This is not a SideFX release, is not supported by SideFX, and is not endorsed by them.
  SideFX's official repositories: <https://github.com/sideeffects/HoudiniEngineForUnreal>.
- Houdini and Houdini Engine are trademarks of Side Effects Software Inc. Unreal Engine is a trademark
  of Epic Games, Inc.
