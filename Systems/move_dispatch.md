# Move Effect Dispatch

## Summary

- Every move's effect is a `DoMoveX` wrapper with a uniform signature
- Move-use flows through `FUN_02322374` (multi-strike executor) → `ExecuteMoveEffect` (per-target loop + dispatcher)
- `ExecuteMoveEffect` dispatches to the correct `DoMoveX` via a jump table indexed by move ID
- The jump table is an array of ARM `b` instructions, NOT function pointers
- Each entry branches to a small trampoline that `bl`s the real handler
- Decode any handler with: `entry_addr + 8 + (offset << 2)`

## Call Chain

```
FUN_02322374 (move-use executor)
  ├─► Metronome / Nature Power substitution
  ├─► Multi-strike loop (Skill Link, Twineedle, etc.)
  ├─► BuildMoveTargetList
  └─► ExecuteMoveEffect (per iteration)
        ├─► per-target loop (Snatch / Magic Coat / Mirror Move / hit-check)
        └─► jump table dispatch → DoMoveX
```

## FUN_02322374 (Move-Use Executor)

The core move execution function. Reached from `ExecuteMonsterAction`'s move-use path and from the Bide unleash tick.

Responsibilities:
- **Metronome** (`move->id == DAT_02322d18`): `DungeonRandInt(0xa8)` picks a random move, re-inits into a local `move` struct
- **Nature Power** (`move->id == 0x77`): `GetNaturePowerVariant()` selects the terrain move
- **Multi-strike loop**: strike count from `FUN_02324514`, or `GetMoveNbStrikes` when Skill Link is active (defaults to 5 if 0)
- **Target list**: `BuildMoveTargetList` + `FUN_02324c9c`
- **Dispatch**: calls `ExecuteMoveEffect` (normal) or `FUN_023230fc` (projectile path, gated by `FUN_022bfd6c`)
- Decrements PP, handles two-turn/charge moves, muzzle-fail, Gravity move-blocking, Intimidator

**Evidence:** `FUN_02322374` Metronome path
```c
if (*(ushort *)(param_6 + 4) == DAT_02322d18) {
    uVar8 = DungeonRandInt(0xa8);
    *DAT_02322d1c = uVar8;
    InitMove(&local_138,(uint)*(ushort *)(DAT_02322d20 + uVar8 * 8));
    // ...
    move = &local_138;
}
```

## ExecuteMoveEffect (Dispatcher)

Handles post-use effects: the per-target loop, ability checks, and the giant move-ID switch that executes the effect.

Signature:
```c
void ExecuteMoveEffect(target_list *targets, entity *attacker, move *move,
                       undefined4 param_4, undefined4 param_5);
```

The per-target loop body handles, in order: Snatch (`CanBeSnatched`), Magic Coat / bounce (`IsReflectedByMagicCoat` + `EXCLUSIVE_EFF_MAY_BOUNCE_STATUS_MOVES`), Mirror Move (`MirrorMoveIsActive`), Soundproof, Forewarn, then `MoveHitCheck`, then the dispatch.

`TryEndPetrifiedOrSleepStatus(attacker, target)` fires per-target inside this loop, before the hit-check (wake-on-hit).

> See `status_visual_pipeline.md` for the wake-on-hit interaction

## The Jump Table

At the end of the per-target loop, dispatch occurs via an indirect branch:

**Evidence:** `ExecuteMoveEffect` dispatch
```c
mVar15 = DAT_0232f85c;                        // max valid move_id (0x21E)
peVar26[1].field_0xaa = '\x01';
if (move_id <= mVar15) {
    (*(code *)(move_id * 4 + 0x232f8b8))();    // jump table
    return;
}
```

**Evidence:** ARM disassembly at the dispatch site
```
0232f8a0  ldr    r0,[DAT_0232f85c]        = 0000021Eh
0232f8ac  cmp    r6,r0
0232f8b0  addls  pc,pc,r6, lsl #0x2        ; jump: pc += move_id*4
0232f8b4  b      LAB_023326c8             ; default (move_id > 0x21E)
0232f8b8  <jump table begins>
```

### Table Properties

