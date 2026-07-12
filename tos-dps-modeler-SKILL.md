---
name: tos-dps-modeler
display_name: ToS DPS Modeler
description: "Model Tree of Savior combat economics: ATK vs DEF thresholds, SP economy over sustained fights, dead time taxonomy, and Vaivora impact at budget/intermediate/high-end gear tiers. Uses web_search to contextualize stat expectations at target level before modeling. Activate for ToS DPS checks, SP economy analysis, stat threshold calculations, dead time breakdowns, Vaivora effectiveness, or farming tier comparisons."
icon: "sword"
trigger: ToS Tree of Savior DPS model dead time rotation vaivora SP economy stat threshold farming budget intermediate endgame kill time ATK DEF mana cost
patterns:
  - {pattern: "\\b(?:tree of savior|tos)\\b.*(?:dps|damage|dead time|rotation|farm)", confidence: 0.92}
  - {pattern: "\\b(?:budget|intermediate|high.?end)\\b.*(?:tier|atk|gear|clear)", confidence: 0.85}
  - {pattern: "\\bvaivora\\b.*(?:tier|compare|damage|proc|fill|gap)", confidence: 0.90}
  - {pattern: "\\bsp\\b.*(?:econ|drain|potion|regen|sustain|mana)", confidence: 0.85}
  - {pattern: "\\bdead time\\b.*(?:graph|visual|model|farm|category)", confidence: 0.85}
inputs:
  - name: class_name
    description: "Primary class build to model (e.g., 'Warlock-Shadowmancer-Featherfoot' or single class name)"
    type: string
    required: true
  - name: target_level
    description: "Content level to model against (e.g., 430, 460, 550)"
    type: number
    default: 430
  - name: fight_duration
    description: "Simulation window in seconds (60 for burst, 300 for sustained SP economy)"
    type: number
    default: 300
  - name: content_type
    description: "Target scenario: 'cm', 'field_farming', 'saalus', 'boss_raid'"
    type: string
    default: cm
  - name: output_format
    description: "Visualization: 'scatter', 'sp_economy', 'tier_comparison', 'all'"
    type: string
    default: all
tools: [web_search, url_fetch, run_python, file_read, file_write, open_in_session_tab]
depends-on: [highcharts, html_design]
---


## Purpose

Test threshold viability for Tree of Savior builds across gear investment tiers. Measures:
- Whether a build at a given gear tier can clear target content
- Where dead time exists and what category it falls into
- How SP economy constrains sustained farming
- What Vaivora weapons change about the damage profile

This skill does NOT find optimal rotations. It quantifies differences between gear tiers, measures investment thresholds, and identifies structural design constraints.


## Domain Rules

### Damage Formula (Post-Re:Build)

```
effective_atk = max(1, player_ATK - enemy_DEF)
damage_per_hit = effective_atk * (SFR% / 100) * enhance_mult
total_skill_damage = damage_per_hit * hits * (1 + sum_of_FD_bonuses)
```

- **ATK - DEF is flat subtraction.** This is the primary bottleneck at budget gear. If enemy DEF is 78% of your ATK, you deal 22% of your theoretical output.
- **SFR%** = Skill Factor Rate. Scales per skill level. Found in class JSON files.
- **Enhance multiplier** = `1 + (0.005 * enhance_lv) + (0.10 if enhance_lv == 100)`. Arts Enhanced Upgrade adds `1.25% * arts_lv`.
- **Final Damage (FD)** bonuses are additive with each other, then multiplicative with base: `(1 + FD1 + FD2 + ...)`.
- **Hits** are per-skill. Some skills have multi-tick mechanics (Pole of Agony: 2 hits per 0.5s for 5s = 20 hits).

### SP Economy

SP regen is **tick-based**: one tick every 10 seconds while standing/moving in combat. NOT continuous.

| Tier | SP Pool | Regen/10s tick | Regen/sec | Sustain Tools |
|------|---------|----------------|-----------|---------------|
| Budget | 18-22k | 450-600 | 45-60 | Lv15 Condensed SP Potions, Basic Firewood |
| Intermediate | 24-30k | 900-1,300 | 90-130 | Lv5-7 Blue Gems (Gloves), 10-star Rikaus Cards |
| High-End | 35-42k+ | 2,200-3,000+ | 220-300 | Lv9-10 Blue Gems, Max SPR Ichors, Goddess Card sets |

**SP Potion:** Alchemist Lv15 Condensed = ~2,500-3,500 SP restored. Cooldown ~30s. SP is a silver tax at budget tier.

**Bonfire regen is out-of-combat only.** Do not include in fight simulations.

**Full build (3 classes) costs 130-170 SP/s.** Single-class rotations NEVER run out because there aren't enough buttons. Always model full 3-class builds for SP economy.

### Dead Time Categories

