# FFXIV Job Mechanics Index (Patch 7.5)

## Mechanic Archetype Classification

Each job is assigned to one or more archetypes that determine how the template system models its rotation and value.

---

## A. Deterministic Combo Jobs

Fixed combo routes. Rotation is a solved sequence repeated cyclically. Optimization = execution precision, not decision-making.

### Paladin (Tank)
- **Resources:** MP (for magic phase), Oath Gauge (for defensive abilities)
- **Rotation type:** Deterministic with phase alternation (physical combo + magic burst phase)
- **Party contribution:** Divine Veil (party shield), Passage of Arms (party DR)
- **Unique mechanic:** Alternating melee combo phase and magic burst phase (Requiescat window). Dual resource tracks.
- **Pet/summon:** None

### Warrior (Tank)
- **Resources:** Beast Gauge (builds from combo, spends on Fell Cleave/Decimate)
- **Rotation type:** Deterministic builder-spender
- **Party contribution:** Shake It Off (party shield), Nascent Flash (ally heal)
- **Unique mechanic:** Inner Release window (guaranteed crits + free Fell Cleaves for 15s). Highest self-sustain of all tanks.
- **Pet/summon:** None

### Gunbreaker (Tank)
- **Resources:** Cartridge system (builds from combo, spends on Gnashing Fang/Burst Strike)
- **Rotation type:** Deterministic, very busy (most oGCDs of any tank)
- **Party contribution:** Heart of Corundum (strong single-target mit), Aurora (HoT)
- **Unique mechanic:** Continuation combo (oGCD chain triggered by specific GCDs). Highest APM tank.
- **Pet/summon:** None

### Dragoon (Melee DPS)
- **Resources:** Life of the Dragon gauge (builds from jump abilities)
- **Rotation type:** Deterministic 5-GCD combo chains + jump oGCDs
- **Party contribution:** Battle Litany (+10% party crit rate, 20s/120s)
- **Unique mechanic:** Two alternating 5-hit combo paths. Jump animations lock movement.
- **Pet/summon:** None

### Samurai (Melee DPS)
- **Resources:** Kenki gauge + 3 Sen (Setsu/Getsu/Ka from different combos)
- **Rotation type:** Deterministic (which combo to do is fixed by Sen needed)
- **Party contribution:** NONE (pure selfish DPS -- highest personal potency)
- **Unique mechanic:** Three combo routes each grant one Sen. Spending 1/2/3 Sen = different Iaijutsu finishers. Highest personal DPS in exchange for zero raid utility.
- **Pet/summon:** None

### Viper (Melee DPS)
- **Resources:** Rattling Coil (gauge from combo finishers)
- **Rotation type:** Deterministic dual-combo system (two parallel combo paths)
- **Party contribution:** None (selfish DPS)
- **Unique mechanic:** Two interleaving combo chains that must alternate. Reawaken phase (burst state with unique actions). Very high APM.
- **Pet/summon:** None

### Reaper (Melee DPS)
- **Resources:** Soul gauge + Shroud gauge (Soul builds from combo, Shroud from Soul spenders)
- **Rotation type:** Deterministic builder into transformation phase
- **Party contribution:** Arcane Circle (+3% party damage, 20s/120s)
- **Unique mechanic:** Enshroud transformation (replaces entire action set for burst window). Two-stage resource pipeline.
- **Pet/summon:** Avatar (Lemure attacks during Enshroud -- autonomous hits)

---

## B. Proc-Reactive Jobs

Rotation depends on random proc outcomes. Optimization = correct priority decisions under uncertainty.

### Dancer (Physical Ranged DPS)
- **Resources:** Esprit gauge (builds from party members attacking during Technical), Feathers (from procs)
- **Rotation type:** Heavily proc-based (Flourish grants random proc combos)
- **Party contribution:** Technical Finish (+5% party damage, 20s/120s), Devilment (+20% crit/DH to partner, 20s/120s), Standard Step (+5% to self + partner)
- **Unique mechanic:** Most RNG-dependent job in the game. Standard/Technical Steps are dance minigames. Closed Position (dance partner) buffs one ally permanently. Lowest personal DPS but highest rDPS contribution.
- **Pet/summon:** None

