# Function Reference

## Summary

Comprehensive list of functions analysed during reverse engineering research. Organized by system/purpose. All addresses are for NA region unless noted.

---

## Effect Animation System

### Core Accessors

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `GetEffectAnimation` | `0x022BFE9C` | Look up effect_animation by ID | `int effect_id` → `effect_animation*` |
| `GetMoveAnimation` | - | Look up move_animation by ID | `move_id id` → `move_animation*` |
| `GetMoveAnimationSpeed` | - | Read projectile_speed from move_animation | `uint move_id` → `int speed` |
| `GetMoveAnimationId` | - | Resolve animation ID with weather/charging | `move*, weather, is_charging` → `uint16_t` |
| `GetSpecialMonsterMoveAnimation` | - | Look up per-Pokemon override entry | `int index` → `special_monster_move_animation*` |

### Flag Readers

| Function | Address | Purpose | Returns |
|----------|---------|---------|---------|
| `FUN_022bfd58` | `0x022bfd58` | Read flag bits 0-2 (animation category) | `flags & 0x07` |
| `FUN_022bfd6c` | `0x022bfd6c` | Read flag bit 3 (dual-target) | `flags & 0x08` |
| `FUN_022bfd8c` | `0x022bfd8c` | Read flag bit 4 (skip fade-in) | `flags & 0x10` |
| `FUN_022bfdac` | `0x022bfdac` | Read flag bit 5 (unknown) | `flags & 0x20` |
| `FUN_022bfdcc` | `0x022bfdcc` | Read flag bit 6 (post-delay) | `flags & 0x40` |

### Override Lookups

| Function | Address | Purpose | Returns |
|----------|---------|---------|---------|
| `FUN_022bf01c` | `0x022bf01c` | Get attachment point with override | `int8_t` |
| `FUN_022bf088` | `0x022bf088` | Get attachment point for species | `int` |
| `FUN_022bf0f4` | `0x022bf0f4` | Get sound effect with override | `uint16_t` |
| `FUN_022bfa3c` | `0x022bfa3c` | Get animation type with override | `uint8_t` |

### Effect Allocation & Setup

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `FUN_022be44c` | `0x022be44c` | Allocate effect context from pool | `int, int*, int` → `int handle` |
| `FUN_022be730` | `0x022be730` | Allocate + initialize effect | `int, int*, int` → `int handle` |
| `FUN_022be780` | `0x022be780` | Main effect dispatcher | `int dispatch_type, uint* params, int` → `int handle` |
| `FUN_022bdfc0` | `0x022bdfc0` | Initialize animation from effect_animation | `int context, int param` |
| `FUN_022be9a0` | `0x022be9a0` | Convert effect handle to context index | `int handle` → `int index` |
| `FUN_022bf274` | `0x022bf274` | Initialize effect params struct | `undefined* params` |
| `FUN_022bf2b4` | `0x022bf2b4` | Spawn effect from params | `uint* params, int` → `int handle` |

### Effect Cleanup

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `FUN_022bde50` | `0x022bde50` | Stop/cleanup effect | `int handle` |
| `FUN_022bdec4` | `0x022bdec4` | Force-stop effect | `entry*, int` |

### Type-Specific Handlers

| Function | Address | Purpose | Effect Types |
|----------|---------|---------|--------------|
| `FUN_022c01fc` | `0x022c01fc` | Screen effect setup | Type 5 |
| `FUN_022c0280` | `0x022c0280` | WBA effect setup | Type 6 |
| `FUN_022c03f4` | `0x022c03f4` | Standard effect allocation | Types 3, 4 |
| `FUN_022c0450` | `0x022c0450` | Special effect allocation | Types 5, 6 |
| `FUN_022c067c` | `0x022c067c` | Effect resource lookup | Types 5, 6 |

---

## Move Effect Pipeline

### Main Entry Points

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `ExecuteMoveEffect` | - | Main move execution orchestrator | `target_list*, entity* attacker, move*, ...` |
| `PlayMoveAnimation` | - | Single-target animation handler | `entity* user, entity* target, move*, position*` |
| `FUN_023258ec` | `0x023258ec` | Dual-target animation handler (flag bit 3) | `int* attacker, int* target, int move, ...` |
| `FUN_023250d4` | `0x023250d4` | Monster animation + special behaviors | `int*, uint, ...` |

