# Doctor Who and the Mines of Terror — Item Reference
## Micropower, 1986 (Commodore 64)

---

## How to Use This Document

### Coordinate System

The game world uses a two-level coordinate system:

| What | Description | Range |
|------|-------------|-------|
| **Room col** | Which screen column you're in | 0–15 |
| **Room row** | Which screen row you're in | 0–6 (0=surface, 6=deepest) |
| **Sub-tile X** | Fine horizontal position within the room | $00–$FF (0–255) |
| **Sub-tile Y** | Fine vertical position within the room | $00–$FF (0–255) |

A full map position is: **(room col, room row) + sub-tile offset within that room.**

### Player Position — Memory Addresses to Monitor

Use a memory monitor in your emulator to watch these addresses while playing:

| Address | What it shows | Updates when |
|---------|--------------|--------------|
| `$1BC0` | Player's current room **column** | Crossing a room boundary |
| `$1C60` | Player's current room **row** | Crossing a room boundary |
| `$CB` (ZP) | Player's sub-tile **X** (fine horizontal) | Every frame during movement |
| `$CC` (ZP) | Player's sub-tile **Y** (fine vertical) | Every frame during movement |
| `$1D80` | **Held item** type ID (0 = nothing) | On pickup/drop |

### Item Position Addresses

Every entity (including items) has its position stored in four tables, indexed by the item's **type ID**:

| Table | Base address | Formula | Stores |
|-------|-------------|---------|--------|
| `f1BC0` | `$1BC0` | `$1BC0 + type_id` | Room column |
| `f1C60` | `$1C60` | `$1C60 + type_id` | Room row |
| `f1B70` | `$1B70` | `$1B70 + type_id` | Sub-tile X |
| `f1C10` | `$1C10` | `$1C10 + type_id` | Sub-tile Y |

**Example:** AIR MASK has type ID `$3A`. Its room column is at `$1BC0 + $3A = $1BFA`.

---

## Items

---

### AIR MASK
**Type ID:** `$3A`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BFA` | `$06` |
| Room row | `$1C9A` | `$00` |
| Sub-tile X | `$1BAA` | `$10` |
| Sub-tile Y | `$1C4A` | `$60` |

**Found at:** Room **(6, 0)**, sub-tile ($10, $60) — upper mine, surface level.

**How it works:** Passive — must be **held** (in `$1D80`). Each game tick in a toxic zone, the game checks if `$1D80 == $3A`. If yes: countdown reset to 50. If no: countdown decrements. At 0: death.

**Where it's needed — Toxic Air Rooms:**

These are the only rooms in the game containing tile type `$3F` (toxic air tile), which triggers the countdown:

| Room | Address of tile in map RAM | Notes |
|------|---------------------------|-------|
| **(2, 2)** | `$F1A1` | One toxic tile |
| **(4, 2)** | `$F1C6` | One toxic tile |
| **(3, 5)** | `$E4B1`, `$E5B7` | Two toxic tiles |

> **Important:** Once triggered, the countdown continues even in non-toxic rooms (row ≥ 2) until you step on a "fresh air" tile (type `$39` or `$04`). You can't just leave the toxic room — you must reach a safe tile or carry the mask.

---

### PASS CARD
**Type ID:** `$37`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BF7` | `$0A` |
| Room row | `$1C97` | `$04` |
| Sub-tile X | `$1BA7` | `$A8` |
| Sub-tile Y | `$1C47` | `$D8` |

**Found at:** Room **(10, 4)**, sub-tile ($A8, $D8).

**Used at:** The security door. The game checks proximity to the door entity automatically when you hold the PASS CARD and press use — no fixed coordinate table, the door location is entity-driven.

**Effect:** Unlocks the security gate.

---

