# ESPresense

![GitHub release (latest by date)](https://img.shields.io/github/v/release/ESPresense/ESPresense)
![GitHub all releases](https://img.shields.io/github/downloads/ESPresense/ESPresense/total)
[![.github/workflows/main.yml](https://github.com/ESPresense/ESPresense/actions/workflows/build.yml/badge.svg)](https://github.com/ESPresense/ESPresense/actions/workflows/build.yml)


An ESP32 based presence detection node for use with the [Home Assistant](https://www.home-assistant.io/) [`mqtt_room` component](https://www.home-assistant.io/components/sensor.mqtt_room/) or [ESPresense-companion](https://github.com/ESPresense/ESPresense-companion) for indoor positioning

**Documentation:** https://espresense.com/

**Building:** [building](./BUILDING.md).

**Release Notes:** [changelog](./CHANGELOG.md).

# Serial Forwarding Additions (ESP32-C3)

> _These modifications were created with assistance from ChatGPT — apologies for the AI-authored commits, but they were genuinely helpful!_

## Overview

These changes add **bidirectional serial forwarding** between MQTT and a hardware UART (UART1) on the **ESP32-C3** used in ESPresense.  
This allows the device to act as a bridge between an MQTT topic and another ESP32-based controller (for example, a WLED node).

---

## Summary of Changes

### 1. UART Configuration

Added a configuration block near the top of `main.cpp`:

```cpp
#define FORWARD_TOPIC "espresense/serial_out"
#define FORWARDED_SERIAL_TOPIC "forwarded_serial"
#define UART1_TX_PIN 7   // TX → other ESP32 RX
#define UART1_RX_PIN 6   // RX ← other ESP32 TX (optional)
#define UART_BAUD 115200
```

This defines:

- **`FORWARD_TOPIC`** — MQTT topic whose payloads are sent over the UART.  
- **`FORWARDED_SERIAL_TOPIC`** — MQTT topic where received UART messages are published.  
- **Pin assignments** for the UART1 TX/RX lines on the ESP32-C3 SuperMini.  
- **Baud rate** for communication (115200 bps).

---

### 2. UART Initialisation

In `setup()`:

```cpp
Serial1.begin(UART_BAUD, SERIAL_8N1, UART1_RX_PIN, UART1_TX_PIN);
```

Initialises **UART1** for serial communication with another microcontroller.

---

### 3. MQTT → Serial Forwarding

In `onMqttConnect()`:

```cpp
mqttClient.subscribe(FORWARD_TOPIC, 0);
```

Subscribes to a dedicated MQTT topic for serial forwarding.

At the top of `onMqttMessage()`:

```cpp
if (topic && strcmp(topic, FORWARD_TOPIC) == 0) {
    Serial1.println(payload ? payload : "");
    Log.printf("%d UART1  | %s\r\n", xPortGetCoreID(), payload ? payload : "");
    return;
}
```

Whenever an MQTT message arrives on `FORWARD_TOPIC`, the payload is written to the UART as plain text (terminated by a newline).  
This allows JSON commands (such as WLED control messages) to be forwarded to another device.

---

### 4. Serial → MQTT Forwarding

Added a new helper function `PumpSerial1ToMqtt()`:

```cpp
void PumpSerial1ToMqtt() {
    static String line;
    while (Serial1.available() > 0) {
        char c = (char)Serial1.read();
        if (c == '\r') continue;
        else if (c == '\n') {
            if (line.length() > 0 && mqttClient.connected()) {
                mqttClient.publish(FORWARDED_SERIAL_TOPIC, 0, false, line.c_str());
                Log.printf("%d MQTT -> %s: %s\r\n", xPortGetCoreID(),
                           FORWARDED_SERIAL_TOPIC, line.c_str());
            }
            line = "";
        } else {
            if (line.length() < 512) line += c;
        }
    }
}
```

Called once per iteration in the main `loop()`:

```cpp
PumpSerial1ToMqtt();
```

This function:

- Reads incoming text from UART1.  
- Buffers until a newline (`\n`) is received.  
- Publishes each complete line to the MQTT topic **`forwarded_serial`**.  

This enables data responses (for example, WLED JSON status replies) to flow back into MQTT automatically.

---

## Usage

- Publish a message to `espresense/serial_out`:

  ```bash
  mosquitto_pub -t espresense/serial_out     -m '{"on":true,"bri":255,"seg":[{"col":[[255,0,0]]}]}'
  ```

  → The JSON payload is forwarded via UART to the attached device.

- Any line received back on UART (terminated by `\n`) is automatically published to:

  ```
  forwarded_serial
  ```

  e.g. WLED status updates.

---

## Notes

- Designed specifically for **ESP32-C3** (which provides `Serial` for USB and `Serial1` for extra UART).  
- Pins **GPIO7 (TX)** and **GPIO6 (RX)** were chosen as safe hardware UART pins on the C3 SuperMini.  
- Ensure both devices share a **common GND**.  
- Baud rate and topic names can be modified in the config block.  

---

> _Again, sorry for the messy AI fingerprints — this addition was generated with ChatGPT’s help to quickly enable MQTT↔Serial bridging functionality._

