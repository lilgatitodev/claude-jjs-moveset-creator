# Goku (Dragon Ball Z) — JJS Moveset Design Brief

Source: Dragon Ball Fandom wiki (Kamehameha, Spirit Bomb, Dragon Fist, Instant Transmission, Super Saiyan). Visual palette is canon: Super Saiyan yellow `255, 235, 120`, Kamehameha cyan-white `120, 200, 255`, Spirit Bomb pale green `180, 255, 200`.

Cursed-energy palette borrowed per SKILL.md §5.4 conventions (re-tinted to DBZ canon).

---

**CHASE — Shunkan Idou (Instant Transmission)** | 5s
Concept: a yellow ki-flash forward blink; identity replacement of the base front-dash.
Timeline: 0.00 anim+flash glow → 0.05 VELO Z+55 (matches base dash distance per §2.5) → 0.17 trailing afterimage wind streaks.
Counterplay: short, no hitbox; pure mobility. Cooldown gates spam.
Assets: ANIM `[1,5]` (safe pair). No external SFX.

**MELEE m1/m2/m3** — light cursed-energy strikes
Concept: ki-charged right/left jab and a right-leg kick. All three share rhythm (4/4/5 dmg, ~0.18s wait → hitbox → 12–14 stud forward push).
Timeline: ANIM + arm/leg `Melee Trail` → 0.18s WAIT → 5×5×5 melee HITBOX → small forward VELO. `BRANCH TARGET: hitFlash` paints the wiki-known white-flash trick (`Glow` OPACITY 0.75 → 1, TIME 0.25) on the target via target-side branch.
Counterplay: blockable, low stun, classic m-string.
Assets: ANIM `[2,7]`, `[2,8]`, `[1,9]`.

**MELEE m4 — Finisher (Uppercut / Downslam branch hub)**
Concept: §10.7 m4 convention. The default `Line` is a thin router pointing to either `uppercut` (Req AIR FLIP true) or `downslam` (Req AIR FLIP false). The first-matching branch fires; failed-Req branches fall through.
Timeline (downslam): ANIM `[2,7]` → wind-ring telegraph → 0.22 WAIT → 7×5×6 ragdoll HITBOX at foot level (Y -1) → VELO `(0, -10, 26)` sendback with `RAGDOLL: 0.6` + shake. (Uppercut variant launches Y +32 with smaller wind-expand telegraph.)
Counterplay: telegraph 0.22s+ before the hitbox; blockable melee.

**SPECIAL — Kamehameha** | 22s
Concept: signature charge-and-fire beam. Two-phase via BRANCH (router fires `fire`; `fire` spawns the projectile, target-side `beamHit` plays impact). Demonstrates §10.7 visual density at the windup beat.
Timeline: 0.00 ANIM `[15,25]` (special-anim slot) + DirectionLock 1.6s → charge cluster (Cursed Energy AMOUNT 25 + Energy Sparks 18 + growing Sphere 0.3→3 + 360 Wind rotating + SFX 128213617026859) all tagged `khCharge` → 0.6s WAIT → `Cancel` tag flush → branch `fire`: FoV zoom -15, Shake Heavy, 30-particle Burst → PROJECTILE 5×5×100 Bullet, SPEED 280 TIME 0.55 (≈154 studs reach, CONTINUE true pierces), 22 dmg, FILTER INTERVAL 0.2 → Cylinder + Beam visuals riding `PROJECTILE TAG: khBeam`.
Counterplay: ≈1.5s telegraph (charge cluster) before fire; blockable bullet; long cooldown.
Assets: ANIM `[15,25]`, SFX `128213617026859`, `129465573909487` (both from §5.5 known-safe IDs).

**SKILL 1 (KEY 1) — Ki Blast Volley** | 10s
Concept: quick 3-shot ki-pellet volley, spread positions (center / left+1 / right-1).
Timeline: anim `[3,3]` → spark windup → three PROJECTILEs (3-stud Sphere visuals riding each tag) at 0.12s gaps; each 4 dmg Bullet, CONTINUE false (dissipate on impact). Target-side `kiHit` plays a small `Burst` flash.
Counterplay: blockable bullets, plenty of space between shots.

**SKILL 2 (KEY 2) — Spirit Bomb** | 28s
Concept: heavy multi-phase ranged finisher. Hands-up charge above head, then hurl. Demonstrates BRANCH-with-multiple-phases requirement (`charge` → `hurl` → `bombHit`).
Timeline: ANIM `[1,12]` + DirectionLock 1.6s → branch `charge` (Cursed Energy 30 + sparks 20 + growing Sphere + Glow at Head Y+5, ~1.1s) → branch `hurl` (Cancel `spirit` tag → PROJECTILE 10×10×10, SPEED 40 TIME 1.4 ≈ 56 studs, 18 dmg `Explosion` unblockable, riding-Sphere + Glow + Cursed Energy visuals) → target-side `bombHit` (Shake Heavy + cartoony-Clash with `ALT OPACITY: 10` per §5.23 + 8-stud expanding Ring).
Counterplay: ~1.5s charge telegraph; slow projectile (40 studs/s); long cooldown (28s, > 18 dmg budget per §8).

