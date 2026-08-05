# Type Matchup Pipeline

## Summary

- EoS has exactly **four** matchup buckets, not continuous multipliers
- Multipliers form a √2 ladder: `0.5×`, `~0.707×`, `1.0×`, `~1.398×`
- **`MATCHUP_IMMUNE` is 0.5×, not 0×** — the name is misleading
- True zero damage comes from only two hardcoded paths (Flash Fire, Levitate/floating), never from the type chart
- `TYPE_MATCHUP_TABLE` is a plain 18×18 array of `s16` — **not** bit-packed
- Dual types are resolved by a 4×4 combinator table, roughly (but not exactly) addition of √2 exponents
- The resulting `damage_data->type_matchup` is what selects hit reaction effects 8–11

## Memory Locations

| Symbol | Address (NA) | Type |
|--------|--------------|------|
| `TYPE_MATCHUP_TABLE` | `0x022C56B0` | `s16[18][18]` |
| `TYPE_MATCHUP_COMBINATOR_TABLE` | `0x022C4D14` | `s32[4][4]` |
| `MATCHUP_IMMUNE_MULTIPLIER` | `0x022C4758` | `fx32_8` |
| `MATCHUP_NOT_VERY_EFFECTIVE_MULTIPLIER` | `0x022C4810` | `fx32_8` |
| `MATCHUP_NEUTRAL_MULTIPLIER` | `0x022C481C` | `fx32_8` |
| `MATCHUP_SUPER_EFFECTIVE_MULTIPLIER` | `0x022C4818` | `fx32_8` |

Pointer indirections in `CalcTypeBasedDamageEffects`: `DAT_0230b784` → immune,
`DAT_0230b790` → not very effective, `DAT_0230b78c` → neutral, `DAT_0230b794` → super
effective. Note the pointers are **not** in enum order; the loading order into
`afStack_44[0..3]` is what establishes the correspondence.

## The Enum

```c
enum type_matchup {
    MATCHUP_IMMUNE              = 0,
    MATCHUP_NOT_VERY_EFFECTIVE  = 1,
    MATCHUP_NEUTRAL             = 2,
    MATCHUP_SUPER_EFFECTIVE     = 3,
};
```

Ascending by damage multiplier. Sourced and confirmed three independent ways:

1. **pret/pmd-sky** — `headers/types/dungeon_mode/enums.h`, explicit values.
2. **ROM row 0.** `TYPE_MATCHUP_TABLE[TYPE_NONE]` is entirely `02`. `TYPE_NONE` is the
   all-neutral row, so neutral is 2.
3. **Multiplier ordering.** The four constants are strictly ascending in enum order
   (see below), which only holds under this ordering.

## Damage Multipliers

| Enum | Value | Raw | fx32_8 | Decimal | √2 exponent |
|------|-------|-----|--------|---------|-------------|
| `MATCHUP_IMMUNE` | 0 | `0x080` | 128/256 | **0.5×** | −2 |
| `MATCHUP_NOT_VERY_EFFECTIVE` | 1 | `0x0B5` | 181/256 | ~0.707× | −1 |
| `MATCHUP_NEUTRAL` | 2 | `0x100` | 256/256 | 1.0× | 0 |
| `MATCHUP_SUPER_EFFECTIVE` | 3 | `0x166` | 358/256 | ~1.398× | +1 |

Each step is a factor of √2, which is why EoS damage swings are far gentler than
mainline's factor-of-2 steps.

**`MATCHUP_IMMUNE` does not mean zero damage.** A Ghost-type move against a Normal-type
target deals half damage and produces a normal hit reaction. This resolved a long-standing
apparent contradiction where Poison Fang visibly connected with Bronzor (Poison → Steel)
and Shadow Claw visibly connected with Zigzagoon (Ghost → Normal); both are correct ROM
behaviour.

### Erratic Player variant set

When either the attacker or defender has `IQ_ERRATIC_PLAYER` (and `partial` is false), a
different set of four multipliers is loaded: `DAT_0230b778`, `DAT_0230b77c`,
`DAT_0230b780`, `DAT_0230b788` → `afStack_44[0..3]`. **Not yet dumped.**

