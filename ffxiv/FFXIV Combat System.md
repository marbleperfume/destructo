# FFXIV Combat System Reference (Patch 7.5, Level 100)

## GCD and Action Timing

### Global Cooldown (GCD)
- Base recast: **2.50s** for weaponskills/spells
- Reduced by Skill Speed (physical) or Spell Speed (magical)
- Minimum practical GCD: ~2.10s (extreme speed builds)
- Each GCD triggers one weaponskill or spell

### oGCD (Off-Global Cooldown) Abilities
- Usable between GCDs during the "weave window"
- Animation lock per oGCD: **0.6s**
- Single-weave window: ~0.7s free (one oGCD between GCDs safely)
- Double-weave window: ~1.2s free (two oGCDs between GCDs)
- Triple-weave: NOT possible without clipping the next GCD
- Clipping = delaying the next GCD = DPS loss

### Casts and Slidecasting
- Spells with cast time lock the player until ~0.5s before completion
- Slidecast window: can move in the last 0.5s without canceling
- Instant-cast GCDs allow full movement during the GCD timer

### Auto-Attacks
- Melee/physical ranged only (casters have none)
- Fires every ~3.0s (weapon delay, varies by weapon type)
- Effective potency: ~90 per swing
- Affected by haste/speed buffs
- Pauses during cast bars (melee) or out of range
- Accounts for ~15-20% of total melee DPS passively

---

## Damage Formula

### Base Damage Calculation

```
D = floor(Potency * f(ATK) * f(DET) * f(TNC) * f(WD) / 100) * Trait * Buff * Crit/DH * Variance
```

### Stat Subfunctions (Level 100 approximations)

```
f(ATK)  = floor(195 * (MainStat - 390) / 390 + 100) / 100
f(DET)  = floor(140 * (DET - 390) / 390 + 1000) / 1000
f(TNC)  = floor(100 * (TEN - 390) / 390 + 1000) / 1000  [tanks only]
f(WD)   = floor(MainStat * 195 / 390 + WeaponDamage) / 100
```

Note: 390 is the level 100 base substat value. These are approximations of the actual floor-heavy integer math.

### Critical Hits
- Rate determined by CRIT stat: ~5% base, scales to ~25% at high gear
- Damage multiplier: base x1.4, increases slightly with CRIT stat
- Can proc on any damage source independently per hit

### Direct Hits
- Rate determined by DH stat: 0% base, scales to ~25% at high gear
- Damage multiplier: base x1.25 (fixed, does not scale)
- Can proc simultaneously with Critical Hit (CDH = x1.4 * x1.25 = x1.75)
- Tanks and healers often meld DH; some abilities guarantee DH

### Variance
- +/- 5% random roll on every damage instance
- Applied after all other multipliers

---

## Buff and Debuff System

### Multiplicative Stacking
All damage buffs multiply with each other:
```
Total multiplier = buff_1 * buff_2 * buff_3 * ...
Example: 1.05 * 1.03 * 1.10 = 1.1897 (not 1.18)
```

### Snapshotting
- Buffs are "snapshotted" at the moment of cast/application
- A DoT applied during a +10% buff keeps +10% for its entire duration
- Removing the buff after application does NOT reduce the DoT
- This rewards aligning DoT refreshes inside buff windows

### Common Buff Windows
- 60s cycle: some raid buffs (Chain Stratagem, Divination)
- 120s cycle: most major raid buffs align here (Technical Finish, Battle Litany, Brotherhood)
- Opener: first ~20s of a fight where ALL buffs stack maximally

---

## DoT and HoT Mechanics

### Damage over Time (DoTs)
- Tick every 3.0s server-side
- Speed stats add extra ticks (faster speed = more total ticks in same duration)
- Snapshot all buffs/debuffs at application time
- Clipping a DoT early = lost ticks; optimal refresh at ~3s remaining

### Heal over Time (HoTs)
- Same 3.0s tick rate
- Snapshot MND and buffs at application
- Regen, Aspected Benefic, Sacred Soil, etc.

