---
name: jjs-moves-catalog
description: >
  Reference catalog of every built-in skill, special, awakening,
  passive, and variant for all 16 canonical Jujutsu Shenanigans
  (Roblox) characters — Honored One, Vessel, Restless Gambler, Ten
  Shadows, Mahoraga, Perfection, Blood Manipulator, Switcher, Defense
  Attorney, Cursed Partners, Puppet Master, Head of the Hei,
  Salaryman, Disaster Plants, True Cannon, Black Death. Each entry
  has category, cooldown, damage, and a one-line description sourced
  from the JJS Fandom wiki. Use this skill when the user wants a
  built-in move plugged into a custom moveset (`SKILL.MOVE`,
  `SPECIAL.SPEC`, or borrowed animation), when they mention a named
  ability (Rapid Punches, Dismantle, Hollow Purple, etc.), or when
  the companion `jjs-moveset` skill asks for the exact name of a
  built-in move.
---

# JJS Character Moves Catalog

Source: https://jujutsu-shenanigans.fandom.com/wiki/Characters and each
character's individual wiki page.

Every move below is **already built into the game**. You can use it in
a custom moveset in three ways:

1. **As a full skill** — `{ "K_NAME": "SKILL", "MOVE": "<exact name>",
   "SPEED": 1, "CANCEL LAST": false, "ENABLE VARIANTS": true,
   "HOLD FOR": 0, "START": 0 }` — plays the entire built-in skill.
   This is the "lazy" route: zero animation work, full canon move.

2. **As a special** — `{ "K_NAME": "SPECIAL", "SPEC": "<exact name>",
   ... }` — for moves listed below as `SPECIAL` or `SPECIAL VARIANT`.

3. **As just an animation** — `{ "K_NAME": "ANIM", "ANIM_USE": [<group>,
   <index>], ... }` — borrow the *animation* of an existing move while
   building your own hitboxes/visuals around it. The player picks the
   matching slot in-game from the Animation node's category list.
   When you do this, **call out the source move in the design brief**
   so the user knows it's from an existing skill.

Tags after each move: `[CATEGORY, cooldown, damage]`. `-` means N/A.
Variants share the parent skill's cooldown and only trigger when the
Variants tab's timing/trigger conditions are met.


## Honored One (Gojo)

- **Infinity** `[COSMETIC, -, -]` — Posing block animation; a blindfold covers their eyes while their technique passively guards them.
- **Limitless** `[SPECIAL, 15s, -]` — Aiming at an enemy, the user raises a hand and instantly teleports beside them.
- **Limitless** `[SPECIAL VARIANT (Aerial), 15s, 8]` — On an airborne enemy, the user appears above and air-kicks them toward the floor.
- **Lapse Blue** `[SKILL, 13s, 5 pull + 7.5 kick]` — Vacuum pulls a target at least 35 studs away in for an unblockable kick.
- **Reversal Red** `[SKILL, 20s, 12.5]` — A vibrant red orb travels 40 studs forward, then explodes with massive repulsive force.
- **Reversal Red** `[SPECIAL VARIANT (Limitless / Behind), puts used abilities on cooldown, 12.5]` — Limitless during the windup teleports behind the target for a point-blank Red.
- **Reversal Red** `[SPECIAL VARIANT (Aka / Black Flash), puts used abilities on cooldown, 15]` — If the target was acting during the Limitless variant, the projectile becomes an enhanced "赫" Black Flash.
- **Rapid Punches** `[SKILL, 15s, 17.25]` — A spinning kick locks a nearby enemy in place for a barrage ending in a knockback blow.
- **Face Grater** `[SPECIAL VARIANT, puts used abilities on cooldown, 10.2]` — Limitless right after Rapid Punches drags the opponent across the floor before tossing them forward.
- **Twofold Kick** `[SKILL, 18s, 10]` — Kicks the enemy upward, anchors them mid-air, then unblockable second kick bounces them higher.
- **Six Eyes** `[AWAKENING, -, +25 HP heal]` — Removes the blindfold and proclaims "Let's get... a little crazy". Duration: 60s.
- **0.2 Domain** `[AWAKENING ATTACK / ALTERNATE AWAKENING, -, scales]` — Pressing special during the Awakening sequence expands Infinite Void for 0.2s then chains into a three-phase blitz with a scaling kill multiplier.
- **Lapse Blue MAX** `[SKILL, 17s, 44]` — Unleashes a controllable blue orb that sucks in everything near it.
- **Unlimited Purple** `[VARIANT, -, 50-100]` — If Lapse Blue MAX kills, shooting the lingering orb with Reversal Red MAX detonates a Purple bomb.
- **Reversal Red MAX** `[SKILL, 10s, ~35]` — Fires a powerful red orb that ragdolls hit enemies; damage falls off with distance.
- **Reversal Red MAX** `[SPECIAL VARIANT (Limitless Rebound), 10s, 10 punch / 15 self-damage]` — Limitless before the shot causes the orb to rebound; on rebound hit, the target is pulled for a Black Flash punch.
- **Hollow Purple** `[SKILL, 40s, 70]` — Combines Blue and Red into an imaginary purple mass that atomizes everything in its path.
- **Infinite Void** `[DOMAIN EXPANSION, 120s, -]` — Expands Infinite Void; enemies caught are stunned with endless stimuli. Duration: 14s.

## Vessel (Sukuna)

- **Combat Instincts** `[SPECIAL, 2s, -]` — During an M1 or skill windup (excluding Manji Kick), instantly cancels the action with no endlag, keeping the move off cooldown.
- **Combat Instincts** `[SPECIAL VARIANT (Throwable), 2s, 15]` — Triggering the special near a throwable hurls the object forward with a punch.
- **Cursed Strikes** `[SKILL, 14s, 17.5]` — Dashes forward with glowing red eyes; on hit, a flurry of punches ends with a calf kick that immobilizes the target.
- **Cursed Strikes** `[AIR VARIANT (Dropkick), 14s, 14]` — Jumps and launches into a powerful dropkick toward the ground.
- **Crushing Blow** `[SKILL, 15s, 12]` — Grabs the enemy by the torso, slams them into the ground twice, launching them upward.
- **Crushing Blow** `[AIR VARIANT, 15s, 12]` — Dashes through the air, grabs the enemy, performs the same slam combo.
- **Divergent Fist** `[SKILL, 18s, 10]` — A wound-up punch followed by delayed cursed-energy impact that launches the opponent.
- **Divergent Fist** `[VARIANT (Stun), 18s, 5]` — If the delayed impact interrupts a blocking/acting target, they get stunned briefly instead of ragdolled.
- **Black Flash** `[VARIANT, 18s, 10 (20 on interrupt)]` — Pressed at the right time, the wound-back fist releases a Black Flash that blasts the enemy away.
- **Black Flash Chain** `[VARIANT, 18s, 7 per (15 final)]` — A Black Flash on the enemy's back stuns instead of ragdolls, allowing up to four chained Black Flashes ending in a heavy final blow.
- **Manji Kick** `[COUNTER, 20s, 8.5]` — Raises arm and leg in a Taido evade stance; on any attack, performs a roundhouse kick.
- **King of Curses** `[AWAKENING, -, +45 HP heal]` — Sukuna's alter ego takes over with black tattoos and a red aura. Duration: 60s.
- **Shrine** `[PASSIVE, 1.7s, 14]` — M1s become Dismantle slashes with 2.5x range; blockable from all sides.
- **Cleave** `[SPECIAL, 12s, 40% current HP (min 10)]` — Winds back to grab forward, then unleashes Cleave slashes that launch the target.
- **Dismantle** `[SKILL, 13s, 17.5]` — Swings forward, unleashing a barrage of Dismantle slashes within 30 studs.
- **Dismantle** `[AIR VARIANT, 13s, 25]` — Jumps, hovers briefly, then throws a long Dismantle slash.
- **World Cutting Slash** `[VARIANT, puts all used moves on cooldown, 80]` — Using Rush during Dismantle's windup, then Open, then Cleave chants "SCALE OF THE DRAGON / RECOIL / TWIN METEORS" before a massive horizontal slash.
- **World Cutting Slash** `[AIR VARIANT, same as ground, 80]` — Started in the air, the whole move plays out midair.
- **Open** `[SKILL, 40s, 30]` — Summons fire from hands, claps, and fires a flaming arrow that produces a towering explosion.
- **Rush** `[SKILL, 15s, 25]` — Rushes forward at high speed; on collision, hurls the target, chases, and slams them down.
- **Malevolent Shrine** `[DOMAIN EXPANSION, 120s, 2 per slash]` — Expands the temple-domain; constant Dismantles deal up to ~220 total over 110 slashes. Duration: 18s.

