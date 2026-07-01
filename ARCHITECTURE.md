# Architecture Overview

## State Machine

The game operates as a finite state machine with four distinct states:

```
                    START
                      ↓
                  ┌─ MENU ←──────────────────┐
                  │   (display high scores)   │
                  │   (await SELECT)          │
                  │                           │
                  ↓                           │
              ┌─ PLAY                   GAME_OVER
              │   (main game loop)       (collision/fuel)
              │   (Timer0 ISR)                ↑
              │   (100 Hz)                    │
              ├─→ PAUSE                       │
              │   (freeze state)              │
              │   (SELECT to resume)          │
              │                               │
              └─────────────────────→─────────┘
```

### State Constants
```assembly
STATE_MENU     = 0
STATE_PLAY     = 1
STATE_PAUSE    = 2
STATE_GAMEOVER = 3
```

---

## Interrupt-Driven Execution

### Timer0 Configuration
- **Prescaler:** 256:1 (bits OPTION_REG<2:0> = 111)
- **Reload Value:** 6 (TMR0_RELOAD)
- **Effective Frequency:** ~100 Hz (at 10 MHz clock)
- **Interrupt Vector:** 0x0004

### ISR Handler (`ISR_HANDLER`)
```
1. Save context (W, STATUS, PCLATH)
2. Check INTCON.2 (Timer0 overflow flag)
3. Call TIMER0_ISR:
   - Reload TMR0 with 6
   - Increment TICK_COUNTER
   - Set FLAG_TICK in ISR_FLAGS
   - Clear interrupt flag
4. Restore context
5. Return with RETFIE
```

**Timing:** ISR overhead ~20 cycles; ~2% CPU overhead.

---

## Game Loop (Main)

Located at `MAIN_LOOP` label, executes continuously:

```assembly
MAIN_LOOP:
    BTFSC ISR_FLAGS, FLAG_TICK
    CALL GAME_TICK_HANDLER       ; Process one game frame
    
    BANKSEL GAME_STATE
    MOVF GAME_STATE, W
    ANDLW 0x03                    ; Mask to valid range
    
    ; Jump table dispatch
    ADDWF PCL, F
    GOTO MENU_LOOP
    GOTO PLAY_LOOP
    GOTO PAUSE_LOOP
    GOTO GAMEOVER_LOOP
    
    GOTO MAIN_LOOP
```

### MENU_LOOP
- Display high score header
- Draw menu options
- Scan buttons
- On SELECT: transition to PLAY state via `GAME_RESET`

### PLAY_LOOP
- Check `ISR_FLAGS.FLAG_TICK`
- Update game state:
  - Read button inputs (`BUTTON_SCAN`)
  - Move player (`MOVE_PLAYER`)
  - Update enemies (`UPDATE_ENEMIES`)
  - Spawn new enemies (`SPAWN_ENEMY`)
  - Decrement fuel (`PROCESS_FUEL`)
  - Update score
  - Check collisions (`CHECK_COLLISION`)
- Render frame (`RENDER_FRAME`)
- Check for level-up or game-over conditions

### PAUSE_LOOP
- Freeze `GAME_STATE` = PAUSE
- On SELECT: resume (clear flag, return to PLAY_LOOP)

### GAMEOVER_LOOP
- Display final score & high score comparison
- Check for new record (`CHECK_AND_SAVE_HISCORE`)
- Draw "NEW RECORD!" if applicable
- Wait for SELECT to return to MENU

---

## GLCD Driver

### Dual-Chip Architecture
The KS0108 splits a 128×64 display into **two 64×64 halves**:
- **CS1 (Chip Select 1):** Left half (columns 0–63)
- **CS2 (Chip Select 2):** Right half (columns 64–127)

Each chip manages 8 rows (pages) of 8 bits each.

### GLCD Commands
```assembly
GLCD_ON        = 0x3F    ; Enable display
GLCD_STARTLINE = 0xC0    ; Start line (typically 0)
GLCD_SET_PAGE  = 0xB8    ; Page select (0xB8 + page)
GLCD_SET_Y     = 0x40    ; Column select (0x40 + col)
```

