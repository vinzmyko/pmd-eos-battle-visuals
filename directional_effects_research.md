# Directional Effect Animation System

## Overview

This document describes how PMD: Explorers of Sky determines whether an effect sprite uses directional animation sequences (8 pre-rotated variants) or a single animation with projectile translation.

**Key Finding:** Direction is added to `animation_index` when `sequence_count % 8 == 0`.

---

## Effect Initialization: FUN_022bdfc0

**Address:** `0x022bdfc0`

This function initializes an effect and determines which animation sequence to use.

### Relevant Code Flow

```c
// 1. Get base animation_index from effect_animation_info table
*(int *)(param_1 + 0x50) = peVar2->animation_index;

// 2. Get sprite sequence count via FUN_0201da20
if (anim_type != 5 && anim_type != 6) {  // Skip for screen effects
    uVar3 = FUN_0201da20(*(short *)(param_1 + 0x64));  // sprite_id
    *(undefined4 *)(param_1 + 0x10) = uVar3;  // Store sequence_count
}

// 3. Conditionally add direction to animation_index
iVar6 = *(int *)(param_1 + 0x1c);  // direction (0-7, or -1 if none)
if ((iVar6 != -1) && (sequence_count % 8 == 0)) {
    *(int *)(param_1 + 0x50) += iVar6;  // animation_index += direction
}
```

### Effect Context Structure (param_1)

| Offset | Type | Description |
|--------|------|-------------|
| 0x10 | int | Sequence count from FUN_0201da20 |
| 0x14 | int | Effect ID (index into effect_animation_info) |
| 0x18 | int | Timing offset base |
| 0x1c | int | Direction (0-7) or -1 if not directional |
| 0x40 | int | Animation type (from effect_animation_info.anim_type) |
| 0x44 | int | File index (from effect_animation_info.file_index) |
| 0x50 | int | Final animation_index (may have direction added) |
| 0x54 | int | Sprite pointer |
| 0x58 | int | Sound effect ID |
| 0x5c | int | Timing value |
| 0x60 | u8 | is_non_blocking flag |
| 0x61 | u8 | Loop/repeat flag |
| 0x64 | short | Sprite ID (for WAN_TABLE lookup) |
| 0x68 | animation_control | Animation control structure |

---

## Sprite Sequence Count Lookup: FUN_0201da20

**Address:** `0x0201da20`

Retrieves the number of animation sequences for a sprite from the WAN_TABLE.

### Assembly

```asm
ldr   r2,[DAT_0201da3c]           ; r2 = pointer at 0x020AFC64
ldr   r2,[r2,#0x4]                ; r2 = WAN_TABLE base
mov   r1,#0x38                    ; Entry size = 0x38 (56 bytes)
smlabb r0,r0,r1,r2                ; r0 = sprite_id * 0x38 + WAN_TABLE
ldr   r0,[r0,#0x30]               ; r0 = entry->sprite_start (wan_header*)
bx    r12=>LAB_0201da00           ; Jump to sequence count extraction
```

### C Equivalent

```c
short FUN_0201da20(short sprite_id) {
    wan_table* table = *(wan_table**)(0x020AFC64 + 4);
    wan_table_entry* entry = &table->sprites[sprite_id];
    wan_header* header = entry->sprite_start;  // offset 0x30
    return LAB_0201da00(header);
}
```

### WAN_TABLE Entry Structure

From `pmd-sky/include/wan.h`:

```c
struct wan_table_entry {
    char path[32];                   // 0x00: Null-terminated path
    bool8 file_externally_allocated; // 0x20
    u8 source_type;                  // 0x21: 1=direct file, 2=pack file
    s16 pack_id;                     // 0x22
    s16 file_index;                  // 0x24
    u8 field5_0x26;
    u8 field6_0x27;
    u32 iov_len;                     // 0x28
    s16 reference_counter;           // 0x2C
    u8 field9_0x2e;
    u8 field10_0x2f;
    struct wan_header* sprite_start; // 0x30 ← Used by FUN_0201da20
    void* iov_base;                  // 0x34: Points to SIR0 container
};
// Total size: 0x38 (56 bytes)
```

---

## Sequence Count Extraction: LAB_0201da00

**Address:** `0x0201da00`

Extracts the sequence count from a WAN header based on sprite type.

### Assembly

