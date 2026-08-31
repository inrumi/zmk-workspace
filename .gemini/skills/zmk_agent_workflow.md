# ZMK Firmware Agent Workflow

## Project Context
- **Domain:** ZMK Firmware configuration for ergonomic keyboards (e.g., crossesV2).
- **Core Files:** Most logic changes happen in `config/boards/` (DTS, overlays, confs).
- **Keymaps:** Files in `config/*.keymap` and `config/*.dtsi` (like `base.keymap`, `combos.dtsi`) are purely for keymap layouts and do not impact core firmware logic or peripheral hardware interaction.

## Building (No Testing)
- **Environment:** The repository operates inside a Nix environment.
- **Build Command:** Always use `direnv exec . just build <target>` (e.g., `direnv exec . just build crosses_v2_dongle`). This utilizes the heavily cached setup and prevents rebuilding the nix ecosystem unnecessarily.
- **Testing:** **DO NOT** attempt to run, flash, or test the firmware yourself. The USER will manually deploy the compiled `zmk.uf2` to the physical hardware and verify.

## Zephyr / West Modules (WARNING: DO NOT PATCH DIRECTLY)
- **`modules/` is like `node_modules`:** External modules (e.g., `zmk-trackball-config`, `zmk-input-processor-*`) are checked out locally under `modules/` via Zephyr's `west` tooling (driven by `config/west.yml`). 
- **Ephemeral Code:** Any direct modifications you make to C code inside the `modules/` directory will compile successfully in the user's local `direnv` workspace but **WILL DEFINITELY be overwritten/deleted** on a fresh clone or a GitHub Actions CI build (`west update`).
- **Fixing Upstream C Bugs:** If you find a hardcoded logic bug in an upstream C module that cannot be bypassed via DTS overrides, **do not just patch the C file silently**. Instead, ask the user if they'd like to convert it to a local module. If approved:
  1. Option A (Fork): Commit the fix to a GitHub fork, push it, and update the module's `remote` and `revision` in `config/west.yml`.
  2. Option B (Local Directory): Move the module's folder into the `config/` directory (e.g., `config/modules/<module-name>`) and update `config/west.yml` to remove the `remote` URL and use a local `path`.

## Debugging Hardware & Splitting (Dongle vs Non-Dongle)
When debugging trackballs, sensors, or cross-split communication:
- **ZMK Locality:** Pay close attention to ZMK Behavior Locality (e.g., `BEHAVIOR_LOCALITY_EVENT_SOURCE`). Central vs Peripheral splitting dictates *where* a behavior executes.
- **Dongle Layouts:** In a dongle setup, the Dongle itself operates as the Central, and both the Left and Right keyboard halves operate as Peripherals. In Dongle-less, the Right (or Left) half acts as Central.
- **Input Split & Pipelines:** ZMK's `input-split` system natively beams **RAW, unprocessed** sensor inputs directly to the Central. Therefore, input processors (like `zmk,input-processor-pipeline-switch`) MUST logically attach to the central's `zmk,input-listener` nodes. **Peripherals do not run trackball pipelines!**
- **State Change Routing:** Because pipelines execute entirely on the Central, any firmware logic switching those pipelines (e.g., via C modules calling `drive()`) MUST target `ZMK_POSITION_STATE_CHANGE_SOURCE_LOCAL` regardless of where the physical trackball physically resides. If you route a pipeline state change to a remote peripheral over BLE, it will vanish, breaking mode toggles entirely.
- **Compiler Safety (Peripheral vs Central):** If a base `.dts` file enables a `zmk,input-split` node (e.g., `&trackball_peripheral_split { status = "okay"; }`), ZMK's compiler rigidly asserts that any *Peripheral* executing that DTS must also bind a physical `device` sensor to it. If the base `.dts` is shared by both a Central build and a Peripheral build, the Peripheral's `.overlay` MUST explicitly `status = "disabled";` the split nodes of the opposite half, otherwise the compiler will crash!

## Workflow Rules
1. **Analyze First:** Check `config/boards/<board>` before looking anywhere else for firmware behavior.
2. **Read Macros/Module Source:** If a feature isn't working, immediately look at `modules/...` to read the underlying ZMK module C/C++ source. Hardware issues are usually ZMK locality mismatch or missing `dts` phandle properties.
3. **Draft Fixes via DTS:** Always attempt to fix logic via `.overlay` and `.dts` files first.
4. **Compile:** Always compile and verify the build exits smoothly using `direnv exec . just build <target>` before handing it back to the user.
5. **No Automatic Commits:** Do not commit or push. The user handles version control.