### SPANNER
**Type ID:** `$38`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BF8` | `$06` |
| Room row | `$1C98` | `$03` |
| Sub-tile X | `$1BA8` | `$20` |
| Sub-tile Y | `$1C48` | `$08` |

**Found at:** Room **(6, 3)**, sub-tile ($20, $08).

**Used at:** Room **(12, 5)**, sub-tile ($D0, $87) — this is the **only valid use location**.

**Use location coordinates** (from handler table at `$BB94`/`$BBA4`/`$BB9C`/`$BBAC`):

| | Value |
|---|-------|
| Room col | `$0C` (12) |
| Room row | `$05` (5) |
| Sub-tile X | `$D0` |
| Sub-tile Y | `$87` |

**Effect:** Repairs machinery.

> **⚠ Warning:** There are 7 other proximity spots near the same area that look similar but trigger **instant death** (sequence `$E1`). The only safe spot is exactly the one above. The dangerous spots are in rooms (7–14, row 5–6).
>
> Rooms (13, 6) and (14, 6) are particularly dangerous as they overlap with ACTIVATOR use locations — using the SPANNER there kills you.

---

### PICK AXE
**Type ID:** `$34`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BF4` | `$01` |
| Room row | `$1C94` | `$00` |
| Sub-tile X | `$1BA4` | `$60` |
| Sub-tile Y | `$1C44` | `$60` |

**Found at:** Room **(1, 0)**, sub-tile ($60, $60).

**Used at:** 3 specific mine walls. Coordinate tables at `$BC1C` (room col), `$BC22` (room row), `$BC1F` (sub-tile X), `$BC25` (sub-tile Y):

| Wall | Room | Sub-tile X | Sub-tile Y | Animation | Result |
|------|------|------------|------------|-----------|--------|
| 1 | **(5, 0)** | `$C0` | `$74` | `$0E` | Breaks wall |
| 2 | **(5, 0)** | `$E0` | `$74` | `$09` | Breaks wall |
| 3 | **(9, 1)** | `$00` | `$54` | `$0A` | **⚠ Cave-in — death** |

> Wall 3 deliberately kills the player. Don't use the Pick Axe at room (9, 1).

---

### The Bomb System — DETONATOR, EXPLOSIVES, and components `$48`/`$4C`

These four items form a single bomb system. The game tracks all four via the `f1CB0` table (base `$1CB0`, indexed by type ID). The bomb component list is hard-coded at `$C081`: `[$3D, $3C, $48, $4C]`.

**How the bomb system works:**

1. Using any component sets its `f1CB0` entry non-zero, starting a **136-tick countdown** (`$88` ticks)
2. Each frame a main loop at `$C0C4` checks all four components — once triggered, a trajectory/animation begins
3. When the countdown reaches 0: the **player's current position is locked as the explosion site** (code at `$C194` copies player position tables into the entity)
4. The explosion entity is then proximity-checked to determine the outcome (see EXPLOSIVES below)

**Bomb activation state** (monitor while playing):

| Item | Type ID | f1CB0 address | Value when armed |
|------|---------|--------------|-----------------|
| EXPLOSIVES | `$3D` | `$1CED` | `$FF` |
| DETONATOR | `$3C` | `$1CEC` | `$88` (136-tick countdown) |
| detonator variant | `$48` | `$1CF8` | `$88` |
| detonator variant | `$4C` | `$1CFC` | `$88` |

---

### DETONATOR
**Type ID:** `$3C`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BFC` | `$0A` |
| Room row | `$1C9C` | `$05` |
| Sub-tile X | `$1BAC` | `$1E` |
| Sub-tile Y | `$1C4C` | `$08` |

**Found at:** Room **(10, 5)**, sub-tile ($1E, $08) — same room as `$48` and `$4C`.

**Used:** Press use to arm — sets `$1CEC = $88`, starting the 136-tick countdown. Part of the 4-component bomb system. Must be combined with EXPLOSIVES for effect.

---

### EXPLOSIVES
**Type ID:** `$3D`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BFD` | `$07` |
| Room row | `$1C9D` | `$00` |
| Sub-tile X | `$1BAD` | `$80` |
| Sub-tile Y | `$1C4D` | `$A8` |

**Found at:** Room **(7, 0)**, sub-tile ($80, $A8).

**Used at:** Room **(14, 0)**, sub-tile ($60, $68) — the **heatonite machine** location.

**Effect:** When the explosion is placed at the heatonite machine, sets flag `a51 = 1` ("heatonite production halted") — this is a bonus completion condition that doubles the score. The explosion site is determined by **where the player is standing when the 136-tick countdown fires**, so you must be at room (14, 0) when the timer expires.

> **⚠ Warning:** If the explosion fires while the player is near room **(11, 5)** (sub-tile $E0, $88) — the ACTIVATOR pickup room — it causes **death**. Plan your route carefully after arming the bomb.

---

