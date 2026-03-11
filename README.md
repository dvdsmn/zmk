# ZMK Firmware — Keychron B1 Pro UK (Personal Fork)

This is a personal fork of [Keychron's ZMK firmware](https://github.com/Keychron/zmk/tree/keychron_bpro) for the **B Pro series** keyboards (B1, B2, B6) with NRF52840. It adds support for USB, BLE (5 profiles), and Keychron's proprietary 2.4GHz wireless protocol on top of [ZMK Firmware](https://zmk.dev/).

**Customizations in this fork (B1 UK layout):**
- Top-right key (Mac layer): `GLOBE` — triggers macOS Globe shortcuts and emoji picker
- `Fn + Backspace` (Mac layer): `DEL`
- `Fn + -` (hold 3s, both Fn layers): enter UF2 bootloader for flashing

---

## Setup

Requires the [devcontainer](https://zmk.dev/docs/development/setup) or a manually installed Zephyr SDK + west toolchain.

```bash
git clone -b keychron_bpro https://github.com/dvdsmn/zmk.git
cd zmk

# 1. Initialize the west workspace
west init -l app

# 2. Fetch all Zephyr dependencies
west update

# 3. Apply the required Zephyr/ESB patch (BLE + 24G coexistence fix)
cd zephyr
git am ../0001-esb-nrf-fix.patch
```

> The patch is mandatory. Without it, 24G wireless (`CONFIG_ZMK_NRF_24G_ECB=y`) will malfunction at runtime. Re-apply if `zephyr/` is ever reset or updated.

---

## Build

```bash
cd app

# B1 Pro
west build -b keychron -p -- -DSHIELD=keychron_b1_us
west build -b keychron -p -- -DSHIELD=keychron_b1_uk
west build -b keychron -p -- -DSHIELD=keychron_b1_jis

# B2 Pro
west build -b keychron -p -- -DSHIELD=keychron_b2_us
west build -b keychron -p -- -DSHIELD=keychron_b2_uk
west build -b keychron -p -- -DSHIELD=keychron_b2_kr

# B6 Pro
west build -b keychron -p -- -DSHIELD=keychron_b6_us
```

Output: `build/zephyr/zmk.uf2`

---

## Flashing

**Entering UF2 bootloader mode:**

- **Software (this firmware):** Hold `Fn + -` for 3 seconds while USB is connected
- **Hardware:** Unplug USB → set switches to WIN + CABLE → hold the pinhole reset button → plug USB back in while holding → release button

The keyboard mounts as a USB drive. Copy `zmk.uf2` onto it. It reboots automatically.

---

## Notes

- [CLAUDE.md](CLAUDE.md) has detailed architecture notes and build tips
- The `lib_nrf_esb_24G.a` precompiled library means this fork cannot be cleanly merged with upstream ZMK
- GLOBE key (`0x029D`) works natively on macOS — if using Karabiner-Elements, add the Keychron to the **Ignore this device** list in the Devices tab
