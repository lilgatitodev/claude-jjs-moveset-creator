# Tidekeeper — Design Brief

**Concept.** Conjures water-pressure as cursed energy; switches between **Ebb** (defensive, displacement) and **Flow** (offensive, AOE) stances. The same skill behaves differently in each stance. Awakening (**Maelstrom Form**) collapses the duality into a lightning-laced storm-master (Pattern C full-swap). Palette: teal `100,200,230`, ocean `40,80,140`, foam `230,245,255`.

## Base form

**CHASE — Riptide Dash (CD 4s).** Dir-lock → twin foot Wind Streaks → forward VELO 28 / 0.28s → foam Cancel. *Counter:* no hitbox, pure reposition.

**m1 / m2 — Right/left water-trail jabs (4 dmg, Melee).** 0.12s startup, Melee Trail in foam, 5x5x5 hitbox, 8-stud nudge. *Counter:* parry/block.

**m3 — Whirl kick (5 dmg).** Whirl Slash + leg Melee Trail; wider 7x5x5 hitbox at 0.14s. *Counter:* same as m1 but with a tell.

**m4 — Dual-variant finisher (§10.7 router).** Default Line is just `BRANCH → m4Hub`. The hub fires `m4Downslam` (Req: AIR true) **then** `m4Uppercut` (Req: AIR false). Each has ~0.18s telegraph (Wind Expand / Cleave arc), lands 6 dmg + 0.7-0.8 stun with `CLEAR KNOCKBACK`, and applies per-target launch on hit (Y=-25 ragdoll downslam, Y=+28 uppercut). *Counter:* blockable Melee, clear visual per variant.

**SPECIAL — Tidal Wedge (CD 10s, COUNTER).** Dir-lock → Sphere + Ring + dual-arm Cursed Energy → SFX → 0.8s COUNTER window (Melee+Bullet, stuns 1.6s). On parry: user gets IFrame + Burst + 10 evasive; attacker gets Sphere collapse + 0.3s Z=-30 / Y=8 knockback with 1.5s ragdoll. *Counter:* visible Sphere telegraph; Explosion/Swarm/Domain bypass.

**SKILL 1 — Stance Shift (CD 6s, TAG-state + BRANCH hub).** Toggles `TideFlow` tag (unset = Ebb, 1 = Flow for 60s) and drives SKILL 2. `stanceHub` checks the tag; if Flow → `shiftToEbb` (Ring contracts, Sphere settles, +8 evasive, drops the 1.15× speed). If Ebb → `shiftToFlow` (Cursed Energy aura tagged `flowAura`, Ring expands, arm Sparks, +1.15× speed for 60s). *Counter:* no damage; 0.5s recovery window.

**SKILL 2 — Undertow (CD 12s, stance-aware via TAG check + BRANCH hub).**
- *Ebb:* dir-lock → backstep VELO `(0, 4, -25)` → 8x6x6 frontal Melee 9 dmg → on-hit knockback + 0.6s ragdoll.
- *Flow:* 0.9s dir-lock → Whirl Slash + Ring + Cursed Energy + Sparks (visual density) → 0.4 wait → Burst + Shake Medium → 14x7x14 Explosion 360-block 14 dmg → on-hit knockup `(0, 18, 18)` 1.4s ragdoll.
*Counter:* Ebb blockable Melee; Flow has 0.95s telegraph and is also blockable.

**SKILL 3 — Pressure Lance (CD 14s).** 0.45s arm Cursed Energy + Sparks + Glow charge tagged `lanceCharge` (visual density) → Cancel charge → Beam telegraph (0,0,0)→(0,0,40) Exponential → Wind Streak → projectile 13 dmg / 95 speed / 1.2s lifetime, Bullet. *Counter:* charge is the tell; Bullet is counterable by some, blockable.

**SKILL 4 — Whirlpool Snare (CD 16s).** 0.9s dir-lock → Ring + Sparks + Cursed Energy at the spawn point 8 studs forward (visual density) → 0.55s wait → Cylinder + Whirl Slash visuals → Swarm hitbox 7 dmg / 0.55 stun. On-hit applies `TRACK: true` inward pull + 0.6 Stun. *Counter:* clearly-marked AoE telegraph; Swarm uncounterable but blockable, pre-block works.

## AWAKENING — Maelstrom Form (DURATION 60s, Pattern C)

2.5s cinematic: IFrame+DirectionLock, [15,25] ANIM, Screen Color wash, server-side Cursed Energy aura, perpendicular expanding Rings, Sphere, Sparks, Shake Heavy, 60s rotating crown **Mesh** (placeholders), global SFX. Then `+30 HP`, `1.25× speed`, `1.2× jump`, `ULTGIB`. Base skills carry `AWK2`; awakened skills carry `AWK`. Awakened CD spread 5/10/12/20 — two ≤12s satisfies §8.1.

**Awakened SKILL 1 — Squall Step (CD 5s).** 0.3s IFrame + Visibility fade + Wind Streak + Glow → VELO 38 / 0.3s → trailing 8x6x8 360-block 8 dmg. *Counter:* short telegraph, Melee/blockable.

**Awakened SKILL 2 — Tempest Lash (CD 10s, LOOP flurry).** Looped M1-anim: 3-cycle Whirl Slash punches (3 dmg, 0.18s gap) ending in 7 dmg 360-block finisher with sendback VELO `(0, 6, -28)` + 1.2s ragdoll. LOOP `HOLD: false`. *Counter:* each hit blockable, finisher Burst-telegraphed.

**Awakened SKILL 3 — Stormpiercer (CD 12s).** Bigger Pressure Lance: Weak Lightning + Glow charge → Beam telegraph (80 studs Exponential) → projectile 16 dmg / 140 speed / `CONTINUE: true` (pierces). *Counter:* 0.38s charge, Bullet/blockable.

**Awakened SKILL 4 — Eye of the Storm (CD 20s).** Heavy finisher. 0.85s charge (contracting Ring + Sphere + Sparks + Cursed Energy, Shake Heavy) → 20x10x20 Explosion 22 dmg / 1.1 stun, 360-block, on-hit Y=+28 / 2s ragdoll, surrounded by Burst + Ring + Sphere + 4 Weak Lightning. *Counter:* 0.85s telegraph, blockable.

## Meshes I need (§7.5)

**MESH_ID_FROM_USER_1 / TEXTURE_ID_FROM_USER_1** — rotating crown in AWAKENING. Shape: curled-wave halo ~3 studs across, flat. Texture: translucent teal-to-ocean swirl. Mount `(0, 6, 0)` above HumanoidRootPart, Y-rotates 360° over 60s. Vibe: "tide-king's halo". If unsourced, delete that one VISUAL; rest still fires.

## Asset notes

All SFX IDs re-used from SKILL.md §5.5 safe set — no external scraping. Only the AWAKENING mesh is user-supplied.
