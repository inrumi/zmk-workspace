# Firmware Topologies & Flashing Guide

This document explains how build targets in [`build.yaml`](../build.yaml) map to compiled `.uf2` artifacts in `firmware/`, and which firmware files to flash for **Dongle-less** vs. **Dongle** setups.

---

## Core Concept: Central vs. Peripheral in ZMK

Every ZMK split keyboard requires **exactly one Central** device and **one or more Peripheral** devices:

- **Central:** Connects to the host computer (via USB or Bluetooth), runs the keymap, layers, combos, and trackball input processor pipelines (`zip_periph_ball`, `zip_central_ball`), and coordinates BLE traffic from peripherals.
- **Peripheral:** Scans its local key matrix and raw trackball sensor, forwarding raw events over internal BLE to the Central. A peripheral **cannot** talk to a host computer on its own (`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`).

> [!WARNING]
> If you flash **Peripheral** firmware to both the Left and Right halves without using a Dongle, **neither half is a Central**. Nothing will connect to your computer or process keypresses/trackball movement.

---

## Crosses V2: Which Firmware Do I Flash?

Crosses V2 supports two hardware topologies from the same codebase:

### Option A: Dongle-less Setup (2 Devices: Left + Right)

In a **dongle-less** setup, the **Right half is the Central** (`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y` by default in `crosses_v2_right`) and connects directly to your computer via USB or Bluetooth. The Left half connects to the Right half as a peripheral.

| Physical Device | Role | `just build` Target | Compiled Firmware File (`firmware/`) |
| :--- | :--- | :--- | :--- |
| **Left Half** | Peripheral | `just build crosses_v2_left` | `crosses_v2_left.uf2` |
| **Right Half** | **Central** | `just build crosses_v2_right` | **`crosses_v2_right.uf2`** |

* Do **not** flash `crosses_v2_peripheral_right-crosses_v2_right.uf2` in a dongle-less setup.

### Option B: Dongle Setup (3 Devices: Dongle + Left + Right)

In a **dongle** setup, the external `nice!nano` Dongle is the **Central**. Because the `crosses_v2_right` board definition defaults to being a Central, the `crosses_v2_peripheral_right` shield is layered on top of `crosses_v2_right` to demote the Right half to a Peripheral (`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=n`) and route its local trackball over BLE (`&trackball_central_split`) to the Dongle.

| Physical Device | Role | `just build` Target | Compiled Firmware File (`firmware/`) |
| :--- | :--- | :--- | :--- |
| **Dongle (`nice!nano`)** | **Central** | `just build crosses_v2_dongle` | `crosses_v2_dongle+dongle_display-nice_nano__zmk.uf2` |
| **Left Half** | Peripheral | `just build crosses_v2_left` | `crosses_v2_left.uf2` |
| **Right Half** | Peripheral | `just build crosses_v2_peripheral_right` | **`crosses_v2_peripheral_right-crosses_v2_right.uf2`** |

---

## Why `just build crosses_v2_right` Produces Two Files

The `just build <expr>` recipe filters `build.yaml` by substring match across board and shield names. Because both Right-half targets use `board: crosses_v2_right`:

```yaml
include:
  - board: crosses_v2_right
  - board: crosses_v2_right
    shield: crosses_v2_peripheral_right
```

Running `just build crosses_v2_right` builds **both** variants:

1. `firmware/crosses_v2_right.uf2` — **Dongle-less Right (Central)**
2. `firmware/crosses_v2_peripheral_right-crosses_v2_right.uf2` — **Dongle-paired Right (Peripheral)**

Always check the filename before copying the `.uf2` to the Right half's bootloader drive.

---

## Switching Between Dongle and Dongle-less

When switching a Crosses V2 pair between Dongle mode and Dongle-less mode:

1. **Reflash the Right half** with the matching firmware (`crosses_v2_right.uf2` for Dongle-less, or `crosses_v2_peripheral_right-crosses_v2_right.uf2` for Dongle).
2. **Left half firmware stays the same** (`crosses_v2_left.uf2` is always a Peripheral), **but** if BLE bonding fails after switching which device acts as Central (Right half vs. Dongle), clear BLE bonds (`&bt BT_CLR` or a settings reset) so the Left half pairs with the active Central's MAC address.

---

## Delta Omega: Firmware Reference

Delta Omega is configured as a **dongle-less** split keyboard on `xiao_ble//zmk`, where the **Left half is the Central**:

| Physical Device | Role | `just build` Target | Compiled Firmware File (`firmware/`) |
| :--- | :--- | :--- | :--- |
| **Left Half** | **Central** | `just build delta_omega_left` | `delta_omega_left-xiao_ble__zmk.uf2` |
| **Right Half** | Peripheral | `just build delta_omega_right` | `delta_omega_right-xiao_ble__zmk.uf2` |
