---
name: jjs-moveset
description: >
  Decode, edit, generate, and encode custom movesets for Jujutsu Shenanigans
  (JJS) Build Mode / Skill Builder. Handles the Base64+zstd import-code
  format end-to-end and uses real in-game block types, fields, states,
  effects, and built-in skill names. Use this skill whenever the user
  pastes a JJS import code, asks for a custom moveset, describes a
  character build, or mentions "Skill Builder", "moveset code", "JJS
  moves", or specific JJS abilities (Limitless, Blue, Red, Purple,
  Domain, Rapid Punches, etc.).
---

# JJS Moveset Skill

You are working with **Jujutsu Shenanigans** (Roblox) Build Mode import codes
produced by the in-game **Skill Builder**. An import code is:

```
JSON  --(utf-8)-->  bytes  --(zstd)-->  bytes  --(base64)-->  ASCII code
```

The JSON is an array of "skill slot" objects. Each slot's `DATA` field is
itself a JSON-encoded string holding the timeline. The codec (§1)
auto-expands `DATA` into nested objects on decode and re-stringifies
it on encode so you edit normal JSON.

---

## 0. WHEN INVOKED

Decide intent from the user's message:

| User says... | Intent | What to do |
|---|---|---|
| Pastes a long base64 code | **decode** | Run the codec, return the structured JSON |
| "edit/buff/nerf/change X in this moveset" | **edit** | Decode → modify the relevant slot → re-encode |
| "make/generate a moveset for Y" | **generate** | Build JSON from scratch using the catalogs in §6–§9 → encode |
| "encode this JSON" | **encode** | Run the codec, return the base64 code |
| "is this moveset balanced / what does it do" | **analyze** | Decode → walk slots → describe in plain English |

**Never hallucinate the encode step.** Always run the actual codec
(see §1 — write the script to disk and shell out). If `zstandard` is
not installed, say so up front — do not invent a fake base64 string.

### Companion skills (load these when relevant)

Two companion skills sit alongside this one and contain reference data
that is **not** duplicated here. Load them when their topic comes up;
don't try to invent the content they hold.

- **JJS Moves Catalog** — every built-in skill, special, awakening,
  passive, and variant for all 16 canonical characters, with per-move
  cooldown / damage / one-line description. **Use it when you need a
  `SKILL.MOVE` name, a `SPECIAL.SPEC` name, or want to *borrow a
  built-in move's animation* via `ANIM_USE` (per §7).** Every built-in
  character animation lives in the in-game animation picker — so any
  move from this catalog can be reduced to "just the animation" for a
  custom hitbox.
- **JJS Emotes Catalog** — every emote, idle, stance, and cosmetic
  animation in the game (~326 entries) with a one-line motion
  description. **Use it when you need an idle pose, pre-combat
  stance, cinematic gesture, or awakening idle** (Infinity hover,
  Bizarre, Honored, Insanity, Jackpot, Insane2, etc.). These
  animations are also indexed in the same in-game animation picker
  and can be plugged into `ANIM` blocks.

Both catalogs feed the same in-game animation picker that `ANIM_USE`
references — so an entry from either catalog can become an `ANIM`
block. The Moves Catalog *additionally* feeds the `SKILL` / `SPECIAL`
node lists. §6, §7, and §9 reference these catalogs throughout — load
the relevant one before naming a move or picking an emote.

---

## 1. THE CODEC (SELF-CONTAINED — WRITE IT, RUN IT)

The import-code format is **`JSON → utf-8 → zstd compression → base64`**.
Inside the JSON, each slot's `DATA` field is itself a JSON-encoded string
(a doubly-encoded blob); on decode you expand that string back into a
nested object so the slot becomes editable, and on encode you re-stringify
it back into the compact JSON the game expects.

**The skill is self-contained.** Don't look for a `.py` file next to
SKILL.md — there isn't one. Instead, **write the codec to disk** the
first time you need it in a session, then shell out:

### Codec — write to `jjs_codec.py` then run

Save this exact content to `jjs_codec.py` (or any filename you like)
in the working directory:

```python
#!/usr/bin/env python3
"""JJS Skill Builder import-code codec — JSON ⇄ base64(zstd(JSON))."""
import base64, json, sys, zstandard

def decode_file(src, dst):
    raw = open(src, "rb").read().strip()
    compressed = base64.b64decode(raw)
    ctx = zstandard.ZstdDecompressor()
    data = ctx.decompress(compressed)
    slots = json.loads(data)
    # Expand inner DATA string into a nested object per slot
    for slot in slots:
        if "DATA" in slot and isinstance(slot["DATA"], str):
            slot["DATA"] = json.loads(slot["DATA"])
    open(dst, "w", encoding="utf-8").write(
        json.dumps(slots, indent=2, ensure_ascii=False))
    print(f"Decoded {len(slots)} slot(s) -> {dst}")

def encode_file(src, dst):
    slots = json.loads(open(src, encoding="utf-8").read())
    # Re-stringify inner DATA per slot (the game expects a JSON string)
    for slot in slots:
        if "DATA" in slot and isinstance(slot["DATA"], dict):
            slot["DATA"] = json.dumps(
                slot["DATA"], separators=(",", ":"), ensure_ascii=False)
    raw = json.dumps(slots, separators=(",", ":"),
                     ensure_ascii=False).encode("utf-8")
    cctx = zstandard.ZstdCompressor(level=19)
    compressed = cctx.compress(raw)
    code = base64.b64encode(compressed).decode("ascii")
    open(dst, "w").write(code)
    print(f"Encoded -> {dst}  ({len(code)} chars)")

if __name__ == "__main__":
    cmd, src, dst = sys.argv[1], sys.argv[2], sys.argv[3]
    if cmd == "decode":
        decode_file(src, dst)
    elif cmd == "encode":
        encode_file(src, dst)
    else:
        print("Usage: jjs_codec.py decode|encode src dst")
        sys.exit(1)
```

Requirements: Python 3.x and the `zstandard` package
(`pip install zstandard`). Compression level 19 matches what the game
emits in real exports — don't change it; lower levels produce a
larger code that still imports but doesn't round-trip byte-equal with
the game's own export.

### Usage

```bash
# decode an import code into editable JSON
python jjs_codec.py decode  path/to/code.b64   out.json

# encode JSON back into an import code
python jjs_codec.py encode  path/to/build.json out.b64
```

Both subcommands require three positional arguments (`cmd src dst`) —
no stdin/stdout piping. Always write inputs to a file first, even if
the user pasted the code inline; codes are long, never pass them as
a shell argument.

The codec **auto-handles the inner `DATA` string**: on decode, each
slot's `DATA` becomes a real object; on encode, it is re-serialized
back into the compact JSON string JJS expects.

After encoding, **always verify by round-tripping**: decode your own
output and confirm it parses and the slot count/names match what you
intended. If verification fails, say so — do not present a broken code.

### If you absolutely can't run Python

You can still produce import codes by literally executing the
algorithm in another language: minify the JSON (no spaces, no trailing
commas), zstd-compress at level 19, base64-encode the result. Most
languages have both libraries available. The skill assumes Python is
fine; flag explicitly if the environment forces a different runtime.

---

## 2. FILE STRUCTURE

Top-level: **array** of slot objects (one per ability key/slot).

```jsonc
[
  {
    "K_NAME": "CHASE" | "MELEE" | "SPECIAL" | "SKILL" | "AWAKENING",
    "NAME":   "Dash" | "m1" | "Limitless" | ...,
    "COOLDOWN": 5,             // seconds; not used by MELEE/AWAKENING slots
    "KEY":      1 | 2 | 3 | 4 | 5 | 6 | 7 | 8, // NUMBER 1-8 ONLY — never a letter. SKILL slots only. Defaults: 1=Q, 2=E, 3=R, 4=T. NEVER write "Q", "E", "R", "T" — always the integer.
    "ADD":      false,         // SKILL slots: replace the base move if slot is occupied
    "DURATION": 60,            // AWAKENING only: awakening duration in seconds
    "DELAY":    0,             // AWAKENING only: pre-activation delay
    "DATA":     { "Line": [...], "Branch": {...}, "Req": [...], "Prop": [...] }
  },
  ...
]
```

Slot `K_NAME` values seen in real exports:

| Slot K_NAME | Purpose | Required fields |
|---|---|---|
| `CHASE` | Front-dash replacement | NAME, COOLDOWN, DATA |
| `MELEE` | M1 string entry (`m1`/`m2`/`m3`/`m4`) | NAME, DATA |
| `SPECIAL` | The character's right-click special | NAME, COOLDOWN, DATA |
| `SKILL` | Numbered ability (1–4) | NAME, KEY, ADD, COOLDOWN, DATA |
| `AWAKENING` | The whole awakening transformation | NAME, DURATION, DELAY, DATA |

A complete moveset usually contains: 1× CHASE, 4× MELEE (m1..m4), 1× SPECIAL,
4× SKILL (KEY 1–4) for base form, sometimes 4× more SKILL slots for the
awakened form (same KEY values, distinguished by the awakened state), and
1× AWAKENING.

---

## 2.5 COORDINATE SYSTEM, SCALE & SPACING (read before any spatial field)

Every `POSITION`, `SIZE`, `ROTATION`, `FORCE`, and `ALT POSITION` string in
the moveset format is `"x, y, z"` (comma-space separated). The axes are
**relative to the user's current facing**, not world-space:

| Axis | Positive | Negative |
|---|---|---|
| **X** | User's **LEFT** side | User's **RIGHT** side |
| **Y** | **Above** user (up) | **Below** user (down) |
| **Z** | **In front** of user (forward) | **Behind** user (backward) |

Some Roblox-world conventions invert X — when in doubt, trust this
table. It reflects hands-on testing by experienced builders.

### Unit: studs

The entire game runs in **studs**. One stud ≈ one integer unit. Use the
following reference scale **religiously** when sizing hitboxes, choosing
visual offsets, and writing velocity forces:

- **Character height:** ~5 studs (head crown to feet).
- **Character width / shoulder span:** ~1.35 studs.
- **Character depth (front-to-back):** ~1 stud.
- **Eye / head level:** ~Y +2.0 to +2.5 from the HumanoidRootPart.
- **Hand / arm height:** ~Y +1.0 to +1.5 from HumanoidRootPart, depending
  on pose. **Empirical rule:** to make a visual appear "in the hand"
  when `BODY PART` is set to `Right Arm` or `Left Arm`, the visual's
  `POSITION` usually needs to be ≈ `"0, -1, 0.15"` — the arm limb's
  pivot is at the shoulder, so offsets are reckoned from there.
- **Arm reach (forward):** ~1.5–2 studs of Z forward from HumanoidRootPart.

Common sizing reference points (these are the **defaults to lean
toward** — generous, not stingy):

| Effect / hitbox you want | Typical SIZE (studs) | Typical POSITION (studs) |
|---|---|---|
| Pinpoint punch / stab | `"5, 5, 6"` | `"0, 0, 3"` |
| Standard M1 swing | `"8, 8, 8"` (preferred default) | `"0, 0, 4"` |
| Wide horizontal sweep | `"12, 7, 8"` | `"0, 0, 4"` |
| Stomp / ground slam | `"10, 5, 10"` | `"0, -1, 1"` |
| Cone forward AoE | `"9, 7, 10"` | `"0, 0, 5"` |
| Big finisher AoE | `"18, 8, 12"` | `"0, 0, 5"` |
| Wall / barrier | `"22, 10, 3"` | `"0, 2, 4"` |
| Long beam / lance | `"5, 5, 100"` | `"0, 0, 50"` — **half** of size Z |

**General rule: when in doubt, go BIGGER.** A character is 5 studs
tall and ~1.35 wide — a `6, 6, 6` hitbox is barely larger than the
character itself, which leaves zero positional margin for a moving
target. **`8, 8, 8` is a safer default for a standard hit; `10, 10, 10`
is fine and even desirable for sweeping/finisher hits.** Only scale
down to `5, 5, 6` or smaller when the design *explicitly* calls for
a precision strike. If the user asks for a smaller hitbox, honor that —
otherwise default to the larger end of the table above.

### Z-position convention for stretched hitboxes/projectiles

When you make a hitbox or projectile that's long in the Z axis (a beam,
a lance, a wall stretching forward), the Z **position** should equal
**half the Z size** — that way the hitbox starts at the user and
extends forward by `size_z` studs. A 100-stud-long beam with
`POSITION: "0, 0, 100"` starts 50 studs in front of you and reaches
150 studs out, which almost certainly misses everything close-up.
Correct: `POSITION: "0, 0, 50"`, `SIZE: "x, y, 100"`.

### Velocity scale (FORCE magnitudes)

- **Tiny reposition** (a step): `~5` studs/s · `TIME ~0.1`
- **Standard knockback** (M1 hit): `~15–20` · `TIME 0.2`
- **Sendback / launch** (m4 finisher): `~25–40` · `TIME 0.2–0.3`
- **Dash (regular CHASE)**: `~50` · `TIME 0.1` (community reports the
  base-game forward dash matches `"0, 0, 50"` at `TIME 0.1`)
- **Long mobility / sustained sprint**: `~25–40` · `TIME 0.5–1.0`
- **Hard cap (jitter zone):** `|FORCE| > ~1000` causes the player to
  visibly jitter — never exceed.

### Velocity scale (PROJECTILE.SPEED + TIME math)

`SPEED` is studs per second; `TIME` is total lifetime in seconds.
**Total travel distance = `SPEED * TIME`.** If you want a projectile
to cover 25 studs in 0.1 seconds, you can't just write `SPEED: 25, TIME: 0.1`
(that gives 2.5 studs). Worked example: to cross 25 studs in 0.1s,
divide distance by time — `25 / 0.1 = 250` — and set `SPEED: 250,
TIME: 0.1`. Simpler version: pick a normal `SPEED` and dial `TIME`
to give the distance you want.

---

## 3. THE `DATA` (TIMELINE) OBJECT

Inside each slot, `DATA` is an object with four sections:

```jsonc
{
  "Line":   [ ... timeline blocks in order ... ],
  "Branch": { "branchName": { "Req": [...], "Line": [...] }, ... },
  "Req":    [ ... conditions for the whole move to fire ... ],
  "Prop":   [ ... properties / passive flags ... ]
}
```

- **`Line`** is the main timeline. Blocks execute top-to-bottom; only
  `WAIT` and the `TIME` fields of other blocks introduce delay.
- **`Branch`** is a dictionary of named sub-timelines. They fire when
  another block references them (`HITBOX.BRANCH TARGET`, `TAG.BRANCH`,
  `COUNTER.BRANCH`, `BRANCH` block, etc.).
- **`Req`** is an array of *gates* (`AIR`, `HP`, `ULT`, `JUMP`, `BAR`...);
  if any fails, the slot cannot be activated. Each Req has a `FLIP` boolean
  for inversion.
- **`Prop`** is an array of property tags (e.g. `AWK` for "hide in
  awakening" mode, applied to base SKILL slots that should disappear
  while awakened). Properties apply to the whole slot. **See §5.19
  for the corrected AWK ↔ AWK2 mapping — earlier drafts had it
  inverted.**

---

## 4. EXECUTION ORDER & VELOCITY/ANIM INTERACTION

Blocks on the timeline run **top-to-bottom in the order they appear**.
Each block is buffered from the next by ~0.01s by default — only `WAIT`
blocks and the `TIME` fields on other blocks introduce meaningful delay.

There is no documented "tick reorder" — the order you write blocks in
is the order they fire. So if you want `STATE` (lock the user) before
`ANIM`, put `STATE` first.

Important behavioural rules (used by analyze and generate):

- **`VISUAL` blocks can cut animations** (wiki, Current Issues). The
  fix is to insert a `WAIT` matching the animation's runtime before the
  next `VISUAL`. Some custom builders also report `VELO` interfering
  with the playing `ANIM`, especially large upward forces — when in
  doubt, insert a `WAIT` first.
- `HITBOX` placed **before** the corresponding `ANIM` swing reads as a
  phantom hit; place it after the wind-up's first ~30%.
