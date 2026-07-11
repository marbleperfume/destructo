# FFXIV Analysis Framework (Patch 7.5)

## Purpose

This document defines the evaluation models the template system uses to compare builds across roles and jobs. It bridges raw potency data (from job JSONs) to meaningful performance metrics.

---

## A. Personal DPS Model (PPS)

### Core Metric: Potency Per Second

```
PPS = total_potency_in_rotation / rotation_duration_seconds
```

This is the FFXIV equivalent of ToS's SFR/second — the universal throughput unit that normalizes across different action timings.

### Burst PPS vs Filler PPS

Most jobs have two distinct output modes:
- **Burst PPS**: potency output during buff/cooldown alignment windows (typically 20-30s every 120s)
- **Filler PPS**: potency output during maintenance/waiting periods

```
effective_pps = (burst_pps × burst_duration + filler_pps × filler_duration) / total_cycle
```

Jobs with high burst/filler ratio (Samurai, Ninja) benefit more from shorter kill times.

### Proc-Adjusted PPS

For non-deterministic jobs (Dancer, Bard, Black Mage):

```
expected_pps = sum(action_potency × expected_uses_per_cycle) / cycle_duration
expected_uses = base_uses + proc_rate × opportunities_per_cycle
```

Variance matters: a Dancer with bad luck can output 15% below expected. Model both E[PPS] and σ[PPS].

### Phase-Weighted PPS

For transformation jobs (Summoner, Reaper, Dark Knight):

```
cycle_pps = sum(phase_pps_i × phase_duration_i) / sum(phase_duration_i)
```

Summoner example:
- Bahamut phase (15s): high potency summon + player actions
- Phoenix phase (15s): similar structure, different potency
- Demi-summon phases (3 × ~15s): primal summons + player actions
- Filler between phases: lower potency GCDs

### Resource Waste Penalty

Overcapping gauge/resource = lost potency opportunity:

```
waste_penalty = overcap_events_per_cycle × potency_per_gauge_spend × gauge_overflow
adjusted_pps = raw_pps - (waste_penalty / cycle_duration)
```

Examples:
- Warrior overcapping Beast Gauge at 100 = lost Fell Cleaves
- Samurai letting Kenki sit at 100 during combo = lost Shinten casts

---

## B. Raid DPS (rDPS) Model

### Why This Exists

A Dancer with 85% of a Samurai's personal DPS can contribute MORE total raid damage through party buffs. rDPS captures this.

### Formula

```
rDPS = personal_dps + contributed_dps

contributed_dps = sum_over_party_members(
    ally_pps_during_buff × damage_formula_multiplier × buff_strength × buff_uptime
)
```

### Standard Party Assumption

Full party: 2 tanks + 2 healers + 4 DPS

Approximate personal PPS by role (level 100, BIS gear):
- Melee DPS: ~95-110 PPS
- Physical Ranged: ~85-95 PPS
- Caster: ~90-105 PPS
- Tank: ~70-80 PPS
- Healer: ~55-65 PPS

### Buff Alignment (120s Cycle)

Most raid buffs operate on a 120s cycle. The standard burst window structure:

```
T=0.00  Opener (all buffs stack)
T=120s  2-minute burst (all buffs realign)
T=240s  Next cycle (repeat)
```

Buffs that don't align to 120s (Bard songs at 45s cycle) have different contribution models.

### Major Raid Buffs (120s Alignment)

| Job | Buff | Effect | Duration |
|-----|------|--------|----------|
| Dancer | Technical Finish | +5% party damage | 20s |
| Dancer | Devilment | +20% crit/DH (partner) | 20s |
| Ninja | Mug (Dokumori) | +5% target vuln | 20s |
| Dragoon | Battle Litany | +10% party crit rate | 20s |
| Reaper | Arcane Circle | +3% party damage | 20s |
| Bard | Battle Voice | +20% DH rate (party) | 15s |
| Bard | Radiant Finale | +2-6% party damage | 20s |
| Monk | Brotherhood | +5% party damage | 20s |
| Scholar | Chain Stratagem | +10% crit rate (target) | 20s |
| Astrologian | Divination | +6% party damage | 20s |
| Red Mage | Embolden | +5% party damage | 20s |
| Summoner | Searing Light | +5% party damage | 20s |
| Pictomancer | Starry Muse | +5% party damage | 20s |

### Selfish DPS Jobs (No Raid Buff)

These compensate with higher personal PPS:
- Samurai (highest personal melee)
- Viper (very high personal melee)
- Black Mage (highest caster personal)

