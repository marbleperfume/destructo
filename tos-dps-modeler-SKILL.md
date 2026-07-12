---
name: tos-dps-modeler
display_name: ToS DPS Modeler
description: "Analyze Tree of Savior player-to-enemy and enemy-to-player numeric relationships. Tests whether a given ATK budget and class build can clear content thresholds, measures dead time as a function of CD structure, and identifies where Vaivora procs fill (or fail to fill) rotation gaps. Activate for ToS DPS checks, dead time analysis, stat threshold calculations, TTK modeling, Vaivora effectiveness comparisons, or farming efficiency."
icon: "🗡️"
trigger: ToS Tree of Savior DPS model graph scatter dead time rotation combat vaivora SP economy TTK kill time stat cap skill cycling burst farming efficiency threshold
patterns:
  - {pattern: "\\b(?:tos|tree of savior)\\b.*(?:graph|chart|dps|rotation|damage|dead time|threshold|farm)", confidence: 0.95}
  - {pattern: "\\b(?:scout|assassin|swordsman|wizard|archer|cleric)\\b.*(?:dps|rotation|model|graph|farm|dead time)", confidence: 0.80}
  - {pattern: "\\bvaivora\\b.*(?:tier|compare|damage|proc|fill|gap)", confidence: 0.90}
  - {pattern: "\\bdead time\\b.*(?:graph|visual|model|farm)", confidence: 0.85}
inputs:
  - name: class_name
    description: "Primary class build to model (e.g., 'Assassin-Outlaw-Corsair' or single class name)"
    type: string
    required: true
  - name: comparison_class
    description: "Optional second class build for head-to-head comparison"
    type: string
    required: false
  - name: total_atk
    description: "Total ATK stat (collapses gear, Goddess sets, accessories, seals, gems into one number)"
    type: number
    default: 50000
  - name: vaivora_tier
    description: "Vaivora weapon tier (0=none, 1-4) — modifies skill factors"
    type: number
    default: 0
  - name: fight_duration
    description: "Simulation window in seconds (60 for burst, 300+ for sustained farming)"
    type: number
    default: 60
  - name: content_type
    description: "Target scenario: 'single_target', 'field_farming', 'cm', 'saalus', 'boss'"
    type: string
    default: single_target
  - name: output_format
    description: "Visualization type: 'scatter', 'deadtime', 'threshold', 'ttk', 'all'"
    type: string
    default: all
tools: [url_fetch, run_python, file_read, file_write, open_in_session_tab]
depends-on: [highcharts, html_design]
---

## Overview

Tests Tree of Savior combat viability from a numeric standpoint: can a given ATK budget and class build meet content thresholds? Measures dead time as a structural property of CD layout, calculates player-to-enemy damage (TTK) and enemy-to-player threat, and identifies where Vaivora weapon effects fill rotation gaps versus where they're band-aid solutions for fundamentally broken CD structures.

The skill does NOT find optimal builds. It quantifies:
1. **Threshold viability** — given stat investment, does DPS clear the content check?
2. **Dead time as design flaw** — where CD structure creates unfillable gaps (Assassin excels single-target but farms terribly because of this)
3. **Vaivora as balance lever** — some procs linearize rotation proficiency (good design), others paper over extraordinary dead time that shouldn't exist (bad design)
4. **Stat-to-cap breakpoints** — where diminishing returns hit for crit, accuracy, evasion, block penetration against specific monsters

## Data Source

All class data lives at: `https://raw.githubusercontent.com/marbleperfume/destructo/main/tos/`

**The repo is public** — no authentication needed. Use `url_fetch` directly on raw.githubusercontent.com URLs.

- **Class JSONs**: `classes/{tree}/{class}.json` — skill names, skill factors (SFR%), cooldowns, SP costs, hit counts, overheat charges, AoE coefficients
- **Vaivora database**: `tos_vaivora_database.json` — weapon-to-class mapping, effect descriptions, role classifications
- **Gear system context**: `tos_gear_system_context.md` — ATK calculation, stat ranges by gear tier, why gear is collapsed to one number

Tree folders: `swordsman/`, `wizard/`, `archer/`, `scout/`, `cleric/`

## Critical Domain Rules

### ATK as Collapsed Input
ToS gear is too combinatorial to model piece-by-piece (Goddess sets, Ark effects, accessories, seals, gems, enhancement levels, transcendence). Total ATK collapses all of this into one number. The model accepts ATK directly and applies the damage formula without caring HOW that ATK was achieved. Realistic ranges:
- Fresh Lv460: 30k-50k
- Mid endgame: 80k-150k
- High investment: 200k-350k
- Whale: 400k-600k+