- `LOOP` repeats the previous `LOOP BACK` blocks `LOOP AMOUNT` times.
  When `HOLD` is true it loops while the key is held — never combine
  HOLD=true with no exit (move becomes uninterruptible spam). Note:
  `HOLD` is intended for built-in hold-skills inside the loop (e.g.
  Star Rage's Mass Build-Up); it does **not** work for most custom
  hitboxes or arbitrary moves.
- `SFX` is purely audio; most sounds need `VOLUME` ≥2 to be audible
  in-game.

---

## 5. BLOCK REFERENCE

Every block has `K_NAME: "<TYPE>"`. Use only the types and fields below
— **do not invent block types or fields.** Field names are case- and
space-sensitive (`"BRANCH TARGET"` with a space, not `BRANCH_TARGET`).

### 5.1 `ANIM` — play an animation

```jsonc
{
  "K_NAME": "ANIM",
  "ANIM_USE": [group, index],   // animation reference: [category, slot] from the in-game picker
  "PREVIEW": [start, end],      // editor preview window in seconds
  "SPEED": 1.0,
  "LOOPED": false,
  "FADE IN": 0.1,
  "FADE OUT": 0.1,
  "LAST HIT": -1                // seconds; -1 = no time limit on hit-target retargeting
}
```

`ANIM_USE` is a `[group, index]` pair into the in-game animation
picker. The picker contains **every animation in the game** — skill
animations, special animations, M1 (melee) animations, awakening
animations, **and** every emote. That means **any move from the JJS
Moves Catalog skill can be used purely as an animation** for one of
your custom moves: borrow Limitless's pose, Hollow Purple's wind-up,
Manji Kick's spin, etc. — and build your own hitboxes/visuals around
it. When you do this, name the source move in the design brief so the
user knows which slot in the picker to pick.

Real exports re-use a small set; common pairs include
`[1,3] [1,4] [1,5] [1,6] [1,9] [1,12] [2,7] [2,8] [2,20] [3,3] [11,4]
[15,25] [17,2]`. When generating fresh, copy pairs from a decoded
sample — do not invent indices.

### 5.2 `VELO` — apply velocity

```jsonc
{
  "K_NAME": "VELO",
  "FORCE": "x, y, z",           // STRING, comma-space separated, studs
  "TIME": 0.3,
  "FADE": true,                 // ease out instead of hard stop
  "TRACK": false,               // re-orient with the user's facing each tick
  "RELATIVE FROM BRANCH": false,
  "RAGDOLL": 0,                 // seconds of ragdoll applied (0 = none)
  "TRUE RAGDOLL": false,        // disables ragdoll-cancel during RAGDOLL
  "LAST HIT": -1                // -1 (DEFAULT) = velocity applied to the USER.
                                //  1 (or any positive seconds) = velocity applied to the
                                //  TARGET hit in the last 1 second.
}
```

> **⚠ THE #1 m4 BUG: VELO defaults to LAST HIT: -1 (USER). ⚠**
>
> If you write a `VELO` block at the end of an m4 (or any move that's
> supposed to knock back / launch / ragdoll the target), and you don't
> set `LAST HIT: 1`, the velocity is applied to the **USER** instead.
> The user gets launched, the target stands still, the move looks
> nonsensical. **Confirmed from playtest.**
>
> Whenever the design intent is "knock the target back / up / down":
> ```jsonc
> { "K_NAME": "VELO", "FORCE": "0, 30, 5", "TIME": 0.25,
>   "RAGDOLL": 0.6, "LAST HIT": 1 }   // <-- LAST HIT: 1 is mandatory
> ```
>
> Whenever the design intent is "the user dashes forward / leaps / jumps":
> the default `LAST HIT: -1` is correct — leave it.

**Ragdoll application:** `RAGDOLL > 0` puts whoever the velocity targets
(user or target, see above) into ragdoll for that many seconds. Standard
ragdoll for m4 finishers and launches: `0.5–0.8s`. `TRUE RAGDOLL: true`
prevents the ragdolled player from using ragdoll-cancel during that
window — useful for combo lockdown.

### 5.3 `HITBOX` — damage volume

```jsonc
{
  "K_NAME": "HITBOX",
  "POSITION": "0, 0, 4",        // relative to user, studs
  "SIZE":     "6, 6, 6",
  "ROTATION": "0, 0, 0",
  "DAMAGE": 8,
  "STUN":   0.5,
  "STUN ANIM": false,           // wiki default: off (toggle on for the default hurt anim)
  "ATTACK TYPE": "Melee" | "Bullet" | "Explosion" | "Swarm" | "Domain",
  "BLOCKABLE": true,
  "360 BLOCK": false,
  "CANCEL ENEMY": true,
  "CLEAR KNOCKBACK": false,
  "CAN KILL": true,
  "HIT RAGDOLL": false,
  "HIT USER": false,            // CAUTION: HIT USER + CAN KILL = true can kill the USER outright,
                                //   unlike Add Health which is HP-clamped. Combine only if intentional
                                //   (e.g. a sacrifice/cost mechanic).
  "SINGLE TARGET": false,
  "DEBREE": 0,                  // (sic: spelled DEBREE in real exports) >0 lets it harm domains
  "PREVIEW": [start, end],
  "BRANCH": "...",              // run on user if anyone is hit
  "BRANCH TARGET": "...",       // run on the hit target
  "BRANCH FINISHER": "...",     // run if the hit kills
  "PROJECTILE TAG": "..."       // attach to a same-tag PROJECTILE
}
```

**Counterability:** `Melee` is countered by everything, `Bullet` by some,
`Explosion`/`Swarm`/`Domain` are uncounterable. Don't mark every move
`Explosion` — that breaks counterplay (see §10).

**Sizing & position convention:** see §2.5. The format's *technical*
default of `SIZE: "6, 6, 6"` is **too small for most uses** — it's
barely larger than the character (5×1.35×1 studs), so a moving target
slips out of it constantly. **Use `SIZE: "8, 8, 8"` as your real
default for a standard hit, and lean larger (10+) for sweeps and
finishers.** Only ship `6, 6, 6` (or smaller) when the design
explicitly calls for a precision pinpoint. For long hitboxes (beams,
lances), set `POSITION.z = SIZE.z / 2` so the hitbox extends from the
user instead of starting far away. For ground stomps, use `POSITION.y`
slightly negative (e.g. `"0, -1, 1"`) so the hitbox sits at foot
level. For overhead strikes, raise `POSITION.y` (e.g. `"0, 2, 3"`).

**The "Debree" field controls map destruction AND domain damage.**
Any value `> 0` destroys breakable terrain in the hitbox AND lets the
move harm domains. Value `-1` deletes terrain *without* debris
particles. `0` (default) leaves terrain alone and cannot harm domains.
**Almost every offensive hitbox should set this** — the lack of
debris particles makes hits feel weightless.

The non-zero value also controls **debris fidelity** (size/density
of the destruction effect). Per playtest:

| `DEBREE` value | Use for |
|---|---|
| `4` | Standard / "everything" — the safe default for any normal-sized hitbox. |
| `6` | Big hitboxes — wide sweeps, sendback finishers, AoE moves. |
| `8` | GIGANTIC hitboxes — beam projectiles, screen-wide AoE, ground-slam impact craters. |
| `10+` | Insane / debris-spam — multi-hit chains, very large hitboxes. Higher values reduce lag (fewer fine particles) and read better at scale. |
| `0` (default) | Never destroys terrain, can't harm domains. Use only for utility hitboxes that aren't supposed to look impactful. |
| `-1` | Deletes terrain *silently* (no debris). For stealth / clean-erase moves. |

When unsure, use `DEBREE: 4` — it's almost always right for an M1 or
small SKILL hit. Bump up for finishers and large hitboxes.

**Single Target = the closest entity wins.** When `SINGLE TARGET: true`,
the hitbox affects exactly the entity closest to the hitbox's center.
Set this on precision grabs and finishers; leave it off for AoE moves.

**`BRANCH` vs `BRANCH TARGET` vs `BRANCH FINISHER`:**
- `BRANCH` (string, name in `DATA.Branch`): runs **on the user** when
  at least one target is hit.
- `BRANCH TARGET`: runs **on the target** when at least one is hit.
  Used for inflicting status, stagger, bleed, lock-on.
- `BRANCH FINISHER`: runs **on the user** specifically when the hit
  drops the target to 0 HP (kill). Used for kill-confirm cinematics.

### 5.4 `VISUAL` — cosmetic effect

`VISUAL` is the heaviest block by field count, and the most expressive
node in the entire format — it carries 95% of the *identity* of every
move (the slash arcs, the cursed-energy aura, the cinematic flash, the
mesh weapon, the screen tint, etc.). Every field has a
default-but-not-always-zero meaning, and several pairs (`COLOR` /
`ALT COLOR`, `OPACITY` / `ALT OPACITY`, `SIZE` / `ALT SIZE`,
`POSITION` / `ALT POSITION`, `ROTATION` / `ALT ROTATION`) form a
*start → end* interpolation pair that the easing functions tween over
`TIME` seconds.

**Who sees the visual:** By default, a `VISUAL` block spawns **on the
user**. The only exceptions are:
1. The visual is inside a branch invoked via `BRANCH TARGET` (a target
   was hit and is now running the branch on themselves) — then the
   visual appears on the target. Use `RELATIVE FROM BRANCH: true` if you
   want it to stay relative to the user instead.
2. The visual has a `PROJECTILE TAG` matching a live `PROJECTILE` — then
   it rides on the projectile.
3. The visual has `LAST HIT ≥ 0` AND the user has recently hit a target
   — then it uses the target's transform.

This matters because newer builders sometimes try to make a hit-flash
"appear on the target" by setting POSITION fields — that doesn't work
unless one of the three conditions above is met. To put a hit-flash on
the target, either invoke the visual from a `BRANCH TARGET` branch,
attach it to a projectile, or use `LAST HIT: <TIME>` to track the
last-hit target.

```jsonc
{
  "K_NAME": "VISUAL",
  "EFFECT": "Mesh",             // see catalog below
  "AMOUNT": 1,                  // count of instances, OR special meanings:
                                //   - EFFECT="Mesh"        -> Roblox Mesh asset ID
                                //   - EFFECT="Field Of View" -> view distance (negative zooms IN)
  "TEXTURE": "<asset id>",      // only compatible with Overlay, Billboard, Mesh
  "COLOR":     "255, 255, 255", // tint at spawn (defaults 255,255,255)
  "ALT COLOR": "255, 255, 255", // tint at end / secondary color (effect-dependent)
  "OPACITY":     0,             // 1 = fully invisible, 0 = fully visible
  "ALT OPACITY": 1,             // opacity at end of TIME
  "POSITION":     "0, 0, 0",    // relative to user (or projectile if tagged)
  "ALT POSITION": "0, 0, 0",    // position at end of TIME
  "ROTATION":     "0, 0, 0",    // roll, yaw, pitch
  "ALT ROTATION": "0, 0, 0",
  "SIZE": 1,                    // multiplier; default 1
  "ALT SIZE": 1,
  "TIME": 0.5,                  // total duration; also paces the alt-pair tween
  "LAST HIT": -1,               // for >=0 seconds, the effect uses the target's
                                //   transform instead of the user's. Set equal
                                //   to TIME for clean target-tracking effects.
  "BODY PART": "HumanoidRootPart",   // anchor limb
  "VISUAL TAG": "nil",          // pair with EFFECT="Cancel" to remove all
                                //   tagged effects on the same limb
  "PROJECTILE TAG": "nil",      // attach effect to a same-tag PROJECTILE
  "RUN ON SERVER": false,       // server-sided render = visible to all,
                                //   harder to override, visible to newly-joined
                                //   players. Use for *important* effects only.
  "CANCEL ON INTERRUPT": true,  // remove effect if the move is interrupted
  "RELATIVE FROM BRANCH": false,// if invoked via BRANCH TARGET, stay relative
                                //   to user (true) vs. target (false)
  "EASING STYLE": "Linear",     // Linear | Sine | Quad | Cubic | Exponential | Back
  "EASING DIRECTION": "Out"     // In | Out | InOut
}
```

### Field-by-field deep dive (what each field actually does)

- **`EFFECT`** — picks which visual primitive to spawn. The full list and
  per-effect description follow below; an unknown effect silently no-ops.
- **`AMOUNT`** — usually the *number of instances* of the effect to spawn
  in a single block. **A higher `AMOUNT` is the single biggest dial for
  making a visual look concentrated/exaggerated** — e.g. one `Cursed Energy`
  particle is a thin wisp; `AMOUNT: 30` is a roaring aura. Boost it for
  power moments. Two effects have a different meaning for `AMOUNT`:
  - `EFFECT: "Mesh"` → `AMOUNT` is the **Roblox Mesh asset ID** (a number,
    not a count). One mesh per block.
  - `EFFECT: "Field Of View"` → `AMOUNT` is the **viewing distance**;
    negative zooms toward the face, positive zooms away.
  - `EFFECT: "Blood"` → `AMOUNT` is the **number of blood drops**, and
    `SIZE` is the random radius (in studs) those drops scatter around the
    user.
  - `EFFECT: "Screen Color"` → `AMOUNT` between `0.01` and `1` controls
    opacity (lower = more transparent). **Negative `AMOUNT` inverts colors**
    — the canonical Black-Flash-style screen-invert is a Screen Color with
    a negative amount.
- **`TEXTURE`** — Roblox image/decal asset ID. Only compatible with
  `Overlay`, `Billboard`, and `Mesh` effects; ignored everywhere else.
- **`COLOR`** — the start tint (RGB string, `"R, G, B"`, each 0–255).
  Default `"255, 255, 255"` (white). For canon cursed-energy palettes see
  the table further down.
- **`ALT COLOR`** — the color the effect **fades to** over `TIME`. For
  most effects this tweens linearly between `COLOR` and `ALT COLOR`. For a
  few effects with a "secondary color" slot (e.g. some particle effects),
  it's a second tint rather than a fade target — behavior is
  effect-specific.
- **`OPACITY`** — start transparency. **0 = fully visible, 1 = fully
  invisible.** (Counter-intuitive — high opacity = invisible.) Useful
  intermediate values: `0.25` heavy, `0.5` half, `0.75` faint.
- **`ALT OPACITY`** — end transparency. Pair with `OPACITY: 0` and
  `ALT OPACITY: 1` for a smooth fade-out, or the reverse for a fade-in.
  Setting `ALT OPACITY > 1` (e.g. `10`) is a known trick to over-saturate
  certain effects (see the cartoony explosion preset).
- **`POSITION`** — `"x, y, z"` offset from the chosen `BODY PART`, in
  studs (see §2.5 for scale). Default `"0, 0, 0"` (at the limb origin).
- **`ALT POSITION`** — end position. Effect interpolates from `POSITION`
  to `ALT POSITION` over `TIME`. **For two effects (`Dismantle` and
  `Mass Hit`), this isn't an interpolation — it's a required field that
  defines the second endpoint of a line/path.** Without `ALT POSITION` set,
  Dismantle and Mass Hit don't render at all. Community workaround for
  Dismantle: set `ALT POSITION: "0, 0, 0.01"` to force visibility even
  for a "stationary" use. For Mass Hit, set `POSITION: "0, 0, 0.01"`
  (note: Mass Hit's rotation is then controlled by `ALT POSITION` rather
  than `ROTATION`, which is a separate quirk).
- **`ROTATION`** — start orientation as `"roll, yaw, pitch"`, each in
  degrees. Roll = cartwheel (around Z), Yaw = turn (around Y), Pitch =
  somersault (around X). Relative to the user's facing.
- **`ALT ROTATION`** — end orientation. Interpolates from `ROTATION` to
  `ALT ROTATION` over `TIME`. Setting them equal = no spin. Setting them
  far apart = the effect rotates over its lifetime (good for slash arcs
  swinging in real time).
- **`SIZE`** — start size multiplier. Default `1`. Note: for meshes and
  some big effects, `1` is huge — you may need `0.1` or even smaller.
- **`ALT SIZE`** — end size *as a multiplier of the starting `SIZE`*,
  **not an absolute size.** This is the most-misunderstood field in the
  whole block:
  > If `SIZE = 25` and `ALT SIZE = 2`, the final rendered size is `50`
  > (25 × 2), not `2`. To make the effect *end* at literal size `2`, you
  > would need `ALT SIZE = 2 / 25 = 0.08`.
  When you want a "grows by 4×" effect: `ALT SIZE: 4`. When you want a
  fixed final size, compute `desired_size / SIZE`.
- **`TIME`** — total duration the effect plays, in seconds. Also paces
  every `ALT *` interpolation. Most particle effects despawn after
  `TIME`; meshes/overlays/billboards usually disappear too.
- **`LAST HIT`** — `-1` (default) means the effect uses the **user's**
  transform. Any value `≥ 0` means: for that many seconds after the
  user last landed a hit, the effect uses the **target's** transform
  instead. Set `LAST HIT == TIME` for a clean "lock the visual onto the
  target for the whole effect" behavior.
- **`BODY PART`** — anchor limb (wiki UI calls it "Limb"). One of
  `HumanoidRootPart` (default, invisible torso-ish anchor),
  `Head`, `Torso`, `Right Arm`, `Left Arm`, `Right Leg`, `Left Leg`.
  Effects anchored to a limb move with that limb during the playing
  animation (i.e. they obey arm-swing in melee strings). To bypass anim
  influence entirely, use `HumanoidRootPart`. **Tip:** for arm-attached
  visuals (sword glow, hand-blast charge), most effects look right at
  `POSITION: "0, -1, 0.15"` since the arm's pivot is at the shoulder.
- **`VISUAL TAG`** — string label. The corresponding `EFFECT: "Cancel"`
  block (same `BODY PART`, same `VISUAL TAG`) will remove this effect on
  command. Use for aura blocks you want to extinguish at a checkpoint.
- **`PROJECTILE TAG`** — if matches a live `PROJECTILE.PROJECTILE TAG`,
  this visual rides on the projectile (instead of the user). Combine with
  Beam/Cylinder/Mesh effects to make a "moving body with a moving aura".
- **`RUN ON SERVER`** — `false` (default) renders the effect client-side.
  `true` runs it server-side: visible to all players (even newly joined
  ones), survives load lag, but is harder to override and costlier. Use
  for *important* aura/identity effects (Awakening aura, equipped weapon
  mesh) only — never for routine impact flashes.
- **`CANCEL ON INTERRUPT`** — `true` (default) removes the effect if the
  move gets interrupted (stunned out, hit-cancelled, etc.). Turn off for
  effects that should persist even after the move ends (a poison cloud
  left behind, a planted sigil).
- **`RELATIVE FROM BRANCH`** — only matters if this visual is invoked via
  a `BRANCH TARGET` branch. `false` (default) → visual is relative to the
  target. `true` → stays relative to the **user**.
- **`EASING STYLE` / `EASING DIRECTION`** — math curve for tweening every
  `ALT *` interpolation. Catalog and shapes below.

**Allowed `EFFECT` values** (the wiki's verbatim list — invalid effects
silently no-op): `Cancel`, `Overlay`, `Shake Light`, `Shake Medium`,
`Shake Heavy`, `Clash`, `Melee Trail`, `Beams`, `Beam`, `Wind Streak`,
`Wind Expand`, `360 Wind`, `Wind Ring`, `Whirl Slash`, `Slash`, `Cleave`,
`Dismantle`, `Star`, `Star Outline`, `Mesh`, `Light`, `Glow`, `Billboard`,
`Camera`, `Circle Glow`, `Flames`, `Distortion`, `Field Of View`,
`Cylinder`, `Black Flash`, `Energy Sparks`, `Sparks`, `Cursed Energy`,
`Rough Energy`, `Blood`, `Block`, `Ring`, `Burst`, `Mass Hit`, `Wedge`,
`Screen Color`, `Sphere`, `Weak Lightning`, `Shine`, `Visibility`.

### Per-effect visual catalog (what each EFFECT actually looks like)

Each entry below describes how the effect renders, the fields that
matter most for it, and when to reach for it. Visual descriptions
marked **[UNVERIFIED]** are educated guesses pending a user-supplied
reference. When the user provides a description, replace these.

**Camera / screen / utility effects:**

- **`Cancel`** — Not visible. Erases all prior visuals on the same
  `BODY PART` matching the same `VISUAL TAG`. Use to gracefully end a
  long-running aura or charge effect at a specific moment in the timeline.
- **`Camera`** — Repositions the user's camera to a forced position and
  rotation for `TIME` seconds. Only `POSITION` / `ALT POSITION`,
  `ROTATION` / `ALT ROTATION`, `TIME`, and `LAST HIT` matter; colors and
  sizes are ignored. Great for cinematic awakening sequences or to force
  a dramatic side-on shot on a finisher.
- **`Field Of View`** — Zooms the user's camera. `AMOUNT` is the view
  distance: positive zooms **out** (wider FoV / pulls camera back),
  negative zooms **in** (towards the face — classic anime-impact zoom).
  Smooth with `ALT *` fields? Actually `AMOUNT` is a single scalar here,
  not a pair — chain two FoV blocks with different AMOUNTs to ramp.
- **`Screen Color`** — Full-screen tint. `COLOR` / `ALT COLOR` set the
  tint. `AMOUNT` between `0.01` and `1` is opacity (lower = more
  transparent). **Negative `AMOUNT` inverts the screen colors** — this
  is how custom Black Flashes / impact-frame visuals are made.
- **`Shake Light`**, **`Shake Medium`**, **`Shake Heavy`** — Camera shake
  of the named intensity. Only `TIME` matters. Use Light for footsteps /
  light impacts, Medium for M1-end and decent hits, Heavy for finishers
  / domains / impact-craters. Don't shake-spam — players hate it.
- **`Visibility`** — Toggles the user's mesh visibility. **[UNVERIFIED:
  please confirm — does it hide the whole user, only the body, or fade
  with OPACITY? Does it tie to TIME or persist? Send a description.]**

**Texture / image-based effects:**

- **`Overlay`** — Flat decal stretched over the user's camera view (HUD-
  style overlay). Requires `TEXTURE` (Roblox decal ID). Used for cutscene
  flashes, manga-panel inserts, screen-wide impact decals (e.g. Switcher's
  "Todo cuts Choso's hair" overlay). Sized to the screen; `SIZE` is a
  multiplier.
- **`Billboard`** — A small 2D decal that floats in 3D space always
  facing the camera. Requires `TEXTURE`. Use for kanji impact, floating
  damage counters, screen-facing logos.
- **`Mesh`** — A custom 3D mesh. `AMOUNT` = mesh asset ID; `TEXTURE` =
  wrap texture (optional). Use for swords, scythes, halos, sigils,
  coffins, planets — anything that needs a real 3D body. The most
  open-ended effect in the system but also the most fragile (wrong ID =
  invisible). See §7.5 for the "ask the user for IDs" workflow.

**Cursed-energy / aura effects:**

- **`Cursed Energy`** — Wispy curling energy particles. Canon
  cursed-energy aura. Color it via the palette in the next subsection
  (Blue/Purple/Pink/etc.). High `AMOUNT` (15+) → roaring aura;
  low `AMOUNT` (1–3) → faint wisp. Use for character idle auras,
  charge-up visuals, awakening aura.
- **`Rough Energy`** — Choso/Black-Death's signature jagged flipbook
  texture animation — rougher and more chaotic than Cursed Energy.
  **[UNVERIFIED: confirm flipbook is the right description; the wiki
  flags it has a known load bug where the flipbook frames render
  improperly.]**
- **`Energy Sparks`** — Bright energy sparks shooting outward, denser
  and more crackly than plain `Sparks`. Used during cursed-energy
  build-ups (Lapse Blue MAX windup, Reversal Red windup). **[UNVERIFIED:
  confirm relative density vs Sparks.]**
- **`Sparks`** — Small bright sparks scattering in random directions.
  Lighter / more diffuse than Energy Sparks. **[UNVERIFIED.]**
- **`Black Flash`** — JJK trademark black-flash impact — the dark-purple
  electric ring that briefly distorts. Color usually defaults to
  purple-black; can be re-tinted via `COLOR`. Burst-style — short
  `TIME` (0.1–0.2s). Pair with `Shake Heavy` and a `Screen Color`
  negative-amount frame for the full canon look.
- **`Weak Lightning`** — Small lightning bolts crackling between points.
  **[UNVERIFIED: confirm whether it draws between POSITION and ALT
  POSITION, or random crackles around the limb.]**

**Wind / motion effects:**

- **`Wind Streak`** — A directional wind line/streak emitting forward
  from the user. Use for fast moves, speedblitzes, dash trails.
- **`Wind Expand`** — Wind effect expanding outward from the user (a
  shockwave-style burst, but wind-textured). Use for impacts, ground
  slams, "powering up" frames.
- **`360 Wind`** — Wind effect rotating around the user 360°. **[UNVERIFIED:
  confirm whether this is a swirling ring around the user vs ambient
  rotating wind.]** Use for "powering up" stances, awakening starts,
  charged-attack windups.
- **`Wind Ring`** — A flat wind ring radiating outward, usually on the
  ground plane. Great for impact-from-landing telegraphs, stomp moves.
- **`Distortion`** — Heat-shimmer / refraction warp distortion. Use for
  heat, sun-breathing-style effects, energy density "you can feel it"
  cues. Subtle by default; bump `AMOUNT` for thickness.

**Slash / weapon effects:**

- **`Melee Trail`** — An afterimage trail following a moving limb. Best
  on `BODY PART: "Right Arm"` or `"Left Arm"` so it tracks the swinging
  hand. Use for sword/fist trails on M1s and skill swings.
- **`Slash`** — A single curved slash arc visual. **[UNVERIFIED: confirm
  shape — single curved blade arc, or straight blade flash?]**
- **`Cleave`** — A heavier/wider slash arc. **[UNVERIFIED: confirm
  difference vs Slash — is it bigger, two-handed-style, or radially
  oriented?]**
- **`Whirl Slash`** — A spinning slash visual, the rotating-blade
  effect. **[UNVERIFIED.]**
- **`Dismantle`** — The signature "two-line cross-slash" Dismantle
  effect from Sukuna's kit. **Renders a path between `POSITION` and
  `ALT POSITION` — both must be set to non-zero, even
  `ALT POSITION: "0, 0, 0.01"` works to force visibility.**
- **`Mass Hit`** — Multi-hit "X"-style visual lines. Like Dismantle,
  **requires `POSITION` and `ALT POSITION` to be set** to render. Quirk:
  Mass Hit's rotation is altered by `ALT POSITION` rather than `ROTATION`.

**Geometric / shape effects:**

- **`Star`** — A filled star particle/burst. **[UNVERIFIED: confirm
  whether this is a star-shaped particle, a flat 2D star icon, or
  burst-of-stars.]**
- **`Star Outline`** — Outline version of `Star`. **[UNVERIFIED.]**
- **`Cylinder`** — A cylindrical primitive. **[UNVERIFIED: confirm
  whether this is a solid cylinder mesh (for pillars/beams) or a flat
  ring projected as a cylinder.]** Likely useful for pillar-of-light
  type effects.
- **`Sphere`** — A spherical primitive. **[UNVERIFIED: solid or
  wireframe? With OPACITY 0 + bright color, plausibly an energy
  orb / domain core.]**
- **`Wedge`** — A wedge / triangular shape. **[UNVERIFIED: pizza-slice?
  Door wedge? Triangular blade?]**
- **`Ring`** — A flat ring shockwave (usually on the ground plane).
  Standard "ground-impact ring" you see under finishers and ground
  slams.
- **`Burst`** — A radial particle burst from the limb. Generic explosion-
  -style burst when you want energy radiating out. Tune density with
  `AMOUNT`.

**Light / glow effects:**

- **`Light`** — A localized light source. Brightens the surrounding area
  without a strong visible body. **[UNVERIFIED: confirm whether this is
  a real Roblox PointLight (lighting the scene) or just a bright sprite.]**
- **`Glow`** — A soft glow halo around the limb. The wiki's known trick
  uses `EFFECT: "Glow"`, `OPACITY: 0.75`, `ALT OPACITY: 1`, `TIME: 0.25`
  to replicate the on-hit white flash that most attacks apply.
- **`Circle Glow`** — A flat disc-shaped glow on the ground or in a
  plane. **[UNVERIFIED.]**
- **`Shine`** — A bright glint/shine highlight. **[UNVERIFIED — guess:
  the brief reflective sparkle on a polished surface, like the gleam on
  a sword pommel or a JoJo-style flex.]**
- **`Clash`** — A bright burst-with-radiating-lines visual — the classic
  anime "BOOM" impact star. The wiki's cartoony-explosion trick uses
  `Clash` with `COLOR: "255, 128, 64"` and `ALT OPACITY: 10`, `ALT SIZE: 10`
  for an oversized orange blast.

**Combat utility effects:**

- **`Beam`** — A single beam/laser line. Direction follows the user's
  facing by default. **[UNVERIFIED: confirm whether `Beam` renders a
  solid laser line from limb forward, or a draws-between-points line
  like Dismantle.]**
- **`Beams`** — Multiple beams (plural). **[UNVERIFIED: confirm how it
  differs from `Beam` — a fan of beams? Multiple parallel beams?
  `AMOUNT` controlling count?]**
- **`Flames`** — A fire / flame particle effect. Burning columns or
  flame jets. Use for fire-themed moves (Soul Fire, Sukuna's open-flame
  moves, fire breathing).
- **`Blood`** — Blood-drop particles scattered around the user. `AMOUNT`
  is the **number of drops**; `SIZE` is the **random radius** (in studs)
  the drops fly around the user. Color it via `COLOR` (canon red, dark
  red, etc.).
- **`Block`** — The block-stance visual. **[UNVERIFIED: confirm whether
  this triggers the blocking pose-and-shield visual, or just the
  shield-impact spark when a block lands.]**

### EFFECT-by-EFFECT compatibility table

| Effect | TEXTURE | AMOUNT meaning | POSITION required? | Notes |
|---|---|---|---|---|
| Mesh | ✅ | mesh ID | n/a | Use `SIZE: 0.1` for normal meshes |
| Overlay | ✅ | count | n/a | Screen-locked |
| Billboard | ✅ | count | n/a | 3D billboard |
| Field Of View | ❌ | FOV distance | n/a | Negative = zoom IN |
| Screen Color | ❌ | opacity (0.01–1) or negative=invert | n/a | Full-screen tint |
| Blood | ❌ | drop count | n/a | SIZE = scatter radius |
| Dismantle | ❌ | count | **YES + ALT POSITION** | Renders a line |
| Mass Hit | ❌ | count | **YES + ALT POSITION** | ALT POSITION controls rotation |
| Cancel | ❌ | n/a | n/a | Uses VISUAL TAG to remove other effects |
| Camera | ❌ | n/a | yes | Only POS/ROT/TIME matter |
| All others | ❌ | instance count | optional | High AMOUNT = concentrated/dense |

**Effect-specific quirks (full list, repeated for emphasis):**