Their rDPS contribution = personal DPS only. They're valued when raw DPS checks matter more than buff stacking.

---

## C. Healer Efficiency Model

### Core Tension

Every GCD spent healing is a GCD NOT spent dealing damage.

```
dps_loss_per_heal_gcd = glare_equivalent_potency
```

Values:
- WHM Glare IV: 310 potency lost per heal GCD
- SCH Broil IV: 295 potency lost per heal GCD
- AST Malefic IV: 270 potency lost per heal GCD (but has card buffs)
- SGE Dosis III: 330 potency lost per heal GCD

### Healing Priority Hierarchy

1. **oGCD heals (FREE)** — no DPS cost, use first always
2. **Lily/resource heals (partially free)** — builds toward damage payoff (WHM Misery)
3. **GCD heals (EXPENSIVE)** — potency lost = damage not dealt

### Efficiency Metric

```
healer_efficiency = damage_pps_maintained / damage_pps_maximum

or equivalently:

efficiency = 1 - (gcd_heals_used × glare_potency) / (total_gcds_available × glare_potency)
```

### Overheal as Waste

```
effective_healing = raw_healing × (1 - overheal_rate)
wasted_potency = overheal_rate × heal_potency (if GCD heal)
```

Shield-based healers (Scholar, Sage) can waste shields if damage doesn't arrive before shield expires.

### Sage Paradigm: Kardia

Sage's Kardia makes every damage action also heal a target for ~170 potency. This means:
- Sage's "cost" of healing is partially zero (damage = healing simultaneously)
- Sage efficiency model must account for passive Kardia HPS as always-on
- Addersgall charges (oGCD heals) are the primary active healing resource

### Model Output

For each healer, report:
- Max damage PPS (100% Glare/Broil/Dosis uptime, zero healing)
- Realistic damage PPS (given encounter healing requirements)
- Free healing budget (total oGCD + Kardia + passive HPS before GCD heals needed)
- GCD heal threshold (incoming DPS level that forces GCD healing)

---

## D. Tank Value Model

### Composite Value

```
tank_value = personal_dps + mitigation_value + self_heal_value + party_mitigation_value
```

### Personal DPS

Tanks have lower PPS than DPS jobs but it's still the largest single contributor to their value. Model same as Section A.

### Mitigation Value

```
mitigation_value = sum_over_abilities(
    damage_prevented × ability_uptime / encounter_duration
)

damage_prevented = incoming_damage × (1 - (1 - mit_strength))
```

With multiplicative stacking:
```
total_mitigation = 1 - (1 - mit_1) × (1 - mit_2) × (1 - mit_3)
Example: 30% + 10% + 10% = 1 - 0.7 × 0.9 × 0.9 = 43.3% total
```

### Self-Healing as Healer Relief

Every HP a tank restores themselves = one less HP the healer must spend GCDs to restore:

```
self_heal_value = self_hps × (healer_potency_per_heal_gcd / heal_potency_per_gcd_heal)
```

Warrior is the extreme case: Inner Release + Bloodwhetting makes them effectively self-sufficient during burst.

### Party Mitigation

Party-wide mitigation abilities reduce total party damage taken:
- Dark Missionary (DRK): -10% magic damage, 15s/90s
- Shake It Off (WAR): party shield (~15% HP), 30s/90s
- Divine Veil (PLD): party shield (~10% HP), 30s/90s
- Heart of Light (GNB): -10% magic damage, 15s/90s

Value = party_members_affected × damage_reduced_per_member × uses_per_fight

### Invulnerability Windows

Not generalizable to a metric — binary (alive or dead). Fight-specific:
- Hallowed Ground (PLD): 10s invuln, 420s CD (longest CD, cleanest invuln)
- Holmgang (WAR): 10s can't drop below 1 HP, 240s CD (shortest CD)
- Living Dead (DRK): complex condition (must be healed to full after)
- Superbolide (GNB): drops to 1 HP then 10s invuln, 360s CD

---

## E. Pet/Summon Track Model

### Fundamental Principle

Pets and summons are **autonomous DPS sources on their own timeline**. They do NOT consume the player's GCD or oGCD budget. They must be modeled as a SEPARATE damage track running in parallel.

This is the same paradigm as ToS "additional attacks" (Enchant Fire, Sacrament) — separate formula, separate timing, additive to player damage without competing for action resources.

### Pet Coefficient

Pet damage uses a DIFFERENT potency-to-damage conversion than player damage:

```
pet_damage = pet_potency × pet_coefficient × f(owner_stats)
```