### Dead Time Is a Design Property
Dead time is NOT player error — it's a structural property of a build's CD layout. When all skills are on cooldown and the player can only auto-attack, that gap exists because the class designers didn't provide enough skills to fill the rotation. This is the #1 differentiator between:
- **Good farming builds**: near-zero dead time (enough low-CD skills to chain continuously)
- **Bad farming builds**: high dead time despite high burst DPS (Assassin — huge single-target damage, then nothing for 20s)

The model measures dead time objectively. It doesn't judge builds — it shows the gap.

### Vaivora as Balance Lever
Vaivora weapons modify skill behavior in two distinct patterns:

**Linear proficiency (good design):** The proc adds damage proportional to rotation frequency. More skill use = more Vaivora value. The rotation doesn't change shape — it just does more damage. These builds have smooth DPS curves.

**Gap-filling (bad design):** The proc fires during dead time, partially covering a CD gap that shouldn't exist. The class can't function without the Vaivora because its base rotation has too much dead time. These builds have jagged DPS curves where the Vaivora proc is load-bearing rather than supplementary.

**Incomplete data problem:** Vaivora effect descriptions tell you the trigger mechanic but often not the proc's actual damage value. Those values live in patch notes (most recent mention of that specific Vaivora). The model flags unknown proc values rather than guessing. To resolve: search specifically for the unknown value, verify it's PvE-applicable (not PvP-only), apply.

### Damage Formula
```
Damage = (ATK × Skill_Factor% × Hit_Count) × (1 + Vaivora_Bonus%) × Enhance_Mult
         ÷ DEF_Factor
```
Where:
- `DEF_Factor = (ATK + Enemy_DEF) / ATK` (capped at ATK → 50% minimum damage)
- `Enhance_Mult` = gear enhancement level multiplier (default 1.5 for +16 endgame)
- `Vaivora_Bonus%` = tier-specific percentage from database (0 if tier=0)

### SP Economy
Every skill costs SP. The rotation degrades when SP depletes:
- Passive regen: ~100/sec
- SP potions: one-time restore (shared CD)
- Class-specific recovery (Cleric Indulgentia, etc.)
- At SP=0: auto-attack only (effective dead time from resource exhaustion, not CD structure)

Model must distinguish dead time from **CD gaps** (structural) vs **SP exhaustion** (investment problem).

### Stat-to-Cap Breakpoints (Re:Build formula, n=951)
Player stats have hard breakpoints against specific monsters:
- **Accuracy**: must exceed monster EVA to avoid misses (acc_to_cap = monster_eva + 1)
- **Crit Rate**: must exceed monster CDEF + 12750 for 60% crit chance
- **Block Pen**: must match or exceed monster BLK to avoid 25% damage penalty
- **Evasion**: must exceed monster ACC + 15092 for 50% dodge chance

These are HARD thresholds — being 1 point below produces a large penalty. The model must check these before calculating DPS (missing the ACC cap means your SFR/s is reduced by miss rate).

### Content-Type Scaling
Different content types expose different weaknesses:
- **Single target (boss)**: burst DPS matters most, dead time between burst cycles is acceptable
- **Field farming**: sustained DPS with minimal dead time matters (kill speed × density)
- **CM/wave defense**: AoE coverage + sustained rotation without SP exhaustion
- **Saalus Convent**: timed DPS check with waves — both burst and sustain needed

## Workflow

### Step 1: Fetch Class Data
- **Mode**: `deterministic`
- **Tool**: `url_fetch`
- **Input**: `{{class_name}}` → resolve to tree folder(s) and filename(s)
- **Output**: Parsed class JSON(s) + vaivora DB + gear context
- **Validate**: JSONs parse, skill arrays non-empty
- **On failure**: Check repo tree for filename capitalization differences

### Step 2: Check Stat Thresholds
- **Mode**: `deterministic`
- **Tool**: `run_python` (use `compute_tos_stat_to_cap`)
- **Input**: `{{total_atk}}` + target monster stats (from content_type default profiles)
- **Output**: Which caps are met, which are missed, penalty % from misses
- **Validate**: All cap values are positive integers
- **On failure**: Check monster stat source data

