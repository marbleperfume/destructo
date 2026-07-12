---
name: ffxiv-rotation-grapher
display_name: FFXIV Rotation Grapher
description: "Model FFXIV job rotations and produce DPS visualizations (scatter plots, box-and-whisker, cumulative potency timelines). Activate when user asks to graph, visualize, model, simulate, or compare FFXIV job DPS, rotations, potency timelines, or damage distributions. Also activates for questions about GCD clipping losses, DoT uptime, burst windows, or level-sync scaling."
icon: "⚔️"
trigger: FFXIV graph scatter DPS rotation model job comparison potency timeline visualization box whisker burst window GCD clip DoT uptime level sync
patterns:
  - {pattern: "\\bffxiv\\b.*(?:graph|chart|plot|scatter|visual)", confidence: 0.95}
  - {pattern: "\\b(?:brd|dnc|nin|war|drg|rpr|smn|blm|whm|sch|sge|ast|pld|drk|gnb|mnk|sam|mch|rdm|vpr|pct)\\b.*(?:dps|rotation|damage)", confidence: 0.85}
  - {pattern: "\\bpotency\\b.*(?:timeline|graph|scatter)", confidence: 0.90}
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
    description: "Fight length in seconds for the simulation"
    type: number
    default: 120
  - name: level
    description: "Level sync target (affects trait potency scaling). 90 or 100."
    type: number
    default: 100
  - name: boss_invuln_windows
    description: "Array of [start_sec, end_sec] pairs for boss invulnerability (the ONLY source of DoT/damage loss)"
    type: string
    required: false
  - name: output_format
    description: "Visualization type: 'scatter', 'box', 'cumulative', 'all'"
    type: string
    default: all
tools: [url_fetch, run_python, file_read, file_write, open_in_session_tab]
depends-on: [highcharts, html_design]
---

## Overview

Tests FFXIV rotation execution fidelity against the game's 2-minute cadence design. The skill calculates potency-per-second for a given rotation sequence, then measures how much is lost to encounter constraints (boss invulnerability, forced movement, burst window misalignment). It does NOT find optimal rotations — it quantifies the difference between rotation variants and measures distance from theoretical maximum. Uses job data and trait potency scaling from the `marbleperfume/destructo` public GitHub repo. Output validates whether a job's numbers meet DPS check thresholds and exposes where small potency differences compound over 7+ minute encounters.

## Data Source

All job data lives at: `https://raw.githubusercontent.com/marbleperfume/destructo/main/ffxiv/`

**The repo is public** — no authentication needed. Use `url_fetch` directly on raw.githubusercontent.com URLs. No browser navigation required for data access.

- **Job JSONs**: `classes/{role}/{job}.json` — contains skill names, potency values, cooldowns, GCD/oGCD classification, combo chains, proc rates
- **Trait scaling**: `ffxiv_trait_potency_scaling.json` — contains pre_94/post_94 potency values, mastery chains, enhanced traits, conditional traits, classifications

Role folders: `tank/`, `healer/`, `melee/`, `phys_ranged/`, `magic_ranged/`

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

**For the model:** When comparing jobs, the "fair" comparison is aDPS (how much each job contributes personally). When evaluating a PARTY'S output, rDPS captures buff contribution. The scatter/box-whisker output should default to personal PPS but note the rDPS adjustment for buff jobs.

## Workflow

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
- **Input**: Job JSON + trait scaling JSON + `{{fight_duration}}` + `{{level}}` + `{{boss_invuln_windows}}`
- **Output**: Array of damage events: `[{time_sec, skill_name, potency, type, is_crit_window}]`
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

### Step 3: Run Monte Carlo (if proc-based job)
- **Mode**: `deterministic`
- **Tool**: `run_python`
- **Input**: Rotation model from Step 2, iteration count (1000)
- **Output**: Distribution of total potency values + per-event timing arrays for all iterations
- **Validate**: Standard deviation is non-zero for proc jobs; deterministic jobs should have σ ≈ 0
- **On failure**: Check RNG seeding; ensure proc rates match wiki values

### Step 4: Generate Visualization
- **Mode**: `agentic`
- **Tool**: `run_python` + `file_write`
- **Input**: Simulation results + `{{output_format}}`
- **Output**: HTML file with Highcharts visualization(s)
- **Validate**: HTML file exists and contains valid Highcharts configuration
- **On failure**: Check that Highcharts is loaded from `/vendor/highcharts/`; verify container divs have fixed heights

