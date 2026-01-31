# WadESP-PowerSW

Lightweight and stable ESP8266-based **remote Power Switch controller** for desktop PCs.  
Designed to work with **Wadboard** (or any HTTP-capable system) over a local network.

This project simulates a **physical press of the motherboard PowerSW button** using a relay, controlled via HTTP.
<details>
<summary>Expand Scheme</summary>
  
<img width="4284" height="2324" alt="image" src="https://github.com/user-attachments/assets/e256b96a-ae6a-46b1-a340-1bc5a7c20a9a" />
</details>


## Features

- ESP8266 (Wemos D1 mini / NodeMCU compatible)
- Simple HTTP API (`/power/on`, `/power/off`, `/status`)
- Non-blocking relay pulse logic
- Wi-Fi auto-reconnect
- No MQTT, no cloud, no dependencies
- Designed for **local LAN usage**
- Stable long-term operation (no freezes after hours/days)

---

## How It Works

1. ESP8266 connects to your Wi-Fi network
2. A minimal HTTP server listens on port `80`
3. When a request is received:
   - ESP briefly activates a relay (`~200 ms`)
   - Relay closes the **PowerSW** pins on the motherboard
4. This emulates a real power button press:
   - PC OFF → turns ON
   - PC ON → shuts down (depends on OS settings)

**PC power state detection is done externally**  
(e.g. by pinging the PC IP from Wadboard).

---

## Hardware Requirements

- ESP8266 board  
  (Wemos D1 mini recommended)
- 1-channel relay module (5V or 3.3V logic)
- Dupont wires
- Desktop PC motherboard with standard **PowerSW pins**

---

## Wiring

### ESP8266 → Relay
- `D2 (GPIO4)` → Relay **S / IN**
- `5V` → Relay **VCC**
- `GND` → Relay **GND**

### Relay → Motherboard
- Relay **COM** → PowerSW pin 1
- Relay **NO**  → PowerSW pin 2

⚠️ Polarity does **not** matter for PowerSW.



---

## Firmware

### Full Stable Firmware (ESP8266)

<details>
<summary>Show firmware code</summary>

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>

/* ===== CONFIG ===== */
const char* ssid = "Your Network SSID";
const char* password = "Network PSK";

const int relayPin = 4;                  // D2
const unsigned long PULSE_MS = 200;      // Relay pulse duration
/* ================== */

ESP8266WebServer server(80);

/* ===== RELAY STATE ===== */
bool relayActive = false;
unsigned long relayStart = 0;
/* ====================== */

void startPulse() {
  if (!relayActive) {
    digitalWrite(relayPin, HIGH);
    relayActive = true;
    relayStart = millis();
  }
}

/* ===== HTTP HANDLERS ===== */

void handleOn() {
  startPulse();
  server.sendHeader("Connection", "close");
  server.send(200, "application/json", "{\"status\":\"pulse\"}");
}

void handleOff() {
  startPulse();
  server.sendHeader("Connection", "close");
  server.send(200, "application/json", "{\"status\":\"pulse\"}");
}

void handleStatus() {
  server.sendHeader("Connection", "close");
  server.send(200, "application/json", "{\"status\":\"idle\"}");
}

void handleNotFound() {
  server.sendHeader("Connection", "close");
  server.send(404, "text/plain", "Not found");
}

/* ========================= */

void ensureWiFi() {
  if (WiFi.status() != WL_CONNECTED) {
    WiFi.disconnect();
    WiFi.begin(ssid, password);

    unsigned long start = millis();
    while (WiFi.status() != WL_CONNECTED && millis() - start < 5000) {
      delay(100);
      yield();
    }
  }
}

void setup() {
  pinMode(relayPin, OUTPUT);
  digitalWrite(relayPin, LOW);

  WiFi.mode(WIFI_STA);
  WiFi.setSleepMode(WIFI_NONE_SLEEP);
  WiFi.setAutoReconnect(true);
  WiFi.persistent(false);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    yield();
  }

  server.on("/power/on", HTTP_ANY, handleOn);
  server.on("/power/off", HTTP_ANY, handleOff);
  server.on("/status", HTTP_ANY, handleStatus);
  server.onNotFound(handleNotFound);

  server.begin();
}

void loop() {
  server.handleClient();
  ensureWiFi();

  if (relayActive && millis() - relayStart >= PULSE_MS) {
    digitalWrite(relayPin, LOW);
    relayActive = false;
  }

  yield();
}
```
</details>

## Integration with Wadboard

### Recommended logic on Wadboard side:
1. Simply add new WOL item and choose Power-SW
2. Set IP of ESP8266 (you can find it form Serial Monitor in Arduino IDE)

⚠️ It's strongly recommended to make IP of ESP8266 static from your router

## Stability Notes (Important)

This firmware avoids common ESP8266 pitfalls that commonly cause freezes or unresponsive HTTP servers after some uptime.

### What is intentionally avoided

- ❌ No blocking `delay()` during relay pulse
- ❌ No HTTP keep-alive connections
- ❌ No hanging or reused TCP sockets

### What is explicitly enforced

- ✅ Explicit `Connection: close` on every HTTP response
- ✅ Wi-Fi auto-reconnect enabled
- ✅ Watchdog-friendly non-blocking `loop()`

This design allows the device to run for long periods without requiring reboots or manual intervention.

## What This Project Does NOT Do

This project is intentionally minimal and focused.

- ❌ Does not detect PC power state via hardware (LEDs, PSU signals, etc.)
- ❌ Does not expose a web UI
- ❌ Does not use MQTT or any cloud services
- ❌ Does not manage users, authentication, or authorization

All decision-making logic belongs in **Wadboard** or another external controller.

## Minimal Wiring Diagram (ASCII)

ESP8266 (Wemos D1 mini)          Relay Module                Motherboard
-----------------------         ------------                ------------
        D2 (GPIO4)  ----------> IN (S)                       
        5V          ----------> VCC                          
        GND         ----------> GND                          

                                                      COM ---+--- PowerSW
                                                       NO ---+--- PowerSW


## Logical Flow (Minimal Diagram)

[ Wadboard ] —> HTTP request (/power/on) —> [ ESP8266 ]  —> 200 ms relay pulse —> [ Relay ] —> Short contact —> [ Motherboard PowerSW ]
