# Phase 2: Modding Feasibility Assessment

Based on the analysis of `PartPrefabs`, `BuildingPart`, and `PlanePart` classes.

## 2.1 Capability Classification

| Capability | Feasibility | Classification | Notes |
| :--- | :--- | :--- | :--- |
| **Injecting New Parts** | High | ⚠️ Requires Harmony | Parts are in a `GameObject[]` array (`PartPrefabs`). We must patch `Awake`/`Start` to resize and append to this array *before* other systems read it. |
| **Cloning Parts** | High | ✅ Clean | Instantiating an existing Prefab and registering it as a new entry is trivial and safe. |
| **External Stat Loading** | High | ✅ Clean | All critical stats (mass, thrust, drag) are public fields on MonoBehaviours. Can be set via Reflection/JSON easily. |
| **External Model Loading** | Medium | ⚠️ Complex | Loading a mesh is easy, but `snapPoints` (Transforms) and Colliders must be manually realigned to match the new mesh shape. |
| **UI Integration** | High | ✅ Clean | UI appears to iterate the `PartPrefabs` list. Injecting into the list should auto-populate the build menu. |
| **Save Compatibility** | Medium | ⚠️ Caution | If saves store parts by Index, adding parts might shift indices and break saves. If stored by Name (String), it's safe. *Needs verification of SaveSystem if possible, but assuming Name is standard for robust systems. If Index, we must append to end.* |
| **Multiplayer** | Low | ❌ Unsafe | Client-side only custom parts will desync. Both clients need the same mod/configs. No checksum validation is visible yet. |

## 2.2 Recommendation

**Target Approach**: "Soft-Coded" Part Loader.
1.  **Core**: A BepInEx plugin that patches `PartPrefabs` initialization.
2.  **Loader**: Reads JSON files from `Config/Parts`.
3.  **Injector**:
    *   Clone a "Base Prefab" (e.g., standard Engine).
    *   Apply stats from JSON.
    *   (Optional) Swap Mesh if OBJ provided.
    *   Register into `partPrefabs` array.

## 2.3 Risk Assessment

*   **Registry Race Condition**: Usage of `PartPrefabs.partPrefabs` by other scripts (like `BuildManager`) might happen before we inject. *Mitigation: Patch early (Awake) or patch the accessor.*
*   **Physics Stability**: Setting invalid stats (e.g., 0 mass, Infinity thrust) will crash the solver. *Mitigation: Input validation sanitization.*
