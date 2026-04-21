# Cosmos ZMK Config

Split keyboard firmware for ZMK. Hardware: 2x Nice Nano (left/right) + Xiao BLE (dongle), PMW3610 trackball on right side, EC11 encoders on both sides.

## Building

```
west build -p always -d build/left    -b nice_nano -- -DZMK_CONFIG=config -DZMK_EXTRA_MODULES=-DZMK_SHIELD=cosmos_left
west build -p always -d build/right   -b nice_nano -- -DZMK_CONFIG=config -DZMK_EXTRA_MODULES=-DZMK_SHIELD=cosmos_right
west build -p always -d build/dongle  -b xiao_ble_nrf52840 -- -DZMK_CONFIG=config -DZMK_EXTRA_MODULES=-DZMK_SHIELD=cosmos_dongle -DZMK_EXTRA_MODULES=-DZMK_SHIELD=prospector_adapter
west build -p always -d build/settings_reset -b nice_nano -- -DZMK_CONFIG=config -DZMK_EXTRA_MODULES=-DZMK_SHIELD=settings_reset
```

Or use `build.yaml` with `west build` (selected by shield name). CI uses `zmkfirmware/zmk/.github/workflows/build-user-config.yml`.

## Project structure

| Path | Purpose |
|---|---|
| `config/cosmos.keymap` | Keymap (9 layers: base, mouse, WASD, ESDF, sym, nav, SAIL, scroll) |
| `config/cosmos.conf` | Shared config (BT power, encoders) |
| `config/cosmos_right.conf` | Right-side-only config (SPI, trackball, rotate-plane) |
| `config/cosmos_dongle.conf` | Dongle-only config (central peripherals=2, no sleep) |
| `boards/shields/cosmos/cosmos_behaviors.dtsi` | Custom behaviors: `ht` (hold-tap), `atab` (tri-state alt-tab), `tdmedia` (tap-dance media), `shft_ret` (macro), `scroll_up_down`, `tb_scroll` |
| `boards/shields/cosmos/cosmos.dtsi` | Matrix transform, encoder definitions, physical layout |
| `boards/shields/cosmos/cosmos_left.overlay` | Left side: kscan GPIO matrix, left encoder wiring |
| `boards/shields/cosmos/cosmos_right.overlay` | Right side: kscan (note `gpio1` for some rows), right encoder, SPI + trackball input processor |
| `boards/shields/cosmos/cosmos_dongle.overlay` | Dongle: mock kscan, trackball split listener + rotate-plane input processor (20-deg rotation, 3/4 pointer scale, scroll layer 7 at 1/16) |
| `boards/shields/cosmos/split_input.dtsi` | Shared split input device & listener nodes (disabled by default) |
| `boards/shields/cosmos/pmw3610.dtsi` | PMW3610 trackball SPI device node + pinctrl (right side + dongle) |

## Architecture notes

- **Split architecture**: Right side is peripheral with trackball; dongle is central. `ZMK_SPLIT_ROLE_CENTRAL=y` only on `cosmos_dongle`.
- **Trackball split input**: `trackball_split_input` (split device) on right → `trackball_split_listener` on dongle. Dongle applies rotate-plane (20-deg) and scaling via input processors in `cosmos_dongle.overlay`.
- **Keymap includes both central and peripheral**: The `.keymap` is compiled for all builds. Split listener nodes are defined as `disabled` in shared `split_input.dtsi` so the keymap can reference them without undefined-reference errors on peripheral builds.
- **`cosmos_right.overlay` applies `col-offset = <6>`** to `default_transform` to account for the split gap.

## Keymap conventions

- `&ht` = custom hold-tap (tap-preferred, 200ms tapping term, 125ms quick-tap)
- `AC()` / `AS()` = autotyped macros using `&ht` (replaced old `&ac` auto-ctrl behavior)
- `&tdg` = tap-dance: tap→ESDF, hold→WASD (gaming layers)
- `&tdmedia` = tap-dance: tap×2→mute, tap→next, hold→previous
- `&tb_scroll` = hold-tap on trackball area key: hold→scroll layer, tap→momentary nav
- SAIL layer (6) = unicode symbols via `&uc` from urob/zmk-unicode module
- Layer 7 = scroll layer (maps trackball motion to scroll via `zip_xy_to_scroll_mapper`)

## External modules (west.yml)

| Module | Remote | Purpose |
|---|---|---|
| zmk | zmkfirmware | Core ZMK firmware |
| zmk-pmw3610-driver | badjeff | PMW3610 trackball sensor driver |
| prospector-zmk-module | carrefinho | Prospector dongle display (feat/new-status-screens) |
| zmk-input-processor-rotate-plane | efogdev | Rotate-plane input processor |
| zmk-tri-state | dhruvinsh | Tri-state behavior (alt-tab) |
| zmk-unicode | urob | Unicode key symbols |

## Gotchas

- **`efogdev` remote missing from west.yml remotes list** — the `zmk-input-processor-rotate-plane` project references remote `efogdev` but it is not defined. West will fail unless the remote is added.
- Right side kscan uses `&gpio1 7` and `&gpio1 2` for two row pins (nRF52840 has two GPIO banks). Left side uses only `&pro_micro`.
- Dongle must stay awake: `CONFIG_ZMK_SLEEP=n`. Do not enable sleep on dongle build.
- Trackball CPI = 600, report queue size = 1 (per @efog recommendation).
- Encoders are `disabled` by default in `cosmos.dtsi`; enabled per-side in left/right overlays.
- `build.yaml` builds `settings_reset` as a separate target — run this first after major config changes to clear persistent state.