### Core Functions

**`GLCD_WRITE_CMD`** — Write command byte
- Precondition: W register holds command
- Sets DI=0 (command mode), RW=0 (write), pulse EN
- Uses macros: `GLCD_CMD_MODE`, `GLCD_WRITE_MODE`, `GLCD_EN_PULSE`

**`GLCD_WRITE_DATA`** — Write data byte
- Precondition: W register holds byte
- Sets DI=1 (data mode), RW=0 (write), pulse EN

**`GLCD_CLEAR_HALF`** — Clear one 64×64 half
- Iterates all 8 pages (0–7)
- For each page: iterates 64 columns
- Writes 0x00 (black) to all bytes

**`GLCD_CLEAR_ALL`** — Clear entire display
- Calls `GLCD_CLEAR_HALF` twice (CS1 then CS2)

**`GLCD_PUT_BYTE`** — Write single byte at (DRAW_X, DRAW_PAGE)
- Determines which chip (CS1 or CS2) based on DRAW_X
- Sets page and column addresses
- Writes byte from DISP_BYTE

---

## Graphics Subsystem

### Sprite Storage
Car sprite occupies two rows (pages) of memory:

**Top Half** (CAR_TOP_0 through CAR_TOP_B):
```
     Bit Pattern (hex)
Row 0:  0x00
Row 1:  0x3C  (binary: 00111100 — top of hood)
Row 2:  0x7E  (binary: 01111110 — body)
...
Row 10: 0x3C  (bottom of hood)
Row 11: 0x00
```

**Bottom Half** (CAR_BOT_0 through CAR_BOT_B):
```
Similar format, representing chassis and wheels.
```

### Font System
Characters stored in `FONT_TABLE` (program memory):
- Each character = 5 bytes (5 pixels wide)
- Lookup indices: FC_A=2, FC_B=3, ..., FC_Z=27
- Numbers: FC_0=28, FC_1=29, ..., FC_9=37
- Special: FC_SPC=0, FC_GT=1, FC_EX=40, etc.

**`DRAW_CHAR` function:**
1. Scale character index: `TEMP2 = TEMP1 * 5 + TEMP1` (offset calculation)
2. Loop 5 times:
   - Fetch byte from FONT_TABLE
   - Write to GLCD via `GLCD_PUT_BYTE`
   - Increment DRAW_X
3. Add space (one blank column)

---

## Enemy & Obstacle System

### Enemy Structure
Three enemies tracked simultaneously:

```assembly
EN0_LANE, EN0_ROW    ; Enemy 0 position
EN1_LANE, EN1_ROW    ; Enemy 1 position
EN2_LANE, EN2_ROW    ; Enemy 2 position
ACTIVE_ENEMIES       ; Bitmask: bit=1 if active
```

**Lane Values:** 0, 1, or 2 (left, center, right)
**Row Values:** 1–6 (page index; player at pages 5–6)
**Inactive Sentinel:** 0xFF (use to mark dead enemies)

### Spawning
`SPAWN_ENEMY` function:
1. Check spawn timer against spawn rate for current level
2. If spawn_timer > 0: decrement and return
3. Find first inactive enemy slot
4. Use LFSR (Linear Feedback Shift Register) to pick random lane
5. Set row to top (0) and mark active
6. Reset spawn timer to level-specific value

**LFSR State:**
```assembly
LFSR_LO = 0x3C
LFSR_HI = 0x3D
```
Polynomial: x^16 + x^15 + x^13 + x^4 + 1

### Movement
`UPDATE_ENEMIES`:
1. For each active enemy:
   - Increment row (move down)
   - If row > 7: mark inactive (scrolled off screen)

---

## Collision Detection

### Hitbox Definition
**Player Car:** Occupies pages 5–6 in current lane
- Lane 0: columns 1–12
- Lane 1: columns 44–55
- Lane 2: columns 87–98

**Enemies:** Single-page rectangles, same column ranges per lane

### `CHECK_COLLISION`
1. For each active enemy:
   - If enemy.lane != player.lane: continue
   - If enemy.row not in (5, 6): continue
   - **COLLISION DETECTED:** Set FLAG_COLLISION, transition to GAMEOVER

