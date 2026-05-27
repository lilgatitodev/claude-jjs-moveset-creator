# Cinderwitch — Build Summary

**Files:**
- Design brief: `tests/cinderwitch_design.md`
- Moveset JSON: `tests/cinderwitch_moveset.json`
- Encoded import code: `tests/cinderwitch_moveset.b64` (3428 chars)
- Round-trip: decoded JSON matches source exactly (`EQUAL: True`,
  10 slots, names preserved).

## Slot layout (10 slots)

| K_NAME | NAME | KEY/COOLDOWN | One-line |
|---|---|---|---|
| CHASE | Ember Glide | 4s CD | Fire-skate forward (55 studs/0.1s) + small terminal tag hit. |
| MELEE | m1 | — | Standard punch + ember melee-trail (right arm). |
| MELEE | m2 | — | Mirror of m1 on the left arm. |
| MELEE | m3 | — | Slightly wider sweep with a whirl-slash visual. |
| MELEE | m4 | — | Conditional finisher: uppercut (airborne target) or downslam (grounded), each ragdoll-launches via VELO LAST HIT 1. |
| SPECIAL | Witch's Pyre | 18s CD | 3-hit forward Explosion cone with a 0.55s windup + final ragdoll-send finisher beat. |
| SKILL | Firebrand | KEY 1, 12s CD | Close-mid lunge punisher, 14 dmg, target ragdoll-launch. |
| SKILL | Firewall | KEY 2, 15s CD | Wide stationary wall hitbox (15x8x2), 12 dmg, knocks pressure off her. Defensive read. |
| SKILL | Ember Burst | KEY 3, 10s CD | Radial Explosion AoE centered on user, 10 dmg, panic-button. |
| SKILL | Cinder Step | KEY 4, 8s CD | IFrame blink forward 25 studs, no hitbox. Escape/engage. |

No AWAKENING slot (intentionally skipped — see below).
No awakened-form SKILL slots (no awakening).

## §10.7 etiquette rules applied

- **CHASE dashes meaningfully** — `FORCE: "0, 0, 55"` over `TIME: 0.1`
  matches the documented "base-game forward dash" baseline; above the
  §10.7 minimum of 50. *Not* the 28-stud weak chase that was flagged in
  round-1.
- **CHASE has the optional terminal hit** — small `HITBOX` (DAMAGE 4,
  STUN 0.3) + target-side VELO with `LAST HIT: 1` so a player who
  dashes into someone catches them.
- **m1–m3 are SIMPLE** — 5–6 nodes each, all reuse `ANIM_USE [3, 3]`
  with slight SPEED variation, vary only the trail body part. No
  bespoke per-m1 design exercise.
- **m4 has both variants** — uppercut for `AIR FLIP true` (airborne)
  and downslam for `AIR FLIP false` (grounded), routed via a `m4Hub`
  branch with two `BRANCH` blocks (per the §10.7 m4 mini-recipe).
  Each branch ends with a Shake Heavy + Ring/Wind-Ring finisher beat.
- **VELO LAST HIT: 1 on every target-launching block** — m4 uppercut
  (`+Y, +Z`), m4 downslam (`-Y, +Z`), CHASE terminal tag (`+Y, +Z`),
  Firebrand finisher VELO (`+Y, +Z`). User-side VELO (Cinder Step
  blink, m1/m2/m3 forward shuffle, Firebrand approach-VELO, Pyre's
  final blowback applied to user) correctly stays at default
  `LAST HIT: -1`.
- **DEBREE set on every offensive HITBOX:** 4 on m1/m2/m3/CHASE-tag/
  m4 variants/Firebrand, 6 on Firewall + Pyre 1st & 2nd ticks +
  Ember Burst, 8 on the Pyre finisher hit (gigantic per §5.3).
- **Hit-feedback five-pillar pattern** on every offensive HITBOX/
  PROJECTILE:
  1. Target-side VELO with `LAST HIT: 1` (m4 variants, CHASE-tag,
     Firebrand, Firewall hit-branch, Ember Burst hit-branch, Pyre
     finisher).
  2. `STUN ANIM: true` on every offensive hitbox.
  3. `HIT RAGDOLL: true` on m4 variants, Firebrand, Pyre finisher,
     Ember Burst (the ones where target should be hittable while
     ragdolled).
  4. `BRANCH TARGET` firing a target-side Glow + Clash / Flames
     branch on **every** offensive hit (m1/m2/m3/m4-up/m4-down, CHASE,
     Firebrand, Firewall, Ember Burst, Pyre 1/2/3) — that's the
     visual-on-target trick.
  5. One `SFX` per hitbox, not one per move — including all 3 Pyre
     ticks fire separate SFX with slight `SPEED` variation
     (1.0 / 0.9 / 0.9 — last with a different ID for the finisher
     beat).