Erratic Player also changes the application rule. Normally the multiplier is skipped
entirely when the bucket is `MATCHUP_NEUTRAL`; with Erratic Player it is always applied,
so the neutral entry in that set is presumably not 1.0×.

**Evidence:** `CalcTypeBasedDamageEffects`
```c
bVar7 = IqSkillIsEnabled(attacker, IQ_ERRATIC_PLAYER);
if (bVar7 == '\0') {
    if (tVar8 != MATCHUP_NEUTRAL) {
        MultiplyFixedPoint64(damage_mult_out, damage_mult_out, afStack_44 + tVar8);
    }
} else {
    MultiplyFixedPoint64(damage_mult_out, damage_mult_out, afStack_44 + tVar8);
}
```

## TYPE_MATCHUP_TABLE

18×18 array of `s16`. Row index is the **attacking** type, column index the **defending**
type. Row stride `0x24` (18 × 2). Total size 648 bytes (`0x022C56B0`–`0x022C5937`).

**Evidence:** `GetTypeMatchup`
```c
return (int)*(short *)((uint)*(byte *)((int)pvVar3 + target_type_idx + 0x5e) * 2 +
                      attack_type * 0x24 + DAT_0230ad00);
```

`DAT_0230ad00` holds `0x022C56B0`.

### Accessor

```c
type_matchup GetTypeMatchupEntry(type_id attack, type_id defend) {
    return (type_matchup)*(s16 *)(TYPE_MATCHUP_TABLE + attack * 0x24 + defend * 2);
}
```

### Decoded Chart

Values are the enum: `0` = immune (0.5×), `1` = not very effective, `2` = neutral,
`3` = super effective. Column order matches row order.

```
                NON NRM FIR WAT GRS ELE ICE FGT PSN GRD FLY PSY BUG RCK GHO DRG DRK STL
TYPE_NONE       2   2   2   2   2   2   2   2   2   2   2   2   2   2   2   2   2   2
TYPE_NORMAL     2   2   2   2   2   2   2   2   2   2   2   2   2   1   2   2   2   1
TYPE_FIRE       2   2   1   1   3   2   3   2   2   2   2   2   3   1   2   1   2   3
TYPE_WATER      2   2   3   1   1   2   2   2   2   3   2   2   2   3   2   1   2   2
TYPE_GRASS      2   2   1   3   1   2   2   2   1   3   1   2   1   3   2   1   2   1
TYPE_ELECTRIC   2   2   2   3   1   1   2   2   2   0   3   2   2   2   2   1   2   2
TYPE_ICE        2   2   1   1   3   2   1   2   2   3   3   2   2   2   2   3   2   1
TYPE_FIGHTING   2   3   2   2   2   2   3   2   1   2   1   1   1   3   2   2   3   3
TYPE_POISON     2   2   2   2   3   2   2   2   1   1   2   2   2   1   1   2   2   0
TYPE_GROUND     2   2   3   2   1   3   2   2   3   2   0   2   1   3   2   2   2   3
TYPE_FLYING     2   2   2   2   3   1   2   3   2   2   2   2   3   1   2   2   2   1
TYPE_PSYCHIC    2   2   2   2   2   2   2   3   3   2   2   1   2   2   2   2   0   1
TYPE_BUG        2   2   1   2   3   2   2   1   1   2   1   3   2   2   1   2   3   1
TYPE_ROCK       2   2   3   2   2   2   3   1   2   1   3   2   3   2   2   2   2   1
TYPE_GHOST      2   0   2   2   2   2   2   2   2   2   2   3   2   2   3   2   1   1
TYPE_DRAGON     2   2   2   2   2   2   2   2   2   2   2   2   2   2   2   3   2   1
TYPE_DARK       2   2   2   2   2   2   2   1   2   2   2   3   2   2   3   2   1   1
TYPE_STEEL      2   2   1   1   2   1   3   2   2   2   2   2   2   3   2   2   2   1
```

Cross-verified against `pret/pmd-sky` `data/type_matchup_table.c`.

### The Five Immune Entries

Exactly five entries in the whole table are `0`. All were verified byte-by-byte in ROM.

