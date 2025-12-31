# WAN Header Structure

## Summary

- WAN files store sprite data (animations, meta-frames, palettes)
- Sprite type at offset 0x08 determines animation selection behavior
- CHARA (type 1) uses group+direction, PROPS_UI (type 0) uses flat sequence index
- Header contains pointers to animation info, image data, and palette

## Structure Definition

```c
struct wan_header {
    void* ptr_anim_info;       // 0x00: Pointer to animation info structure
    void* ptr_image_data;      // 0x04: Pointer to image/tile data
    uint8_t sprite_type;       // 0x08: Sprite type (0, 1, 2, or 3)
    uint8_t unk_0x09;          // 0x09: Unknown
    uint16_t unk_0x0a;         // 0x0A: Unknown
    // ... additional fields
};
```

## Sprite Types

| Value | Name | Description | Animation Selection |
|-------|------|-------------|---------------------|
| 0 | PROPS_UI | Effects, UI elements, props | `animations[0][animation_id]` |
| 1 | CHARA | Character sprites (Pokémon, NPCs) | `animations[group_id][animation_id]` |
| 2 | Unknown | Treated same as type 0 | `animations[0][animation_id]` |
| 3 | WAN_SPRITE_UNK_3 | Unknown, special 3D rendering | Special path |

**Evidence:** `SetAnimationForAnimationControl` at `0x0201C4F4`
```c
wVar2 = SpriteTypeInWanTable(anim_ctrl->loaded_sprite_id);
if ((wVar2 == WAN_SPRITE_PROPS_UI) || ((wVar2 + 0xfe & 0xff) < 2)) {
    // PROPS_UI: Uses animation_key directly, IGNORES direction parameter
    SetAnimationForAnimationControlInternal(anim_ctrl, pwVar3, 0, animation_key, ...);
}
else {
    // CHARA: Uses animation_key as GROUP, direction as SEQUENCE within group
    if (WanTableSpriteHasAnimationGroup(sprite_id, animation_key)) {
        SetAnimationForAnimationControlInternal(anim_ctrl, pwVar3, animation_key, direction, ...);
    }
}
```

**Evidence:** `SetAnimationForAnimationControlInternal` at `0x0201C5C4`
```c
if (*(char *)&wan_header->sprite_type == '\x01') {
    // CHARA: animations[group_id][animation_id]
    pwVar3 = pwVar8->animations[animation_group_id].pnt[animation_id];
}
else {
    // PROPS_UI/Effects: animations[0][animation_id] - IGNORES group
    pwVar3 = pwVar8->animations->pnt[animation_id];
}
```

## Animation Info Structure

Pointed to by `wan_header->ptr_anim_info` (offset 0x00).

```c
struct wan_anim_info {
    void* unk_0x00;                  // 0x00: Unknown pointer
    void* unk_0x04;                  // 0x04: Unknown pointer
    void* anim_group_table;          // 0x08: Pointer to animation group table
    int16_t nb_anim_groups;          // 0x0C: Number of animation groups (CHARA)
    // ...
};
```

### Animation Group Table Entry

```c
struct anim_group_entry {
    void* pnt;           // 0x00: Pointer to animation sequence pointers
    int16_t anim_length; // 0x04: Number of sequences in this group
    // ...
};
```

## Sequence Count Extraction

Different sprite types report sequence count differently.

**Evidence:** `LAB_0201da00`
```c
short LAB_0201da00(wan_header* header) {
    uint8_t sprite_type = *(uint8_t*)(header + 0x8);
    void* ptr_anim_info = *(void**)(header + 0x0);
    
    if (sprite_type == 0 || sprite_type == 2) {
        // Effect sprites (PROPS_UI): return sequence count in group 0
        void* anim_group_table = *(void**)(ptr_anim_info + 0x8);
        return *(int16_t*)(anim_group_table + 0x4);  // anim_length
    } else {
        // Character sprites (CHARA): return number of animation groups
        return *(int16_t*)(ptr_anim_info + 0xC);  // nb_anim_groups
    }
}
```

| Sprite Type | Sequence Count Source | Used For |
|-------------|----------------------|----------|
| 0 (PROPS_UI) | `anim_group_table[0].anim_length` | Total sequences in group 0 |
| 1 (CHARA) | `ptr_anim_info->nb_anim_groups` | Number of animation groups |
| 2 (Unknown) | Same as type 0 | Total sequences in group 0 |

## WAN Table

Sprites are loaded into a global WAN table for runtime access.

### WAN Table Entry Structure

```c
struct wan_table_entry {
    char path[32];                   // 0x00: Null-terminated path string
    bool8 file_externally_allocated; // 0x20: External allocation flag
    uint8_t source_type;             // 0x21: 1=direct file, 2=pack file
    int16_t pack_id;                 // 0x22: Pack archive ID
    int16_t file_index;              // 0x24: File index within pack
    uint8_t field5_0x26;             // 0x26: Unknown
    uint8_t field6_0x27;             // 0x27: Unknown
    uint32_t iov_len;                // 0x28: Data length
    int16_t reference_counter;       // 0x2C: Reference count
    uint8_t field9_0x2e;             // 0x2E: Unknown
    uint8_t field10_0x2f;            // 0x2F: Unknown
    wan_header* sprite_start;        // 0x30: Pointer to WAN header
    void* iov_base;                  // 0x34: Points to SIR0 container
};
// Total size: 0x38 (56 bytes)
```

### WAN Table Access

**Evidence:** `FUN_0201da20`
```c
short FUN_0201da20(short sprite_id) {
    wan_table* table = *(wan_table**)(0x020AFC64 + 4);
    wan_table_entry* entry = &table->sprites[sprite_id];
    wan_header* header = entry->sprite_start;  // offset 0x30
    return LAB_0201da00(header);
}
```

## Memory Locations

| Symbol | Address (NA) | Description |
|--------|--------------|-------------|
| WAN_TABLE_PTR | `0x020AFC64 + 4` | Pointer to WAN table |

## Open Questions

- Complete wan_header structure beyond offset 0x0C
- Full wan_anim_info structure layout
- What sprite type 3 is used for (special 3D rendering mentioned)
- How SIR0 container wrapping works

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `SpriteTypeInWanTable` | `0x0201DA98` | Get sprite type from WAN table |
| `WanTableSpriteHasAnimationGroup` | `0x0201DAA0` | Check if sprite has animation group |
| `FUN_0201da20` | `0x0201DA20` | Sprite sequence count lookup from WAN_TABLE |
| `LAB_0201da00` | `0x0201DA00` | Extract sequence count from WAN header |
