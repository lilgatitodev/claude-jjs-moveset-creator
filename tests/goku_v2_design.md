# Goku v2 — Design Brief (round 2)

Re-build of round-1 Goku (scored 7/10). This brief documents how each
piece of round-1 playtest feedback is addressed slot-by-slot. No
AWAKENING slot is shipped this round — the round-1 awakening "read as
empty" bug from §8.1 is unresolved, so per §10.7 minimal-moveset rule
we skip the slot entirely. The player keeps the default awakening.

Slot layout: 1 CHASE + 4 MELEE + 1 SPECIAL + 4 SKILL.

Palette: Saiyan gold/orange (`255, 235, 120` → `255, 200, 80` →
`255, 100, 50`) for Ki/aura, cyan/white (`120, 200, 255` →
`200, 240, 255`) for Kamehameha, light green (`180, 255, 200`) for
Spirit Bomb. Same audio set as round-1.

## CHASE — Shunkan Idou (Instant Transmission flash-step)

Round-1: chase barely moved (FORCE 55 was OK in code but felt short).
Fix: bump to FORCE `0, 0, 70` over `TIME 0.1` (above the §10.7 floor
of 50). Add a terminal m1-like hit at the end of the dash per §10.7
"Optional but nice: a small m1-like hit at the end of the chase" —
HITBOX 5/5/5, DAMAGE 4, STUN 0.3, DEBREE 4, BRANCH TARGET hitFlash,
followed by small target VELO with `LAST HIT: 1`.

## MELEE m1–m3 — keep SIMPLE

Round-1: "m2 animation is weird, dont try too hard." Fix per §10.7:
m1 and m2 reuse the SAME `ANIM_USE [2, 7]` pair (right-arm swing
animation); m3 uses `[2, 8]` (matching pair) for variety. No
bespoke per-m animation hunting. Each m1–m3 is ≤5 nodes: ANIM →
Melee Trail → WAIT → HITBOX → SFX → small forward user-VELO.
Every HITBOX now has `DEBREE: 4` (round-1 had 0).

## MELEE m4 — downslam + uppercut (§10.7 mini-recipe verbatim)

Round-1: "downslam and uppercut don't work" + "m4 flings the user, not
the target". Fix: implement the §10.7 m4 mini-recipe exactly. Two
BRANCH blocks in the parent Line pointing at `uppercut` (`AIR FLIP:
true`) and `downslam` (`AIR FLIP: false`). Each variant ends with a
target-VELO with `LAST HIT: 1` MANDATORY, RAGDOLL 0.6, DEBREE 4 on
each hitbox. Both have a closing finisher beat (Clash + Ring + Shake)
via a target-side `hitFx` branch.

## SPECIAL — Kamehameha (with proper impact feedback)

Round-1: "projectile was invincible — no animation on target, no
ragdoll." Fix per §10.7 "Make projectiles HIT": change `CONTINUE:
true` (round-1) → `CONTINUE: false` so the beam dissipates on first
contact. BRANCH TARGET `beamHit` fires on the target: target-side
Glow + Clash + SFX + target VELO with `RAGDOLL: 0.8, LAST HIT: 1`.
Bump beam DEBREE to 8 (gigantic). Add a closing finisher beat
(Shake Heavy + Wind Expand + Ring) per §10.7 "Finishers".

## SKILL 1 — Ki Blast Volley (sound effects per shot)

Round-1: "would be good with more sound effects." Fix per §10.7
"SFX abundance": one SFX per projectile (3 projectiles → 3 SFX with
varied SPEEDs 0.95/1.0/1.05). Each projectile's BRANCH TARGET fires
a Burst + Glow + small target VELO with `LAST HIT: 1` so hits read.

## SKILL 2 — Spirit Bomb (visuals + ragdoll on hit)

Round-1: "lacking sound effects, ragdolls or more visuals on hit
targets." Fix: charge phase gains a second SFX during the build-up
pulse. `bombHit` branch now contains: target-side VELO with
`RAGDOLL: 1.0, LAST HIT: 1`, target-side Clash + Black Flash +
extra Ring, plus a closing Shake Heavy + Wind Expand finisher beat
on the user side. Hitbox-projectile DEBREE bumped to 8.

## SKILL 3 — Dragon Fist (kept; bumps to per §10.7)

Round-1 praised. Keep structure; bump DEBREE 1 → 6 on the main
hitbox per §10.7 ("big finisher = 6"). Closing beat already present.

## SKILL 4 — Meteor Combo (SFX per hit + finisher beat)

Round-1: "sound effect for each hit?" Fix: each iteration of the
loop fires its own SFX paired with the HITBOX. Final finisher HITBOX
gets DEBREE: 6 plus a closing Clash + Ring + Shake Heavy + Burst
cluster (already partially present — amplify).

## Assets

All asset IDs are reused verbatim from the user's round-1 validated
moveset. No scraping; no new SFX/texture/mesh IDs.
