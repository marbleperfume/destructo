# Tree of Savior: Re:Build Combat System

## Core Damage Formula (Official, n=948, 2017)

```
damage = ATK * (SFR/100) * min{1, log10((ATK/(DEF+1))^0.8 + 1)} + flat_bonus
```

- ATK == DEF -> ~30% of ATK gets through
- ATK >> DEF -> approaches 100%
- DEF can never make invincible (log-based)
- **Damage cap: 40,000,000 per hit**

## Modifier Chain (Player-Tested, Validated Structure)

Cross-tier MULTIPLICATIVE, within-tier ADDITIVE:

1. **Base Modifier** (SFR from skill level)
2. **Enhance Modifier** (+0.5%/lv, +10% at max. Per-skill attribute.)
3. **Arts Enhanced Upgrade** (+1.25%/lv. Per-skill. Max 30.)
4. **Common Modifier** (element matchup, Quick Cast, Commission to Kill +10%)
5. **Target Modifier** (armor matchup: Slash vs Plate x1.5, Physical vs Ghost x0.5, etc.)
6. **Bonus Damage** added flat at end (Blessing, Concentration)
7. **Additional Attacks** (Enchant Fire, Sacrament) have separate formula ignoring defense

## Critical Chance (Official n=951 + confirmed 60% cap)

```
crit_chance% = min(60, (max(0, crit_rate - crit_resistance))^0.6) + flat_class_bonus
```

- Hard capped at 60% for formula output
- Flat class buffs (Rogue Sneak Hit, Doppel Zornhau) bypass cap
- Diff of ~920 needed to reach cap: crit_to_cap = monster_CDEF + 12,750

## Critical Damage

```
crit_damage = normal_damage * 1.5 + critical_attack_stat
```

- Base crit multiplier: x1.5
- Critical Attack (CATK) adds flat bonus on crits

## Evasion Chance (Official n=951 + confirmed 50% cap)

```
evasion_chance% = min(50, (max(0, evasion - accuracy))^0.65)
```

- 50% HARD CAP
- Dead at endgame raids (boss accuracy 2,500-17,000+ erases player evasion)
- DEX no longer gives evasion or accuracy
- Removed from base armor

## Block Chance

```
block_chance = (max(0, block - block_penetration))^0.7
```

- Block effect: 50% damage reduction (halved damage, NOT nullification)
- Magic bypasses block entirely
- Endgame monsters DO have Block (ep18+ normals: BLK 12,480)
- Offensive penalty: if player BLK Pen < monster BLK -> physical hits glanced for half damage
- 90% of bosses have Block = 0 (BLK Pen irrelevant for boss DPS)

## Stat Breakpoint Constants (Reverse-Engineered, +/-4 accuracy)

```
EVA_DIFF_FOR_CAP  = 15,092   # 50% dodge
CRIT_DIFF_FOR_CAP = 12,750   # 60% crit
BLK_DIFF_FOR_CAP  =  3,642   # block cap
```

Breakpoint formulas:
- acc_to_cap = monster_eva + 1
- crit_to_cap = monster_cdef + 12,750
- blk_pen_to_cap = monster_blk
- eva_to_cap = monster_acc + 15,092
- cdef_to_cap = monster_crit + 1
- blk_to_cap = monster_blk_pen + 3,642

## AoE Defense Ratio (Corrected)

- Does NOT reduce hits on same target
- Determines how many TARGETS an AoE can hit (subtracts from AoE Attack Ratio budget)
- Solo boss: AoE DR irrelevant -- all multi-hit skills land fully
- Farming: Large mobs consume more AAR budget

## Armor Matchup Table

| Attack Type | vs Cloth | vs Leather | vs Plate | vs Ghost |
|------------|----------|------------|----------|----------|
| Slash      | x1.0     | x1.0       | x1.5     | x0.5     |
| Pierce     | x1.0     | x1.5       | x0.5     | x1.0     |
| Strike     | x1.5     | x0.5       | x1.0     | x1.0     |
| Magic      | x1.0     | x1.0       | x1.0     | x1.5     |
| Missile    | x1.0     | x1.0       | x1.0     | x1.0     |

## Skill System (Re:Build)

- No GCD. Animation lock per skill.
- Overheat (OH): charges that fire instantly, then one CD refills all.
- CD: cooldown timer per skill (starts when last OH charge used).
- SP: consumed per cast.
- Stance: weapon requirement (skills only usable in compatible stance).
- Property: innate element of skill (most physical = None).

## Attribute System

Three categories:
1. **Enhance** (max 100): +0.5%/lv to skill's SFR. +10% at max. Expensive.
2. **Arts: Enhanced Upgrade** (max 30): +1.25%/lv to SFR. Separate mult tier.
3. **Modifier**: Toggle on/off. Usually adds effect + SP cost increase.
4. **Arts: Behavior Change**: Fundamentally alters how skill works.

Enhance and Arts are MULTIPLICATIVE with each other.
At max investment: x1.60 * x1.375 = x2.20 per skill.
