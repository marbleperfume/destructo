# Tree of Savior — Gear System & ATK Calculation Context

## Overview

ToS gear does NOT decompose into individual equipment slots for the template system.
The user inputs **total ATK/MATK** as a single number representing all gear combined.
This document teaches the agent what contributes to that number so it can:
1. Validate whether a user's stated ATK is realistic for their level/investment
2. Understand which multipliers stack and how
3. Model Vaivora investment tiers correctly

---

## ATK Sources (What Feeds Into "Total ATK")

### 1. Weapon Base Attack
- Primary stat source. Endgame weapons: 8,000-15,000 base ATK
- Enhancement (+21 max): roughly doubles base ATK
- Transcendence (×10 stages): +10% per stage to base
- Awakening: flat +500-2000 per awakening level

### 2. Armor Set Bonuses
- Endgame sets (Luciferie, Assistera, Skiaclipse): +2000-6000 ATK from set effects
- Fixed armor isn't swappable per fight — flat contribution to total

### 3. Accessories
- Bracelets, necklaces: +1000-3000 ATK each
- Goddess/Demon accessories: conditional +% bonuses

### 4. Cards (Equip + Legend)
- Equip cards: flat +ATK (200-800 per slot, 4 slots)
- Legend cards: conditional multipliers (+15% vs Boss, etc.)

### 5. Gems (3 types)
- Red gems: flat +ATK in weapon slots
- Endgame contribution: 2000-5000 total from gems

### 6. Stat Points (CON/STR/INT/SPR/DEX)
- STR → +1 physical ATK per point (trivial at endgame)
- INT → +1 magic ATK per point
- Stats are ~1000 points at lv460 — negligible vs gear

### 7. Buffs & Consumables (NOT modeled in total ATK)
- SP potions, ATK buffs from other players
- Template treats these as EXTERNAL — user can add them as a multiplier if desired

---

## Realistic ATK Ranges by Investment Level

| Tier | Total ATK | Description |
|------|-----------|-------------|
| Fresh 460 | 30,000-50,000 | Just hit endgame, basic Skiaclipse set |
| Mid investment | 80,000-150,000 | Enhanced +16, decent accessories |
| High investment | 200,000-350,000 | +21 weapon, full legend cards, ark |
| Whale | 400,000-600,000+ | Perfect rolls, all goddess gear, maxed everything |

At endgame ATK (200k+), most content dies instantly (DEF factor → 1.0).
The interesting modeling space is 50k-200k where defense matters.

---

## Vaivora Weapon System (Multiplicative Layer)

Vaivora weapons sit OUTSIDE the base ATK number. They apply as:
```
effective_skill_damage = base_damage × vaivora_multiplier
```

Where `vaivora_multiplier` includes:
- Skill level bonus (+SFR from higher array index)
- Final Damage % (+30-100% on designated skill)
- Proc expected value (dead time fills)
- Weapon swap bonus (+40% during swap window)

### Modeling Tiers

| Investment | Components | Effective Multiplier |
|------------|-----------|---------------------|
| None | No Vaivora | ×1.00 |
| Low | Lv1 Vaivora, no swap | ×1.15-1.30 |
| Mid | Lv2 Vaivora, basic swap | ×1.40-1.70 |
| Max | Lv3 Vaivora, full swap system | ×1.80-2.50 |

These multiply with the Enhance/Arts chain:
```
total_modifier = enhance_mult × arts_mult × commission_mult × vaivora_mult
```

At max everything: ×2.42 (Enhance+Arts+Commission) × ×2.50 (Vaivora) = **×6.05 total**

---

## Bonus Damage ("Attack vs. X") System

Separate from main ATK. Grants percentage bonus AFTER defense calculation:
```
bonus_percent = attack_vs_property / (monster_level × 30) × 100
```

Sources:
- Vaivora weapons (large values: 5,000-15,000)
- Ark equipment (2,000-8,000)
- Enchant scrolls (1,000-3,000)
- Set effects (conditional)

At lv500 target:
- 7,500 attack_vs = +50% damage
- 15,000 attack_vs = +100% damage (doubles)
- 30,000 attack_vs = +200% damage (triples)

**Key insight**: This is where whale builds separate from mid builds.
A mid player might have 5,000 attack_vs (33% bonus).
A whale has 25,000+ attack_vs (166% bonus) — 2× DPS from this stat alone.

---

## Template System Input Design

User provides:
- `total_atk`: Single number (all gear sources combined)
- `vaivora_tier`: none | low | mid | max
- `attack_vs_value`: Optional, for bonus damage modeling (default 0)
- `enhance_level`: Per-skill or global default (0-100)
- `arts_level`: Per-skill or global default (0-30)

The template does NOT ask about:
- Individual equipment pieces
- Enhancement levels on armor
- Card compositions
- Gem layouts

This keeps the input clean while capturing the meaningful variables.

---

## Why Equipment Is Collapsed

1. **Too many combinations** — 12+ equipment slots × 100+ options each = unmeasurable
2. **Not gameplay-relevant** — gear is a number, builds are about skill choices
3. **Changes constantly** — new gear every major patch, old gear obsolete
4. **P2W variance** — impossible to set "standard" gear (whale vs F2P is 3-6×)
5. **User already knows their ATK** — visible on character sheet

The template system's job: given YOUR numbers, what does your ROTATION produce?
