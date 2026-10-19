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
   - **a)** Is the move in the real TM/HM list? → `include/constants/tms_hms.h`. **That file
     branches on `#if IS_HNS` vs `#else`** — this repo builds under `IS_HNS`, so only the first
     branch is real: **92 TM moves + 8 HM moves (Cut/Fly/Surf/Strength/Flash/Rock Smash/
     Waterfall/Whirlpool) = 100 total**, not 158. (158 was an earlier mistake in this session —
     that number came from grepping the whole file and summing both the `IS_HNS` and `#else`
     branches together, which don't coexist in any actual build.) Extract the correct list with:
     `sed -n '6,98p' include/constants/tms_hms.h | grep -oP '(?<=F\()[A-Z_]+(?=\))'` (TM-only,
     92 moves) — add `sed -n '100,108p' ...` for the 8 HMs if needed.

   **Map/region caveat (learned the hard way on the Snorlax/Pokémon Tower question):** this repo
   contains THREE map sets: `_hns`-suffixed maps (the real, connected Johto+Kanto world this hack
   is actually built around — confirmed starter town `NewBarkTown_Lab_hns`), `_Frlg`-suffixed maps
   (leftover vanilla FireRed/LeafGreen Kanto content, confirmed NOT connected to the `_hns` world —
   e.g. `PokemonTower_7F_Frlg` has zero incoming connection from `LavenderTown_hns`), and plain
   unsuffixed maps (base pokeemerald Hoenn content, e.g. `DewfordTown_Gym`, `MauvilleCity`,
   `SlateportCity`) whose connection to the `_hns` world is **unconfirmed/likely absent** — a
   repo-wide warp search found only one single edge (`BirthIsland_Harbor_hns` → `MAP_LILYCOVE_CITY_HARBOR`,
   almost certainly a Mystery-Gift-only event island, not a real route). **Never cite a plain or
   `_Frlg` map as something the player can actually reach without first checking it connects back
   to an `_hns` map** — prefer the `_hns` source for anything (TM location, item, NPC) when one
   exists, and flag it explicitly when it doesn't (e.g. TM32 Double Team's only known locations,
   Mauville Game Corner and Route113, are both non-hns and unconfirmed-reachable).
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