## Restless Gambler (Hakari)

- **Door Guard** `[SPECIAL, 16s, 5]` — Two shutter doors protect the user; melee punches them open and bullets shatter the doors.
- **Reserve Balls** `[SKILL, 12s, 7.5]` — Flicks a small steel ball that travels 65 studs and ragdolls enemies within 15 studs.
- **Reserve Balls** `[VARIANT (Shutter Doors), puts both on cooldown, 7.5]` — Using Shutter Doors during the windup manifests doors where the ball lands or maxes out.
- **Shutter Doors** `[SKILL, 15s, 8]` — Summons pachinko shutter doors that crush and stun the target.
- **Shutter Doors** `[VARIANT (Lingering), 15s, 6]` — If they miss, the doors linger for 7 seconds; the user can bounce on them or send ragdolled enemies bouncing.
- **Rough Energy** `[SKILL, 14s, 12.5]` — Winds up a sharp cursed-energy fist for a heavy launching punch.
- **Rough Energy** `[AIR VARIANT, 14s, 8]` — In the air, hovers briefly and stomps the ground for a launching mini-shockwave.
- **Rough Energy** `[AIR VARIANT (High), 14s, 16]` — From higher than jump distance, the aerial variant becomes unblockable with doubled damage.
- **Fever Breaker** `[SKILL, 23s, 15]` — A ranged kick suspends the target before shutter doors and a dropkick launch them.
- **Fever Crush** `[VARIANT, puts both on cooldown, 8 doors + 12 stomp]` — Shutter Doors during the windup pins the target so the user stomps down to crush them.
- **Fever Crush** `[VARIANT (Ragdoll), puts both on cooldown, 24 + 2 per bounce]` — A ragdolled target takes doubled stomp damage as the doors crush down on them.
- **Idle Death Gamble** `[AWAKENING / DOMAIN, -, +15 HP heal]` — Casts a Richii-trigger domain. 4 scenarios; 4th attempt grants a pity Jackpot. Duration: 80s.
- **Renewal** `[PASSIVE, 12s, -]` — Landing Reserve Balls inside Idle Death Gamble, then using it again within 8s, rewinds both players to the prior position and undoes user damage.
- **Jackpot** `[AWAKENING, -, -]` — A successful Richii grants infinite cursed energy with a green aura. Duration: 100s (50s if pity).
- **Jackpot** `[PASSIVE, -, heals over time]` — Reverse Cursed Technique regenerates the user; Awakening empties after 333 damage.
- **Rhythm** `[SPECIAL, 8s, -]` — Dancing reduces cooldowns by 0.6s and stacks an attack-speed buff.
- **Lucky Volley** `[SKILL, 10s, 20.7 barrage + 8 last]` — A flurry of punches ends with a powerful unblockable swipe.
- **Lucky Rushdown** `[SKILL, 15s, 24]` — Sprints forward; on contact, grabs by the leg, drags, and throws forward.
- **Overwhelming Luck** `[SKILL, 20s, 40]` — Charges cursed energy, sprints, grabs, pummels, ends with a launching punch.
- **Energy Surge** `[SKILL, 25s, 20]` — A short dash with a heavy punch; landed, propels target up before a mid-air follow-up kick.

## Ten Shadows (Megumi)

- **Lurking Shadow** `[SPECIAL, 10s, -]` — Sinks into shadow for ~2 seconds with vastly increased walk speed but disabled abilities.
- **Lurking Shadow** `[SPECIAL VARIANT (Item Storage), 10s, -]` — Holding an item during the special stores it in the shadow until M1 retrieves it.
- **Rabbit Escape** `[SKILL, 18s, 14]` — Handsign releases shadow rabbits that swarm the target's legs, halting movement.
- **Rabbit Escape** `[VARIANT (Lurking Shadow), 18s, 14]` — Used from inside Lurking Shadow, rabbits propel the user upward and knock back nearby foes.
- **Nue** `[SKILL, 20s, 16]` — Summons the winged shikigami Nue, who swoops down from the sky toward the cursor.
- **Nue** `[VARIANT (Flight), 22s, 16]` — Pressing special during Nue's windup lets the user ride Nue's leg briefly.
- **Toad** `[SKILL, 15s, -]` — Summons a giant frog whose tongue grabs a target within 75 studs and pulls them in.
- **Well's Unknown Abyss** `[VARIANT, 15s, 12]` — Using Nue during Toad's startup combines them into a swarm of winged toads that lift, slam, and launch the target.
- **Divine Dog: Totality** `[SKILL, 20s, 18]` — Summons an improved Divine Dog; the next three uses command it to attack the closest enemy.
- **Insanity** `[AWAKENING, -, +15 HP heal]` — Kneels and chants "Picture it in your head!" / "Oh whatever... here we go!". Duration: 60s.
- **Switch** `[PASSIVE, -, -]` — Pressing the Awakening button during Insanity swaps between awakened and base movesets (8 total moves).
- **Max Elephant** `[SKILL, 25s, 35]` — A massive elephant crashes from the sky onto the targeted area.
- **Great Serpent** `[SKILL, 25s, 29]` — Summons Orochi; controls it for ~2s, grabbing enemies in its mouth and applying poison.
- **Shadow Swarm** `[SKILL, 15s, 13 + 5 final]` — Two shadow clones plus the user rush forward, batter the target, then launch with a metal bat.
- **Domain Expansion: Chimera Shadow Garden** `[VARIANT / DOMAIN, 30s, drains user 4 HP/s]` — Using Shadow Swarm against a domain border opens a tunnel that disables the invaded domain's sure-hit effect.
- **Mahoraga** `[AWAKENING ATTACK / SUMMON, 120s, -]` — Aiming at an enemy within 60 studs, drains 40% Awakening to chant a summoning Ritual that calls Mahoraga.

