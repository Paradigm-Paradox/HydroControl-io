# HydroControl-io: System Architecture & Hardware Specification

> **Platform:** Home Assistant Add-on  
> **Repository:** [HydroControl-io](https://github.com/Paradigm-Paradox/HydroControl-io)  
> **Edge Microcontroller Platform:** ESP32 (Wi-Fi, Ethernet, Bluetooth)  
> **Target Systems:** DWC, NFT, Ebb & Flow (Flood & Drain), Aeroponics, Automated Reservoirs

---

## 1. System Vision & Scope

**HydroControl-io** provides an edge-native, automated closed-loop management platform for indoor and greenhouse hydroponic setups. By combining the low cost, energy efficiency, and dual wireless/wired networking of **Espressif ESP32** microcontrollers with Home Assistant's high-level intelligence and Ingress dashboard, HydroControl-io automates:
- Real-time pH balancing and nutrient EC dosing with strict anti-shock mixing intervals.
- Reservoir level monitoring and fail-safe Automated Top-Off (ATO).
- High-efficiency pump scheduling and duty-cycle irrigation.
- Temperature regulation via inline chillers and immersion heaters.
- Bluetooth proxying for BLE soil and canopy sensors.

---

## 2. Hardware Architecture & The ESP32 Ecosystem

The ESP32 family by Espressif Systems serves as the physical hardware interface layer for HydroControl-io.

```
                           ┌──────────────────────────────────────────────┐
                           │          HydroControl-io Add-on              │
                           │   (FastAPI + Home Assistant Ingress UI)      │
                           └──────────────────────┬───────────────────────┘
                                                  │
                 ┌────────────────────────────────┴───────────────────────────────┐
                 │ Native Home Assistant API / Encrypted Noise Protocol / MQTT    │
                 ▼                                                               ▼
┌─────────────────────────────────┐                             ┌─────────────────────────────────┐
│     ESP32 Wi-Fi + BLE Node      │                             │       ESP32 Ethernet Node       │
│     (ESP-WROOM-32 / S3)         │                             │     (WT32-ETH01 / Olimex PoE)   │
│                                 │                             │                                 │
│  - 2.4GHz 802.11 b/g/n Wi-Fi    │                             │  - 10/100M LAN8720 RMII PHY     │
│  - Active Bluetooth LE Proxy    │                             │  - Power-over-Ethernet (PoE)    │
│  - DS18B20 Temp Probe (1-Wire)  │                             │  - Zero RF Interference         │
│  - JSN-SR04T Ultrasonic Level   │                             │  - ADS1115 I2C 16-Bit ADC       │
│  - ADS1115 I2C Isolated ADC     │                             │  - Galvanic Isolated Probes     │
│  - 4x Peristaltic Pump Relays   │                             │  - Peristaltic & Main Pumps     │
│  - High/Low Float Safeguards    │                             │  - High/Low Float Safeguards    │
└─────────────────────────────────┘                             └─────────────────────────────────┘
```

### 2.1 ESP32 Microcontroller Selection
1. **ESP32 Wi-Fi + Bluetooth (Standard Edition):**
   - Utilizes dual-core Tensilica Xtensa 32-bit LX6 running at 240 MHz.
   - Built-in Wi-Fi connects directly to your Home Assistant wireless IoT VLAN.
   - Built-in Bluetooth LE operates as an active **Home Assistant Bluetooth Proxy**, capturing data from wireless BLE plant sensors (e.g. Xiaomi Flora, RuuviTag, Inkbird temp/humidity).
2. **ESP32 Ethernet / PoE (Industrial Edition):**
   - Built around the LAN8720 or W5500 Ethernet transceiver (e.g. WT32-ETH01, Olimex ESP32-POE, or QuinLED-ESP32-Ethernet).
   - Immune to high-frequency RF interference caused by electronic grow light ballasts and high-voltage pump relays.
   - Single-cable deployment delivering power (802.3af PoE) and bidirectional telemetry directly to the reservoir.

---

## 3. Sensor & Probe Electrical Isolation

Hydroponic nutrient solutions are highly conductive ionic electrolytes. When multiple submerged probes (pH, EC, dissolved oxygen) share the same water volume, ground loops produce severe measurement drift, noise, and accelerated electrode oxidation.

HydroControl-io specifies:
1. **Galvanic Isolation:** The ADS1115 16-bit analog-to-digital converter is isolated from the main ESP32 3.3V rail using an I2C digital isolator (e.g. ADuM1250) and an isolated DC-DC converter (e.g. B0505S).
2. **Probes Supported:**
   - **pH Probes:** Standard analog BNC probes or Atlas Scientific EZO-pH over I2C.
   - **EC Probes:** K=1.0 platinum/graphite probes with temperature compensation or Atlas Scientific EZO-EC.
   - **Water Temperature:** Dallas DS18B20 waterproof stainless steel probes (1-Wire bus with 4.7kΩ pull-up).
   - **Water Level:** JSN-SR04T waterproof ultrasonic transducer mounted on the reservoir lid, backed by dual mechanical stainless float switches for hardware-level overflow and dry-run cutoffs.

---

## 4. Closed-Loop Control & Safety Engine

### 4.1 Proportional Pulse-Pause Dosing
Chemical shock causes root burn and severe nutrient precipitation. HydroControl-io executes dosing via a calibrated pulse-pause sequence:

$$\text{Dose Volume (mL)} = \min\left( K_p \times |X_{\text{current}} - X_{\text{target}} |, \text{MaxSingleDose} \right)$$

$$\text{Pump Run Duration (s)} = \frac{\text{Dose Volume (mL)}}{\text{Pump Flow Rate (mL/min)}} \times 60$$

- **Mixing Delay:** Following pump actuation, all dosing is locked out for a configurable dwell period (default 900 seconds / 15 minutes) while the circulation pump distributes the chemicals evenly.
- **Nutrient Stagger:** Part A and Part B are dosed with a minimum 5-second interval between dispenses to prevent concentrated mineral lockout.

### 4.2 Multi-Tier Safety Guards
1. **Rolling 24-Hour Volumetric Cap:** A hard limit (default: 100 mL/day) prevents fatal overdosing in the event of sensor failure or siphoning.
2. **Sensor Sanity Boundary:** Dosing is automatically inhibited if pH < 3.0 or pH > 9.5, or if EC > 4.5 mS/cm.
3. **Dry-Run Interlock:** If the low-float switch triggers or water level drops below 15%, all dosing and heater elements are immediately de-energized.
4. **Emergency Stop (E-Stop):** A physical or software latch instantly brings all relays to a normally-open state.

---

## 5. Firmware Configurations (ESPHome)

Pre-built firmware files are provided in [`esphome/`](esphome/):
- [`esphome/esp32_wifi_reservoir.yaml`](esphome/esp32_wifi_reservoir.yaml): Full Wi-Fi, BLE proxy, ADS1115 ADC, DS18B20, and 5-channel relay configuration.
- [`esphome/esp32_ethernet_reservoir.yaml`](esphome/esp32_ethernet_reservoir.yaml): Hardwired LAN8720 Ethernet POE reservoir controller.
