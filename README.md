# MyRoutes-ESP32-S3-LCD-3.16
MyRoutes London Bus Arrivals on ESP32
Link to device Wiki: https://www.waveshare.com/wiki/ESP32-S3-LCD-3.16
Link to My Routes Android App: https://play.google.com/store/apps/details?id=com.htfl.myroutes
Serial terminal: https://googlechromelabs.github.io/serial-terminal/
Lessons learnt:

# Lessons Learned: ESP32 Bus Tracker Development

This document compiles the key technical challenges ("gotchas") and solutions we discovered while building the **MyRoutes-ESP32** firmware. It serves as a guide for anyone developing similar graphics-heavy, internet-connected applications on the ESP32-S3.

## 1. Display Driver & Graphics (Waveshare ESP32-S3-Touch-LCD-4.3)

### The "Dirty Green" & Dim Display Issue
**Symtoms**: The detailed display looked washed out, dim, and had a greenish tint, even when setting "White".
**The Fix**:
1.  **PWM Setup is Critical**: We couldn't just use `digitalWrite(PIN_LCD_BL, HIGH)`. We had to use the ESP32 `ledc` (LED Control) API.
2.  **Inverted Logic**: This was the main "gotcha". For this specific hardware, the PWM logic is **inverted**.
    *   `0` duty cycle = **100% Brightness**
    *   `255` duty cycle = **0% Brightness (Off)**
3.  **Frequency**: We settled on **50kHz** (`50000`). Lower frequencies caused visible flickering or audible high-pitched complications.

### Initialization Sequence
**The Gotcha**: The ST7701 generic controller is extremely picky. Manufacturer example code often includes "patch" sequences that don't always work with every update of the ESP32 board definitions.
**The Solution**: We stuck to a "Known Good" initialization byte sequence (documented in `main.cpp`).
*   **Key Insight**: Reverting specific timing registers (like `0xE5`) to their default values was necessary to stabilize the image effectively.

### Flicker-Free UI Updates
**The Problem**: Clearing the screen (`fillScreen`) and redrawing everything for every second of the countdown caused massive flickering.
**The Solution**: **Partial Redraws**.
*   We separated the drawing logic into `drawBusList(bool fullUpdate)`.
*   **Full Update (`true`)**: Draws the bus line number, static text, dividers, and background. Done only when the bus list *content* changes (new API data).
*   **Partial Update (`false`)**: Done every second. It *only* redraws the countdown number.
*   **Technique**: Before drawing the new number, we draw a black rectangle *only over the area where the number sits*. This is much faster than clearing the whole screen.

## 2. Network Stability & SSL (The "Smart Client")

### The Heap Fragmentation Trap
**The Problem**: Creating a `new WiFiClientSecure` for every single HTTP request (every 30 seconds) rapidly fragmented the ESP32's heap RAM, eventually causing it to crash or fail to allocate SSL contexts after a few hours.
**The Solution**: **Smart Object Reuse**.
*   We use a global pointer: `WiFiClientSecure *secureClient = nullptr;`.
*   We only allocate it `new` if it doesn't exist.
*   We reuse this single instance for subsequent requests.

### The "Stale Connection" Deadlock
**The Problem**: Sometimes an SSL connection would drop silently or get stuck in a "connected" state that wouldn't actually transmit data, causing the requests to time out indefinitely.
**The Solution**: **Aggressive Self-Healing**.
*   If an HTTP request fails (returns an error code or connection refused), we don't just retry. We **destroy** the client.
    ```cpp
    delete secureClient;
    secureClient = nullptr;
    ```
*   This forces the firmware to build a completely fresh, clean SSL handshake on the very next cycle, clearing out any stuck states.

## 3. Configuration & User Experience

### Hardcoding vs. WiFiManager
**Lesson**: Never hardcode WiFi credentials. We used `WiFiManager` to create an automatic Access Point (`MyRoutes-Setup`) when WiFi is not found.
**Pro Tip**: You can inject *custom* HTML fields into the WiFiManager portal. We added fields for:
*   **Naptan ID** (The specific bus stop ID).
*   **Route Filter** (Comma-separated list of buses to show).
*   **Color Thresholds** (Customizing when the text turns Red/Amber).

### The "Panic Button" (Factory Reset)
**The Problem**: If a user enters a typo in their settings (or invalid JSON config), the device could boot loop or get stuck.
**The Solution**: **Physical Override**.
*   We monitored the **BOOT button (GPIO 0)** in the main loop.
*   **Logic**: If held for **5 seconds**, we trigger a full factory reset: cleans the `Preferences` storage and wipes the `WiFiManager` saved settings.
*   Visual Feedback: We draw a countdown on the screen ("Resetting in 3... 2... 1...") so the user knows exactly what is happening.

## 4. Time Synchronization
**The Gotcha**: Bus arrival times are relative to "Now". If the ESP32 thinks it's 1970, the math fails.
**The Solution**:
*   We use standard NTP (`configTime`).
*   **Crucial Step**: We added a `while(!getLocalTime)` loop in `setup()`. We *do not proceed* to the main logic until we have confirmed a valid time sync. This prevents showing negative arrival times or "DUE" for buses that are hours away.

## 5. ArduinoJSON & Memory
**Lesson**: Using the `ArduinoJson` library is great, but keep the `JsonDocument` scope small.
*   We declare `JsonDocument doc` *inside* the `fetchArrivals` function.
*   This ensures the memory is allocated on the stack (or heap for larger docs) and, most importantly, **freed immediately** when the function exits. Making this global is a recipe for memory leaks.

## Summary Checklist for New Projects
1.  [ ] **Check PWM Logic**: detailed control always beats simple GPIO toggling for backlights.
2.  [ ] **Reuse SSL Clients**: Don't `new/delete` heavy objects in a loop unless they break.
3.  [ ] **Partial UI Updates**: Never `fillScreen` inside a loop.
4.  [ ] **Escape Hatch**: Always code a physical buttons combination to reset settings.
5.  [ ] **Sync Time First**: Don't run logic dependent on `millis()` or `time()` until NTP returns success.