| Attack → Defend | Address |
|---|---|
| Electric → Ground | `0x022C5776` |
| Poison → Steel | `0x022C57F2` |
| Ground → Flying | `0x022C5808` |
| Psychic → Dark | `0x022C585C` |
| Ghost → Normal | `0x022C58AA` |

**Ghost's other immunities are not in this table.** Normal → Ghost and Fighting → Ghost are
encoded as neutral (`2`) and handled by separate hardcoded logic:
`IsTypeIneffectiveAgainstGhost` and `GhostImmunityIsActive`, with `ScrappyShouldActivate`
as the bypass. This is called out in pret's `type_matchup_table.h` comment.

**Evidence:** `CalcTypeBasedDamageEffects`
```c
if (((bVar6 == '\0') && (bVar4 != '\0')) &&
   (bVar7 = GhostImmunityIsActive(attacker, defender, (int)(short)iVar12), bVar7 != '\0')) {
    tVar8 = MATCHUP_IMMUNE;
    *(undefined *)(*DAT_0230b798 + 0x1d4) = 1;
} else {
    tVar8 = GetTypeMatchup(attacker, defender, (int)(short)iVar12, attack_type);
}
```

Note the Ghost path yields `MATCHUP_IMMUNE` — the 0.5× bucket — not zero damage.

## GetTypeMatchup Special Cases

Three conditions short-circuit the table lookup:

| Condition | Result |
|---|---|
| Psychic attack, Dark defender, Miracle Eye active (status or exclusive item) | `MATCHUP_NEUTRAL` |
| Ground attack, gravity active, Flying defender | `MATCHUP_NEUTRAL` |
| Ground attack, gravity inactive, defender floating | `MATCHUP_IMMUNE` |

The attacker and defender pointers are used **only** for these condition checks. The table
lookup itself depends solely on the two type parameters.

## TYPE_MATCHUP_COMBINATOR_TABLE

Resolves the two per-type matchups into the single value stored in
`damage_data->type_matchup`. Row = first type's matchup, column = second type's.
`s32` entries, row stride `0x10`.

**Evidence:** `CalcTypeBasedDamageEffects`
```c
tVar8 = *(type_matchup *)(iVar12 + local_54[0] * 0x10 + local_54[1] * 4);
damage_out->type_matchup = tVar8;
```

```
        b=0  b=1  b=2  b=3
a=0  [   0    0    0    1  ]
a=1  [   0    1    1    2  ]
a=2  [   0    1    2    3  ]
a=3  [   1    2    3    3  ]
```

Symmetric. Row and column 2 are the identity, so single-typed monsters pass through
unchanged (the unused second type slot reads `TYPE_NONE`, whose entire chart row is
neutral).

### The formula, and its one exception

Adding √2 exponents and clamping gives `clamp(a + b - 2, 0, 3)`, which reproduces
**fifteen of the sixteen entries**. The exception is `[1][1]`, which is `1` where the
formula predicts `0`.

