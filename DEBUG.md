# Fireball Feature - Debug Log

## Feature Goal
Press H to shoot a fireball that travels horizontally in the kid's facing direction, damages opponents on contact, and uses torch flame sprites for visuals.

## Current State
- Fireball spawns, moves, and **hits enemies correctly**
- **Problem**: Ghost sprite trail - old flame images are never erased, leaving a dotted trail across the screen
- No crash, no freeze - purely a rendering cleanup issue

## Architecture Overview

### Game Loop Order (TOPCTRL.S)
```
MainLoop:
  jsr demokeys
  jsr misctimers
  jsr NextFrame        ← computes next frame state
    jsr animmobs       ← MOVER.S: animates MOBs (our animfireball runs here)
    jsr animtrans
    jsr DoKid          ← calls ctrlplayer → keys (SPECIALK.S key handler)
    jsr DoShad
    jsr checkstrike/checkstab
    ...
  jsr flashon
  jsr FrameAdv         ← draws frame
    jsr DoFast
      jsr zerolsts     ← CLEARS all image lists (bg/mid/fg)
      jsr addmobs      ← MOVER.S ADDMOBS: our renderfb runs here
      jsr addchars
      jsr fast         ← FRAMEADV.S: builds image lists from objects + bg
      jsr dispmsg
      jmp drawall      ← GRAFIX.S: renders all lists to screen
        SNGPEEL        ← restores peels from previous frame
        DRAWBACK       ← draws background list (fastlay)
        DRAWMID        ← draws mid list (fastlay/lay/layrsave)
        DRAWFORE       ← draws foreground list
  jsr playback
  ...
```

### Key Rendering Insight
- `zerolsts` clears all image lists at the START of DoFast/FrameAdv
- Anything added to lists during NextFrame (before FrameAdv) is lost
- `ADDMOBS` runs inside DoFast, AFTER zerolsts - correct time to add to lists
- `addback` stamps permanently with FASTLAY (STA opacity) - no cleanup mechanism for moving objects
- `addmidez` with type 2 (layrsave) should save/restore background under sprites via the peel system

### How Torch Flames Work (for reference)
- FRAMEADV.S `drawtorchb`: sets XCO/YCO, calls `setupflame`, then `jmp addback`
- `setupflame` (GAMEBG.S): sets IMAGE from torchflame table, TABLE=bgtable1, OPACITY=sta
- Works without ghost trails because torches are STATIONARY - the background tile at that position is always redrawn with the flame on top

### How Falling Floors Work (the other MOB type)
- ATM in MOVER.S: marks blocks dirty via `markfloor`/`markfred`, then calls `addmobobj`
- `addmobobj`: adds to the OBJECT list (objX/objY/objIMG/objTYP arrays), NOT directly to bg/mid/fg lists
- Objects are processed by `fast` in FRAMEADV.S which handles dirty tracking and inserts into appropriate image lists
- Uses character frame images (mobframe), not bg table images

## Files Modified

### SPECIALK.S (key handler + fireball spawn)
- Added `kfireball = "h"` constant (~line 107)
- Added key handler dispatch in LegitKeys (~line 354): `cmp #kfireball / bne :20 / jmp DoFireball`
- Added `DoFireball` routine: creates mob in mob arrays (mobx, moby, mobscrn, mobtype=1, moblevel=direction, mobvel=1)
- Spawn coordinates: `KidBlockX * 4 + 2` for mobx (byte columns), `KidY - 30` for moby

### MOVER.S (mob animation + rendering)
- Modified `animmob`: dispatches to `animfireball` via `jmp` for mobtype != 0 (bypasses mobvel inc/dec that interferes with deletion)
- Modified ANIMMOBS loop: skips `checkcrush` for non-floor mobs
- Modified ADDMOBS: dispatches to `renderfb` for mobtype != 0
- Added `animfireball`: moves mob 1 byte-column/frame, checks opponent collision (X-block match + same screen), sets mobvel=$ff to delete on hit or screen edge
- Added `renderfb`: sets up rendering params and adds fireball to image list

## Approaches Tried and Results

### Attempt 1: Direct FASTLAY call from ADDMOBS
- `drawfireball` set up XCO/YCO/IMAGE/TABLE/OPACITY and called `jmp fastlay`
- **Result**: Game crashed (hard freeze)
- **Root cause**: MOVER.S binary was 2862 bytes, overflowing the 2816-byte limit ($EE00-$F900) by 46 bytes, corrupting MISC module at $F900

### Attempt 2: Direct FASTLAY from SPECIALK.S (standalone, no mob system)
- Moved all fireball code to SPECIALK.S with standalone variables (fbactive, fbx, fby, fbdir, fbscrn)
- UpdateFireball called from KEYS routine: animated + rendered via `jmp fastlay`
- **Result**: No crash, but ghost flicker trail + no visible fireball animation
- **Root cause**: KEYS runs during NextFrame, before FrameAdv. Drawing with FASTLAY during key handling draws on the wrong page / at the wrong time. `zerolsts` at start of DoFast doesn't help because FASTLAY writes directly to hi-res memory, not through the image list system. The sprites are drawn but immediately overwritten by the background redraw, OR drawn on the visible page instead of the back buffer.