The pet coefficient is lower than the player coefficient. A pet action with "300 potency" does NOT deal the same damage as a player action with 300 potency. The ratio is approximately 0.85-0.95x depending on the pet type.

### Buff Interaction

- **Party buffs**: Most affect pet damage (Technical Finish, Chain Stratagem)
- **Personal buffs**: Some do, some don't (job-specific)
- **Snapshotting**: Pets snapshot owner stats at summon time (some) or dynamically update (others)

### Job-Specific Pet Tracks

#### Summoner (SMN)
- **Demi-summons** (Bahamut/Phoenix): 15s each, fire powerful oGCDs autonomously while player also attacks
- **Primal summons** (Ifrit/Titan/Garuda): grant player different action sets but summon also auto-attacks
- Pet is ALWAYS present — DPS track never has downtime
- Player must cycle: Bahamut → Primals → Phoenix → Primals → repeat

#### Scholar (SCH)
- **Fairy** (Eos/Selene): passive healing every ~3s, plus commandable oGCD heals
- Not a DPS track — contributes HEALING autonomously
- Embrace (auto-heal): ~180 potency every 3s to lowest-HP party member
- Fairy gauge abilities: Fey Blessing, Fey Illumination (player-commanded)
- Dissipation: dismisses fairy for 30s, grants healer DPS boost (sacrifice autonomy for burst)

#### Machinist (MCH)
- **Automaton Queen**: summoned for 12-20s (scales with Battery gauge spent)
- Battery gauge: 50-100 stored, higher = longer summon + stronger finisher
- Autonomous attack sequence: Arm Punch → Roller Dash → Pile Bunker (finisher)
- Finisher potency scales with remaining time: deploy at 100 Battery for maximum value
- Key optimization: when to deploy (align with party buffs vs overcap prevention)

#### Dark Knight (DRK)
- **Living Shadow**: 24s clone that fires a fixed sequence of attacks autonomously
- 6 attacks over 24s (every 4s approximately)
- Each hit: 420-570 potency (on pet coefficient)
- Summoned on 120s CD — aligns with burst windows
- Unique: the ONLY "pet" in the tank role

#### Reaper (RPR)
- **Lemure attacks**: during Enshroud phase, autonomous Lemure slashes fire between player GCDs
- Not a persistent pet — only active during transformation (15s window)
- Each Lemure Slice: 100 potency (on pet coefficient)
- 4-5 Lemure hits per Enshroud phase
- Functionally acts like "free bonus hits" during burst — similar to ToS Enchant Fire

### Model Output

For each pet/summon job, report:
- Player PPS (actions the player executes)
- Pet PPS (autonomous actions the pet executes, on pet coefficient)
- Combined PPS = player_pps + (pet_pps × pet_coefficient_ratio)
- Buff alignment: does pet benefit from burst window stacking?
- Pet uptime%: what fraction of fight duration is the pet active?

---

## F. Fight Context Variables

### Kill Time

```
kill_time_factor(job) = actual_pps_over_kill_time / theoretical_infinite_fight_pps
```

- Short kills (<3 min): favor burst jobs (Ninja, Samurai, Summoner opener)
- Medium kills (3-8 min): balanced — all jobs reach steady state
- Long kills (>8 min): favor sustain + MP management (Black Mage, Dancer)
- Fight ending mid-burst-cycle vs post-burst shifts job value significantly

Kill time also interacts with buff alignment:
- Kill at 7:00 = 3 full 2-minute windows + extra filler (bad for burst jobs)
- Kill at 6:00 = 3 full 2-minute windows ending on burst (good for burst jobs)

### Target Count (AoE Breakpoints)

Each job has a target count where AoE rotation exceeds single-target PPS:

| Archetype | AoE Breakpoint | Notes |
|-----------|---------------|-------|
| Most melee | 2-3 targets | AoE combo replaces ST combo |
| Physical ranged | 2 targets | Very efficient AoE |
| Casters | 3 targets | AoE spells have lower per-target potency |
| Tanks | 2 targets | AoE combo usually matches ST at 2 |

Model should calculate: `aoe_pps(n_targets) = per_target_potency × n_targets` and find crossover with ST PPS.

### Uptime Percentage

```
effective_pps = theoretical_pps × uptime_fraction
```

- Melee jobs: ~95-98% uptime in hard content (forced disengagement for mechanics)
- Physical ranged: ~99% (full mobility, can always hit)
- Casters: ~97-99% (slidecasting + instants manage most mechanics)
- Ranged tax: physical ranged have lower potency to compensate for perfect uptime

Uptime is the primary reason physical ranged have lower PPS — they NEVER lose uptime, so their theoretical max is reduced by design.