That deviation is deliberate and load-bearing: a double resist (~0.707² = 0.5× total,
mainline's 0.25×) would otherwise display using the bottom bucket's reaction, visually
indistinguishable from an immunity. The special case keeps it displaying as "not very
effective".

**Do not implement the formula.** Use the table verbatim.

### Displayed vs actual multiplier

The damage multiplier is the true **unclamped product** of both per-type multipliers,
accumulated in the loop before the combinator runs. The combinator output only affects
what is *displayed* — message log text and hit reaction effect ID. A quad-weakness
(~1.398² ≈ 1.95×) deals its full unclamped damage but still renders as bucket 3.

## True Immunity (Zero Damage)

Only two code paths produce an actual zero multiplier. Both are hardcoded in
`CalcTypeBasedDamageEffects`, both set `full_type_immunity`, and neither consults the type
chart.

| Path | Condition | Diagnostic byte |
|---|---|---|
| Flash Fire | Fire attack, `GetFlashFireStatus != 0` | `+0x1C6` |
| Levitate / floating | Ground attack, not Mold Breaker, `LevitateIsActive` or `IsFloating` | `+0x1C7` |

**Evidence:** `CalcTypeBasedDamageEffects` (Levitate path)
```c
if ((attack_type == TYPE_GROUND) &&
   (((bVar5 = AbilityIsActiveVeneer(attacker, ABILITY_MOLD_BREAKER), bVar5 == '\0' &&
     (bVar5 = LevitateIsActive(defender), bVar5 != '\0')) ||
    (bVar5 = IsFloating(defender), bVar5 != '\0')))) {
    *(undefined *)(*DAT_0230b798 + 0x1c7) = 1;
    IntToFixedPoint64(damage_mult_out, 0);
    bVar4 = '\0';
    damage_out->type_matchup = MATCHUP_IMMUNE;
    damage_out->critical_hit = '\0';
    damage_out->full_type_immunity = '\x01';
}
```

Both set `type_matchup = MATCHUP_IMMUNE`, but the zero multiplier means `ApplyDamage`
takes its `damage == 0` branch and returns before reaching the hit reaction call. **No
reaction effect plays for a true immunity** — only the miss/block sound via
`FUN_022e576c`.

> See `Systems/damage_visual_pipeline.md` → "Zero-Damage Path Skips the Reaction".

## Hit Reaction Effect Selection

`FUN_022e5478` maps the stored `type_matchup` directly onto effect IDs 8–11.

| type_matchup | Value | Effect ID | Multiplier | Meaning |
|---|---|---|---|---|
| `MATCHUP_IMMUNE` (and `default`) | 0 | **8** | 0.5× | Heavily resisted |
| `MATCHUP_NOT_VERY_EFFECTIVE` | 1 | **9** | ~0.707× | Resisted |
| `MATCHUP_NEUTRAL` | 2 | **10** | 1.0× | Neutral |
| `MATCHUP_SUPER_EFFECTIVE` | 3 | **11** | ~1.398× | Super effective |

`case 0` sharing a target with `default` is consistent with the jump table at
`0x022E54A4` (`cmp r0,#0x3` / `addls pc,pc,r0,lsl #0x2`), where the case-0 slot and the
out-of-range fallthrough both branch to `mov r1,#0x8`.

> See `Systems/damage_visual_pipeline.md` for the full reaction sequence and
> `Data Structures/effect_animation_info.md` for rows 8–11.

## Hidden Effectiveness Override (FUN_0230d618)

Gated on dungeon flag `+0x1C5`, which is **not** an IQ skill. It is a damage-calculation
diagnostic byte set by `CalcTypeBasedDamageEffects` in exactly two places — **Thick Fat**
and **Heatproof** — each alongside a halving multiplier. The diagnostics block
(`+0x184`–`+0x1D5`) is cleared per hit by `ResetDamageCalcDiagnostics`.

```c
undefined4 FUN_0230d618(int param_1)
{
    return *(undefined4 *)(DAT_0230d624 + param_1 * 4);
}
```

The table at `0x02352894` is `{0, 0, 0, 1}`:

| Real matchup | Displayed as |
|---|---|
| `MATCHUP_IMMUNE` (0) | `MATCHUP_IMMUNE` (0) |
| `MATCHUP_NOT_VERY_EFFECTIVE` (1) | `MATCHUP_IMMUNE` (0) |
| `MATCHUP_NEUTRAL` (2) | `MATCHUP_IMMUNE` (0) |
| `MATCHUP_SUPER_EFFECTIVE` (3) | `MATCHUP_NOT_VERY_EFFECTIVE` (1) |

A one-step downgrade of the visual: the ability blunted the hit, so a weaker reaction
plays. It affects the effect ID only, not the damage — the halving is applied separately
in `CalcTypeBasedDamageEffects`.

## Other Matchup-Dependent Effects

Read from `damage_out->type_matchup` **after** the combinator, so these key off the
displayed bucket:

| Ability / Item | Condition | Effect |
|---|---|---|
| Tinted Lens | matchup == `NOT_VERY_EFFECTIVE` | `DAT_0230b7a0` multiplier |
| Solid Rock / Filter | matchup == `SUPER_EFFECTIVE` | `DAT_0230b7a4` multiplier |
| Wonder Guard | matchup != `SUPER_EFFECTIVE`, type != `TYPE_NONE` | forces multiplier to `DAT_0230b774 + 0x3C` |
| Type-Advantage Master (IQ) | matchup == `SUPER_EFFECTIVE` | function returns true → boosts crit chance |

`CalcTypeBasedDamageEffects` returns whether Type-Advantage Master should activate, which
in practice means "was this super effective" (also true when the defender is invalid).

## Client Implementation Guidance

A client using a mainline Gen 9 chart with continuous multipliers must map into the four
ROM buckets. Decompose the mainline result into its per-type components, run them through
the combinator, and read off the bucket:

| Server multiplier | Per-type decomposition | Bucket | Effect ID |
|---|---|---|---|
| 4.0 | super + super → `[3][3]` | 3 | 11 |
| 2.0 | super + neutral → `[3][2]` | 3 | 11 |
| 1.0 | neutral + neutral → `[2][2]` | 2 | 10 |
| 1.0 | super + resist → `[3][1]` | 2 | 10 |
| 0.5 | resist + neutral → `[1][2]` | 1 | 9 |
| 0.25 | resist + resist → `[1][1]` | 1 | 9 |
| 0.0 | immune present → row/col 0 | 0 | 8, or none |

**Collapsing 0.25 with 0.5 into a single bucket is correct** and follows directly from the
`[1][1]` special case. Collapsing 2.0 with 4.0 is likewise correct.

For 0.0 there is a genuine design choice, because a mainline `0.0` corresponds to two
different ROM situations:

- **ROM-faithful:** treat it as true immunity — play the miss/block sound and no reaction
  effect at all, matching the Flash Fire / Levitate path.
- **Pragmatic:** play effect 8, the weakest reaction, so every attack produces some visual
  feedback. Defensible in a fast-paced client where silence reads as a dropped frame.

What should not happen is effect 11 on an immunity, which is what the previous
(mislabelled) mapping produced.

Note also that a client with a Gen 9 chart will emit `Immune` for Normal → Ghost and
Fighting → Ghost, which EoS routes through `GhostImmunityIsActive` rather than the chart,
and for type pairs EoS's chart does not contain at all.

## Cross-References

> See `Systems/damage_visual_pipeline.md` for the hit reaction sequence and effect IDs 8–11

> See `Data Structures/effect_animation_info.md` for effect rows 8–11 and `anim_type` 1 resolution

## Open Questions

- The Erratic Player multiplier set (`DAT_0230b778`, `0230b77c`, `0230b780`, `0230b788`)
- Why `[1][1]` in the combinator deviates from the exponent-addition rule — presumed
  deliberate, but no confirming evidence beyond the behaviour itself
- Full layout of the damage-calc diagnostics block at `DUNGEON_PTR + 0x184`–`0x1D5`
- Whether `damage_data->full_type_immunity` is read anywhere that changes visuals, beyond
  the `damage == 0` branch in `ApplyDamage`
- What `DAT_0230b774 + 0x1C` / `+0x20` hold (the fallback matchup pair used when the
  per-type loop breaks early on a zeroed multiplier)

## Functions Used

| Function | Address (NA) | Purpose |
|----------|--------------|---------|
| `GetTypeMatchup` | — | Chart lookup with Miracle Eye / gravity / floating special cases |
| `CalcTypeBasedDamageEffects` | — | Per-type loop, combinator, multipliers, STAB, pinch abilities |
| `CalcDamage` | — | Main damage formula; calls the above |
| `GhostImmunityIsActive` | — | Hardcoded Normal/Fighting → Ghost immunity |
| `IsTypeIneffectiveAgainstGhost` | — | Type check for the Ghost immunity path |
| `ScrappyShouldActivate` | — | Bypass for the Ghost immunity path |
| `GetFlashFireStatus` | — | Flash Fire true-immunity check |
| `LevitateIsActive` | — | Levitate true-immunity check |
| `IsFloating` | — | Floating check, used in both `GetTypeMatchup` and the true-immunity path |
| `GravityIsActive` | — | Gates the Ground/Flying special case |
| `ResetDamageCalcDiagnostics` | — | Clears the `+0x184`–`+0x1D5` block per hit |
| `FUN_022e5478` | `0x022e5478` | Maps `type_matchup` to hit reaction effect ID |
| `FUN_0230d618` | `0x0230d618` | Thick Fat / Heatproof reaction downgrade |