**SKILL 3 (KEY 3) — Dragon Fist** | 18s
Concept: flying single-target charge punch with afterimage trail. Borrows ANIM `[11,4]` (skill-anim slot) and builds bespoke wind/fire visuals around it.
Timeline: red-orange Cursed Energy + Energy Sparks windup 0.35s → trail visuals + VELO `(0, 4, 45)` flying chase + impact SFX → 0.18 WAIT → 16-dmg single-target Melee HITBOX (ragdoll) → follow-up VELO `(0, 8, 30)` with `RAGDOLL: 0.6`, `LAST HIT: 1` applies to target. Target-side `dragonHit`: cartoony Clash + Ring + Shake Heavy.
Counterplay: 0.35s windup + traveling VELO is visible; blockable melee; 18s cooldown ≥ 16 dmg budget.

**SKILL 4 (KEY 4) — Meteor Combo** | 14s
Concept: speed-blitz rapid punches ending in a launch — Goku's "afterimage-punch flurry."
Timeline: ANIM `[17,2]` + DirectionLock 1.4s → forward dart VELO Z+25 → LOOP `LOOP BACK 3, LOOP AMOUNT 6` of (WAIT 0.05 + Clash visual + 3-dmg melee hitbox + WAIT 0.1) = 6 rapid hits over ~1s → 0.15 WAIT → 20-particle Burst + Shake Medium + 8-dmg ragdoll HITBOX + VELO `(0, 6, 22)` launch.
Counterplay: all hits blockable melee; the loop body has STUN 0.2 (under the §8 "≥12 dmg = blockable OR stun ≤ 0.4" threshold — each individual hit is low-damage so the threshold doesn't apply, and the 8-dmg finisher is blockable).

**AWAKENING — Super Saiyan** | DURATION 60s, Pattern B
Concept: per §8.1 Pattern B — transformation cinematic, then a lingering buff. Hides itself in awakening (`AWK2` Prop), requires base form (`Req ULT FLIP: false`).
Timeline: ANIM `[1,12]` 2s + DirectionLock + IFrame → roaring yellow aura cluster (Cursed Energy AMOUNT 35 + Energy Sparks 25 + spinning 360 Wind + expanding Wind Expand + Shake Heavy + FoV zoom -8 + SFX) → 1.6s WAIT → `ULTGIB` (awakening burst) + `ADDHP 25` → lingering `STATE SpeedMultiplier 1.35 TIME 60` + `STATE JumpMultiplier 1.25 TIME 60` + persistent low-density Cursed Energy + Glow aura tagged `ssjAuraLinger` for the full 60s.
Counterplay: 1.8s IFrame window during transformation (canon — opponent can't interrupt SSJ ascension), but no offensive damage during the awakening itself; the buff is +35% speed, +25% jump, +25 HP.

---

## Concerns / limitations encountered

- **Asset IDs:** I used only the four SFX IDs explicitly listed as known-safe in SKILL.md §5.5. I did **not** scrape new audio or use any meshes/textures (no `<MESH_ID_FROM_USER_X>` placeholders were necessary). All four SFX IDs are reused from samples cited in the skill — they may or may not be ideal for Kamehameha/Spirit Bomb but they're proven to load.
- **ANIM_USE pairs** are drawn only from §7's safe list `[1,3] [1,4] [1,5] [1,6] [1,9] [1,12] [2,7] [2,8] [2,20] [3,3] [11,4] [15,25] [17,2]`. Animations are not guaranteed to match Goku-specific moves; the user will need to swap the picker slots if a different animation reads better for, e.g., the Kamehameha pose.
- **Mesh-driven visuals omitted:** SKILL.md §7.5 mandates *asking* the user for mesh IDs. The Spirit Bomb in particular would benefit from a custom green orb mesh; I used a stacked Sphere/Glow/Cursed Energy proxy instead. The aura is convincing without a mesh.
- **SPECIAL.BRANCH note:** `PROJECTILE.BRANCH` is documented as broken (§5.24); I used `BRANCH TARGET` exclusively for projectile hit reactions.
- **AWAKENING Req ULT:** Set `FLIP: false` (must be in base) per the §5.18/§5.20 correction that earlier docs had ULT semantics backwards.
- **VELO + ANIM cancellation risk:** Dragon Fist's `VELO 45 studs` over 0.3s is over the §10.5.A "30 stud / 0.15s safe" threshold. The animation may be partially cut; I accepted the risk because the move needs the long travel. If the anim is visibly clipped in-game, apply the §7 patch (split ANIM into two PREVIEW windows around the VELO).
