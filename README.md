# ESPHome CRAFTY+

[![ESPHome compile](https://github.com/valentinvoigt/esphome-storzundbickel-craftyplus/actions/workflows/esphome.yml/badge.svg)](https://github.com/valentinvoigt/esphome-storzundbickel-craftyplus/actions/workflows/esphome.yml)

This ESPHome package connects to a STORZ & BICKEL CRAFTY+ over BLE and exposes its telemetry, settings, controls, and diagnostics to Home Assistant.

Add the package to an ESP32 configuration, replacing name and the MAC address:
```yaml
substitutions:
  crafty_name: "CRAFTY+"
  crafty_mac_address: "AA:BB:CC:DD:EE:FF"

packages:
  crafty_plus: github://valentinvoigt/esphome-storzundbickel-craftyplus/crafty-plus.yaml@main
```
Install the resulting configuration with ESPHome as usual.

I had fun using an LLM to help me write repetetive code and do some minor changes. I did the reverse engineering myself. I read and understood every single line of code myelf.

I am not affiliated with STORZ & BICKEL in any way. I provide no warranty, especially regarding the device's more dangerous features.