## Mahoraga

- **Ritual** `[PASSIVE, full bar = 25s, drains HP if empty]` — Mahoraga's Ritual Bar refills by landing any move or M1 except Ground Pitch and World Slash.
- **Adaptation Wheel** `[SPECIAL, -, -]` — Press Awakening to cycle modes, Special to enter; modes: Attack (sword), Defense (shield), Special (star).
- **Attack Mode: Sword of Extermination** `[PASSIVE VARIANT (Attack Mode), -, 14]` — All M1s have 1.5x range; in Attack Mode they're much faster than normal.
- **Defense Mode: Parry** `[PASSIVE VARIANT (Defense Mode), -, 15]` — Basic attack becomes a parry stance that retaliates and deflects ranged attacks.
- **Special Mode: Sword of Extermination** `[PASSIVE VARIANT (Special Mode), -, 14]` — Last two M1s bypass block while keeping extended range and destruction.
- **Divine Pummel** `[SKILL, 12s, 15]` — Swings to grab forward; on hit, slams the target twice and punches them away.
- **Ground Pitch** `[SKILL, 20s, 20]` — Grabs a chunk of debris and throws it at the nearest visible target.
- **Ground Pitch** `[VARIANT (Grab), 20s, 25]` — If the opponent is close, they get trapped within the debris and thrown.
- **Earthquake** `[SKILL, 15s, 12.5]` — Slams a fist into the floor, smashing anyone in front.
- **Earthquake** `[HOLD VARIANT, 15s, 12.5 (total 25)]` — Holding ~0.8s releases a second larger shockwave that lifts enemies.
- **Attack Mode: Takedown** `[SKILL, 7s landed / 15s missed, 12.5]` — Crouches then dashes instantly to a target within 70 studs, swinging upward.
- **Defense Mode: Adaptation** `[SKILL, 2s landed / 5s missed / 15s on Infinite Void, -]` — Adapt-stance; on hit, grants healing, full Ritual refill, and permanent damage reduction to that attack type.
- **Special Mode: World Slash** `[SKILL, 30s, 40]` — Winds up a Sword of Extermination swing that cuts through space in a piercing straight line.

## Perfection (Mahito)

- **Self Transfiguration** `[SPECIAL, -, -]` — Cycles arm shape through normal / blades / clubs, altering M1s, front dash, and Focus Strike.
- **Soul Reserves** `[SPECIAL VARIANT, -, -]` — Using the special while holding a Transfigured Human/Flesh stores it inside the user for later retrieval.
- **Blade Mode** `[PASSIVE VARIANT, 8s (Dash), 10 M1 / 6.5 Dash]` — Faster M1s with less damage; front dash becomes a teleporting slice with nearly double range.
- **Club Mode** `[PASSIVE VARIANT, 10s (Dash), 18 M1 / 8.5 Dash]` — Slower stronger M1s; front dash becomes an unblockable spin attack.
- **Stockpile** `[SKILL, 12s, 12]` — Conjures two flesh hammers (green and purple) for a two-hit launching combo.
- **Stockpile** `[AIR VARIANT, 12s, 10]` — Uses a purple flesh hammer to hop and slam down on the opponent.
- **Soul Fire** `[SKILL, 12s, 12]` — Manifests a flesh-arm gun and fires three transfigured-human projectiles.
- **Soul Fire** `[HOLD VARIANT, 12s, 12]` — Holding the move uses up stored Transfigured Humans/Flesh as bullets.
- **Focus Strike** `[SKILL, 15s, 10]` — A cursed-energy fist strike to the torso that pushes the target back with stun.
- **Focus Strike** `[VARIANT (Black Flash), 15s, 12]` — Pressed at the right moment, the wound-back arm releases a heavy Black Flash.
- **Chainwhip** `[VARIANT (Blade Mode), 15s, 3]` — In Blade Mode, Focus Strike becomes an upward-swinging chainwhip that drags targets toward the user.
- **Homerun** `[VARIANT (Club Mode), 15s, 18]` — In Club Mode, Focus Strike becomes a giant-armed uppercut that launches targets away.
- **Body Repel** `[SKILL, 20s, 14]` — Four transfigured humans form an amalgamation that rams anyone in front for 70 studs.
- **Body Repel** `[VARIANT (Passenger), 20s, 14]` — Focus Strike during the windup lets the user ride inside the amalgamation but lose steering.
- **Essence Of The Soul** `[AWAKENING, -, 15]` — Regurgitates a stack of transfigured humans and slams the ground, releasing a damaging worm swarm. Duration: 60s, +45 HP.
- **Awakening Black Flash** `[VARIANT, 15s, 10]` — Pressing Awakening while awakened performs a long lunge with arm wound back, grabbing for a Black Flash.
- **Idle Transfiguration** `[SKILL, 15s, 15 / 300]` — Long run applying their technique; the second touch on a target blows their head up; killing extends the run.
- **Body Disfigure: Drill Splitter** `[SKILL, 15s, 10 kick + 25 drill]` — Hoof-foot dropkick then drill-arm into the opponent's body.
- **Body Disfigure: Heart Piercer** `[VARIANT (Blade Mode), 15s, 42.5]` — In Blade Mode, body tendrils lift the user and impale anyone in their path.
- **Body Disfigure: Force Grab** `[VARIANT (Club Mode), 15s, 10 per slam]` — In Club Mode, the user's hand grows and extends to grab a target within 40 studs.
- **Spike Wrath** `[SKILL, 25s, 25]` — Morphs into a blob and sends constant spikes to latch onto targets within 25 studs before slamming them down.
- **Embodiment of Self Perfection** `[DOMAIN, 120s, Insta-kill]` — Domain with flower-pattern hands; caught targets get a transfiguration meter that empties when near the caster. Duration: 14s.
- **Instant Spirit Body Of Distorted Killing** `[SUPER AWAKENING, -, -]` — Requires landing Awakening Black Flash twice; true form sprouts blade extensions. Duration: 50s, +10 HP.
- **Sinister Spurs** `[PASSIVE VARIANT, 1.7s, 18]` — M1s lunge forward using razor-sharp joints; higher damage than normal.
- **Distorted Dash** `[PASSIVE VARIANT, 6s, 12]` — Dash crouches briefly then launches forward with multiple circular slices.
- **Widespread Strikes** `[SKILL, 20s, 56.55]` — Rushes with 8 afterimages around a purple-glowing area; trapped targets are barraged ending with a powerful punch.
- **Face Blitz** `[SKILL, 10s, 27.5]` — Dashes forward and grabs the target's face, jumping three times before a massive face-slam AoE.
- **Crushing Rushdown** `[SKILL, 15s, 16.65]` — Sends tendrils forward to grab a target within 70 studs and ram them along a long distance.
- **Head Splitter** `[COUNTER, 40s, 100]` — A slow-paced block stance; on hit, the user appears behind and slices the target's head clean off.

