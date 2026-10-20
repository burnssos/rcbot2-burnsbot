# RCBot2 – Day of Defeat: Source improvements

Based on APGRoboCop/rcbot2 `master` (commit afa2d4e3, Sept 2026), which already
contains ~11 DoD:S fixes made after the v2.0-alpha8 release. Licence: AGPLv3
(unchanged). All original copyright and licence headers are kept.

## Build fixes (affected every game, not just DoD:S)

- `bot_fortress.h`: the base declaration of `CBotFortress::chooseClass()` had
  been commented out while `CBotFF` still overrides it and `bot_fortress.cpp`
  still defines it, so upstream `master` did not compile.
- `bot_plugin_meta.h`: declared `m_iFFHudLuaMsg` / `m_pPendingHudLuaBuf`, which
  `bot_plugin_meta.cpp` uses but no header declared.
- `AMBuildScript`: `-Wno-reorder-ctor` for clang, so the stock hl2sdk headers
  build under `-Werror` with clang 18.

## DoD:S bot fixes

### Classes
- **Class numbering was off by one.** A past change made the internal class
  list start at 1, but the game numbers classes from 0. Every class check was
  shifted: sniper logic fired for machine gunners, MG logic for rocket
  soldiers, `rcbot_force_class 6` (rocket) did nothing, and the "spread classes
  across the team" weighting applied to the wrong classes.
- `rcbot_force_class` now also switches bots that are already playing, and its
  help text lists the DoD:S values (1 rifleman … 6 rocket). Values outside 1–6
  no longer leave a bot stuck unable to pick a class.
- Class re-evaluation after death no longer divides by zero when nobody has
  scored yet.

### Combat awareness
- **Axis bots didn't react to enemy grenades or rockets.** The enemy team was
  only set correctly for bots that had once been on Allies.
- Smoke: a bot decided once whether it could see through a smoke cloud and
  then never re-checked while the smoke lasted. It now re-checks every
  0.15–0.3 s as intended.
- After a fight, the idle reload no longer wipes the "last saw an enemy" time,
  which had cut the follow-up grenade window from 10 s to 5 s.
- Grenade throws now happen even when no cover waypoint is nearby (previously
  the bot reported success and did nothing).
- Axis bots now also call "use grenade" on snipers/MGs (the check only worked
  for US grenades).

### Objectives — flags
- **Squad members never attacked or defended the flag they were at.** The
  objective lookup used the member's slot number as a map position, cleared
  the bot's nearest flag every think, and could read outside an array.
- **Two-capper flags (usually the middle flag):** teammates used to answer
  "Need backup" only if they could already see both the caller and the flag.
  They now work out which flag the caller is on, check it still needs more
  players, and come if they can see the caller or are within ~1500 units.
  Bots that are fighting, already capturing or on a bomb run ignore the call,
  and each listener goes with a probability that scales with how many players
  are still missing (35% each per missing player, doubled if they can see the
  caller), so the rest of the team keeps its own jobs. The waiting capper calls
  at most every 7–10 s instead of every 2–4 s.
- The attack-vs-defend decision table was always read at entry [0][0] because
  of integer division, so the score never influenced it. **Playtest this:** the
  table now works as designed, so a team that has lost most flags leans
  towards defending what it has (about 20% attack chance at 1/5 flags) where
  before every bot always used the "attack" value.
- On plain flag maps the "last flag" / "last owned flag" priority boost read
  stale values that were only set on bomb maps.
- Control points keep the game's numbering. If a point in the middle had no
  flag entity, the highest-numbered points were dropped and never contested.
  Unused slots are skipped when counting flags or finding the nearest one.
- Mixed maps (some flags, some bomb points) are now recognised as both, so
  bots capture the plain flags as well as planting.
- Random flag selection can no longer return a point that failed its checks.

### Objectives — bombs
- **Bots never acted on "plant/defuse the nearest bomb" or "defuse a planted
  bomb" decisions.** These cases share code by falling through to the
  schedule builder; `break`s added later made them return "failed" every time.
- Path bombs (bomb targets not tied to a flag) have no control point id; that
  id (-1) was used to index the flag arrays. All id-based helpers now
  bounds-check, and path bombs are checked by position.
- Two bots reaching the same planted bomb each saw the other holding +use,
  assumed it was defusing, and both left. Only players the game reports as
  defusing now count.
- A bomb task that failed could be marked complete later in the same frame.
- Bots no longer pick an already armed or exploded bomb target to plant on.
- "Plant the nearest bomb" now requires actually carrying a bomb, and "defuse
  the nearest bomb" goes to the armed target when a point has two.
- On mixed maps, bots only fetch a bomb when their team has something to plant.
- Teammate-defusing checks look at both targets of a point and are
  bounds-checked.
- Utility scores for a team with no bomb targets of its own were NaN
  (divide by zero) and could scramble the bot's priorities.
- "Defend point" on bomb maps checked flag id -1 instead of the visible flag.

### Snipers and machine gunners
- Sniper/MG posts now end once the planned hold time (20–40 s) has passed and
  no enemy has been seen for 10 s. The old condition could never be true, so
  they camped until out of ammo.

### Robustness
- Null-pointer guards: player leaving during a "stick together" call, missing
  client data when chasing the last enemy, grenade with no weapon data.
- Zero-length vector normalisation guarded (MG move-up, smoke geometry).
- Drop-ammo task gives up after 10 s instead of chasing forever.
- `CBotTask::timedOut()` comparison corrected (was true before the deadline;
  currently unused, so no behaviour change).

## Known issues not changed
- `CBotHL2DMSnipe` (HL2:DM, not DoD:S) has the same never-ending sniper
  condition. Left alone because it can't be tested here.
- The 32-bit (`bin/rcbot.2.dods.so`) build was not produced: this build
  machine has no 32-bit libraries. DoD:S servers have been 64-bit since 2025.
- Nothing here has been run on a live server yet; it compiles cleanly and was
  reviewed, but needs in-game testing.
