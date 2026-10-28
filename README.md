# NIU BMS over WebUSB

Read a NIU battery BMS (N1S protocol, RS485 9600 8E1) from Chrome on Android with a
CP2102 USB-RS485 dongle and a USB OTG adapter. No app to install.

**Open:** https://niu-tools.github.io/niu-bms-webusb/

- Live pack voltage, current, SOC, temperatures, cycles and all 17 cell voltages
- Cell balance rating (Δ < 50 mV good, 50–100 mV medium, > 100 mV bad)
- Battery selector with aliases; live view or the last stored reading per battery
- Health summary with suggested actions (balance, weak cell, voltage limits, temperature, sensing, wear)
- Statistics per battery and time range: first/last reading, span, counts, read success,
  pack/Δ/current/temperature averages and ranges, weakest cell, per-cell avg/min/max
- Voltage charts and CSV export
- All readings are kept on the phone (IndexedDB, persistent storage requested) until you
  clear them; nothing is uploaded anywhere

Wiring: dongle A → battery A, B → battery B, GND → battery GND.