1. **CD dead time** -- All skills on cooldown. Structural. The class doesn't have enough short-CD skills to fill the timeline.
2. **SP exhaustion dead time** -- Skill is off CD but player can't afford the SP cost. Investment problem (silver tax).
3. **Dependency dead time** -- Skill off CD but prerequisite not met (spirits not accumulated, buff not active, stance not correct).
4. **Autonomous fill** -- Damage during player CD gaps from: spirits attacking, DoTs ticking, summon AI, persistent AoE zones. This is NOT true dead time -- classify separately.

### Spirit/Summon Mechanics (Warlock Example)

Spirits are autonomous damage sources that attack DURING player CD gaps:
- Each spirit attacks with Invocation SFR per hit
- Standard spirit: 3 hits. High-level (10% attribute chance): 5+ hits
- Expected: 3.2 hits/spirit average
- **Ghastly Trail buff:** +100% spirit damage (8.5s CD, 10s duration = near-permanent uptime)
- **Pole of Agony [Arts]:** Generates 1 spirit per 0.5s during 5s pillar = 10 spirits/cast
- **Demon Scratch (Vaivora):** 50% chance to create 1 spirit per cast
- **Evil Sacrifice:** 0 CD, 37 SP -- sends all accumulated spirits to attack. Spam freely in gaps.

Demonische (Vaivora proc):
- Each spirit hit generates stacks (1 standard, 2 high-level). Expected: 3.7 stacks/spirit.
- At 10 stacks: auto-fires 5 hits at 200% Invocation SFR.
- With 40 spirits/min -> ~14-18 Demonische procs/min.

### Vaivora Design Patterns

Effect data in: `tos/vaivora/vaivora_effects_{tree}.json`

- **linear_proficiency** -- Buffs existing rotation without changing shape. +FD%, +skill levels, CD reduction, extra hits appended to existing skills.
- **gap_filling** -- Adds NEW damage sources during CD gaps. Persistent AoE zones, counter/reactive mechanics, autonomous summons, DoTs from triggers.

Model before/after Vaivora to quantify the delta at each gear tier.

### Crit/Block Layer (Optional Emphasis)

Enable when modeling against a specific monster with known combat stats, or when
demonstrating the full compounding budget penalty. This is a multiplier layer on
top of the base damage calc -- it does not change the rotation simulation.

**Critical Hit:**
```python
# Pre-loaded function (already in run_python scope):
crit_chance = compute_tos_crit_chance(player_crit_rate, enemy_crit_resist)
# Formula: min(60, max(0, crit - cdef)^0.6)
# Cap: 60% from formula. Flat class bonuses (Rogue Sneak Hit) bypass cap.
# Diff ~920 = cap. Budget players often sit at 15-25%, endgame at 50-60%.
crit_multiplier = 1 + (crit_chance / 100) * 0.5  # crits deal +50% damage
```

**Block:**
```python
# If monster has Block stat and player lacks Block Penetration:
# Blocked hits deal 0.75x damage (25% penalty)
block_rate = max(0, enemy_block - player_block_pen)  # simplified
block_penalty = 1 - (block_rate_chance * 0.25)  # effective multiplier
```

**Combined budget penalty example (Lv430 CM, worst case):**
- Effective ATK after DEF: 22% (ATK-DEF wall)
- x0.75 (no Block Pen vs blocking mob)
- x1.12 crit mult (budget 25% crit vs endgame x1.30 at 60% crit)
- = ~18.5% of theoretical SFR damage actually lands

**When to enable:**
- Monster data includes acc/eva/crit/cdef/blk/blk_pen stats
- User specifically asks about crit scaling or "why does my damage feel low"
- Modeling a named boss (Zmei, Kepari, etc.) where stats are scraped/known

**When to skip:** Generic tier comparison where the emphasis is on ATK-DEF wall
and SP economy. Crit/Block adds precision but the core argument holds without it.


## Workflow

### Step 1: Contextualize via Web Search

Before modeling ANY class, establish stat context for the target level:

```python
# Player stat expectations
web_search(query="tree of savior level {LEVEL} {TREE} attack range budget endgame 2024")
web_search(query="tos {CONTENT} monster defense HP stats level {LEVEL}")
web_search(query="tos level {LEVEL} SP pool regen budget intermediate high end")
```

Extract and structure into 3 tiers:
- **Budget:** Free quest gear, un-transcended, no ichors, f2p
- **Intermediate:** Transcended, decent ichors, some gem investment
- **High-End:** +11-16 enhance, perfect ichors, max gems, Goddess card sets

Required per tier: ATK, SP Pool, SP Regen/tick, CDR%
Required per enemy: DEF, HP, Level

### Step 2: Load Class Data

```python
# From repo: class skills with SFR arrays, CDs, SP costs, hit counts
# Access: https://raw.githubusercontent.com/marbleperfume/destructo/main/tos/classes/{tree}/{class}.json
class_data = url_fetch(f"https://raw.githubusercontent.com/marbleperfume/destructo/main/tos/classes/{tree}/{class_name}.json")

# From repo: Vaivora effects
vaivora_data = url_fetch(f"https://raw.githubusercontent.com/marbleperfume/destructo/main/tos/vaivora/vaivora_effects_{tree}.json")
```