### ACTIVATOR
**Type ID:** `$3F`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BFF` | `$0B` |
| Room row | `$1C9F` | `$05` |
| Sub-tile X | `$1BAF` | `$20` |
| Sub-tile Y | `$1C4F` | `$88` |

**Found at:** Room **(11, 5)**, sub-tile ($20, $88).

**Used at:** 3 spots, all in room row 6. Coordinate tables at `$BCDA` (room col) and `$BCDD` (sub-tile X):

| Spot | Room col | Room row | Sub-tile X | Sub-tile Y |
|------|----------|----------|------------|------------|
| 1 | `$0D` (13) | `$06` (6) | `$08` | `$48` |
| 2 | `$0D` (13) | `$06` (6) | `$88` | `$48` |
| 3 | `$0E` (14) | `$06` (6) | `$08` | `$48` |

**Effect:** Sets `speed_mod = 2`. Likely opens access to room (14, 6) where the CAPSULE is found.

> Room (14, 6) is where the **CAPSULE (Tiru Plans)** is located — you probably need to use the ACTIVATOR here before you can collect it.

---

## Detonator Variants (type `$48` and `$4C`)

Both found in room **(10, 5)** alongside the main DETONATOR. No display name. Part of the 4-component bomb system. Handlers at `$BCB9` and `$BCC4` set their `f1CB0` entry to `$88` (136-tick countdown) when activated. Dispatched via `$BB6B`/`$BB6F` in the item-use handler at `$BB5D`.

| Type ID | Address (room col) | Address (room row) | Sub-tile X addr | Sub-tile Y addr | Found at | f1CB0 addr | Armed value |
|---------|-------------------|-------------------|-----------------|-----------------|----------|-----------|-------------|
| `$48` | `$1C08` = `$0A` | `$1CA8` = `$05` | `$1BB8` = `$00` | `$1C58` = `$08` | (10, 5) sub ($00, $08) | `$1CF8` | `$88` |
| `$4C` | `$1C0C` = `$0A` | `$1CAC` = `$05` | `$1BBC` = `$3C` | `$1C5C` = `$08` | (10, 5) sub ($3C, $08) | `$1CFC` | `$88` |

---

## Creatures

These are non-item entities with AI behaviour. They have positions in the entity tables but no name in the game's text table.

### MADRAG (adult) — type `$44` and `$45`

Two adult Madrags placed in the deepest mine level. When they get close to the player they trigger **"SUFFERING A MADRAG BITE"** death (speed_mod = `$14`).

| Type ID | Room col addr | Room row addr | Sub-tile X addr | Sub-tile Y addr | Found at |
|---------|--------------|--------------|-----------------|-----------------|----------|
| `$44` | `$1C04` = `$09` | `$1CA4` = `$06` | `$1BB4` = `$08` | `$1C54` = `$A8` | **(9, 6)** sub ($08, $A8) |
| `$45` | `$1C05` = `$06` | `$1CA5` = `$06` | `$1BB5` = `$60` | `$1C55` = `$48` | **(6, 6)** sub ($60, $48) |

---

### BABY MADRAG — type `$46` and `$47`

Two baby Madrags, both in the same surface room. When they reach the player they trigger **"BEING ATTACKED BY A BABY MADRAG"** death (speed_mod = `$16`). The game processes both with a loop at `$B8E4`–`$B931`, iterating entity types `$46` and `$47`.

| Type ID | Room col addr | Room row addr | Sub-tile X addr | Sub-tile Y addr | Found at |
|---------|--------------|--------------|-----------------|-----------------|----------|
| `$46` | `$1C06` = `$04` | `$1CA6` = `$00` | `$1BB6` = `$08` | `$1C56` = `$60` | **(4, 0)** sub ($08, $60) |
| `$47` | `$1C07` = `$04` | `$1CA7` = `$00` | `$1BB7` = `$28` | `$1C57` = `$60` | **(4, 0)** sub ($28, $60) |

---

### Entity `$49` — Unimplemented Moving Entity

Positioned at room **(5, 5)** sub ($88, $C8) — same level as SPLINX ($2F). No name in text table. No item-use handler. No movement code. **Dead code / unimplemented feature.**

| | Address | Value |
|---|---------|-------|
| Room col | `$1C09` | `$05` |
| Room row | `$1CA9` | `$05` |
| Sub-tile X | `$1BB9` | `$88` |
| Sub-tile Y | `$1C59` | `$C8` |

**Code reference:** After the SPLINX patrol loop ($BA3C–$BA84, entity types $2C–$2F), there is an isolated check at `$BA8D` that sets X=$49 and calls `sA3FB` to test whether entity $49 has arrived at hardcoded position **(col=$02, row=$05)**, sub ($90, $88):

```asm
$BA86 LDA $1CCA        ; skip if already triggered
$BA8B BEQ skip
$BA8D LDX #$49
$BA8F–$BA9D            ; load target pos (col=2, row=5, sub-X=$90, sub-Y=$88)
$BA9F JSR sA3FB        ; proximity check: entity $49 vs target
$BAA2–$BAAE            ; bail out if $DE≠0 (row mismatch) or $DA≠0 (col mismatch) or $D9≥$50 (too far)
$BAB0–$BABD            ; on success: set $1CCA=$1CCB=$03, $2FA1=$2F3A=$0C
$BAC0 LDY #$09
$BAC2 JSR sC7C7        ; play sound #$09
$BAC5 JMP jAE81        ; call 3-page sprite/screen copy routine
```

**Why it never fires:** Entity $49 starts at col=$05. The proximity check requires col=$02. There is no code anywhere that moves entity $49 (it is not in the entity movement loop). The check at `$DA ≠ 0` always fails ($05 − $02 = $03 ≠ 0).

**Intended behaviour (reconstructed):** Entity $49 was apparently meant to be a moving object (possibly a mine vehicle, lift, or escape craft) travelling from room (5,5) leftward to room (2,5). On arrival it would play a sound and trigger `jAE81` (a 3-page screen/sprite data copy — probably a cutscene or vehicle sprite display). The movement system was never implemented. The entity is effectively invisible and inert in the shipped game.

---

## Story / Score Items

These multiply the end-of-game completion score bonus when collected or delivered.

### CRYSTAL
**Type ID:** `$36`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BF6` | `$02` |
| Room row | `$1C96` | `$04` |
| Sub-tile X | `$1BA6` | `$48` |
| Sub-tile Y | `$1C46` | `$D0` |

