# WiFi_Hacking — ESP32 Beacon Broadcasting

## 1. Project Overview

This project demonstrates Wi-Fi beacon frame generation and broadcasting using an ESP32 development board.

It uses low-level IEEE 802.11 frame transmission to broadcast multiple custom Wi-Fi network names (SSIDs) without creating conventional Wi-Fi access points.

The project is designed to understand:
- Wi-Fi management frames.
- IEEE 802.11 beacon frames.
- SSID broadcasting.
- MAC address manipulation.
- Wi-Fi channel selection and hopping.
- Raw 802.11 packet transmission.
- WPA2 information elements.

**Attack category:** Wireless Management Frame Manipulation  
**Technique:** Beacon Flooding / SSID Broadcasting  
**Hardware:** ESP32  
**Language:** C++ (Arduino Framework)

---

## 2. How Does It Work?

Normally, a Wi-Fi router periodically broadcasts beacon frames to announce its presence to nearby devices.

A beacon frame contains information such as:
- Network SSID.
- BSSID (access point MAC address).
- Supported data rates.
- Operating channel.
- Security capabilities.
- Beacon interval.

This project constructs beacon frames manually and transmits them using the ESP32's Wi-Fi hardware interface.

Instead of advertising one network name, it iterates through a predefined list of SSIDs and broadcasts beacon frames containing different network names.

### Working architecture

```text
          ESP32
            |
            v
      Initialize Wi-Fi
       Station Mode
            |
            v
     Configure Channels
        1, 6, 11
            |
            v
      Load SSID List
            |
            v
      Construct Beacon
          Frame
            |
            v
     Set MAC / BSSID
            |
            v
      Set Wi-Fi Channel
            |
            v
     Transmit 802.11 Frame
            |
            v
      Repeat for SSIDs
            |
            v
       Channel Rotation
            |
            v
       Packet Statistics
```

---

## 3. Code Explanation

### A. Configuration Settings

```cpp
const uint8_t channels[] = {1, 6, 11};
const bool wpa2 = true;
const bool appendSpaces = true;
```

**channels[]**

Defines the Wi-Fi channels used by the program.

- Channel 1
- Channel 6
- Channel 11

**wpa2**

Controls whether WPA2-related security information is included in the beacon frame.

- `true`: Includes an RSN information element describing WPA2.
- `false`: Omits the RSN information element.

**appendSpaces**

Controls whether SSIDs are padded to 32 characters.

- `true`: Uses a fixed-length SSID field.
- `false`: Constructs a packet using the actual SSID length.

### B. SSID Configuration

```cpp
const char ssids[] PROGMEM = {
    "Mom Use This One\n"
    "Abraham Linksys\n"
    "LAN Solo\n"
};
```

This section contains the network names to be advertised.

Important concepts:

- `PROGMEM`: Stores constant data in program memory.
- `\n`: Separates individual SSID entries.
- Maximum SSID length: 32 bytes.

The program reads each SSID individually and inserts it into the beacon frame.

### C. Wi-Fi Libraries

```cpp
#include "WiFi.h"
#include "esp_wifi.h"
```

These libraries provide access to ESP32 Wi-Fi functionality.

The following functions are particularly important:

| Function | Purpose |
|---|---|
| `WiFi.mode()` | Configures Wi-Fi operating mode |
| `esp_wifi_set_channel()` | Selects the Wi-Fi channel |
| `esp_wifi_80211_tx()` | Provides raw 802.11 frame transmission |
| `random()` | Generates pseudorandom values |
| `memcpy()` | Copies memory |
| `memcpy_P()` | Copies data from program memory |

### D. Beacon Frame Structure

```cpp
uint8_t beaconPacket[109];
```

The beacon packet is manually constructed as a byte array.

Its important sections are:

| Offset | Field | Description |
|---|---|---|
| 0–3 | Frame Control | Identifies a management beacon frame |
| 4–9 | Destination | Broadcast MAC address |
| 10–15 | Source | Transmitting MAC address |
| 16–21 | BSSID | Network identifier MAC address |
| 22–23 | Sequence | Frame sequence information |
| 24–31 | Timestamp | Beacon timestamp |
| 32–33 | Beacon Interval | Beacon transmission interval |
| 34–35 | Capabilities | Network capability information |
| 36–69 | SSID | Network name information |
| 70–79 | Supported Rates | Supported Wi-Fi data rates |
| 80–82 | Channel | Operating channel information |
| 83–108 | RSN Information | WPA2 security information |