### Layer Handlers

| Function | Address | Purpose | Layer | Dispatch Type |
|----------|---------|---------|-------|---------------|
| `FUN_022bfaa8` | `0x022bfaa8` | Charge/prep effects | 0 (offset 0x00) | 5 |
| `FUN_022bed90` | `0x022bed90` | Secondary effects + Razor Leaf | 1 (offset 0x02) | 1 |
| `FUN_022bfc5c` | `0x022bfc5c` | Primary visual effects | 2 (offset 0x04) | 6 |
| `FUN_022be9e8` | `0x022be9e8` | Projectile effects | 3 (offset 0x06) | 2 |

### Helper Functions

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_022bf160` | `0x022bf160` | Check if move has type 5 effect |
| `FUN_022bf1fc` | `0x022bf1fc` | Validate specific effect layer |
| `FUN_02325644` | `0x02325644` | Trigger secondary effect (layer 1) |
| `FUN_02325d20` | `0x02325d20` | Pre-animation check |
| `FUN_02324e78` | `0x02324e78` | Screen fade-in handling |
| `ChangeMonsterAnimation` | - | Set monster sprite animation |

---

## Projectile Motion

### Main Handlers

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `FUN_023230fc` | `0x023230fc` | Forward projectile motion | `uint* entity, ushort* move, int range, int wave_pattern, ...` |
| `FUN_0232393c` | `0x0232393c` | Reverse direction projectile | `int entity, int* target, int move, uint range, int wave_pattern` |

### Position Update

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_022beb2c` | `0x022beb2c` | Update effect position during flight |

### Trigonometry

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `SinAbs4096` | - | Sine with 4096-step angles | `int angle` → `int` |
| `CosAbs4096` | - | Cosine with 4096-step angles | `int angle` → `int` |

---

## Animation Timing

### Frame Advance

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `AdvanceFrame` | - | Wait one frame | `char debug_marker` (unused) |
| `FUN_022ea370` | `0x022ea370` | Wait n frames | `int frame_count, ...` |
| `FUN_022ea324` | `0x022ea324` | Frame advance path A | `..., int, ...` |
| `FUN_022ea2a4` | `0x022ea2a4` | Frame advance path B | `..., int, ...` |

### Task Scheduler

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_022ddef8` | `0x022ddef8` | Core task scheduler |

### Animation State

| Function | Address | Purpose | Returns |
|----------|---------|---------|---------|
| `AnimationHasMoreFrames` | - | Check if animation still playing | `bool` |
| `AnimationDelayOrSomething` | - | Add standard post-animation delay | - |

### Screen Effects

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_0234ba54` | `0x0234ba54` | Wait for screen ready (pre-type 5) |
| `FUN_0234b4cc` | `0x0234b4cc` | Enable/disable something (called with 1/0) |

---

## Entity Positioning

### Pixel Position

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `SetEntityPixelPosXY` | - | Set pixel position directly | `entity*, uint32_t x, uint32_t y` |
| `IncrementEntityPixelPosXY` | - | Add to pixel position | `entity*, uint32_t x, uint32_t y` |
| `UpdateEntityPixelPos` | - | Set from tile pos or pixel_position | `entity*, pixel_position*` |

### Attachment Points

| Function | Address | Purpose | Parameters |
|----------|---------|---------|------------|
| `FUN_0201cf90` | `0x0201cf90` | Calculate attachment point offset | `short* out, ushort* anim_ctrl, uint index` |

### Direction

| Function | Address | Purpose |
|----------|---------|---------|
| `GetDirectionTowardsPosition` | - | Calculate direction from position delta |

---

## Entity Management

### Spawning

| Function | Address | Purpose |
|----------|---------|---------|
| `SpawnMonster` | `0x022fd084` | Spawn monster entity |
| `SpawnItemEntity` | - | Spawn blank item entity |
| `SpawnItem` | - | Spawn item with full setup |

### Validation

| Function | Address | Purpose | Returns |
|----------|---------|---------|---------|
| `EntityIsValid` | - | Check entity pointer and type | `bool` |
| `EntityIsValidMoveEffects` | - | Check if valid monster for effects | `bool` |
| `ShouldDisplayEntityWrapper` | - | Check if entity should be rendered | `bool` |
| `ShouldDisplayEntityAdvanced` | - | Extended display check | `bool` |

