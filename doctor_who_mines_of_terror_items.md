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
| (unknown) | `$48` | `$1CF8` | `$88` |
| (unknown) | `$4C` | `$1CFC` | `$88` |

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

## Unknown Items (type `$48` and `$4C`)

Both found in room **(10, 5)** alongside the DETONATOR. Names not found in the entity name table. Part of the 4-component bomb system (see DETONATOR/EXPLOSIVES above).

| Type ID | Address (room col) | Address (room row) | Sub-tile X addr | Sub-tile Y addr | Found at | f1CB0 addr | Armed value |
|---------|-------------------|-------------------|-----------------|-----------------|----------|-----------|-------------|
| `$48` | `$1C08` = `$0A` | `$1CA8` = `$05` | `$1BB8` = `$00` | `$1C58` = `$08` | (10, 5) sub ($00, $08) | `$1CF8` | `$88` |
| `$4C` | `$1C0C` = `$0A` | `$1CAC` = `$05` | `$1BBC` = `$3C` | `$1C5C` = `$08` | (10, 5) sub ($3C, $08) | `$1CFC` | `$88` |

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
| bomb component | `$48` | `$1C08` | `$1CA8` | `$1BB8` | `$1C58` |
| bomb component | `$4C` | `$1C0C` | `$1CAC` | `$1BBC` | `$1C5C` |

---

## Player Position Addresses (monitor while playing)

| What | Address | Notes |
|------|---------|-------|
| Room column | `$1BC0` | Changes on room transition |
| Room row | `$1C60` | Changes on room transition |
| Sub-tile X | `$CB` (ZP) | Changes every frame |
| Sub-tile Y | `$CC` (ZP) | Changes every frame |
| Held item | `$1D80` | Type ID of item in hand (0 = none) |