**Found at:** Room **(2, 4)**, sub-tile ($48, $D0).
**Effect:** Collected state checked at `$1B20 + $36 = $1B56` — value `$04` means delivered/complete.

---

### CAPSULE ("Tiru Plans")
**Type ID:** `$3E`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BFE` | `$0E` |
| Room row | `$1C9E` | `$06` |
| Sub-tile X | `$1BAE` | `$E0` |
| Sub-tile Y | `$1C4E` | `$48` |

**Found at:** Room **(14, 6)**, sub-tile ($E0, $48) — same room as ACTIVATOR use spots.
**Effect:** Collected state at `$1B20 + $3E = $1B5E` — value `$04` means delivered/complete.

> The ACTIVATOR must be used at spots in rooms (13–14, row 6) before this item can be reached.

---

### SPLINX
**Type ID:** `$2F`

| | Address | Value |
|---|---------|-------|
| Room col | `$1BEF` | `$06` |
| Room row | `$1C8F` | `$05` |
| Sub-tile X | `$1B9F` | `$70` |
| Sub-tile Y | `$1C3F` | `$08` |

**Found at:** Room **(6, 5)**, sub-tile ($70, $08).
**Effect:** One of three key story items checked in `completion_score_init` (`$9163`). Delivering or interacting with SPLINX triggers scroll sequence "SPLINX". Collecting all three story items (CRYSTAL, Tiru Plans, SPLINX) maximises the completion score bonus.

---

## The Heatonite System

Halting heatonite production is a **bonus ending condition** that doubles the completion score multiplier. It requires using EXPLOSIVES at the heatonite machine location.

**Heatonite machine:** Room **(14, 0)**, sub-tile ($60, $68) — this is where you must be standing when the bomb countdown expires.

**Flag:** `a51` (ZP `$51`) — set to `1` when production is halted. Checked at `$918B` during end-of-game score calculation.

CHEMICALS (`$42`), GEM (`$43`), and entity `$4A` (all at room **2, 3**) appear to be the raw materials being fed into the heatonite machine. They are checked in the heatonite condition at `$9199` (completion score) alongside `a51`.

| Entity | Type ID | Found at | f1CB0 address | Notes |
|--------|---------|----------|--------------|-------|
| CHEMICALS | `$42` | **(2, 3)** sub ($08, $90) | `$1CF2` | Heatonite ingredient |
| GEM | `$43` | **(2, 3)** sub ($28, $90) | `$1CF3` | Heatonite ingredient |
| (unknown) | `$4A` | **(2, 3)** sub ($48, $90) | `$1CFA` | Heatonite ingredient |

---

## Collectible Items (score only)

Walked over for score points. No use handler.

| Item | Type ID | Room col addr | Room row addr | Found at | Sub-tile (X, Y) |
|------|---------|--------------|--------------|----------|-----------------|
| PLATFORM | `$39` | `$1BF9` = `$02` | `$1C99` = `$06` | **(2, 6)** | ($80, $48) |
| BOX | `$3B` | `$1BFB` = `$05` | `$1C9B` = `$02` | **(5, 2)** | ($40, $88) |
| CIRCUIT | `$40` | `$1C00` = `$01` | `$1CA0` = `$03` | **(1, 3)** | ($C8, $90) |
| EGG | `$41` | `$1C01` = `$01` | `$1CA1` = `$03` | **(1, 3)** | ($E8, $90) |

---

## Quick Reference — All Item Memory Addresses

| Item | Type ID | Room col | Room row | Sub-tile X | Sub-tile Y |
|------|---------|----------|----------|------------|------------|
| PICK AXE | `$34` | `$1BF4` | `$1C94` | `$1BA4` | `$1C44` |
| CRYSTAL | `$36` | `$1BF6` | `$1C96` | `$1BA6` | `$1C46` |
| PASS CARD | `$37` | `$1BF7` | `$1C97` | `$1BA7` | `$1C47` |
| SPANNER | `$38` | `$1BF8` | `$1C98` | `$1BA8` | `$1C48` |
| PLATFORM | `$39` | `$1BF9` | `$1C99` | `$1BA9` | `$1C49` |
| AIR MASK | `$3A` | `$1BFA` | `$1C9A` | `$1BAA` | `$1C4A` |
| BOX | `$3B` | `$1BFB` | `$1C9B` | `$1BAB` | `$1C4B` |
| DETONATOR | `$3C` | `$1BFC` | `$1C9C` | `$1BAC` | `$1C4C` |
| EXPLOSIVES | `$3D` | `$1BFD` | `$1C9D` | `$1BAD` | `$1C4D` |
| CAPSULE | `$3E` | `$1BFE` | `$1C9E` | `$1BAE` | `$1C4E` |
| ACTIVATOR | `$3F` | `$1BFF` | `$1C9F` | `$1BAF` | `$1C4F` |
| CIRCUIT | `$40` | `$1C00` | `$1CA0` | `$1BB0` | `$1C50` |
| EGG | `$41` | `$1C01` | `$1CA1` | `$1BB1` | `$1C51` |
| CHEMICALS | `$42` | `$1C02` | `$1CA2` | `$1BB2` | `$1C52` |
| GEM | `$43` | `$1C03` | `$1CA3` | `$1BB3` | `$1C53` |
| detonator variant | `$48` | `$1C08` | `$1CA8` | `$1BB8` | `$1C58` |
| detonator variant | `$4C` | `$1C0C` | `$1CAC` | `$1BBC` | `$1C5C` |
| SPLINX | `$2F` | `$1BEF` | `$1C8F` | `$1B9F` | `$1C3F` |
| MADRAG (adult) #1 | `$44` | `$1C04` | `$1CA4` | `$1BB4` | `$1C54` |
| MADRAG (adult) #2 | `$45` | `$1C05` | `$1CA5` | `$1BB5` | `$1C55` |
| BABY MADRAG #1 | `$46` | `$1C06` | `$1CA6` | `$1BB6` | `$1C56` |
| BABY MADRAG #2 | `$47` | `$1C07` | `$1CA7` | `$1BB7` | `$1C57` |
| entity $49 (unimplemented) | `$49` | `$1C09` | `$1CA9` | `$1BB9` | `$1C59` |

---

## Player Position Addresses (monitor while playing)

| What | Address | Notes |
|------|---------|-------|
| Room column | `$1BC0` | Changes on room transition |
| Room row | `$1C60` | Changes on room transition |
| Sub-tile X | `$CB` (ZP) | Changes every frame |
| Sub-tile Y | `$CC` (ZP) | Changes every frame |
| Held item | `$1D80` | Type ID of item in hand (0 = none) |

---

## Death Cause System

All deaths, endings, and game events are controlled through ZP `$E8` (`speed_mod`). The death display handler at `$9105` reads `speed_mod`, shifts right by 1 (→ Y), then looks up the scroll sequence index in the table `f90EF` (`$90EF`).

**Negative values** (`$80`, `$C0`, `$FF`) are keyboard scan states set at `$8FAA`/`$8FBA` and cause the handler to branch away — they are the normal running game state, not deaths.

### speed_mod values — death/event causes

| `speed_mod` | Y (÷2) | Scroll seq | Message shown | Trigger |
|------------|--------|------------|---------------|---------|
| `$02` | 1 | Seq `$20` | "...IN AN ESCAPE POD" | ACTIVATOR used at room (13–14, row 6) — set at `$BD0F` |
| `$04` | 2 | Seq `$1F` | "YOU HAVE RETURNED TO GALLIFREY IN THE TARDIS" | Tile `$B2` with sub-tile Y & `$1F` == 0 — set at `$BD2F` |
| `$06` | 3 | Seq 4 | "THE EXPLOSION" | Bomb explodes near player (room 11, 5 danger zone) — set at `$C1F6` |
| `$08` | 4 | Seq `$11` | "GAME OVER" *(wrong — should be Seq 5 "EXPOSURE TO RADIATION", see bug note below)* | Post-explosion radiation: after heatonite machine destroyed, ZP `$51` increments every 128 ticks. If player depth < radiation counter → death. Set at `$C0C0`. |
| `$0A` | 5 | Seq 6 | "A SHOCK FROM A CONTROLLER" | Controller entity proximity — set at `$CE58` |
| `$0C` | 6 | Seq 7 | "A FALL" | Room scroll handler: falling too fast — set at `$CD89` |
| `$0E` | 7 | Seq 8 | "FALLING ON A STALAGMITE" | Stalagmite collision; also used as TARDIS/escape-pod detection flag — set at `$CA65` |
| `$10` | 8 | Seq 9 | "SUFFOCATING" | Air countdown `aF1` reaches 0 (toxic tile without AIR MASK) — set at `$CAC0` |
| `$12` | 9 | Seq `$0A` | "FORCED REGENERATION" | Player presses **R key** — set at `$904F` |
| `$14` | 10 | Seq `$0B` | "SUFFERING A MADRAG BITE" | Adult MADRAG entity proximity — set at `$B971` |
| `$16` | 11 | Seq `$0C` | "BEING ATTACKED BY A BABY MADRAG" | Baby MADRAG entities `$46`/`$47` proximity — set at `$B904` |

### Radiation death — "EXPOSURE TO RADIATION" bug

speed_mod=`$08` is set at `$C0C0` (inside `$C09A`–`$C0C3`). The radiation counter ZP `$51` is only non-zero after the heatonite machine has been destroyed. Every 128 ticks it increments. The player's mine depth is computed as `(room_row × 8 + sub_tile_Y >> 3)` — deeper = safer. If `depth < $51`, radiation death fires.

The dispatch table `f90EF` at `$90EF` maps `speed_mod/2` (Y) to a flat scroll-pointer index:

```
Y:  0    1    2    3    4    5    6    7    8    9    10   11
    $00  $58  $55  $0c  $2a  $10  $12  $14  $16  $18  $1a  $1c
