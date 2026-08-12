# Renac hybrid inverter — Modbus RTU register map + ESPHome package

Reading a Renac hybrid inverter over RS485 into Home Assistant, without the
cloud and without the vendor dongle.

Renac does not publish a Modbus document for its hybrid range. The only
protocol PDF that circulates (*JXNT MODBUS Interface Definition*) covers the
on-grid NAC series and contains no battery, SOC or EPS registers at all — its
map starts at 10000 and does not apply to hybrids. The map below was pieced
together from [Yety/Renacci](https://github.com/Yety/Renacci) and extended by
probing a live unit.

**Tested on:** Renac N3-HV, three-phase hybrid, AC-coupled retrofit
(PV on a separate inverter), firmware as shipped mid-2026.

If you confirm this on another model, please open an issue saying which —
particularly single-phase units, where the S/T registers presumably read zero.

## Wiring

| Inverter RJ45 pin | Signal |
|---|---|
| 3 | RS485_A |
| 4 | RS485_B |

9600 baud, 8N1, slave address 1, function code 0x03 (holding registers).

Any ESP8266 or ESP32 with an RS485 transceiver works. With a plain MAX485 you
must drive DE+RE from a GPIO — set `flow_control_pin`. Modules with automatic
direction control don't need it; delete that line.

The inverter has a 120 Ω terminator built in, so if you're daisy-chaining
other devices, count that as one of your two terminations.

## Register map

All values are **16-bit signed big-endian**. Reading them as unsigned turns
every negative number into roughly 65000.

The block is **11000–11143**. Anything outside returns exception 2 — including
everything below 11000, which is why address scanning from zero finds nothing
and makes the inverter look unresponsive.

### PV input

| Register | Value | Scale |
|---|---|---|
| 11000 | PV1 voltage | ÷10 |
| 11001 | PV1 current | ÷10 |
| 11002 | PV1 power | 1 |
| 11003 | PV2 voltage | ÷10 |
| 11004 | PV2 current | ÷10 |
| 11005 | PV2 power | 1 |

Reads zero on AC-coupled retrofits.

### Temperatures

| Register | Value | Scale |
|---|---|---|
| 11016 | Internal | ÷10 |
| 11017 | Charger | ÷10 |
| 11018 | Boost | ÷10 |
| 11019 | Inverter | ÷10 |

### Battery and BMS

| Register | Value | Scale |
|---|---|---|
| 11020 | Battery voltage | ÷10 |
| 11021 | Battery current | ÷10 |
| 11022 | Battery power (+ charging) | 1 |
| 11023 | BMS voltage | ÷10 |
| 11024 | BMS current | ÷10 |
| 11025 | BMS temperature | ÷10 |
| 11026 | SOC % | 1 |
| 11027 | SOH % | 1 |
| 11040 | Battery status (bitfield) | — |
| 11041 | Cycle count | 1 |
| 11056 | Capacity, Ah | 1 |

11020 and 11023 measure the same thing from the inverter and BMS sides
respectively — a useful cross-check. A growing gap between them points at
cabling or contactor resistance.

### Battery limits and cell data

| Register | Value | Scale |
|---|---|---|
| 11028 | Charge voltage limit | ÷10 |
| 11029 | Discharge cutoff voltage | ÷10 |
| 11030 | Charge current limit | ÷10 |
| 11031 | Discharge current limit (negative) | ÷10 |
| 11032 | Highest cell voltage, mV | 1 |
| 11033 | Lowest cell voltage, mV | 1 |
| 11034 | Cell number, highest | 1 |
| 11035 | Cell number, lowest | 1 |
| 11036 | Cell temperature max | ÷10 |
| 11037 | Cell temperature min | ÷10 |

The per-cell registers are the most useful thing in the whole map and are
absent from the vendor app. The difference between 11032 and 11033 is your
pack imbalance; watching it over months catches a failing cell long before
the BMS raises an alarm.

### Grid

| Register | Value | Scale |
|---|---|---|
| 11076–11078 | Grid voltage R/S/T | ÷10 |
| 11079–11081 | Inverter current R/S/T | ÷10 |
| 11082–11084 | Inverter power R/S/T | 1 |
| 11085–11087 | Grid frequency R/S/T | ÷100 |

### EPS (backup output)

| Register | Value | Scale |
|---|---|---|
| 11088–11090 | EPS voltage R/S/T | ÷10 |
| 11091–11093 | EPS current R/S/T | ÷10 |
| 11094–11096 | EPS power R/S/T | 1 |
| 11097 | EPS frequency | ÷100 |

Zero while the grid is present.

### Meters and load

| Register | Value | Scale |
|---|---|---|
| 11098–11101 | Meter 1 power R/S/T/total | 1 |
| 11102–11105 | Meter 2 power R/S/T/total | 1 |
| 11106–11109 | Meter 3 power R/S/T/total | 1 |
| 11110–11113 | Load power R/S/T/total | 1 |
| 11126–11133 | Meter 1 power as INT32 (R/S/T/total) | 1 |

Meters 2 and 3 read zero unless extra metering is installed. The INT32 copies
at 11126 only matter above 32 kW.

On an AC-coupled retrofit the "load" registers reflect what passes through the
inverter's own CT, which is not the same as house consumption. Check against
your own metering before building automations on them.

### Status

| Register | Value | Normal |
|---|---|---|
| 11040 | Battery status | 0x0101 |
| 11057 | Inverter state | 0x0002 |
| 11058 | BMS state | 0x0001 |
| 11059 | Meter state | 0x0001 |
| 11060 | Alarm | 0x0000 |

Bit meanings are not yet worked out. If you catch one of these during a fault,
please report the value and what the display showed.

### Empty or unidentified

11006–11015 and 11061–11075 read zero in every state observed. 11114–11125 and
11134–11143 hold constants (131, 100, 201, 102, 101 and then zeros) that look
like firmware or configuration identifiers.

**There are no cumulative energy registers anywhere in the block.** For kWh in
Home Assistant, apply a Riemann sum integration to the power sensors.

## Sign conventions

Verified against a live system with known charge and discharge:

- **Inverter AC power** — negative means importing from the grid (charging),
  positive means exporting (discharging)
- **Battery power** — positive charging, negative discharging
- **Meter power** — positive importing, negative exporting

Sanity check: with the battery taking 1119 W, the three AC phases summed to
−1190 W. The 71 W difference is conversion loss, about 6%, which is what you'd
expect. If your numbers don't close like this, something in the map is shifted
for your model — please report it.

## Installation

### Option 1 — remote package (recommended)

ESPHome can pull the package straight from git, so there are no files to copy
and updates arrive on their own. Paste this into your device configuration:

```yaml
substitutions:
  renac_name: "Renac"
  renac_address: "1"
  renac_update_interval: "15s"
  renac_flow_control_pin: "4"

packages:
  renac:
    url: https://github.com/YOUR-USER/renac-hybrid-modbus
    files: [lang/en.yaml]
    refresh: 1d
```

Swap `lang/en.yaml` for `lang/pl.yaml` and every entity is in Polish. Nothing
else changes.

This works fine inside the Home Assistant **ESPHome Builder** add-on: it is
just four lines in the device YAML, and the dashboard's editor is all you
need. ESPHome clones the repo into `.esphome/packages/` at build time.

### Option 2 — local files

If you would rather keep a copy, put the files in your ESPHome directory:

```
/config/esphome/
├── my-inverter.yaml        your device config
├── renac-core.yaml
└── lang/
    ├── en.yaml
    └── pl.yaml
```

and in `my-inverter.yaml`:

```yaml
packages:
  renac: !include lang/en.yaml
```

The ESPHome Builder dashboard cannot create folders or non-device files, so
you need file access: the **Studio Code Server** or **File Editor** add-on, or
a Samba share. The dashboard will not list `renac-core.yaml` or the language
files — that is expected, they are includes, not devices. Relative paths
resolve from the file doing the including, which is why `lang/en.yaml`
references `../renac-core.yaml`.

### Option 3 — paste it all in

With no file access at all, paste the contents of a language file and the core
into your device configuration directly. It works, but you lose updates and
the point of the split.

## How the language split works

```
renac-core.yaml     register addresses, scaling, logic — never translated
lang/en.yaml        English entity names + include of the core
lang/pl.yaml        Polish entity names + include of the core
```

Every entity in the core is named `${renac_name} ${e_something}`. The language
file defines those `e_*` substitutions and pulls the core in as a package;
substitutions from the including file win over the package's, so the language
file decides the names and the core decides everything else.

**Adding a language:** copy `lang/en.yaml`, translate the quoted values on the
right, leave the `e_*` keys and the `packages:` block alone. Send a pull
request — a translation cannot break the register map, because it never
touches it.

Note that Home Assistant derives entity IDs from the entity name, so switching
language later gives you a fresh set of entities and the old history stays
attached to the old IDs. Pick a language before you start collecting data, or
rename the entities in HA afterwards.

The core defines its own `uart`, `modbus` and `modbus_controller`. If your
device already has a UART, remove that section and point `uart_id` at yours.

Around 60 entities are polled per cycle. At 9600 baud a full pass takes roughly
10 seconds, so don't set the interval below 15 s — and raise it to 30 s if the
logs show timeouts.

## Contributing

Translations are the easiest contribution — see above. Beyond that, useful
things to report: another model confirmed working, status register
values during a fault, anything identified in the unknown registers, or a
single-phase unit's behaviour on the S/T registers.

## Licence

CC0 / public domain.
