---
name: zmk-agent-workflow
description: >-
  Guidelines, build/sync commands, git workflow rules, and architectural rules for ZMK firmware development in this repository.
  Use when configuring keymaps, modifying board DTS/overlays, building firmware targets with Nix/direnv/just,
  managing west modules, tuning trackball input processors, configuring PAW32xx sensors, handling git commits/branches,
  or working with split dongle/peripheral setups.
---

# ZMK Firmware Agent Workflow

## Git Branching & Commit Policy
- **NEVER Commit to `main`:** We NEVER commit directly to `main`. Always check the current working branch (`git branch --show-current`) before performing any git operations.
- **Commit Only When Explicitly Requested:** Even when on a feature branch (not `main`), do NOT create commits automatically. Only commit if the user explicitly asked or authorized you to commit.
- **Keep Changes Visible:** Keep all changes uncommitted in the working tree so they remain clearly visible for the user to review.
- **No Deploy / Flash:** Do NOT attempt to flash hardware or run deployment tasks. The user tests all physical firmware flashes independently.
- **Pull Requests (PRs):** NEVER open Pull Requests directly against upstream or other people's repositories unless explicitly instructed. ALWAYS open the PR against the user's own fork (e.g., `gh pr create --repo <user>/<repo> ...`).

## Building & Workspace Toolchain (`Justfile` via Nix/direnv)
The repository operates inside a Nix dev environment managed with direnv. Always execute `just` commands wrapped in `direnv exec .`:

| Command | Purpose |
| --- | --- |
| `direnv exec . just list` | List all available build targets from `build.yaml` |
| `direnv exec . just build <target>` | Build firmware for a target (e.g. `direnv exec . just build crosses_v2_dongle`, `direnv exec . just build all`). Compiled firmware lands in `firmware/`. |
| `direnv exec . just sync` | Synchronize west workspace after manifest changes in `config/west.yml` (DO NOT run `west update` directly). |
| `direnv exec . just clean` | Remove `.build` directory and `firmware/` artifacts. |
| `direnv exec . just draw` | Regenerate keymap SVG diagrams (`draw/base.svg`, `draw/overview.svg`). |
| `direnv exec . just format <paths>` | Format devicetree DTS files using `dts-format`. |
| `direnv exec . just bump-west` | Bump pinned module revisions in `config/west.yml` via pin-west and sync. |
| `direnv exec . just bump-nix` | Update `flake.lock` Nix toolchain. |

## Project Context
- **Domain:** ZMK Firmware configuration for ergonomic keyboards (e.g., crossesV2, Glove80, Corne-ish Zen).
- **Core Files:** Hardware DTS, overlays, and board defconfigs conventionally reside in `config/boards/` and `config/boards/shields/`, though shifting them into dedicated `modules/boards/` repositories is the modern ZMK best practice.
- **Keymaps & Behaviors:** Files in `config/*.keymap` and `config/*.dtsi` (like `base.keymap`, `combos.dtsi`, `trackball_autolayer.dtsi`) define layer bindings, combos, mod-morphs, and adaptive keys.

## Zephyr / West Modules & Extensibility
- **`modules/` is like `node_modules`:** External modules (e.g., `zmk-trackball-config`, `zmk-input-processor-*`, `zmk-adaptive-key`, `zmk-helpers`) are checked out under `modules/` via `config/west.yml`.
- **Moving to Modules is Encouraged:** Moving custom behaviors, sensor drivers, or entire board definitions into dedicated modules under `modules/zmk/` or `modules/boards/` is **allowed and healthy**, provided they are backed by a forked/synced repository defined in `config/west.yml`. This resolves modern ZMK CMake deprecation warnings about `config/boards`.
- **Ephemeral Code:** Direct changes inside `modules/` compile locally but **WILL BE OVERWRITTEN/DELETED** on CI or `direnv exec . just sync`.
- **Adding Custom Modules to `west.yml` Rules:**
  When adding a new module or fork, you MUST follow this explicit format in `config/west.yml`:
  ```yaml
      - name: <module-name>
        remote: <remote-name> # Note: 'remote' can be skipped if pulling from urob's repositories
        path: modules/<logical-group>/<module-name>
        revision: <full-commit-sha> # <branch> (YYYY-MM-DD)
  ```
  *   **`path`:** You must explicitly define `path:` pointing into the `modules/` directory (e.g. `modules/boards/...` or `modules/zmk/...`). If omitted, `west` will incorrectly dump the module into the root workspace folder.
  *   **`revision` & Comments:** You must pin specific full commit SHAs for reproducibility instead of branches. You MUST append an inline comment with the targeted branch and the date of the commit (`# main (YYYY-MM-DD)`) so humans can track the branch and age of the pinned code without checking GitHub.
  *   **`remote`:** Explicitly declare the custom remote name unless pointing to `urob` repositories (as `urob` acts as the manifest default remote and can be omitted).
