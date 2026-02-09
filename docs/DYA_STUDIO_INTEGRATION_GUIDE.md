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

## Known Issues

---

### Issue 1: BLE Freeze on Split Keyboards (central.c patch)

#### Symptom

Central side **completely freezes** when the peripheral connects (green LED stays on, USB doesn't enumerate). Works fine when peripheral is powered off.

#### Who is Affected?

Any split keyboard that has **ALL** of the following:
- `CONFIG_ZMK_SPLIT_RELAY_EVENT=y`
- Sensors (e.g., EC11 encoder with `CONFIG_EC11=y`)
- Battery level fetching (`CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y`)

This creates **4+ concurrent GATT subscriptions** (position_state, sensor, relay_event, battery) which triggers the bug.

#### Root Cause

In `zmk/app/src/split/bluetooth/central.c`, all GATT subscriptions share a **single** `sub_discover_params` structure in the `peripheral_slot` struct for CCC (Client Characteristic Configuration) descriptor discovery.

When multiple subscriptions are initiated during GATT characteristic discovery, the shared structure's state gets corrupted mid-flight:

1. `subscribe_params` (position_state) initiates GATT discovery using `sub_discover_params`
2. Before that completes, `sensor_subscribe_params` also uses the same `sub_discover_params`
3. The second write overwrites the first's state → BLE stack deadlocks

**Original code** (`peripheral_slot` struct):
```c
struct peripheral_slot {
    struct bt_conn *conn;
    struct bt_gatt_discover_params discover_params;
    struct bt_gatt_subscribe_params subscribe_params;
    struct bt_gatt_subscribe_params sensor_subscribe_params;
#if IS_ENABLED(CONFIG_ZMK_SPLIT_RELAY_EVENT)
    struct bt_gatt_subscribe_params relay_event_subscribe_params;
#endif
    struct bt_gatt_discover_params sub_discover_params;   // ← SHARED by ALL subscriptions!
    uint16_t run_behavior_handle;
#if IS_ENABLED(CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING)
    struct bt_gatt_subscribe_params batt_lvl_subscribe_params;
    // ...
```

And in `split_central_chrc_discovery_func()`, all subscriptions reference the same struct:
```c
slot->sensor_subscribe_params.disc_params = &slot->sub_discover_params;       // shared!
slot->relay_event_subscribe_params.disc_params = &slot->sub_discover_params;  // shared!
slot->batt_lvl_subscribe_params.disc_params = &slot->sub_discover_params;     // shared!
```

#### Fix

Each subscription gets its own dedicated `bt_gatt_discover_params`:

| Subscription | Before (shared) | After (dedicated) |
|---|---|---|
| position_state | `&slot->sub_discover_params` | `&slot->sub_discover_params` (unchanged) |
| sensor | `&slot->sub_discover_params` | `&slot->sensor_sub_discover_params` |
| relay_event | `&slot->sub_discover_params` | `&slot->relay_event_sub_discover_params` |
| battery | `&slot->sub_discover_params` | `&slot->batt_lvl_sub_discover_params` |

#### Applying the Patch

```bash
# From your zmk-workspace directory:
cd zmk
git apply /path/to/your-config/patches/central_sub_discover_params_fix.patch

# Verify the patch applied correctly:
git diff --stat
#  app/src/split/bluetooth/central.c | 13 +++++++------
#  1 file changed, 7 insertions(+), 6 deletions(-)
```

#### Patch Content

File: `patches/central_sub_discover_params_fix.patch`

```diff
diff --git a/app/src/split/bluetooth/central.c b/app/src/split/bluetooth/central.c
index 9fb41a5f..4260b551 100644
--- a/app/src/split/bluetooth/central.c
+++ b/app/src/split/bluetooth/central.c
@@ -49,14 +49,17 @@ struct peripheral_slot {
     struct bt_conn *conn;
     struct bt_gatt_discover_params discover_params;
     struct bt_gatt_subscribe_params subscribe_params;
+    struct bt_gatt_discover_params sub_discover_params;
     struct bt_gatt_subscribe_params sensor_subscribe_params;
+    struct bt_gatt_discover_params sensor_sub_discover_params;
 #if IS_ENABLED(CONFIG_ZMK_SPLIT_RELAY_EVENT)
     struct bt_gatt_subscribe_params relay_event_subscribe_params;
+    struct bt_gatt_discover_params relay_event_sub_discover_params;
 #endif
-    struct bt_gatt_discover_params sub_discover_params;
     uint16_t run_behavior_handle;
 #if IS_ENABLED(CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING)
     struct bt_gatt_subscribe_params batt_lvl_subscribe_params;
+    struct bt_gatt_discover_params batt_lvl_sub_discover_params;
     struct bt_gatt_read_params batt_lvl_read_params;
 #endif
 #if IS_ENABLED(CONFIG_ZMK_SPLIT_PERIPHERAL_HID_INDICATORS)
@@ -711,7 +714,7 @@ static uint8_t split_central_chrc_discovery_func(...)
-            slot->sensor_subscribe_params.disc_params = &slot->sub_discover_params;
+            slot->sensor_subscribe_params.disc_params = &slot->sensor_sub_discover_params;
@@ -726,7 +729,7 @@ static uint8_t split_central_chrc_discovery_func(...)
-            slot->relay_event_subscribe_params.disc_params = &slot->sub_discover_params;
+            slot->relay_event_subscribe_params.disc_params = &slot->relay_event_sub_discover_params;
@@ -775,7 +778,7 @@ static uint8_t split_central_chrc_discovery_func(...)
-            slot->batt_lvl_subscribe_params.disc_params = &slot->sub_discover_params;
+            slot->batt_lvl_subscribe_params.disc_params = &slot->batt_lvl_sub_discover_params;
```

> **Important**: This patch is overwritten by `west update`. Re-apply after each update.
> The actual patch file in `patches/` can be applied directly with `git apply`.

---

### Issue 2: DYA Studio Cannot Detect Keyboard via Bluetooth

#### Symptom

DYA Studio detects the keyboard via USB (CDC ACM serial) but **cannot find it via Bluetooth** pairing. The BLE device picker in the browser shows no matching devices.

#### Root Cause Analysis

There are **two independent issues** preventing BLE detection:

**Issue A: Studio GATT Service UUID Not Advertised**

DYA Studio (based on [zmk-studio-ts-client](https://github.com/zmkfirmware/zmk-studio-ts-client)) uses the Web Bluetooth API to discover keyboards:

```typescript
// zmk-studio-ts-client/src/transport/gatt.ts
const SERVICE_UUID = '00000000-0196-6107-c967-c5cfb1c2482a';

let dev = await navigator.bluetooth.requestDevice({
  filters: [{ services: [SERVICE_UUID] }],  // ← filters by THIS UUID
  optionalServices: [SERVICE_UUID],
});
```

The Web Bluetooth API **only shows devices that advertise the filtered service UUID**. However, the ZMK firmware's BLE advertisement data (`ble.c:74-80`) only includes HID and Battery 16-bit UUIDs:

```c
// zmk/app/src/ble.c
static struct bt_data zmk_ble_ad[] = {
    BT_DATA_BYTES(BT_DATA_GAP_APPEARANCE, 0xC1, 0x03),
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_UUID16_SOME, 0x12, 0x18,   /* HID Service */
                  0x0f, 0x18                          /* Battery Service */
                  ),
};
```

The 128-bit Studio service UUID (`00000000-0196-6107-c967-c5cfb1c2482a`) is **not included**, so the keyboard is invisible to the Web Bluetooth device picker.

The GATT service itself IS properly registered in `gatt_rpc_transport.c`, but registration alone doesn't make it appear in advertisements.

**Issue B: Advertising Stops When Connected to Host**

When the keyboard is paired and connected to a BLE host (e.g., PC/tablet), `zmk_ble_active_profile_is_connected()` returns `true`. The advertising logic in `ble.c` then sets `desired_adv = ZMK_ADV_NONE` (stop advertising) — unless `directed_advertising_enabled` is `true`.

The `directed_advertising_enabled` flag is only set to `true` when `zmk_studio_core_unlock()` is called (`core.c:50`). But with `CONFIG_ZMK_STUDIO_LOCKING=n`, the keyboard starts in UNLOCKED state and `unlock()` is **never called**, so the flag remains `false`.

```
Flow when CONFIG_ZMK_STUDIO_LOCKING=n:
  Boot → state = UNLOCKED (already) → unlock() never called
       → directed_advertising_enabled = false (forever)
       → Connected to BLE host → desired_adv = ZMK_ADV_NONE
       → Keyboard stops advertising → Invisible to DYA Studio
```

#### Potential Fixes

**Fix for Issue B (try first — config change only):**

Enable Studio locking to activate the directed advertising flow:

```ini
# Replace CONFIG_ZMK_STUDIO_LOCKING=n with:
CONFIG_ZMK_STUDIO_LOCKING=y
CONFIG_ZMK_STUDIO_LOCK_IDLE_TIMEOUT_SEC=600
```

With locking enabled, the user performs a physical unlock action → `zmk_studio_core_unlock()` is called → `directed_advertising_enabled = true` → keyboard continues advertising even when connected.

**Fix for Issue A (firmware patch — may be needed):**

The Studio service 128-bit UUID needs to be added to the BLE advertisement data in `ble.c`. This likely requires a patch to the cormoran ZMK fork. The 128-bit UUID is 16 bytes, so it may need to go in the scan response data due to the 31-byte advertisement limit.

#### Current Status

This issue is **upstream in the cormoran ZMK fork** and may require coordination with the fork maintainer. The USB transport works reliably as a workaround.

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
