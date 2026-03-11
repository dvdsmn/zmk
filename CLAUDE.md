# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Keychron fork of [ZMK Firmware](https://zmk.dev/), an open source keyboard firmware built on Zephyr RTOS. This `keychron_bpro` branch adds support for Keychron B Pro series keyboards (B1, B2, B6) with NRF52840, including USB, BLE (5 profiles), and a proprietary 2.4GHz wireless protocol.

## Working Style

Be critical of instructions. If a request is wrong, suboptimal, risky, unsafe, or a bad idea, say so and explain why before (or instead of) complying. Suggest a better approach when one exists. Do not silently go along with a flawed premise.

## First-Time Setup (after cloning)

The devcontainer provides the toolchain, but three manual steps are required:

```bash
# 1. Initialize the west workspace (run from repo root)
west init -l app

# 2. Fetch all Zephyr dependencies
west update

# 3. Apply the required Zephyr patch (24G + BLE coexistence fix)
cd zephyr
git am ../0001-esb-nrf-fix.patch
```

The patch is a hard prerequisite — without it, builds with `CONFIG_ZMK_NRF_24G_ECB=y` will malfunction at runtime. It must be re-applied if the `zephyr/` subproject is ever reset or updated.

## Build Commands

```bash
cd app

# Build for a specific keyboard model and layout
west build -b keychron -p -- -DSHIELD=keychron_b1_us
west build -b keychron -p -- -DSHIELD=keychron_b1_uk
west build -b keychron -p -- -DSHIELD=keychron_b1_jis
west build -b keychron -p -- -DSHIELD=keychron_b2_us
west build -b keychron -p -- -DSHIELD=keychron_b6_us

# -p flag does a pristine (clean) build
# Output: build/zephyr/zmk.uf2
```

Available shields: `keychron_b{1,2,6}_{us,uk,jis}` and `keychron_b2_kr`.

## Running Tests

Tests run on `native_posix_64` (no hardware needed):

```bash
cd app

# Run all tests (parallel, default 4 jobs)
./run-test.sh

# Run with more parallelism
./run-test.sh -P 8

# Run a single test
./run-test.sh tests/hold-tap/balanced/1-key-hold-tap
```

Test cases are in `app/tests/`. Each test has a `keycode_events.snapshot` file that is compared against actual output.

## Architecture

### Data Flow

Physical key press → kscan (matrix/GPIO) → position event → keymap layer resolution → behavior → HID report → endpoint (USB/BLE/24G)

### Key Source Files

| File | Purpose |
|------|---------|
| `app/src/keymap.c` | Layer state management, key binding resolution |
| `app/src/ble.c` | BLE stack, multi-profile management |
| `app/src/hid.c` | HID report building |
| `app/src/endpoints.c` | Output routing (USB/BLE/24G) |
| `app/src/main.c` | App entry, battery shutdown, transport init |
| `app/src/kscan.c` | Key matrix scanning |
| `app/src/behaviors/` | 23+ behavior implementations |
| `app/src/24G/` | Proprietary 2.4GHz protocol (precompiled `.a` library) |

### Board vs Shield

ZMK separates hardware into two layers:
- **Board** (`app/boards/arm/keychron/`): The NRF52840 mainboard — GPIO, battery, LEDs, USB/BLE config
- **Shield** (`app/boards/shields/keychron/`): The keyboard PCB overlay — matrix mapping, keymap, per-model config

The shield selects the key matrix layout and default keymap; the board defines the MCU and peripherals.

### Keyboard Models

Each model has regional variants (US/UK/JIS/KR) under `app/boards/shields/keychron/`:
- `b1/{us,uk,jis}/` — B1 Pro
- `b2/{us,uk,kr,jis}/` — B2 Pro
- `b6/{us,uk,jis}/` — B6 Pro

Each variant contains: `.overlay` (matrix/GPIO mappings), `.keymap` (default keymap), `.conf` (overrides).

### Behaviors

Behaviors are DTS-configured drivers in `app/src/behaviors/` that handle key actions: hold-tap, sticky keys, mod-morph, macros, layer switching, combos, RGB/backlight control, etc. They are referenced in `.keymap` files via `&behavior_name`.

### 24G Wireless

The 2.4GHz protocol uses `lib_nrf_esb_24G.a` (precompiled Nordic ESB library). Enabled via `CONFIG_ZMK_NRF_24G_ECB=y` in `keychron_defconfig`. The Zephyr patch (`0001-esb-nrf-fix.patch`) must be applied to the `zephyr/` subproject after `west update` for correct NRF radio ISR handling when BLE and 24G are both active.
