---
name: ffxiv-rotation-grapher
display_name: FFXIV Rotation Grapher
description: "Model FFXIV job rotations and produce DPS visualizations (scatter plots, box-and-whisker, cumulative potency timelines, FFLogs-style DPS-over-time curves). Uses web_search to contextualize stat expectations at target ilvl before modeling. Activate when user asks to graph, visualize, model, simulate, or compare FFXIV job DPS, rotations, potency timelines, or damage distributions. Also activates for questions about GCD clipping losses, DoT uptime, burst windows, level-sync scaling, or substat tier comparisons."
icon: "⚔️"
trigger: FFXIV graph scatter DPS rotation model job comparison potency timeline visualization box whisker burst window GCD clip DoT uptime level sync stat tier BiS fresh substat
patterns:
  - {pattern: "\\bffxiv\\b.*(?:graph|chart|plot|scatter|visual|model|sim)", confidence: 0.95}
  - {pattern: "\\b(?:brd|dnc|nin|war|drg|rpr|smn|blm|whm|sch|sge|ast|pld|drk|gnb|mnk|sam|mch|rdm|vpr|pct)\\b.*(?:dps|rotation|damage)", confidence: 0.85}
  - {pattern: "\\bpotency\\b.*(?:timeline|graph|scatter)", confidence: 0.90}
  - {pattern: "\\b(?:bis|fresh|midrange|substat)\\b.*(?:dps|compare|tier|gap)", confidence: 0.85}
inputs:
  - name: job
    description: "Primary job to model (3-letter abbreviation: WAR, BRD, NIN, etc.) or 'all' for full roster"
    type: string
    required: true
  - name: comparison_job
    description: "Optional second job for head-to-head comparison"
    type: string
    required: false
  - name: fight_duration
    description: "Fight length in seconds for the simulation (120 = short, 300 = mid, 540+ = savage/ultimate)"
    type: number
    default: 120
  - name: level
    description: "Level sync target (affects trait potency scaling). 90 or 100."
    type: number
    default: 100
  - name: stat_tier
    description: "'fresh' (crafted/normal raid), 'midrange' (mix of savage/tome), 'bis' (full best-in-slot), or 'all' for 3-tier comparison"
    type: string
    default: bis
  - name: boss_invuln_windows
    description: "Array of [start_sec, end_sec] pairs for boss invulnerability (the ONLY source of DoT/damage loss)"
    type: string
    required: false
  - name: output_format
    description: "Visualization type: 'scatter', 'box', 'cumulative', 'fflogs_timeline', 'all'"
    type: string
    default: all
tools: [web_search, url_fetch, run_python, file_read, file_write, open_in_session_tab]
depends-on: [highcharts, html_design]
---

## Overview

Tests FFXIV rotation execution fidelity against the game's 2-minute cadence design. The skill calculates potency-per-second for a given rotation sequence, then measures how much is lost to encounter constraints (boss invulnerability, forced movement, burst window misalignment). It does NOT find optimal rotations — it quantifies the difference between rotation variants and measures distance from theoretical maximum.

Uses web_search to contextualize stat expectations at current ilvl BEFORE modeling, same pattern as the ToS DPS Modeler. This grounds the output in real BiS data rather than guessed stat totals.

Uses job data and trait potency scaling from the `marbleperfume/destructo` public GitHub repo. Output validates whether a job's numbers meet DPS check thresholds and exposes where small potency differences compound over 7+ minute encounters.

## Data Source

All job data lives at: `https://raw.githubusercontent.com/marbleperfume/destructo/main/ffxiv/`

**The repo is public** — no authentication needed. Use `url_fetch` directly on raw.githubusercontent.com URLs. No browser navigation required for data access.

- **Job JSONs**: `classes/{role}/{job}.json` — contains skill names, potency values, cooldowns, GCD/oGCD classification, combo chains, proc rates
- **Trait scaling**: `ffxiv_trait_potency_scaling.json` — contains pre_94/post_94 potency values, mastery chains, enhanced traits, conditional traits, classifications

Role folders: `tank/`, `healer/`, `melee/`, `phys_ranged/`, `magic_ranged/`


## Stat Formulas (Lv100)

Constants: SUB = 420, DIV = 2780