- **Projectiles** — none used in this moveset (Cinderwitch's identity
  is pure hand-thrown fire, no beams). The "make projectiles HIT"
  etiquette is satisfied vacuously. If a future variant adds one,
  apply `CONTINUE: false` + `BRANCH TARGET` per §10.7.
- **Finishers on multi-stage moves** — Pyre's 3rd tick is a clear
  finisher beat (Clash 4x + Wind Expand + Ring + Shake Heavy + a
  bigger HITBOX with DEBREE 8 + a USER backstep VELO with RAGDOLL on
  -28 Z); m4 finishes its chain with Shake Heavy + Ring.
- **Visuals lead, hitboxes follow** — every offensive HITBOX is
  preceded by a 0.12–0.55s wind-up (DirectionLock, charging cursed
  energy + sparks + glow, then a `Cancel` VISUAL TAG to drop the
  charge cleanly).
- **Counterability mix** — m1/m2/m3/m4/CHASE/Firebrand/Firewall are
  `Melee` (counterable), Pyre & Ember Burst are `Explosion`
  (uncounterable but with long telegraphs). 8 Melee : 4 Explosion
  spread satisfies "don't make every move Explosion."
- **Cooldown vs damage budget** — Pyre's 8+8+10 = 26 total dmg / 18s
  CD; Firebrand 14/12s; Firewall 12/15s; Ember Burst 10/10s; rough
  match of the heuristic `COOLDOWN ≥ DAMAGE * 1.0`. Pyre is below
  ratio because it's a SPECIAL — which the rule allows ("Specials
  can be longer 15–30s"; the heavy telegraph balances it).
- **Range spread** — m1–m4/CHASE/Firebrand close-range, Firewall
  mid wall, Pyre mid-range cone, Ember Burst point-blank radial,
  Cinder Step mobility. Two ranges minimum satisfied.
- **Defensive coverage** — Firewall (space-control wall), Ember Burst
  (panic AoE), Cinder Step (IFrame blink). Per §10.7 "Counters —
  design space (not a quota)": no `COUNTER` block, but the kit has
  three defensive reads. Cinderwitch's identity is aggressive caster,
  not parry duelist — coherent without a counter.

## Round-1 bugs avoided