Visualization types:
- **scatter**: X = time (sec), Y = potency per hit. Color-code by skill type (GCD/oGCD/DoT). Mark burst windows with plotBands. Mark boss invuln with gray bands. For BLM, marker X-width should reflect actual cast time (wide markers for Fire IV, narrow for instants).
- **box**: Horizontal box-and-whisker ranked by aDPS (like FFLogs zone statistics). Jobs as Y-axis categories, PPS as X-axis. Shows median, IQR, whiskers, outliers. Color-code by job role.
- **cumulative**: Line chart of cumulative potency over time. Shows burst ramp and sustain phases. DoT contribution is a stacked area beneath.
- **all**: Dashboard with all three charts in a 2x2 grid.

### Step 5: Present Output
- **Mode**: `deterministic`
- **Tool**: `open_in_session_tab`
- **Input**: Path to generated HTML file
- **Output**: Interactive chart visible to user
- **Validate**: File opens without error
- **On failure**: Check file path; ensure HTML is well-formed

## Output

An interactive Highcharts HTML visualization showing:
- Scatter plot of damage events over fight duration (color = skill type, size = potency magnitude)
- Box-and-whisker of PPS distribution ranked by job (horizontal, FFLogs-style)
- Cumulative potency timeline with DoT contribution as stacked area
- Boss invulnerability windows shown as gray shaded bands (the only dead zones)
- Summary stats: mean PPS, median, p10/p90, burst phase PPS vs sustain phase PPS

## Lessons Learned

### Do
- Always fetch BOTH the job JSON and trait scaling JSON — potency values in the job JSON may be base (pre-trait) values
- Use `level_breakpoint` field to determine which potency tier applies at `{{level}}`
- Model DoT ticks as first-class damage events in the scatter plot (same marker style as GCDs, just different color)
- For box-and-whisker comparisons, normalize to "potency per second" so jobs with different GCD speeds are comparable
- Include the `classification` field from trait JSON to determine skill types for color-coding
- Target output that approximates FFLogs aDPS distributions for validation — if relative rankings diverge significantly from FFLogs, the model has an error
- Apply haste modifiers FIRST, then Skill Speed substats on top — they're multiplicative, not additive
- For BLM: model the full AF/UI cycle with per-spell cast times, not a flat GCD

### Don't
- Never exclude DoT ticks from DPS calculations or visualizations — they are continuous damage, only paused by boss invuln
- Never model "dead time" for FFXIV DPS jobs — the GCD is always rolling
- Don't assume all jobs are proc-based — fixed-rotation jobs (WAR, DRG, SAM) need only 1 iteration
- Don't use hardcoded potency values — always pull from the JSON, which reflects the latest patch data
- Don't model gear effects — FFXIV gear has no procs or rotation-altering mechanics
- Don't use 2.50s GCD for all jobs — NIN/MNK/SAM have fundamentally different GCD speeds
- Don't use a flat GCD for BLM — it has per-spell cast times where Fire IV (2.8s), Despair (3.0s), and Flare Star (3.0s) are all longer than the GCD recast

### Common Failures
- **GitHub raw URL 404**: The repo structure may have changed. Fall back to checking the tree via `url_fetch` on the repo page.
- **Infinite loop in proc sim**: BRD's Apex Arrow gauge can overflow if you don't cap it at 100. Always clamp resource gauges.
- **Trait JSON mismatch**: Some skills have `conditional: true` traits (e.g., NIN Meisui). Check the `condition` field before applying flat potency deltas.
- **Level sync edge case**: Some traits don't exist below Lv84 at all. If `level < level_breakpoint`, use the `base` potency (not pre_94).
- **Haste omission**: Forgetting to apply NIN/MNK/SAM haste results in ~15-20% lower GCD count, making these jobs appear much weaker than they are.
- **BLM flat GCD error**: Modeling BLM with a uniform 2.5s tick vastly overestimates GCD count. The real cycle is ~32s for one full AF/UI rotation due to long cast times.
- **BLM AF damage multiplier**: Fire spells deal 1.8x damage in Astral Fire III. Forgetting this makes BLM appear to deal half its actual damage.

### When to Ask the User
- Which boss fight to model (determines invulnerability windows)
- Whether to use "ping-adjusted" GCD (2.50s vs 2.48s for low-ping players)
- For DNC specifically: whether to model partner synergy contribution or solo only
- If comparing two jobs: whether to normalize for raid buff contribution or raw personal DPS
- For BLM: whether to model standard rotation (6x Fire IV) or Paradox-heavy rotation (more instants, shorter AF cycles, viable at high SpS)
