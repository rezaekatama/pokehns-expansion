# Move/TM verification protocol for this repo (pokehns-expansion)

Standing rule from the user: **never use knowledge from outside this repo.** Every claim about
base stats, abilities, move effects, or move availability must be verified against this repo's
own source code.

## How to check if a species can learn a move — correct order

1. **Level-up learnset**: `src/data/pokemon/level_up_learnsets/gen_<N>.h` (array `s<Species>LevelUpLearnset`).
   **`gen_9.h` is NOT automatically the right file — check `P_LVL_UP_LEARNSETS` in
   `include/config/pokemon.h` first.** `src/pokemon.c:2867-2885` `#include`s exactly ONE of
   `gen_1.h` through `gen_9.h` based on that config value (highest matching `#elif`). In this repo
   `P_LVL_UP_LEARNSETS` is **`GEN_7`** (`include/config/pokemon.h:14`), so the real, compiled
   learnset file is **`level_up_learnsets/gen_7.h`**, not `gen_9.h`. This was a standing error for
   this entire session before it was caught (triggered by the user providing a screenshot of their
   own in-game Clefable with Seismic Toss, which neither `gen_9.h` nor `gen_7.h` actually explained
   — see point 3a below for the real explanation — but re-deriving it is what surfaced this
   separate, real bug). Re-check any earlier level-up-learnset claim made against `gen_9.h` if it
   matters for the current question; `gen_7.h` and `gen_9.h` can differ (Clefable's own learnsets
   differ move-for-move between the two files).
2. **Egg move**: `src/data/pokemon/egg_moves.h` (array `s<Species>EggMoveLearnset`). Check the
   species AND every pre-evolution in its line (egg moves are inherited through evolution). This
   file is NOT gen-conditional (single file, always compiled), unlike level-up learnsets.
