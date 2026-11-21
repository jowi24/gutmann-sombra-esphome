🇬🇧 **English** · 🇩🇪 [Deutsch](README.de.md)

# gutmann-sombra-esphome

ESPHome firmware that connects a Gutmann Sombra range hood to Home
Assistant. It runs on the [hood-control board](https://github.com/jowi24/gutmann-sombra-pcb)
(ESP8266), which sits in the cable between the hood electronics and the
front panel, reads which front-panel LEDs are lit and can simulate key
presses. The original front panel keeps working as before.

> **Status:** running on the author's hood since 2026-11-21 (ESPHome
> 2026.11). Tested: reading the front panel, all key presses, run-on timer,
> reboot without affecting the hood, OTA. Not yet tested: filter reset.

## Related repositories

- **Hardware:** [gutmann-sombra-pcb](https://github.com/jowi24/gutmann-sombra-pcb) –
  schematic, KiCad PCB, fabrication files and a
  [description of the circuit](https://github.com/jowi24/gutmann-sombra-pcb/blob/main/docs/CIRCUIT.md).
- **Plain Arduino variant:** for anyone who does not want to use ESPHome,
  [gutmann-sombra-mqtt](https://github.com/jowi24/gutmann-sombra-mqtt)
  contains an Arduino/PlatformIO firmware for the same board that talks
  MQTT with Home Assistant discovery. It is no longer actively developed;
  new features go into this repository.

## How it works

The time-critical part cannot be built from YAML: the hood scans its key
matrix in 1 ms steps (see
[CIRCUIT.md](https://github.com/jowi24/gutmann-sombra-pcb/blob/main/docs/CIRCUIT.md),
section 3), and ESPHome's main loop is far too slow and jittery for that.
The external component [`sombra_hood`](components/sombra_hood) therefore
contains the interrupt logic of the Arduino firmware almost unchanged:

- An interrupt on the SCAN line (GPIO13) resynchronises roughly every 50 ms
  on the start of a 4 ms scan round.
- The hardware timer `timer1` fires every 100 µs and, in the right window,
  reads the LED columns or drives the key columns HIGH.
- `loop()` debounces the LED states (80 ms, 1.5 s when the blinking filter
  indicator goes off) and reports them once they have been stable for
  200 ms, so the speed steps of the run-on timer do not show up as a short
  "off".

Everything else (Wi-Fi with setup hotspot, OTA, status LED, diagnostics) is
handled by ESPHome itself.

## Entities

| Platform | Default name | Function |
|---|---|---|
| `fan` | Lüftung | Speeds 1–4 (25/50/75/100 %), off |
| `light` | Haubenlicht | Light |
| `switch` | Nachlauf-Timer | Run-on timer |
| `binary_sensor` | Filterreinigung | Filter cleaning indicator (`problem`) |
| `button` | Filteranzeige zurücksetzen | Resets the indicator (6 s key press, only while it is on) |

The states always come from the front-panel LEDs. A command is stored as a
target state: the key is pressed at most once and only if the LEDs differ,
so two identical commands in quick succession cannot toggle twice. Every
2 s the entities are reconciled with the LEDs, so a command that got lost
does not leave a wrong state behind.

The hood's run-on timer steps the fan down on its own (e.g. 3 → 2 → 1);
the fan entity follows these steps.

## Building and flashing

```sh
cp secrets.yaml.example secrets.yaml   # fill in
esphome run hood-control.yaml
```

Without a local checkout, the component can be pulled from GitHub:

```yaml
external_components:
  - source: github://jowi24/gutmann-sombra-esphome@main
    components: [sombra_hood]
```

A new board is flashed serially the first time, see
[gutmann-sombra-pcb](https://github.com/jowi24/gutmann-sombra-pcb#first-flash).

## Switching from the Arduino firmware (and back)

Both firmwares can be swapped over the air in both directions; the hood
does not need to be opened. Both use the same flash layout (1 MB sketch,
2 MB LittleFS), so neither overwrites the other's settings: the Wi-Fi
credentials (SDK area) and the MQTT config of the Arduino firmware
(LittleFS) survive the round trip.

1. Build the rollback image first, while you are sure it matches what is
   installed (in a checkout of gutmann-sombra-mqtt, same `build_flags` as
   before):

   ```sh
   pio run -e nodemcuv2-ota
   ```

2. Switch to ESPHome via the Arduino OTA (no password):

   ```sh
   esphome compile hood-control.yaml
   python3 ~/.platformio/packages/framework-arduinoespressif8266/tools/espota.py \
     -i hood-control.local -f .esphome/build/hood-control/.pioenvs/hood-control/firmware.bin
   ```

   Home Assistant discovers the new ESPHome device (API encryption key from
   `secrets.yaml`). The MQTT entities become unavailable meanwhile, and
   automations using them do nothing.

3. Test: `esphome logs hood-control.yaml` shows `Synchronised with the hood`
   and every LED change. Compare with the front panel, then switch speed,
   light and timer from Home Assistant.

4. To stay with ESPHome, remove the old MQTT device in Home Assistant by
   clearing the retained topics `homeassistant/+/hood-control_*/config`,
   then rename the new entities to the old entity ids so automations keep
   working.

   To switch back instead, use the ESPHome OTA:

   ```sh
   esphome upload hood-control.yaml --device hood-control.local \
     --file path/to/gutmann-sombra-mqtt/.pio/build/nodemcuv2-ota/firmware.bin
   ```

   Then delete the ESPHome device in Home Assistant; the MQTT entities come
   back on their own.

Safety nets if something goes wrong:

- After 10 failed boots in a row ESPHome starts in *safe mode* with only
  Wi-Fi and OTA, so switching back still works.
- Without Wi-Fi, ESPHome opens the `HoodControl-Setup` hotspot; its portal
  at `http://192.168.4.1` also accepts a firmware upload.
- Until the firmware drives them, the key outputs are held LOW by R7/R8, so
  a crashing firmware cannot press keys.
- Last resort: serial flashing via J2 with J5 set to FLASH – this requires
  opening the hood.

## Notes

- `timer1` is used exclusively by the component. Do not combine it with
  components that also use `timer1` on the ESP8266 (e.g. `esp8266_pwm`,
  `ac_dimmer`, `opentherm`).
- During an OTA update the timer and the SCAN interrupt are paused
  automatically.
- Not ported from the Arduino firmware: event buffer, boot stage and MQTT
  error counters. ESPHome's log (`esphome logs`) and the `debug` component
  take over this role. If MQTT is wanted, add the `mqtt:` component to the
  YAML.

## License

Copyright (c) 2017–2026 Joachim Wilke.

- **Firmware** (`components/`, `*.yaml`): [MIT](LICENSE)
- **Documentation** (Markdown files): [CC BY-NC-SA 4.0](LICENSE-DOCS)

The hardware in gutmann-sombra-pcb is free for non-commercial use only.