## Blood Manipulator (Choso)

- **Convergence** `[SPECIAL, 20s, -]` — Conjures 4 blood orbs that strengthen attacks and change how Supernova behaves.
- **Piercing Blood** `[SKILL, 15s, 12]` — Claps to fire an uncounterable thin beam of blood with 30 studs range.
- **Piercing Blood** `[HOLD VARIANT, 15s, 20]` — With a Convergence orb, holding ~1.5s doubles range and knockback.
- **Flowing Red Scale** `[SKILL, 12s, 13]` — Short windup rush; on contact, performs a 3-kick combo ending in a ragdoll-launching kick.
- **Flowing Red Scale: Stack** `[VARIANT, 12s, 17]` — A Convergence orb strengthens the final kick into a blood explosion.
- **Flowing Red Scale** `[AIR VARIANT, 12s, 6]` — Airborne, performs a downward axe kick that ragdolls upward.
- **Flowing Red Scale: Stack** `[AIR VARIANT, 12s, 10]` — A Convergence orb places a static blood mine beneath the user for later detonation.
- **Supernova** `[COUNTER, 15s missed / none landed, 14]` — Without orbs, a cross-armed guard counters melee with a shoulder tear and heavy punch, then grants 4 orbs.
- **Supernova** `[VARIANT (Orb), 4s, 8]` — Tosses a Convergence orb that becomes a blood bomb mimicking the user's movement.
- **Blood Edge** `[SKILL, 13s, 5 + 6]` — Two blades of blood: one grabs, then a dash and unblockable ground slam.
- **Duty as a Brother** `[AWAKENING, -, +20 HP heal]` — Says "Brothers, lend me your strength!" as Eso, Yuji, and Kechizu appear behind. Duration: 60s.
- **Slicing Exorcism** `[SKILL, +3.25s per orb up to 13s, 3 per tick +12 per orb up to 39]` — Manipulates Convergence orbs into a pressurized blood stream that ragdolls hit targets.
- **Slicing Exorcism** `[VARIANT (Chakram), 13s, 17.5]` — Special during windup forms blood into a rotating chakram thrown in a straight line; ricochets between targets.
- **Wing King** `[SKILL, 16s, 28.75]` — A lunging flurry of kicks and punches ending with a blood-rope pull, face punch, and slam.
- **Wing King** `[PASSIVE VARIANT (Blood King), once per Wing King with Convergence, 5 per orb]` — After Wing King, available orbs form a halo; pressing special launches one that homes on a target.
- **Blood Rain** `[SKILL, 35s, 2 per tick]` — A giant blood sphere above the user rains pellets within 35 studs; drains Awakening 5x faster while active.
- **Plasma Wave** `[SKILL, 45s, 95]` — Spins to create 14 orbs then fires a massive cone of blood with 120 studs range.

## Switcher (Aoi Todo / Takada-chan)

- **Boogie Woogie** `[SPECIAL, 6s (3s on block), -]` — A clap swaps places with a target within 60 studs; unblockable if within 25 studs.
- **Boogie Woogie: Item Swap** `[SPECIAL VARIANT, 3s, -]` — Swaps with throwables, banana peels, or TNTs; half cooldown.
- **Boogie Woogie: Fakeout** `[SPECIAL VARIANT, none (6s if feinting), -]` — An M1 before the swap cancels it and refunds Awakening.
- **Boogie Woogie: Perfect Swap** `[SPECIAL VARIANT, none (3s on block), -]` — An M1 timed right after the clap keeps special off cooldown and refunds Awakening.
- **Boogie Woogie: Target Swap** `[SPECIAL VARIANT, 6s, -]` — Hover the cursor between two targets and re-press special to swap them with each other.
- **Swift Kick** `[SKILL, 17s, 7 + 3 + 7]` — Spins twice into a non-lethal launching kick, then grabs and tosses the helpless target.
- **Brute Force** `[SKILL, 20s, 17.5 / 6 gust]` — A massive non-cursed-energy punch hurls opponents; a shockwave hits if they're too far.
- **Pebble Throw** `[SKILL, 12s, 4]` — Knocks up a pebble with a stomp and punches it 45 studs forward as a cursed projectile.
- **Pebble Combo** `[VARIANT, 12s / 6s special, 11.25]` — Swapping within 1.5s of pebble contact appears in front of the target for a stunning punch combo.
- **Pebble Glide** `[VARIANT, 12s, -]` — Swapping with the pebble in flight transfers its momentum to the user for a fast glide with a fire trail.
- **Brutal Impact** `[VARIANT, 20s / 12s, 17.5 / 6 gust]` — Brute Force used during a Pebble Glide becomes a devastating roundhouse kick.
- **Brutal Impact** `[VARIANT (Black Flash), 20s / 12s, 18 / 6 gust]` — Pressing Brute Force again as the kick unleashes turns it into a Black Flash with red sparks.
- **Elbow Drop** `[SKILL, 20s, 5 + 5]` — A charged uppercut launches the target up, then a dive crushes them back down.
- **False Memories** `[AWAKENING, -, +20 HP heal]` — Cutscene reveals the user's brother and idol motivating them; pink sparkles fill the air. Duration: 60s.
- **False Memories** `[COSMETIC VARIANT, -, -]` — From other players' POV, the awakened user is bleeding profusely from the nose.
- **Idol's Debut** `[SKILL, 17s, 30]` — A short swing-running rush; on hit, a cutscene with their idol kicking the target back down to the floor.
- **Climax Jumping** `[SKILL, 22s, 42.4]` — Lunges ~40 studs with i-frames; on contact, the idol and user pummel the target before celebrating.
- **Dreams** `[SKILL, 10s, 21]` — Three heavy forward punches with pink heart effects while the idol Gangnam-styles.
- **Brothers** `[COUNTER, 45s, 80]` — A "high five" counter; on melee hit, swaps with Yuji Itadori who delivers a devastating Black Flash.
- **Brothers** `[VARIANT (No Counter), 45s, 70]` — Without a counter, the user swaps on their own so Yuji can lunge with the Black Flash.

## Defense Attorney (Higuruma)

