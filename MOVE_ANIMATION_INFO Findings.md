# Reverse Engineering Findings: MOVE_ANIMATION_INFO

**Base Address:** `0x022C9064` (MOVE_ANIMATION_INFO)
## 1. Likely Data Structure

| Offset   | Size  | New Name             | Old Name    | Purpose                                                          |
| :------- | :---- | :------------------- | :---------- | :--------------------------------------------------------------- |
| **0x00** | `u16` | `effect_charge`      | `field_0x0` | **Phase 1 Effect:** Spawns on User immediately (or after delay). |
| **0x02** | `u16` | `effect_impact`      | `field_0x2` | **Phase 3 Effect:** Spawns on Target upon impact.                |
| **0x04** | `u16` | `effect_target`      | `field_0x4` | **Phase 3 Alt:** Spawns on Target (Status/Debuff visuals).       |
| **0x06** | `u16` | `effect_projectile`  | `field_0x6` | **Phase 2 Effect:** Spawns at User, travels to Target.           |
| **0x08** | `u8`  | `delay_timer`        | `field_0x8` | Frames to wait before spawning Charge/Projectile.                |
| **0x09** | `u8`  | `unknown_09`         | `field_0x9` | *Unused/Padding in most moves.*                                  |
| **0x0A** | `u8`  | `unknown_0a`         | `field_0xa` | *Unused/Padding in most moves.*                                  |
| **0x0B** | `u8`  | `unknown_0b`         | `field_0xb` | *Unused/Padding in most moves.*                                  |
| **0x0C** | `u8`  | `spawn_bone`         | `field_0xc` | Visual origin point (0=Feet, 1=Head, 2=Hand/Center).             |
| **0x0D** | `u8`  | `unknown_0d`         | -           | Part of old "speed" u32.                                         |
| **0x0E** | `u8`  | `unknown_0e`         | -           | Part of old "speed" u32.                                         |
| **0x0F** | `u8`  | `unknown_0f`         | -           | Part of old "speed" u32.                                         |
| **0x10** | `u8`  | `user_anim_id`       | `animation` | The physical sprite action (Attack, Special, etc).               |
| **0x11** | `i8`  | `height_offset`      | `point`     | Signed Y-axis offset for effect spawning.                        |
| **0x12** | `u16` | `sfx_id`             | `sfx_id`    | Sound Effect ID (0x3F00 = Silence).                              |
| **0x14** | `u16` | `special_anim_count` | -           | Count of special overrides (Weather ball, etc).                  |
| **0x16** | `u16` | `special_anim_idx`   | -           | Index for overrides.                                             |

---

## 2. Evidence & Logic
### 4 Effect IDs (Offsets 0x00 - 0x06)
Different functions in the code read specific offsets at specific times.
*   **0x00 (`effect_charge`):** 
    *   **Ghidra:** `FUN_022bfaa8` reads offset `0x0`.
    *   **Context:** Called early in the sequence via `FUN_023250d4` (The Sequence Orchestrator).
    *   **Data:** Razor Leaf (ID 251) has `296` here. Visually, leaves swirl around the *user* before firing.
*   **0x06 (`effect_projectile`):**
    *   **Ghidra:** `FUN_023230fc` calls `GetMoveAnimationSpeed` and performs `SinAbs4096`/`CosAbs4096` math. This implies path calculation.
    *   **Ghidra:** `FUN_022be9e8` reads offset `0x6`.
    *   **Data:** Bubble (ID 145) has `212` here. Visually, a bubble travels from User -> Target.
*   **0x02 (`effect_impact`):**
    *   **Ghidra:** `FUN_02325644` (The Trigger Function) checks if `field_0x2 == 0` before calling `FUN_022bed90`.
    *   **Data:** Tackle (ID 33) has `62` here. Tackle has no projectile, just a "Bash" graphic on the target.
*   **0x04 (`effect_target`):**
    *   **Ghidra:** `PlayMoveAnimation` (the wrapper) reads offset `0x4`.
    *   **Logic:** Since `PlayMoveAnimation` runs *after* the sequence logic in many cases, this is likely for "Result" effects (Status conditions, stat drops) rather than the projectile itself.

### The Timing & Config (Offsets 0x08 - 0x0C)
*   **0x08 (`delay_timer`):**
    *   **Scan Data:** Move 22 (Vine Whip/SolarBeam charge?) has `0x60` (96 frames). Move 145 (Bubble) has `0x00`.
    *   **Ghidra:** `FUN_022bfd58` reads this byte. The main loop in `FUN_023250d4` waits for flags derived from this.
*   **0x0C (`spawn_bone`):**
    *   **Scan Data:** Bubble has `0x01` (Mouth). Razor Leaf has `0x02` (Body/Hands).
    *   **Logic:** Essential for projectiles. Bubble needs to spawn at the mouth. If this was "speed", 0x01 would be incredibly slow.
    *   **Ghidra:** `FUN_023230fc` (Projectile Logic) uses this value to calculate the starting `local_2c` (Position Vector).

### The Audio (Offset 0x12)
*   **Scan Data:**
    *   Bubble: `16128` (`0x3F00`).
    *   Razor Leaf: `258` (`0x0102`).
*   **Ghidra:**
    *   In `FUN_023250d4`:
     ```c
        uVar11 = ReadOffset0x12(...);
        if (uVar11 != 0x3f00) {
            PlaySeByIdIfNotSilence(uVar11 & 0xffff);
        }
        ```
    *   `0x3F00` is the engine's constant for "No Sound".
    *   This explains why Bubble (which might use a default cast sound, or relies on the Effect WAN's internal sound) has "Silence" here.

---

## 3. Function Reference Map (NA Region)

| Address      | Ghidra Name            | Purpose                                             | Key Offsets Read       |
| :----------- | :--------------------- | :-------------------------------------------------- | :--------------------- |
| `0x023250d4` | `SequenceOrchestrator` | Controls the timing loop of the move.               | `0x00`, `0x08`, `0x12` |
| `0x02325644` | `EffectTrigger`        | Called when animation frame reached. Spawns impact. | `0x02`                 |
| `0x023230fc` | `ProjectileLogic`      | Calculates trajectory and movement.                 | `0x06`, `0x0C`         |
| `0x022bed90` | `ImpactSpawner`        | Handles spawning the hit effect.                    | `0x02`                 |
| `0x02325410` | `PlayMoveAnimation`    | Wrapper/Entry point. Handles target status effects. | `0x04`                 |