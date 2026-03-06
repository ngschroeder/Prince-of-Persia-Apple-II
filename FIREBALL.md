# Fireball Feature Plan for Prince of Persia Apple II

Press **H** to throw a fireball that travels horizontally and damages the opponent on contact.

---

## Context: How This Codebase Works

All source files are 6502 assembly in `01 POP Source/Source/`. The game uses two custom build tools (`snap` assembler, `crackle` disk imager) — 64-bit versions have already been built and placed in `Build/osx32/`. Build with `make clean all` from the project root. Output is `PrinceOfPersia_3.5.hdv` (load in Virtual II with an Apple IIe + UniDisk 3.5 config).

### Architecture at a Glance

| System | File(s) | Address | Purpose |
|--------|---------|---------|---------|
| Main loop | TOPCTRL.S | $2000 | `NextFrame` calls DoKid, DoShad, checkstrike, FrameAdv |
| Player control | CTRL.S | $3a00 | `PlayerCtrl`, `FightCtrl`, `DoStrike` — input-to-action |
| Control utils | CTRLSUBS.S | $d000 | `jumpseq`, `GETOPDIST`, `SETUPCHAR`, frame lookup |
| Keyboard input | SPECIALK.S | $d900 | `KEYS`, `LegitKeys`, `DevelKeys`, key constants |
| Animation sequences | SEQTABLE.S | $2800/$3000 | Bytecode sequences + frame definitions |
| Sequence constants | SEQDATA.S | — | Named entry points (1-114) and instruction opcodes |
| Collision | COLL.S | $4500 | `ANIMCHAR`, `CHECKCOLL`, `CHECKBARR` |
| Combat | MISC.S | $f900 | `STABCHAR`, `DECSTR` — damage dealing |
| Mobile objects | MOVER.S | $ee00 | `ANIMMOBS`, `ADDMOBS`, torch/spike/gate animation |
| Background gfx | GAMEBG.S | $4c00 | `SETUPFLAME`, torch sprite selection |
| Hi-res rendering | HIRES.S | — | `LAY`, `FASTLAY`, `LAYRSAVE` — sprite drawing |
| Graphics pipeline | GRAFIX.S | — | `DRAWALL`, `ADDMID` — compositing layers |
| Frame rendering | FRAMEADV.S | — | `DrawFF`, block redraw, object rendering |
| Sound | SOUND.S / SOUNDNAMES.S | $ea00 | `addsound`, `PLAYBACK` |
| Game equates | GAMEEQ.S / EQ.S | — | Memory map, character tables, constants |

### Key Mechanisms

**Keyboard input:** `$C000` bit 7 = strobe (new key), bits 0-6 = ASCII. Cleared by writing `$C010`. SPECIALK.S reads this in `KEYS`, routes through `LegitKeys` and `DevelKeys` via cascading `cmp`/`bne` chains.

