# DYA Studio Integration Guide for ZMK Keyboards

> Verified working on KUKEY42 (seeeduino_xiao_ble, split keyboard with PMW3610 trackball + EC11 encoder)
> All features confirmed: Keymap editing, BLE management, Battery history, Settings RPC, Runtime input processor

This guide documents how to add [DYA Studio](https://studio.dya.cormoran.works/) support to an existing ZMK keyboard.

## What is DYA Studio?

[DYA Studio](https://studio.dya.cormoran.works/) is a browser-based keyboard management tool built by [cormoran](https://github.com/cormoran) that extends ZMK Studio with:

| Feature | Description |
|---------|-------------|
| Keymap Editor | ZMK Studio standard keymap editing |
| BLE Management | View/manage BLE connections and profiles |
| Battery History | Track battery levels over time with charts |
| Settings RPC | Configure idle/sleep timeouts at runtime |
| Runtime Input Processor | Adjust trackball speed, rotation, scroll settings without rebuilding |
| Split Event Relay | Synchronize settings between central and peripheral halves |

## Prerequisites

- Working ZMK split keyboard firmware
- Local ZMK build environment (west, Zephyr SDK)
- nRF52840-based board (e.g., seeeduino_xiao_ble)

---

## Quick Start (5 Steps)

### Step 1: west.yml - Add cormoran's ZMK Fork + Modules

Replace your ZMK project entry and add DYA Studio modules:

```yaml
manifest:
  defaults:
    revision: main
  remotes:
    - name: cormoran
      url-base: https://github.com/cormoran
    # Keep your existing remotes (trackball driver, LED widget, etc.)
  projects:
    # ZMK: cormoran fork (REPLACES standard zmkfirmware/zmk)
    - name: zmk
      remote: cormoran
      revision: v0.3-branch+dya
      import: app/west.yml

    # DYA Studio modules (all from cormoran)
    - name: zmk-module-ble-management
      remote: cormoran
    - name: zmk-module-battery-history
      remote: cormoran
    - name: zmk-module-settings-rpc
      remote: cormoran
    - name: zmk-module-runtime-input-processor
      remote: cormoran

    # Your existing modules...
  self:
    path: config
```

Then run:
```bash
west update
```

### Step 2: Central Side .conf

Add to your **central (USB-connected) side** .conf:

```ini
# ========================================
# ZMK Studio (required base)
# ========================================
CONFIG_ZMK_STUDIO=y
CONFIG_ZMK_STUDIO_LOCKING=n

# ========================================
# DYA Studio modules (Central side)
# ========================================
# BLE management
CONFIG_ZMK_BLE_MANAGEMENT=y
CONFIG_ZMK_BLE_MANAGEMENT_STUDIO_RPC=y

# Battery history
CONFIG_ZMK_BATTERY_HISTORY=y
CONFIG_ZMK_BATTERY_HISTORY_STUDIO_RPC=y

# Settings RPC (idle/sleep)
CONFIG_ZMK_SETTINGS_RPC=y
CONFIG_ZMK_SETTINGS_RPC_STUDIO=y

# Split event relay (central <-> peripheral)
CONFIG_ZMK_SPLIT_RELAY_EVENT=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_SPLIT_RUN_STACK_SIZE=768

# Runtime input processor (pointing device settings via DYA Studio)
CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR=y
CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC=y

# Setting persistence
CONFIG_ZMK_SETTINGS_SAVE_DEBOUNCE=10000
```

### Step 3: Peripheral Side .conf

Add to your **peripheral side** .conf:

```ini
# ========================================
# DYA Studio modules (Peripheral side)
# ========================================
# Battery history (data collection on peripheral)
CONFIG_ZMK_BATTERY_HISTORY=y

# Settings RPC (receive settings changes from central)
CONFIG_ZMK_SETTINGS_RPC=y

# Split event relay (peripheral -> central)
CONFIG_ZMK_SPLIT_RELAY_EVENT=y

# Setting persistence
CONFIG_ZMK_SETTINGS_SAVE_DEBOUNCE=10000
```

### Step 4: Device Tree Changes

#### 4a. Shared .dtsi (both halves)

Add the battery history behavior at the top of your shared `.dtsi` file:

```dts
#include <behaviors/battery_history_request.dtsi>
```

Without this, battery history will show "No battery history available."

#### 4b. Central .overlay (pointing device setup)

If you have a trackball, switch from **driver-based** to **input processor-based** configuration:

```dts
#include <input/processors.dtsi>
#include <input/processors/runtime-input-processor.dtsi>

/ {
    /* Add label "trackball_listener:" for input processor reference */
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;
        /* Runtime-configurable mouse speed/rotation via DYA Studio */
        input-processors = <&mouse_runtime_input_processor>;

        /* Scroll layer: convert cursor to scroll on specified layer */
        scroller {
            layers = <YOUR_SCROLL_LAYER_NUMBER>;
            input-processors = <&zip_xy_to_scroll_mapper &scroll_runtime_input_processor>;
        };
    };
};
```

**Important**: Remove these driver-based properties from the PMW3610 device node:
```dts
/* DELETE these from your trackball@0 node: */
automouse-layer = <...>;
scroll-layers = <...>;
```

These are replaced by `&mouse_runtime_input_processor` and the `scroller` child node.

### Step 5: Build and Flash

```bash
# Central side (with studio-rpc-usb-uart snippet)
west build -s zmk/app -b seeeduino_xiao_ble -d .build/RIGHT -p \
  -- -DSHIELD="YOUR_SHIELD_R rgbled_adapter" \
  -DZMK_CONFIG="/path/to/config" \
  -DZMK_EXTRA_MODULES="/path/to/zmk-config-repo" \
  -DEXTRA_CONF_FILE="/path/to/SHIELD_R.conf" \
  -DSNIPPET="studio-rpc-usb-uart"

# Peripheral side (no snippet)
west build -s zmk/app -b seeeduino_xiao_ble -d .build/LEFT -p \
  -- -DSHIELD="YOUR_SHIELD_L rgbled_adapter" \
  -DZMK_CONFIG="/path/to/config" \
  -DZMK_EXTRA_MODULES="/path/to/zmk-config-repo" \
  -DEXTRA_CONF_FILE="/path/to/SHIELD_L.conf"
```

Flash order:
1. **settings_reset** firmware on both halves first (clears BLE bonding data)
2. Central side firmware (R)
3. Peripheral side firmware (L)
4. Connect via USB, open https://studio.dya.cormoran.works/

---

## Known Issue: BLE Freeze on Split Keyboards with Sensors

### Symptom

Central side **completely freezes** when the peripheral connects (green LED stays on, USB doesn't enumerate). Works fine when peripheral is powered off.

### Who is Affected?

Any split keyboard that has **ALL** of the following:
- `CONFIG_ZMK_SPLIT_RELAY_EVENT=y`
- Sensors (e.g., EC11 encoder with `CONFIG_EC11=y`)
- Battery level fetching (`CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y`)

This creates **4+ concurrent GATT subscriptions** (position_state, sensor, relay_event, battery) which triggers the bug.

### Root Cause

In `zmk/app/src/split/bluetooth/central.c`, all GATT subscriptions share a **single** `sub_discover_params` structure for CCC (Client Characteristic Configuration) descriptor discovery. When multiple subscriptions are initiated during GATT characteristic discovery, the shared structure's state gets corrupted, causing the Zephyr BLE stack to deadlock.

### Fix

Apply the patch from [`patches/central_sub_discover_params_fix.patch`](../patches/central_sub_discover_params_fix.patch):

Each subscription gets its own dedicated `bt_gatt_discover_params`:

| Subscription | Before (shared) | After (dedicated) |
|---|---|---|
| position_state | `&slot->sub_discover_params` | `&slot->sub_discover_params` (unchanged) |
| sensor | `&slot->sub_discover_params` | `&slot->sensor_sub_discover_params` |
| relay_event | `&slot->sub_discover_params` | `&slot->relay_event_sub_discover_params` |
| battery | `&slot->sub_discover_params` | `&slot->batt_lvl_sub_discover_params` |

Apply the patch:
```bash
cd zmk-workspace/zmk
git apply /path/to/patches/central_sub_discover_params_fix.patch
```

> **Note**: This patch is overwritten by `west update`. Re-apply after each update. Consider reporting this to cormoran for an upstream fix.

---

## Module Reference

| Module | west.yml | Central .conf | Peripheral .conf | DTS |
|--------|----------|---------------|-------------------|-----|
| ZMK Studio | `zmk` (cormoran fork) | `ZMK_STUDIO=y` | - | snippet: `studio-rpc-usb-uart` |
| BLE Mgmt | `zmk-module-ble-management` | `ZMK_BLE_MANAGEMENT=y` + `_STUDIO_RPC=y` | - | - |
| Battery History | `zmk-module-battery-history` | `ZMK_BATTERY_HISTORY=y` + `_STUDIO_RPC=y` | `ZMK_BATTERY_HISTORY=y` | `battery_history_request.dtsi` |
| Settings RPC | `zmk-module-settings-rpc` | `ZMK_SETTINGS_RPC=y` + `_STUDIO=y` | `ZMK_SETTINGS_RPC=y` | - |
| Event Relay | (in settings-rpc) | `ZMK_SPLIT_RELAY_EVENT=y` | `ZMK_SPLIT_RELAY_EVENT=y` | **Requires central.c patch** |
| Input Processor | `zmk-module-runtime-input-processor` | `ZMK_RUNTIME_INPUT_PROCESSOR=y` + `_STUDIO_RPC=y` | - | `runtime-input-processor.dtsi` |

## Debugging Tips

1. **Freeze at boot**: Apply the `sub_discover_params` patch (see above)
2. **"No battery history"**: Add `#include <behaviors/battery_history_request.dtsi>` to shared .dtsi
3. **"Runtime input processor not available"**: Add the module to west.yml + conf + dtsi (see Step 4b)
4. **Flash settings_reset first**: Always do this when adding/removing GATT characteristics
5. **Test without peripheral**: Boot central alone to verify it works before connecting peripheral
6. **Incremental approach**: Enable modules one at a time to isolate issues

## References

- **KUKEY42 implementation**: https://github.com/t-ogura/zmk-config-KUKEY42/tree/support-dya-studio
- **dya-dash (cormoran's reference)**: https://github.com/cormoran/zmk-keyboard-dya-dash
- **DYA Studio web app**: https://studio.dya.cormoran.works/
- **ZMK Input Processors**: https://zenn.dev/kot149/articles/zmk-input-processor-cheat-sheet
- **ZMK Auto Mouse Layer**: https://zenn.dev/kot149/articles/zmk-auto-mouse-layer
