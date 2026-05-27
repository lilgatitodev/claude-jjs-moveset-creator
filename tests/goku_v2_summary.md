# Goku v2 — Round-1 Feedback Response

Round-trip verified: `goku_v2_moveset.b64` (4132 chars), 10 slots,
decode(encode(json)) == json. No AWAKENING slot.

## Per-bullet round-1 fix log

| Round-1 complaint | Slot | What changed in v2 |
|---|---|---|
| "chase doesnt really dash forward all that much" | CHASE | `VELO FORCE` raised from `"0, 0, 55"` → `"0, 0, 70"` over `TIME 0.1` (§10.7 floor is 50; bumped well above). |
| "chase normally do a normal m1-like hit at the end" (general) | CHASE | Added terminal `HITBOX` (DAMAGE 4, STUN 0.3, SIZE 5/5/5, DEBREE 4, BRANCH TARGET `chaseHit`), an impact SFX, then a small target-VELO `LAST HIT: 1` so the chase catches enemies. |
| "Special — projectile was invincible, no animation on target, no ragdoll" | SPECIAL Kamehameha | Beam now `CONTINUE: false` (dissipates on first contact) per §10.7. `BRANCH TARGET: beamHit` runs target-side Glow + oversized Clash + Burst + impact SFX **and** target-side `VELO FORCE: "0, 8, 30"` with `RAGDOLL: 0.8, LAST HIT: 1`. Beam DEBREE: 8 (gigantic). |
| "m4 flings the user, not the target" | MELEE m4 | Both `uppercut` and `downslam` end with `VELO ... LAST HIT: 1` (mandatory per §5.2). Round-1 had `LAST HIT: -1` on the uppercut branch — fixed. |
| "downslam and uppercut don't work" | MELEE m4 | Rebuilt verbatim against the §10.7 m4 mini-recipe: parent Line contains two BRANCH blocks pointing at `uppercut` (Req `AIR FLIP: true`) and `downslam` (Req `AIR FLIP: false`). Each branch ends with a `LAST HIT: 1` target-knockback VELO, RAGDOLL 0.6, DEBREE on every hitbox, and an impact SFX. Closing `hitFx` target-side branch fires Glow + Clash + Ring. |
| "m2 animation is weird, dont try too hard, just hit animations" | MELEE m1/m2/m3 | m1 and m2 now reuse the SAME `ANIM_USE [2, 7]` (m2 just slightly faster SPEED 1.25); m3 uses the matching pair `[2, 8]`. Each m1–m3 is exactly 6 nodes (ANIM → Melee Trail → WAIT → HITBOX → SFX → tiny VELO) per §10.7 "≤4–5 nodes long". |
| "Ki Blast Volley would be good with more sound effects" | SKILL 1 Ki Blast Volley | One `SFX` per projectile (3 projectiles → 3 SFX with SPEEDs 0.95/1.0/1.05 for variation). Each projectile's BRANCH TARGET runs Glow + Burst + small target-VELO `LAST HIT: 1` so hits read. Every projectile got `DEBREE: 4`. |
| "Spirit Bomb lacking sound effects, ragdolls or more visuals on hit targets" | SKILL 2 Spirit Bomb | Charge phase gained a second SFX during pulse. `bombHit` branch heavily amplified: Glow + Shake Heavy + Clash (oversized, ALT SIZE 8) + Black Flash + Ring (ALT SIZE 10) + Burst (AMOUNT 30) + impact SFX + target VELO `FORCE: "0, 15, 20"`, `RAGDOLL: 1.0`, `LAST HIT: 1`. Bomb projectile DEBREE: 8. |
| "Meteor Combo — sound effect for each hit?" | SKILL 4 Meteor Combo | LOOP body now includes the SFX (LOOP BACK widened from 3→4 so the loop covers VISUAL + HITBOX + SFX + WAIT — one impact sound per iteration, 6 iterations). Finisher hitbox got DEBREE 6 + final cluster (Burst 25, Clash, Ring, Shake Heavy) + closing impact SFX + target VELO `LAST HIT: 1, RAGDOLL 0.7`. |
| "awakening — empty" | AWAKENING | **Skipped entirely** per §10.7 minimal-moveset rule and §8.1 unresolved AWAKENING-empty-timeline bug. Player keeps default awakening; better than shipping a broken one. |
| "im seeing a lack of debree in hitboxes" (general) | All offensive HITBOX/PROJECTILE | Every offensive hitbox and projectile now has `DEBREE` set: 4 on m1/m2/m3 + chase + small ki blasts (standard), 6 on m4 downslam + Dragon Fist + Meteor Combo finisher (big), 8 on Kamehameha beam + Spirit Bomb (gigantic). Round-1 had DEBREE 0/1 on most. |
| "finishers might be nice" (general) | Kamehameha, Spirit Bomb, Meteor Combo, m4 | Every multi-stage move now has an explicit closing beat per §10.7 "Finishers": Kamehameha → Wind Expand + Ring on dissipation; Spirit Bomb → Shake Heavy + Clash oversized + Black Flash + Ring + Burst cluster on bombHit; Meteor Combo → Burst + Clash + Ring + Shake Heavy + bigger HITBOX with knockback; m4 → hitFx branch with Glow + Clash + Ring. |

