🇬🇧 **English** · 🇩🇪 [Deutsch](CHANGELOG.de.md)

# Changelog

## 0.1.0 - 2026-11-21

First release. Ported from the Arduino firmware 0.0.11 in
[gutmann-sombra-mqtt](https://github.com/jowi24/gutmann-sombra-mqtt).

- External component `sombra_hood` with fan, light, switch, binary sensor
  and button platforms.
- Commands are stored as target states; each key is pressed at most once
  per command.
- LED changes are reported once they have been stable for 200 ms.
- Same flash layout as the Arduino firmware, so both can be swapped over
  the air without losing settings.