| Bug | How avoided here |
|---|---|
| **m4 VELO defaults to USER instead of target** | All target-launching VELO blocks set `LAST HIT: 1` explicitly (m4 up/down, CHASE-tag, Firebrand, Firewall hit, Ember Burst hit, Pyre finisher). User-side displacement VELO correctly left at `-1`. |
| **DEBREE 0 makes hits weightless** | Every offensive hitbox sets DEBREE per §5.3 tier (4 normal, 6 big, 8 gigantic). No `DEBREE: 0` on offensive hitboxes. |
| **One SFX for multi-hit chains** | Pyre fires 3 separate SFX (one per tick); m4 variants each fire their own SFX. |
| **Projectile "feels invincible"** | N/A — no projectiles in this moveset. |
| **Visuals show only on user** | Every BRANCH TARGET branch contains the visual feedback so it renders on the target, not the user. |
| **Over-engineered m1s** | m1/m2/m3 reuse `ANIM_USE [3, 3]` with body-part swaps only; ≤6 nodes each (in §10.7's safe budget). |
| **AWAKENING activation bug** | Slot SKIPPED entirely (allowed per §10.7 "Minimal movesets"). |
| **AWK/AWK2 Prop unverified** | No AWK/AWK2 Props used anywhere. |
| **m4 single-variant** | m4 has both uppercut + downslam variants via the `m4Hub` branch hub. |

## Meshes I need (per §7.5)

**None.** Cinderwitch's identity is built entirely from built-in
`EFFECT` types — no custom mesh required:
- `Cursed Energy`, `Flames`, `Sparks`, `Energy Sparks` — aura/charge.
- `Burst`, `Ring`, `Wind Ring`, `Wind Expand` — impact shockwaves.
- `Clash`, `Glow`, `Shake Medium/Heavy` — hit feedback / camera.
- `Melee Trail`, `Whirl Slash`, `Wedge` — weapon trails / shapes.
- `Visibility` — Cinder Step blink.

If the user later wants a custom mesh (e.g. a witch's broom, a
floating sigil, a cracked-skull crown for an awakening), it can be
slotted as a `Mesh` VISUAL with a placeholder ID.

## Assets used (verify in-game; all reused from prior validated moveset)

All asset IDs in Cinderwitch are re-used from
`tests/original_moveset.json` (user's prior validated reference);
**no external scraping was done.**

| Where in moveset | Type | ID | Source |
|---|---|---|---|
| CHASE Ember Glide (dash SFX) | audio | 120714138513879 | original_moveset reference |
| CHASE terminal tag, m1, m2, m3, Firebrand windup, Firewall windup, Ember Burst windup, Pyre windup | audio | 129465573909487 | original_moveset reference |
| Firebrand impact, Pyre ticks 1+2, Ember Burst impact, m4 variants | audio | 128213617026859 | original_moveset reference |
| Pyre finisher, Firewall impact | audio | 137243377137274 | original_moveset reference |
| Cinder Step blink SFX | audio | 120714138513879 | original_moveset reference |

No `TEXTURE` IDs used (no Overlay/Billboard effects in this moveset).
No `Mesh` IDs used (no Mesh effects in this moveset).

## §10.5 self-review summary

- **A. Velocity:** All FORCE magnitudes in 5–55 range; `|FORCE| > 50`
  only on CHASE (55, by-design). All target-launching VELO have
  `LAST HIT: 1`; all user-self-displacement VELO have `LAST HIT: -1`.
  No `|FORCE| > 1000`.
- **B. Timeline:** every move has `WAIT` between wind-up VISUALs and
  HITBOX. Total durations: m1–m3 ~0.36s, m4 variants ~1.05s, Firebrand
  ~1.4s, Firewall ~1.4s, Ember Burst ~0.95s, Pyre ~2.2s, CHASE ~0.46s,
  Cinder Step ~0.55s. All well under 150s cap. Telegraph windows of
  0.2–0.6s on every ≥10-damage move.
- **C. Hitbox:** sizes deliberate per §2.5 (5x5x5 punches, 15x8x2
  wall, 12x6x12 radial). Positions appropriate (Y -1 for downslam,
  Y +2 for uppercut, Z 4 for forward punches, 0,0,0 for radial).
  No `DAMAGE: 0` no-op hitboxes. `BRANCH TARGET` strings all match
  defined branches. `HIT USER` is false everywhere.
- **D. Branch:** every named branch is reachable (m4Hub → m4Uppercut/
  m4Downslam → m4UpHit/m4DownHit; pyreHit1/2/3 from PyreHIts;
  firebrandHit/firewallHit/emberBurstHit/emberTagHit from their
  respective hitboxes; m1Hit/m2Hit/m3Hit from each MELEE hitbox).
  Every branch's Line terminates cleanly.
- **E. Cross-slot:** cooldown spread 4/8/10/12/15/18 — at least two
  ≤15s (CHASE 4, key3 10, key4 8, key1 12, key2 15). Range spread:
  close/mid/wall/radial/mobility. Defensive reads: Firewall +
  Ember Burst + Cinder Step.

## Known issues / concerns

1. **AIR Req FLIP semantics conflict (wiki vs community guide).**
   §5.20 / §10.7 flag the source conflict. This moveset follows the
   **wiki convention**: m4 uppercut on `AIR FLIP: true` (airborne),
   m4 downslam on `AIR FLIP: false` (grounded). If during playtest
   the uppercut fires while grounded and vice versa, **invert both**
   `FLIP` booleans in the `m4Uppercut` and `m4Downslam` branch Reqs.
2. **Branch fallthrough on failed Req is documented but unverified.**
   The m4 hub relies on the convention that a `BRANCH` block whose
   target's Req fails silently no-ops and execution continues to the
   next block (so `m4Uppercut` tries first; if `AIR FLIP true` fails,
   `m4Downslam` runs). If both branches fire or neither fires, the
   workaround per §5.8 is to use `TAG CHECK` routing instead.
3. **`SINGLE TARGET`-style precision finishers not used.** Cinderwitch
   is AoE-flavored; this is intentional, not an oversight.
4. **No awakening.** Skipped per §10.7 / §8.1. The player keeps any
   default in-game awakening behavior for whichever base character
   this moveset is bound to.
5. **DESIGN sub-agent was not run as a separate context** because no
   Agent/Task tool was available in the harness. The two-phase
   discipline was preserved (design brief was authored in plain
   English first, no JSON, before any block was written), but the
   process separation across two LLM contexts that §7.6.A recommends
   was not possible here. The brief is in `cinderwitch_design.md` for
   the user to audit.
6. **First-use lag warning (§5.24 known bug).** Every custom ability
   shows a first-use delay/lag in JJS. Cinderwitch's first cast of
   each move will be slightly choppy; the second onward will play
   cleanly. This is engine-side, not a moveset bug.