**Animation:** Characters have a `CharSeq` 16-bit pointer into SEQTABLE.S bytecode. `ANIMCHAR` reads bytes: negative = instruction (chx, chy, act, goto, tap, effect, etc.), positive = frame number stored in `CharPosn`. Frame numbers index into `Fdef` (5 bytes each: image#, sword data, dx, dy, collision marks). Images come from character tables (chtable1-7) in aux memory.

**Sword strike flow:** Button press in `FightCtrl` → `DoStrike` validates frame position → `jumpseq` with `faststrike` (seq 75) → animation plays → `checkstrike`/`checkstab` in TOPCTRL.S `NextFrame` loop tests sword overlap → `STABCHAR` in MISC.S applies damage via `DECSTR` → opponent jumps to `stabbed` (seq 74) or `stabkill` (seq 85).

**Mobile objects (mobs):** 6 parallel arrays (mobx/moby/mobscrn/mobvel/mobtype/moblevel), max 16. Currently only used for falling floor debris. Physics is gravity-only (vertical). Each frame: `ANIMMOBS` updates positions → `ADDMOBS` adds visible mobs to object draw list → `DrawFF` in FRAMEADV.S renders them. Deleted by setting `mobvel=$ff`.

**Torch flames:** 18 sprite frames in `torchflame` table (images 52-64 from bgtable1). `GETFLAMEFRAME` uses pseudo-random flickering. Drawn via `SETUPFLAME` → `FASTLAY`. These are background-plane sprites, not character-plane.

---

## Implementation Plan

### Strategy

Implement the fireball as a **new mobile object type** (`mobtype=1`). This reuses the existing mob infrastructure (creation, per-frame update, rendering, deletion) while adding horizontal movement and opponent collision. The fireball uses torch flame sprites (already loaded in bgtable1) for its appearance — no new art needed.

### Step 1: Define Constants

**File: SEQDATA.S** — Add after `Mraise = 114`:
```assembly
* Fireball sequence (player casting animation)
fireballcast = 115
```

**File: SPECIALK.S** — Add after existing key constants (~line 129):
```assembly
kfireball = "h"      ; Fireball key (lowercase h = $e8 with high bit)
```

**File: SOUNDNAMES.S** — Reuse existing sound. `SwordClash1 = 17` or `Impaled = 14` work as fireball-launch sounds. No new sound slot needed.

**File: GAMEEQ.S or MOVER.S** — Add constants:
```assembly
TypeFireball = 1      ; mobtype value for fireballs
fbspeed = 6           ; pixels per frame horizontal speed
fbmaxdist = 140       ; max travel distance (pixels) before expiring
```

### Step 2: Add Key Handler in SPECIALK.S

**Where:** In `LegitKeys` routine (line 265+), add a new check before the final `rts`. Insert after the `:19` / `]rts rts` at line 352-353:

```assembly
* --- FIREBALL KEY ---
:19 cmp #kfireball+$80  ; 'h' with high bit set (keyboard strobe)
 bne :fbrts
 jsr DoFireball
:fbrts
]rts rts
```

The `+$80` is because `$C000` returns ASCII with bit 7 set. Adjust if the game strips bit 7 before comparison (check how other keys like `kleft = "j"` are compared — the `KREAD` routine in SPECIALK.S handles directional keys separately from `LegitKeys`, but the `LegitKeys` comparisons use the raw `keypress` value which has bit 7 set, so the comparison values for special keys like `krestart = "r"-CTRL` already account for this). Look at how `kblackout = "B"` is handled in `TempDevel` — it compares directly with `keypress`. Follow that pattern.

**Important:** Study how existing keys in `LegitKeys` and `TempDevel` are compared. The `keypress` variable stores the raw `$C000` value. The existing key constants like `ksound = "s"-CTRL` are defined without the high bit, and comparisons work because the assembler character literals include the high bit in Merlin syntax. Match whatever convention the file uses.

### Step 3: Implement DoFireball Routine

**File: SPECIALK.S** — Add new routine (find free space near end of file, before the data tables). This is the core logic:

```assembly
*-------------------------------
*
* D O   F I R E B A L L
*
* Spawn a fireball mob traveling in the direction the kid faces
*
*-------------------------------
DoFireball
 lda KidLife
 bpl ]rts            ; Dead? No fireball

 lda KidSword
 cmp #2
 bne :nosword        ; Allow both en-garde and normal stance
:nosword

 ldx nummob
 cpx #maxmob
 bcs ]rts            ; Mob list full? Abort

* Create fireball mob
 inx
 stx nummob

 lda KidX
 sta mobx,x          ; Start at kid's X position

 lda KidY
 sec
 sbc #30             ; Offset upward to chest height
 sta moby,x

 lda KidScrn
 sta mobscrn,x

 lda #TypeFireball   ; = 1
 sta mobtype,x

 lda KidFace
 sta moblevel,x      ; REPURPOSE moblevel to store direction
                      ; (0 = facing right, $FF/-1 = facing left)

 lda #fbspeed
 sta mobvel,x         ; REPURPOSE mobvel as horizontal speed

* Play launch sound
 lda #SwordClash1
 jsr addsound

]rts rts
```

**Key design decisions:**
- `moblevel` is repurposed to store the fireball's facing direction (0=right, -1=left). For falling floors it stores the Y-layer, but fireballs don't need that.
- `mobvel` is repurposed as horizontal speed (not vertical velocity). The `ANIMMOBS` dispatcher will route based on `mobtype` before the gravity code runs.
- Fireball spawns at the kid's position, offset upward by ~30 pixels (chest height).
- We skip sequence animation for the kid (no casting pose) in the minimal version. To add a casting animation, define `fireballcast=115` in SEQTABLE.S (see Optional Enhancements).

### Step 4: Animate Fireball in MOVER.S

**Where:** In `ANIMMOBS` loop. Currently, `animmob` (the per-mob update routine) assumes all mobs are falling floors. We need to dispatch by type.

Find the call to the individual mob animation routine inside the ANIMMOBS loop. The loop calls `jsr animmob` (or equivalent) for each mob. Add a type check before it:

```assembly
* In ANIMMOBS loop, replace the direct call to mob animation:
 lda mobtype,x
 beq :isfloor        ; Type 0 = falling floor (existing code)
 cmp #TypeFireball
 beq :isfireball
 jmp :nextmob        ; Unknown type, skip

:isfloor
 jsr animmob         ; Existing falling floor logic
 jsr checkcrush
 jmp :nextmob

:isfireball
 jsr animfireball    ; NEW: fireball movement
 jmp :nextmob
```

**New routine `animfireball`** — add in MOVER.S:

```assembly
*-------------------------------
*
* A N I M   F I R E B A L L
*
* Move fireball horizontally, check for opponent hit, expire if out of range
*
*-------------------------------
animfireball
* Move horizontally based on direction
 lda moblevel        ; Direction: 0=right, $FF=left
 bne :goleft

:goright
 lda mobx
 clc
 adc mobvel           ; Add speed (positive = rightward)
 sta mobx
 cmp #255             ; Off right edge?
 bcs :expire
 bcc :checkscreen

:goleft
 lda mobx
 sec
 sbc mobvel           ; Subtract speed (move leftward)
 sta mobx
 bcc :expire          ; Underflow = off left edge

:checkscreen
* Check if fireball is still on a valid screen
 lda mobscrn
 beq :expire          ; Null screen

* Check for opponent collision
 jsr fbcheckhit
 bcs :hit

* Cycle flame animation frame
 jsr rnd
 and #$0f             ; 0-15
 cmp #torchLast
 bcc :frok
 lda #0
:frok sta fbframe      ; Store for rendering

 rts                   ; Continue next frame

:hit
* Fireball hit opponent — deal damage and expire
 jsr LoadShad          ; Load opponent data into Char* variables
 lda CharLife
 bpl :expire           ; Already dead

 lda #1
 jsr decstr            ; Deal 1 point of damage
 beq :killed

 lda #stabbed          ; Non-fatal: play "stabbed" animation
 jsr jumpseq
 jsr animchar
 jsr SaveShad
 jmp :expire

:killed
 lda #stabkill         ; Fatal: play death animation
 jsr jumpseq
 jsr animchar
 jsr SaveShad

:expire
 lda #$ff
 sta mobvel            ; Mark for deletion
 rts

*-------------------------------
* Check if fireball is hitting the opponent
* Returns: C=1 if hit, C=0 if no hit
*-------------------------------
fbcheckhit
 lda mobscrn
 cmp OpScrn
 bne :nohit            ; Different screens

 lda moby
 cmp OpY
 bcs :nohit            ; Fireball below opponent feet
 lda OpY
 sec
 sbc #50               ; Opponent height (~50 pixels)
 cmp moby
 bcs :nohit            ; Fireball above opponent head

* Check X proximity
 lda mobx
 sec
 sbc OpX
 bpl :posX
 eor #$ff              ; Absolute value
 clc
 adc #1
:posX
 cmp #12               ; Within 12 pixels horizontally?
 bcs :nohit

 sec                   ; HIT
 rts

:nohit
 clc
 rts
```

**Local variable needed:**
```assembly
fbframe ds 1           ; Add to the locals dum block at top of MOVER.S
```

### Step 5: Render Fireball in FRAMEADV.S / MOVER.S

**Where:** In `ADDMOBS` (MOVER.S). Currently it only draws `mobtype=0`. Add fireball rendering:

In the `ADDMOBS` loop, after the existing type check:

```assembly
* In ADDMOBS loop:
 lda mobtype,x
 beq :drawfloor       ; Type 0 = falling floor
 cmp #TypeFireball
 beq :drawfireball
 jmp :nextmob

:drawfloor
 jsr ATM               ; Existing: Add This Mob (falling floor)
 jmp :nextmob

:drawfireball
 jsr ATFB              ; NEW: Add This Fireball to draw list
 jmp :nextmob
```

**New routine `ATFB`** — Render fireball using torch flame sprites:

```assembly
*-------------------------------
*
* A T F B — Add This Fireball to object draw list
*
* Uses torch flame sprites from bgtable1 for the fireball image.
*
*-------------------------------
ATFB
 lda mobscrn
 cmp VisScrn
 bne ]rts              ; Not on visible screen

* Set up drawing parameters
 lda mobx
 lsr
 lsr                   ; Convert pixel X to byte X (divide by 7 approx)
                        ; NOTE: actual conversion depends on hi-res coord system
                        ; Study how ATM converts mobx to XCO
 sta XCO

 lda moby
 sta YCO

* Select flame sprite from torchflame table
 ldx fbframe
 lda torchflame,x      ; Get sprite image # (52-64 range)
 sta IMAGE

* Set drawing parameters
 lda #<bgtable1
 sta TABLE
 lda #>bgtable1
 sta TABLE+1

 lda #3                ; Bank = aux memory
 sta BANK

 lda #sta_opcode       ; OPACITY = STA (opaque draw)
 sta OPACITY

 lda #0
 sta OFFSET

* Draw using FASTLAY (same as torch rendering)
 jmp FASTLAY
```

**IMPORTANT:** The coordinate conversion and drawing parameter setup above is **pseudocode**. You must study how `ATM` (the existing mob renderer in MOVER.S) and `SETUPFLAME` (in GAMEBG.S, lines 735-757) set up `XCO`, `YCO`, `IMAGE`, `TABLE`, `BANK`, `OPACITY`, and `OFFSET` for their respective sprites, and replicate that pattern exactly. Key references:
- `SETUPFLAME` in GAMEBG.S ~line 735: sets up torch flame for FASTLAY
- `ATM` / `addmobobj` in MOVER.S ~line 2086: sets up falling floor for object list
- `FASTLAY` in HIRES.S ~line 1740: the actual blitting routine

The fireball can either:
1. **Use FASTLAY directly** (like torches) — simpler, but no background save/restore (fireball will leave trails unless background is redrawn)
2. **Use ADDMID + LAYRSAVE** (like characters) — cleaner, fireball properly composited with background restoration via the peel system

Option 2 is cleaner. Follow the `addmobobj` pattern to add the fireball to the object list with its own type flag, then handle it in `drawobjs` (FRAMEADV.S) similar to `DrawFF` but using torch flame sprites.

### Step 6: Handle Screen Transitions

When a fireball crosses a screen boundary (X < 0 or X >= screen width), it should expire. The simplest approach: just let it expire (set `mobvel=$ff`) when X goes out of bounds, as implemented in `animfireball` above.

If you want the fireball to cross into adjacent screens (more complex):
- Look up the adjacent screen number from the level blueprint
- Update `mobscrn` accordingly
- Adjust `mobx` to wrap around

For a first implementation, **just expire at screen edges**. This is simpler and avoids edge cases.

### Step 7: Memory Considerations

The Apple II has very tight memory. Key constraints:
- Each source file has a fixed org address and must not overflow into the next module
- Zero page ($00-$FF) is heavily used; only use existing local variable blocks (`dum locals` / `dend` sections)

**Where to put new code:**
- `DoFireball` (~40 bytes): Add to SPECIALK.S. Check available space between end of code and next module at $e000 (`subs`). SPECIALK.S starts at $d900 — you have until $dfff.
- `animfireball` + `fbcheckhit` (~100 bytes): Add to MOVER.S. It starts at $ee00 — check available space before $f900 (`misc`). That's ~2.75 KB available.
- `ATFB` rendering (~40 bytes): Add to MOVER.S alongside ADDMOBS.

**Check available space before coding:** Assemble with `VERBOSE=1` and examine the .LST files in `obj/` to see actual code sizes and remaining space in each module.

---

## File Edit Summary

| File | Changes |
|------|---------|
| **SEQDATA.S** | Add `fireballcast = 115` constant |
| **SPECIALK.S** | Add `kfireball` key constant; add handler in `LegitKeys`; add `DoFireball` routine |
| **MOVER.S** | Add `TypeFireball`, `fbspeed` constants; modify `ANIMMOBS` loop to dispatch by type; add `animfireball`, `fbcheckhit`, `ATFB` routines; modify `ADDMOBS` to render fireballs |
| **GAMEBG.S** | No changes — `torchflame` table and `SETUPFLAME` are read-only references |
| **FRAMEADV.S** | Potentially add `DrawFB` if using object-list rendering (option 2) |
| **SOUNDNAMES.S** | Optionally add `FireballLaunch` alias for an existing sound |

---

## Build & Test

```bash
cd /Users/nick/Projects/Prince-of-Persia-Apple-II
make clean all
# Load PrinceOfPersia_3.5.hdv in Virtual II (Apple IIe + UniDisk 3.5)
# Get to a level with a guard
# Press H to throw fireball
```

**Test scenarios:**
1. Press H with no enemy on screen — fireball should fly and expire at screen edge
2. Press H facing a guard — fireball should hit and deal 1 damage
3. Press H multiple times rapidly — should spawn multiple fireballs (up to mob limit)
4. Fireball should visually look like a flickering torch flame flying horizontally
5. Confirm no graphical corruption or crashes

---

## Optional Enhancements (Future)

### Casting Animation
Add a new sequence (115) in SEQTABLE.S — a 3-frame arm-thrust animation. Call `jumpseq` with `fireballcast` in `DoFireball` before spawning the mob. Reuse existing frames (e.g., sword thrust frames 150-157) or define new ones in FRAMEDEF.S.

### Cooldown Timer
Add a `fbcooldown` variable. Set to ~30 frames after each fireball. Check in `DoFireball` and skip if non-zero. Decrement in `NextFrame`.

### Only When En Garde
Restrict fireball to sword-drawn state (`KidSword = 2`) to make it a combat ability rather than a universal one.

### Fireball Sound
Create a distinctive sound by calling the speaker toggle (`$C030`) in a rapid descending pattern, or reuse `Impaled` (14) which has an aggressive tone.

### Trail Effect
Spawn a second "fading" mob 1 frame behind the fireball with a different flame frame, creating a comet-tail effect.

### Guard Fireballs
Let guards throw fireballs too — in AUTO.S, add random fireball attacks. Use the same mob system with `moblevel` direction reversed.
