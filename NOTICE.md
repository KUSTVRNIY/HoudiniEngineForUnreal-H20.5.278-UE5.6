# Notice — modifications to SideFX Houdini Engine for Unreal

This repository redistributes the **Houdini Engine for Unreal** plugin (version 2) originally
copyright © Side Effects Software Inc. The upstream licence is kept unchanged in
[`LICENSE.md`](LICENSE.md) and applies to all plugin code, assets and documentation in this
repository, including the modifications listed below.

**This is an unofficial fork. It is not a SideFX release, is not supported by SideFX, and is not
endorsed by SideFX.** Houdini and Houdini Engine are trademarks of Side Effects Software Inc.
Unreal Engine is a trademark of Epic Games, Inc.

## Upstream base

- Product: Houdini Engine for Unreal, version 2, as distributed for **Houdini 20.5.278**
  (`HoudiniVersion = "20.5.278"` in the plugin's `Build.cs`; HAPI headers generated for Houdini 20.5.278,
  HAPI 7.0 / api 0).
- This base does **not** equal a single public upstream tag: the GitHub tag `v2.2.0` of the upstream
  project targets Houdini 20.5.292, and this base predates the UE 5.5 landscape-edit-layer API.
- `Version` in the descriptor: `20050278` (= Houdini build 20.5.278).
- Upstream project: <https://github.com/sideeffects/HoudiniEngineForUnreal>

## Modifications made in this fork

1. **Unreal Engine 5.6 port** — landscape edit-layer API migration, `TObjectPtr` / `TWeakObjectPtr`
   lifetime corrections, IWYU includes and small compile fixes, all guarded by
   `ENGINE_MAJOR_VERSION` / `ENGINE_MINOR_VERSION` so earlier engine paths remain intact.
   Source patch: [`Docs/UE56-PORT.diff`](Docs/UE56-PORT.diff) (covers `Source/` only; descriptor edits are
   listed under point 6 below); rationale: [`Docs/UE56-PORT-NOTES.md`](Docs/UE56-PORT-NOTES.md).
2. **Instance transform recovery** — recovers per-instance scale/rotation from the Houdini point
   `scale` / `orient` attributes when HAPI reports zero scale or identity rotation.
3. **DataTable StaticMesh socket export** ("props MVP") — exports mesh sockets as point attributes
   for DataTable inputs.
4. **Shutdown crash fixes** — weak reference for the tool-icon cache and safe component
   unregistration, removing `Assertion failed: Index >= 0` failures on editor exit.
5. **Landscape bake fixes** — always commit a mask heightfield, record the baked landscape path in the
   baked-output state, use the 5.6 edit-layer lock/visibility API.
6. **Plugin descriptor** — `VersionName` changed to `2.2.0 - H20.5.278 (UE5.6 port)` to make the
   version lock explicit, and `"Installed"` set to `false` because this fork is distributed as source
   that must be compiled once in the target project (no prebuilt binaries are published).
   `CreatedBy` and SideFX URLs are left as in the original; the descriptor is not part of the source
   patch in `Docs/UE56-PORT.diff` (see `Docs/UE56-PORT-NOTES.md`, section 11).

## What was not modified

- `LICENSE.md` (SideFX licence) — unchanged.
- The plugin's functional behaviour, HDA assets, examples, icons and Python/API surface, except where
  listed above.
- SideFX copyright headers in source files — unchanged.

## Redistribution requirements

The SideFX licence requires that (a) the copyright notice, this list of conditions and the disclaimer
are retained, and (b) the names *Side Effects Software* / *SideFX* are not used to endorse or promote
products derived from this software without prior written permission. If you redistribute this fork or
a build of it, keep `LICENSE.md` and this `NOTICE.md` together with it, and do not present it as an
official SideFX release.