- `Mesh` — `AMOUNT` is a Mesh asset ID, not a count. `TEXTURE` wraps.
- `Field Of View` — `AMOUNT` is view distance. Negative zooms in.
- `Dismantle`, `Mass Hit` — *require* both `POSITION` and `ALT POSITION`
  to render at all.
- `Overlay`, `Billboard`, `Mesh` — only effects that accept `TEXTURE`.
- `Cancel` — pair with `VISUAL TAG` to remove prior tagged effects on
  the same `BODY PART`.
- `Camera` — only `POSITION`/`ALT POSITION`, `ROTATION`/`ALT ROTATION`,
  `TIME`, `LAST HIT` work.
- `Blood` — `AMOUNT` = drop count, `SIZE` = random scatter radius.
- `Screen Color` — negative `AMOUNT` inverts colors.

**Easing styles** (from the wiki — graphs all shown in the "In"
direction; the curve flips for Out / InOut):

| Style | Behavior |
|---|---|
| `Linear` | Constant speed throughout. |
| `Sine` | Gentle ease, soft start and end. |
| `Quad` | Mild acceleration / deceleration. |
| `Cubic` | Stronger curve than Quad. |
| `Exponential` | Slow start, sudden sky-rocket. Read as "explosive". |
| `Back` | Slight overshoot at the end (or start). Good for "snap" feel. |

`EASING DIRECTION` flips the curve:
- `In` — starts slow, speeds up.
- `Out` — starts fast, slows down at the end.
- `InOut` — eases both ends, faster middle.

**Allowed `BODY PART` values** (the wiki UI labels this field `Limb`,
but real exports key it `BODY PART`): `HumanoidRootPart` (default —
invisible anchor unaffected by animation; effectively torso-without-anim-offset),
`Head`, `Torso`, `Right Arm`, `Left Arm`, `Right Leg`, `Left Leg`.