- **Gavel** `[PASSIVE, 1.7s, 14]` — M1s use a gavel; the second M1 is split into 2 hits. M1 count resettable to this attack after a block break.
- **No Escape** `[SPECIAL, 5s, 2]` — Launches the gavel forward; if it misses, it rebounds; manually recallable via special, move, or M1.
- **No Escape** `[SPECIAL VARIANT (Leap), resets to 8s, -]` — If the gavel hits, special again starts a quick run ending in a hop over the target's head.
- **Extended Swings** `[SKILL, 15s, 15]` — Turns the gavel into a hammer for 3 swings ending in an uncounterable knockdown slam.
- **Extended Swings** `[VARIANT (Sweep), 15s, -]` — Without the gavel, the user instantly regains it and extends its handle in a block-breaking sweep.
- **Justice Served** `[SKILL, 18s, 12]` — Charges a powerful ground slam with a comedically enlarged gavel that launches the target skyward.
- **Justice Served** `[VARIANT (Block Break), 18s, 9]` — A blocking target instead gets crushed under the tool's weight with block broken.
- **Judgement's Reach** `[SKILL, 13s, 7]` — Extends the gavel's handle to attack from a distance with slight stun.
- **Judgement's Reach** `[VARIANT (Double), 13s, 13]` — Pressing it twice in the windup follows the handle with an uncounterable grounding slam.
- **Judgement's Reach** `[VARIANT (No Gavel), 13s, 6]` — Without the gavel, the user recalls and skips straight to the slam.
- **Judgement's Reach** `[VARIANT (Special Dash), 7s, -]` — Special during the windup extends the handle backwards to launch 35 studs forward.
- **Pressing Charges** `[SKILL, 16s, 7.5]` — Kicks the target then rushes in for a swing, sets to the 3rd M1.
- **Pressing Charges** `[VARIANT (No Gavel), 16s, 4]` — Without the gavel, the second rush is skipped.
- **Deadly Sentencing** `[AWAKENING DOMAIN, -, +40 HP heal]` — Domain with podiums and guillotines; trial picks among Confess/Silence/Denial. Duration: 300s, breaks after 6 wrong guesses.
- **Death Penalty** `[SUPER AWAKENING, -, +40 HP from Deadly Sentencing]` — Gavel breaks into the Executioner's Sword: any cut kills. Duration: 60s (90s if domain ended at 2 bars).
- **Executioner's Sword** `[PASSIVE VARIANT, 1.7s, 8]` — 1.5x M1 range, 6 M1s per string; opponents visually shown dodging with golden afterimages.
- **Domain Amplification** `[SPECIAL, 8s after dispelled, -]` — Casts a thin veil of domain expansion granting ~0.5s i-frames and defending against sure-hit effects.
- **Execution** `[SKILL, 12s, 10 + 15]` — A forward stab; on hit, the sword impales the target's left arm followed by a spinning kick (applies Execution stack).
- **Execution** `[VARIANT (2nd stack), 12s, 10 + 15]` — A second hit impales the right foot before another knockback.
- **Execution** `[VARIANT (3rd stack / Kill), 12s, 10 + 300]` — A third hit drops the target for a head-plunge and finishing stomp.
- **Execution** `[AIR VARIANT, 12s, same as ground]` — Airborne, hovers briefly with 360° aim and bypasses ragdoll.
- **Final Judgement** `[SKILL, 25s, 10 / 300 on win]` — A spinning swipe; on landing, triggers a QTE clash that ends in a kill if the user wins.
- **Final Judgement** `[VARIANT (Loss), 25s, 10 + 25]` — Losing the QTE results in the target dodging and kicking the user away.
- **Final Judgement** `[VARIANT (Feint), 25s, -]` — Move again during the lunge feints the attack with no endlag.
- **Verdict** `[SKILL, 10s, 5 + 5 (300 on kill)]` — Holds the sword by its blade to hilt-strike, then dashes for an edge slash that applies an Execution stack.
- **Verdict** `[VARIANT (Blocked), 10s, 5]` — A blocked second hit simply lets the target dodge the swing.
- **Verdict** `[VARIANT (Mordschlag), 10s, 5 + 10]` — Pressing again before the first hit lands keeps holding by the blade for a loud whirring weapon swing.
- **Verdict** `[VARIANT (Mordschlag Blocked), 10s, 5 + 50]` — A blocked Mordschlag briefly touches the target with the blade for heavy damage.
- **Triple Sentence** `[SKILL, 12s (+6s per extra throw), 8 per throw / 310 killing blow]` — Launches the sword forward to apply an Execution stack; up to three throws.

## Cursed Partners (Yuta)