2a. **Pre-evolution LEVEL-UP moves do NOT carry the same way — check `P_PRE_EVO_MOVES` before ever
   claiming a post-evolution species can relearn a move its pre-evolution had.** Discovered via
   the Tyrogue → Hitmontop / Fake Out case: Fake Out is NOT in `sHitmontopLevelUpLearnset` (nor in
   Tyrogue's own egg-move list, nor a TM, nor taught by any Move Tutor NPC), but IS a native
   level-up move for **Tyrogue itself** at level 1 (`sTyrogueLevelUpLearnset`,
   `level_up_learnsets/gen_7.h:3170-3179`). The Move Relearner's own code
   (`src/move_relearner.c`, `HasRelearnerLevelUpMoves`, lines 1330-1357) walks back through
   pre-evolutions to check their level-up learnsets too — but ONLY `if (P_PRE_EVO_MOVES)` (line
   1352: `species = (P_PRE_EVO_MOVES ? GetSpeciesPreEvolution(species) : SPECIES_NONE);`), and in
   this repo **`P_PRE_EVO_MOVES` is `FALSE`** (`include/config/summary_screen.h:17`). So the
   Move Relearner here flatly refuses to look at a pre-evolution's level-up moves — there is no
   way to teach Hitmontop Fake Out after it has already evolved from Tyrogue. The ONLY way to get
   it onto Hitmontop is to make sure the move is still in the moveset *while the mon is still
   Tyrogue* (it's native at level 1, so usually already there unless overwritten) and then evolve
   it — moves already known are preserved through evolution, this is just not something the
   Relearner can restore afterward. This generalizes: any move that is a pre-evolution's own
   level-up move but NOT in the final-stage species' own level-up/egg/TM/tutor list is a
   "keep-it-before-you-evolve-or-lose-it-forever" move in this repo, not a "relearn it anytime"
   move — always check `P_PRE_EVO_MOVES` before asserting otherwise.
3. **Move Tutor (NPC) — a 4th acquisition path, separate from TM, distinct from both
   `all_learnables.json`-filtered TM/tutor compatibility AND `special_movesets.json`'s
   `extraTutors`.** This was missed for this whole session until the same Clefable/Seismic Toss
   case forced it: Seismic Toss is not a TM, not in Clefable's level-up learnset in EITHER
   `gen_7.h` or `gen_9.h`, and not an egg move — it's taught by an actual Move Tutor NPC at
   **Battle Frontier Lounge 7 (`data/maps/BattleFrontier_Lounge7_hns/scripts.inc`)** for 24 Battle
   Points, reachable via `BattleFrontier_OutsideWest_hns` ← `BattleFrontier_OutsideEast_hns`. These
   NPCs use the `ChooseMonForMoveTutor` special and set `VAR_0x8005` to the move constant; grep
   `grep -rn "MOVE_<NAME>" data/maps/*/scripts.inc` and look for that pattern nearby, independent
   of `all_tutors.json`/`extraTutors` (those feed the auto-generated TM-relearner pool, which is a
   different, narrower mechanism — a Move Tutor NPC existing in a map script is reachable even if
   it never shows up in that generated pool). Prefer an `_hns`-suffixed tutor map per the existing
   map-set caveat below; `move_tutors_frlg.inc` has the same Seismic Toss tutor for the
   disconnected Frlg world (`PewterCity_Museum_1F`) — don't cite that one as reachable.
5. **TM — DO NOT just grep `items.h` for `[ITEM_TM_<MOVE>]`.** That only finds the
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
     TM for ANY species, full stop (no need to check all_learnables.json for it) — but it can
     still be a Move Tutor move (point 3 above) or a level-up/egg move, so "not a TM" is never the
     end of the check. Confirmed NOT real TMs: Confuse Ray, Foul Play, Strength Sap, Leech Life,
     Seismic Toss (→ Move Tutor instead, see point 3), Recover, Ice/Fire/Thunder Punch, Knock Off.
   - Confirmed real TMs people got wrong earlier in this session: **Thunder Wave, Drain Punch,
     Sleep Talk, Roost, U-Turn, Swords Dance, Rest, Toxic, Earthquake, Protect, Substitute,
     Aerial Ace.**
6. Universal moves (granted to basically everyone, from `special_movesets.json`'s
   `universalMoves`): Bide, Frustration, Hidden Power, Mimic, Natural Gift, Rage, Return,
   Secret Power, Substitute, Tera Blast. `extraTutors` in that file is empty `[]` in this repo —
   that field is NOT the same thing as the map-script Move Tutors in point 3; don't treat it as
   covering them.

## Map/region caveat (learned the hard way on the Snorlax/Pokémon Tower question)

This repo contains THREE map sets: `_hns`-suffixed maps (the real, connected Johto+Kanto world
this hack is actually built around — confirmed starter town `NewBarkTown_Lab_hns`), `_Frlg`-suffixed
maps (leftover vanilla FireRed/LeafGreen Kanto content, confirmed NOT connected to the `_hns` world
— e.g. `PokemonTower_7F_Frlg` has zero incoming connection from `LavenderTown_hns`), and plain
unsuffixed maps (base pokeemerald Hoenn content, e.g. `DewfordTown_Gym`, `MauvilleCity`,
`SlateportCity`) whose connection to the `_hns` world is **unconfirmed/likely absent** — a
repo-wide warp search found only one single edge (`BirthIsland_Harbor_hns` → `MAP_LILYCOVE_CITY_HARBOR`,
almost certainly a Mystery-Gift-only event island, not a real route). **Never cite a plain or
`_Frlg` map as something the player can actually reach without first checking it connects back
to an `_hns` map** — prefer the `_hns` source for anything (TM location, item, NPC, Move Tutor)
when one exists, and flag it explicitly when it doesn't (e.g. TM32 Double Team's only known
locations, Mauville Game Corner and Route113, are both non-hns and unconfirmed-reachable).

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
Blissey, Crobat, Togekiss.

Specific corrections to earlier (wrong) answers in this session:
- **Metagross**: CAN learn Rest + Sleep Talk via TM (previously wrongly said impossible).
- **Scizor**: CAN learn Rest + Roost + Sleep Talk + U-Turn via TM (previously wrongly said no Roost).
- **Poliwrath**: CAN learn Drain Punch via TM (previously said no, based on level-up/egg only check — TM was missed).
- **Clefable**: CAN learn Thunder Wave + Drain Punch + Sleep Talk via TM (opens up a real Unaware/Magic
  Guard-style wall build with Rest+SleepTalk, not just Moonblast spam).
- **Regirock/Regice/Registeel/Garganacl**: CAN learn Sleep Talk + Rest via TM — this reopens the
  "Clear-Body-family RestTalk wall" question that was earlier answered "not found," since the TM
  check was missing at the time.
- **Togepi/Togetic/Togekiss**: CAN learn Swagger via TM (confuses target but also raises its Attack
  +2 — a real risk, weigh against Sweet Kiss, which the line gets as a native level-up move at lv9
  on Togepi/Togetic only, pure confuse with no stat side-effect, but 75% accuracy vs Swagger's 85%).
- Still correctly unavailable (confirmed NOT real TMs, applies to every species): Confuse Ray,
  Foul Play, Strength Sap, Leech Life, Seismic Toss, Recover.

Moves still only obtainable via level-up/egg (not TM) when claimed for a species — e.g. Confuse
Ray on Natu/Xatu, Foul Play's absence on Umbreon, Strength Sap on the Hoppip line, Seismic Toss on
Chansey — remain correctly verified as before; only the TM-based "not available" claims were wrong.

## Battle engine mechanics verified in this repo's source (cite exact file:line when reusing)

- **Status exclusivity**: `STATUS1_ANY` means only one major status (poison/toxic/burn/paralysis/
  sleep/freeze/frostbite) can be on a battler at once — e.g. Thunder Wave and Toxic can never
  coexist on the same target. Confusion and flinch are NOT status1 (they're volatile conditions),
  so paralysis + confusion, or paralysis + confusion + flinch, CAN all stack on the same target
  simultaneously.
- **Canceler pipeline & ordering matters** (`src/battle_move_resolution.c`, array
  `sMoveSuccessOrderCancelers` at lines 1989-2036, loop `DoAttackCanceler` at 2038-2059): every
  turn, a battler's ability to act is checked against ordered gates; the loop stops at the FIRST
  gate that returns `CANCELER_RESULT_FAILURE`, so a gate's side effects (like decrementing a
  status counter) never run if an earlier gate in the array already failed that turn. Order in
  this repo: `CANCELER_ASLEEP_OR_FROZEN` (very early) → ... → `CANCELER_FLINCH` → ... →
  `CANCELER_CONFUSED` → `CANCELER_GHOST` → `CANCELER_PARALYZED` → ...
  - **Sleep counter** (`CancelerAsleepOrFrozen`, line 108-183) is checked/decremented BEFORE
    flinch — so sleep's turn-counter always ticks down regardless of flinch that turn. `toSub = 1`
    normally, `toSub = 2` with Early Bird (wakes up twice as fast).
  - **Confusion counter** (`CancelerConfused`, line 375-415) is checked/decremented AFTER flinch —
    so if a battler flinches, its confusion counter does NOT decrement that turn (confusion
    duration effectively freezes/extends when paired with flinch, it does not get cured early).
  - **Paralysis has no counter at all** — it's a fresh `RandomPercentage(RNG_PARALYSIS, 75)` roll
    every turn (`CancelerParalyzed`, line 431-442; 75% act, 25% full paralysis), with no stored
    duration to decrement, freeze, or cancel. Confusion and paralysis do NOT "cancel" each other's
    counters; they just run in that fixed order, and if confusion's self-hit fires first (50%
    chance in this repo, see below), the paralysis roll for that turn simply never happens — it's
    not "blocked," it's never reached.
  - Confusion self-hit chance: `GetConfig(B_CONFUSION_SELF_DMG_CHANCE)` is `GEN_3` in this repo
    (`include/config/battle.h`), giving **50% self-hit**, NOT the modern Gen7+ 33%.
    `B_CONFUSION_TURNS = 5` → duration is `RandomUniform(2, 5)` turns when first inflicted.
  - Full paralysis chance: 25% (`RandomPercentage(RNG_PARALYSIS, 75)` → 75% act normally).
  - Rest sets sleep to a FIXED 3 turns (`STATUS1_SLEEP_TURN(3)`, `battle_script_commands.c:7508`),
    not random — unlike natural sleep-inducing moves (Spore etc.) which use
    `RandomUniform(2, 4/5/8)` depending on the move.
  - While asleep, ONLY `EFFECT_SNORE` and `EFFECT_SLEEP_TALK` moves can execute
    (`CancelerAsleepOrFrozen` lines 140-148); any other chosen move auto-fails with
    `BattleScript_MoveUsedIsAsleep`. Uproar (`EFFECT_UPROAR`) is NOT one of these and additionally
    actively prevents sleep for the whole field (`UproarWakeUpCheck`,
    `battle_script_commands.c:7515-7534`) — never pair Rest with Uproar on the same set.
  - **King's Rock** (`TryKingsRock`, `src/battle_hold_effects.c:197-221`) explicitly returns
    NO_EFFECT if the move used already has its own `MOVE_EFFECT_FLINCH` (line 204:
    `MoveHasAdditionalEffect(gCurrentMove, MOVE_EFFECT_FLINCH)`) — so King's Rock does NOT stack a
    second flinch roll on moves like Air Slash or Snore that already flinch on their own; it only
    adds value on moves with no inherent flinch chance. Serene Grace doubles a move's OWN
    secondary-effect chance (`B_SERENE_GRACE_BOOST`) and also doubles King's Rock's chance via the
    same code path (line 210-211) when both are present, but that's moot on a move that already
    disqualifies King's Rock via line 204.
  - **Snore** (`moves_info.h:4724-4744`): can only be used while asleep, 30% base flinch
    (`MOVE_EFFECT_FLINCH` chance 30, doubles to 60% with Serene Grace) — the correct move to pair
    with Rest for a "still useful while asleep" flinch-lock set, NOT Uproar.
  - **Uproar locks the user into repeating it** for 3 turns in this repo
    (`B_UPROAR_TURNS = GEN_LATEST` → `B_UPROAR_TURN_COUNT(5) - 2 = 3`,
    `battle_script_commands.c:2584-2591`, sets `volatiles.multipleTurns = TRUE` and
    `gLockedMoves[gEffectBattler] = gCurrentMove`), the same mechanism as Thrash/Outrage/Petal
    Dance — so once used, Rest/other moves literally cannot be selected until the lock ends, on
    top of Uproar's own sleep-prevention effect.

## Source files/locations established as reliable for recurring lookups

- Species base stats/types/abilities: `src/data/pokemon/species_info/*.h` (split by family/gen).
- Move data/effects: `src/data/moves_info.h`.
- Move Tutor NPC scripts: `data/maps/*_hns/scripts.inc`, search for `ChooseMonForMoveTutor`.
- Gym Leader rematches: NOT at the original gym. All 16 (Johto: Falkner/Bugsy/Whitney/Morty/Chuck/
  Jasmine/Pryce/Clair; Kanto: Brock/Misty/Surge/Erika/Sabrina/Blaine/Janine/Blue) stand as object
  events inside the single room `SaffronCity_FightingDojoVIP_hns` (reached via the normal
  `SaffronCity_FightingDojo_hns`), gated by `FLAG_IS_KANTO_CHAMPION` for the first rematch tier and
  additionally `VAR_SINJOH_STORYLINE >= 13` OR `VAR_ALOLA_STATE >= 4` OR `FLAG_USED_TELEPORTER` for
  a stronger `_PostOBC` tier. Each leader's sprite is individually gated by its own
  `FLAG_HIDE_DOJO_<NAME>` flag (see each leader's entry in
  `SaffronCity_FightingDojoVIP_hns/map.json`'s `object_events`), which is only cleared by finding
  and talking to a one-time cameo NPC for that leader scattered elsewhere on the overworld
  (`grep -rn "clearflag FLAG_HIDE_DOJO_" data/maps/*/scripts.inc` to find them) — NOT by revisiting
  their original gym (revisiting the gym after the badge fight is just flavor text, no rematch).
  Two leaders (Chuck, Blue) are the exception: their cameo trigger script is inside their own gym
  map, but it's a separate post-badge dialogue (`*_GotoDojo`), not the original badge battle.