---

## Fuel & Scoring

### Fuel Mechanics
```assembly
FUEL_LEVEL      ; Current fuel (0–100)
FUEL_DEC_TIMER  ; Counter (decrements every FUEL_DEC_INT ticks)
FUEL_LANE       ; Fuel pickup lane (0xFF if none)
FUEL_ROW        ; Fuel pickup row
```

**Fuel Depletion:**
- Decrements once per 10 game ticks (every ~0.1 seconds)
- Affects HUD (needs redraw when changed)

**Fuel Pickup:**
- Random spawn (similar to enemies)
- Collision grants +20 fuel (capped at FUEL_MAX=100)
- Resets fuel disappearance timer

**Game Over Condition:**
- FUEL_LEVEL reaches 0

### Scoring
```assembly
SCORE_LO, SCORE_HI  ; 16-bit score
LEVEL               ; Current level (1–5)
```

**Score Increment (per frame):**
- Level 1: +1 pt
- Level 2: +10 pts
- Level 3: +25 pts
- Level 4: +50 pts
- Level 5: +80 pts

**Level Progression:**
- Auto-increment LEVEL when SCORE reaches thresholds
- SCORE_LVL2=10, SCORE_LVL3=25, SCORE_LVL4=50, SCORE_LVL5=80

---

## EEPROM Persistence

### High Score Storage
```assembly
EE_HISCORE_LO  = 0x00  ; Low byte
EE_HISCORE_HI  = 0x01  ; High byte
EE_SENTINEL    = 0x02  ; Validity marker
EE_VALID       = 0xA5  ; Sentinel value
```

### `EEPROM_LOAD_HISCORE`
1. Read sentinel from address 0x02
2. Compare with 0xA5
3. If valid: load bytes 0x00 and 0x01 into HISCORE_LO/HI
4. If invalid: clear both to 0x00 (new device)

### `EEPROM_SAVE_HISCORE`
1. Write HISCORE_LO to address 0x00
2. Write HISCORE_HI to address 0x01
3. Write sentinel 0xA5 to address 0x02

**Note:** EEPROM write requires magic sequence:
```assembly
BCF INTCON, 7           ; Disable interrupts
MOVLW 0x55
MOVWF EECON2
MOVLW 0xAA
MOVWF EECON2
BSF EECON1, 1           ; Start write
; Wait for EECON1.1 to clear
```

---

## RAM Organization

| Address | Symbol | Purpose | Bytes |
|---------|--------|---------|-------|
| 0x20 | GAME_STATE | Current FSM state | 1 |
| 0x21 | LEVEL | Game difficulty (1–5) | 1 |
| 0x22 | ISR_FLAGS | Interrupt flags | 1 |
| 0x23 | TICK_COUNTER | Frames since boot | 1 |
| 0x24 | GAME_TICK | Local frame counter | 1 |
| 0x25 | PLAYER_LANE | Current lane (0–2) | 1 |
| 0x26 | PREV_LANE | Previous lane | 1 |
| 0x27 | FUEL_LEVEL | Current fuel | 1 |
| 0x28 | FUEL_DEC_TIMER | Fuel countdown | 1 |
| 0x29–0x2A | SCORE_LO/HI | 16-bit score | 2 |
| 0x2B | SPEED_LEVEL | Ticks between updates | 1 |
| 0x2C | ROAD_SCROLL | Animation counter | 1 |
| 0x2D–0x2F | DRAW_X, DRAW_PAGE, DISP_BYTE | Render state | 3 |
| 0x30–0x3F | DISP_TEMP, LOOP_CNT, etc. | Temporary/working | 16 |
| 0x50–0x58 | EN0/1/2_LANE/ROW, FUEL_LANE/ROW | Enemy data | 9 |
| 0x59–0x5F | SCORE_DISP, HISCORE_LO/HI, etc. | Display cache | 7 |
| 0x60–0x7D | CAR_BOT/TOP_* | Sprite memory | 28 |

**Free RAM:** ~10 bytes (0x3E–0x47) — very tight!