### Attempt 3: addback (background image list)
- Used `jsr addback` instead of direct FASTLAY call
- Properly called from ADDMOBS during DoFast (correct timing)
- **Result**: Fireball visible and moving, hits enemies, but leaves ghost trail
- **Root cause**: `addback` adds to the background list rendered by DRAWBACK with FASTLAY STA opacity. This permanently stamps pixels onto the hi-res page. Next frame, only blocks marked "dirty" get the background rebuilt. The fireball's old positions are never cleaned up because nothing marks them dirty.

### Attempt 4: addback + markfloor/markfred
- Added dirty-block marking in `animfireball` before moving: computed block position, called `markfloor` and `markfred` to trigger background redraw
- Theory: marking old position dirty → `fast` rebuilds background there → old stamp cleaned
- **Result**: Still ghost trail, no improvement
- **Possible reasons**: (a) markfloor/markfred might not fully redraw the dark corridor area where the fireball flies (only floor/wall tiles, not empty space), (b) the dirty flag might be consumed before DRAWBACK runs, (c) the rebuilt background might not cover the exact pixels the flame sprite occupied

### Attempt 5: addmidez with type 2 (layrsave/peel)
- Changed `renderfb` to use `addmidez` with A=2 (layrsave mode) instead of `addback`
- Set TABLE = bgtable1 low byte for DRAWMID's `setbgimg` path
- Theory: DRAWMID with layrsave saves background under sprite, draws sprite, restores background next frame via SNGPEEL
- **Result**: Still ghost trail, no improvement
- **Possible reasons**: Need to investigate - maybe TABLE setup is wrong (addmidez stores only low byte of TABLE, but setbgimg uses bit 7 of IMAGE to select table), or layrsave/lay might not work correctly with bg table images in this context, or the peel buffer might be full

## Key Open Questions

1. **Why doesn't layrsave/peel work?** The peel system works for characters - what's different about the fireball? Possible issues:
   - TABLE param: `addmidez` stores TABLE low byte in midTAB. DRAWMID loads it back and calls `setbgimg`. But setbgimg computes TABLE from IMAGE bit 7, ignoring the stored TABLE value. IMAGE=$52 has bit 7 clear → bgtable1 ($6000). This should be correct.
   - SNGPEEL timing: peels are restored at the start of `drawall` (before DRAWBACK). If ADDMOBS adds the fireball to the mid list, and DRAWMID draws it with layrsave, the peel should be restored next frame. Unless the peel buffer is full or the peel data is being overwritten.
   - Double buffering: PAGE flips between $00/$20. The peel system saves/restores on the current PAGE. If the fireball is drawn on page A, the peel restores page A next frame. But next frame might be drawing on page B. Need to check if the peel system handles both pages.

2. **Is the flame image compatible with LAY?** FASTLAY and LAY read the same image format (width byte, height byte, then pixel data). The torch flame images in bgtable1 are used with FASTLAY for normal torches. LAY should handle them too, but maybe there's a subtlety.

3. **Should we use the OBJECT list (addmobobj) instead?** Falling floors go through `addmobobj` → object list → processed by `fast`. This integrates deeply with the dirty-block tracking system. The challenge: addmobobj uses `mobframe` which indexes into character tables (chtable1-7), not bg tables. Flame sprites are in bgtable1, not character tables.

4. **Could we draw with EOR opacity?** EOR (exclusive-or) can erase by drawing the same image twice. We'd need to draw at the old position with EOR to erase, then draw at the new position. But this requires two FASTLAY calls per frame and precise position tracking.

5. **Is there a page/timing mismatch?** The game uses double buffering. ADDMOBS runs during DoFast which draws on the HIDDEN page. SNGPEEL restores peels on the hidden page too. If our sprite is being drawn on one page but the peel is being restored on the other, we'd get ghosts.

## Recommendations for Next Session

1. **Deep-dive the peel system**: Read SNGPEEL, ADDPEEL, and trace exactly what happens frame-by-frame. Check if the peel buffer has capacity, if PAGE is correct, if the save/restore addresses match.

2. **Try addmobobj approach**: Despite the bg-vs-char table mismatch, investigate whether we can add a flame frame to a character table, or modify addmobobj to handle bg images.

3. **Try manual EOR erase**: Store the previous position, and in renderfb, first draw EOR at the old position (erasing), then draw STA at the new position. This bypasses the peel system entirely.

4. **Instrument/test with simple rectangle**: Replace the flame image with a simple known pattern to rule out image format issues.

5. **Check if addmidez is actually being called**: Verify via the listing file (obj/MOVER.LST) that the `jmp addmidez` instruction is at the expected address and the addmidez vector is correct.

## Module Size Budget
| Module | Current | Max | Spare |
|--------|---------|-----|-------|
| MOVER.S | 2793 | 2816 | 23 bytes |
| SPECIALK.S | 1396 | 1752 | 356 bytes |