### Phase Transitions (Forced Downtime)

Boss becomes untargetable → burst window alignment breaks:
- 2-minute buffs may come off cooldown during downtime (waste)
- DoTs fall off, must reapply (DPS loss for DoT-heavy jobs)
- Gauge may overcap during downtime (resource waste)
- Phase transition timing is fight-specific — model as "uptime holes"

### GCD Drift

Over time, tiny animation delays accumulate:
```
drift_loss = (actual_gcd - theoretical_gcd) × actions_per_minute
```

High-APM jobs (Gunbreaker, Viper, Ninja) suffer more from drift. Ping affects this directly.

---

## G. Cross-Game Template Mapping

### Equivalent Concepts

| FFXIV | ToS | Shared Abstraction |
|-------|-----|-------------------|
| PPS (potency/second) | SFR/second | **Throughput rate** |
| GCD (2.5s cycle) | Animation lock (~0.7s) | **Action timing constraint** |
| Burst window (20s every 120s) | Burst dump (all OH spent in 6s) | **Front-loaded output phase** |
| Filler rotation | Dead time (CD recovery) | **Low-output maintenance phase** |
| Pet/summon track | Additional attacks (Enchant Fire) | **Autonomous parallel damage** |
| AoE breakpoint (2-3 targets) | AoE Attack Ratio budget | **Multi-target efficiency threshold** |
| oGCD weaving | No equivalent (instant + no GCD) | *FFXIV-only constraint* |
| rDPS (party buff contribution) | No equivalent (solo game) | *FFXIV-only metric* |
| Proc variance (Dancer RNG) | No equivalent (deterministic) | *FFXIV-only uncertainty* |
| Uptime% (forced disengage) | Not applicable (no disengage) | *FFXIV-only penalty* |
| Mitigation stacking (multiplicative) | No equivalent (no party) | *FFXIV-only* |
| DoT snapshotting | No equivalent (no DoTs) | *FFXIV-only* |
| — | Armor matchup (Ghost×0.5) | *ToS-only multiplier* |
| — | Non-linear DEF formula (log) | *ToS-only damage reduction* |
| — | Element system (weakness ×1.5) | *ToS-only matchup layer* |
| — | 136-class mixing (pick 4) | *ToS-only build combinatorics* |
| — | SP sustainability (rotation cost) | *ToS-only resource drain* |
| — | Block penalty (half damage) | *ToS-only defensive stat* |
| — | Enhance/Arts investment (silver cost) | *ToS-only progression variable* |

### Template System Implications

The shared abstraction layer (throughput rate, timing constraint, burst/filler, parallel tracks) forms the GAME-AGNOSTIC template core. Game-specific layers extend it:

```
Template Core (game-agnostic):
├── throughput_rate (PPS or SFR/s)
├── action_timing_model (GCD or animation lock)
├── burst_cycle (window or dump)
├── parallel_tracks (pet or additional attacks)
└── target_scaling (AoE efficiency)

FFXIV Extension:
├── raid_buff_contribution (rDPS)
├── proc_variance (E[PPS], σ[PPS])
├── uptime_modeling (forced disengage)
├── role_value (healer efficiency, tank mitigation)
└── dot_management (snapshot + tick optimization)

ToS Extension:
├── armor_matchup (element × armor type grid)
├── defense_penetration (log-based DEF formula)
├── sp_economy (cost/income sustainability)
├── class_combination (4-pick synergy from 136)
├── enhance_investment (silver cost → damage scaling)
└── block_interaction (pen check vs monster BLK)
```

---

## H. Model Priority for Implementation

### Phase 1: Core (Already Built)
- [x] SFR/second for ToS
- [x] Burst dump → dead time cycle (ToS)
- [x] TTK calculation with DEF formula

### Phase 2: Next
- [ ] PPS calculation for FFXIV (using scraped action data)
- [ ] Pet/summon as parallel track (SMN, MCH, DRK)
- [ ] AoE breakpoint calculator (both games)
- [ ] SP sustainability check (ToS long fights)

### Phase 3: Advanced
- [ ] rDPS modeling (requires party composition input)
- [ ] Proc simulation (Dancer/Bard expected value)
- [ ] Healer efficiency model
- [ ] Tank mitigation timeline
- [ ] Kill time sensitivity analysis

### Phase 4: Integration
- [ ] Cross-game comparison dashboard
- [ ] Template agent auto-evaluation given user inputs
- [ ] Level slider (ToS) + ilvl slider (FFXIV)
- [ ] Scenario presets (speedkill vs prog vs farming)