- **Fixing Module Bugs using github CLI:**
  1. **Fork:** Use `gh repo fork <org>/<repo> --clone=false` inside the target `modules/` directory.
  2. **Push:** Add the fork as a remote (`git remote add <user> git@github.com:<user>/<repo>.git`), commit the local `modules/` changes, and push it up (`git push -u <user> <branch>`).
  3. **Open PR:** If instructed to open a PR for the module, ALWAYS use the user's fork as the target repository (`gh pr create --repo <user>/<repo> --head <user>:<branch> ...`). `[CRITICAL: DO NOT OPEN PRs AGAINST UPSTREAM REPOS]`
  4. **Pin Reference:** Grab the new commit SHA (`git rev-parse HEAD`), and update `remote` and `revision` strings in `config/west.yml` to point to the newly pushed fork following the format above.
  5. **Sync Space:** Run `direnv exec . just sync` to lock in the workspace.
  6. **Commit Workspace:** Finally, commit and push the `config/west.yml` changes in the main workspace repo.

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

## Hardware Target Definition in `build.yaml`
- **Use ZMK Board Variants (`//zmk`):** When adding targets to `build.yaml`, always use the `//zmk` suffix for boards that have ZMK-specific overrides (e.g., use `xiao_ble//zmk` instead of `xiao_ble`). 
  - **Reasoning:** Upstream Zephyr board definitions often rigidly lock pins for hardware features (like UART on D6 or SPI on D8). If you compile against the pure Zephyr definition (`xiao_ble`), those pins will silently fail to work for keyboard matrix scanning (causing entire dead rows or columns), and simply attempting to `status = "disabled";` the serial nodes in your `.overlay` will often **not** fix it. The `//zmk` out-of-tree variant provides the properly neutralized pin states required for keyboard matrices.

## CI Workflows & PR Required Checks
- **Dedicated PR Checks Workflow (`.github/workflows/checks.yml`):**
  - Pull request validation is consolidated into `.github/workflows/checks.yml` running on a single `macos-15` (aarch64-darwin) runner to conserve GitHub Actions runner minutes.
  - **Adding a New Keyboard / Central Target Checklist:**
    Whenever a new keyboard is introduced to the repository (in `build.yaml`, `config/`, or `modules/boards/`):
    1. **Identify the Central Target:** Identify which part acts as Central (a dedicated dongle e.g. `<keyboard>_dongle`, or in dongle-less setups, the designated central half—frequently Right e.g. `<keyboard>_right` or Left). The Central target is critical because it compiles the central BLE coordinator stack, all shared keymaps, behaviors, display widgets, and input processors.
    2. **Add Check Step to `.github/workflows/checks.yml`:** Add a build step for the new central target inside the `check-central-boards` job:
       ```yaml
       - name: Build <KeyboardName> Central
         run: nix develop --command just build <new_central_target>
       ```
    3. **Validate Locally First:** Run `direnv exec . just build <new_central_target>` locally to verify that it builds with zero errors before committing.
- **On-Demand Release Workflows:**
  - `.github/workflows/build-nix.yml`, `.github/workflows/build.yml`, and `.github/workflows/test-build-env.yml` are set to `workflow_dispatch` (manual execution only). They should NEVER be configured with automatic `push:` triggers to prevent matrix bloat and duplicate check runs.

## Verification & Build Validation
- **Always Validate with a Build:** After making any firmware, keymap, overlay, or DTS changes, ALWAYS verify that compilation succeeds by running:
  ```bash
  direnv exec . just build <target>
  ```
  Confirm there are zero compilation errors before finishing your task.

## Upstream Fork Synchronization & Unused Board Pruning
- **Synchronizing from Upstream (`urob/zmk-config`):**
  - Fetch upstream updates: `git fetch upstream`.
  - Inspect changes: `git log --oneline <merge-base>..upstream/main`. Upstream primarily bumps pinned west modules (`zmk`, `zmk-helpers`, `zmk-tri-state`, `zmk-unicode`) and workflow definitions.
  - Review `config/west.yml` carefully during merges to keep custom board/sensor modules (`crosses-v2-zmk-firmware`, `keyboard-delta-omega`, trackball input processors) intact.
  - Run `direnv exec . just sync` to update the West workspace after updating `west.yml`.
- **Pruning Unused Boards (`build.yaml` and `config/`):**
  - Upstream default boards that are not owned or built (e.g. `corneish_zen`, `glove80`, `planck_rev6`) are pruned from `build.yaml` to eliminate unnecessary GitHub Actions CI matrix runs (saving 10–15 min per push) and keep `direnv exec . just list` concise.
  - The corresponding `.keymap` and `.conf` adapter files in `config/` can be safely removed.
  - **Handling Future Merge Conflicts (modify/delete):** If upstream ever commits an update to a deleted board file (e.g. `config/corneish_zen.keymap`), git merge will report a `CONFLICT (modify/delete)`.
    - **Resolution:** Simply execute `git rm config/<board>.keymap config/<board>.conf` and complete the merge. We do not maintain or flash those hardware targets.
