🇬🇧 [English](CHANGELOG.md) · 🇩🇪 **Deutsch**

# Änderungsverlauf

## 0.1.0 - 2026-11-21

Erste Version, portiert aus der Arduino-Firmware 0.0.11 in
[gutmann-sombra-mqtt](https://github.com/jowi24/gutmann-sombra-mqtt).

- External Component `sombra_hood` mit Plattformen für Lüfter, Licht,
  Schalter, Binärsensor und Button.
- Befehle werden als Zielzustand gespeichert; jede Taste wird pro Befehl
  höchstens einmal gedrückt.
- LED-Änderungen werden gemeldet, sobald sie 200 ms stabil sind.
- Gleiches Flash-Layout wie die Arduino-Firmware, beide lassen sich ohne
  Verlust der Einstellungen per OTA tauschen.