### Step 3: Build Rotation Model
- **Mode**: `agentic`
- **Tool**: `run_python` (use `compute_tos_rotation`)
- **Input**: Class JSONs + `{{vaivora_tier}}` + `{{fight_duration}}` + CDR assumptions
- **Output**: Timeline of damage events + dead time windows + SP curve
- **Validate**: Timeline spans fight duration, SP never negative, dead time windows don't overlap casts
- **On failure**: Check overheat logic, CDR application

The model must:
1. Sort skills by SFR-per-cooldown-second (damage priority)
2. Cast highest-priority available skill at each decision point
3. Track SP drain; mark SP-exhaustion dead time separately from CD dead time
4. Track overheat charges (burst N times → long lockout)
5. Apply CDR (default 30% endgame assumption — ask user if uncertain)
6. Record dead time windows with cause tag (CD_GAP vs SP_EXHAUSTION)
7. If `{{vaivora_tier}}` > 0: apply Vaivora bonus to affected skills; track when Vaivora proc fires during dead time vs during active rotation

### Step 4: Calculate Metrics
- **Mode**: `deterministic`
- **Tool**: `run_python` (use `compute_tos_ttk`)
- **Input**: Rotation SFR/s + `{{total_atk}}` + target stats
- **Output**: TTK, DPS, dead time %, SP efficiency, burst vs sustain comparison
- **Validate**: DPS > 0, TTK finite
- **On failure**: If DPS = 0, stat threshold was missed (Step 2 should have caught this)

### Step 5: Generate Visualization
- **Mode**: `agentic`
- **Tool**: `run_python` + `file_write`
- **Input**: All metrics + `{{output_format}}`
- **Output**: HTML with Highcharts chart(s)

Visualization types:
- **scatter**: X=time, Y=damage per hit. Dead time regions as red shaded plotBands. Markers sized by hit count. Vaivora proc hits in a distinct color to show where they land (during active rotation vs during dead time).
- **deadtime**: Gantt-style xrange showing skill casts as bars, gaps as empty space. Tags each gap with cause (CD vs SP). Shows rotation density at a glance.
- **threshold**: Bar chart showing ATK required to clear specific content at various TTK targets. Horizontal line at player's current ATK. Shows "you are here" vs "you need to be here."
- **ttk**: Comparative horizontal bars for multiple builds against same target. Shows which builds clear and which don't at the given ATK budget.
- **all**: Dashboard grid with all charts.

### Step 6: Present Output
- **Mode**: `deterministic`
- **Tool**: `open_in_session_tab`
- **Input**: Generated HTML path
- **Output**: Interactive chart

## Output

Highcharts visualization exposing:
- Dead time structure (where gaps exist, why, how Vaivora procs interact with them)
- Threshold viability (does this build clear at this ATK?)
- Content-type suitability (boss-killer vs farmer vs wave-defender)
- Comparison between builds showing dead time %, burst DPS, sustained DPS, SP runway

## Lessons Learned

### Do
- Always check stat caps BEFORE calculating DPS — missing ACC cap invalidates the entire rotation model
- Distinguish CD dead time (design flaw) from SP dead time (investment problem) — they have different solutions
- Track Vaivora proc timing relative to dead time — this reveals whether the proc is linear enhancement or gap-filling
- Use `content_type` to select appropriate target profiles (field mobs vs bosses have wildly different DEF/HP)
- Apply CDR before rotation timing — endgame assumes 30% baseline

### Don't
- Don't model ToS like FFXIV (no fixed GCD, no 2-minute cadence — purely CD-driven with animation locks)
- Don't assume Vaivora is always beneficial — if the proc fires during active rotation it might be irrelevant (already casting something)
- Don't ignore SP costs — many builds OOM in 30s of burst, making sustained scenarios reveal their weakness
- Don't treat dead time as player error — it's structural to the build's CD layout

### Common Failures
- **SP underflow**: Forgot to check SP before casting. Always validate sp_remaining >= cost.
- **Overheat miscalculation**: Overheat is burst → lockout, not N separate CDs.
- **CDR compounding**: `effective_cd = base × (1 - cdr)` — multiplicative, not additive.
- **Missing stat cap**: Player ACC below monster EVA = significant miss rate, ruins the DPS calculation silently.
- **Vaivora unknown value**: Proc damage not in database. Flag it, don't guess. Search patch notes for the specific skill factor.

### When to Ask the User
- Total ATK budget (varies enormously by investment level)
- Which 3 classes compose the build (ToS uses 3-class system)
- Target content type (determines monster stat profile and success criteria)
- CDR % if they have a specific gear setup
- Whether Vaivora is equipped and at what tier (0-4)
- For Vaivora with unknown proc values: whether to skip it or search for the value