- **Swordsmanship** `[COSMETIC / PASSIVE, 1.7s, 14]` — Holster for the katana plus a cursed ring necklace; katana sheathes/unsheathes via melee/dash/blocks.
- **Rika** `[SPECIAL, -, -]` — Rika partially manifests beside the user; pressing special toggles to her moveset. Double-press dismisses her.
- **Severing Path** `[SKILL, 15s, 9 + 1 per slash]` — Imbues the sword with cursed energy and steps forward slashing; fifth slash locks the target for a 4-slash combo.
- **Severing Path** `[AIR VARIANT, 15s, 12.5]` — Above jump height, pummels the target before grounding them with the fifth blow.
- **Resolute Slash** `[SKILL, 16s, 10]` — Vanishes and reappears behind a target within 35 studs to wind up a horizontal slash.
- **Veilstep** `[SKILL, none (18s if blocked), 5]` — Hops backwards with an upward kick that launches; if unblocked, a second part is available.
- **Veilstep** `[VARIANT, 18s, 10]` — Leaps up and dives with a cursed-energy-imbued fist slam that sheathes the katana.
- **Revolve** `[SKILL, 14s, 4]` — Jumps with two katana swings that slice and pull targets along, briefly ragdolling them.
- **Revolve** `[AIR VARIANT, 14s, 6]` — Midair or pressed again, drives the katana downward to impale targets through the chest.
- **Rika Downslam** `[SKILL, 16s, 10]` — Rika hovers to a target within 10 studs and slams them to the ground; otherwise punches forward.
- **Rika Downslam** `[VARIANT, 16s, 10]` — A ragdolled target above Rika is slammed with both arms, slightly longer ragdoll.
- **Rika Launch** `[SKILL, 8s, -]` — User hops on Rika's hands so she boosts them upward.
- **Rika Launch** `[VARIANT, 8s (puts both on cooldown), -]` — During a move, this feints it as Rika propels the user up.
- **Rika Throw** `[SKILL, 20s, 8-18 (or 0.5-20 self)]` — Rika picks up the user and hurls them; impact damages targets, missing damages the user.
- **Rika Haymaker** `[SKILL, 16s, 12]` — Rika winds up a heavy blow that knocks the target far back; blocking gets them ragdolled longer.
- **True Love** `[AWAKENING, -, +15 HP heal]` — Binding vow with Rika: "Come, Rika. Give me everything." Rika fully manifests. Duration: 60s.
- **Elbow Rush** `[SKILL, 15s, 33]` — Metal-cased elbow strike; user and Rika appear behind the opponent for a barrage and launching blow.
- **Copy: Cursed Speech** `[SKILL, 15s, -]` — Activates copied Cursed Speech, commanding all players within 35 studs to stop moving for 3 seconds.
- **Copy** `[VARIANT, 15s (none for Transfiguration kill / Adaptation / Clairvoyance), depends on move]` — Rika contact (Downslam / Slam / Grab) or Elbow Rush stores an enemy-specific copy: 21 possible copies — Purple (Hollow Purple, Honored One), Shrine (Dismantle, Vessel/Block NPC), Doors (Shutter Doors, Restless Gambler), 10 Shadows (Max Elephant, Ten Shadows), Adaptation (Adaptation, Mahoraga), Transfiguration (Idle Transfiguration, Perfection/Soul), Blood (Wing King, Blood Manipulator), Boogie Woogie (Brothers, Switcher), Execution (Final Judgment, Defense Attorney), Puppets (Boost On, Puppet Master), Projection (Cursory Impact, Head of the Hei), Ratio (Sharpen, Salaryman), Disaster Plants (Root Swarm, Disaster Plants), Granite Blast (Held Granite Blast, True Cannon), Black Mucus (Black Mucus, Locust Guy), Mass (Mass Breaker, Star Rage), Clairvoyance (Eye Catching, Aspiring Mangaka), Handsword (Ambush, Lucky Coward), Bird Control (Bird Control, Crow Charmer), Dismantle (Strong Dismantle, Strongest Of History), Kamehameha (Kamehameha, Monkey Kid).
- **True Love Beam** `[SKILL, 40s, 102.5]` — Rika grows large and unleashes a massive pink beam that erases anything in its path.
- **True Love Beam** `[VARIANT, 15s, 22.5]` — Move again during windup releases a smaller faster beam.
- **Authentic Mutual Love** `[DOMAIN, 120s, 8 katana]` — Stone-platform domain with blades raining; pickup-katana run activates one of five copied techniques. Duration: 45s.
- **Jacob's Ladder** `[VARIANT, once per domain, 63]` — After 4 katana swings, re-using the domain calls a divine ray that lifts and drains a target.
- **Rika Slam** `[SKILL (awakened Rika), 13s, 3 + 20]` — Rika grabs a target by the leg and slams them five times against the ground.
- **Rika Grab** `[SKILL (awakened Rika), 20s, -]` — On a ragdolled target, Rika hovers and constricts them tightly until attacked or 2s pass.
- **Shrine** `[DOMAIN SKILL, N/A, 8 katana + 5 per slash]` — Katana hit followed by 4 dismantle slashes that stun the target.
- **Shrine** `[VARIANT, N/A, 20]` — If the katana misses, sends a large horizontal slash forwards bisecting the path.
- **Thin Ice Breaker** `[DOMAIN SKILL, N/A, 8 katana + 20]` — Sky Manipulation extension: breaks the sky like a thin veil of ice.
- **Thin Ice Breaker** `[VARIANT, N/A, 15]` — The shattered atmosphere still harms enemies in the way for reduced damage.
- **Clairvoyance** `[DOMAIN SKILL, N/A, 8 katana]` — Marks the target with a manga panel for 10s, automatically dodging incoming attacks.
- **Clairvoyance** `[VARIANT, N/A, -]` — Marks the opponent from afar for 5 seconds if the katana missed.
- **Cursed Speech** `[DOMAIN SKILL, N/A, 8 katana + 15 "Plummet"]` — Commands the target to "Plummet", grounding them to the floor for a long time.
- **Cursed Speech** `[VARIANT, N/A, -]` — Without contact, shouts "Don't Move" freezing everybody in the domain.
- **Shikigami** `[DOMAIN SKILL, N/A, 8 katana + 27]` — Sends 3 winged-Rika-head shikigamis to swarm the opponent for 3 seconds.
- **Shikigami** `[VARIANT, N/A, 27]` — On miss, shikigami still swarm for 3 seconds but are blockable for half damage and cannot kill.

## Puppet Master (Kokichi Muta / Mechamaru)

- **Ultimate Proxy** `[COSMETIC, -, -]` — Pilots a cursed mechanical corpse to compensate for the user's Heavenly Restriction.
- **Energy Reserves** `[PASSIVE, -, -]` — A second Awakening Bar charges 1.5x faster after the first is full; extends Awakening duration.
- **Offload** `[SPECIAL, 10s, 3 puppet explosion]` — Cursed energy on the puppet keeps the next move off cooldown; an expendable puppet executes it and self-destructs.
- **Übercharge** `[SPECIAL VARIANT, resets to 10s, 3 puppet explosion]` — Used while Offload is on cooldown: glows red, puts the move on cooldown.
- **Puppet Barrage** `[SPECIAL VARIANT, -, 6 + 8]` — Special after Offload calls down two corpses to grab the target and toss them up for an axe kick.
- **Ultra Spin** `[SKILL, 15s, 9.5 drill + 4 slam]` — Spinning-claw arm lunges, drills through the torso, then slams the target into the ground.
- **Ultra Spin** `[VARIANT (Offload), none (15s with Übercharge), 4.5 drill + 4 slam]` — Offload-puppet performs Ultra Spin then self-destructs.
- **Boost On** `[SKILL, 16s, 4 + 6]` — Inverts to grab with the legs; on contact, boosts up and delivers a point-blank Ultra Cannon to the gut.
- **Ultra Cannon** `[SKILL, 17s, 10]` — Winds up a long blast of energy from the palm; user can briefly delay the shot.
- **Ultimate Cannon** `[HOLD VARIANT, 17s, 16.15]` — Holding ~1.3s switches to Mode: Albatross with a mouth cannon and combined-hand beam.
- **Ultra Cannon** `[VARIANT (Offload), 2s (17s with Übercharge), 7]` — Offload-puppet keeps Ultra Cannon wound up while aiming until commanded to fire by blocking.
- **Heat Emission** `[SKILL, 16s, 13]` — Boost On ejects the puppet forward, releasing scorching heat at the opponent.
- **Heat Emission** `[VARIANT, 16s, 9]` — Pressed again after the hop, ends the charge early with a smaller exhaust blast.
- **Heat Emission** `[VARIANT (Offload), none (16s with Übercharge), 16]` — Offload-puppet Boost On lunges forward; on contact, lifts and crashes the target into the original puppet then explodes.
- **Absolute** `[AWAKENING, -, sets HP to 117]` — Original puppet dismissed; the giant mech climbs out and roars. Duration: 8s-180s based on stored years.
- **Mode: Absolute** `[PASSIVE, 1.7s, 6 per M1]` — Each mech body part counts separately; staggers every ~17 damage; last two M1s unblockable.
- **Mode: Absolute** `[PASSIVE (Front Dash), 6s, 10]` — Front dash becomes a heavy kick that only targets stunned/ragdolled enemies.
- **Mode: Absolute** `[PASSIVE (Air Dash), 6s / 2s jump, 12]` — Airborne front dash becomes an AoE dropkick that launches enemies upward.
- **Mode: Absolute** `[PASSIVE (Death Drill), once per death, 10 + 5]` — At 0 HP, a puppet hops out and dives with a last-ditch drill; landing restores HP.
- **Energy Output** `[SPECIAL, -, -]` — Press to set a 5-segment output bar; higher level boosts the next move and its Awakening cost.
- **Miracle Cannon** `[SKILL, 8s (~5.5% Awakening), 20]` — Uses one year of cursed energy for a fiery blast that ignites targets.
- **Miracle Cannon** `[VARIANT (Energy Output 2+), 8s (~11% Awakening), 35]` — At Output 2+, consumes 2 years for a massive sphere; level 5 allows aim-tilt.
- **Pigeon Viola** `[SKILL, 8s (+5.5% per level, up to 39%), 5 per bullet (up to 35)]` — Three homing rainbow beams aimed at a target within 250 studs; +1 bullet per Output level.
- **Absolute Destruction** `[SKILL, 8s (+5.5% per level, up to ~28%), 4 per stomp + 5 + 20]` — Two years to rampage with ~3 stomps and a leap-slam; level 5 grants 4 extra stomps.
- **Absolute Destruction** `[AIR VARIANT, 7s (~5.6% Awakening), 20]` — Airborne, skips to the final slam directing forward.
- **Technique Charge** `[SKILL, 8s (+5.5% per level, up to ~28%), 20 (+1 per level) / 5 simple domain]` — Index-finger compartment deploys a technique-imbued projectile; level 5 grants a Simple Domain shatter charge.

