# Development Guide

## Building the Project

### From MPLAB X IDE
1. Open the project: **File → Open Project** → select repo directory
2. Select configuration: **right-click project → Set Configuration → default**
3. Build: **Run → Build Project** or **Ctrl+F11**
4. Output: `dist/embedded-racing-game.hex` and `dist/embedded-racing-game.elf`

### From Command Line
```bash
# Using MPLAB X tools (if in PATH)
make build

# Or directly with pic-as
pic-as main.asm -Werror -v -fmax-errors=10
```

---

## Simulating with SimulIDE

### Setup
1. **Download SimulIDE** from [https://www.simuide.org](https://www.simuide.org)
2. Open `Project.sim1` in SimulIDE
3. Double-click the PIC16F877A virtual chip
4. Set firmware: `dist/embedded-racing-game.hex`
5. Set clock: 10 MHz (HS oscillator)

### Running
- **Play button** (▶) — start simulation
- **Step button** — step one instruction
- Click virtual **buttons** (connected to RA0, RA1, RA2) to play

### Inspecting State
- **Hover over registers** — see live values
- **Double-click RAM** — open memory browser (watch addresses)
- **View → Oscilloscope** — monitor timing (PORTB, etc.)

---

## Debugging Tips

### Using MPLAB X Debugger
```
Run → Debug Project   (or F5)
```

**Breakpoints:**
- Click line number gutter to set breakpoints
- Program pauses at breakpoints; inspect watches

**Watches:**
- **Window → Debugging → Watches**
- Add register/RAM address: `GAME_STATE`, `FUEL_LEVEL`, etc.
- Watch updates on pause

**Call Stack:**
- **Window → Debugging → Call Stack**
- Trace execution path (useful for ISR debugging)

### Common Issues

**Hang on startup:**
- Check GLCD wiring (RST signal critical)
- Add delays after RST pulse
- Verify PORTB init (all outputs low)

**Display flicker:**
- Likely ISR collision with rendering
- Reduce ISR frequency (increase prescaler)
- Use volatile flags instead of repeated polling

**Button not responding:**
- Check PORTA TRIS (RA0–2 should be inputs)
- Verify pull-up resistors (internal weak pull-ups in PIC; external recommended)
- Test BUTTON_SCAN subroutine in isolation

**EEPROM not persisting:**
- Check magic sequence (0x55, 0xAA)
- Verify interrupts disabled during write
- Use sim with EEPROM support (some sims don't)

---

## Modifying the Code

### Adding a Feature

**Example: Sound on collision**

1. **Edit ISR/main loop** to set flag:
   ```assembly
   CHECK_COLLISION:
       ; ... collision detection ...
       COLLISION_FOUND:
           BSF ISR_FLAGS, FLAG_COLLISION
           MOVLF 50, FUEL_TICK        ; Beep duration
   ```

2. **Add PWM routine** (RC2/PWM5):
   ```assembly
   SOUND_BEEP:
       ; Setup CCP1 for PWM mode (10-bit, PR2=99 for ~1kHz)
       BANKSEL CCP1CON
       MOVLW 0x0C                    ; PWM mode
       MOVWF CCP1CON
       BANKSEL PR2
       MOVLW 99
       MOVWF PR2
       ; Fade out with delay loop
   ```

3. **Call from PLAY_LOOP**:
   ```assembly
   BTFSC ISR_FLAGS, FLAG_COLLISION
   CALL SOUND_BEEP
   ```

### Optimizing for Space

The project is **tight on ROM** (~4KB used of 4KB available).

**Space-saving techniques:**
- Use macro expansions judiciously (inline vs. subroutine trade-off)
- Combine related variables (use bit fields)
- Eliminate redundant data (reference FONT_TABLE instead of copying)
- Share code paths (e.g., use generic loops with counters)

### Changing Game Parameters

**Located in constants section (lines 99–154):**

```assembly
FUEL_MAX        EQU 100      ; Change max fuel
FUEL_DEC_INT    EQU 10       ; Change depletion rate
SPEED_LVL1      EQU 8        ; Ticks per update (lower = faster)
SPAWN_LVL1      EQU 5        ; Enemy spawn rate
SCORE_LVL2      EQU 10       ; Score threshold for level-up
```

**To make game easier:**
- Increase `FUEL_MAX`
- Decrease `SPAWN_LVL*` (fewer enemies)
- Increase `SPEED_LVL*` (slower enemies)

**To make game harder:**
- Opposite changes

---

## Hardware Bring-Up Checklist

- [ ] **Oscillator:** HS @ 10 MHz (verify with scope)
- [ ] **Power & Ground:** Clean supply (decoupling caps on Vdd/Vss)
- [ ] **GLCD wiring:** Test data bus and control signals individually
  - [ ] CS1, CS2 toggle independently
  - [ ] EN pulse visible on scope
  - [ ] Data lines valid during write
- [ ] **Button inputs:** Test with multimeter (toggle between 5V and GND)
- [ ] **Reset circuit:** RC filter (100Ω, 10µF typical)
- [ ] **ICSP header:** Program via MPLAB ICD or similar
- [ ] **PIC internal oscillator (if used):** Set FOSC = INTRC in config

### First Power-On
1. Slowly ramp supply (use bench PSU, not direct battery)
2. Monitor current (should be <20 mA)
3. Watch GLCD for splash screen (should appear within 1 second)
4. Test buttons (menu should respond)

---

## Testing Harness

To test individual subroutines in isolation:

```assembly
; At top of main.asm, before SYSTEM_INIT
#define TEST_MODE 1

#ifdef TEST_MODE
TEST_DRAW_CHAR:
    ; Initialize GLCD
    CALL GLCD_INIT
    CALL GLCD_CLEAR_ALL
    
    ; Test drawing letter 'A'
    MOVLF 3, DRAW_PAGE
    MOVLF 10, DRAW_X
    MOVLF FC_A, TEMP1
    CALL DRAW_CHAR
    
    GOTO $ ; Infinite loop (freeze here to inspect output)
#endif
```

Then in SYSTEM_INIT:
```assembly
#ifdef TEST_MODE
GOTO TEST_DRAW_CHAR
#endif
```

---

## Performance Profiling

### Measuring Frame Time
Use Timer1 to measure elapsed cycles between frames:

```assembly
PROFILE_START:
    BANKSEL TMR1H
    CLRF TMR1H
    CLRF TMR1L
    BSF T1CON, 0            ; Enable Timer1
    ; ... game code ...
    BCF T1CON, 0            ; Stop Timer1
    BANKSEL TMR1H
    MOVF TMR1H, W           ; Read high byte
    ; Store for inspection
```

**Typical frame: ~1000–2000 cycles @ 10 MHz = 100–200 µs per frame**

---

## Common ASM Pitfalls

1. **Bank Selection Forgotten**
   ```assembly
   ; WRONG: assumes BANK0
   MOVF PORTB, W
   
   ; RIGHT:
   BANKSEL PORTB
   MOVF PORTB, W
   ```

2. **W Register Not Preserved in ISR**
   ```assembly
   ISR_HANDLER:
       MOVWF W_TEMP        ; MUST save first!
       ; ... ISR code ...
       SWAPF W_TEMP, W
       SWAPF W_TEMP, W     ; Restore without affecting flags
   ```

3. **16-bit Arithmetic Without Carry**
   ```assembly
   ; WRONG: loses carry
   INCF SCORE_LO, F
   INCF SCORE_HI, F
   
   ; RIGHT:
   INCF SCORE_LO, F
   BTFSC STATUS, 0        ; Check carry
   INCF SCORE_HI, F
   ```

4. **Infinite Delays in ISR**
   ```assembly
   ; WRONG: can deadlock main loop
   ISR_HANDLER:
       MOVLW 100
       CALL DELAY_TICKS    ; ISR blocks!
   
   ; RIGHT: Use flags, handle in main
   ISR_HANDLER:
       BSF ISR_FLAGS, FLAG_WORK
       RETURN
   
   MAIN_LOOP:
       BTFSC ISR_FLAGS, FLAG_WORK
       CALL DO_WORK        ; Process in main context
   ```

---

## Release Checklist

Before committing to production:

- [ ] **All game states tested** (Menu, Play, Pause, GameOver)
- [ ] **Button debouncing stable** (no ghost inputs)
- [ ] **No ROM overflow** (size within PIC16F877A 4KB limit)
- [ ] **RAM not overflowed** (stack OK, no unintended overwrites)
- [ ] **EEPROM persistence verified** (power cycle test)
- [ ] **GLCD rendering smooth** (no flicker or visual glitches)
- [ ] **Frame rate stable** (~10 fps, no stutters)
- [ ] **Comments updated** (document recent changes)
- [ ] **Version tag created** (git tag v1.0, etc.)

---

## Useful References

- [PIC16F877A Datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU08/ProductDocuments/DataSheets/40300E.pdf)
- [MPLAB X IDE User Guide](https://www.microchip.com/mplab)
- [pic-as Assembler Manual](https://www.microchip.com/en-us/tools-resources/develop/mplab-xc-compilers)
- [KS0108 GLCD Controller Datasheet](https://www.digchip.com/datasheets/parts/datasheet/187/KS0108.html)
- [SimulIDE Documentation](https://www.simuide.org/wiki/index.php)

