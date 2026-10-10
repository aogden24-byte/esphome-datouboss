# ESPHome package for the Datouboss 4863U (SRNE-based) inverter

Local, fast RS485/Modbus monitoring and control of a Datouboss 4863U hybrid inverter from Home Assistant, using an ESP32 and ESPHome. No cloud, no WiFi dongle, and updates every 1 to 20 seconds instead of the official dongle's 5 minutes.

The Datouboss is a rebrand of an SRNE design, but its register map does **not** match the published SRNE maps (the standard `0x0100`+ telemetry and `0xDF00` power registers are rejected with Modbus exception 2). Everything here was mapped by scanning the device and sniffing the official Solar of Things app while it changed settings.

## Tested hardware

- Datouboss 4863U: 6300 W, 48 V battery, 120 V / 60 Hz single phase, dual MPPT
- M5Stack Atom Lite with the Atomic RS485 base (any ESP32 plus RS485 transceiver should work)

## Wiring

Use the inverter's **WiFi dongle / COM RJ45 port**, not the BMS port.

| RJ45 pin | Signal |
|---|---|
| 1 | RS485 A |
| 2 | RS485 B |
| 8 | GND |

Serial settings: 9600 baud, 8N1, Modbus slave address 1. The package polls as a Modbus master, so unplug the WiFi dongle while it is connected. (Sharing the bus with the dongle is possible: the dongle polls `0x0000`–`0x0043` every 30–60 s, plus single-register reads while the app or portal is open. A bus-sharing version that only transmits in quiet slots is being tested.)

## Installation

1. Create a device in the ESPHome dashboard and replace its YAML with `example-m5-atom-lite.yaml`.
2. The `packages:` URL already points at this repo; change it only if you fork it.
3. Add `datouboss_api_key`, `datouboss_ota_password`, `wifi_ssid` and `wifi_password` to your `secrets.yaml`.
4. Adjust `uart_rx_pin`, `uart_tx_pin` or `modbus_address` under `substitutions:` if your hardware differs.
5. Install.

Raw and unidentified registers are included but disabled by default in Home Assistant; enable them from the device page if you want to help map more of them.

## Register map

Mapped 2026-10-10 by pressing **Read** on each row of the Solar of Things portal's control page while sniffing the bus: every row sends one Modbus read of its register, which ties each setting's name to an address and value. Earlier entries were confirmed by write tests.

**Split registers:** where two settings share a register, the one listed **first** on the portal is the **high byte**.

Confidence: **W** = write-verified (read back after writing), **S** = seen in an app write, **R** = matched by portal Read (name, address and value), **I** = inferred.

| Register | Byte | Setting | Values / scale | Read 2026-10-10 | Conf. |
|---|---|---|---|---|---|
| `0x005C`–`0x005F` | hi/lo | RGB lights (8 settings) | not used on the 4863U | 0 | R |
| `0x0060` | hi | Output Priority | 1 Mains, 2 PV, 3 Battery | 2 | S (2, 3), I (1) |
| `0x0060` | lo | ID Address | — | 0 | R |
| `0x0061` | word | Rated output power | W | — | earlier scan |
| `0x0062` | hi / lo | Output Voltage / Output Frequency | V / Hz | 120 / 60 | R |
| `0x0063` | hi | Charging Priority | 0 PV + Mains (SNU), 1 Only PV (OSO), 2 PV Priority (CSO) | 2 | W (0, 1), R (2) |
| `0x0063` | lo | Battery Free Mode Selection | 0 Battery Connected | 0 | R |
| `0x0064` | hi / lo | Max Charging Current / Mains Max Charging Current | A | 55 / 12 | R |
| `0x0066` | hi | Input Mains Range | 0 = 165–280 V, 1 = 120–280 V | 1 | W |
| `0x0066` | lo | Battery Nominal Voltage | V | 48 | R |
| `0x0067` | word | Equalization Voltage | ×0.1 V | 55.7 | R, W |
| `0x0068` | word | Floating Charge Voltage | ×0.1 V | 54.2 | R, W |
| `0x0069` | word | Battery High Voltage Protection | ×0.1 V | 66.9 | R |
| `0x006A` | word | Battery High Voltage Alarm | ×0.1 V | 60.9 | R |
| `0x006B` | word | Battery High Voltage Recovery | ×0.1 V | 55.4 | R, W |
| `0x006C` | word | Battery Low Voltage Protection | ×0.1 V | 44.0 | R |
| `0x006D` | word | Battery Low Voltage Alarm | ×0.1 V | 45.0 | R |
| `0x006E` | word | Battery Low Voltage Recovery | ×0.1 V | 52.0 | R |
| `0x0070` | word | Inverter Priority Battery Supply Voltage | ×0.1 V | 52.8 | R, W |
| `0x0071` | word | Inverter Priority Mains Supply Voltage | ×0.1 V | 48.0 | R, W |
| `0x0072` | hi | Over Temperature Restart | 0 off, 1 on | 0 | W |
| `0x0072` | lo | Work Mode | 0 Hybrid, 1 Grid Connected Power Supply, 2 Off Grid, 3 Grid Connected Anti Backflow | 1 | W |
| `0x0075` | hi | HFP Grid Connected Current (export limit) | A | 14 | W |
| `0x0075` | lo | Neutral Grounding | 0 not grounding, 1 grounding (see warning below) | 0 | R |
| `0x0076` | word | Output Off Battery Voltage | 0 = disabled; ×0.1 V assumed | 0 | R |
| `0x0077` | word | Output Restore Battery Voltage | 0 = disabled; ×0.1 V assumed | 0 | R |
| `0x0078` | hi / lo | Restore Main Output SOC / Turn Off Main Output SOC | % (0 = disabled) | 0 / 0 | R (byte order I) |
| `0x007B` | word | Intelligent Output Switch | 0 off, 1 on | 1 | W |
| `0x0087` | hi | BMS Lithium Battery Address | — | 1 | R |
| `0x0087` | lo | Remote Power On/Off | 0 off, 1 on; actually cuts AC output | 1 | W, S |
| `0x0088` | hi | Lithium Battery Shutdown SOC | % | 5 | R |
| `0x0088` | lo | Battery Protocol | 0 Lead-acid, 1 Growatt, 2 Voltronic, 3 Pylontech, 4 PACE, 5 Pylontech-A | 4 | R |
| `0x0089` | hi / lo | Charging Cut-off SOC / Low SOC Recovery | % | 100 / 15 | R |
| `0x008A` | hi / lo | Priority Inverter To Mains Supply SOC / To Battery Powered SOC | % | 20 / 100 | W |

