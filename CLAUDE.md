# Move/TM verification protocol for this repo (pokehns-expansion)

Standing rule from the user: **never use knowledge from outside this repo.** Every claim about
base stats, abilities, move effects, or move availability must be verified against this repo's
own source code.

## How to check if a species can learn a move — correct order

1. **Level-up learnset**: `src/data/pokemon/level_up_learnsets/gen_9.h` (array `s<Species>LevelUpLearnset`).
   Most reliable — always check first.
2. **Egg move**: `src/data/pokemon/egg_moves.h` (array `s<Species>EggMoveLearnset`). Check the
   species AND every pre-evolution in its line (egg moves are inherited through evolution).
3. **TM/Tutor move — DO NOT just grep `items.h` for `[ITEM_TM_<MOVE>]`.** That only finds the
   50 "legacy" TMs that have hand-written flavor text/price in `items.h` (TM01–TM50, classic
   Emerald set). **It misses the other ~108 TMs.** This was a real mistake made earlier in this
   session (wrongly concluded Sleep Talk, Roost, Thunder Wave, Drain Punch, U-Turn, Swords Dance
   had no TM — all of these actually do).

   The correct check has two parts, both required:
   - **a)** Is the move in the real TM/HM list? → `include/constants/tms_hms.h`, the
     `FOREACH_TM` macro block (158 moves total as of this check). Extract with:
     `grep -oP '(?<=F\()[A-Z_]+(?=\))' include/constants/tms_hms.h`
   - **b)** Is the move listed for that species in `src/data/pokemon/all_learnables.json`
     (key = species name in SCREAMING_CASE, value = list of `MOVE_X` strings)?
     **Correction to earlier standing belief:** this file is NOT "just a potential/reference list
     copied from official games that must never be treated as proof." It is the literal build-time
     input to `tools/learnset_helpers/make_teachables.py`, which generates
     `src/data/pokemon/teachable_learnsets.h` (the real per-species TM/tutor compatibility table,
     gitignored, built at compile time — `P_LEARNSET_HELPER_TEACHABLE TRUE` in
     `include/config/pokemon.h`). So for **TM/tutor moves specifically**, `all_learnables.json`
     ∩ the real TM/tutor lists IS the ground truth — not something to dismiss.
   - Also check `.teachingType` on the species in `species_info/*.h`: if `TM_ILLITERATE`, the
     universal-moves bonus (see below) is skipped, but the species' own `all_learnables.json`
     TM moves are still granted as long as `P_TM_LITERACY` is `GEN_LATEST`/`> GEN_6` (it is, in
     this repo: `include/config/pokemon.h:57`) — so `TM_ILLITERATE` in practice does NOT block
     Rest/Sleep Talk/etc. here. If `ALL_TEACHABLES`, the species gets every TM/tutor move minus
     `signatureTeachables`.
   - A move NOT present at all in `include/constants/tms_hms.h`'s `FOREACH_TM` list cannot be a
     TM for ANY species, full stop (no need to check all_learnables.json for it). Confirmed NOT
     real TMs: Confuse Ray, Foul Play, Strength Sap, Leech Life, Seismic Toss, Recover, Ice/Fire/
     Thunder Punch, Knock Off.
   - Confirmed real TMs people got wrong earlier in this session: **Thunder Wave, Drain Punch,
     Sleep Talk, Roost, U-Turn, Swords Dance, Rest, Toxic, Earthquake.**
4. Universal moves (granted to basically everyone, from `special_movesets.json`'s
   `universalMoves`): Bide, Frustration, Hidden Power, Mimic, Natural Gift, Rage, Return,
   Secret Power, Substitute, Tera Blast. `extraTutors` in that file is empty `[]` in this repo.

### Quick re-usable check (python)
```python
import json
with open('src/data/pokemon/all_learnables.json') as f:
    data = json.load(f)
import re
tm_text = open('include/constants/tms_hms.h').read()
tm_list = set(re.findall(r'(?<=F\()[A-Z_]+(?=\))', tm_text))
# data['SPECIESNAME'] is a list of 'MOVE_X' strings (the species' full potential pool)
# a move is TM-teachable to that species iff: move.replace('MOVE_','') in tm_list AND ('MOVE_'+move) in data[species]
```

## Corrected facts discovered during the TM re-audit (species discussed this session)

Rest + Sleep Talk are TM-obtainable for essentially every species discussed, since `REST` and
`SLEEP_TALK` are both in `all_learnables.json` for all of them (checked explicitly): Dragonite,
Mewtwo, Muk, Grimer, Starmie, Staryu, Chandelure/Lampent/Litwick, Ampharos, Vileplume, Bellossom,
Gyarados, the whole Eeveelution line, Kangaskhan, Quagsire, Scizor, Breloom, Ludicolo, Meganium,
Typhlosion, Feraligatr, Clefable, Clodsire, Dondozo, Pyukumuku (confirmed despite its
`TM_ILLITERATE` tag — doesn't matter per rule above), Garganacl, Regirock/Regice/Registeel, Xatu/
Natu, Hatenna/Hattrem/Hatterene, Metagross, Poliwrath, Jumpluff, Tangrowth/Tangela, Chansey/
Blissey, Crobat.

Specific corrections to earlier (wrong) answers in this session:
- **Metagross**: CAN learn Rest + Sleep Talk via TM (previously wrongly said impossible).
- **Scizor**: CAN learn Rest + Roost + Sleep Talk + U-Turn via TM (previously wrongly said no Roost).
- **Poliwrath**: CAN learn Drain Punch via TM (previously said no, based on level-up/egg only check — TM was missed).
- **Clefable**: CAN learn Thunder Wave + Drain Punch + Sleep Talk via TM (opens up a real Unaware/Magic
  Guard-style wall build with Rest+SleepTalk, not just Moonblast spam).
- **Regirock/Regice/Registeel/Garganacl**: CAN learn Sleep Talk + Rest via TM — this reopens the
  "Clear-Body-family RestTalk wall" question that was earlier answered "not found," since the TM
  check was missing at the time.
- Still correctly unavailable (confirmed NOT real TMs, applies to every species): Confuse Ray,
  Foul Play, Strength Sap, Leech Life, Seismic Toss, Recover.

Moves still only obtainable via level-up/egg (not TM) when claimed for a species — e.g. Confuse
Ray on Natu/Xatu, Foul Play's absence on Umbreon, Strength Sap on the Hoppip line, Seismic Toss on
Chansey — remain correctly verified as before; only the TM-based "not available" claims were wrong.
