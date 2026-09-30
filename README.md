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

Serial settings: 9600 baud, 8N1, Modbus slave address 1. Only one Modbus master can be on the bus, so unplug the WiFi dongle while the ESP32 is connected.

## Installation

1. Create a device in the ESPHome dashboard and replace its YAML with `example-m5-atom-lite.yaml`.
2. The `packages:` URL already points at this repo; change it only if you fork it.
3. Add `datouboss_api_key`, `datouboss_ota_password`, `wifi_ssid` and `wifi_password` to your `secrets.yaml`.
4. Adjust `uart_rx_pin`, `uart_tx_pin` or `modbus_address` under `substitutions:` if your hardware differs.
5. Install.

Raw and unidentified registers are included but disabled by default in Home Assistant; enable them from the device page if you want to help map more of them.

## Controls

| Entity | Register | Values | Status |
|---|---|---|---|
| Inverter Power Switch | `0x0087` low byte (high byte fixed at `0x01`) | 1 on, 0 off | Confirmed both ways; actually cuts AC output |
| Intelligent Output Switch | `0x007B` | 1 on, 0 off | Confirmed both ways |
| Grid Connected Mode | `0x0072` low byte | 0 Hybrid, 1 Grid Connected Power Supply, 2 Off Grid, 3 Grid Connected Anti Backflow | Confirmed, all four |
| Over Temperature Restart | `0x0072` high byte | 1 on, 0 off | Confirmed both ways |
| Charging Priority | `0x0063` high byte | 0 SNU (PV + mains), 1 OSO (only solar), 2 CSO (PV priority) | 0 and 1 confirmed, 2 inferred |
| Load Priority Supply Mode | `0x0060` high byte | 2 PV, 3 Battery | **Read-only here**; low byte untested, change it from the app |
| Master AC Input Limit | `0x0075` high byte | amps | Confirmed |
| Priority Inverter To Mains Supply SOC | `0x008A` high byte | % | Confirmed |
| Priority Inverter To Battery Powered SOC | `0x008A` low byte | % | Confirmed |
| Inverter Priority Battery Supply Voltage | `0x0070` | ×0.1 V | Confirmed |
| Inverter Priority Mains Supply Voltage | `0x0071` | ×0.1 V | Confirmed |
| Charge Disconnect Voltage | `0x0067` | ×0.1 V | Register confirmed, but see quirks |
| BMS Protocol | `0x0088` high byte | 5 = PACE | Label confirmed against the manual |
| AC Input Voltage Range | `0x0066` high byte | 0 = 165–280 V, 1 = 120–280 V | Confirmed |

Float voltage (`0x0068`), bulk voltage (`0x006B`), grid charge enable voltage (`0x006E`) and max charge current (`0x0064`) are inherited guesses that were never write-tested.

## Telemetry

Battery voltage (`0x0008`), battery net current (`0x0011`, signed), PV1/PV2 current and voltage (`0x000B`–`0x000E`), grid voltage (`0x0005`) and frequency (`0x0012`), DC bus voltage (`0x0004`), AC output active power (`0x0020`), status word (`0x0017`, bit 2 grid present, bit 8 inverter output active), heatsink and battery temperature, BMS pack values (`0x0030`–`0x0035`), and per-string lifetime generation (`0x0039` PV1, `0x003A` PV2, in kWh). PV power, total PV power, battery power and load percentage are calculated in ESPHome.

## Known quirks

- **Split registers.** Several registers hold two settings, one per byte (`0x0060`, `0x0072`, `0x0088`, `0x008A`). The package preserves the other byte on every write, but the official app does not always: selecting a Grid Connected Mode in the app can silently change Over Temperature Restart.
- **No boot writes.** All template switches use `restore_mode: DISABLED`. ESPHome's default (`ALWAYS_OFF`) would switch the inverter output off on every reboot.
- **Inverter Output Active** reads off whenever the grid is present and healthy, in every Grid Connected Mode, because the unit bypasses the grid straight to the load. AC Output Active Power has also read 0 W while loads were running on bypass, so treat it with caution in that state.
- **Backfeed did not work** in testing. Grid Connected Power Supply Mode, a full battery, and a lowered Charge Disconnect Voltage never produced export. Charge Disconnect Voltage also never stopped charging when the battery sat above it.
- **The Solar of Things app** shows stale values on its dashboard (for example the input voltage range) until it is fully restarted.

## Disclaimer

This writes settings to a high-power inverter using a reverse-engineered register map. Wrong values can cut power to your loads or change how your battery is charged. Use it at your own risk and check settings against the inverter's own display.

## Contributing

To map a new setting: flash a listen-only sniffer to the ESP32, wire it in parallel with the WiFi dongle, change the setting in the Solar of Things app, and look for the function 06 write (`01 06 <addr hi> <addr lo> <value hi> <value lo> <crc>`). Pull requests with confirmed registers are welcome.