Not yet located: **Load Overload Restart Enable** and **Battery Low Voltage Recovery Enable**. Candidates are `0x006F` (reads 0), `0x0065` and `0x0086`.

The portal labels battery protocols by transliterated Chinese brand names: Guriwat = Growatt, Sun Moon Yuan = Voltronic, Paineng = Pylontech, Peicheng = PACE.

> **Neutral Grounding** closes the inverter's internal neutral–ground bond relay. If your system is already bonded at the main panel (normal in North America), setting it creates a second bond and puts neutral current on the ground conductor. Do not toggle it to test it.

**Corrections to earlier versions of this README:** `0x0088` high byte is the shutdown SOC, not the BMS protocol (the protocol is the low byte, and PACE = 4). `0x0067` is Equalization Voltage, not Charge Disconnect Voltage. `0x006B` is Battery High Voltage Recovery, not bulk voltage. `0x006E` is Battery Low Voltage Recovery, not grid charge enable. The `0x0087` high byte is the BMS address, not a fixed `0x01`. The package YAML still uses the old entity names; it will be updated to match.

## Telemetry

Battery voltage (`0x0008`), battery net current (`0x0011`, signed), PV1/PV2 current and voltage (`0x000B`–`0x000E`), grid voltage (`0x0005`) and frequency (`0x0012`), DC bus voltage (`0x0004`), AC output active power (`0x0020`), status word (`0x0017`, bit 2 grid present, bit 8 inverter output active), heatsink and battery temperature, BMS pack values (`0x0030`–`0x0035`), per-string lifetime generation (`0x0039` PV1, `0x003A` PV2, in kWh), and grid input power (`0x003B`, signed W; import confirmed, export sign not yet observed). PV power, total PV power, battery power and load percentage are calculated in ESPHome.

## Known quirks

- **Split registers.** Many registers hold two settings, one per byte (`0x0060`, `0x0062`–`0x0066`, `0x0072`, `0x0075`, `0x0078`, `0x0087`–`0x008A`). The package preserves the other byte on every write, but the official app does not always: selecting a Grid Connected Mode in the app can silently change Over Temperature Restart.
- **No boot writes.** All template switches use `restore_mode: DISABLED`. ESPHome's default (`ALWAYS_OFF`) would switch the inverter output off on every reboot.
- **Inverter Output Active** reads off whenever the grid is present and healthy, in every Grid Connected Mode, because the unit bypasses the grid straight to the load. AC Output Active Power has also read 0 W while loads were running on bypass, so treat it with caution in that state.
- **Backfeed not yet observed.** The portal's "PV Priority On Grid Work Flag" reads "Can Connect To Grid" only with Output Priority = PV and Work Mode = Grid Connected Power Supply; in Battery Priority the unit stays islanded. The one qualifying sample had all PV surplus going into the battery, so export of a genuine surplus is still untested. ("Charge Disconnect Voltage" never stopped charging because `0x0067` is actually Equalization Voltage.)
- **The Solar of Things app and portal** show cached values that can lag the inverter (for example the input voltage range, float voltage and HV recovery). The portal's per-row **Read** button forces a live read.

## Disclaimer

This writes settings to a high-power inverter using a reverse-engineered register map. Wrong values can cut power to your loads or change how your battery is charged. Use it at your own risk and check settings against the inverter's own display.

## Contributing

To map a new setting: flash a listen-only sniffer to the ESP32, wire it in parallel with the WiFi dongle, change the setting in the Solar of Things app, and look for the function 06 write (`01 06 <addr hi> <addr lo> <value hi> <value lo> <crc>`). Pull requests with confirmed registers are welcome.
