# PMD Explorers of Sky: Battle Visual System Documentation

> Reverse engineering the visual and animation systems used in Pokémon Mystery Dungeon: Explorers of Sky.

This repository contains reverse engineering research for Pokémon Mystery Dungeon: Explorers of Sky, focused on understanding the move effect animation system. The goal is to document how the game renders move animations, projectiles, and visual effects to enable accurate recreation in external tools.

**ROM Version:** NA release (C2SE)

**Tools Used:**
- Ghidra (decompilation)
- pmdsky-debug (symbols and structure definitions)
- pmd-sky decompiled source (reference)
- DeSmuME (runtime testing)

## Quick Reference

### Memory Addresses

| Table | NA | EU | JP |
|-------|----|----|-----|
| `MOVE_ANIMATION_INFO` | `0x022C9064` | `0x022C99BC` | `0x022CA74C` |
| `EFFECT_ANIMATION_INFO` | `0x022CC52C` | `0x022CCE84` | `0x022CDC14` |

### Constants

| Constant | Value | Hex | Description |
|----------|-------|-----|-------------|
| TILE_SIZE | 24 | 0x18 | Pixels per tile |
| FIXED_POINT_SHIFT | 8 | - | 8.8 fixed point format |
| FIXED_POINT_MULTIPLIER | 256 | 0x100 | Multiply to convert to fixed point |
| ITEM_OFFSET_X | 4 | 0x04 | Item X offset from tile corner |
| ITEM_OFFSET_Y | 4 | 0x04 | Item Y offset from tile corner |
| ENTITY_OFFSET_X | 12 | 0x0C | Monster X offset (tile center) |
| ENTITY_OFFSET_Y | 16 | 0x10 | Monster Y offset (feet position) |
| SILENCE_SFX | 16128 | 0x3F00 | Sound effect ID for silence |
| MAX_EFFECTS | 32 | 0x20 | Maximum simultaneous effects |
| EFFECT_CONTEXT_SIZE | 316 | 0x13C | Bytes per effect context |

### Direction Values

| Index | Direction | X Delta | Y Delta |
|-------|-----------|---------|---------|
| 0 | Down | 0 | +1 |
| 1 | Down-Right | +1 | +1 |
| 2 | Right | +1 | 0 |
| 3 | Up-Right | +1 | -1 |
| 4 | Up | 0 | -1 |
| 5 | Up-Left | -1 | -1 |
| 6 | Left | -1 | 0 |
| 7 | Down-Left | -1 | +1 |

### Projectile Speed

| Raw Value | Mapped | Frame Count | Description |
|-----------|--------|-------------|-------------|
| 0 | 6 | 4 | Fast/instant |
| 1 | 2 | 12 | Slow |
| 2 | 3 | 8 | Medium |
| Other | 6 | 4 | Fast (default) |

### Attachment Points

| Index | Name | Description |
|-------|------|-------------|
| -1 / 0xFF | None | No lookup, default position |
| 0 | Head | Top of sprite |
| 1 | LeftHand | Left appendage |
| 2 | RightHand | Right appendage |
| 3 | Centre | Body center |

### Effect Animation Types

| Value | Name | Description |
|-------|------|-------------|
| 0 | Invalid | No effect (returns -1) |
| 1 | WanFile0 | Standard WAN, requires state == 0 |
| 2 | WanFile1 | Standard WAN, requires state == 1 |
| 3 | WanOther | Standard WAN, clears conflicting type-3 |
| 4 | Wat | WAT format, clears conflicting type-4 |
| 5 | Screen | Screen effect, uses file_index + 0x10C |
| 6 | Wba | WBA format, special allocation |

### Move Animation Flags (offset 0x08)

| Bit | Mask | Purpose |
|-----|------|---------|
| 0-2 | 0x07 | Animation category |
| 3 | 0x08 | Dual-target effect |
| 4 | 0x10 | Skip fade-in |
| 5 | 0x20 | Unknown |
| 6 | 0x40 | Post-animation delay |
| 7 | 0x80 | Unknown |

### Monster Animation Types

| Value | Animation | Notes |
|-------|-----------|-------|
| 0-12 | Standard | Walk, Attack, Sleep, etc. |
| 13-34 | Starter-Specific | Wake, Eat, Tumble, Pose, Nod, etc. |
| 98 | Multi-direction | 9 attacks, +2 direction each |
| 99 | Spin | 8 directions sequentially |