### Entity Attachment

| Function | Address | Purpose |
|----------|---------|---------|
| `FUN_022e6d68` | `0x022e6d68` | Attach effect to entity |

---

## Animation Control

### Setup

| Function | Address | Purpose |
|----------|---------|---------|
| `InitAnimationControlWithSet` | - | Initialize animation_control struct |
| `SetSpriteIdForAnimationControl` | - | Set WAN table sprite reference |
| `SetAnimationForAnimationControl` | - | Configure animation parameters |
| `SetAnimationForAnimationControlInternal` | `0x0201C5C4` | Inner setup (critical group/sequence logic) |

### WAN Table

| Function | Address | Purpose |
|----------|---------|---------|
| `InitWanTable` | - | Initialize with 96 entries |
| `AllocateWanTableEntry` | - | Find and allocate free slot |
| `DeleteWanTableEntry` | - | Decrement refcount, optionally free |
| `FindWanTableEntry` | - | Search by path string |
| `GetLoadedWanTableEntry` | - | Search by pack_id + file_index |
| `LoadWanTableEntry` | - | Load from filesystem |
| `LoadWanTableEntryFromPack` | - | Load from pack archive |
| `GetWanForAnimationControl` | - | Get WAN header pointer |
| `WanHasAnimationGroup` | - | Check if animation group exists |

---

## Move State Checks

| Function | Address | Purpose |
|----------|---------|---------|
| `ShouldMovePlayAlternativeAnimation` | - | Check two-turn move charging |
| `IsChargingTwoTurnMove` | - | Check if charging specific move |
| `IsChargingAnyTwoTurnMove` | - | Check if charging any two-turn move |
| `Is2TurnsMove` | - | Check if move is two-turn type |
| `GetApparentWeather` | - | Get weather for Weather Ball |

---

## Sound

| Function | Address | Purpose |
|----------|---------|---------|
| `PlaySeByIdIfNotSilence` | - | Play sound effect if not 0x3F00 |
| `PlaySeByIdIfShouldDisplayEntity` | - | Play sound if entity visible |

---

## Pack File

| Function | Address | Purpose |
|----------|---------|---------|
| `GetFileLengthInPackWithPackNb` | - | Get file size from pack archive |

---

## Utility

| Function | Address | Purpose |
|----------|---------|---------|
| `GetTile` | - | Get tile at coordinates |
| `GetTileSafe` | - | Get tile with bounds checking |
| `FUN_022e2ca0` | `0x022e2ca0` | Validate position |
| `DungeonRandInt` | - | Random integer |
| `DungeonRandOutcome` | - | Random bool with percentage |
| `_s32_div_f` | - | Signed 32-bit division |

---

## Memory Addresses - Data Tables

| Symbol | Address (NA) | Purpose |
|--------|--------------|---------|
| `MOVE_ANIMATION_INFO` | `0x022C9064` | Move animation table base |
| `EFFECT_ANIMATION_INFO` | `0x022CC52C` | Effect animation table base |
| `DIRECTIONS_XY` | `0x0235171C` | 8-direction vector table |
| `DAT_023238f8` | `0x023238f8` | Pointer to X velocity table |
| `DAT_023238fc` | `0x023238fc` | Pointer to Y velocity table |
| `DAT_02323900` | `0x02323900` | Additional direction data |
| `DAT_02323908` | `0x02323908` | Z priority data |
| `DAT_022bfb64` | `0x022bfb64` | Layer 0 parameter template |
| `DAT_022bfd18` | `0x022bfd18` | Layer 2 parameter template |
| `DAT_022beb20` | `0x022beb20` | Layer 3 parameter template |
| `DAT_022befd4` | `0x022befd4` | Layer 1 parameter template |
| `DAT_022bfed8` | `0x022bfed8` | special_monster_move_animation table base |

---

## Notes

- Addresses marked `-` are either inlined, not yet located, or vary by context
- All addresses are for NA region; EU and JP offsets differ
- Function names starting with `FUN_` are Ghidra auto-generated names
- Named functions come from pmdsky-debug symbols