**White-flash hit-react trick** (from the wiki's Known Tricks): the
white flash most attacks apply on hit can be replicated by adding a
`Glow` VISUAL with `OPACITY: 0.75`, `ALT OPACITY: 1`, `TIME: 0.25`.

**Cursed-energy palette** (from the wiki — pick from this list for
canon-feel auras):

| Color | RGB | Used by |
|---|---|---|
| Blue | `85, 170, 255` | Vessel, Switcher, Salaryman, some Defense Attorney moves |
| Pink | `245, 105, 255` | Cursed Partners |
| Purple | `170, 85, 255` | Perfection |
| Darker purple | `111, 67, 255` | Star Rage |
| Green | `85, 255, 127` | Restless Gambler |
| Red | `255, 110, 110` | "Strongest Of History" (wiki label; this is **not** one of the 16 canonical characters in §6 — likely refers to an awakening / variant skin / NPC. Use the red anyway when a move calls for blood-red cursed energy.) |

**Cartoony explosion preset** (Known Tricks): `EFFECT: "Clash"`,
`COLOR: "255, 128, 64"`, `ALT COLOR: "255, 128, 64"`, `ALT OPACITY: 10`,
`ALT SIZE: 10`, extend `TIME` for a longer puff.

### 5.5 `SFX` — sound effect

```jsonc
{
  "K_NAME": "SFX",
  "ID": "<roblox asset id>",    // e.g. "128213617026859" — string of digits
  "VOLUME": 0.5,
  "SPEED": 1.0,
  "START": 0,
  "END": 500,                   // playhead end in seconds
  "CANCEL": false,              // when true: stop any playing sound with the same ID instead
  "GLOBAL": false,
  "PROJECTILE TAG": "nil"
}
```

Do **not** invent sound asset IDs. Either copy IDs from a decoded
sample (e.g. `128213617026859`, `129465573909487`, `137243377137274`,
`120714138513879`) or omit `SFX` blocks entirely.

**Volume note:** **most sounds need `VOLUME` of 2 or 3 to be audible
in-game** — the default `0.5` is usually too quiet to hear over
combat. Crank it up. `GLOBAL: true`
(wiki) plays the sound *for all players anywhere on the map* —
ignores distance falloff entirely. Use for announcer-style audio,
domain narration, or anything you want every player to hear; leave
`false` for local positional SFX.

**`CANCEL: true` is the SFX "stop" switch.** When the block fires with
`CANCEL: true`, it doesn't play the audio — it **stops any currently
playing audio with the matching `ID`**. Useful for cutting off a long
chant sound mid-move.

**`PROJECTILE TAG` makes a sound ride a projectile** — the sound emits
from the projectile's current position, useful for whooshing fireballs
or beam-trailing audio.

### 5.6 `WAIT` — delay the timeline

```jsonc
{ "K_NAME": "WAIT", "TIME": 0.3 }
```

`WAIT` is the only block whose sole purpose is to advance time. It is
required to gap an `ANIM`/`VISUAL` from a follow-up `VELO`/`HITBOX`.

### 5.7 `STATE` — apply a state to the user

```jsonc
{
  "K_NAME": "STATE",
  "STATE": "Stun" | "IFrame" | "NoM1" | "NoSprint" | "NoJump" | "NoDash" |
           "DisableChase" | "Block" | "DirectionLock" | "SpeedMultiplier" |
           "JumpMultiplier" | "Scale",
  "VALUE": 1,                   // multiplier for SpeedMultiplier/JumpMultiplier/Scale
  "TIME":  1,                   // exception: Scale ignores TIME and persists until
                                // the NEXT State node fires (wiki)
  "CANCEL ON END": true,        // drop the state if the move ends early
  "DISABLE BURST": false,       // disables the user's anti-team evasive dash
  "LAST HIT": -1                // -1 = state applied to user; 1 = applied to target
                                //   (within 1-second hit window)
}
```

**State semantics (what each one actually does):**

| STATE | Effect on the affected player |
|---|---|
| `Stun` | Prevents all actions for `TIME`. **Does not stop currently-active moves** — only blocks new actions. |
| `IFrame` | Total invincibility for `TIME` (no damage taken). |
| `NoM1` | Blocks M1 attacks; everything else allowed. |
| `NoSprint` | Blocks sprinting; walking still works. |
| `NoJump` | Blocks jump. Some jump-emotes (e.g. Sword Trick) bypass this. |
| `NoDash` | Blocks **all** dashes (forward, side, back) — the practical empirical behavior; some older sources annotate this as "(side dash)" only. Test in-game if a move design depends on this. |
| `DisableChase` | Blocks ONLY the forward dash (chase). Side/back dashes still work. |
| `Block` | Forces blocking without the block animation. Niche but useful. |
| `DirectionLock` | Locks the user's facing — they cannot turn for `TIME` seconds. Use during cinematic windups so the user can't aim-swap mid-cast. |
| `SpeedMultiplier` | Multiplies the user's walk speed by `VALUE`. Base = 16, so `VALUE: 2` → 32 speed. |
| `JumpMultiplier` | Multiplies the user's jump height by `VALUE`. Base = 40. |
| `Scale` | Resizes the user (giant mode / shrink). **Persists until the next State node fires** — `TIME` is ignored. |

**Targeting via `LAST HIT`:** if `LAST HIT ≥ 0` and the user has hit a
target within 1 second, the state is applied to that **target** instead
of the user. This is how you inflict `Stun`, `NoM1`, etc. on the enemy
post-hit — set `LAST HIT: 1` (or higher, up to the hit-window length).

**`DISABLE BURST: true`** prevents the affected player from using their
burst-cancel (the anti-team evasive dash) during the state. Use during
combo-locking sequences when you don't want the target burst-cancelling out.

### 5.8 `BRANCH` — jump to a branch

```jsonc
{ "K_NAME": "BRANCH", "BRANCH": "branchName", "LAST HIT": -1 }
// or, randomized:
{ "K_NAME": "BRANCH", "RANDOM": "A,B,C" }    // no spaces between names
```

**Behavior:** when a `BRANCH` block fires, the **current `Line` halts**
and the named branch's `Line` begins running. Branches are "groups of
nodes" — labeled sub-timelines that act as alternate execution paths.

**`LAST HIT` semantics:**
- `-1` (default): the **user** runs the branch. Current timeline ends,
  branch's timeline takes over on the user.
- `1` (or any positive seconds): the **last-hit target** runs the branch
  on themselves. The user's original timeline continues uninterrupted.
  Use this for inflicting target-side effects (bleed, ragdoll,
  knockback) without halting the user.

**Branches in `DATA.Branch`** are dictionaries: `{ "name": { "Req": [...],
"Line": [...] } }`. Each branch can carry its own `Req` (conditions)
that gate whether the branch fires — useful for "if airborne, run
`air`, else run `ground`" routing patterns. Properties (`Prop`) are
**global** to the whole slot — they cannot be changed per branch.

**Branch hub pattern (recommended for complex moves):** use the default
`Line` as a thin router that fires `BRANCH` blocks pointing at named
branches; put the actual content in those branches. This avoids
"branchlag" (delays from deeply-nested branches) and is the canonical
way to handle aerial/grounded splits, hit/no-hit splits, etc.

> **⚠ UNVERIFIED FALLTHROUGH SEMANTICS ⚠**
>
> The hub pattern depends on this behavior: when a `BRANCH` block in
> the parent `Line` points at a branch whose `Req` *fails*, the
> `BRANCH` block silently no-ops and execution **continues** to the
> next block in the parent `Line` (typically another `BRANCH` for the
> alternate condition). This matches the wiki's "branchlag hub" trick
> from §5.23, and is the only natural way to write
> "if airborne → uppercut, else → downslam" inside a single slot.
>
> **But:** failed-Req BRANCH fall-through is not *explicitly*
> documented anywhere — only implied by the hub example, never directly
> stated. Multiple test movesets have flagged this as a risk.
>
> **Workaround if it doesn't fall through:** instead of branch-Req
> gating, encode the state in a `TAG` (e.g. set `Airborne` tag when
> the user takes off via the move's own logic, or use a `Req.AIR` at
> the **slot** level which is reliable). Then use `TAG CHECK` blocks
> in the parent `Line` to route, since TAG-CHECK fall-through **is**
> confirmed — a CHECK that doesn't match "will decline and continue the
> timeline." The trade-off: TAG-CHECK can't observe intrinsic state
> like airborne — only what you've explicitly set.
>
> **When you ship a hub-based move, ask the user to verify it in-game
> as part of their playtest report.** If the wrong branch fires, this
> is the first thing to investigate.

**Branch names are case-sensitive.** Spell the name exactly how it
appears in `DATA.Branch`; otherwise the branch silently does nothing.

### 5.9 `LOOP` — repeat the previous N nodes

```jsonc
{
  "K_NAME": "LOOP",
  "LOOP BACK":   1,             // how many nodes above to include in the loop body
  "LOOP AMOUNT": 3,             // how many times to repeat (3 = the body runs a total of 3 times,
                                //   not 4 — original + 3 repetitions is a common misread)
  "HOLD": false                 // when true: loop while key is held (must have an exit elsewhere)
}
```

**Loop body design:** put a small `WAIT` (e.g. `WAIT 0.05`) as the first
node in the loop body to give frame separation between iterations. Without
it, all iterations can fire on the same frame and look like one effect.

**Use cases:**
- **Afterimage trails** — `WAIT 0.05` + `VISUAL Wind Streak` + `LOOP
  BACK 2, AMOUNT 10` produces 10 wind streaks fading in over 0.5s.
- **Repeated checks** — a `TAG CHECK` + `BRANCH` + small `WAIT`, looped
  many times, polls for a tag value. Set `LOOP AMOUNT` very high (50+)
  for long-running checks.
- **Multi-hit punches** — `VISUAL Clash` + `HITBOX (tiny size, low
  damage)` + `SFX punch` + `WAIT 0.1` + `LOOP BACK 4, AMOUNT 8` for a
  classic Rapid Punches-style burst.

**Caveats:**
- `LOOP AMOUNT: 0` is invalid — nothing loops.
- `HOLD: true` is intended for built-in hold-skills inside the loop
  body (e.g. Star Rage's Mass Build-Up). It does **not** work for most
  custom hitboxes or arbitrary moves. Don't rely on `HOLD` to make a
  custom move's loop hold-controlled — gate it via a `BRANCH` + `STATE`
  pattern instead.

### 5.10 `COUNTER` — parry window

```jsonc
{
  "K_NAME": "COUNTER",
  "ATTACK TYPE": "Melee",       // comma-separated list allowed: "Melee,Bullet,Explosion"
                                //   wiki says multiple types can be selected; none of the
                                //   built-in characters counter Explosion or Domain
  "TIME": 1,                    // duration of the parry window in seconds
  "STUN": 1,                    // how long the countered attacker is stunned
  "CANCEL ENEMY": true,         // also cancels the attacker's move/m1 on parry
  "REMOVE ON HIT": true,        // true: window ends as soon as a counter lands
                                //   false: window stays open for full TIME even after first parry
  "BRANCH": "...",              // user-side branch (fires on user when counter activates)
  "BRANCH TARGET": "..."        // attacker-side branch (fires on the parried attacker)
}
```

**No animation built-in.** The COUNTER block performs the *parry logic*
(damage nullification + branch trigger) without any animation or visual.
You must pair it with an `ANIM` (a defensive stance) and `VISUAL`
(typically `Block` or a custom shimmer) blocks to make the parry
read on screen.

**Counter design pattern:** the typical counter move structure is:

```jsonc
"Line": [
  { "K_NAME": "ANIM", "ANIM_USE": [g, i], "PREVIEW": [0, 1] },   // parry stance
  { "K_NAME": "VISUAL", "EFFECT": "Block", "TIME": 1 },          // shimmer telegraph
  { "K_NAME": "COUNTER", "ATTACK TYPE": "Melee,Bullet",
    "TIME": 0.6, "STUN": 1.2,
    "BRANCH TARGET": "punished" },                                // <- on parry, target runs this
  { "K_NAME": "WAIT", "TIME": 0.6 }                              // end of window
]
```

Then `Branch.punished` does the punish — staggers, ragdolls, runs a
follow-up attack, etc.

### 5.11 `PROJECTILE` — moving hitbox

```jsonc
{
  "K_NAME": "PROJECTILE",
  "PROJECTILE TAG": "MYTAG",    // join SFX/VISUAL/HITBOX nodes via the same tag
  "POSITION":  "0, 0, 4",
  "ROTATION":  "0, 0, 0",
  "SIZE":      "6, 6, 6",
  "SPEED": 60,
  "TIME":  1.5,
  "DAMAGE": 10,
  "STUN": 0.5,
  "STUN ANIM": true,
  "BLOCKABLE": true,
  "ATTACK TYPE": "Bullet",
  "CAN KILL": true,
  "CONTINUE": false,            // false = projectile ends on first collision (single-hit, "dissipate");
                                // true  = projectile pierces walls/targets, ending only when TIME runs out.
                                // (Source conflict: wiki's prose phrases this inverted, but the community
                                //  guide — written by an active builder testing in-game — is unambiguous
                                //  that GREEN/TRUE = pierce. Use community convention.)
  "AIM LAST HIT": 0,
  "FILTER INTERVAL": 1,
  "BRANCH TARGET": "...",
  "BRANCH COLLIDED": "..."
}
```

**Wiki bug notes on `PROJECTILE`:**
- The plain `BRANCH` field (user-side branch on hit) is **currently broken**
  — use `BRANCH TARGET` instead when you need a branch on hit.
- `BRANCH COLLIDED` (wall-collision branch) is also reported as
  non-functional by experienced players. Don't rely on it.
- The `Single Target` wording on the wiki contradicts the Hitbox version
  (says "on = multi-target" while Hitbox says "off = multi-target") —
  most likely a wiki typo; in real exports it behaves like Hitbox
  (`true` = single target).

**SPEED × TIME math (CRITICAL):** projectiles travel `SPEED × TIME`
studs total. To make a projectile cross 25 studs in 0.1 seconds, you
**cannot** write `SPEED: 25, TIME: 0.1` (that gives 2.5 studs). The
community trick: pick the target distance, divide by `TIME` to get the
required `SPEED`. So 25 studs / 0.1 s = `SPEED: 250`. Use generous
`TIME` and modest `SPEED` for slow-moving fireball-style projectiles;
use short `TIME` and high `SPEED` for instant lasers/bullets.

**Projectiles are not just for ranged attacks.** A projectile is the
canonical way to make a **moving hitbox or moving visual**. Common
patterns:
- A wave/shockwave traveling along the ground (Z-flat projectile,
  `Continue: false`).
- A meteor / boulder dropping from above (projectile with `Y < 0`
  rotation, big SIZE, `Continue: true` so it pierces).
- A spirit/clone chasing the target (projectile with `AIM LAST HIT: 1`
  for homing).
- A sword/weapon mesh flying forward (PROJECTILE TAG matched by a
  Mesh VISUAL block that rides it).

**`FILTER INTERVAL` controls re-hit timing.** Default 1 second. Set to
`0` if you want a projectile to deal damage continuously while inside
a target (e.g. a "burning beam" that ticks every frame). Lower this
when you want piercing projectiles to multi-hit.

**`Continue` semantics:** wiki/community phrasing differs. In real
exports: `Continue: false` = projectile **ends on first collision**
(dissipates on impact, single-hit feel). `Continue: true` = projectile
**pierces** all targets and walls, ending only when `TIME` runs out
(beam-style multi-hit feel).

### 5.12 `GRAB` — stick limbs together

```jsonc
{
  "K_NAME": "GRAB",
  "BODY PART":  "Right Arm",    // user's limb (wiki UI: "Limb")
  "BODY PART2": "Torso",        // target's limb (wiki UI: "Target Limb")
                                //   **UNVERIFIED encoded key.** Real exports may use a
                                //   different field name (e.g. "TARGET LIMB" / "PART2").
                                //   Confirm against a decoded sample that uses GRAB.
  "POSITION": "0, 0, 1",
  "ROTATION": "0, 0, 0",
  "TIME": 0.5,                  // grab duration
  "LAST HIT": 1                 // wiki default 1 second: how long the user has to
                                // damage/stun a target in order to apply the grab
}
```

Wiki bug: if the ability ends before the GRAB registers stun, the user
gets **teleported to the target** instead. Always ensure a HITBOX with
stun fires before the GRAB, or that the move has time to complete.

**Useful rotations:** the user's limb and target's limb are oriented in
the local axes of their respective limbs (see §2.5). For a "make the
target face the user" grab (i.e. spin them around to face you), use
`ROTATION: "0, 180, 0"`. For pure repositioning with no rotation, leave
at `"0, 0, 0"`. Spacing of `"0, 0, 4"` (4 studs in front of user) is
the conventional "grappling distance" — large enough that limbs don't
clip, small enough to read as physical contact.

**Grab as repositioning trick:** set `TIME: 0.01` and the grab acts as
a near-instant teleport that snaps the target into the desired POSITION
and ROTATION, then immediately releases — useful for "reposition the
target before a follow-up", e.g. spinning them to face away before a
backstab finisher.

### 5.13 `SKILL` — invoke a built-in skill or `Cancel`

```jsonc
{
  "K_NAME": "SKILL",
  "MOVE": "Rapid Punches",      // see §7 catalog of in-game skills, or "Cancel"
  "SPEED": 1,
  "START": 0,
  "CANCEL LAST": false,
  "ENABLE VARIANTS": true,
  "HOLD FOR": 0
}
```

`MOVE: "Cancel"` halts every still-running block above it.

### 5.14 `SPECIAL` — invoke a built-in special

```jsonc
{
  "K_NAME": "SPECIAL",
  "SPEC": "Limitless" | "Combat Instincts" | "Convergence" | ...,
  "SPEED": 1,
  "CANCEL LAST": false,
  "ENABLE VARIANTS": true
}
```

### 5.15 `TAG` — set / check a tag variable

```jsonc
// set the tag "Cooldown" for 5s with value 1
{ "K_NAME": "TAG", "TAG": "Cooldown", "VALUE": 1, "TIME": 5,
  "ADD/REMOVE": "Add", "SET": true, "CHECK": false, "LAST HIT": -1 },

// check tag, branch on match
{ "K_NAME": "TAG", "TAG": "Cooldown", "VALUE": 1, "CHECK": true,
  "BRANCH": "onCooldown" }
```

**Field encoding caveat — UNVERIFIED.** Documentation describes the
`ADD/REMOVE`, `SET`, and `CHECK` toggles as separate UI booleans. The
encoded JSON form may serialize them differently —
e.g. `ADD/REMOVE` could be a string `"Add"`/`"Remove"` (as shown
above, the documented form) or paired booleans `ADD: true`/`REMOVE: true`.
If `TAG` blocks misbehave in-game, decode a moveset that uses a
working TAG and copy the literal shape.

Wiki notes:
- `CHECK` is **prioritised** over `ADD/REMOVE` and `SET` — when true,
  the other toggles do nothing.
- `ADD/REMOVE` "Add" adds `VALUE` to the tag; "Remove" nullifies the
  tag entirely.
- `SET` combined with `Add` on overrides the tag's value to `VALUE`.
- Tags are case-sensitive.

### 5.16 `HITCNCL` — hit cancel

```jsonc
{
  "K_NAME": "HITCNCL",
  "TIME": 1,                    // lookback window (seconds) for the stun/damage check.
                                //   HARD CAP: 1 second. Cannot check further back.
  "FLIP": false,                // false (default — red toggle in-game): cancel the rest
                                //   if stun/damage WAS dealt within the lookback. ("cancel-if-hit")
                                // true  (green toggle in-game): cancel the rest if NO
                                //   stun/damage was dealt. ("cancel-if-miss" — the COMMON pattern)
  "ENDLAG": 1,                  // forced freeze after the cancel fires
  "BRANCH": "fallback"          // optional: run this branch instead of just ending
}
```

**Standard usage** (gate the rest of a move on a hit landing — the
common pattern needs `FLIP: true`):

```jsonc
{ "K_NAME": "WAIT",    "TIME": 0.3 },             // wait long enough for the hit to land
{ "K_NAME": "HITCNCL", "TIME": 0.3, "FLIP": true }, // if no hit dealt, cancel and endlag
{ "K_NAME": "WAIT",    "TIME": 0.1 },             // recommended trailing wait (wiki note)
// ...rest of move follows only if a hit was dealt
```

**Wiki-known bug:** `HITCNCL` can incorrectly register stun *from
outside this custom move* (an in-progress M1, a different ability)
if its `TIME` exceeds the move's startup. Workaround:
1. `WAIT` for the move's startup duration first.
2. Set `HITCNCL.TIME` equal to that same startup duration.
3. Add a second `WAIT` after the `HITCNCL`.

Without this fix, your "if you missed, cancel" logic can be fooled by
hits the user landed *before* the move began.

### 5.17 `ULTGIB` — awakening burst (AWAKENING slots only)

```jsonc
{ "K_NAME": "ULTGIB", "AMOUNT": 1 }
```

### 5.17a `CONNECT` — send a signal to Connect Blocks in the map

```jsonc
{
  "K_NAME": "CONNECT",
  "SIGNAL": 1,                  // matches Connect Block's Signal ID
  "TIME":  1,                   // how long the signal stays active
  "RANGE": 50                   // user must be within this distance to fire
                                //   **UNVERIFIED encoded key.** Wiki/community
                                //   alternately call this "Distance"; the actual
                                //   encoded field name may differ. If your CONNECT
                                //   signal has no range gating in-game, try renaming
                                //   to "DISTANCE".
}
```

Connect is the bridge from Skill Builder out to map-level Logic Blocks
(door triggers, hidden hitboxes, environmental hazards, etc.). Both the
`K_NAME` and the `RANGE` field name are unverified across exports — if a
decoded moveset uses different keys, match it.

### 5.17b `ADDAWK` — modify the user's awakening percentage

```jsonc
{ "K_NAME": "ADDAWK", "AMOUNT": 5 }   // +5% awakening; negative = drain
```

### 5.17c `ADDHP` — modify the user's HP

```jsonc
{ "K_NAME": "ADDHP", "AMOUNT": 5 }    // +5 HP; negative drains but can't kill
```

### 5.17d `ADDEV` — modify the user's evasive/ragdoll-cancel meter

```jsonc
{ "K_NAME": "ADDEV", "AMOUNT": 5 }    // +5% evasive bar (caps at 50%)
```

> **⚠ DO NOT SHIP `ADDAWK` / `ADDHP` / `ADDEV` BLINDLY — CONFIRMED
> EMPTY-TIMELINE BUG ⚠**
>
> The wiki UI labels these blocks *Add Awakening*, *Add Health*,
> *Add Evasive*, but their **encoded `K_NAME` strings are not
> documented anywhere.** The names `ADDAWK` / `ADDHP` / `ADDEV`
> above were an educated guess from earlier iterations of this
> skill. **They are wrong (or at least one of them is).**
>
> Playtest (Goku Super Saiyan, May 2026): an AWAKENING Line that
> included `{"K_NAME":"ADDHP", "AMOUNT":40}` and
> `{"K_NAME":"ADDEV", "AMOUNT":25}` caused the **entire** AWAKENING
> slot's Skill Builder timeline to render as completely empty in
> JJS — even though the JSON contained 25 valid blocks and the codec
> round-trip verified losslessly. Removing those two blocks
> restored the Line.
>
> Generalized rule: **any unknown `K_NAME` in a Line silently voids
> the parent slot's timeline.** The parser doesn't skip the bad
> block — it drops the whole Line. This is the "empty timeline"
> failure mode previously misattributed in §8.1 to a slot-shape
> issue; it is in fact an unknown-K_NAME issue.
>
> **Rule:** do not use `ADDAWK` / `ADDHP` / `ADDEV` (or any other
> unverified K_NAME from §5) **until you have decoded a moveset
> that uses the corresponding in-game block and copied the literal
> encoded K_NAME out.** Until then, work around their absence:
> - For healing, use ADDHP-like mechanics via cost-free means (a
>   `HITBOX` with `HIT USER: true` and... actually no, that *damages*
>   the user. There is no clean substitute; just live without
>   healing until the encoded key is known).
> - For awakening-percentage adjustments, leave them off.
> - For evasive-meter top-up, leave it off.
>
> If you really must include a "feels like healing" beat, do it
> with a `VISUAL Glow` flash + a `STATE` buff that increases
> effective survivability (speed multiplier dodges hits, scale
> change makes hitboxes miss). Cosmetic > broken.

### 5.18 `Req` entries (gates for whole slot)

```jsonc
{ "K_NAME": "AIR",  "FLIP": true  }   // FLIP true = must be airborne (FLIP false = grounded)
{ "K_NAME": "JUMP", "FLIP": false }   // must not be holding jump
{ "K_NAME": "HP",   "FLIP": false, "AMOUNT": 30 }   // HP <= 30 (FLIP true = HP >= 30)
{ "K_NAME": "BAR",  "AMOUNT": 5 }     // awakening bar <= AMOUNT (FLIP true = bar >= AMOUNT); default AMOUNT 5
{ "K_NAME": "ULT",  "FLIP": false }   // FLIP false = must be in BASE state (FLIP true = awakened)
```

**Important:** SKILL.md previously described `ULT` as "must be in
awakening" by default — that's backwards. Wiki: with `FLIP` off (the
default), Is Awakened gates to the **base** state. To require
awakening, use `"FLIP": true`.

### 5.19 `Prop` entries (slot-wide flags)

The full Properties list (from the wiki's Properties tab — these are
the toggles in the in-game Properties tab, encoded as `Prop` entries on
the slot):

| Property | Behavior |
|---|---|
| `Damage Multiplier` | Multiplies the whole ability's damage. |
| `Knockback Multiplier` | Multiplies the whole ability's knockback. |
| `Invincible` | Total i-frames for the ability's duration. |
| `Replace Skill if Occupied` | If the chosen Moveset slot already has a move, replace it. |
| `Prevent Override` | Stops this ability from being replaced by another with the above property. |
| `Hide in Awakening` (`AWK`) | Don't show this ability while awakened. |
| `Keep When Moveset Switches` | Stays bound when base ↔ awakening swap happens. |
| `Hide in Base` (`AWK2`) | Don't show this ability outside awakening. |
| `Use When Obtained` | Auto-fires the moment the player gets the ability. |
| `Use On Death` | Auto-fires when the player dies. |
| `No Stun` | Disables skill-check stuns (the move can fire even mid-stun). |
| `No Cancel` | The ability is uninterruptible. |
| `Variant Tag` | Lets the ability bypass cooldown/skill checks while a tag is active on the user. |

Encoded shorthand seen in real exports (mapping **user-corrected
May 2026** — earlier drafts had these inverted):

```jsonc
{ "K_NAME": "AWK"  }   // Hide in Awakening (base-only ability — goes on base SKILL slots)
{ "K_NAME": "AWK2" }   // Hide in Base      (awakened-only ability — goes on awakened SKILL slots)
```

**Mnemonic to keep them straight:** `AWK` stays on base moves
("Awakening hides this — it's a base move"). `AWK2` goes on
awakened moves ("the *second* form's ability — only shows in
awakening"). The naming is genuinely confusing; double-check the
Prop assignment before encoding any Pattern C / Shape-2 moveset.

> **⚠ STILL UNVERIFIED BEYOND THE INVERSION — DO NOT ASSUME IT
> WORKS PERFECTLY ⚠**
>
> Earlier drafts of this skill had the AWK ↔ AWK2 mapping backwards
> (AWK was documented as "Hide in Base"). The May 2026 Goku
> playtest caught the inversion — user feedback: *"awakening moves
> must have hide in base property... whatever is the opposite of
> base moves."* The mapping shown above is the user-corrected
> version.
>
> But: it is still possible that
> - The K_NAMEs themselves are wrong (maybe the real encoded keys
>   are `HIDEBASE` / `HIDEAWK` or similar, and what we see as
>   `AWK`/`AWK2` in decoded movesets is something else entirely),
> - One of the two Props does nothing,
> - Or both Props do nothing and Pattern C is unbuildable from a
>   single import code.
>
> Until a decoded known-working Pattern C moveset is on hand,
> **always tell the user that the in-game Skill Builder Properties
> tab is the source of truth**: if the wrong moves appear in the
> wrong state after import, they should open each affected SKILL
> slot's Properties tab and manually toggle "Hide in Base" /
> "Hide in Awakening." That manual toggle is guaranteed to work
> regardless of which encoded K_NAME the format uses.

### 5.20 Conditions reference

Conditions (the in-game "Conditions" tab) are the `Req` gates from §5.18.

> **⚠ SOURCE CONFLICT — TEST IN-GAME BEFORE SHIPPING ⚠**
>
> Two interpretations of FLIP semantics exist in the wild. One
> reading is the wiki's: FLIP off = condition's plain meaning, FLIP
> on = its negation. The other reading — reported by hands-on
> builders — is inverted: *"In Air: When toggled red, the move can
> only be used when aerial; when toggled green, the move can only
> be used when grounded."*
>
> Red toggle = off = `FLIP: false`; green toggle = on = `FLIP: true`.
> So under the empirical reading **`FLIP: false` = airborne-only**,
> **`FLIP: true` = grounded-only** — the OPPOSITE of the wiki table
> below.
>
> The table below mirrors the **wiki's** semantics (because they are
> the cleaner-stated source). If a built moveset behaves backwards
> in-game — e.g. an "airborne-only" move fires only when grounded —
> **invert every `FLIP` on Req conditions** and the rest of the move
> should work. Mark this gotcha in any design brief that touches Req
> gates until the user confirms which source is correct.

For each, the wiki's True/False semantics (subject to the warning above):

| Condition | `FLIP: false` (True / off) | `FLIP: true` (True / on) |
|---|---|---|
| `AIR` (In Air) | Must be **on the ground** | Must be **airborne** |
| `JUMP` (Is Jumping) | Must **not** be holding jump | Must **be** holding jump |
| `TARGET` (Has Target) | Must have **no** valid target in sight | Must have a valid target |
| `ULT` (Is Awakened) | Must be in **base** state | Must be **awakened** |
| `BAR` (Has Awk Bar) | Awakening % `≤ AMOUNT` | Awakening % `≥ AMOUNT` |
| `HP` (Has Health) | HP `≤ AMOUNT` | HP `≥ AMOUNT` |
| `DOMAIN` (Is In Domain) | Must be **outside** a domain | Must be **in** a domain |
| `HOLDING` (Is Holding) | Must **not** be holding the keybind | Must **be** holding the keybind |
| `DURABILITY` | Use-limit (default 1); the ability erases after `AMOUNT` uses | not toggleable |

`FLIP` defaults to `false`. The wiki uses "True / False" for each
toggle's column; that maps to `FLIP: false` / `FLIP: true` respectively.

### 5.21 Variant triggering (from the wiki's Variants tab)

`"Enable Variants": true` on a `SKILL` block lets the underlying skill's
variant fire **automatically** when the right preceding-skill + wait
timing is in place on the same timeline. The full mapping (wiki):

**"Use Twice / Again" variants** — chain skill A then `WAIT` then
skill B with the wait time below to trigger B's stronger variant:

| Variant Type | Variant Skill | Triggered After | Wait Time |
|---|---|---|---|
| Black Flash | Focus Strike, Divergent Fist, Cleave Rush | (same skill, repeated) | 0.3 – 0.4 |
| Black Flash | Idle Transfiguration | Focus Strike | 0.75 – 1.0 (depends on distance) |
| Black Flash | Brute Force | Earthquake | 0.6 – 0.7 |
| Other | Reserve Balls | Shutter Doors | 0.01 – 1 |
| Other | Toad | Nue | 0.3 – 0.4 (Nue still summons; add a Cancel 0.2s after to suppress) |
| Other | Body Repel | Focus Strike | ~0.7 (TBA) |
| Other | Revolve | (after itself) | <1s (limit TBA) |
| Other | Veilstep | (after itself) | None — set immediate Cancel if you want the alt |
| Other | Garuda Rebound | (after itself) | 1.27 – 1.53 (2.47 – 2.73 if previously held) |
| Other | Rika Downslam / Slam / Haymaker | True Love Beam | 0.6 |

**"Hold" variants** — set `HOLD FOR` on the `SKILL` block to the listed
duration to trigger the held variant:

| Skill | Hold For |
|---|---|
| Max Elephant | As long as needed |
| Earthquake | 0.75 – 0.9 (ideally 0.85) |
| Piercing Blood | 1.4s+ (requires Convergence Orb) |
| Garuda Rebound | 0.55s+ |
| Ultra Cannon | 1.2+ |

**Special-chained variants** (use a `SPECIAL` block as the trigger):

| Variant | Triggers (in order) | Wait Time |
|---|---|---|
| World Cutting Slash | Dismantle → Rush → Open → Cleave | 1.2 between Open and Cleave |
| Strong Dismantle | Incantation ×3 | 0.6+ (Incantation can stack) |
| Limitless variants | Reversal Red → Limitless | 0.5 – 1.2 |
| Limitless variants | Reversal Red MAX → Limitless | <1.2 |
| Lurking Shadow | Nue → Lurking Shadow | None required |
| Convergence | Slicing Exorcism → Convergence | None required |
| Mass Buildup | Rising Rage / Mass Breaker / Garuda Stab (×2 for Rising Rage charge) | None required |
| Garuda Rebound (chained) | Garuda Rebound → Garuda Rebound | 1 – 1.4 |

**Aerial variants:** trigger by simply being airborne *before* the
custom move starts. All implemented skills with aerial variants will
fire their aerial forms one after another. Severing Path and Rough
Energy's stronger aerial variant need higher ground. Becoming airborne
*during* the custom move (after a Rush, Wing Throw, etc.) does **not**
trigger aerial variants.

### 5.22 Known skill timings (from the wiki's W.I.P. timings tab)

Times are in seconds and vary with ping. Use these to pace your `WAIT`
blocks around built-in skills. Format: `0.X – 0.Y description`.

| Skill | Timing breakdown |
|---|---|
| Lapse Blue | 0–0.55 no pull · 0.8–1 full pull · 1–1.75 suspension · 1.76+ full move |
| Reversal Red | 0–0.2 SFX+red light · 0.2–0.5 VFX · 0.51+ full move + special window |
| Rapid Punches | 0–0.5 windup no grab · 0.5–0.6 grab+1 punch · 3.7+ final punch · 3.8+ full move |
| Twofold Kick | 0.35+ first VFX · 0.41+ first hitbox · 0.94+ second VFX · 1.09+ second hitbox · 1.3+ full |
| Lapse Blue MAX | 0.07+ blue light · 0.53+ hitbox |
| Reversal Red MAX | 0.37+ VFX · 0.5+ SFX · 1.21+ full move + special window |
| Hollow Purple | 0+ SFX · 0.10+ VFX · 1.25+ smash · 2.72+ full |
| Infinite Void | 0+ cutscene · 1.25+ domain |
| Cursed Strikes | 0.3+ SFX · 0.6+ dash · 1+/1.15+/1.47+/1.73+/1.9+ punches 1–5 · 2.2+ full |
| Aerial Cursed Strikes | 0–0.66 ascent · 0.67+ descent · 0.7 hitbox |
| Crushing Blow | 0.02+ SFX · 0.45+ windup · 0.6+ slam 1 · 1+ grab · 1.2+ full |
| Divergent Fist | 0.3+ VFX · 0.36 exact Black Flash window · 0.4+ special window · 0.55+ full |
| Manji Kick | 0.02+ SFX · 0.05+ anim · 0.06–0.4 dodge window · 0.5+ full |
| Dismantle | 0–0.4 windup · 0.41+ hitbox · 1.1+ full |
| Aerial Dismantle | 0–0.61 ascent · 0.62+ hitbox · 1+ full |
| Rush | 0.4666 dash · 1.1–2.1 run · 2.2 kick · 2.3–2.95 jump · 3.5 land · 3.9 end |
| Open | 0.02 particles · 0.89 VFX · 2+ arrow drawn · 2.7+ shot · 3.7 full lifetime |
| Malevolent Shrine | 0+ cutscene · 1.25+ domain |
| Reserve Balls | 0.11+ SFX · 0.32+ hitbox |
| Shutter Doors | 0.15 VFX · 0.435 hitbox |
| Rough Energy | 0.1+ VFX · 0.71 full windup · 0.72+ hitbox |
| Fever Breaker | 0.44 windup · 0.45 kick 1 · 0.5–0.8 KB+VFX · 1–1.25 rush · 1.32 kick 2 · 1.4+ full |
| Lucky Volley | 0–0.4 windup · 0.405 first punch · 0.41+ flurry · 1.37 flurry end · 1.65 end strike |
| Lucky Rushdown | 0–0.55 windup · 0.62 first hitbox · ~2.2 drag · 0.85 throw |
| Overwhelming Luck | 0.05 VFX · 1.15+ dash · punches at 2.1 / 4.17 / 5.7 / 5.9 / 6.27 / 7.17 |
| Energy Surge | 0.05 VFX · 0.57 first hitbox · 1.55 pre-teleport · 1.6 teleport · 1.95 aerial kick |
| Rabbit Escape | 0–0.5 handsign · 0.51+ rabbits · 1.75 stop |
| Nue | 0–0.4 handsign · 0.41+ appears · 0.9 peak · 1.4 reaches target · 2.2 disappears |
| Nue Variant | 0.41+ Nue · 0.75+ user rises · 1.1+ flies with Nue · 1.4+ target · 2.2 disappears |
| Toad | 0–0.4 handsign · 0.41+ attack |
| Well's Unknown Abyss | 0.41+ toads · 1.2+ grab · 1.7+ lift · 2.1+ slam · 3+ disappear |
| Divine Dog: Totality | 0.17+ appears · 1+ full (the second use is currently impossible in Skill Builder) |
| Great Serpent | 0–0.5 jump · 1.2+ summon · 2.8+ full |
| Shadow Swarm | 0–0.55 summon · 0.95+ run · 2.95+ clones gone · 3.65+ full |
| Divine Pummel | 0–0.68 windup · 0.66 grab · 1.27+ slam 1 · 1.82+ slam 2 · 2.42+ full |
| Ground Pitch | 0–0.38 windup · 0.39–1.81 grab · 1.82+ throw |
| Earthquake | 0–0.46 windup · 0.47+ no-hold · 0.47–0.96 hold · 0.97+ full |
| Takedown | 0–0.4 windup · 0.41–0.83 teleport · 0.84+ full |
| World Slash | 0–0.16 pre-VFX · 0.17+ first VFX · 1+ second VFX · 1.37+ full |
| Stockpile | 0–0.48 windup · 0.49–0.74 first hit · 0.75+ full |
| Aerial Stockpile | 0–0.38 ascent · 0.39–0.6 descent · 0.61+ full |
| Soul Fire | 0–0.45 windup · 0.46 bean 1 · 0.59 bean 2 · 0.74+ full |
| Focus Strike | 0–0.4 windup · 0.41+ full |
| Chainwhip | 0–0.45 windup · 0.46+ hit · 1.03+ full |
| Homerun | 0–0.96 windup · 0.97+ hit · 1.88+ full |
| Body Repel | 0–1.04 windup · 1.05+ first VFX · 1.24+ second VFX · 1.4+ full |
| Idle Transfiguration | 0–0.81 windup · 0.82–0.88 dash no-hit · 0.89+ hit · 1.65+ full |

For any skill not listed, decode a moveset that uses it and inspect the
`WAIT`/`ANIM` gaps around it, or check the Build_Mode/Skill_Builder
wiki page for fresher timings.

### 5.23 Known Tricks (from the wiki)

**General**

- **Cartoony explosion preset.** `EFFECT: "Clash"`, `COLOR / ALT COLOR:
  "255, 128, 64"`, `ALT OPACITY: 10`, `ALT SIZE: 10`. Extend `TIME` for
  longer puff.
- **Instant speedblitz.** Set Sacrilege's `START` to 0.45; speed doesn't
  matter.
- **Bluetooth Idol's Debut.** Add a 0.25s `WAIT` then Resolute Slash;
  you can turn and "bluetooth-kick" the enemy down. Works best out of
  awakening (no cutscene gating). Climax Jumping has the same quirk.
- **Blockable / unblockable two-stage.** Start with an unblockable
  skill, add a blockable hitbox, then a `HITCNCL` that ends the move if
  no stun was dealt. If the blockable connected, the move continues.
- **Branch hub.** Use the default branch as a hub linking
  ground/air variants — avoids the "branchlag" of nested branches.
  Add a false-AIR `Req` on the air branch and true-AIR on the ground
  branch, then branch-block the air variant *before* the ground variant
  inside the hub.
- **White-flash hit-react.** Replicate the on-hit white flash with a
  `Glow` VISUAL, `OPACITY: 0.75`, `ALT OPACITY: 1`, `TIME: 0.25`.
- **Unlimited Purple variant.** Use Lapse Blue MAX on Flash Freezing's
  frames — the only reliable way. You can't cancel Flash Freezing
  mid-summon; it always spawns 15 frames.
- **Random branch via Hit Cancel.** Configure a `HITCNCL` normally,
  then inside the branch it points to, add a `BRANCH` node with
  `RANDOM: "A,B,C"` (no spaces).

**Cancel Tricks**

- **Suspended enemy.** Launch upward, then use Fluttering Pounce + Revolve
  simultaneously and Cancel both immediately. Enemy stays floating.
- **Plasma Wave aimable.** Pair Plasma Wave / Ultimate Cannon (no aim
  after windup) with Slicing Exorcism at `SPEED: 1000000` — you can turn
  during the move.
- **Hover via Execution.** Slow Execution's speed and/or Cancel before
  it dashes (at least 0.9s in) to hover briefly. World Cutting Slash's
  first chant (Dismantle + Rush) gives the same effect without needing
  a Cancel.
- **Elbow Rush leftover hits.** Cancel Elbow Rush just as the barrage
  starts — punches continue after Cancel (last hit doesn't land). Same
  trick applies to Drill Splitter.
- **Cursory Impact afterimage.** Cancel Cursory Impact after 0.06s for
  an afterimage trail.

**Speed Tricks**

- **`SPEED: 0` on Rush** — disables movement and locks rotation;
  Cancel to restore both.
- **`SPEED: 0` on Resolute Slash** — makes the character track the enemy
  like lock-on; add a Cancel to disable.
- **`SPEED > 10000` triggers finisher after base** — instantly fires the
  variant/finisher of a skill after the base ends. Bleedout's finisher
  goes blue + afterimages. With Resolute Slash (and Takedown, Limitless)
  forced as the target's branch last-hit, the target **vanishes** and
  respawns instantly. Head Splitter (and most counters) auto-trigger
  the counter. Idol's Debut / Climax Jumping fire the final hit
  directly.
- **High-speed animation residue.** When using a move at extreme
  speeds, the animation can play *after* the custom move ends. Mask it
  by adding the move's animation as an `ANIM` block after the move
  (any part/speed), then optionally adding another quick custom anim
  to overwrite.
- **Awakening outfit without domain.** Use Time Cell Moon Palace at
  `SPEED: 0` and immediately Cancel. Applies the awakening outfit
  without casting the domain or showing the popup.

### 5.24 Known bugs (from the wiki's Current Issues tab)

These affect what you can ship. Design around them:

- **`VISUAL` blocks may cut animations** — fix by adding a `WAIT` block
  matching the cut animation's runtime before the visual.
- **All custom abilities show first-use delay/lag.** Only fix is to
  warm them up by using the move once.
- **Cancelled skills can leak sound** (Body Disfigure, Open FURNACE,
  Plasma Wave, Strong Dismantle variants, etc.).
- **Rika is required for all Rika moves** (Rika Downslam, True Love
  Beam, Rika Slam). She can be summoned within the custom move but
  cannot be desummoned.
- **`SPECIAL` blocks have no `Hold For` setting** — one Mass Buildup
  node will not charge the Mass Bar; you need at least 4. Star Rage's
  charged variants don't require the Mass Bar at all if you only want
  the charged form.
- **Several Puppet Master moves are broken or break the user.**
- **Veilstep's dive freezes** when combined with other moves.
- **`HITCNCL` can register stun from outside the custom move** (M1s,
  other moves) if `TIME` exceeds the move's startup. Workaround:
  `WAIT` for the startup duration, then `HITCNCL` with `TIME` set to
  that same duration, then another `WAIT` after the HITCNCL.
- **Capture unit only works with a `WAIT` in front of it.**
- **Custom special cooldowns swap on Awakening** — they use the
  awakening special's cooldown instead of the base one.
- **`VELO` `FORCE` magnitude over ~1000 causes player jitter.**
- **Rough Energy's flipbook visual** can load improperly sometimes.
- **`GRAB` teleports the user to the target** if the ability ends
  before the grab registers stun.
- **`PROJECTILE.BRANCH`** (user-side branch on hit) is currently broken
  — use `BRANCH TARGET` instead.
- **Divine Dog: Totality's second use** is currently impossible in
  Skill Builder.
- The Skill Builder is **not optimized for mobile/console** users.

---

## 6. CHARACTER & BUILT-IN MOVE CATALOG

The full catalog of every built-in skill, special, awakening, passive,
and variant — with per-move cooldown, damage, and a one-line
description — lives in the **JJS Moves Catalog skill** (separately
installed). Load it before generating any moveset.

That file covers all 16 canonical characters:

> Honored One (Gojo), Vessel (Sukuna), Restless Gambler (Hakari),
> Ten Shadows (Megumi), Mahoraga, Perfection (Mahito),
> Blood Manipulator (Choso), Switcher (Aoi Todo), Defense Attorney
> (Higuruma), Cursed Partners (Yuta + Rika), Puppet Master (Kokichi /
> Ultimate Mechamaru), Head of the Hei (Naoya Zenin),
> Salaryman (Naoya, base form), Disaster Plants, True Cannon (Toji),
> Black Death.

### How to reference a built-in move

You can plug a built-in move into a custom timeline three ways — the
catalog file repeats this, but here's the short version:

| Pattern | Use when |
|---|---|
| `{ "K_NAME": "SKILL", "MOVE": "<name>", ... }` | You want the **entire** built-in skill to play. Lazy, canon-accurate. |
| `{ "K_NAME": "SPECIAL", "SPEC": "<name>", ... }` | You want a built-in **special** (right-click move). |
| `{ "K_NAME": "ANIM", "ANIM_USE": [g,i], ... }` | You want **only the animation** of an existing move. See §7. |

Names are case- and punctuation-sensitive. `Cancel` is the universal
no-op skill that halts every still-running prior block.

**Never invent a move name** that isn't in the JJS Moves Catalog skill.
If the move you want doesn't exist as a built-in, build it from
primitives (ANIM/VELO/HITBOX/...) or ask the user.

---

## 7. ANIMATION CATEGORIES

The `ANIM_USE` field is `[group, index]` where `group` is the broad
category and `index` is the slot inside it. The in-game Animation node
picker has five categories:

- **Emote / Idle / Stance** — cosmetic / identity (e.g. Infinity hover,
  Bizarre, Insane2). Use for pre-combat poses, cinematic startups, and
  buff/Awakening idles.
- **Movement** — dashes, sprints, repositions; tightly coupled to `VELO`.
- **Melee / M1** — the shared attack-string pool. Reuse the same indices
  across m1/m2/m3/m4 for clean combos.
- **Skill animations** — the actual swing/cast animations of every
  built-in character skill (Cursed Strikes' lunge, Rapid Punches' kick,
  Rough Energy's wind-up...). **You can borrow these as just animations**
  while building your own hitboxes/velocity around them — that's
  pattern #3 from §6. When you do, *cite the source move* in the design
  brief ("ANIM borrowed from Vessel's Cursed Strikes lunge").
- **Special / Awakening / Domain animations** — cinematic / root-locking
  sequences; treat them like dedicated set-piece slots.

When generating fresh, **copy `[group, index]` pairs from a real decoded
moveset** rather than inventing new ones — the index space is dense and
guessing wrong silently picks the wrong animation. A set of known-valid
pairs (observed in real exports) you can fall back on if no reference
moveset is available:
`[1,3] [1,4] [1,5] [1,6] [1,9] [1,12] [2,7] [2,8] [2,20] [3,3] [11,4]
[15,25] [17,2]`.

**Warning:** these are pairs *valid as animation indices* — the game's
animation picker accepts them — but **what each one looks like is
unknown without testing**. Don't assume `[2,7]` is "a sword swing"
unless you've seen it in-game; it's just "the 7th animation in
category 2" and could be a celebration emote. When the user has a
specific animation in mind, the best workflow is:
1. Ask them for the `[g, i]` pair directly if they know it, OR
2. Pick a built-in character move whose canon animation matches the
   feel they want (e.g. Cursed Strikes' lunge, Hollow Purple's
   wind-up), then borrow that move's `ANIM_USE` — the picker has
   every built-in character animation indexed. The JJS Moves Catalog
   companion skill, if installed, lists every built-in move by name
   with its animation slot; load it before picking.

### Companion catalogs — load them before naming moves or picking emotes

(Introduced in §0 alongside the codec — repeated here because §7 is
where you actually reach for them.)

| Skill | Content | Use it when |
|---|---|---|
| **JJS Emotes Catalog** | All ~326 emotes / idles / stances / cosmetic animations, by name, with a one-line motion description. | Picking an idle, pre-combat stance, cinematic gesture, or awakening idle (Infinity hover, Bizarre pose, Honored aura, Insanity multi-pose, Jackpot, etc.). The animations live in the in-game picker — they can be dropped straight into an `ANIM` block. |
| **JJS Moves Catalog** | Every built-in skill / special / awakening / passive / variant for the 16 canonical characters, with cooldown, damage, and one-line description. | Picking a `SKILL.MOVE`, `SPECIAL.SPEC`, OR borrowing a built-in character-move animation as a pure `ANIM` (every move's animation is in the same picker `ANIM_USE` reads). |

Both are loadable as separate skills — invoke them when their topic
comes up, then return here for the encoding and design logic.

**Animation reusability** — anything from either catalog can become
an `ANIM` block in a custom move. Borrow Hollow Purple's wind-up,
Manji Kick's spin, the Bizarre emote pose, Cursed Strikes' lunge —
build your own hitboxes/visuals around the animation. This is the
canonical anti-laziness pattern from §10.7 ("don't just slap a
built-in SKILL.MOVE into every slot — borrow only the animation and
build the rest").

**Cite the source.** When you use anything from these catalogs,
**name the source in your design brief** so the player knows which
slot in the in-game animation picker to pick. Example: "ANIM slot
for the heavy-startup uses *Bizarre* emote (Honored One category) —
pick that in the Animation picker." This is non-negotiable —
`[g, i]` pairs are opaque without a name attached.

### Velocity ↔ Animation cancellation patch

`VISUAL` blocks can cut animations (wiki, Current Issues), and many
custom builders also report that large `VELO` forces visually clip the
playing `ANIM`. When this happens, the simplest fix is to add a `WAIT`
that covers the animation's runtime before the disruptive block. When
you *need* both the displacement AND the rest of the animation in one
move, you can also restart the animation on a different timestamp
*after* the velocity block:

```jsonc
// before-patch (broken): VELO eats the slash arc
{ "K_NAME": "ANIM",  "ANIM_USE": [2, 7],  "PREVIEW": [0, 0.8] },
{ "K_NAME": "VELO",  "FORCE": "0, 0, 35", "TIME": 0.2 },
{ "K_NAME": "HITBOX", ... },           // hits but animation is cut

// patched: re-play the back-half of the same ANIM after the VELO
{ "K_NAME": "ANIM",  "ANIM_USE": [2, 7],  "PREVIEW": [0, 0.35] },
{ "K_NAME": "VELO",  "FORCE": "0, 0, 35", "TIME": 0.2 },
{ "K_NAME": "ANIM",  "ANIM_USE": [2, 7],  "PREVIEW": [0.35, 0.8],
  "FADE IN": 0.05 },                    // resume from where the cut was
{ "K_NAME": "HITBOX", ... }
```

Alternative patches:

- **Lower the VELO magnitude** — keep `FORCE` ≤30 studs and `TIME`
  ≤0.15; most animations survive that.
- **Pre-WAIT before VELO** — let the wind-up frames play, then
  apply velocity after the cosmetic part is over.
- **Use `TRACK: true`** — when you only need the user to re-orient
  rather than translate, `TRACK` carries less cancellation risk than
  raw `FORCE`.

(See also the wiki's "Speed Tricks" note on residue from glitched
animations and how to mask it by re-playing the move's animation
after a Cancel.)

---

## 7.5 SOURCING ROBLOX ASSET IDs

`SFX.ID`, `VISUAL.TEXTURE`, and `VISUAL.AMOUNT` (when `EFFECT="Mesh"`) all
take **Roblox asset IDs**. The safest source is any decoded moveset
the user has already validated in-game — IDs from a working import code
are proven to load. When the user explicitly asks for a fresh sound /
new texture / custom mesh, you may scrape one of these public catalog
sites:

| Site | Use it for | Search URL |
|---|---|---|
| **robloxsong.com** | audio / sound effect IDs | `https://robloxsong.com/search?q={query}` |
| **robloximageid.com** | decal / image / texture IDs | `https://robloximageid.com/search?q={query}` |
| **robloxden.com/item-codes** | meshes, models, decals | `https://robloxden.com/item-codes` (browse by category) |

### How to scrape

Search URLs (replace `{query}` with a plain text query):
- Audio: `https://robloxsong.com/search?q={query}`
- Images/decals: `https://robloximageid.com/search?q={query}`
- Meshes/models: `https://robloxden.com/item-codes` (browse by category)

1. Use `WebFetch` (or `mcp__Claude_in_Chrome__*` if the site blocks fetch)
   with a *narrow* prompt: "Return a JSON array of `{name, id, url,
   description}` for the first 5 results matching `<query>`. Skip ads and
   sponsored entries."
2. Pull at most **3–5 candidates** per query; never bulk-scrape.
3. Pick **one** ID per slot you need. Prefer entries with high
   view/like counts and descriptions that match the use-case.
4. Treat every scraped ID as **unverified** until JJS confirms it loads.
   Roblox moderates assets aggressively — IDs go private/deleted often.

### Mandatory: cite every external asset you use

When you ship a moveset that includes **any** SFX/TEXTURE/Mesh ID — scraped
or borrowed — **end your reply with an "Assets used" table** listing every
ID, what it is, **which move slot it appears in**, where it came from, and
how confident you are. This table is **non-negotiable**: the user needs to
know exactly where each ID lives so they can swap it if it fails to load.
Example:

```
Assets used (verify in-game; swap if any fail to load):

| Where in moveset    | Type   | ID            | Source                                 | Note                           |
|---------------------|--------|---------------|----------------------------------------|--------------------------------|
| SKILL 2 "Blast", SFX| audio  | 4376217120    | robloxsong.com "explosion impact" #2   | Scraped — verify it still loads|
| Domain, VISUAL      | decal  | 7234918273    | robloximageid.com "purple energy" #1   | Scraped — verify it still loads|
| m4 finisher, VISUAL | mesh   | 84168736164752| user's prior validated moveset         | Re-used; known good            |
```

Do not hide this list — if you used an external ID, the user must know
which line to swap. If you only re-used IDs from a moveset the user
already had working, say so explicitly ("All asset IDs re-used from the
user's prior validated moveset; no external scraping done").

### Meshes: ask the user, don't scrape

**Meshes are a special case.** You **cannot** view, render, or verify a 3D
mesh from an asset ID — the scraper just returns a number. Don't pick mesh
IDs blindly from `robloxden.com` and hope they fit; a wrong mesh ruins the
move's look more visibly than a wrong sound or decal.

When your creative vision for a move calls for a custom mesh (a sword, a
skull, a chain, a sigil, a planet, a halo, a coffin, etc.), **describe what
you want and ask the user to source it**:

1. In the response, before/around the moveset, write a clearly-labeled
   **"Meshes I need"** block. For each mesh slot, give:
   - **Where it goes** (which move, which `VISUAL` node, what role).
   - **Shape**: a short physical description ("curved single-edged katana,
     ~3 studs long", "8-pointed star, flat, ~2 studs wide", "ribbed
     vertical pillar").
   - **Texture / color**: what should be wrapped on it ("matte black with
     red glowing cracks", "translucent cyan energy", "gold filigree").
   - **Scale & orientation hint**: rough size in studs and whether it
     should face the camera, follow the user, etc.
   - **Vibe / reference**: a one-line "looks like X from Y" if applicable.
2. Leave the corresponding `VISUAL.AMOUNT` (Mesh ID) and `VISUAL.TEXTURE`
   fields as placeholders like `"<MESH_ID_FROM_USER_1>"` /
   `"<TEXTURE_ID_FROM_USER_1>"` and reference them by the same label in
   the "Meshes I need" block. **Do not invent numeric IDs.**
3. When the user replies with IDs, do a single follow-up edit that
   substitutes them into the placeholders — no other changes.

You may still mesh-scrape **only** when the user explicitly tells you to,
or when they say something like "just pick something close." Otherwise
default to asking. For SFX and image decals, the §7.5 scrape flow is fine
because those are cheaper to swap and easier for the user to audition
in-game; meshes are not.

### When the user wants no scraping

If the user says "use only known-good IDs" or "no scraping", restrict
yourself to IDs from a moveset the user has already validated in-game
(ask them to paste one if you don't have one), or omit `SFX` /
`TEXTURE` / `Mesh` blocks entirely. Don't fall back to scraping
silently.

---

## 7.6 SKILL ARCHITECTURE: TWO PHASES — DESIGN FIRST, BUILD SECOND

The skill is structured around **two phases that you must execute in
order** every time you generate a non-trivial moveset. They are not
optional and not interchangeable.

1. **Design phase** — research the character/move, decide what each
   move *is* in plain English. **No JSON. No block names. No field
   values.** Just the move's identity, what the opponent sees, what
   the target feels, where the telegraph and counterplay live.
2. **Build phase** — translate the Design phase's plain-English brief
   into JJS blocks (ANIM, VELO, HITBOX, VISUAL, SFX, etc.) with the
   right field values per §5, the right etiquette per §10.7, and the
   right hit-feedback patterns.

These two tasks use different cognitive modes — Design is creative
research, Build is mechanical translation against strict format
rules. Doing them simultaneously produces compromises: moves get
bolted together field-by-field with no coherent identity, or designs
get written that the format can't actually express. **Always split
the phases.**

### 7.6.A The Design phase (run it like a self-contained pass)

**Output of the Design phase: a written brief, fully in plain English,
that the Build phase consumes.** The brief is the contract — once
the Design phase ends, the Build phase implements *exactly what the
brief says* and does not retroactively change it.

The brief has this structure:

```
1. Character identity (2–3 sentences)
   Vibe, archetype, color palette, signature visual motif, signature
   sound feel.

2. Per-slot design — for each slot you intend to ship (CHASE / m1–m4 /
   SPECIAL / SKILL 1–4 / AWAKENING / optional awakened-SKILL 1–4):

   Name: <move name>
   Concept: <1–2 sentences in plain English — what it does>
   Range / archetype: point-blank | mid | long | AoE | counter |
     mobility | utility
   Telegraph (visual + sound): <what the opponent sees and hears
     before the hit, and how long the window is>
   Active: <hitbox/projectile shape; hit count; damage tier
     (light / medium / heavy / finisher); single-target vs AoE;
     ragdoll vs stun-only>
   Hit feedback on target: <knockback direction; ragdoll yes/no;
     stagger animation; impact VFX; impact sound — per §10.7 "Hit
     feedback on target">
   Resolution: <the closing beat — especially for finishers and
     multi-stage moves; see §10.7 "Finishers">
   Counterplay: <opponent's read/parry/block window>

3. Awakening identity
   Pattern (A / B / C), what the transformation LOOKS and FEELS like,
   the aura that lasts the full DURATION, the theme of any awakened
   kit. (Re-check §8.1 for the AWAKENING-empty-timeline bug — you
   may choose to skip the AWAKENING slot entirely.)

4. Asset wishlist (mesh descriptions ONLY)
   For every move whose visual identity needs a custom 3D mesh (sword,
   halo, sigil, planet, coffin, etc.), write a "Meshes I need" entry
   per §7.5 — where it goes, shape, texture/color, scale, vibe. The
   Build phase will surface these to the user verbatim.

   Do NOT propose SFX IDs or texture IDs in the brief; those are the
   Build phase's job (see §5.5 / §7.5).
```

**Rules of the Design phase:**
- **Plain English only.** No JSON. No `K_NAME`s. No field names.
  No `ANIM_USE` pairs. No SFX IDs.
- **Lock the brief before encoding.** Once Phase 2 begins, the brief
  is fixed. If you find the format can't express something, **stop
  encoding and flag it to the user**; don't silently rewrite the
  design mid-build.
- **For named pop-culture characters/moves**, research first per
  §7.6.B (1–2 fetches max). Extract visual identity, sound cues,
  hand-seals, hit counts, range, and feel — not canon damage numbers
  (JJS damage is its own scale).
- **For original characters**, take imaginative liberty. Decide
  missing parameters yourself rather than ping-ponging back to the
  user for every detail. One or two clarifying questions at most.

**Two execution tactics — pick one:**

**Tactic A: in-context (default).** Write the brief in your reply
before any JSON, as a clearly-labeled section. Then proceed to the
Build phase in the same response. Useful when the user wants quick
turnaround, when you don't have access to sub-agent spawning, or
when the moveset is small enough to fit cleanly in one context.

**Tactic B: delegate Design to a sub-agent (when available).** If
your harness exposes a sub-agent tool (e.g. `Agent` / `Task` /
similar), spawn a fresh agent with this self-contained prompt and
collect its brief before your own Build phase starts:

```
You are a JJS moveset DESIGNER, not a builder. You produce plain-
English design briefs that another agent will translate into JSON
blocks. You have access to the same skill file (SKILL.md) — read it
before answering.

Character / concept: <USER'S REQUEST>

Output exactly the brief structure in SKILL.md §7.6.A: (1) character
identity; (2) per-slot design with Name / Concept / Range / Telegraph /
Active / Hit feedback / Resolution / Counterplay; (3) awakening
identity (mind the §8.1 unresolved AWAKENING bug); (4) mesh wishlist
per §7.5 format.

DO NOT write any JSON. DO NOT pick ANIM_USE pairs or SFX/texture IDs.
DO NOT write block field values. Plain English only.

For pop-culture characters, do 1–2 fandom-wiki fetches max to extract
visual identity, sound cues, range, and feel — not canon damage.

Output the brief and stop. Do not encode anything.
```

Tactic B's advantages: keeps the Build phase's context budget focused
on the format and the JSON it's writing; the user can audit the brief
independently before encoding starts. Disadvantage: needs an extra
round-trip and a tool that isn't always available.

**When you may skip the Design phase entirely.** Very narrow window:
the user's request is already so specific it *is* a brief ("make a
single SKILL slot that's just a 10-stud forward dash with no
hitbox"), or you're doing a tiny edit (changing one damage value).
Anything broader — a character, a kit, "make me a moveset for X" —
runs Design first.

### 7.6.B Design research (for pop-culture characters)

**This sub-step is mandatory inside the Design phase for any move
or character drawn from real pop-culture.** Do not invent canon you
haven't checked.

When the user describes a move or character, classify the request:

### A. Named, recognizable pop-culture move/character

When the user describes a move or character, classify the request:

### A. Named, recognizable pop-culture move/character

Examples: "Gojo's Hollow Purple", "Naruto's Rasengan", "Ichigo's Getsuga
Tensho", "Tanjiro's Water Breathing 1st Form", "Goku's Kamehameha",
"All Might's Detroit Smash", "Zoro's Three-Sword Style: Onigiri",
"Bakugo's Howitzer Impact", "Yoriichi's Sun Breathing 13th Form".

For these:

1. **Research the move** before designing it. Order of preference:
   - Try `WebFetch` on a fandom wiki page (e.g.
     `https://<series>.fandom.com/wiki/<Move_Name>`). Fandom wikis are
     the best source for: incantation, hand-seals, hit count, range,
     visual identity, sound cues, and canonical damage tier.
   - If fandom is blocked or thin, fall back to a curated source
     (Wikipedia for the character, official manga/anime synopsis sites).
   - At most 1–2 fetches per move. Don't spider the whole wiki.
2. **Extract the canon details** that map onto JJS blocks:
   - **Visual identity** → `VISUAL.EFFECT`, `COLOR`/`ALT COLOR`
   - **Sound / chant / "kiai" cue** → `SFX.ID` (sourced per §7.5)
   - **Wind-up / chant length** → startup `WAIT`/`ANIM` duration
   - **Range** (point-blank, mid-range, long-beam, AOE) → `HITBOX.SIZE`
     or `PROJECTILE` (`SPEED`/`TIME`)
   - **Hit count** (single, multi-hit, beam-pierce) → number of
     `HITBOX` blocks, `SINGLE TARGET` flag, `LOOP` over a punch chain
   - **Element / cursed-energy type** → `ATTACK TYPE`
     (Cleave / Dismantle / pure-blast usually map to `Explosion` or
     `Domain`; weapon swings to `Melee`; projectiles to `Bullet`)
   - **Counter-style move** (parry, shave) → use a `COUNTER` block
   - **Power-cost trade-off** (lasts X turns, costs HP, requires
     awakening) → `Req` gate (`HP`, `BAR`, `ULT`)

### B. Not a recognizable named move

The user said "make something kinda like a wind blade" or just "a
slow heavy slam". The character/move isn't in canon, or it's their own
OC.

For these:

1. **Take the imaginative liberty.** Decide the missing details
   yourself — name, archetype, range, element, hit count, telegraph,
   sound cue feel. Don't ping-pong back to the user for every
   parameter; one or two clarifying questions at most.
2. If the user gave a *partial* description (e.g. "fire punch") fill
   in only the missing pieces — keep their named choices intact.

### Output: the design brief (write this to the user **before** the JSON)

In **both** cases A and B, produce a short design brief *before* you
start writing blocks. Format:

```
Move: <name>   |   Slot: SKILL key 2 (E)   |   Cooldown: 12s
Source: One Piece wiki, "Three-Sword Style: Onigiri"
        ( https://onepiece.fandom.com/wiki/Three_Sword_Style:_Onigiri )
        — or "Original design (user said 'wind blade', rest invented)".

Concept (1–2 lines):
  Zoro lowers into a crouched stance, grips two swords in his teeth and
  hands, then explodes forward into a three-blade horizontal cross-cut
  on a single target.

Timeline (what the player will see):
  0.00s  ANIM crouched-stance startup, DirectionLock on user, slight
         wind-up SFX
  0.45s  VISUAL: three white slash arcs telegraph in front of user
  0.55s  VELO forward (FORCE "0, 0, 25", TIME 0.2) — closes the gap
  0.65s  HITBOX: single-target, SIZE "5, 6, 7", DAMAGE 18, ATTACK TYPE
         Melee, BLOCKABLE true
  0.65s  SFX: blade impact (id sourced from <site>)
  0.80s  VISUAL: cross-slash X effect on target
  1.10s  Recovery WAIT 0.4, no DirectionLock

Counterplay window:
  ~0.55s of forward telegraph (visual arcs + dash) before HITBOX. Move
  is Melee/blockable, gives the opponent a parry window. Cooldown 12s.

Budget check (§8):
  Damage 18 → cooldown ≥ 18s? Set to 18s. Single ranged-but-melee, fits
  alongside a long-range SKILL on key 3. Has a clear telegraph and
  blockable flag → counterplay rule satisfied.
```

Only **after the user implicitly or explicitly accepts the brief**
(or if they asked for a one-shot moveset, after you write the brief
for each slot) do you start filling in concrete `Line` blocks. The
brief is the contract — the JSON must implement what the brief
described, not drift mid-build.

### Anti-laziness

- Don't skip the brief just because the move "feels obvious". A
  1-paragraph brief catches almost every bad design before encoding.
- Don't paste the wiki article verbatim into the brief — distill it
  to the 6 fields above.
- Don't research moves that aren't real to "make them feel canon".
  Imagination is fine — just label it "Original design" in the
  Source line so the user knows.
- If a fandom page 404s or you can't fetch it, say so in the Source
  line ("Wiki page 404'd — design reconstructed from memory; double-check
  any canonical-feeling claims") rather than silently faking it.

---

## 8. DESIGN RULES (apply during generate / analyze)

Every offensive skill must have these three readable phases:

1. **Startup** — `ANIM` + optional `STATE` (e.g. `DirectionLock`)
   + `VISUAL` telegraph. Defines what the opponent reads.
2. **Active** — `HITBOX` (or `PROJECTILE`) aligned with the swing frame
   of the animation. Damage happens here.
3. **Resolution** — `VELO` follow-through, `SFX`, exit / branch.

A complete moveset must satisfy:

- At least **two viable combat ranges** (don't make every move melee
  Explosion-type — that has no counterplay).
- At least **one counterplay window per strong action**: each move
  ≥12 damage should be `BLOCKABLE: true` *or* have a `STUN` ≤0.4 *or*
  a multi-second recovery `WAIT` after its `HITBOX`.
- **No infinite loops without exits**. `LOOP` with `HOLD: true` must
  pair with a `HITCNCL` or branch escape.
- **Cooldowns scale with damage**: rough heuristic — `COOLDOWN ≥
  DAMAGE * 1.0` seconds (8 dmg → 8s+, 20 dmg → 20s+). Domains/ultimates
  bypass this on purpose.
- M1 chain (`m1`..`m4`) should share a rhythm: the `ANIM` `SPEED` and
  the post-hit `VELO TIME` are similar across all four; the **last**
  M1 (`m4`) breaks rhythm with a knockback `VELO` (`FORCE` with
  Y > 5 for launch, or |Z| > 20 for sendback).
- **m4 hitbox tip:** by convention, `m4` **hitboxes the target** (lands
  the finishing damage and applies the launch/sendback) — that's what
  closes the chain. Only break this convention if you're explicitly
  going for something different (e.g. a feint-finisher that resets
  combo, or an M4 that grabs and tags for a branch). When you do break
  it, call it out in the design brief so the user isn't surprised.
- Damage budget per move:
  - M1 single: 3–6
  - Skill (key 1–4): 8–18
  - Special: 10–25
  - Awakening burst / ultimate: 30–60 (must be telegraphed)
  - Domain inside: scales per tick

### 8.1 Awakening patterns

The `AWAKENING` slot is more open-ended than the others. There are
three common patterns — pick one and commit; don't half-do all three.

**Awakening abilities are *allowed* to be OP — that's literally
what an awakening IS.** The standard balancing heuristics in §8
("cooldown ≥ damage in seconds", "single move ≤ 25 dmg", etc.)
apply to **base-form** abilities. They do NOT apply to awakened
ones. An awakening is a finite-duration power spike with a cost
(meter to fill, transformation telegraph, limited duration); the
player is *supposed* to feel meaningfully, lopsidedly stronger
during it. So:

- **Bigger damage numbers are fine.** Awakened SKILLs hitting for
  25–40 dmg, awakened SPECIAL hitting for 40–60+ — fair game.
- **Shorter cooldowns are fine.** Awakened SKILLs can run 6–10s
  cooldowns even on heavy moves; the awakening's own DURATION
  is the soft cap on how much the player can spam.
- **Bigger hitboxes / longer ranges / more attack types uncounterable
  are fine.** Pattern C kits routinely have moves that would feel
  unfair as base abilities.
- **Healing, damage-multiplier Props, speed/jump buffs > 1.3, ki/HP
  regen** — all fine.

What balances this is **not** individual-move restraint, it's the
*structural* cost: how hard the awakening is to trigger (does it
require a full Awakening Bar? a low-HP threshold? a tag chain?),
how long it lasts before the player goes back to base, and what
the cooldown is on the awakening itself. Counterplay reasserts
itself the moment the awakening ends. **Don't water down awakened
moves to match base-form damage budgets** — that produces an
awakening that the player can't *feel*, which is the failure mode
the §8.1 mandatory checklist (item 4) exists to prevent.

When in doubt: the awakened version of a base move should hit
clearly harder, recover faster, or do something the base version
can't (extra hits, AoE, target-tracking, uncounterable type).
"Slightly better" is not awakening.

### What a complete awakening looks like (when you include one)

The AWAKENING slot is **optional** — minimal movesets, demo
builds, and "just a few skills" requests are fine without one
(see §10.7 "Minimal movesets"). But **if you include an awakening,
commit to one of these two complete shapes.** Half-doing both
produces an awakening that activates, plays effects, and then
*does nothing else* — which is the worst outcome.

**Shape 1: ONE big OP cinematic move (Pattern A done with
commitment).** The awakening is a single high-impact move and
nothing more. No `DURATION` buff phase, no kit swap. The Line:

- Plays a transformation cinematic (~2-3s).
- Fires a massive AoE `HITBOX` (15-25 dmg, big SIZE, Explosion
  type, unblockable, ragdoll-launches everyone nearby).
- May include a `STATE: "Scale"` step so the player visibly grows.
- Set `DURATION: 0` (or omit) — there is no buff phase.
- Cooldown on the slot is long (60-120s+) so it can't be spammed.

Player keeps base-form Q/E/R/T/SPECIAL the whole time. Examples
from canon: any "ultimate finisher" that fires once and ends.

**Shape 2: A set of awakening-only SKILL slots (Pattern C done
right).** This is the "real" awakening structure — the one the
player is asking for when they say "awakening." The AWAKENING
slot itself plays the transformation cinematic and applies
fire-and-forget buffs (per §8.1 Fire-and-forget pattern below),
and the kit's SKILL slots **swap** to awakened-only versions for
the buff DURATION. Requires:

- One or more awakened `SKILL` slots with `Prop`
  `{"K_NAME": "AWK2"}` (Hide in Base — only shown during
  awakening).
- The corresponding base `SKILL` slots tagged with `Prop`
  `{"K_NAME": "AWK"}` (Hide in Awakening — disappear during
  awakening).
- Awakened SKILLs hit clearly harder than base (per the OP rule
  above) — 20-40+ damage, short cooldowns, bigger hitboxes, or
  capabilities the base versions don't have (multi-hit, AoE,
  homing, uncounterable type).
- The AWAKENING slot itself sets `DURATION: 30-60s` and uses the
  Fire-and-forget pattern so its Line ends quickly and base
  abilities (CHASE/MELEE/SPECIAL) stay live throughout.

**How many awakened SKILL slots? Four is the standard, but the
count is flexible.** A full awakened kit replaces all four SKILL
keys (Q/E/R/T) with awakened versions — that's the canonical
"Super Saiyan" / "Sage Mode" / "Bankai" shape and what most
players expect from "awakening." But you can also ship:

- **Fewer than four** — e.g. only awakened R + awakened T, while
  Q and E keep their base versions during awakening. Valid when
  some base abilities are universal enough to stay live in both
  forms (a utility dash, a defensive parry, a universal Cancel).
- **More than four** — uncommon but possible if you're using the
  `ADD` field on SKILL slots (§2) to layer extra moves on a key.
- **One single awakened slot** — borderline; at that point you
  might prefer Shape 1 (single big move) since the player isn't
  getting a meaningfully different *kit*.

The number isn't enforced by the format. Pick what fits the
character. Just remember: every awakened SKILL slot needs its
Prop, and every base SKILL slot whose key is being replaced
needs its Prop.

**Mapping reminder (easy to get wrong — see §5.19):**

| Slot type           | Prop K_NAME | Meaning            |
|---------------------|-------------|--------------------|
| Base SKILL          | `AWK`       | Hide in Awakening  |
| Awakened SKILL      | `AWK2`      | Hide in Base       |

CHASE / MELEE / SPECIAL do **not** get these Props — they
auto-use the active moveset and aren't swapped by AWK/AWK2.

> **⚠ THE `AWK` / `AWK2` PROP ENCODING WAS DOCUMENTED BACKWARDS
> UNTIL MAY 2026 — AND IS STILL UNVERIFIED ⚠**
>
> Earlier drafts of this skill had AWK and AWK2 inverted. The
> May 2026 Goku playtest caught and corrected the mapping (see
> §5.19). The corrected mapping is **AWK on base, AWK2 on
> awakened** — counter-intuitive, but that's what the user
> playtest confirmed.
>
> Even with the inversion fixed, the Prop encoding has not been
> definitively confirmed working in a Pattern C build. If the
> user imports a Shape-2 moveset and sees both base and awakened
> moves appearing on the same key in either state, the workaround
> is to open each affected SKILL slot's Properties tab in-game
> and manually toggle "Hide in Base" / "Hide in Awakening" until
> a decoded reference reveals the correct Prop encoding.
>
> **Always warn the user when shipping Shape 2** that:
> 1. The AWK/AWK2 mapping was recently corrected and the new
>    mapping should be tested.
> 2. The in-game Properties tab is the source of truth — if the
>    encoded props don't take effect, manual toggles will.

**Shape pickers:** Pattern B from later in this section
("transformation + sustained buffs but no kit swap") is a hybrid
— it CAN work, but it shares the "feels like nothing changed"
risk of the half-done awakening: the player only sees a stat
buff and an aura, no new buttons. Recommend B only when the
character explicitly is supposed to be "the same fighter,
stronger" rather than "a new form."

**Don't ship an awakening that does neither shape.** A pure
transformation cinematic that activates and ends with no
follow-through (no hitbox, no buff phase, no kit swap) is just
a 3-second emote. Either commit, or skip the AWAKENING slot.

> **⚠ AWAKENING HAS TWO DISTINCT FAILURE MODES — KNOW WHICH ONE YOU HIT ⚠**
>
> Both failure modes have been observed in playtest. Diagnose
> before you "fix":
>
> **Failure mode 1: Timeline reads as completely empty in the
> Skill Builder; awakening can't activate.**
>
> Cause: an **unverified `K_NAME`** in `DATA.Line` voids the entire
> timeline at parse time. Confirmed (Goku, May 2026): `ADDHP` and
> `ADDEV` blocks — which use unverified encoded names per §5.17 —
> caused the AWAKENING slot to read as empty in-game even though
> the JSON contained 25 valid blocks and the codec round-trip
> verified. Removing those two blocks restored the Line.
>
> Earlier rounds of this skill misattributed this to a slot-shape
> bug ("K_NAME: AWAKENING is wrong" / "the DATA shape is different
> for AWAKENING"). **That diagnosis was wrong.** The slot shape
> (`K_NAME: "AWAKENING"`, `DATA.Line`, `DURATION`, `DELAY`) is
> correct. The bug is in the Line's contents.
>
> Fix: audit every block in the Line. Strip any block whose
> `K_NAME` is not in the verified §5 list. The current unverified-
> until-decoded list: `ADDAWK`, `ADDHP`, `ADDEV`, `CONNECT`. See §9
> rule 1 for the generalized parser behavior.
>
> **Failure mode 2: Awakening DOES activate, but locks out base
> abilities for the whole "awakening."**
>
> Cause: the Line's blocking duration is too long. The fix is
> §8.1's Fire-and-forget pattern. The diagnosis below describes
> this mode specifically.
>
> **How to tell them apart:** open the AWAKENING slot in the
> in-game Skill Builder after import. If the timeline appears
> empty / has no blocks visible, you hit mode 1. If the timeline
> has blocks visible AND the awakening plays its cinematic but
> base abilities are locked out during/after, you hit mode 2.
>
> ---
>
> The **lock-out** failure mode (mode 2):
>
> **While the AWAKENING Line is still executing, the player is in
> "casting" state and cannot use base m1s, CHASE, SPECIAL, or SKILL
> slots — even though no `AWK`/`AWK2` Prop is set.** The Line's
> total blocking duration (sum of `WAIT`s, `LOOP × WAIT` bodies, and
> blocking VELOs) directly determines how long the player is locked
> out of every other ability.
>
> User report verbatim (Goku playtest): *"amazing effects, great
> work however it just hid all moves and i couldnt use m1s or chase
> or special and just had effects, it should either be a move and
> not last or last and you need to make moves that are hidden in
> base and shown in awakening."*
>
> Root cause in that build: the AWAKENING Line ended with a
> `LOOP back=3 amount=20` whose body contained `WAIT 1.3`. Line ran
> for ~28 seconds. Visuals played beautifully; base abilities were
> unreachable for the whole "awakening."
>
> **Fix: fire long-running visuals and STATE buffs as fire-and-forget
> blocks, then END the Line quickly (~3s).** Details and recipe in
> the next subsection.

### Once activation works — making awakening *feel* like an awakening

(Assume the activation-blocking issue above is fixed and the Line
actually plays.) The next failure mode is "the cinematic plays for 2
seconds and then *nothing visibly changes* for the remaining 58s of
DURATION." Speed/jump multipliers running silently in the background
don't count — the player needs to see and feel the awakening for the
full duration, not just stat-buff numerics.

**Mandatory checklist for any non-trivial awakening Line:**
1. **An aura that lasts the full `DURATION`.** A `VISUAL` block
   (typically `Cursed Energy`, `Wind Expand`, or `Flames`) with
   `TIME` ≈ `DURATION`, anchored to `HumanoidRootPart`, with a
   `VISUAL TAG` so you can `Cancel` it cleanly if the awakening
   ends early. **Without this, the awakening looks empty in seconds
   3+.**
2. **A `STATE: "Scale"` and/or visible outfit change.** Trick:
   `Time Cell Moon Palace` at `SPEED: 0` + immediate Cancel applies
   the awakening outfit without playing the domain cutscene
   (see §5.23 Known Tricks).
3. **Animated aura, not static** — use `ALT POSITION` / `ALT SIZE`
   / `ALT COLOR` on the persistent aura visual so it tweens/pulses
   over its `TIME`, OR use an effect with intrinsic motion
   (`Cursed Energy`, `Wind Expand`, `Flames` all move on their own).
   **DO NOT use a `LOOP` at the end of the Line to spawn repeat
   pulses** — see the warning block above. The Line must end early;
   pulsing belongs *inside* the persistent visual block, not in the
   Line's structure.
4. **Either** a transformed kit (Pattern C) **or** clearly stronger
   versions of base abilities (Pattern B with damage/knockback
   `Damage Multiplier` Prop > 1.0). Buffs the player can't feel are
   invisible.

**Don't ship an awakening Line without at least items 1 and 4.** Test
in private game: enter awakening, wait 5 seconds without moving — is
there any visible difference from base? If no, the awakening looks
empty even if it activated.

### Fire-and-forget pattern (REQUIRED for any persistent awakening)

`VISUAL` and `STATE` blocks fire instantly — the *block instruction*
executes in zero time and the Line continues. What `TIME` controls
is how long the *effect itself* persists. Critically: **the visual
or state keeps running AFTER the Line has ended.**

So for a 60-second gold aura with sustained stat buffs, you do
**not** need a 60-second Line. You fire short-lived blocks for the
cinematic, then fire one-shot blocks with `TIME = DURATION` and let
the Line terminate:

```jsonc
// Spawned at Line-time, persists 60s independently of the Line:
{ "K_NAME": "VISUAL", "EFFECT": "Cursed Energy",
  "COLOR": "255, 220, 50", "AMOUNT": 20, "TIME": 60,
  "BODY PART": "HumanoidRootPart",
  "VISUAL TAG": "ssj_aura", "RUN ON SERVER": true,
  "CANCEL ON INTERRUPT": false }

// Stat buff spawned at Line-time, persists 60s:
{ "K_NAME": "STATE", "STATE": "SpeedMultiplier", "VALUE": 1.3,
  "TIME": 60, "CANCEL ON END": false }
```

**Set `CANCEL ON INTERRUPT: false` on persistent aura visuals.** The
default `true` was meant for short cosmetic effects — for an
awakening-duration aura, leaving it on means an interrupt to the
Line yanks the aura mid-transformation. Same applies to
`CANCEL ON END: false` on the long STATE buffs (don't lose the buff
the moment the Line ends).

**Canonical AWAKENING Line skeleton:**

```jsonc
"Line": [
  // ---- 1. Cinematic transformation (blocks for ~3s) ----
  { "K_NAME": "STATE", "STATE": "DirectionLock", "TIME": 2.5 },
  { "K_NAME": "STATE", "STATE": "IFrame",        "TIME": 2.0 },
  { "K_NAME": "ANIM",  "ANIM_USE": [15, 25], "PREVIEW": [0, 2.0] },
  { "K_NAME": "VISUAL", "EFFECT": "Shake Heavy",   "TIME": 2.5 },
  { "K_NAME": "VISUAL", "EFFECT": "Cursed Energy", "TIME": 2.0,
    "AMOUNT": 45, "COLOR": "255, 220, 50" },
  { "K_NAME": "VISUAL", "EFFECT": "Wind Expand",   "TIME": 1.2,
    "AMOUNT": 6, "COLOR": "255, 220, 50" },
  { "K_NAME": "SFX",   "ID": "...", "VOLUME": 3.5 },
  { "K_NAME": "WAIT",  "TIME": 1.5 },
  { "K_NAME": "VISUAL", "EFFECT": "Clash",   "TIME": 0.5,
    "ALT OPACITY": 10, "ALT SIZE": 6, "COLOR": "255, 220, 50" },
  { "K_NAME": "WAIT",  "TIME": 0.8 },

  // ---- 2. Fire-and-forget persistent buffs ----
  { "K_NAME": "STATE", "STATE": "SpeedMultiplier", "VALUE": 1.3,
    "TIME": 60, "CANCEL ON END": false },
  { "K_NAME": "STATE", "STATE": "JumpMultiplier",  "VALUE": 1.2,
    "TIME": 60, "CANCEL ON END": false },

  // ---- 3. Fire-and-forget sustained aura (TIME = DURATION) ----
  { "K_NAME": "VISUAL", "EFFECT": "Cursed Energy", "TIME": 60,
    "AMOUNT": 20, "COLOR": "255, 220, 50",
    "BODY PART": "HumanoidRootPart",
    "VISUAL TAG": "aura", "RUN ON SERVER": true,
    "CANCEL ON INTERRUPT": false },
  { "K_NAME": "VISUAL", "EFFECT": "Glow", "TIME": 60,
    "OPACITY": 0.4, "ALT OPACITY": 0.5,
    "COLOR": "255, 220, 50",
    "BODY PART": "HumanoidRootPart",
    "VISUAL TAG": "glow", "RUN ON SERVER": true,
    "CANCEL ON INTERRUPT": false }

  // END of Line. Blocking duration ~3.3s.
  // Player can now use m1s, CHASE, SPECIAL, SKILLs while the speed
  // buff, jump buff, and gold aura persist for 60 more seconds.
]
```

**What NOT to do (the bug from Goku playtest):**

```jsonc
// ❌ WRONG: trailing LOOP keeps the Line running for ~26 seconds.
//    Player is "casting the awakening" for that whole time and
//    cannot use m1, CHASE, SPECIAL, or any SKILL.
{ "K_NAME": "VISUAL", "EFFECT": "Wind Expand",   "TIME": 0.7 },
{ "K_NAME": "VISUAL", "EFFECT": "Energy Sparks", "TIME": 0.7 },
{ "K_NAME": "WAIT",   "TIME": 1.3 },
{ "K_NAME": "LOOP",   "LOOP BACK": 3, "LOOP AMOUNT": 20 }

// ❌ WRONG: a single huge trailing WAIT — same problem, simpler.
{ "K_NAME": "WAIT", "TIME": 60 }
```

If you want the aura to *visibly* throb (rather than just sit
there), animate the **persistent visual itself** — use `ALT
POSITION`/`ALT SIZE`/`ALT COLOR` on the aura's VISUAL block so it
tweens across its `TIME`. Don't loop the spawning in the Line.

**Pattern A: it's just a move.** The awakening slot fires a single
cinematic move (transformation animation + healing + maybe a hitbox)
and that's it — no `DURATION` buff phase. Base-form `SKILL` slots stay
live and unchanged. Easiest pattern; matches the wiki's "Awakening
Skill" category for most characters. Required fields:
`K_NAME: "AWAKENING"`, `NAME`, `DURATION: 0` (or short), `DELAY`, `DATA`.
Pattern A doesn't have an "empty awakening" problem because it's
explicitly one-shot — but you still must commit to a hitbox or healing
effect so the cinematic does *something*.

**Pattern B: move + lingering buff.** The awakening fires a
transformation move, then a `STATE` + `VISUAL` aura lingers for
`DURATION` seconds. **The buff must be both visible and felt.**
Example: `Six Eyes` heals 25 HP + grants speed/damage buffs *and shows
a cursed-energy aura the whole time*. Implement with:

- A short `Line` for the transformation (ANIM + VISUAL burst + healing
  via `ADDHP` or a custom self-hitbox).
- A long-duration `VISUAL` aura with `TIME` ≈ `DURATION`, tagged via
  `VISUAL TAG`, with `RUN ON SERVER: true` so it's visible to all
  players. **This is the part that prevents "empty awakening."**
- A long `STATE` block (`STATE: "SpeedMultiplier"`, `VALUE: 1.25`,
  `TIME: DURATION`) covering the rest of the awakening.
- A `Damage Multiplier` Prop > 1.0 on the slot so base skills hit
  harder during awakening — players *feel* the awakening, not just
  see the aura.
- Set `DURATION: 60` (or whatever the buff lasts) on the slot.
- Base-form skills stay live and unchanged but now empowered.

**Pattern C: full moveset swap.** The most ambitious pattern. The
awakening transforms the character and **swaps every skill slot to an
awakened version**. To pull this off correctly:

1. Mark every base-form ability with `Prop` `{"K_NAME": "AWK"}` ("Hide
   in Awakening") so they disappear during awakening.
2. Add a parallel set of `SKILL` slots that are the awakened versions,
   each tagged with `Prop` `{"K_NAME": "AWK2"}` ("Hide in Base") so
   they only appear while awakened. Typically four (one per KEY 1–4)
   but the count is flexible — see Shape 2 in §8.1 above.
3. The `AWAKENING` slot itself plays the transformation animation +
   sets the buff state for `DURATION` **AND maintains the sustained
   aura** (per the empty-awakening checklist above).
4. The CHASE and MELEE slots typically *aren't* swapped (those auto-use
   the active moveset), unless you also want awakened M1s — in which
   case you'd need to use the Character Block (not just Moveset Block)
   to override them, which is out of scope for a single import code.
5. Optional: set `Prop` `{"K_NAME": "KEEP"}` (Keep When Moveset Switches)
   on universal utilities like Cancel-skills.

> **⚠ AWK / AWK2 Prop ENCODING IS UNVERIFIED — AND WAS DOCUMENTED
> BACKWARDS UNTIL MAY 2026 ⚠**
>
> Earlier drafts of this skill had the AWK ↔ AWK2 mapping inverted:
> they said AWK was "Hide in Base" (awakened-only) and AWK2 was
> "Hide in Awakening" (base-only). The May 2026 Goku playtest
> corrected this — the right mapping is `AWK = Hide in Awakening`
> (goes on base SKILLs) and `AWK2 = Hide in Base` (goes on
> awakened SKILLs). See §5.19 for the corrected table.
>
> However, even with the inversion fixed: earlier playtest of an
> AWK/AWK2-tagged Pattern C moveset still reported **both sets
> appearing on each key in both states** — the hiding didn't
> happen. That report was made with the backwards mapping, so we
> can't tell whether the inversion fix resolves it. Until a
> Pattern C moveset is playtested with the *correct* mapping and
> confirmed working, treat the encoding as unverified.
>
> Open possibilities if it still doesn't work:
> - The Prop entry needs additional fields (e.g., a `VALUE`,
>   `ENABLED`, or boolean flag) beyond just `K_NAME`.
> - The encoded short-name is wrong entirely (the wiki's
>   user-facing labels are "Hide in Awakening" / "Hide in Base";
>   the encoded keys `AWK`/`AWK2` were *inferred* from real
>   exports, not from documentation).
> - The Prop block belongs at a different level of the JSON tree.
>
> **Until we decode a known-working Pattern C moveset to see the real
> encoding, don't ship Pattern C blind.** Either:
> 1. Ask the user to paste a working Pattern C import code so we can
>    decode it and copy the literal Prop shape, OR
> 2. Default to Pattern B (which doesn't need AWK/AWK2 to work — base
>    skills stay live with empowered stats), OR
> 3. Ship Pattern C with the awakened SKILL slots **as the only set**
>    and skip the base versions entirely — the user loses the base-form
>    abilities but the awakening "works."

**Awakened-skill cooldowns** typically take effect immediately on
awakening, so be careful not to make the player wait 25s into their
60s awakening before they can use anything — set `COOLDOWN` ≤ 12s on
at least one awakened SKILL key.

---

## 9. ANTI-HALLUCINATION RULES (strict)

When you generate or edit a moveset, never:

1. **Invent block `K_NAME` values.** Only the types listed in §5
   exist. An unknown `K_NAME` in a `Line` does **not** silently
   no-op — playtest (Goku, May 2026) confirmed it **voids the
   entire parent slot's timeline**, making the slot read as empty
   in the in-game Skill Builder even though the JSON round-trips
   correctly. The parser drops the Line on first unknown block.
   This applies to MELEE/CHASE/SPECIAL/SKILL/AWAKENING alike, but
   it's most visible on AWAKENING (since the slot has no fallback
   behavior — an "empty" AWAKENING just doesn't activate). When in
   doubt, omit the block rather than guess at the encoded name.
   The unverified §5 blocks to avoid until decoded from a working
   moveset: `ADDAWK`, `ADDHP`, `ADDEV` (all three encoded names are
   guesses); any Prop K_NAME beyond `AWK` / `AWK2`; `CONNECT`.
2. **Invent field names.** `BRANCH TARGET` has a space and is upper-case.
   `DEBREE` is mis-spelled in the format — keep it. Renaming
   fields breaks the import.
3. **Invent `EFFECT` strings.** Use only the §5.4 list. An unknown
   effect makes the VISUAL block do nothing and wastes timeline space.
4. **Invent `STATE` strings.** Use only the §5.7 list.
5. **Invent built-in skill names.** Use only the §6 catalog.
6. **Invent SFX / TEXTURE / Mesh asset IDs.** IDs are real numeric
   strings registered on Roblox. Get them from the reference moveset or
   scrape per §7.5, then **cite each ID's source** in the reply. Never
   make up a number.
7. **Invent `ANIM_USE` pairs.** Copy from a real moveset (see §7).
8. **Output an `EXAMPLE_IMPORT_CODE_HERE`-style placeholder.** Either
   actually run the codec to produce a real base64 string, or be honest
   that the encode failed.

Hollow / forbidden moves to refuse to ship:

- A slot whose `Line` has zero `HITBOX`, zero `SKILL`, and zero `STATE`
  changes — that's a no-op. Don't pad it with VISUAL alone.
- A slot whose only damage is from `Cancel` skills.
- A `HITBOX` with `DAMAGE: 0` and no `STUN`, no `BRANCH*`, and no
  side effect — same: drop it.
- Two adjacent identical `VISUAL` blocks. Merge them, or extend `TIME`.

When in doubt, ask the user for: archetype (melee/ranged/mobility/
counter/cinematic), character inspiration, and which slot keys (1–4) to
fill — don't fill four slots with copy-pasted strings.

---

## 10. VALIDATION CHECKLIST

Before declaring a moveset "done", run this checklist mentally:

- [ ] Top-level is an **array**, not an object.
- [ ] Every slot has `K_NAME`, `NAME`, `DATA`.
- [ ] Every `SKILL` slot has a unique **integer** `KEY` from 1–8 (1=Q, 2=E, 3=R, 4=T by default). NEVER a letter like `"Q"`, `"E"`, `"R"`, `"T"` — the field is a JSON number, not a string.
- [ ] Every `MELEE` slot is named `m1` / `m2` / `m3` / `m4`.
- [ ] Every `Line` block has a `K_NAME` and uses only fields from §5.
- [ ] Every `BRANCH TARGET` / `BRANCH FINISHER` / `BRANCH` string
      matches an actual key in `DATA.Branch`.
- [ ] No `HITBOX` with `DAMAGE: 0` *and* no stun/branch effect.
- [ ] No `LOOP` with `HOLD: true` lacking an exit branch.
- [ ] Every move has startup→active→resolution (§8).
- [ ] At least one counterplay window per ≥12-dmg move.
- [ ] Codec round-trip passes: `decode(encode(json)) == json`.
- [ ] If any external (scraped) Roblox asset ID is in the moveset, an
      **"Assets used"** table is included in the reply per §7.5.

If any check fails, fix it and re-encode. If you can't fix it, report
what failed instead of shipping a broken code.

---

## 10.5 SELF-REVIEW PASS (mandatory before shipping)

After you generate or edit a moveset, **walk through each slot one
more time** answering these questions out loud (in your scratch pad or
internal notes) before you hand the encoded code to the user. This
isn't optional — it catches the failure modes that the §10 syntactic
checklist can't see.

### A. Velocity review

For every `VELO` block in the moveset:

1. **Magnitude check.** Is `|FORCE|` plausible? Forward/backward
   movement: 10–30 studs over 0.2–0.4s is normal. >50 is sprint-level
   and almost certainly cancels the playing animation (see §7's
   velocity ↔ anim patch). Upward `Y > 5` is the worst offender.
2. **Direction check.** `FORCE` is `"x, y, z"`. Is the sign correct?
   `Z > 0` is **forward**, `Z < 0` is **back**, `Y > 0` is up,
   `X > 0` is the user's **LEFT** side, `X < 0` is the user's **RIGHT**
   side (relative to user facing — see §2.5). If the user said
   "dash backward" and your `FORCE` is `"0, 0, 20"`, that's wrong.
3. **Duration check.** Does `TIME` match the intent? Burst displacement
   = 0.1–0.2s. Continuous chase = 0.3–0.6s. Sustained = >0.6s only
   for sustained-flight moves.
4. **Cancellation check.** Is there an `ANIM` block playing *during*
   this velocity? If yes, will the velocity cut it? Apply the §7
   patch: re-play the animation from a later `PREVIEW` timestamp
   after the velocity.
5. **Ragdoll consistency.** If `RAGDOLL > 0` is on a self-applied
   velocity, that's a player-knockdown — make sure it's intentional.

### B. Timeline review

For every slot:

1. **Block order.** Read the `Line` from top to bottom. Does each
   block's job make sense in that order? `STATE` (gating) →
   `ANIM`+`VISUAL` (wind-up) → `VELO` (motion) → `HITBOX` (active
   frames) → `SFX` (impact) → resolution → exit.
2. **WAIT gaps.** Are there `WAIT` blocks where time should pass?
   The most common bug: stacking `HITBOX` blocks without `WAIT`
   gaps so they all hit in the same frame.
3. **Total duration.** Sum up `WAIT.TIME` and `VELO.TIME` and the
   longest concurrent `ANIM`. Is the move's wall-clock duration
   reasonable for its cooldown and damage? A 0.4s move that does
   25 damage with 5s cooldown is overpowered.
4. **Telegraph window.** Can you point to a 0.2s+ window before the
   first `HITBOX` where the opponent can see what's coming? If not,
   the move is undodgeable — fix unless it's an Awakening burst.
5. **150-second cap.** A custom move cannot exceed 150s total
   timeline length (wiki cap). Sum and verify.

### C. Hitbox review (most important)

For every `HITBOX` and `PROJECTILE`:

1. **Does it land?** `POSITION` defaults to `"0, 0, 4"` (4 studs
   forward). For a sword swing you want Z ≈ 3–5; for a stomp Y might
   be ≈ -1; for an explosion centered on user, `"0, 0, 0"`.
2. **Size check.** `SIZE` defaults `"6, 6, 6"`. A pinpoint stab is
   `"3, 3, 4"`; a sweeping AoE is `"15, 6, 8"`; a wall is
   `"20, 8, 2"`. Be deliberate, not lazy "6, 6, 6".
3. **Damage / stun / branch sanity.** No `HITBOX` should have
   `DAMAGE: 0` and no `STUN` and no `BRANCH*` and no `CANCEL ENEMY` —
   that's a no-op (already in §10, restated because it's common).
4. **Animation alignment.** Does the `HITBOX` fire on the swing
   frame of the preceding `ANIM`, not before or after? Use the
   §5.22 timings as a reference for built-in animations; for custom
   pairs, the hitbox usually wants to come 30–60% through the animation.
5. **Attack-type counterplay.** `ATTACK TYPE` is a counterplay knob:
   `Melee` (counter by anything) → `Bullet` → `Explosion` →
   `Swarm` → `Domain` (uncounterable). Don't make every move
   `Explosion` — that strips opponents of options. A balanced moveset
   typically has 1 `Domain` (the actual domain move), 1–2
   `Explosion` (heavy finishers), and the rest `Melee` or `Bullet`.
6. **m4 specifically.** Confirm m4 has a hitbox that lands the
   finisher (see §8 m4 hitbox tip). If you intentionally skipped it,
   the design brief should say so.
7. **`HIT USER` is off** unless the move is supposed to hurt the user
   (rare).
8. **`BRANCH TARGET` strings resolve.** Already in §10 — restated
   because it's the #1 cause of "the move just stops mid-way" bugs.

### D. Branch review

For every named branch in `DATA.Branch`:

1. Is it actually reachable? Search the moveset for any block
   referencing this branch name. If nothing points to it, delete it.
2. Does its `Line` terminate? Branches without a final `WAIT` or
   `STATE: "Stun" TIME: 0` can leak state.
3. Air/ground variant branches: are `Req` conditions correct (true
   for ground, false for air, or vice versa)?

### E. Cross-slot review

1. **Cooldown spread.** Looking at all four SKILL slots (KEY 1–4):
   are at least two of them ≤15s? A moveset where every key is on a
   25s+ cooldown leaves long dead periods.
2. **Range spread.** Across SKILL 1–4 + SPECIAL, do you have a mix
   of point-blank / mid-range / long-range options? If everything's
   melee, the player has nothing to do against a flying opponent.
3. **Defensive read.** Does the kit have *any* way to handle incoming
   pressure? Not a hard quota — a pure-aggression character can be
   coherent without a COUNTER — but if every move is offensive with
   zero defensive option (no parry, no mobility escape, no i-frame
   skill, no displacement), the moveset will feel one-dimensional
   in playtest. Flag this for the user; don't auto-add a counter
   unless they want one. See "Counters — design space" in §10.7 for
   more.
4. **Awakening Prop assignment.** If the moveset uses Pattern C /
   Shape 2 from §8.1, verify the correct AWK ↔ AWK2 mapping (see
   §5.19; earlier drafts had it inverted):
   - Base SKILLs get `{"K_NAME": "AWK"}` — Hide in Awakening
   - Awakened SKILLs get `{"K_NAME": "AWK2"}` — Hide in Base

   Forgetting one or getting them backwards means the player sees
   the wrong ability in the wrong state, or both at once. Also
   verify CHASE/MELEE/SPECIAL slots do NOT get these Props — they
   auto-use the active moveset and aren't part of the swap.
5. **m1 chain has trailing WAITs.** Walk each MELEE slot's `Line`
   (m1, m2, m3, m4). Does every one end with a `WAIT` block of
   ~0.3–0.4s (m1–m3) or ~0.2–0.25s (m4)? Without these, the m1
   string fires at machine-gun speed when held and feels OP — see
   §10.7 MELEE rule. This is a top-3 playtest complaint; check it
   before encoding.
6. **AWAKENING Line ends quickly.** If a `AWAKENING` slot is in the
   moveset, sum its `Line`'s total blocking time (`WAIT`s, plus
   `WAIT × LOOP AMOUNT` inside any loop body, plus blocking `VELO`
   TIMEs). It should be **≤ 5 seconds** — typically ~3s for the
   cinematic. Any longer and the player is locked out of m1s, CHASE,
   SPECIAL, and SKILL slots for that whole duration (the awakening
   "casts" the entire time). Persistent visuals/STATEs must use
   `TIME = DURATION` on the block itself (fire-and-forget) — NOT a
   trailing `LOOP` or huge `WAIT`. See §8.1 "Fire-and-forget pattern."
7. **No unverified `K_NAME`s anywhere.** Scan every `Line` (main
   slot Lines and branch Lines) for blocks whose `K_NAME` is on
   the unverified list: `ADDAWK`, `ADDHP`, `ADDEV`, `CONNECT`, or
   any Prop K_NAME other than `AWK` / `AWK2`. **A single unverified
   K_NAME voids the entire parent slot's timeline in-game** — the
   slot appears empty in the Skill Builder even though the JSON
   round-trips. This is the most catastrophic single-block bug in
   the format and it's silent: the codec verifies fine, the JSON
   looks rich, but the slot is dead on import. Mandatory check —
   especially on AWAKENING slots, where an empty timeline means
   the awakening literally cannot activate. See §9 rule 1.
8. **Awakening structure commitment (if AWAKENING slot present).**
   Does the awakening follow §8.1 Shape 1 (one big cinematic move,
   no DURATION buff) or Shape 2 (transformation cinematic + four
   awakened-only SKILL slots with `AWK`/`AWK2` Props)? A "half-done"
   awakening — transformation cinematic ends, no follow-through —
   plays a 3-second light show and does nothing useful. If you
   can't commit to a shape, skip the AWAKENING slot.

### F. If any review fails

Fix it inline. If you can't, **say so explicitly in your reply** — list
the failed checks under a "Known issues" header so the user can patch
in-game. Do not silently ship a moveset that failed review.

---

## 10.7 STYLE & ETIQUETTE CONVENTIONS (read every time)

These are the conventions experienced JJS skill-builders follow. They
aren't enforced by the format — invalid setups still encode — but
ignoring them produces movesets that feel "off" to anyone who plays
the game. **Treat these as defaults; deviate only when there's a
specific design reason, and call it out in the design brief.**

### Slot-specific conventions

- **CHASE — the standard shape.** The chase slot replaces the user's
  normal front-dash. By convention it should **dash the user forward**
  — and *meaningfully*. Playtest has converged on a concrete standard
  that feels right in nearly every kit:

  ```jsonc
  // CHASE timeline — the standard
  { "K_NAME": "ANIM",   "ANIM_USE": [g, i], "PREVIEW": [0, 0.8],
    "FADE IN": 0.05, "FADE OUT": 0.1 },        // dash/run animation
  { "K_NAME": "VELO",   "FORCE": "0, 0, 80", "TIME": 0.8,
    "TRACK": true, "FADE": true },             // 80 stud/s for 0.8s, TRACKED
  // optional: small terminal hit to catch enemies (see below)
  ```

  Three things that matter:
  - **`FORCE` around 80** — strong enough that the player feels the
    dash actually close distance.
  - **`TIME: 0.8`** — long enough that the dash carries through a
    short pursuit, not a tiny burst.
  - **`TRACK: true`** — the dash follows the user's current facing
    while it runs, so the player can curve the dash by turning.
    Without TRACK, the dash locks the original facing-direction and
    feels rigid.
  - **An `ANIM` block playing a dash/run pose** for the duration so
    the character visibly *moves*, not slides.

  This is the **standard, not a cap**. Special scenarios are open —
  e.g. a teleport-style chase (Goku's Shunkan Idou) might use a very
  short TIME with no animation; a sustained-flight chase might run
  TIME 1.5s+. **Don't let the standard limit creativity** when the
  design clearly calls for something else, but DO use it as the
  default shape unless there's a specific reason to deviate.

  Sideways or backward dashes should generally go on a SKILL slot
  instead — CHASE is the *forward* button.

  **Optional polish: a small m1-like hit at the end of the chase.**
  Lets the player chase into an enemy and catch them — feels great.
  Stick a low-damage HITBOX at the end of the dash
  (`DAMAGE: 3–5, STUN: 0.3, SIZE "8, 8, 8"`) plus an impact SFX and
  a small target-knockback `VELO` with `LAST HIT: 1`. Not required,
  but a hallmark of polished chases.

- **MELEE m1 → m4 form a string with a finisher.** All four M1s should
  share rhythm — comparable `ANIM SPEED`, comparable post-hit knockback,
  shared overall length. m1–m3 are light strikes (~3–6 dmg each),
  **m4 is the finisher**.

  **Keep m1–m3 SIMPLE. They're not skills.** Playtest feedback:
  agents over-engineer m1–m3 with weird custom animations and bespoke
  visuals. *Don't.* m1–m3 are short hit animations + a hitbox + maybe
  a melee-trail visual. Reuse the same `ANIM_USE` pair across all
  three (or rotate among 2–3 similar swing animations). Their identity
  comes from the *character's hands moving through standard punches*,
  not from each m1 being its own design exercise. Save the creativity
  for skills and the SPECIAL. **Rule of thumb: an m1 block should be
  ≤ 5–6 nodes long (ANIM → WAIT → HITBOX → SFX → small VELO →
  trailing WAIT).**

  **Every m1–m3 needs a trailing `WAIT 0.3–0.4s`.** Playtest finding
  (Goku, May 2026): *"m1s — great but too fast, just by holding it
  would do them almost instantly and was op and not fun, must have
  a delay/wait after m1 so they have a natural delay."* Without a
  trailing WAIT, holding M1 fires the whole m1→m4 string in a fraction
  of a second — no input window, no counterplay, the combo feels
  like an exploit. The fix is a single `WAIT` block at the *end* of
  each m1's Line, after the post-hit VELO:

  ```jsonc
  // Every m1's Line - note the trailing WAIT
  { "K_NAME": "ANIM",   "ANIM_USE": [2, 7], "PREVIEW": [0, 0.4] },
  { "K_NAME": "WAIT",   "TIME": 0.15 },         // wind-up
  { "K_NAME": "HITBOX", "POSITION": "0, 0, 3.5",
                        "SIZE": "8, 8, 8",
                        "DAMAGE": 5, "STUN ANIM": true,
                        "BRANCH TARGET": "hit", ... },
  { "K_NAME": "SFX",    "ID": "..." },
  { "K_NAME": "VELO",   "FORCE": "0, 2, 10", "TIME": 0.15,
                        "LAST HIT": 1 },        // target knockback
  { "K_NAME": "WAIT",   "TIME": 0.35 }          // <-- chain rhythm gate
  ```

  Total per-m1 duration: ~0.5s, matching built-in M1 cadence (see
  §5.22 for game-default M1 timings). m4 can use a shorter trailing
  wait (~0.2–0.25s) since the launch/ragdoll already provides a
  natural pause — but it should still have *some* trailing wait so
  the user can buffer the next move cleanly.

  **m1–m3 do NOT ragdoll. Only m4 does.** This is the single most
  important M1-string convention, and it's a hard rule unless the
  design *explicitly* calls for the exception. Concretely, m1–m3
  must NOT have:
  - `HITBOX.HIT RAGDOLL: true`
  - `VELO.RAGDOLL > 0` on the post-hit knockback
  - `VELO.TRUE RAGDOLL: true`

  Why: if m1 ragdolls, the target is on the floor before m2 even
  fires — m2/m3/m4 then either miss entirely (target is below the
  hitbox) or chain-stunlock a downed opponent who has no defensive
  option. Either way, the chain stops feeling like an *exchange*
  and starts feeling like a one-button auto-combo. **m1–m3 should
  apply only stun + small knockback** (a `VELO LAST HIT: 1` with
  Z ≤ 12 and `RAGDOLL: 0`); the target stays standing and can still
  block / counter / burst-cancel between hits. m4 is where the
  rules change — it's the finisher, it ragdolls, it launches.

  Exceptions exist (a grappler-archetype whose whole identity is
  "every hit knocks down") but they should be a deliberate design
  choice called out in the brief, not an accident of leaving
  `HIT RAGDOLL` on across all four m1 templates.

  **m4 is where you commit to the finisher feel** — and almost always:
  - **Uses a ragdoll hitbox** (`HITBOX.HIT RAGDOLL: true` *and/or* a
    follow-up `VELO` with `RAGDOLL: 0.5`+ to actually knock the target
    down). This is the convention — m4 ragdolls, m1–m3 don't.
  - **Lands the launch/sendback** — `VELO FORCE` with `Z > 20` for
    sendback, or `Y > 5` plus `Z > 10` for upward launch.
  - **Hits a hitbox** that closes the chain (don't make m4 a
    branch-only feint-finisher unless that's the explicit concept).
  - **Often has two variants via conditions:** the conventional pattern
    is "m4 has a **downslam** variant (ground-grounded target) and an
    **uppercut** variant (airborne target)", switched on with
    `Req` `AIR` condition or by branching on `Req.AIR.FLIP` (false for
    grounded → downslam, true for airborne → uppercut). When you build
    m4, **default to giving it both**, unless the user explicitly
    asks for a single-variant m4. Implement it with a branch hub: m4's
    default `Line` is a tiny `Req`-driven `BRANCH` that points to
    `downslam` or `uppercut` based on whether the target is airborne.

- **SPECIAL is the wild card.** The right-click `SPECIAL` slot is where
  you go big and creative. Specials can be anything — a long-range
  beam, a counter, a teleport, a transformation, a screen-wide AoE, a
  multi-stage skill with `LOOP` and `BRANCH`. **Treat the special as
  the character's signature — flashier and more memorable than any
  SKILL slot.** Cooldown can also be longer (often 15–30s).

- **SKILLs 1–4 fill out the kit.** Together with SPECIAL these should
  cover at least two ranges (see §8) and include at least one
  defensive/mobility option.

- **AWAKENING patterns** — see §8.1 (Patterns A/B/C). Pick one and
  commit.

### Minimal movesets — not every slot is mandatory

The "full kit" layout (1 CHASE + 4 MELEE + 1 SPECIAL + 4 SKILL + 1
AWAKENING) is the *complete* shape, not a *required* shape. For
**simple movesets** — test movesets, one-off character demos, the
user asks for "just a few moves for X", or anything stripped-down —
**skipping the CHASE and the MELEE slots is fine.** When you don't
provide them, the player keeps the game's default forward-dash and
default M1 string, which is usually fine.

When to skip what:

- **Skip the CHASE slot** when: the character doesn't have a signature
  dash, the user only asked for skills, or the moveset is otherwise
  minimal. The default in-game forward dash takes over.
- **Skip all four MELEE slots** when: the character doesn't have a
  signature M1 string, the user only wants special abilities, or
  you're avoiding the well-documented "agents over-engineer m1s" trap.
  The default in-game M1s take over.
- **Skip the AWAKENING slot** when: the moveset is a quick demo, the
  user didn't ask for an awakening, or — given the unresolved
  AWAKENING-empty-timeline bug in §8.1 — you'd rather not ship a
  broken AWAKENING just to fill the slot.
- **You can also skip individual SKILL slots** (e.g. fill only KEY 1
  and KEY 2 for a focused 2-skill demo). Other keys keep their
  defaults.

For a "full character" build, do include everything — but for testing
the format, exercising a specific block type, or demoing one design,
**less is more.** The validation checklist (§10) doesn't require all
slots present; it requires that the slots you *do* ship are valid.

### Originality conventions (anti-laziness)

- **Don't just slap a built-in `SKILL.MOVE` into every slot.** That's
  the laziest possible custom move — it's *literally* the built-in
  move with a custom keybind. If you use a `SKILL` or `SPECIAL` node
  for a built-in, at minimum **pair it with your own VISUAL/SFX
  layer** so it has identity. Better: borrow only the **animation**
  (via `ANIM_USE` from the move's animation slot — see §7), and
  build your own hitbox/velocity/visuals around it. Best: build it
  from primitives entirely.

- **Use branches for any move with more than one phase.** Single-stage
  moves can live entirely in `DATA.Line`. The moment a move has a
  conditional outcome (hits-vs-misses, grounded-vs-airborne, target-
  hit-vs-no-target, finisher-on-kill), put it in a branch. The
  "branch hub" pattern from §5.23 is the canonical way to keep this
  readable: default `Line` is a thin router that fires the right
  branch; the actual content lives in named branches like `air`,
  `ground`, `onHit`, `onKill`, `noTarget`.

- **Projectiles are not just for ranged moves.** A projectile is a
  moving hitbox/visual carrier. Use it whenever you need a hitbox or
  visual to *travel*. Examples: a body-slam slide (`PROJECTILE` with
  the user's animation riding it), a thrown weapon, a wave of energy,
  a meteor coming down, a ghost that chases the target. Tag visuals/
  SFX with `PROJECTILE TAG` to make them ride along.

- **Visuals are shown on the user by default.** Repeating the
  §5.4 rule because it's the #1 misconception: a `VISUAL` block by
  default renders on the user. If you want an explosion *on the
  target*, the visual has to be inside a branch invoked via
  `BRANCH TARGET`, or attached to a projectile, or use `LAST HIT ≥ 0`.

### Density conventions (making things *feel* powerful)

These are the visual-language tricks that separate a moveset that
"works" from one that looks *cinematic*:

- **High `AMOUNT` in VISUAL = concentrated / exaggerated effect.**
  One Cursed Energy particle is a wisp. `AMOUNT: 25` is a roaring
  pyre. Whenever you want a moment to *land*, push AMOUNT up.

- **Spam visuals between WAIT blocks to amplify power.** For moves
  that are supposed to feel fast, intense, or overwhelmingly powerful,
  do not settle for one VISUAL block. Stack multiple short VISUAL
  blocks with tiny WAIT gaps between them. Examples:
  - **"Character running really fast"** → spam `Wind Streak` and
    "afterimage" visuals (small offsets, fading opacity) every
    0.05–0.1s while the user moves. The eye reads density as speed.
  - **"Powerful charge-up"** → spam `Cursed Energy`, `Energy Sparks`,
    and a slow `Wind Expand` every 0.1s while the user holds a pose.
  - **"Heavy impact"** → fire `Clash`, `Ring`, `Shake Heavy`,
    `Wind Ring` within the same 0.1s window, then `WAIT` and let it
    settle.
  - **"Aura ramping"** → loop a VISUAL block via `LOOP` with a small
    `LOOP BACK` and `LOOP AMOUNT`, so the aura pulses repeatedly.

- **Use Wait→Visual→Wait→Visual cadence, not stacked Visual→Visual.**
  Two `VISUAL` blocks back-to-back fire on the same frame and look
  like one big effect — sometimes desired, but you lose the layered
  build-up. To amplify a moment over time, gap them with `WAIT 0.05`
  to `WAIT 0.15` so each one reads as a discrete *beat*.

- **Don't VISUAL-spam in cooldown moves.** Reserve density for
  important moments (the windup beat, the impact beat, the
  resolution beat). A move with 30 VISUAL blocks across 1 second is
  noise, not power. Three carefully-timed clusters of 3–5 visuals
  is the sweet spot.

### SFX abundance + variety (most movesets are under-soundtracked AND repetitive)

Playtest feedback (two iterations): the *quantity* of SFX is usually
too low (one SFX at the start of a multi-hit chain, nothing after),
and the *variety* is usually too low (the same 4 IDs hammered 30
times across the moveset).

**Quantity rule — fire one `SFX` per hitbox in any multi-hit move.**
For a 5-hit Rapid-Punches-style barrage, that's 5 SFX blocks paired
with the 5 HITBOX blocks. For a charge move, layer a held charging
SFX under the visual charge effects so the windup *sounds* like
power building.

**Variety rule — reuse is fine, but don't overdo it.** Reusing the
same SFX ID across multiple slots is acceptable and often necessary
when the known-good ID set is small. But:
- **Don't stack the same ID 5+ times in a single move** unless the
  design explicitly wants a repeating chant/loop. Vary at least
  every 2–3 hits in a multi-hit chain.
- **Vary `SPEED` slightly** (`0.85`, `0.9`, `1.0`, `1.1`, `1.15`) so
  consecutive reuses don't sound identical. A pitch-shifted reuse
  reads as a different impact even when it's the same source.
- **Pair each ID with a tonally appropriate use** — don't fire the
  "explosion" sound on light M1s and the "punch" sound on the
  domain. Match weight to weight.
- **If you'd like more audio identity than the known-good IDs can
  provide**, scrape per §7.5 OR ask the user for additional IDs.
  Don't just reuse the same 4 IDs 40 times across the kit.

**Charge moves should have a charging SFX.** A long windup with no
audio reads as "the move is broken / nothing happening." Layer a
held SFX (loopable hum, energy buildup, gathering sound) under the
visual charge effects.

### Move naming — names should hint at the action

Playtest feedback: agents sometimes pick mysteriously cool names
that give the player no clue what the move does ("Veilspark",
"Cendrillon's Hush", "Quiet Tide"). Mysterious is fine; *opaque* is
not. **A move name should signal at least one of**:

- The **element / theme** (Fire/Water/Cursed/Shadow/Light)
- The **action verb** (Slash/Pierce/Crash/Burn/Bind)
- The **shape or range** (Beam/Sweep/Ring/Field/Strike)
- The **role** (Counter/Dash/Block/Finisher)

Examples of names that hit the brief:
- ✅ "Pressure Lance" (water + projectile + pierce shape)
- ✅ "Ember Burst" (fire + radial AoE)
- ✅ "Riptide Dash" (water + mobility)
- ✅ "Kamehameha" (well-known canon — instantly read)
- ❌ "Hollow Veil" (poetic but tells you nothing — beam? counter? aura?)
- ❌ "Whisper" (a stab? a teleport? a domain?)

Poetic / character-flavored names are fine *if* they pair with at
least one functional cue. "Sun Breathing: First Form, Water Surface
Slash" works because the subtitle does the heavy lifting.

When the user gives you a character with canonical move names
(Hollow Purple, Rasengan, Getsuga Tensho), **use the canonical
names verbatim**. Don't poeticize over an established name.

### Make projectiles HIT, not just travel

Playtest: projectiles "felt invincible" — they crossed the screen,
hit a target, and showed no acknowledgment that anyone was hit.
The projectile kept going, no flash, no ragdoll, no impact sound on
target. **Every projectile needs a "did it land" feedback layer.**

Pattern for an impacting projectile:

```jsonc
{ "K_NAME": "PROJECTILE",
  "PROJECTILE TAG": "myBeam",
  "POSITION": "0, 1, 2", "ROTATION": "0, 0, 0",
  "SIZE": "4, 4, 4", "SPEED": 90, "TIME": 1.5,
  "DAMAGE": 14, "STUN": 0.5, "STUN ANIM": true,    // STUN ANIM on
  "DEBREE": 6,                                      // debris on impact
  "ATTACK TYPE": "Bullet", "BLOCKABLE": true,
  "CONTINUE": false,                                // dissipate on first hit
  "BRANCH TARGET": "beamHit",                       // target runs hit branch
  "FILTER INTERVAL": 0.2 }
```

And `Branch.beamHit` runs **on the target**:

```jsonc
{ "K_NAME": "VISUAL", "EFFECT": "Glow",
  "COLOR": "255,255,255", "OPACITY": 0.75, "ALT OPACITY": 1, "TIME": 0.25 },
{ "K_NAME": "VISUAL", "EFFECT": "Clash",
  "COLOR": "100,200,230", "ALT OPACITY": 5, "ALT SIZE": 3, "TIME": 0.3 },
{ "K_NAME": "SFX",   "ID": "129465573909487", "VOLUME": 2.5 },
{ "K_NAME": "VELO",  "FORCE": "0, 6, 25", "TIME": 0.3,
  "RAGDOLL": 0.6, "LAST HIT": 1 }                  // target ragdoll-launch
```

The combination of: target-side visual flash, target-side SFX,
target ragdoll, AND `CONTINUE: false` so the projectile *dissipates*
on impact, is what makes the beam read as "it connected." Missing
any one of these → "the projectile feels invincible."

For a piercing projectile (`CONTINUE: true`), the `BRANCH TARGET`
fires on each target it pierces, so each victim still gets the
feedback — just don't dissipate the projectile.

### Finishers (when a multi-stage move ends, end it WITH something)

A multi-stage skill that lands several hits but ends on a faint
trailing whimper feels deflating. **For any move with ≥3 hits or a
charge phase ≥1s, give it a finisher beat** — a final cluster that
clearly says "and *this* is the end of the move":

- A bigger HITBOX (`DEBREE: 6+`, larger SIZE) on the last hit
- A heavier `VELO LAST HIT: 1` ragdoll-launch sending the target
  away
- A `Clash` + `Ring` + `Shake Heavy` VISUAL cluster
- A loud finisher SFX

The Tidekeeper Awakening's `Eye of the Storm` and Goku's
`Meteor Combo` are examples where playtest praised the design — both
had clear final beats. The Spirit Bomb, by contrast, was praised for
its build but flagged for "more visuals/ragdoll on hit targets" —
i.e. its finisher beat needed amplification.

### Counters — design space (not a quota)

`COUNTER` blocks are a separate node type from `HITBOX`/`PROJECTILE` —
not a damage volume but a *parry window*. While active for `TIME`
seconds, any incoming `ATTACK TYPE` you've selected (Melee, Bullet,
Explosion, Swarm, Domain — any subset) is **nullified** and triggers
a branch. Two branches available: `BRANCH` runs on the user (e.g.
trigger a punish move, gain evasion, heal), `BRANCH TARGET` runs on
the attacker (e.g. ragdoll them, apply stun, send them back).

Things you can build with COUNTER:
- **Classic parry** — short window (0.4–0.8s), open to Melee+Bullet,
  on parry ragdoll-launch the attacker with `VELO LAST HIT: 1` and
  `BRANCH TARGET` running impact VFX.
- **Stance / guard** — long window (1.5–2.5s), open to fewer attack
  types, less dramatic punish, but lets the player hold the parry up
  as a defensive stance. Pair with `STATE: "Block"` for visual.
- **Riposte** — short window, on parry trigger an offensive `SKILL`
  node via `BRANCH` (e.g. a quick slash that auto-fires after the
  counter lands).
- **Reflective shell** — open to Bullet only, on parry spawn a
  `PROJECTILE` flying back at the attacker (use `BRANCH` on user side
  + spawn the projectile in the branch's Line).
- **Trap** — open to Domain or Swarm only (uncounterable in base
  game) — a specialty counter that only fires against ultimate-class
  attacks; long cooldown, devastating punish.

`COUNTER` has no built-in animation — you must pair it with an `ANIM`
defensive stance and a telegraph `VISUAL` (commonly the `Block` effect
or a colored `Glow` / shimmer) so the opponent reads it. Without a
visual, the counter is invisible and feels like an unfair "I-frame
trick."

COUNTERs are useful, expressive, and underused. Reach for them when
the move's identity is *reactive* — a defensive tank, a duelist, a
trickster who turns offense back. Don't force one into every moveset —
a pure-aggression character can be coherent without any — but consider
whether the kit has *any* defensive read at all; a kit with zero
ways to handle pressure feels one-dimensional in playtest.

### M4 mini-recipe (downslam + uppercut)

A reference shape for an m4 with conditional variants:

```jsonc
// MELEE slot for m4
{
  "K_NAME": "MELEE", "NAME": "m4",
  "DATA": {
    "Line": [
      // Route to the right variant based on the target's airborne state.
      // Each BRANCH has its own Req; if the Req fails, that BRANCH does
      // not fire and execution continues to the NEXT block in this Line
      // (so the next BRANCH tries its Req in turn). The first BRANCH
      // whose Req passes wins.
      { "K_NAME": "BRANCH", "BRANCH": "uppercut" },
      { "K_NAME": "BRANCH", "BRANCH": "downslam" }
    ],
    "Branch": {
      "uppercut": {
        // SOURCE CONFLICT on AIR FLIP semantics — see §5.20. The wiki
        // says FLIP=true means airborne; empirical observation says the
        // opposite. If your uppercut fires when grounded (wrong), flip
        // these booleans on both branches. Test in-game before shipping.
        "Req": [ { "K_NAME": "AIR", "FLIP": true } ],
        "Line": [
          { "K_NAME": "ANIM", "ANIM_USE": [2, 8], "PREVIEW": [0, 0.6] },
          { "K_NAME": "WAIT", "TIME": 0.2 },
          { "K_NAME": "HITBOX",
            "POSITION": "0, 2, 3", "SIZE": "6, 5, 5",
            "DAMAGE": 6, "STUN": 0.4, "DEBREE": 4,
            "ATTACK TYPE": "Melee", "HIT RAGDOLL": true,
            "PREVIEW": [0.2, 0.4] },
          // CRITICAL: VELO defaults to LAST HIT: -1 (apply to USER).
          //   For m4 to LAUNCH THE TARGET (not the user), LAST HIT must
          //   be a positive value within the hit window (1 is standard).
          //   Without this, your m4 sends YOU flying instead of the target.
          { "K_NAME": "VELO", "FORCE": "0, 30, 5", "TIME": 0.25,
            "RAGDOLL": 0.6, "LAST HIT": 1 }   // ragdoll-launch the TARGET upward
        ]
      },
      "downslam": {
        "Req": [ { "K_NAME": "AIR", "FLIP": false } ],
        "Line": [
          { "K_NAME": "ANIM", "ANIM_USE": [2, 7], "PREVIEW": [0, 0.6] },
          { "K_NAME": "WAIT", "TIME": 0.2 },
          { "K_NAME": "HITBOX",
            "POSITION": "0, -1, 3", "SIZE": "6, 5, 5",
            "DAMAGE": 7, "STUN": 0.4, "DEBREE": 4,
            "ATTACK TYPE": "Melee", "HIT RAGDOLL": true,
            "PREVIEW": [0.2, 0.4] },
          // LAST HIT: 1 → apply to the target, not the user
          { "K_NAME": "VELO", "FORCE": "0, -25, 8", "TIME": 0.25,
            "RAGDOLL": 0.6, "LAST HIT": 1 }   // ragdoll-slam target DOWN
        ]
      }
    },
    "Req": [],
    "Prop": []
  }
}
```

The exact `ANIM_USE` pairs, sizes, and damages are placeholders — copy
from the user's reference moveset when available. The structure is
the point.

### Hit feedback on the TARGET (critical pattern)

A common newbie failure mode: a move "lands a hit" in code (HITBOX
fires, damage is applied) but the target doesn't visibly react —
no flinch, no knockback, no ragdoll, no impact VFX *on them*. The
move feels like nothing connected. From playtest, this is the #1
source of "the move feels weak" complaints.

To make a hit *read*, every offensive HITBOX/PROJECTILE should be
followed by **at least one** of these:

1. **Knockback / launch on the target.** A `VELO` block with `LAST HIT: 1`
   (NOT the default `-1`) that pushes the target back/up/down. Pair
   with `RAGDOLL > 0` for ragdoll. *§5.2 covers the LAST HIT bug in
   detail — read it.*
2. **`HITBOX.STUN ANIM: true`** — flips the default hurt animation on
   the target. The easiest "they got hit" signal — toggle it on for
   any hit that's supposed to read as a clean strike.
3. **`HITBOX.HIT RAGDOLL: true`** — lets your hitbox damage targets
   that are already ragdolled. Combine with a target-side `VELO`
   `RAGDOLL: 0.5+` to *put* them in ragdoll and *keep* hitting them.
4. **`BRANCH TARGET` runs a branch on the target.** Use this to fire
   visuals/animations/states on the hit player. Common branch contents:
   - A `VISUAL` block (visuals invoked from a `BRANCH TARGET` branch
     render on the target — see §5.4) for an impact flash (`Glow`,
     `Clash`, blood `Blood`, `Black Flash`).
   - A `STATE` with `LAST HIT: 1` to apply a debuff to the target.
   - An `ANIM` with `LAST HIT >= 0` to force a reaction animation.
5. **`SFX` on the impact frame.** A meaty hit sound paired with the
   hitbox. If a multi-hit move (volley, Rapid Punches-style chain)
   only plays one SFX, players think only one hit landed — even if
   six did. Fire one SFX per hitbox in multi-hit moves.

**Reference recipe — make a single HITBOX feel like a real hit:**

```jsonc
{ "K_NAME": "HITBOX",
  "POSITION": "0, 0, 4", "SIZE": "6, 6, 6",
  "DAMAGE": 12, "STUN": 0.4, "DEBREE": 4,
  "STUN ANIM": true,            // (1) target plays hurt anim
  "HIT RAGDOLL": true,          // can re-hit ragdolled
  "BRANCH TARGET": "hitFx",     // (4) target runs hitFx branch
  "ATTACK TYPE": "Melee", "BLOCKABLE": true, "PREVIEW": [0.3, 0.4] },
{ "K_NAME": "SFX",   "ID": "129465573909487", "VOLUME": 2.5 },   // (5) impact SFX
{ "K_NAME": "VELO",  "FORCE": "0, 8, 22", "TIME": 0.25,
  "RAGDOLL": 0.5, "LAST HIT": 1 }                                // (1) target knockback
// hitFx branch contains:
//   VISUAL Glow, COLOR "255,255,255", OPACITY 0.75, ALT OPACITY 1, TIME 0.25
//   VISUAL Clash for chunky impact
//   optional: ANIM forcing a stagger on the target via LAST HIT >= 0
```

This is the *baseline* — every offensive HITBOX should hit at least
two of (1)–(5). If a move feels limp in playtest, the first thing to
check is which of these five are missing.

### Hitbox / animation alignment etiquette

- **Hitboxes wait for the swing frame.** Put a `WAIT` of ~30–60% of the
  ANIM's runtime before the `HITBOX`. A `HITBOX` placed immediately
  after `ANIM` (no `WAIT`) hits before the swing visually lands —
  reads as a phantom hit.
- **Visuals lead, hitboxes follow.** Telegraph visuals (wind streaks,
  charge sparks, glow) fire *before* the hitbox so the opponent has
  time to react. Counterplay window > immediate hit.
- **SFX fires on the impact frame.** Tie a `SFX` block to the same
  timeline beat as the `HITBOX`. Sound and damage land together.

### When in doubt, ask the user

If the user's brief is ambiguous about an effect's visual style, a
character's archetype, or which built-ins to borrow from — **ask
one or two clarifying questions before encoding.** A single round-trip
clarification is cheaper than three rounds of "the visual isn't what
I wanted."

---

## 11. TYPICAL FLOWS

### Decode a pasted code

1. Save the code text to `tmp_code.b64`.
2. Run `python jjs_codec.py decode tmp_code.b64 tmp_code.json`.
3. Read `tmp_code.json`, summarize per slot.

### Edit one move in a code

1. Decode as above.
2. Locate the slot by `NAME` or `KEY`.
3. Modify fields **inside `DATA.Line`** (don't rewrite the whole slot
   unless asked).
4. Re-encode: `python jjs_codec.py encode tmp_code.json out.b64`.
5. Verify by decoding `out.b64` and diffing slot names / changed fields.
6. Hand the user the contents of `out.b64`.

### Generate from a description

Run the **two-phase workflow** per §7.6 — Design first, Build second.
Both phases are mandatory for any non-trivial moveset (more than 1–2
slots, named character, anything shippable).

0. **DESIGN PHASE.** Produce the plain-English brief per §7.6.A:
   character identity + per-slot Name/Concept/Range/Telegraph/Active/
   Hit feedback/Resolution/Counterplay + awakening identity + mesh
   wishlist. Either Tactic A (in-context) or Tactic B (delegate to a
   sub-agent if your harness has one). **Lock the brief before any
   JSON is written.** No JSON, ANIM_USE pairs, or asset IDs in the
   brief.
1. **BUILD PHASE — slot layout.** Pick the slot layout: usually 1 CHASE
   + 4 MELEE (m1..m4) + 1 SPECIAL + 4 SKILL (KEY 1..4) + optionally 4
   awakened-SKILL + 1 AWAKENING. Per §10.7 "Minimal movesets" you may
   ship fewer slots; per §8.1 the AWAKENING slot may be skipped
   pending the unresolved activation bug.
2. **BUILD PHASE — translate brief to blocks.** For each slot, build
   the `DATA.Line` from primitives in §5, implementing exactly what
   the brief promised, following §8 design rules and §10.7 etiquette.
   Don't drift from the brief — if the format can't express something
   the brief required, **stop and flag it to the user** before
   continuing.
3. **BUILD PHASE — pick concrete asset values.** Reuse `ANIM_USE` pairs
   and `SFX ID`s from a decoded moveset whenever possible. If the user
   has a previous validated import code, decode it and pull values
   from there; otherwise use the known-valid `[group, index]` pairs
   listed in §7 as a fallback. For meshes, ask the user per §7.5.
4. **Run the §10.5 self-review pass.** Walk velocity, timeline,
   hitbox, branch, and cross-slot reviews. Fix anything found; if you
   can't, add a "Known issues" note to the reply.
5. Encode → round-trip verify → hand back the base64 code → **always
   include the §7.5 Assets Used table** if any SFX/Texture/Mesh ID
   appears anywhere.

---

## 12. REFERENCES

The primary source for everything in this skill is the JJS Fandom
wiki. If the agent needs information not covered here (a brand-new
block type from a fresh update, an obscure character move, a precise
timing not in §5.22), fall back to the wiki directly — these pages
are the canonical source:

- **Skill Builder (full reference):**
  <https://jujutsu-shenanigans.fandom.com/wiki/Build_Mode/Skill_Builder>

  **Important:** every Skill Builder tab — Timeline, Conditions,
  Properties, Variants, Timings (W.I.P.), Known Tricks — lives on
  this **single page** inside a `<tabber>` block. The sub-page URLs
  like `Build_Mode/Skill_Builder/Conditions` **do not exist** (they
  404). To fetch a specific tab, fetch the one page and grep the
  wikitext for the tab heading (e.g. `|-|Conditions=`).
- **Characters index (source for the JJS Moves Catalog skill):**
  <https://jujutsu-shenanigans.fandom.com/wiki/Characters>
- **Emotes list (source for the JJS Emotes Catalog skill):**
  <https://jujutsu-shenanigans.fandom.com/wiki/Emotes>
- **Roblox asset-ID lookup sites** (§7.5):
  - <https://robloxsong.com/search?q={query}> (audio — replace `{query}` with your search term)
  - <https://robloximageid.com/search?q={query}> (decals / images)
  - <https://robloxden.com/item-codes> (models / meshes)

Fandom blocks naïve `WebFetch` user-agents (it returns 403). If a
direct fetch fails, the agent can use the Fandom MediaWiki API to
pull wikitext instead:

```
https://jujutsu-shenanigans.fandom.com/api.php?action=parse&page=<PageName>&format=json&prop=wikitext
```

(returns JSON with the wikitext under `parse.wikitext["*"]`).
