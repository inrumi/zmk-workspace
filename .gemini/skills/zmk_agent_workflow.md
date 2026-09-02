# ZMK Firmware Agent Workflow

## Project Context
- **Domain:** ZMK Firmware configuration for ergonomic keyboards (e.g., crossesV2, Glove80, Corne-ish Zen).
- **Core Files:** Hardware DTS, overlays, and board defconfigs reside in `config/boards/` and `config/boards/shields/`.
- **Keymaps & Behaviors:** Files in `config/*.keymap` and `config/*.dtsi` (like `base.keymap`, `combos.dtsi`, `trackball_autolayer.dtsi`) define layer bindings, combos, mod-morphs, and adaptive keys.

## Building (No Testing)
- **Environment:** The repository operates inside a Nix environment managed with direnv.
- **Build Command:** Always use `direnv exec . just build <target>` (e.g., `direnv exec . just build crosses_v2_right`, `direnv exec . just build crosses_v2_dongle`). This utilizes the heavily cached setup and avoids rebuilding the Nix ecosystem.
- **Testing:** **DO NOT** attempt to run, flash, or test the firmware yourself. The USER manually flashes the compiled `.uf2` binaries onto the physical hardware.

## Zephyr / West Modules (WARNING: DO NOT PATCH DIRECTLY)
- **`modules/` is like `node_modules`:** External modules (e.g., `zmk-trackball-config`, `zmk-input-processor-*`, `zmk-adaptive-key`, `zmk-helpers`) are checked out under `modules/` via `config/west.yml`.
- **Ephemeral Code:** Direct changes inside `modules/` compile locally but **WILL BE OVERWRITTEN/DELETED** on CI or `west update`.
- **Fixing Module Bugs using github CLI:**
  1. **Fork:** Use `gh repo fork <org>/<repo> --clone=false` inside the target `modules/` directory.
  2. **Push:** Add the fork as a remote (`git remote add <user> git@github.com:<user>/<repo>.git`), commit the local `modules/` changes, and push it up (`git push -u <user> HEAD:main`).
  3. **Pin Reference:** Grab the new commit SHA (`git rev-parse HEAD`), and update `remote` and `revision` strings in `config/west.yml` to point to the newly pushed fork.
  4. **Commit Workspace:** Finally, commit and push the `config/west.yml` changes in the main workspace repo.

## Trackball Input Processors & Scrolling Architecture
- **Pipeline Order Matters:** For smooth and controllable trackball scrolling, always apply transformations and mappers in this order:
  1. `&zip_xy_transform (...)`: Hardware sensor orientation/inversion.
  2. `&zip_xy_to_scroll_mapper`: Map XY coordinates directly to wheel/hwheel events.
  3. `&zip_scroll_scaler <mult> <div>`: Scale the wheel events with remainder tracking. (Note: Scaling *before* the mapper using `&zip_xy_scaler` results in chunky, stepped/laggy scrolling because sensor counts below the divisor produce 0-ticks).
  4. `&zip_report_rate_limit <ms>`: Limit report pacing (e.g. 16ms) to prevent overflowing BLE HID buffers.
- **Scaler Tuning:**
  - `1 8`: Very responsive / fast (can feel too rapid on 600+ DPI optical trackballs).
  - `1 16`: Balanced, smooth, and controllable.

## PAW32XX Sensor Properties & Resolution Set
- **PAW3222 Hardware Specs:** The PAW3222 sensor operates in distinct hardware steps of 38 CPI, with boundaries from 608 to 4826 CPI.
- **DTS Tooling:** The `min-cpi`, `max-cpi`, and `step-cpi` properties in bindings (e.g. `zmk,behavior-paw32xx-res-set`) exist so that external UI configurator apps (like the web app or generic slider widgets) know the actual device boundaries and can increment/decrement sliders cleanly.
- **Boot Timing (load_delay):** When persisting CPI using the settings subsystem, writing SPI registers synchronously at early boot can fail due to power rail/USB/BLE churn. Using `load_delay = <1000>;` delays the SPI register commit by 1s.
- **Central Dongle Safety:** In dongle configurations, the central MCU doesn't have a physical SPI sensor attached. The behaviors that load settings MUST have `sensor_device` flagged as `required: false` in the `.yaml` and MUST null-check the device pointer in `.c` code (e.g. `DEVICE_DT_GET_OR_NULL`) before attempting SPI transactions, otherwise Zephyr will panic/crash.

## Adaptive Keys (`zmk-adaptive-key`)
- **Trigger Limits:** Each child trigger node inside `zmk,behavior-adaptive-key` allocates a static buffer sized by `CONFIG_ZMK_ADAPTIVE_KEY_MAX_TRIGGER_CONDITIONS` (default: 32).
- **Multiple Sub-blocks:** When matching many keys (e.g., A-Z plus programming punctuation and brackets), split them across multiple child trigger nodes (e.g. `repeat_alpha`, `repeat_sym_unshifted`, `repeat_sym_shifted`). ZMK evaluates child nodes sequentially until a match is found.
- **Key Repeat with Modifiers:** When binding to `&key_repeat`, set `strict-modifiers;` if the trigger should only fire when the exact modifier state of the trigger key matches the prior keypress.

## Split Hardware Architecture (Dongle vs Dongle-less)
- **Central vs Peripheral Roles:**
  - In a Dongle build: The dongle is Central; both left and right keyboard halves are Peripherals.
  - In a Dongle-less build: One half (typically Right) is Central; the other is Peripheral.
- **Input Split & Processing Locality:**
  - ZMK's `input-split` beams **RAW, unprocessed** sensor deltas from Peripherals to the Central.
  - Input processors (e.g., `zmk,input-processor-pipeline-switch`, `input-listener` overrides) **MUST attach on the Central**. Peripherals do not run input pipelines.
- **Pipeline Switching State Routing:**
  - Firmware behaviors or C modules switching trackball pipelines (via `drive()`) MUST target `ZMK_POSITION_STATE_CHANGE_SOURCE_LOCAL`. If routed over BLE to a peripheral, the event is lost.
- **Compiler Safety & Assertions:**
  - If a base `.dts` enables a `zmk,input-split` node, ZMK asserts that a Peripheral executing that DTS must have a local physical `device` bound to it.
  - When a `.dts` is shared between Central and Peripheral builds, the Peripheral overlay MUST explicitly set `status = "disabled";` for split listener/device nodes of the opposite half to prevent compile-time assertions.
- **Kconfig Constraints:**
  - `CONFIG_ZMK_USB=n` must be set for pure split peripheral configurations (peripherals do not manage USB stacks).
  - Register custom vendor prefixes in `config/dts/bindings/vendor-prefixes.txt` to eliminate Zephyr device tree compilation warnings.
