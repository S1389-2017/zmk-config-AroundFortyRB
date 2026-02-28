# CLAUDE.md — AroundFortyRB ZMK Configuration

This file provides AI assistants with comprehensive context about the codebase structure, conventions, and development workflows for the **Around Forty RightBall** split keyboard firmware.

---

## Project Overview

**AroundFortyRB** is a wireless split keyboard with an integrated optical trackball on the right half. It runs [ZMK Firmware](https://zmk.dev) on a pair of **Seeeduino XIAO BLE** (Nordic nRF52840) MCUs.

Key features:
- 42-key split layout (left + right halves connected via BLE)
- PMW3610 optical trackball sensor on the right half
- 4 firmware layers (QWERTY, Numbers/Symbols, Function/Mouse, Settings)
- RGB LED status indicators
- ZMK Studio support for live keymap editing
- Hardware CAD files included (PCB, case, edge cuts)

---

## Repository Structure

```
zmk-config-AroundFortyRB/
├── .github/workflows/build.yml     # GitHub Actions CI — triggers ZMK firmware builds
├── build.yaml                      # Build matrix: board/shield combos built by CI
├── config/
│   ├── west.yml                    # West manifest: ZMK + custom driver dependencies
│   ├── AroundForty-RB.json         # Physical layout for Keyboard Layout Editor
│   ├── AroundForty-RB.keymap       # ALL key bindings, layers, combos, macros
│   └── boards/shields/AroundForty-RB/
│       ├── AroundForty-RB.dtsi         # Shared device tree (matrix, KSCAN, physical layout)
│       ├── AroundForty-RB.zmk.yml      # Shield metadata (id, requires, features)
│       ├── AroundForty-RB_L.conf       # Left half Kconfig options
│       ├── AroundForty-RB_L.overlay    # Left half GPIO/device tree overlay
│       ├── AroundForty-RB_R.conf       # Right half Kconfig options (trackball, Studio)
│       ├── AroundForty-RB_R.overlay    # Right half GPIO/SPI/trackball overlay
│       ├── Kconfig.defconfig           # Default Kconfig values per shield variant
│       └── Kconfig.shield              # Shield-specific Kconfig symbol definitions
├── zephyr/module.yml               # Zephyr module root marker
└── Deta/                           # Hardware design files (NOT firmware)
    ├── V1/
    │   ├── CASE/v1/                # Original case (MCU bottom-mounted)
    │   ├── CASE/v2/                # Revised gasket-mount case (no screws/spacers)
    │   ├── EdgeCuts/               # PCB outline DXF files
    │   ├── Firmware/               # Pre-compiled .uf2 binaries (L, R, settings_reset)
    │   └── PCB/                    # 3D PCB models (STEP files)
    └── V2/
        ├── CASE/                   # MCU top-mounted case variant
        └── EdgeCuts/               # V2 PCB outlines (L same as V1)
```

---

## File Roles and Conventions

### File Naming

| Pattern | Meaning |
|---|---|
| `AroundForty-RB` | Shared/base identifier used for the whole keyboard |
| `AroundForty-RB_L` | Left half variant |
| `AroundForty-RB_R` | Right half variant (includes trackball) |
| `*.dtsi` | Device tree shared includes (used by both halves) |
| `*.overlay` | Per-variant device tree overlays |
| `*.conf` | Kconfig options (compiled into firmware) |
| `*.keymap` | ZMK key binding and layer definitions |
| `*.zmk.yml` | Shield metadata consumed by the ZMK build system |

### The Split Between Files

- **`.dtsi`** — hardware facts shared by both halves (matrix dimensions, row GPIOs, physical key positions)
- **`.overlay`** — per-half differences (column GPIOs, SPI/trackball devices on right)
- **`.conf`** — feature flags and driver parameters (both halves share some; right half has all trackball settings)
- **`.keymap`** — purely logical (layers, bindings, combos, macros); no hardware knowledge

---

## Hardware Details

### MCU

**Seeeduino XIAO BLE** (Nordic nRF52840)

GPIO aliases used in overlays:
- `xiao_d` — XIAO digital header pins (D0–D10)
- `gpio0` — raw nRF52840 GPIO port 0

### Matrix Wiring

| Direction | Pins |
|---|---|
| Rows (shared both halves) | `xiao_d 1`, `xiao_d 2`, `xiao_d 3`, `xiao_d 6` |
| Left cols | `xiao_d 10`, `xiao_d 9`, `xiao_d 8`, `xiao_d 7`, `gpio0 10`, `gpio0 9` |
| Right cols | `xiao_d 10`, `xiao_d 9`, `xiao_d 8`, `xiao_d 7`, `gpio0 10` (5 cols; trackball occupies one position) |

- Diode direction: **col2row**
- Right half uses `col-offset = <6>` so its 5 columns map to matrix columns 6–10

### Trackball (Right Half Only)

| Setting | Value |
|---|---|
| Sensor | PMW3610 optical sensor |
| Interface | SPI at 2 MHz |
| SCK | `gpio0 5` |
| MOSI/MISO | `gpio0 4` |
| CS | `gpio0 9` |
| IRQ | `gpio0 2` |
| CPI | 400 |
| Orientation | 180° rotation, X and Y axes inverted |
| Scroll layers | Layer 3 (Settings) |
| Polling | 125 Hz (software) |
| Auto-mouse timeout | 700 ms |

### Split Roles

- **Right half = Central** (`ZMK_SPLIT_ROLE_CENTRAL=y`) — connects to host via BLE
- **Left half = Peripheral** — connects to right half via BLE

---

## Keymap Architecture

File: `config/AroundForty-RB.keymap`

### Layers

| # | Name | Purpose |
|---|---|---|
| 0 | `default_layer` | QWERTY base + mouse buttons (MB1/MB2) |
| 1 | `NumberSymbol` | Numbers, symbols, Ctrl shortcuts (Z/X/C/V/A) |
| 2 | `FunctionMouse` | F1–F12, arrow navigation, mouse scroll |
| 3 | `Setting` | BT pairing (0–4), BT clear, bootloader, Studio unlock |

### Layer Access

| Key | Tap | Hold |
|---|---|---|
| Space | Space | Momentary Layer 1 |
| Enter | Enter | Momentary Layer 2 |
| Delete | Delete | Momentary Layer 3 |
| Backspace | Backspace | Mouse Button 2 |
| Z | Z | Left Control |
| Tab | Tab | Left Shift |

The custom `lt_to_layer_0` behavior wraps `&lt`: tap sends `&to 0` (return to base layer), hold activates target layer.

### Combos

| Combo | Keys (positions) | Output |
|---|---|---|
| LANG1 | 30 + 31 | LANG1 |
| LANG2 | 32 + 33 | LANG2 |
| ESC | 1 + 2 | Escape |
| TAB | 13 + 14 | Tab |
| NumLock | 28 + 29 | Num Lock |
| CAPS | 10 + 11 | Caps Lock |
| Left Ctrl | 10 + 11 | Left Control |

### Sensor (Rotary Encoder)

`scroll_up_down` behavior: encoder rotation sends mouse scroll up/down. Layer-specific (used in Layer 2 for navigation and Layer 3 for settings).

### ZMK Binding Abbreviations

```
&kp     — key press
&mt     — mod-tap (modifier held / key tapped)
&mo     — momentary layer
&to     — go to layer (stays until changed)
&lt     — layer-tap
&mkp    — mouse key press (MB1, MB2, ...)
&msc    — mouse scroll
&bt     — Bluetooth control (BT_SEL, BT_CLR, BT_DISC)
&bootloader — enter bootloader/DFU mode
&studio_unlock — unlock ZMK Studio
```

---

## Build System

### Dependencies (west.yml)

```yaml
zmkfirmware/zmk          # Core ZMK firmware (main branch)
razilyis/zmk-pmw3610-driver   # PMW3610 trackball sensor driver
caksoylar/zmk-rgbled-widget   # RGB LED status widget
```

Never manually edit west.yml to point at forks without testing the full build. Dependency updates require regenerating lock files via `west update`.

### Build Matrix (build.yaml)

Three firmware images are built:

| Board | Shield | Extra CMake args | Purpose |
|---|---|---|---|
| `seeeduino_xiao_ble` | `AroundForty-RB_R rgbled_adapter` | `--snippet zmk-usb-logging` | Right half (debug) |
| `seeeduino_xiao_ble` | `AroundForty-RB_L rgbled_adapter` | — | Left half |
| `seeeduino_xiao_ble` | `settings_reset` | — | Reset BT pairing |

Output artifacts are `.uf2` files flashed via UF2 bootloader (drag-and-drop to XIAO in bootloader mode).

### CI/CD

File: `.github/workflows/build.yml`

Triggers: `push`, `pull_request`, `workflow_dispatch`

Uses the upstream reusable workflow: `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`

No local build setup is needed — GitHub Actions handles everything. Pre-compiled binaries for the current hardware revision are stored in `Deta/V1/Firmware/`.

---

## Development Workflows

### Modifying the Keymap

1. Edit `config/AroundForty-RB.keymap`
2. Commit and push — CI builds new firmware automatically
3. Download the `.uf2` artifact from the GitHub Actions run
4. Put XIAO BLE into bootloader mode (double-tap reset button)
5. Drag-and-drop the `.uf2` onto the USB mass storage device

### Modifying Hardware Behavior (driver/config)

- For trackball tuning: edit `config/boards/shields/AroundForty-RB/AroundForty-RB_R.conf`
- For left-half features: edit `AroundForty-RB_L.conf`
- For GPIO/wiring changes: edit `AroundForty-RB_L.overlay` or `AroundForty-RB_R.overlay`
- For matrix or physical layout: edit `AroundForty-RB.dtsi`

### Flashing Firmware

Three images are required for a full setup:

1. Flash `AroundForty-RB_L rgbled_adapter-seeeduino_xiao_ble-zmk.uf2` to the left half
2. Flash `AroundForty-RB_R rgbled_adapter-seeeduino_xiao_ble-zmk.uf2` to the right half
3. If BT pairing is broken, flash `settings_reset-seeeduino_xiao_ble-zmk.uf2` to both halves to clear BT state, then re-flash normal firmware

### Adding New Layers

1. Add a new layer block in `AroundForty-RB.keymap` inside the `keymap { ... }` node
2. Add a layer-tap or momentary binding on an existing key to access it
3. Ensure the layer index is consistent across all `&lt`, `&mo`, `&to` references
4. ZMK layers are zero-indexed and must be contiguous

### Adding Combos

Add inside `combos { ... }` in the keymap:

```dts
combo_name {
    timeout-ms = <50>;
    key-positions = <X Y>;
    bindings = <&kp KEY>;
};
```

Key positions follow the matrix transform order defined in `AroundForty-RB.dtsi`.

---

## Key Constraints and Gotchas

- **Right half is always central.** Do not change `ZMK_SPLIT_ROLE_CENTRAL` to the left half without redesigning the pairing and BT configuration.
- **`col-offset = <6>`** must stay on the right overlay — it maps the physical right-half columns into the correct logical matrix positions.
- **The `rgbled_adapter` shield** must be listed alongside the main shield in `build.yaml` for RGB LED support to compile correctly.
- **`CONFIG_NFCT_PINS_AS_GPIOS=y`** is required on both halves because the nRF52840's NFC pins are repurposed as regular GPIO for the matrix.
- **`CONFIG_ZMK_STUDIO=y`** is right-half only — Studio communicates over USB on the central (right) half.
- **PMW3610 SPI CS pin (`gpio0 9`)** overlaps with a column GPIO in the left overlay — this is intentional; the right overlay remaps the column set to only 5 pins.
- **Layer 3 activates trackball scroll mode** (`scroll-layers = <3>` in the overlay). In all other layers the trackball acts as a mouse pointer.
- Hardware files in `Deta/` are design artifacts (STL, STEP, DXF) and do not affect firmware compilation.

---

## Hardware Versions

| Version | MCU Position | Case Style | Notes |
|---|---|---|---|
| V1 | Bottom of PCB | Screwless gasket mount (v2 case) | Original production design |
| V2 | Top of PCB | Same gasket mount | Left PCB edge cut unchanged from V1 |

Current pre-built firmware in `Deta/V1/Firmware/` targets **V1 hardware**.

---

## External References

- [ZMK Firmware Docs](https://zmk.dev/docs)
- [ZMK Keymap Reference](https://zmk.dev/docs/keymaps)
- [ZMK Config Guide](https://zmk.dev/docs/user-setup)
- [PMW3610 Driver (razilyis)](https://github.com/razilyis/zmk-pmw3610-driver)
- [RGB LED Widget (caksoylar)](https://github.com/caksoylar/zmk-rgbled-widget)
- [Seeeduino XIAO BLE Pinout](https://wiki.seeedstudio.com/XIAO_BLE/)
- [ZMK Studio](https://zmk.dev/docs/features/studio)