### Step 3: Build Full 3-Class Rotation

Model as a **full build** (not single-class isolation):
- Primary class: full skill set from JSON
- Sub-class 1: 3-4 skills with typical SP costs and CDs for that tree
- Sub-class 2: 3-4 skills
- Fillers: Evil Sacrifice (37 SP, 0 CD), basic attacks, stance toggles

If sub-class data isn't fully available, use placeholder skills:
- 2x skills at 25s CD, 190-200 SP
- 2x skills at 15-20s CD, 155-180 SP
- 1x skill at 10-12s CD, 120-130 SP

### Step 4: Run Simulation (5 Minutes)

```python
FIGHT_DURATION = 300  # 5 min for SP economy visibility
SP_TICK_INTERVAL = 10  # regen every 10s

# Per tier, simulate:
# - SP tracking (tick-based regen, drain from all casts)
# - Skill priority casting (highest SFR/CD first when off CD and affordable)
# - SP exhaustion events (skill ready but can't cast)
# - Spirit/autonomous generation and damage timeline
# - Vaivora proc accumulation and triggers
# - Log all damage events with timestamps for scatter plot
```

### Step 5: Calculate Outputs

Per tier:
- **DPS** (total damage / fight duration)
- **Kills/min** (total damage / enemy HP per mob)
- **Dead time %** (player CD gaps / fight duration)
- **Autonomous fill %** (spirit/pet/DoT damage time / total time)
- **True idle %** (dead time - autonomous fill)
- **SP deficit** (SP/s cost - SP/s regen)
- **Casts blocked** (SP exhaustion events count)
- **Potions needed** (deficit / potion_value per 5 min)
- **Damage breakdown** by skill category (% of total)
- **Vaivora contribution** if applicable

### Step 6: Visualize (Highcharts + HTML)

Generate HTML artifact with:
1. **3-tier cards** -- Budget / Intermediate / High-End key metrics side by side
2. **Scatter plot** -- damage events over time, color by skill, dead time regions shaded
3. **SP curve** -- SP remaining over 300s, with/without potions, exhaustion zone marked
4. **Finding callout** -- state what the model reveals (no editorializing, just numbers)

Use local Highcharts: `/vendor/highcharts/highcharts.js`

### Step 7: Compare Against Expectations

The model demonstrates compounding bottlenecks at budget tier:
- ATK-DEF wall (most of raw stat absorbed by flat defense)
- SP drain (full rotation > regen without potions)
- CDR starvation (more dead time, slower procs, worse Vaivora cycling)

State these as quantified facts. Let the numbers prove the point.


## What This Skill Does NOT Do

- Does NOT find optimal rotations (only tests execution of given rotation)
- Does NOT model PvP
- Does NOT model party buff stacking (solo sustained only)
- Does NOT account for boss invulnerability phases
- Does NOT prescribe gear choices (shows consequences of each tier)
- Does NOT editorialize about game design (quantify, don't opine)


## File Locations

```
marbleperfume/destructo (public repo)
|- tos/
|   |- classes/{tree}/{class}.json  -- Skill data (SFR arrays, CDs, SP, hits)
|   |- vaivora/
|   |   |- vaivora_effects_swordsman.json (19/19 complete)
|   |   |- vaivora_effects_wizard.json (16/18)
|   |- tos_vaivora_database.json  -- Class index (85 entries, no effects)
|   |- tos_gear_system_context.md -- Gear to ATK calculation guide
|   |- tos_rebuild_combat_system.md
|   |- vaivora/
|   |   |- vaivora_effects_swordsman.json (19/19)
|   |   |- vaivora_effects_wizard.json (17/17, Alchemist excluded - shop class)
|   |   |- vaivora_effects_archer.json (15/15)
|   |   |- vaivora_effects_cleric.json (17/17, Pardoner excluded - shop class)
|   |   |- vaivora_effects_scout.json (16/16, Squire excluded - shop class)
|   |   |- vaivora_effects_common.json (5 non-class-restricted)
|   |   |- screenshots/ (raw Notion source images)
|   |- (shop classes excluded from combat modeling: Alchemist, Pardoner, Squire)
```

Access raw files: `https://raw.githubusercontent.com/marbleperfume/destructo/main/tos/...`


## Validation Checklist

Before presenting results, verify:
- [ ] Web search performed for stat context at target level
- [ ] ATK-DEF produces reasonable effective ATK (not 1, not negative)
- [ ] SP regen is tick-based (every 10s, not continuous)
- [ ] Full 3-class build modeled (not single-class isolation)
- [ ] Spirit/autonomous damage included for pet/summon classes
- [ ] Buff uptimes calculated (Ghastly Trail, stances, etc.)
- [ ] Kills/min against level-appropriate enemy with correct DEF/HP
- [ ] 3 gear tiers shown (Budget, Intermediate, High-End)
- [ ] Vaivora on/off comparison if testing Vaivora impact
- [ ] No editorializing -- numbers speak for themselves