## Head of the Hei (Naoya Zenin)

- **Projectionism** `[PASSIVE, 2s for dash, 8]` — M1 set deals 8 total (2 per M1); final M1 knocks back with evadable stun; second side dash replaces front dash.
- **Frame Freeze** `[PASSIVE, -, triples next hit]` — Moves apply Projection on the user or target; 100% Projection freezes them in a glass frame for 3 seconds.
- **Projection Sorcery** `[SPECIAL, 18s, -]` — Aiming within 50 studs, teleports the distance in a single second leaving an afterimage.
- **Projection Sorcery** `[SPECIAL VARIANT, 18s, 4 (12 on frame)]` — On a framed target, punches them out of the glass and fills their meter by 27%.
- **Projection Breaker** `[SKILL, 18s, 9]` — Back dash with melee i-frames into a spinning heavy kick that throws targets away; resets M1s.
- **Bleedout** `[SKILL, 20s, 5 stab + 3 per stack (up to 50.7 Hemorrhage)]` — Tanto stab applies 3 Bleed stacks; repeat applies Hemorrhage for passive HP drain.
- **Decisive Strike** `[SKILL, 18s, 6 + 8 barrage]` — Run forward and grab; on success, a swift flurry of blows that stuns the target.
- **Decisive Strike** `[VARIANT, 18s, 6 + 8 barrage]` — Repressing extends the rush by planning a longer trajectory.
- **Decisive Strike** `[VARIANT (Unblockable), 18s, 6 + 8 barrage]` — A third press extends the rush further and makes the jab unblockable.
- **Decisive Strike** `[AIR VARIANT, 18s, 6 + 8 barrage]` — Aiming at an airborne enemy locks on and travels to them for a sure hit.
- **Cursory Impact** `[SKILL, 16s, 7.5 smack + 4 sweep]` — Quick step to a back-handed smack followed by a propelling sweep.
- **Cursory Impact** `[VARIANT (Frame), 16s, 27.5]` — A framed target gets launched upward; user appears midair to grab and toss them.
- **Vengeance** `[AWAKENING, requires full bar, 10 transformation]` — On full Awakening, attempts a Bleedout-style strike; if interrupted by melee, dies and reincarnates as a vengeful curse. Duration: 60s, sets HP to 75.
- **Vengeance** `[VARIANT (Miss), requires full Awakening, 5 + 3 per stack (up to 50.7)]` — Without interruption, loses 50% Awakening; the stab still applies regular Bleedout effects.
- **Vengeful Cursed Spirit** `[PASSIVE, -, 14 (full M1 set)]` — Faster movement; red-tendril M1s pull user closer and charge Projection by 10%.
- **Acceleration** `[SPECIAL, 9s, 7]` — Spins rapidly accumulating speed, damaging the area and increasing walk speed; fills a momentum bar.
- **Top Speed** `[SKILL, 14s, 5 (+2.5 per Acceleration, up to 15)]` — Jet boost forward crashing into players; scales melee → bullet → explosion type with Acceleration stacks.
- **Top Speed** `[VARIANT (Flash Freezing), puts both on cooldown, +0.6 per frame]` — Traveling through Flash Freezing frames extends flight ~3s and adds damage.
- **Flash Freezing** `[SKILL, 18s, 7 per frame]` — Freezes air into 15 glass frames; on hit, frames shatter and chain-react.
- **Tendril Grab** `[SKILL, 10s, -]` — Plunges arm underground; tendrils surface to immobilize all enemies within 25 studs for ~1s.
- **Time Cell Moon Palace** `[DOMAIN, 60s (ends awakening), depends on speed]` — Domain forces opponents to abide by 24 FPS at a cellular level; actions freeze and shatter for speed-scaled damage. Duration: 25s.

## Salaryman (Kento Nanami)

- **Blunt Cleaver** `[COSMETIC, 1.7s, 14]` — Cloth-wrapped cleaver and yellow black-dotted tie; held forward when blocking.
- **Ratio Point** `[SPECIAL, 5s, -]` — Aims at a target within 70 studs and starts a QTE; pressing on the 7:3 line marks them for 4.5s with damage boosts.
- **Ratio Black Flash** `[PASSIVE, -, 8]` — Landing the final M1 or an uppercut on a marked enemy adds a precise Black Flash with increased damage and knockback.
- **Cleaving Whirlwind** `[SKILL, 16s, 14]` — Spins forward with the cleaver into a downward slash that knocks enemies back.
- **Cleaving Whirlwind** `[VARIANT (Ratio), 16s, 18]` — On a Ratio-marked target, deals more damage and staggers right arm for 7s if interrupting.
- **Severance Kick** `[SKILL, 14s, 12]` — Right-foot kick with cursed-energy follow-up that pushes the target with evadable stun.
- **Reverse Kick** `[VARIANT, 14s, same as Severance Kick]` — Walking backwards swings the leg the other direction; interrupting stuns in place.
- **Blunt Cut** `[SKILL, 16s, 9 (14 held)]` — Charges a quick swing then flashsteps forward to slice.
- **Blunt Cut** `[VARIANT (Ratio), 16s, 15 (20 held)]` — On a Ratio-marked target, more damage and left-leg stagger for 7s if interrupting.
- **Cross Cut** `[AIR VARIANT, 16s, 9]` — Airborne, a downward dive with weapon aimed forward; on hit, an unblockable swing follows.
- **Cross Cut** `[AIR VARIANT (Ratio dive), 16s, 15]` — Marked-target dive does more damage and disables dashes for 7s.
- **Cross Cut** `[AIR VARIANT (Ratio slash), 16s, 15]` — Marked after the dive, the slash gets enhanced and staggers the right arm.
- **Stabilize** `[SKILL, 12s, 8]` — Thrusts the tool into the opponent's stomach then twirls it, stunning both parties briefly.
- **Stabilize** `[VARIANT (Interrupt), 12s, 8 (12 on interrupt)]` — Interrupting causes the target to cough up blood, with longer stun.
- **Overtime** `[AWAKENING, -, +25 HP heal]` — "How unfortunate. I'm now working overtime." Cursed-energy output rises to 110-120%. Duration: 60s.
- **Working Overtime** `[PASSIVE, 1.7s, 18]` — Higher M1 damage; destroys the environment.
- **Wall of Stone** `[COSMETIC, -, -]` — While blocking, the user does not use the cleaver; attacks feel like hitting a stone wall.
- **Ratio Breaker: 1/4** `[SKILL, 22s (none with Ratio), 10]` — Stage 1: a downward slash plus a cursed-energy right hook with a Black Flash.
- **Ratio Breaker: 2/4** `[SKILL VARIANT, 22s (none with Ratio), 7 (14 with Ratio)]` — Stage 2: a straight punch Black Flash; marked target gets right-arm stagger if interrupting.
- **Ratio Breaker: 3/4** `[SKILL VARIANT, 22s (none with Ratio), 10 (20 with Ratio)]` — Stage 3: a long hop and heavy slam crushing the area with black sparks.
- **Ratio Breaker: 4/4** `[SKILL VARIANT, 22s, 35 (105 with Ratio) + 10 followup]` — Stage 4: an enveloping twirl into a Black Flash swipe with cutscene; marked target damage triples.
- **Sharpen** `[SKILL, 22s, 15 (23 with Ratio)]` — Slides while sharpening the tool on the ground, then slices upward to launch enemies.
- **Erosion** `[AIR VARIANT, 22s, 15]` — Airborne, floats briefly for an overhead cleaver slam that grounds those in front.
- **Erosion** `[AIR VARIANT (Ratio aim), 22s, 15]` — A 7:3 aim circle marks the environment's weak point, expanding the impact and launching targets up.
- **Interrogate** `[SKILL, 20s, 4 + 10]` — Holsters the tool to catch and gut-punch the target; marked target gets right-arm stagger.
- **Interrogate** `[VARIANT (Follow-up), 20s, 4 + 16]` — Chases after the punch for a second grab and cursed-fist blow; marked target gets left-leg stagger.
- **Collapse** `[SKILL, 22s, 5 + 7 (15 with Ratio)]` — Leaps for a heavy slam that marks the environment and suspends cursed debris in the air.
- **Collapse** `[VARIANT (Detonate), 22s, 40]` — Pressed again, the suspended debris crashes downward; auto-detonates after 8s.