---

## Pet and Summon Damage

### Key Principle: Autonomous Damage Tracks
Pets/summons operate on their OWN action timelines, independent of the player's GCD. They are modeled as **separate damage sources**, not merged into player PPS.

### Pet Damage Coefficient
- Pet actions use a DIFFERENT potency-to-damage conversion than player actions
- Pet potency scale is lower (a pet action with "potency 300" deals less than a player action with "potency 300")
- Approximate pet coefficient: ~0.75x of player coefficient (varies by job/patch)

### Buff Interaction with Pets
- **Buffs that affect pets:** most party damage buffs (Technical Finish, Battle Litany)
- **Buffs that do NOT affect pets:** some self-buffs are player-only
- This is job-specific and changes between patches -- verify per summon

### Pet Types by Job
| Job | Pet/Summon | Behavior | Duration |
|-----|-----------|----------|----------|
| Summoner | Bahamut/Phoenix/Solar Bahamut | Attacks alongside player during trance | 15s windows |
| Summoner | Ifrit/Titan/Garuda | Single-use instant attacks (gems) | Instant |
| Scholar | Eos/Selene/Seraph | Passive healing + commandable oGCDs | Permanent/22s |
| Machinist | Automaton Queen | Autonomous damage, potency = f(Battery gauge) | ~12s |
| Dark Knight | Living Shadow | Fires preset attacks independently | 24s |

### Modeling Implications
- Pet DPS must be calculated SEPARATELY then added to player DPS
- Pet uptime may differ from player uptime (pet stays attacking during movement)
- Resource cost to summon is a player-side cost (gauge, MP, GCD)
- Parallel to ToS "additional attacks" (Sacrament, Enchant Fire): separate formula, ignores some defense interactions

---

## Healing Formula

### Base Healing Calculation
Same structure as damage but:
- Uses MND (Mind) as primary stat
- Potency maps to HP restored instead of damage dealt
- No variance roll on healing
- Critical heals = 1.4x (same crit formula)
- Direct hit does NOT apply to healing

### Healing Efficiency Concepts
- GCD heal = DPS loss (every Cure/Benefic = one missed damage spell)
- oGCD heal = free (Tetragrammaton, Essential Dignity cost no DPS)
- Overheal = waste (healing a full-HP target produces zero value)
- Shield overage = waste (shield expires unused = wasted potency)

---

## Mitigation Math

### Multiplicative Damage Reduction
```
Damage taken = incoming * (1 - mit_1) * (1 - mit_2) * (1 - mit_3)
Example: 100k hit with 30% + 10% + 10%:
  100,000 * 0.70 * 0.90 * 0.90 = 56,700 taken (43.3% total reduction)
```

### Mitigation Sources
- Tank cooldowns (Rampart 20%, Sentinel 30%, etc.)
- Role actions (Reprisal 10% on target, Feint 10%)
- Party shields (Divine Veil, Shake It Off)
- Healer mitigation (Kerachole 10%, Sacred Soil 10%)
- Barrier heals (Adloquium shield, Eukrasian Diagnosis)

### Effective HP (EHP)
```
EHP = HP + shield_value / (1 - total_mitigation)
```
A player with 100k HP, 20k shield, and 30% mit has:
```
EHP = 100,000 + 20,000 / 0.70 = 128,571 effective HP against that hit
```

---

## Speed and Haste

### GCD Reduction Formula
```
GCD = floor(2500 * (1000 - f(SPD)) / 1000) / 100
f(SPD) = floor(130 * (SPD - 390) / 390)
```

### Haste Buffs
- Some abilities grant haste (Greased Lightning, Arrow card)
- Haste multiplies with speed stat reduction
- Affects GCD, auto-attack delay, and DoT tick rate

### Speed vs Other Stats (Optimization)
- More speed = more GCDs per fight = more total potency
- But: more GCDs = more MP consumed, harder to double-weave
- Generally: speed is lowest priority stat for most jobs (exceptions: BLM, MNK)