| Property | Value |
|----------|-------|
| Table Base (NA) | `0x0232f8b8` |
| Index | `move_id` (0 to `0x21E`) |
| Entry Size | 4 bytes |
| Entry Type | ARM `b` instruction (opcode `0xEA`) |
| Max move_id | `0x21E` (from `DAT_0232f85c`) |

### Entry Format

Each entry is an ARM branch instruction, **not** a function pointer. Layout (little-endian): low 3 bytes = signed 24-bit offset, high byte = `0xEA` (`b` opcode).

**Decode formula:**
```
branch_target = entry_addr + 8 + (sign_extend(offset_24) << 2)
```

Each branch lands on a short trampoline that loads the arguments and `bl`s the real `DoMoveX`:

**Evidence:** Baton Pass (move `0xFC`) dispatch
```
0232fca8:  4a 06 00 ea    b LAB_023315d8      ; table[0xFC]

LAB_023315d8:
    mov r0,r9              ; attacker
    mov r1,r4             ; defender
    mov r2,r8            ; move
    mov r3,r7           ; item_id
    bl  DoMoveSwitchPositions
    mov r10,r0
    b   LAB_023326cc      ; loop continuation
```

Worked decode for `0xFC`: `0x0232fca8 + 8 + (0x64a << 2) = 0x023315d8`, matching the trampoline above.

### Reading an Arbitrary Handler

To resolve `move_id → DoMoveX` without hand-decoding:

1. In Ghidra, `G` to `0x0232f8b8 + move_id*4`, press `D` — renders as `b LAB_xxxx`
2. `G` to that label, read the `bl` target at the bottom of the trampoline

Or via Script Manager:
```python
base = toAddr(0x0232f8b8)
tgt  = base.add(move_id * 4)
raw  = getInt(tgt) & 0xFFFFFFFF
off  = raw & 0x00FFFFFF
if off & 0x800000: off -= 0x1000000
dest = tgt.add(8 + (off << 2))
f = getFunctionContaining(dest)
print(dest.toString() + "  " + (f.getName() if f else "trampoline"))
```

## DoMoveX Wrapper Pattern

Every move effect is a `DoMoveX` function:
```c
bool DoMoveX(entity *attacker, entity *defender, move *move, item_id item_id);
```

Wrappers are thin. They contain no VFX calls, no stat writes, and no immunity checks — they call one or more shared mechanic functions (`TryInflict*`, `Boost*Stat`, `EndNegativeStatusCondition`, etc.) and return `true`.

Multiple move IDs can share a single handler. Examples:
- `DoMoveDamageConstrict10` — Clamp, Bind, Sand Tomb, Fire Spin, Magma Storm
- `DoMoveShadowHold` — Spider Web, Mean Look
- `DoMoveHealStatus` — Refresh, Heal Bell, Aromatherapy
- `DoMoveSwitchPositions` — Baton Pass, Switcher (item)

For an undecompiled `DoMoveX`, match the `bl` branch target to a known shared mechanic function to identify its behavior.

## Open Questions

- Full `move_id → DoMoveX` mapping (dump the whole table via the script above)
- Purpose of `param_4` / `param_5` passed through `ExecuteMoveEffect`
- Counter's damage-reflection path (Magic Coat's bounce is in the per-target loop; Counter's `0xD5 == 4` damage return is not yet traced)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `FUN_02322374` | `0x02322374` | Move-use executor (multi-strike, Metronome, Nature Power) |
| `ExecuteMoveEffect` | — | Per-target loop + jump-table dispatcher |
| `ExecuteMonsterAction` | — | Top-level action dispatch (reaches move-use path) |
| `BuildMoveTargetList` | — | Populate target list for a move |
| `FUN_023230fc` | `0x023230fc` | Projectile execution path |
| `FUN_02324514` | `0x02324514` | Strike count for multi-hit moves |
| `GetMoveNbStrikes` | — | Skill Link strike count |
| `TryEndPetrifiedOrSleepStatus` | — | Wake-on-hit, per-target in the loop |

## Cross-References

> See `move_effect_mechanics.md` for the `DoMoveX` wrapper pattern and stat-mechanic functions

> See `Moves/binding_moves.md`, `Moves/status_cure_moves.md`, `Moves/reflect_moves.md`, `Moves/baton_pass.md` for specific handlers