## Disaster Plants (Hanami)

- **Arm Wrap** `[BASIC ATTACK PASSIVE, -, 4]` — Left arm wrapped, single right-arm M1; uppercut becomes a giant thorn surging from the ground.
- **Disaster Root** `[FRONT DASH PASSIVE / VARIANT, 10s, 4 (cannot kill)]` — Front dash stomps to call a root that pulls a target from 60 studs away.
- **Flower Field** `[SPECIAL, 14s, 4]` — Spreads flowers 25 studs ahead for 2s; attacking inside the field stuns the attacker and disables dashes for 5s.
- **Root Swarm** `[SKILL, 15s, 10]` — Plunges arm into the ground to release a line of intangible roots that knock enemies above them upward.
- **Root Swarm** `[VARIANT, 15s, 10]` — Facing a gap, roots become tangible and act as a 4-second bridge.
- **Surging Thorns** `[SKILL, 15s, 15]` — Two wooden balls launch sharp branches that push enemies in a 35-stud range toward 3 deadly thorns.
- **Bud Shot** `[SKILL, 20s, 8]` — Chucks 2 cursed buds 70 studs forward that halt the target's ragdoll-cancel gain for 7s.
- **Defense Response** `[SKILL, 15s, 12]` — Reclines with melee i-frames then surges forward with a heavy arm swing that knocks the target backward.
- **Defense Response** `[AIR VARIANT, 15s, 5]` — Air variant: sharp roots rise 25 studs ahead and launch anyone above them skyward.
- **Unwrap** `[AWAKENING, -, +30 HP heal]` — Awakening only heals the user (character still in development).

## True Cannon (Ryu Ishigori)

- **Overheat** `[PASSIVE, -, -]` — Builds heat over time; at 100% Overheat all Cursed Energy Discharge attacks are disabled except Appetizer.
- **Cursed Energy Discharge** `[PASSIVE / VARIANT, -, 14 (10 at 90% Overheat)]` — 3-hit M1 combo; under 90% Overheat, the 3rd M1 shoots an extended-range ray.
- **Restyle** `[SPECIAL, 17s, -]` — "Sweet!" pose cools the user by 60% Overheat; from 100%, slower hair-comb resets to 0%.
- **Granite Blast** `[SKILL, 0.5s, 7]` — A long concentrated-energy beam travels 78.5 studs forward; airborne enables 360° aim. Disabled at 100% Overheat.
- **Granite Blast** `[HOLD VARIANT, 0.5s, 14]` — Holding ~1.1s doubles damage, extends range to 100 studs, and makes it unblockable at double Overheat cost.
- **Fasting** `[VARIANT, 6s (front dash), 4]` — Granite Blast during front dash releases a beam that loops around the user before continuing.
- **Unsatisfied** `[SKILL, 20s, 18]` — Three quick blows ending in a back-clash that overpowers the target.
- **Second Helping** `[SKILL, 15s, 12]` — Dashes over a target within 70 studs with melee i-frames for a merciless launching slam.
- **Second Helping** `[VARIANT, 15s, 12]` — On a ragdolled airborne target, appears over them to ground-slam with increased windup.
- **Appetizer** `[SKILL, 18s, 16]` — Two quick forehead Granite Blasts then a vertical ray that ragdolls the target up and toward the user.
- **Appetizer** `[SPECIAL VARIANT, 18s, 8]` — Overheated, skips the mini-rays straight to the vertical ray that ragdolls the target away.
- **Every Last Drop.** `[AWAKENING, -, 105]` — Charges and fires a hyper-concentrated beam consuming all cursed energy; enters Awakening state if Overheat was 80%. Duration: 90s, +25 HP heal.
- **Decadence** `[PASSIVE VARIANT, ends with Awakening, 10]` — Critical-health damage drains Awakening instead of killing; no regen; constant 100% Overheat.
- **"What are you after?"** `[SKILL, 18s, 34]` — Slams the floor to bounce the target up, then a 360°-aimable lunge into a punch clash.
- **"I had no idea..."** `[SKILL, 15s, 20]` — Winds back a punch; on landing, both clash punches and the user wins.
- **"This is what dessert is like!"** `[SKILL, 20s, 36]` — Runs forward with a stunning kick; on landing, a cutscene exchanging several blows.

## Black Death (Kurourushi)

Upcoming character with mostly placeholder data on the wiki.

- **Festering Life Sword** `[PASSIVE, -, -]` — Passive (effect TBA). Wiki: "You have M1s. Probably."
- **TO BE ADDED** `[SPECIAL, TBA, TBA]` — Unknown special move.
- **TO BE ADDED** `[SKILL, TBA, TBA]` — Unknown skill.
- **TO BE ADDED** `[AWAKENING, -, TBA]` — Unknown Awakening move.

---

_All move names are exact in-game strings sourced from the JJS Fandom wiki._