## Bugs specifically tested-against (regression checklist)

1. **m4 VELO LAST HIT defaults to user (§5.2):** every m4 branch terminator has `LAST HIT: 1`. Verified by grep.
2. **m4 branch-hub routing (§10.7 mini-recipe):** verbatim two-BRANCH-block parent Line with per-branch `Req.AIR.FLIP` toggles. If the AIR FLIP semantics turn out backward in-game (§5.20 conflict), swap both FLIP booleans and the move still works.
3. **AWAKENING empty-timeline bug (§8.1):** avoided entirely by skipping the slot.
4. **Projectile "invincible" feedback bug (§10.7):** Kamehameha is now `CONTINUE: false` with BRANCH TARGET firing target-side flash, SFX, and ragdoll-VELO. Spirit Bomb same shape. Ki Blast Volley same shape.
5. **DEBREE: 0 on hitboxes:** none remain on offensive hitboxes/projectiles.
6. **Single-SFX-per-multi-hit-chain bug (§10.7):** Meteor Combo loop and Ki Blast Volley each fire one SFX per hit.

## §10 Validation pass

- Top-level is an array, 10 slots.
- Every slot has K_NAME, NAME, DATA.
- SKILL KEYs are integers 1–4 (no letters).
- MELEE slots named m1/m2/m3/m4.
- All BRANCH TARGET strings (`hitFlash`, `hitFx`, `chaseHit`, `beamHit`, `kiHit`, `bombHit`, `dragonHit`) resolve to keys in their slot's `DATA.Branch`.
- No HITBOX has DAMAGE 0 with no stun/branch.
- No LOOP with HOLD: true.
- Every move has startup → active → resolution.
- Every ≥12-dmg move has a counterplay window (Kamehameha 1.6s windup, Spirit Bomb 1.4s+1.1s+1.1s charge, Dragon Fist 0.5s+ telegraph, Meteor Combo 0.3s+ dash startup).
- Codec round-trip passes: confirmed `decode(encode(json)) == json`.

## Assets Used

All asset IDs are re-used verbatim from the user's round-1 validated
Goku moveset — no external scraping was done. The user has already
confirmed these IDs load in-game.

| Where in moveset | Type | ID | Source | Note |
|---|---|---|---|---|
| CHASE Shunkan Idou — windup SFX | audio | `120714138513879` | round-1 Goku (validated) | Re-used; known good |
| CHASE Shunkan Idou — terminal hit SFX | audio | `129465573909487` | round-1 Goku (validated) | Re-used; known good |
| MELEE m1/m2/m3 — hit SFX | audio | `129465573909487` | round-1 Goku (validated) | Re-used; known good |
| MELEE m4 uppercut + downslam — hit SFX | audio | `120714138513879` | round-1 Goku (validated) | Re-used; known good |
| SPECIAL Kamehameha — charge SFX | audio | `128213617026859` | round-1 Goku (validated) | Re-used; known good |
| SPECIAL Kamehameha — fire SFX + beamHit | audio | `129465573909487` | round-1 Goku (validated) | Re-used; known good |
| SKILL 1 Ki Blast Volley — per-shot SFX (×3) | audio | `137243377137274` | round-1 Goku (validated) | Re-used; known good |
| SKILL 2 Spirit Bomb — charge SFX (×2) | audio | `128213617026859` | round-1 Goku (validated) | Re-used; known good |
| SKILL 2 Spirit Bomb — bomb travel + bombHit SFX | audio | `129465573909487` | round-1 Goku (validated) | Re-used; known good |
| SKILL 3 Dragon Fist — dash + impact SFX | audio | `120714138513879` | round-1 Goku (validated) | Re-used; known good |
| SKILL 4 Meteor Combo — per-hit SFX (loop) | audio | `129465573909487` | round-1 Goku (validated) | Re-used; known good |
| SKILL 4 Meteor Combo — finisher SFX | audio | `120714138513879` | round-1 Goku (validated) | Re-used; known good |

No TEXTURE or Mesh asset IDs used anywhere in the moveset.

## Animations used (ANIM_USE pairs; from §7 safe list)

| Slot | ANIM_USE | Reused from |
|---|---|---|
| CHASE | `[1, 5]` | round-1 + §7 safe pair |
| m1 | `[2, 7]` | §7 safe pair |
| m2 | `[2, 7]` | §7 safe pair (simple, per feedback) |
| m3 | `[2, 8]` | §7 safe pair |
| m4 uppercut | `[2, 8]` | §10.7 m4 mini-recipe |
| m4 downslam | `[2, 7]` | §10.7 m4 mini-recipe |
| SPECIAL Kamehameha | `[15, 25]` | §7 safe pair |
| SKILL 1 Ki Blast Volley | `[3, 3]` | §7 safe pair |
| SKILL 2 Spirit Bomb | `[1, 12]` | §7 safe pair |
| SKILL 3 Dragon Fist | `[11, 4]` | §7 safe pair |
| SKILL 4 Meteor Combo | `[17, 2]` | §7 safe pair |
