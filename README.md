# Powerpal BLE (Enhanced ESPHome Component)

This repository provides an enhanced ESPHome external component for integrating a **Powerpal energy monitor** over BLE with Home Assistant.

It combines and extends:

- https://github.com/gurrier/esphome-powerpal_ble
- https://github.com/SleepinDevil/esphome-powerpal_ble

---

# ✅ Features

## Core Functionality
- ✅ Live power readings
- ✅ Daily energy tracking
- ✅ Total energy accumulation
- ✅ Pulse and watt-hour tracking
- ✅ Cost calculation

## Enhancements in this fork
- ✅ Modern ESPHome compatibility (ESP-IDF)
- ✅ Required `esp32_ble_tracker` support
- ✅ BLE auto-reconnect
- ✅ Improved WiFi/BLE stability
- ✅ API key retrieval from device
- ✅ Manual RTC sync via BLE
- ✅ Automatic RTC sync on boot (with watchdog protection)
- ✅ Periodic RTC drift correction

---

# 🚀 Installation

Add this to your ESPHome YAML:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/<YOUR_USERNAME>/esphome-powerpal_ble.git
      ref: main
    components: [ powerpal_ble ]

⚙️ Example Configuration
esphome:
  name: powerpal-gateway
  friendly_name: Powerpal Gateway
  on_boot:
    priority: -100
    then:
      - script.execute: sync_powerpal_time

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:

api:
  encryption:
    key: "YOUR_API_KEY"

ota:
  password: "YOUR_OTA_PASSWORD"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none

esp32_ble_tracker:
  scan_parameters:
    interval: 1100ms
    window: 1100ms
    active: true

time:
  - platform: homeassistant
    id: homeassistant_time

ble_client:
  - mac_address: XX:XX:XX:XX:XX:XX
    id: powerpal
    auto_connect: true

sensor:
  - platform: powerpal_ble
    id: powerpal_ble_sensor
    ble_client_id: powerpal

    power:
      name: "Powerpal Power"

    battery_level:
      name: "Powerpal Battery"

    daily_energy:
      name: "Powerpal Daily Energy"
      unit_of_measurement: kWh
      device_class: energy
      state_class: total_increasing

    energy:
      name: "Powerpal Total Energy"
      unit_of_measurement: kWh
      device_class: energy
      state_class: total_increasing

    pulses:
      name: "Powerpal Pulses"

    watt_hours:
      name: "Powerpal Watt Hours"

    timestamp:
      name: "Powerpal Timestamp"

    cost:
      name: "Powerpal Cost"

    pairing_code: 123456
    pulses_per_kwh: 1000
    notification_interval: 1
    time_id: homeassistant_time
    cost_per_kwh: 0.20

text_sensor:
  - platform: template
    name: "Powerpal API Key"
    id: powerpal_api_key

button:
  - platform: restart
    name: "Restart"

  - platform: template
    name: "Powerpal: Set Device Time"
    on_press:
      - ble_client.ble_write:
          id: powerpal
          service_uuid: '59DAABCD-12F4-25A6-7D4F-55961DCE4205'
          characteristic_uuid: '59DA0004-12F4-25A6-7D4F-55961DCE4205'
          value: !lambda |-
            uint32_t t = id(homeassistant_time).now().timestamp;
            return std::vector<uint8_t>{
              (uint8_t)(t),
              (uint8_t)(t >> 8),
              (uint8_t)(t >> 16),
              (uint8_t)(t >> 24)
            };

  - platform: template
    name: "Retrieve API Key"
    on_press:
      then:
        - lambda: |-
            std::string api_key = id(powerpal_ble_sensor).get_apikey();
            id(powerpal_api_key).publish_state(api_key);

interval:
  - interval: 6h
    then:
      - script.execute: sync_powerpal_time


🔄 RTC Auto Sync Script
script:
  - id: sync_powerpal_time
    mode: restart
    then:

      - wait_until:
          condition:
            lambda: 'return id(homeassistant_time).now().is_valid();'
          timeout: 30s

      - wait_until:
          condition:
            lambda: 'return id(powerpal).is_connected();'
          timeout: 30s

      - if:
          condition:
            lambda: |-
              return id(homeassistant_time).now().is_valid() &&
                     id(powerpal).is_connected();
          then:
            - logger.log: "Syncing Powerpal RTC time"

            - ble_client.ble_write:
                id: powerpal
                service_uuid: '59DAABCD-12F4-25A6-7D4F-55961DCE4205'
                characteristic_uuid: '59DA0004-12F4-25A6-7D4F-55961DCE4205'
                value: !lambda |-
                  uint32_t t = id(homeassistant_time).now().timestamp;
                  return std::vector<uint8_t>{
                    (uint8_t)(t),
                    (uint8_t)(t >> 8),
                    (uint8_t)(t >> 16),
                    (uint8_t)(t >> 24)
                  };

          else:
            - logger.log: "Skipping RTC sync (timeout or not connected)"
            - delay: 2min
            - script.execute: sync_powerpal_time

⚠️ Important Notes
Energy Dashboard Compatibility
You MUST define the following in YAML:

device_class: energy
state_class: total_increasing
unit_of_measurement: kWh

These are no longer set automatically in the component.

BLE Stability Tips

Keep power_save_mode: none
Maintain strong signal between ESP32 and Powerpal
Avoid busy WiFi environments where possible


RTC Sync Behaviour

Runs automatically on boot
Waits for:

Home Assistant time
BLE connection


Times out safely (no boot blocking)
Retries if conditions are not met
Periodically re-syncs every 6 hours

🧪 Debugging
Enable debug logging:
logger:
  level: DEBUG
  logs:
    powerpal_ble: DEBUG

🧭 Future Improvements

BLE connection watchdog / auto-recovery
Data smoothing / spike filtering
Auto RTC sync on successful reconnect
MQTT fallback support


🙌 Credits

Original component: gurrier
Enhancements: SleepinDevil
Fork + stability improvements: skinnybuddah


⚡ License
Same as upstream repository.