```

Y=4 (speed_mod=`$08`) → `$2A` → Seq `$11` → shows blank line / "GAME OVER".
The correct value should be `$0E` → Seq 5 → "EXPOSURE TO RADIATION". This is a developer bug — the wrong pointer was written to `f90EF[4]`.

### Lives system

`a1E` (abs `$001E`) tracks remaining lives.

- **Normal death** (speed_mod `$06`–`$16`, Y ≥ 3): decrements lives. If lives remain → "THE DOCTOR REGENERATES AFTER" + cause text, restart level. If no lives → "GAME OVER AFTER" + cause text, game-over screen.
- **Forced regen** (speed_mod `$12`, Y=9): decrements lives without regeneration animation. If no lives → game-over screen.
- **Endings** (speed_mod `$02`/`$04`, Y=1/2): don't decrement lives; go straight to ending scroll sequence.

---

## Scroll Sequences Reference

The scroll engine reads text pointer tables at `$95CB` (lo) / `$9626` (hi). Each entry is a flat byte index into those tables — not a sequence number. `s_scroll_init(X)` jumps the scroll engine to pointer table entry X.

The 33 sequences (0–`$20`) and their starting pointer-table indices:

| Seq # | Ptr idx (X) | Text | Used when |
|-------|-------------|------|-----------|
| 0 | `$00` | "THE DOCTOR REGENERATES AFTER" | Lead-in for every non-final death |
| 1 | `$02` | "GAME OVER AFTER" (2 parts) | Lead-in for final death (no lives left) |
| 2 | `$05` | "YOUR FINAL SCORE IS [score]" (2 parts) | Score display |
| 3 | `$08` | *(3-part message — start / middle / end of score reveal)* | Score display components |
| 4 | `$0C` | "THE EXPLOSION" | Death — bomb explosion |
| 5 | `$0E` | "EXPOSURE TO RADIATION" | **Dead code — never shown.** Bug in `f90EF[4]`: the radiation death (speed_mod=`$08`, Y=4) dispatches to ptr index `$2A` → Seq `$11` (blank/GAME OVER) instead of `$0E` → this seq. The radiation death mechanic is fully implemented but shows the wrong message. |
| 6 | `$10` | "A SHOCK FROM A CONTROLLER" | Death — controller entity |
| 7 | `$12` | "A FALL" | Death — falling |
| 8 | `$14` | "FALLING ON A STALAGMITE" | Death — stalagmite |
| 9 | `$16` | "SUFFOCATING" | Death — toxic air |
| `$0A` | `$18` | "FORCED REGENERATION" | Death — R key / forced |
| `$0B` | `$1A` | "SUFFERING A MADRAG BITE" | Death — adult Madrag |
| `$0C` | `$1C` | "BEING ATTACKED BY A BABY MADRAG" | Death — baby Madrag |
| `$0D` | `$1E` | "THE CRYSTAL" | Story item event |
| `$0E` | `$20` | "THE TIRU PLANS" | Story item event |
| `$0F` | `$22` | "SPLINX" | Story item event |
| `$10` | `$24` | *(5× blank lines — `$97D5`)* | Transition pause (level restart) |
| `$11` | `$2A` | *(single blank line)* | Transition pause (game over) |
| `$12` | `$2C` | *(single blank line)* | Transition |
| `$13` | `$2E` | *(page `$98` text — unidentified)* | Transition |
| `$14` | `$30` | "AND HAVE SUCCESSFULLY HALTED THE PRODUCTION OF HEATONITE" (2 parts) | Ending — heatonite halted |
| `$15` | `$33` | "BUT HAVE NOT HALTED THE PRODUCTION OF HEATONITE" (2 parts) | Ending — heatonite not halted |
| `$16` | `$36` | "THE TIME LORDS ARE" | Score verdict lead-in |
| `$17` | `$38` | "WITH YOUR PERFORMANCE" | Score verdict lead-in |
| `$18` | `$3A` | "VERY DISPLEASED" (2 parts) | Score < 4,100 |
| `$19` | `$3D` | "DISPLEASED" | Score 4,100–19,999 |
| `$1A` | `$3F` | "PLEASED" | Score 20,000–39,999 |
| `$1B` | `$41` | "VERY PLEASED" (2 parts) | Score ≥ 40,000 |
| `$1C` | `$44` | "PRESS F1 TO SAVE ON CASSETTE" (multi-part) | Save prompt — cassette |
| `$1D` | `$4A` | "PRESS F3 TO SAVE ON DISC" (multi-part) | Save prompt — disc |
| `$1E` | `$50` | "PRESS F5 TO RETURN TO GAME" (multi-part) | Save prompt — return |
| `$1F` | `$55` | "YOU HAVE RETURNED TO GALLIFREY IN THE TARDIS" | Ending — TARDIS |
| `$20` | `$58` | "...IN AN ESCAPE POD" | Ending — escape pod |

Text is stored in 6502 ROM pages `$96`–`$98`. Charset: `$DC`=A … `$F5`=Z, `$94`=space, `$00`=end-of-string.