```asm
ldrb   r1,[r0,#0x8]        ; r1 = sprite_type (offset 0x8 in wan_header)
ldr    r0,[r0,#0x0]        ; r0 = ptr_anim_info (offset 0x0)
cmp    r1,#0x0             ; if sprite_type == 0 (PROPS_UI)
cmpne  r1,#0x2             ; OR sprite_type == 2
ldreq  r0,[r0,#0x8]        ; r0 = ptr_anim_group_table
ldrsheq r0,[r0,#0x4]       ; r0 = group_0->anim_length (sequence count)
ldrshne r0,[r0,#0xc]       ; else: r0 = nb_anim_groups
bx     lr
```

### C Equivalent

```c
short LAB_0201da00(wan_header* header) {
    uint8_t sprite_type = *(uint8_t*)(header + 0x8);
    void* ptr_anim_info = *(void**)(header + 0x0);
    
    if (sprite_type == 0 || sprite_type == 2) {
        // Effect sprites (PROPS_UI): return sequence count in group 0
        void* anim_group_table = *(void**)(ptr_anim_info + 0x8);
        return *(int16_t*)(anim_group_table + 0x4);  // anim_length
    } else {
        // Character sprites: return number of animation groups
        return *(int16_t*)(ptr_anim_info + 0xC);  // nb_anim_groups
    }
}
```

### Sprite Types

| Value | Name | Sequence Count Source |
|-------|------|----------------------|
| 0 | PROPS_UI (Effects) | `anim_group_table[0].anim_length` |
| 1 | CHARA (Characters) | `ptr_anim_info->nb_anim_groups` |
| 2 | Unknown | Same as type 0 |

---

## Direction Addition Condition

### The Bit Manipulation

The original decompiled condition:

```c
if ((iVar6 != -1) &&
   (uVar5 = *(int *)(param_1 + 0x10) >> 0x1f,
   (*(int *)(param_1 + 0x10) * 0x20000000 + uVar5 >> 0x1d | uVar5 << 3) == uVar5))
```

### Simplified Logic

This complex bit manipulation tests if `sequence_count % 8 == 0`:

| sequence_count | × 0x20000000 | 32-bit Overflow | Condition Result |
|----------------|--------------|-----------------|------------------|
| 0 | 0 | 0 | TRUE (but no sequences) |
| 1 | 0x20000000 | No | FALSE |
| 2 | 0x40000000 | No | FALSE |
| 7 | 0xE0000000 | No | FALSE |
| **8** | 0x100000000 | **Yes → 0** | **TRUE** |
| 9 | 0x120000000 | Yes → 0x20000000 | FALSE |
| **16** | 0x200000000 | **Yes → 0** | **TRUE** |
| 50 | wraps to non-zero | No | FALSE |

### Final Rule

```c
bool is_directional = (direction != -1) && (sequence_count > 0) && (sequence_count % 8 == 0);

if (is_directional) {
    animation_index += direction;  // 0-7 added to base index
}
```

---

## Practical Examples

### Water Gun (Effect 334)

- **File:** 0
- **Base animation_index:** 45
- **Sequence count in file 0:** 50
- **50 % 8 =** 2 ≠ 0
- **Result:** NOT directional, uses translation only

### Shadow Sneak (Effect 479)

- **File:** 184
- **Base animation_index:** 8
- **Sequence count in file 184:** 16
- **16 % 8 =** 0
- **Result:** DIRECTIONAL, sequences 8-15 are 8 pre-rotated variants

---

## Summary

### Directional Effects (sequence_count % 8 == 0)

- ROM adds direction (0-7) to base `animation_index`
- WAN file contains 8 consecutive sequences with identical frame counts
- Each sequence is a pre-rotated version of the effect
- Used for effects that must visually rotate (shadows, directional beams)

### Non-Directional Effects (sequence_count % 8 != 0)

- ROM uses base `animation_index` directly (no direction added)
- Single animation sequence regardless of attacker direction
- Direction only affects projectile **translation** (X/Y movement)
- Used for symmetric effects (particles, explosions, splashes)

---

## ROM Addresses Reference

| Address | Name | Purpose |
|---------|------|---------|
| 0x022bdfc0 | FUN_022bdfc0 | Effect initialization, direction addition logic |
| 0x0201da20 | FUN_0201da20 | Sprite sequence count lookup |
| 0x0201da00 | LAB_0201da00 | Extract sequence count from WAN header |
| 0x020AFC64 | DAT_020AFC64 | Pointer to WAN_TABLE pointer |
| 0x022beb2c | FUN_022beb2c | Projectile velocity calculation |