```python
import math

def crit_rate(crit_stat):
    """Returns crit chance as decimal (e.g., 0.25 = 25%)"""
    return (math.floor(200 * (crit_stat - 420) / 2780) + 50) / 1000

def crit_multiplier(crit_stat):
    """Returns crit damage multiplier (e.g., 1.60 = +60% on crit)"""
    return (math.floor(200 * (crit_stat - 420) / 2780) + 1400) / 1000

def det_multiplier(det_stat):
    """Returns det multiplier (e.g., 1.043 = +4.3% flat)"""
    return math.floor(140 * (det_stat - 420) / 2780 + 1000) / 1000

def dh_rate(dh_stat):
    """Returns direct hit chance as decimal (e.g., 0.32 = 32%)"""
    return math.floor(550 * (dh_stat - 420) / 2780) / 1000

def effective_gcd(base_gcd_ms, speed_stat):
    """Returns GCD in seconds after speed stat applied"""
    return math.floor(base_gcd_ms * (1000 + math.floor(130 * (420 - speed_stat) / 2780)) / 10000) / 100

def expected_damage_multiplier(crit_stat, det_stat, dh_stat):
    """Combined expected multiplier from all substats"""
    cr = crit_rate(crit_stat)
    cm = crit_multiplier(crit_stat)
    det = det_multiplier(det_stat)
    dhr = dh_rate(dh_stat)
    # Expected value: (1-cr)*(1-dhr)*1 + cr*(1-dhr)*cm + (1-cr)*dhr*1.25 + cr*dhr*cm*1.25
    # Simplified: det * (1 + cr*(cm-1)) * (1 + dhr*0.25)
    return det * (1 + cr * (cm - 1)) * (1 + dhr * 0.25)
```

DH always deals +25% damage (fixed). Crit + DH = crit multiplier × 1.25.


## 3-Tier Stat Framework

### Why This Matters

FFXIV substats create a ~21% DPS gap between fresh Lv100 and full BiS from substats alone (before main stat / weapon damage / ilvl differences, which add another ~30-40%). The model must contextualize which stat tier it's simulating — a rotation that "works" at BiS (enough crit to proc reliably, enough speed for alignment) may not function identically at fresh stats.

### Tier Definitions

| Tier | Source | ilvl | Typical Crit | Det | DH | Speed |
|------|--------|------|-------------|-----|-----|-------|
| Fresh | Crafted + normal raid | i710-720 | ~1800 | ~1600 | ~1200 | Varies |
| Midrange | Mix savage + tome | i730-740 | ~2800 | ~2000 | ~1800 | Varies |
| BiS | Full savage + upgraded tome | i750+ | ~3800-3900 | ~2300-2500 | ~2100-2300 | Job-specific |

These are APPROXIMATE defaults. Step 0 (web search) overrides them with current patch-accurate values.

### Expected Multipliers by Tier

| Tier | Crit Rate | Crit Mult | Det Mult | DH Rate | Combined |
|------|-----------|-----------|----------|---------|----------|
| Fresh | ~14.9% | ~1.450 | ~1.060 | ~15.4% | ×1.18 |
| Midrange | ~22.1% | ~1.521 | ~1.079 | ~27.3% | ×1.30 |
| BiS | ~29.4% | ~1.594 | ~1.095 | ~37.2% | ×1.43 |

**Gap: BiS deals ~21% more damage than Fresh from substats alone.**


## Critical Domain Rules

### DoT Damage
DoTs tick every 3 seconds and are ALWAYS active once applied. They are regular damage events contributing to total DPS — not passive background noise. The **only** time DoT damage is lost is during **boss invulnerability windows** (phase transitions, cutscenes, untargetable phases). Never model DoT ticks as optional, secondary, or excludable from analysis.

### DoT Tick Timing and Clipping
FFXIV DoTs tick on a fixed 3-second server tick aligned to their application time. There is NO pandemic extension — reapplying a DoT overwrites the old one entirely. This means early reapplication loses remaining ticks:

- **Tick loss calculation**: If a DoT has T seconds remaining and you recast, you lose `floor(T / 3)` ticks that would have fired before expiration.
- **Optimal refresh window**: Recast when 0–2.9s remain (the last tick has already fired, next tick would never happen).
- **Cost of clipping**: Each lost tick = one instance of the DoT's tick potency wasted. For BRD (two DoTs at ~20 potency/tick), clipping both 6s early = 4 lost ticks = 80 potency thrown away.
- **Cost of letting it drop**: If DoT expires and you wait N seconds before reapplying, you lose `floor(N / 3)` ticks of zero damage during the gap.

For BLM Thunder specifically: using a Thunderhead proc while the DoT has significant duration remaining is a DPS loss not only from tick clipping but also from displacing a Fire IV (2.8s cast, boosted by AF3 1.8x multiplier) with a Thunder GCD that only contributes the application hit plus a fresh DoT that was already running.

