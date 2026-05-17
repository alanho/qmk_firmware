# AGENTS.md — flashing firmware on this fork

Notes for AI agents (and humans) on **reliably** flashing satisfaction75 firmware from this repo on macOS. Written from painful experience.

## Toolchain (one-time)

```sh
# QMK CLI + ARM toolchain
brew install qmk/qmk/qmk arm-none-eabi-gcc arm-none-eabi-binutils dfu-util

# binutils is keg-only — must be on PATH for objcopy / size / objdump
echo 'export PATH="/opt/homebrew/opt/arm-none-eabi-binutils/bin:$PATH"' >> ~/.zshrc
```

If a build fails with `arm-none-eabi-objcopy: command not found`, re-export the PATH for the current shell:
```sh
export PATH="/opt/homebrew/opt/arm-none-eabi-binutils/bin:$PATH"
```

## Build

VIA is required for in-VIA remapping; bake it into every build:

```sh
qmk compile -kb cannonkeys/satisfaction75/rev1 -km default -e VIA_ENABLE=yes
```

Output `.bin` lands at the repo root: `cannonkeys_satisfaction75_rev1_default.bin`.

`rev1` is the right target for boards reporting USB PID `0x57F5` (`ioreg -c IOHIDDevice -r | grep -A1 -i sat`). Use `rev2` for PID `0x001A`, `satisfaction75_hs` for `0x0011`.

## Flashing — the only path that works reliably

```sh
# 1. Get board into DFU. Either:
#    a) hold Fn (rightmost-bottom key) + LCtrl (leftmost-bottom)  ← if keymap has QK_BOOT on layer 1
#    b) unplug, hold the boot button on the PCB, plug back in
#
# 2. Verify DFU enumerated:
dfu-util -l   # must show "Found DFU: [0483:df11] ..."

# 3. Flash. Run this command directly — do NOT pipe it.
dfu-util -a 0 -s 0x08000000:leave -D cannonkeys_satisfaction75_rev1_default.bin
```

Total elapsed time: ~25–35 seconds. Live progress bars stream to the terminal.

### dfu-util "hangs" at the end — that's normal on macOS

After the download completes you will see:

```
File downloaded successfully
Submitting leave request...
Transitioning to dfuMANIFEST state
```

…and dfu-util then **never exits**. This is a known dfu-util-on-macOS quirk — the program is waiting for a USB re-enumeration event that macOS doesn't deliver. The flash itself completed at `File downloaded successfully`; the board has already rebooted and is running the new firmware.

Safe handling:
- `Ctrl+C` the hung dfu-util once `Transitioning to dfuMANIFEST state` appears.
- Or wrap with GNU `timeout` (`brew install coreutils`):
  ```sh
  gtimeout 35 dfu-util -a 0 -s 0x08000000:leave -D cannonkeys_satisfaction75_rev1_default.bin
  ```
- Do not interpret the hang as a failed flash. Plug-in the board and test — it will be running the new firmware.

## Hard rules — break these and you wedge USB

1. **Never pipe dfu-util through `tail`, `head`, `grep`, or similar.** They buffer until exit, hiding the progress bars and looking like a hang. Reading file `wc -c` of the output stream will show 0 bytes while the flash is actually proceeding — easy to misdiagnose as stuck.

2. **Never kill dfu-util mid-flash.** A killed dfu-util leaves the USB endpoint in `dfuERROR` state. The next attempt may not even open the device. Recovery requires physical unplug → replug.

3. **If a flash appears stuck:** wait at least 60 seconds before doing anything. If the output file is still zero bytes after that, the USB is genuinely wedged — physically unplug, hold the PCB boot button, replug, then retry.

4. **Don't use `qmk flash`.** It compiles + flashes in one step, but adds opaque retry loops. Prefer `qmk compile` followed by an explicit `dfu-util` call so you can see exactly what's happening.

## Keymap edits don't apply once VIA has touched the board

With `VIA_ENABLE=yes`, the keymap lives in EEPROM after the first VIA interaction. Edits to `keyboards/cannonkeys/satisfaction75/keymaps/default/keymap.c` only seed the *initial* keymap when EEPROM is blank.

To re-apply firmware defaults: hold the key at matrix position `[0,0]` (Esc) while plugging in — Bootmagic Lite wipes EEPROM and reloads defaults.

**Practical rule for agents**: don't edit `keymap.c` for the user's bindings. Either:
- Add firmware **logic** (new keycodes, OLED modes, animations) and let the user bind via VIA's "Any" key with the hex value (e.g. `0x7E03` for `QK_KB_3`).
- Or instruct the user to Bootmagic-reset the EEPROM after flashing.

Custom keycodes live in `keyboards/cannonkeys/lib/satisfaction75/satisfaction_keycodes.h` starting at `QK_KB_0 = 0x7E00`.

## OLED idle / sleep behaviour

`CUSTOM_OLED_TIMEOUT = 60000` (ms) in `keyboards/cannonkeys/satisfaction75/config.h` controls when `oled_off()` runs. Any keypress / encoder / layer change calls `oled_request_wakeup()` to reset it. See `keyboards/cannonkeys/lib/satisfaction75/satisfaction_oled.c`.

## VIA's "Custom" tab buttons

The labelled buttons (Encoder Press, Clock Set, OLED Mode, …) come from the **VIA-side JSON definition's `customKeycodes` array**, not from this firmware. Adding a new keycode to firmware does not create a button — it just makes a new hex value reachable via VIA's "Any" key. To label a new button you must edit and load a draft VIA definition.

## Physical layout matters

The `satisfaction75/rev1` firmware uses `LAYOUT_default` (3 keys right of spacebar). If the PCB physically has 2 keys (2u mods + 7u space = `LAYOUT_2x2`), the firmware position `MO(1)` at matrix `[5,10]` lives on a switch that does not exist. Fix in `keymaps/default/keymap.c`:

```c
// bottom row: duplicate MO(1) so the rightmost-existing key engages it
KC_LCTL, KC_LGUI, KC_LALT,    KC_SPC,    KC_RALT, MO(1), MO(1), KC_LEFT, KC_DOWN, KC_RGHT
```

## Repo layout cheat-sheet

| Path | What |
|---|---|
| `keyboards/cannonkeys/satisfaction75/` | Per-rev (rev1/rev2/prototype) board configs, `rules.mk`, `config.h`, `keymaps/` |
| `keyboards/cannonkeys/satisfaction75/rev1/keyboard.json` | Matrix wiring + physical layout. Identifies which PCB this is. |
| `keyboards/cannonkeys/lib/satisfaction75/` | Shared firmware (encoder, OLED, core, keycodes). Pulled in via each rev's `rules.mk` VPATH. |
| `keyboards/cannonkeys/satisfaction75_hs/` | Hotswap variant — separate matrix wiring. |

Don't confuse the three rev directories. `rev1` ≠ `rev2` ≠ `hs` — different MCU pinouts.