**Important:** These offsets describe this particular 109-byte frame layout.

### E. Channel Switching

```cpp
void nextChannel()
```

This function manages the configured channel list.

It maintains a channel index and selects the next channel from the array.

The purpose is to demonstrate channel rotation and how Wi-Fi devices operate across different radio channels.

### F. MAC Address Generation

```cpp
void randomMac()
```

This function generates pseudorandom values for the six-byte MAC address.

The generated address is used in the source and BSSID fields of the beacon frame.

Note that the final MAC byte is subsequently modified by the SSID counter.

### G. Setup Function

```cpp
void setup()
```

The setup function initializes the project.

Its responsibilities include:

1. Initializing serial communication at 115200 baud.
2. Initializing the empty SSID buffer.
3. Configuring packet size.
4. Generating the initial MAC address.
5. Setting ESP32 Wi-Fi mode to station mode.
6. Selecting the initial Wi-Fi channel.
7. Printing configured SSIDs to the serial monitor.

### H. Main Loop

```cpp
void loop()
```

The main loop continuously manages the beacon-generation process.

Its responsibilities include:

1. Checking elapsed time using `millis()`.
2. Processing the SSID list.
3. Selecting the current channel.
4. Updating the beacon frame with the SSID.
5. Updating the channel information.
6. Maintaining packet statistics.

The program also maintains a packet counter to report successful transmission API calls per second.

---

## 4. Important Technical Concepts

| Concept | Meaning |
|---|---|
| SSID | Wi-Fi network name |
| BSSID | MAC address identifying a wireless network |
| Beacon Frame | Management frame advertising a wireless network |
| MAC Address | Hardware-level network identifier |
| IEEE 802.11 | Wireless LAN communication standard |
| RSN | Robust Security Network information |
| WPA2 | Wi-Fi security protocol |
| Channel Hopping | Switching between Wi-Fi channels |
| Raw Packet Injection | Transmitting manually constructed network frames |
| PROGMEM | Storage of constant data in program memory |

---

## 5. Expected Behavior

When the ESP32 runs the program:

- It initializes its Wi-Fi interface.
- It loads the predefined SSID list.
- It constructs beacon frames.
- It cycles through the configured channel list.
- It advertises different SSID values.
- It displays packet submission statistics through Serial Monitor.

Nearby wireless scanners may display multiple advertised network names.

**Important:** These advertisements do not represent fully operational access points. They do not provide internet connectivity, authenticate clients, or establish a functional Wi-Fi network.

---

## 6. Limitations and Code Review

Some implementation details should be corrected or reviewed:

1. **Channel index:** The condition `channelIndex > sizeof(channels)` should use the number of array elements and a `>=` boundary check to prevent out-of-bounds access.
2. **Memory management:** `new uint8_t[]` must be paired with `delete[]`, not `delete`.
3. **Randomness:** `randomSeed(1)` produces a predictable pseudorandom sequence rather than secure randomness.
4. **MAC address:** The generated MAC address does not explicitly enforce locally administered unicast address bits.
5. **SSID parsing:** Every SSID should end with a newline and contain no more than 32 bytes.
6. **Packet statistics:** The counter measures successful API submissions, not independently verified over-the-air transmissions.

---

## 7. Security and Responsible Research

This project is intended for wireless security education and controlled laboratory research.

Use only personally owned equipment or networks for which you have explicit authorization.

Beacon flooding can interfere with wireless network discovery and monitoring. Do not conduct experiments in public places, on shared networks, or against networks belonging to other people.

---

## 8. Learning Roadmap

This project is an introduction to wireless management frames.

Further topics in the WiFi_Hacking research project include:

1. Wi-Fi fundamentals and IEEE 802.11.
2. Beacon frame analysis.
3. Probe requests and probe responses.
4. Monitor mode and packet analysis.
5. ESP32 Wi-Fi capabilities.
6. Rogue access point concepts.
7. WPA/WPA2 security analysis.
8. Handshake and PMKID security research.
9. Deauthentication attack analysis.
10. Evil Twin attack architecture.
11. Captive portal security.
12. Wireless intrusion detection and prevention.

**Project Philosophy:**

Understand the protocol → Study the vulnerability → Experiment in an isolated lab → Analyze the results → Learn how to detect and defend against the attack.