The simulation must track each DoT's remaining duration and only reapply within the optimal refresh window. Early reapplication should be flagged as potency loss in the output.

### GCD and Haste Modifiers
The base GCD is 2.50s, but multiple jobs have built-in haste that changes their effective GCD. Apply these BEFORE Skill Speed substats:

| Job | Haste Source | Effective Base GCD | Notes |
|-----|-------------|-------------------|-------|
| MNK | Greased Lightning (permanent) | 1.94s | Always active at Lv20+. Fastest GCD in game. |
| NIN | Increase Attack Speed trait (Lv45) | 2.12s | 15% haste passive. Always active. Affects GCD + auto-attacks. |
| SAM | Shifu buff (combo action) | 2.18s | 13% haste from Shifu combo. Maintained 100% uptime in rotation. |
| All others | None | 2.50s | Standard GCD (but see BLM exception below). |

After haste, Skill Speed / Spell Speed substats further reduce GCD. The formula:

```
effective_gcd = floor(base_gcd_ms * (1000 - floor(130 * (speed_stat - 420) / 2780)) / 1000) / 1000
```

Where `base_gcd_ms` = the haste-modified GCD in milliseconds (e.g., 2500 for standard, 2120 for NIN).

**Typical endgame GCD values (BiS, Lv100):**
- MNK: ~1.94s (no SkS needed, haste does the work)
- NIN: ~2.10-2.12s (minimal SkS, haste trait carries)
- SAM: ~2.14-2.17s (some SkS for alignment)
- DRG: ~2.50s (no haste, no SkS priority)
- BRD/MCH/DNC: ~2.50s (phys ranged don't stack SkS)
- Tanks: ~2.50s (some WAR players run 2.45s for Inner Release alignment)

This matters because faster GCD = more GCDs per fight = more total potency delivered. A NIN at 2.12s gets ~18% more GCDs than a DRG at 2.50s over the same fight duration.

### BLM: Variable Cast Times (Special Case)
BLM **cannot** be modeled with a flat GCD rate. It has per-spell cast times — many LONGER than the GCD recast. When cast time > recast time, the cast time becomes the effective gap between actions.

| Spell | Cast Time | Effective Gap | Context |
|-------|-----------|---------------|---------|
| Fire IV | 2.8s | 2.8s | Main damage spell, cast 6x per AF cycle |
| Despair | 3.0s | 3.0s | AF finisher, high potency |
| Flare Star | 3.0s | 3.0s | Requires 6 Astral Soul stacks |
| Fire III (raw) | 3.5s | 3.5s | Rarely used raw; usually halved or proc'd |
| Fire III (from UI3) | 1.75s | 2.5s (recast) | Halved by Umbral Ice III |
| Fire III (Firestarter) | instant | 2.5s (recast) | Proc from Paradox in AF |
| Blizzard III (from AF3) | 1.75s | 2.5s (recast) | Halved by Astral Fire III |
| Blizzard IV | 2.5s | 2.5s | Normal cast in UI |
| Paradox (in AF) | instant | 2.5s (recast) | Generates Firestarter proc |
| Paradox (in UI) | 2.5s | 2.5s | Normal cast |
| High Thunder | instant | 2.5s (recast) | Under Thunderhead buff |
| Xenoglossy | instant | 2.5s (recast) | Polyglot resource, highest single-hit potency |

**Astral Fire III multiplier:** Fire spells deal 1.8x damage in AF3. This is why Fire IV is cast despite its long cast time — the effective potency is potency × 1.8.

**Ley Lines:** 15% haste to both cast and recast for 30s (90s CD). Fire IV: 2.8s → ~2.38s. Despair: 3.0s → ~2.55s.

**Enochian / buff state maintenance:**
BLM has a persistent buff state (Enochian) that must be maintained or the rotation collapses:
- **Enochian** — passive damage bonus active while in AF or UI. Dropping it (letting both AF and UI expire without casting) loses Polyglot generation (30s timer for Xenoglossy charges) and forces a full restart.
- **Umbral Soul** — free cast usable out of combat and during boss invulnerability. Maintains Enochian + grants Umbral Hearts during downtime. This is how BLM survives phase transitions without losing state.
- **Manafont** — restores full MP, used for burst rotation extensions (extra AF cycle without entering UI). NOT a recovery tool for dropped Enochian.
- **Interruption cost** — if a boss mechanic forces BLM to stop casting long enough for AF/UI to expire (~15s), Enochian drops and BLM must rebuild from scratch. This is a unique DPS penalty no other job faces during forced movement/downtime.

For modeling: during `boss_invuln_windows`, BLM uses Umbral Soul to maintain state (zero DPS but no restart penalty). If a movement phase is short enough to keep casting (using instants: Xenoglossy, Thunder, Firestarter), no state loss. If forced off for >15s without Umbral Soul access, model an Enochian restart penalty (~5-8s of zero-potency rebuilding).

**Rotation variants exist** (standard 6x Fire IV vs Paradox-heavy shorter cycles). The skill calculates PPS for whatever rotation sequence is input — it does not resolve which is optimal. The difference between variants is quantifiable: fewer long casts = more total actions but lower per-action potency under AF3.

**Substat interaction:** BLM's optimal rotation changes based on Crit vs SpS allocation. The model should accept substat weights as parameters or run both builds for comparison. Thunder procs (Thunderhead) are instant GCDs that displace Fire IV under 1.8x — their value depends on DoT remaining duration and what they replace.

### GCD Timeline
FFXIV has no "dead time" for DPS jobs. The GCD rolls continuously at the job's effective rate (or per-spell for BLM). Between GCDs, 1-2 oGCDs can be weaved during instant-cast GCDs. The timeline is always full — the scatter plot shows **amplitude variance** (which hits are big vs small), not activity gaps.

### Gear Does Not Change Rotation
FFXIV gear is stat sticks with materia. There are no proc-based gear effects, no set bonuses that alter rotation, no weapon effects that add skills. Gear only changes the damage multiplier applied to potency — it never changes WHICH skills you press or WHEN. This means potency-per-second modeling is sufficient for relative job rankings without needing to model gear.

**However**: gear DOES change the effective multiplier via substats. The 3-tier framework quantifies this. A BiS player with 29% crit at ×1.59 multiplier vs a fresh player at 15% crit and ×1.45 multiplier — same rotation, same potency timeline, ~21% different actual DPS from substats alone.

### Level Sync Scaling
When `level` < 100, apply pre-94 potency values from the trait scaling JSON. Multi-tier mastery chains may have breakpoints at Lv84 and Lv94. Check the `level_breakpoint` field.

### Proc-Based Jobs
BRD, DNC, and some others have RNG-driven procs. The simulation must model these stochastically — run multiple iterations (default: 1000) and report the distribution, not a single deterministic rotation.

### Burst Window Grant Chains (120s Cycle)
FFXIV is balanced around a **2-minute burst cycle**. Most offensive cooldowns are 120s, and the game expects all 8 party members to align their bursts simultaneously. The model must understand this structure:

**Grant mechanic:** Executing a 120s cooldown GRANTS a time-limited "Ready" buff (~30s) that unlocks one or more high-potency follow-up skills. These follow-ups don't exist outside the grant window — if unused before expiry, the potency is lost entirely.

Examples:
- SAM: Ikishoten (120s) → grants Kenki +50 + Ogi Namikiri Ready (30s) + Zanshin Ready (30s)
- NIN: Dokumori (120s) → grants Higi (30s, enables Zesho Meppo); Ten Chi Jin (120s) → grants Tenri Jindo Ready (30s)
- PCT: Starry Muse (120s) → party buff + grants Starstruck (enables Star Prism), Subtractive Palette, Hyperphantasia chain

**Window pressure:** During the burst window, a job must execute 5-8 high-potency actions within ~20-30s. Missing any granted action = permanent potency loss (cannot be recovered).

**60s sub-cycle jobs:** SAM (Meikyo Shisui 55s) and NIN (Dokumori 60s) have burst tools on shorter CDs. These fire at BOTH 0s and 60s marks — but the 60s usage is a smaller burst (no party buff backing). The 120s usage aligns with full raid buff stack. The 55s Meikyo timing likely prevents drift from pushing the second use past the 120s window.

**Hold-for-buffs logic:** If a skill comes off cooldown 2-3s before raid buffs go up, the correct play is to HOLD and fire inside the buff window. "Use on cooldown" is NOT universally optimal — alignment to the 2-minute cycle takes priority. The model should track buff windows and delay abilities by up to ~3s to align.

### Zero-Damage Buff Skills (rDPS Contributors)
Some skills deal zero personal damage but provide party-wide damage buffs. These define WHEN the burst window opens:

| Job | Skill | CD | Effect | rDPS Impact |
|-----|-------|-----|--------|-------------|
| PCT | Starry Muse | 120s | Party damage +5% for 20s | ~3,500 rDPS |
| DNC | Technical Finish | 120s | Party damage +5% for 20s | Similar |
| NIN | Dokumori | 120s | Target vuln +5% for 15s | Similar |
| BRD | Radiant Finale | 120s | Party damage +2-6% for 20s | Varies by Coda |
| DRG | Battle Litany | 120s | Party crit rate +10% for 20s | Crit-dependent |
| MNK | Brotherhood | 120s | Party damage +5% for 20s | Similar |
| RPR | Arcane Circle | 120s | Party damage +3% for 20s | Lower |

These skills have DPS = 0 but rDPS > 0. The model must classify them as buff skills that:
1. Don't appear in personal potency scatter plots
2. Define the burst window timing for the party
3. Contribute to rDPS calculation (damage dealt by others during your buff, attributed back to you)

### DPS Metrics (DPS vs rDPS vs aDPS)
The model must be explicit about which metric it's calculating:

- **DPS** (raw): personal damage / fight time. What the scatter plot and cumulative chart show.
- **rDPS** (raid DPS): your damage + damage your buffs gave to others - damage you received from others' buffs. Used for ranking buff jobs fairly. A DNC with low personal DPS but high rDPS is contributing through Technical Finish.
- **aDPS** (adjusted DPS): your personal damage adjusted as if you received no external buffs. Shows what you'd do in a vacuum. FFLogs "Rank by aDPS" uses this.

For the potency model: personal PPS ≈ DPS (before stat multiplication). To approximate rDPS, multiply buff uptime × party members' damage during window × buff percentage and add to personal.

### Guaranteed Crit / Direct Hit Mechanics
Not all crit/DHit is random. Some skills or buff states guarantee crit and/or direct hit:

- PCT Hammer combo (Hammer Stamp/Brush/Polishing Hammer): 100% Direct Hit guaranteed
- PCT Star Prism: 75% Crit rate (from trait/buff, not substat)
- WAR Inner Chaos / Primal Rend: guaranteed crit + DHit
- SAM Midare Setsugekka under Meikyo: guaranteed crit (from Kaiten-equivalent buff)
- Various skills under specific buff windows

The model must flag skills that have forced crit/dhit — these bypass the substat crit rate formula and always deal enhanced damage. When calculating expected potency, multiply by 1.25 (crit modifier baseline) or 1.25 × 1.25 (crit + DHit) for guaranteed hits rather than using probability.

### 2-Minute Synchronization and rDPS
All 8 party members align their 120s bursts. During the ~20s window:
- 2-3 party buffs active simultaneously (+5% each, multiplicative = ~10-15% total)
- Every job dumps their granted burst actions under these stacked buffs
- The multiplicative stacking means each additional buff makes everyone else's burst worth more

Jobs that drift their burst outside this window lose the multiplicative benefit. A SAM firing Ogi Namikiri 5s after buffs expire loses ~10-15% on that hit.

**For the model:** When comparing jobs, the "fair" comparison is aDPS (how much each job contributes personally). When evaluating a PARTY's output, rDPS captures buff contribution. The scatter/box-whisker output should default to personal PPS but note the rDPS adjustment for buff jobs.


## Workflow

### Step 0: Contextualize via Web Search (REQUIRED)

Before modeling ANY job, establish stat context for the current patch:

```python
# Pull current BiS stat totals for the job
web_search(query=f"FFXIV {JOB} BiS {PATCH} stats materia melds icy-veins OR thebalanceffxiv 2024 2025")

# For ilvl context and weapon damage
web_search(query=f"FFXIV {PATCH} best in slot item level savage tome gear")

# For specific speed tier if job uses speed alignment
web_search(query=f"FFXIV {JOB} GCD tier skill speed {PATCH}")
```

Then fetch the relevant icy-veins or thebalanceffxiv page with `url_fetch` to get exact stat totals.

**Extract and structure into 3 tiers:**
- **Fresh:** Crafted HQ + normal raid drops (i710-720 range). Minimal melds, random substats.
- **Midrange:** Mix of savage drops + augmented tome (i730-740). Partial melds, decent but not optimized.
- **BiS:** Full savage + augmented tome, all materia slots optimally melded to stat priority.

**Required per tier:** Crit, Det, DH, Speed → convert to multipliers using formulas above.

**Use discovered stats to derive:**
```python
# For each tier:
tier_multiplier = expected_damage_multiplier(crit, det, dh)
tier_gcd = effective_gcd(base_gcd_ms, speed)
tier_crit_rate = crit_rate(crit)
tier_dh_rate = dh_rate(dh)
```

**Fallback values** if web search fails (use as last resort only):

| Tier | Crit | Det | DH | Speed |
|------|------|-----|-----|-------|
| Fresh | 1800 | 1600 | 1200 | 420 |
| Midrange | 2800 | 2000 | 1800 | varies |
| BiS | 3900 | 2400 | 2200 | varies |

These are approximations — always prefer web-searched values.


### Step 1: Fetch Job Data
- **Mode**: `deterministic`
- **Tool**: `url_fetch`
- **Input**: `{{job}}` abbreviation → resolve to role folder and filename
- **Output**: Parsed job JSON with skills, potencies, cooldowns, procs
- **Validate**: JSON parses successfully and contains a `skills` array
- **On failure**: Try alternate URL path; if repo structure changed, check the repo tree via browser

Job-to-path mapping:
```
WAR/PLD/DRK/GNB → tank/
WHM/SCH/AST/SGE → healer/
MNK/DRG/NIN/SAM/RPR/VPR → melee/
BRD/MCH/DNC → phys_ranged/
BLM/SMN/RDM/PCT → magic_ranged/
```

Also fetch `ffxiv_trait_potency_scaling.json` for the level-adjusted potencies.

### Step 2: Build Rotation Model
- **Mode**: `agentic`
- **Tool**: `run_python`
- **Input**: Job JSON + trait scaling JSON + `{{fight_duration}}` + `{{level}}` + `{{boss_invuln_windows}}` + stat tier data from Step 0
- **Output**: Array of damage events: `[{time_sec, skill_name, potency, type, is_crit_window, guaranteed_crit, guaranteed_dh}]`
- **Validate**: Total events > 0, timeline spans full fight duration, no impossible overlaps
- **On failure**: Check for infinite loops in proc resolution; cap iterations

The rotation model must:
1. Track GCD recast at the job's effective rate (apply haste modifier from table above)
2. For BLM: track per-spell cast times independently — effective gap = max(cast_time, recast_time)
3. Track all oGCD cooldowns independently
4. Apply combo chains in sequence (potency only applies if combo condition met)
5. Resolve procs stochastically (e.g., BRD Refulgent Arrow at 35% per DoT tick)
6. Apply DoT ticks every 3s continuously — only suppress during boss_invuln_windows
7. Apply burst window buffs (e.g., DNC Technical Finish +5% for 20s) as potency multipliers
8. For BLM: model Ley Lines as a haste phase (15% reduction to both cast and recast for 30s every 90s)
9. For BLM: track Astral Fire/Umbral Ice state — fire spells get 1.8x damage multiplier in AF3
10. For comparison mode: run both `{{job}}` and `{{comparison_job}}` with identical fight parameters
11. **NEW:** For each damage event, resolve crit/DH stochastically using tier-appropriate rates (except guaranteed crit/DH skills which always proc)
12. **NEW:** Convert potency to damage units: `damage = potency * stat_multiplier * (crit_roll * dh_roll)` where crit_roll and dh_roll are per-hit random draws

### Step 3: Run Monte Carlo (if proc-based job)
- **Mode**: `deterministic`
- **Tool**: `run_python`
- **Input**: Rotation model from Step 2, iteration count (1000)
- **Output**: Distribution of total damage values + per-event timing arrays for all iterations
- **Validate**: Standard deviation is non-zero for proc jobs; deterministic jobs should have σ ≈ 0 for potency (but nonzero for damage due to crit/DH variance)
- **On failure**: Check RNG seeding; ensure proc rates match wiki values

**NEW: For stat_tier='all', run the entire sim 3 times** (one per tier) with different stat multipliers. Each tier produces its own distribution.

### Step 4: Generate Visualization
- **Mode**: `agentic`
- **Tool**: `run_python` + `file_write`
- **Input**: Simulation results + `{{output_format}}` + stat tier data
- **Output**: HTML file with Highcharts visualization(s)
- **Validate**: HTML file exists and contains valid Highcharts configuration
- **On failure**: Check that Highcharts is loaded from `/vendor/highcharts/`; verify container divs have fixed heights

Visualization types:

#### scatter
X = time (sec), Y = damage per hit. Color-code by skill type (GCD/oGCD/DoT). Mark burst windows with plotBands (light gold for 120s windows). Mark boss invuln with gray bands. For BLM, marker X-width should reflect actual cast time (wide markers for Fire IV, narrow for instants).

**Multi-tier mode:** If `stat_tier='all'`, overlay 3 scatter series (one per tier, different opacity or separate subplot rows).

#### box
Horizontal box-and-whisker ranked by DPS. Jobs as Y-axis categories, DPS as X-axis. Shows median, IQR, whiskers, outliers. Color-code by job role (blue=tank, green=healer, red=melee, orange=phys ranged, purple=magic ranged). If comparing tiers, show grouped boxes (3 per job, one per tier).

This is the FFLogs zone-statistics style — the signature chart for "where does this job rank."

#### cumulative
Line chart of cumulative potency over time. Shows burst ramp and sustain phases. DoT contribution is a stacked area beneath.

**Multi-tier mode:** Show 3 lines (one per tier) diverging as crit/DH variance accumulates.

#### fflogs_timeline (NEW)
**The FFLogs-style "DPS over time" chart for longer fights (300s+).**

This is a smoothed line chart where:
- X = fight time (seconds)
- Y = **running-average DPS** (total damage so far / elapsed time)
- Shows the characteristic FFLogs shape: high spike during opener (inflated DPS from burst cooldowns in first ~20s), rapid decay as the average normalizes, then subtle oscillation on 120s cycle as each burst window temporarily raises the average before it decays again.

Implementation:
```python
# For each point in time t (sample every 0.5s):
running_dps[t] = cumulative_damage_at_t / t

# First 2s: spike to very high DPS (only opener hits counted, low denominator)
# By 30s: decay from opener inflation
# At 120s: small bump as 2nd burst fires
# At 240s: smaller bump as 3rd burst fires
# At 360s+: nearly flat — this IS the sustained DPS
# At 540s+: convergence — variance between iterations narrows
```

**Visual features:**
- Main line: median DPS over time (from Monte Carlo iterations)
- Shaded band: p10–p90 range (shows variance narrowing over longer fights)
- Dotted horizontal: final median DPS (the number you'd report)
- For multi-tier: 3 lines + 3 bands, clearly labeled
- For multi-job: separate colors per job

**Why this matters for game design analysis:**
- Shows exactly how much of FFLogs' "top DPS" is inflated by short kill times (opener hasn't decayed)
- Demonstrates the 2-minute cycle's diminishing impact as fights lengthen
- Reveals which jobs have front-loaded vs sustained DPS profiles
- Quantifies the "kill time tax" — faster kills inflate DPS; slower kills converge to true sustained

**Chart labeling:**
- Title: "{JOB} DPS Over Time — {fight_duration}s ({stat_tier})"
- Y-axis: "DPS (running average)"
- X-axis: "Fight Duration (s)"
- Annotation at t=20s: "Opener inflation zone"
- Annotation at each 120s mark: "Burst window"

#### all
Dashboard with 4 charts in a 2×2 grid:
- Top-left: scatter (damage events)
- Top-right: fflogs_timeline (running DPS)
- Bottom-left: cumulative potency
- Bottom-right: box-and-whisker (job comparison or tier comparison)

### Step 5: Present Output
- **Mode**: `deterministic`
- **Tool**: `open_in_session_tab`
- **Input**: Path to generated HTML file
- **Output**: Interactive chart visible to user
- **Validate**: File opens without error
- **On failure**: Check file path; ensure HTML is well-formed


## Output

An interactive Highcharts HTML visualization showing:
- Scatter plot of damage events over fight duration (color = skill type, size = damage magnitude)
- Box-and-whisker of DPS distribution ranked by job or tier (horizontal, FFLogs-style)
- Cumulative potency/damage timeline with DoT contribution as stacked area
- **FFLogs-style running-DPS timeline** showing opener decay, 2-min cycle bumps, and convergence
- Boss invulnerability windows shown as gray shaded bands (the only dead zones)
- Summary stats: mean DPS, median, p10/p90, burst phase DPS vs sustain phase DPS, opener-inflation ratio
- **Stat tier comparison** when `stat_tier='all'`: same rotation, 3 different multiplier sets, showing the ~21% gap numerically

## Lessons Learned

### Do
- **ALWAYS web_search first** — pull current BiS stats from icy-veins/thebalanceffxiv before running any sim
- Always fetch BOTH the job JSON and trait scaling JSON — potency values in the job JSON may be base (pre-trait) values
- Use `level_breakpoint` field to determine which potency tier applies at `{{level}}`
- Model DoT ticks as first-class damage events in the scatter plot (same marker style as GCDs, just different color)
- For box-and-whisker comparisons, normalize to "DPS" (accounting for stat multipliers) so the output matches what players actually see
- Include the `classification` field from trait JSON to determine skill types for color-coding
- Target output that approximates FFLogs aDPS distributions for validation — if relative rankings diverge significantly from FFLogs, the model has an error
- Apply haste modifiers FIRST, then Skill Speed substats on top — they're multiplicative, not additive
- For BLM: model the full AF/UI cycle with per-spell cast times, not a flat GCD
- For the FFLogs timeline chart: sample running-average DPS every 0.5s for smooth curves
- Use the p10-p90 band to show how variance narrows as fight duration increases
- Label opener inflation clearly — it's a mathematical artifact of dividing by small t, not "better play"

### Don't
- Never exclude DoT ticks from DPS calculations or visualizations — they are continuous damage, only paused by boss invuln
- Never model "dead time" for FFXIV DPS jobs — the GCD is always rolling
- Don't assume all jobs are proc-based — fixed-rotation jobs (WAR, DRG, SAM) need only 1 iteration for potency (but still need Monte Carlo for crit/DH variance)
- Don't use hardcoded potency values — always pull from the JSON, which reflects the latest patch data
- Don't model gear effects — FFXIV gear has no procs or rotation-altering mechanics
- Don't use 2.50s GCD for all jobs — NIN/MNK/SAM have fundamentally different GCD speeds
- Don't use a flat GCD for BLM — it has per-spell cast times where Fire IV (2.8s), Despair (3.0s), and Flare Star (3.0s) are all longer than the GCD recast
- Don't use fallback stat values if web search returned real numbers — always prefer live data
- Don't skip the fflogs_timeline for fights > 180s — it's the most informative chart at longer durations
- Don't display "DPS" without stating which stat tier it uses — always label

### Common Failures
- **GitHub raw URL 404**: The repo structure may have changed. Fall back to checking the tree via `url_fetch` on the repo page.
- **Infinite loop in proc sim**: BRD's Apex Arrow gauge can overflow if you don't cap it at 100. Always clamp resource gauges.
- **Trait JSON mismatch**: Some skills have `conditional: true` traits (e.g., NIN Meisui). Check the `condition` field before applying flat potency deltas.
- **Level sync edge case**: Some traits don't exist below Lv84 at all. If `level < level_breakpoint`, use the `base` potency (not pre_94).
- **Haste omission**: Forgetting to apply NIN/MNK/SAM haste results in ~15-20% lower GCD count, making these jobs appear much weaker than they are.
- **BLM flat GCD error**: Modeling BLM with a uniform 2.5s tick vastly overestimates GCD count. The real cycle is ~32s for one full AF/UI rotation due to long cast times.
- **BLM AF damage multiplier**: Fire spells deal 1.8x damage in Astral Fire III. Forgetting this makes BLM appear to deal half its actual damage.
- **Opener inflation in short sims**: A 60s fight shows "higher DPS" than a 540s fight for the same job because opener cooldowns dominate the average. The fflogs_timeline chart makes this visible rather than hiding it.
- **Stat tier confusion**: Always label which tier is being shown. A box plot comparing jobs must use the SAME stat tier for all jobs (typically BiS) for a fair comparison.
- **Web search returning outdated patch data**: Include the patch number and year in queries. FFXIV patches every ~4 months. Data from 2+ patches ago may have different BiS.

### When to Ask the User
- Which boss fight to model (determines invulnerability windows)
- Whether to use "ping-adjusted" GCD (2.50s vs 2.48s for low-ping players)
- For DNC specifically: whether to model partner synergy contribution or solo only
- If comparing two jobs: whether to normalize for raid buff contribution or raw personal DPS
- For BLM: whether to model standard rotation (6x Fire IV) or Paradox-heavy rotation (more instants, shorter AF cycles, viable at high SpS)
- **NEW:** Which stat tier to model (if not specified, default to BiS but offer tier comparison)
- **NEW:** Whether to show FFLogs-style timeline for fights < 180s (it's less informative on short fights where there's only 1 burst window)


## Validation Checklist

Before presenting results, verify:
- [ ] Web search performed for stat context at current patch/ilvl
- [ ] Stat tier stated explicitly in output labels
- [ ] Stat multipliers derived from formulas (not guessed)
- [ ] GCD matches job's haste tier + speed stat
- [ ] DoT ticks modeled as continuous damage events
- [ ] Burst windows marked at 0s, 120s, 240s, 360s...
- [ ] Guaranteed crit/DH skills flagged and handled differently from random procs
- [ ] Monte Carlo ran (even for deterministic jobs, for crit/DH variance)
- [ ] FFLogs timeline shows opener decay for fights > 180s
- [ ] Box plot uses same stat tier for all compared jobs
- [ ] Summary stats include both burst-phase DPS and sustained DPS
- [ ] No "DPS" number presented without tier label
