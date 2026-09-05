# ESPHome CRAFTY+

This ESPHome package connects to a STORZ & BICKEL CRAFTY+ over BLE and exposes its telemetry, settings, controls, and diagnostics to Home Assistant.

Add the package to an ESP32 configuration, replacing `OWNER` and the MAC address:

```yaml
substitutions:
  crafty_name: "CRAFTY+"
  crafty_mac_address: "AA:BB:CC:DD:EE:FF"

packages:
  crafty_plus: github://OWNER/esphome-storzundbickel-craftyplus/crafty-plus.yaml@main
```

Install the resulting configuration with ESPHome as usual.