### Bard (Physical Ranged DPS)
- **Resources:** 3 Songs (cycle: Wanderer's Minuet 45s > Mage's Ballad 45s > Army's Paeon 45s), Repertoire procs within each song
- **Rotation type:** Proc-reactive within deterministic song cycle
- **Party contribution:** Battle Voice (+20% DH rate to party, 15s/120s), Radiant Finale (+2-6% damage based on Coda collected)
- **Unique mechanic:** Song rotation provides framework, but Repertoire procs within songs are random. Each song gives different proc effects. Requires tracking DoT uptime for proc generation.
- **Pet/summon:** None

### Black Mage (Magical Ranged DPS)
- **Resources:** MP + Astral Fire/Umbral Ice state (stance system)
- **Rotation type:** Semi-deterministic with proc optimization (Firestarter, Thundercloud)
- **Party contribution:** NONE (pure selfish DPS, highest magic DPS potential)
- **Unique mechanic:** Stance dancing between Fire (high damage, drains MP) and Ice (no damage, restores MP). Extremely turret-like (long cast times, punished by movement). Procs allow instant-cast fire spells for movement.
- **Pet/summon:** None

---

## C. Phase/Transformation Jobs

Rotation has distinct phases with different action sets or behaviors. Optimization = phase transition timing and resource management between phases.

### Summoner (Magical Ranged DPS)
- **Resources:** Gems (Ifrit/Titan/Garuda), Trance gauge (Bahamut/Phoenix/Solar Bahamut)
- **Rotation type:** Phase-based (rigid cycle: Bahamut > 3 primals > Phoenix > 3 primals > Solar Bahamut > ...)
- **Party contribution:** Searing Light (+5% party damage, 20s/120s)
- **Unique mechanic:** Each primal gem = 2 GCDs of unique element. Trances summon Bahamut/Phoenix for 15s (attacks alongside player as autonomous pet). Most mobile caster (nearly all instant casts). Pets are autonomous damage tracks.
- **Pet/summon:** Bahamut, Phoenix, Solar Bahamut (15s autonomous attacker per trance)

### Red Mage (Magical Ranged DPS)
- **Resources:** White Mana + Black Mana (dual gauge, must stay balanced)
- **Rotation type:** Builder-spender with melee burst phase
- **Party contribution:** Embolden (+5% party damage, 20s/120s), Magick Barrier (party magic DR + heal amplification)
- **Unique mechanic:** Build both mana gauges via ranged spells (Jolt > Veraero/Verthunder), then dump into melee combo burst phase. Dual-casting system (every hardcast is followed by an instant Dualcast). Raises allies instantly (Verraise, massive prog utility).
- **Pet/summon:** None

### Dark Knight (Tank)
- **Resources:** MP (for Edge/Flood of Shadow), Blood Gauge (for Bloodspiller/Living Shadow)
- **Rotation type:** Deterministic with autonomous summon phase
- **Party contribution:** The Blackest Night (strongest single-target shield in game), Dark Missionary (party magic DR)
- **Unique mechanic:** Living Shadow summons an autonomous clone for 24s that fires preset attacks on its own timeline. TBN shield doubles as DPS resource (breaking shield = free Edge of Shadow).
- **Pet/summon:** Living Shadow (24s autonomous attacker, preset rotation)

---

## D. Builder-Spender with Gauge

Build resource through rotation, spend on burst abilities. Optimization = minimize overcap, maximize spender timing within buff windows.

### Machinist (Physical Ranged DPS)
- **Resources:** Heat gauge (builds from combo, spend on Hypercharge), Battery gauge (builds from Drill/Air Anchor, spend on Automaton)
- **Rotation type:** Deterministic with timed burst windows
- **Party contribution:** NONE (selfish DPS)
- **Unique mechanic:** Hypercharge = 5 rapid-fire GCDs (1.5s each). Automaton Queen = autonomous pet scaled by Battery gauge at deployment. Highest burst ceiling of physical ranged. Wildfire (detonation bomb based on actions during window).
- **Pet/summon:** Automaton Queen (autonomous damage, potency scales with Battery spent)

### Ninja (Melee DPS)
- **Resources:** Ninki gauge (builds from auto-attacks and combo), Mudra system (combinatorial)
- **Rotation type:** Deterministic 60s loop with Trick Attack windows
- **Party contribution:** Mug/Trick Attack (+5% vulnerability on target, 15s/120s -- target debuff, not party buff)
- **Unique mechanic:** Mudra system -- combine Ten/Chi/Jin inputs to produce different Ninjutsu. Bunshin (shadow clones that copy attacks). Trick Attack window concentrates party burst on target.
- **Pet/summon:** Bunshin (shadow clones that mimic attacks -- NOT autonomous, triggered by player GCDs)

### Monk (Melee DPS)
- **Resources:** Chakra (0-5, generated on crits), Nadi (Lunar/Solar, granted by Blitz type), Beast Chakra (form state tracking)
- **Rotation type:** Deterministic form chain (Opo > Raptor > Coeurl > Blitz)
- **Party contribution:** Brotherhood (+5% party damage, 20s/120s; also generates Monk Chakra from party hits)
- **Unique mechanic:** Positional attacks (rear/flank bonuses). Form chain locks action sequence. Blitz finisher determined by Beast Chakra variety. Riddle of Fire burst window. Highest sustained melee DPS.
- **Pet/summon:** None

---

## E. Healers with Damage Optimization

Primary metric is NOT HPS but rather: how much DPS can you maintain while keeping the party alive? GCD healing = DPS loss.

### White Mage (Healer)
- **Resources:** Lily gauge (fills over time, spend on GCD heals; 3 Lilies = 1 Misery damage spell)
- **Rotation type:** Deterministic (Glare spam + Dia DoT maintenance)
- **Party contribution:** Temperance (party DR + heal amp), Liturgy of the Bell (raid-wide healing)
- **Unique mechanic:** Simplest healer rotation (Glare spam). Afflatus system converts healing GCDs into eventual damage (Misery). Blood Lily tracks accumulated healing cost. Strongest raw GCD heals.
- **Pet/summon:** None

### Scholar (Healer)
- **Resources:** Aetherflow (3 stacks/60s, powers key heals and damage), Fairy Gauge (for Seraph/enhanced abilities)
- **Rotation type:** Deterministic (Broil spam + Biolysis DoT), resource management on Aetherflow
- **Party contribution:** Chain Stratagem (+10% crit rate on target, 20s/120s), Expedient (party sprint + DR)
- **Unique mechanic:** Fairy pet operates on separate healing timeline (Embrace auto-heal). Aetherflow as shared resource between damage (Energy Drain) and healing (Lustrate/Indomitability). Shield-focused (Adloquium, Succor).
- **Pet/summon:** Eos/Selene (permanent healing pet), Seraph (enhanced fairy for 22s)

### Astrologian (Healer)
- **Resources:** Cards (draw system, apply damage buffs to party members), Arcana (spending cards charges abilities)
- **Rotation type:** Deterministic (Malefic spam + Combust DoT) with card management layer
- **Party contribution:** Divination (+6% party damage, 20s/120s), Cards (+6% damage to individual targets, 15s each)
- **Unique mechanic:** Card system buffs individual party members during burst windows. Highest raid buff contribution of any healer. Balance between shielding (Nocturnal) and regen (Diurnal) stances removed in recent patches.
- **Pet/summon:** None

### Sage (Healer)
- **Resources:** Addersgall (regenerating charges for oGCD heals), Addersting (from broken shields)
- **Rotation type:** Deterministic (Dosis spam + Eukrasian Dosis DoT)
- **Party contribution:** Kerachole (party DR + regen), Panhaima (party shields)
- **Unique mechanic:** Kardia -- passive where EVERY damage spell also heals the Kardia target. Damage literally equals healing. This makes Sage the only healer where DPS and healing are not in competition. Shield-focused kit.
- **Pet/summon:** None

---

## F. Resource Management Specialists

Jobs where the PRIMARY optimization challenge is resource management rather than just hitting buttons on cooldown.

### Black Mage
(See Proc-Reactive section above)
- Stance management IS the rotation. Astral Fire = spend MP on big damage. Umbral Ice = recover MP while maintaining upkeep. One wrong transpose or dropped enochian = massive DPS loss.

### Paladin
(See Deterministic section above)
- MP management across physical/magic phases. Oath for defensive economy. Two resources that deplete at different rates per phase.

---

## Party Buff Summary Table

| Job | Buff Name | Effect | Duration | Cooldown | Type |
|-----|-----------|--------|----------|----------|------|
| Dancer | Technical Finish | +5% party damage | 20s | 120s | Party buff |
| Dancer | Devilment | +20% crit+DH (partner) | 20s | 120s | Single ally |
| Dragoon | Battle Litany | +10% party crit rate | 20s | 120s | Party buff |
| Monk | Brotherhood | +5% party damage | 20s | 120s | Party buff |
| Ninja | Mug | +5% damage on target | 20s | 120s | Target debuff |
| Bard | Battle Voice | +20% party DH rate | 15s | 120s | Party buff |
| Bard | Radiant Finale | +2-6% party damage | 20s | 120s | Party buff |
| Red Mage | Embolden | +5% party damage | 20s | 120s | Party buff |
| Summoner | Searing Light | +5% party damage | 20s | 120s | Party buff |
| Reaper | Arcane Circle | +3% party damage | 20s | 120s | Party buff |
| Scholar | Chain Stratagem | +10% crit rate on target | 20s | 120s | Target debuff |
| Astrologian | Divination | +6% party damage | 20s | 120s | Party buff |
| Astrologian | Cards | +6% damage (individual) | 15s | 30s | Single ally |

### Buff Timing Notes
- Nearly all major buffs align on 120s windows
- Opener stacks ALL buffs simultaneously (highest burst moment)
- Re-buffs at 2:00, 4:00, 6:00, etc.
- Ninja's Mug is a TARGET debuff (applies to boss, all party benefits)
- Scholar's Chain is also target debuff (stacks with Mug)
- Dancer's Devilment goes to dance partner only (usually highest DPS)
