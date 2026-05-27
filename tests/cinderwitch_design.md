# Cinderwitch — Design Brief

> **Note:** The Agent / Task sub-agent tool was not available in this
> harness, so the DESIGN phase was authored by the main agent in
> "designer hat" mode (plain English only, no JSON, no field values, no
> ANIM_USE pairs) and only *then* re-read with "builder hat" on. The
> two-phase discipline is preserved; only the process separation across
> two distinct LLM contexts was not possible.

Source: Original character (user's brief: "fire-elemental cursed-energy
witch with a tattered cloak, sparks following her movements, aggressive
close-mid range kit"). Design reconstructed from that prompt; no
fandom research applicable.

---

## 1. Character identity

**Cinderwitch** is a fire-elemental cursed-energy spellcaster who fights
in a roaring close-to-mid arc. She wears a tattered cloak that catches
embers and trails ash; orange-and-red cursed energy curls off her hands
in lazy pyres, and bright sparks flick off her every movement. Her
archetype is **aggressive caster** — she closes in fast, drops a
firewall, and burns the opponent down with overlapping cone AoEs and
ember-trailing punches. There is no sword and no projectile beam: every
move is hand-thrown fire, which means she's a constant-pressure brawler
who happens to fight with magic.

- **Vibe:** witch-burning-the-witch-hunters energy. Wicked, predatory,
  in love with her own fire.
- **Color palette:**
  - **Primary fire orange:** `255, 110, 30` — the bulk of every flame VFX.
  - **Hot inner yellow:** `255, 200, 80` — bright cores, sparks, flash
    accents.
  - **Deep ember red:** `200, 40, 20` — outer edges of larger flames,
    blood-fire cursed energy.
  - **Ash gray:** `60, 40, 35` — residual smoke, cloak particles.
- **Signature motif:** a constant low-intensity cinder-spark trail at
  her feet/hands that *never turns off* — so even her idle and walk
  read as "this person is on fire."
- **Sound feel:** cracking embers, low fire whoosh, sharp pops on
  impact. Reuses the safe known-good SFX set.

## 2. Per-slot designs

### CHASE — "Ember Glide"

- **Concept:** Cinderwitch's cloak ignites at the hem and she
  fire-skates forward in a low crouch, trailing an arc of cinders. The
  dash ends with a brushing slap of flame for any enemy she meets at
  the end.
- **Range / archetype:** mobility, point-blank ender hit.
- **Telegraph:** none on purpose — this is a movement primitive, not a
  skill. The flame burst at the end is brief enough not to need a tell.
- **Active:** strong forward dash (~55 studs/0.1s — comfortably above
  the §10.7 minimum 50), terminated by a tiny low-damage hitbox
  (~4 damage, 0.3s stun) that catches an opponent at point-blank.
- **Hit feedback on target:** the dash-tag hitbox plays a small flame
  flash + a tiny target knockback so it feels like she shoulders past
  them as she lands.
- **Resolution:** cinder trail visuals continue ~0.15s after the dash
  cleanly extinguishes.
- **Counterplay:** standard chase dash — no telegraph needed.

### MELEE m1 — "Embered Jab"

- **Concept:** straight punch with a glove of fire — same animation as
  any neutral M1, just trailing a heat shimmer.
- **Range / archetype:** point-blank, light.
- **Telegraph:** standard M1 startup; melee-trail glow on the right arm.
- **Active:** single hitbox, 4 damage, 0.35s stun.
- **Hit feedback:** STUN ANIM on, small ember flash on target via
  BRANCH TARGET, one impact SFX, tiny forward knockback (~8 studs,
  user — *not* target — to keep momentum).
- **Resolution:** small post-hit forward shuffle on the USER (LAST HIT
  -1 — correct for momentum carry).
- **Counterplay:** standard M1.

### MELEE m2 — "Backhand Spark"

- **Concept:** mirror of m1 with the left hand and a sparkier visual
  trail; identical rhythm.
- **Range / archetype:** point-blank, light.
- **Telegraph / Active / Feedback:** same as m1 with BODY PART swapped
  to the Left Arm.

### MELEE m3 — "Cinder Sweep"

- **Concept:** wider arc — Cinderwitch sweeps a horizontal flame
  outward with both hands. Slightly bigger hitbox, slightly more damage.
- **Range / archetype:** close, slightly wider.
- **Telegraph:** brief whirl visual + a melee trail on the right leg.
- **Active:** 5 damage, 0.4s stun, hitbox sized `7, 5, 5` (wider X).
- **Hit feedback:** small ember-flash branch on target, single impact
  SFX, small user forward shuffle.

### MELEE m4 — "Witchbreak Slam" (finisher with airborne/ground variants)

- **Concept:** Cinderwitch raises a hand wreathed in fire, then either
  **slam-dunks** a grounded target (downward overhead) or **uppercuts**
  a launched target with a vertical pillar of flame.
- **Range / archetype:** close, finisher.
- **Telegraph:** raised hand wreathed in growing fire (Cursed Energy +
  Flames cluster on the right arm), audible windup pop.
- **Active:**
  - **Grounded (downslam):** 7 damage, hitbox `6, 5, 5` at
    Y -1; target ragdoll-slammed downward.
  - **Airborne (uppercut):** 7 damage, hitbox `6, 6, 5` at Y +2;
    target ragdoll-launched upward.
- **Hit feedback on target:** STUN ANIM on, HIT RAGDOLL on, target VELO
  with LAST HIT 1 and RAGDOLL 0.6 (downward for slam, upward for
  uppercut), BRANCH TARGET firing a Clash + Ring + Glow flash on the
  target, finisher SFX.
- **Resolution:** Shake Heavy + Wind Ring on user as the finisher
  beat lands.
- **Counterplay:** ~0.2s windup; ATTACK TYPE Melee, BLOCKABLE true.

### SPECIAL — "Witch's Pyre" (right-click, signature move)

- **Concept:** Cinderwitch slams both hands down and erupts a roaring
  **column of fire** directly in front of her, then a second wider
  flame burst sweeps outward. Two-stage SPECIAL with a finisher beat.
- **Range / archetype:** mid-range AoE (~10-stud forward cone),
  multi-hit (3 ticks), strong but very telegraphed.
- **Telegraph:** ~0.6s windup with hands lowering, Cursed Energy
  ramping on both arms, charging SFX layered under it. Plenty of time
  to dodge.
- **Active:** 3 sequential HITBOXes (`Explosion` type, not blockable but
  long telegraph balances it), 8/8/10 damage. One SFX per hit (per
  §10.7).
- **Hit feedback:** each tick fires its own BRANCH TARGET impact flash;
  final hit ragdoll-launches the target backward via LAST HIT 1 +
  RAGDOLL 0.7.
- **Resolution:** Ring + Shake Heavy + Wind Expand cluster on the third
  beat, then a `Cancel` to extinguish the charging Cursed Energy aura.
- **Counterplay:** ~0.6s windup is the parry window. DirectionLock
  during windup so she can't aim-swap. Cooldown 18s.

### SKILL key 1 — "Firebrand" (12s cooldown, fast close-range punisher)

- **Concept:** Cinderwitch lunges forward with a fist of flame — a
  single-hit fast lunge that closes a small gap. Her go-to "you
  whiffed, now eat this" punisher.
- **Range / archetype:** close-mid lunge, 1 strong hit.
- **Telegraph:** short ~0.25s wind-up — arm pulled back, sparks coiling.
- **Active:** 14 damage, hitbox after a small VELO closes ~15 studs.
- **Hit feedback:** STUN ANIM, target VELO with LAST HIT 1 (knockback +
  small ragdoll 0.5), BRANCH TARGET impact flash, single SFX.
- **Resolution:** ~0.4s recovery.
- **Counterplay:** ATTACK TYPE Melee, BLOCKABLE true, 0.25s telegraph.

### SKILL key 2 — "Firewall" (15s cooldown, defensive/space-control)

- **Concept:** Cinderwitch sweeps both hands and a vertical *wall of
  fire* erupts ~3 studs in front of her — a stationary wide-but-short
  hitbox that punishes whoever's standing too close. Functions both
  as space-control and as her *defensive read* — if someone's pressing
  her, she walls them off. (This is the kit's defensive option — the
  rest of the kit is pure aggression, so a "stop hitting me" tool fits.)
- **Range / archetype:** close, AoE wall, no chase.
- **Telegraph:** ~0.3s hand-sweep + bright Cursed Energy ramp.
- **Active:** wide stationary hitbox `15, 8, 2` ~3 studs in front. 12
  damage, 0.5s stun, knocks any caught target backward.
- **Hit feedback:** STUN ANIM, target VELO backward with LAST HIT 1 +
  short ragdoll, BRANCH TARGET impact flash, single SFX.
- **Resolution:** the wall lingers visually ~0.6s after the hitbox so
  it reads as a *zone*, even though only one hit happens.
- **Counterplay:** ATTACK TYPE Melee, BLOCKABLE true; visible Cursed
  Energy charge before the sweep.

### SKILL key 3 — "Ember Burst" (10s cooldown, fast close AoE)

- **Concept:** Cinderwitch explodes outward — a quick radial fire
  shockwave centered on her, immediately useful for crowd-clear and as
  a "get off me" panic button when she's swarmed.
- **Range / archetype:** point-blank radial AoE, 1 hit.
- **Telegraph:** short ~0.2s ember-build crouch.
- **Active:** radial hitbox `12, 6, 12` centered on user. 10 damage,
  0.4s stun.
- **Hit feedback:** STUN ANIM, target VELO outward with LAST HIT 1
  (radial knockback, ragdoll 0.4), BRANCH TARGET flash, single SFX.
- **Resolution:** Ring + Wind Expand on user.
- **Counterplay:** ATTACK TYPE Explosion (uncounterable) but small
  hitbox + obvious telegraph. Short cooldown reflects modest damage.

### SKILL key 4 — "Cinder Step" (8s cooldown, mobility / engage)

- **Concept:** Cinderwitch dissolves into a cloud of embers and
  reappears ~25 studs forward — a teleport-blink. No damage; pure
  mobility. Re-engage or escape.
- **Range / archetype:** mobility, no hitbox.
- **Telegraph:** ~0.15s ember-puff at the start, IFrame during the
  blink.
- **Active:** STATE IFrame 0.3s + STATE NoM1/NoDash 0.3 + VELO forward
  (~25 studs / 0.15s, user-side LAST HIT -1 — correct, this is *user*
  movement).
- **Hit feedback:** N/A (no hitbox).
- **Resolution:** ember-puff on arrival.
- **Counterplay:** no damage, no need.

### AWAKENING

**SKIPPED.** Per §8.1's unresolved "AWAKENING timeline reads as empty"
bug, and per §10.7 "Minimal movesets — not every slot is mandatory"
(skip AWAKENING is explicitly allowed), this moveset ships **without
an AWAKENING slot**. The Build agent will note this in the summary.
Rationale: a broken AWAKENING is worse than no AWAKENING; the unblocked
slot keeps any in-game default behavior.

## 3. Awakening identity

N/A — skipped. If the user later wants one, Pattern B (move + lingering
aura) is the natural fit for Cinderwitch: a transformation surge into a
permanent burning aura with damage multiplier, no per-key skill swap.

## 4. Asset wishlist (meshes I need)

**None.** Cinderwitch's identity is built entirely out of built-in
EFFECTs (`Cursed Energy`, `Flames`, `Sparks`, `Energy Sparks`, `Burst`,
`Ring`, `Wind Expand`, `Wind Ring`, `Glow`, `Clash`, `Shake Heavy`,
`Melee Trail`). No custom 3D mesh is required. SFX/textures are reused
from the user's prior validated moveset (`original_moveset`); no
scraping needed.

## 5. Defensive read / coverage note

Per §10.7 "Counters — design space (not a quota)", the kit is
predominantly offensive. Defensive reads are:

- **Firewall (SKILL 2)** — wall-of-fire space-control / "stop pressing
  me" answer.
- **Ember Burst (SKILL 3)** — radial panic AoE.
- **Cinder Step (SKILL 4)** — IFrame blink escape.

No COUNTER block. The kit reads as "aggressive caster with two
disengage options" — a coherent identity for a pure-aggression mage
without forcing a parry in.
