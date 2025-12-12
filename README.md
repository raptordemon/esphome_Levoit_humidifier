# esphome_Levoit_humidifier
ESPHOME Yaml for Levoit Humidifiers

Flashing instrusctions and wiring diagrams coming soon

Support for the 600s and the 450s models via differnt yaml (they were similar but the 450s series has a differnt set of frames for control)

📘 Levoit LV600S Local UART Control (ESPHome Bridge)

Local, cloud-free control of the Levoit LV600S humidifier using an ESP32 over UART.
This project reverse-engineers the LV600S serial protocol and exposes full control through ESPHome + Home Assistant.

No VeSync cloud.
No app dependency.
Fast, reliable, local control.

⚠️ Disclaimer (Use at Your Own Risk)

This project is not affiliated with Levoit or VeSync.
It is based on reverse-engineered UART communication and is provided “AS IS”, with NO WARRANTY of any kind.

You are solely responsible for any damage to your LV600S, ESP32, or other equipment.

By using this firmware, you agree that the authors and contributors are not liable for any direct or indirect damages.

✨ Features
✔ Full Local Control (ESPHome → Humidifier)

Power On/Off

Display On/Off

Target Humidity (40–80%)

Cool Mist Level (1–9)

Warm Mist Level (0–3)

Sleep Mode (with automatic fan=9)

Manual Mode

Auto/Humidity Mode

Timer (0–720 minutes)

Automatic restoration of user settings when exiting Sleep/Manual

✔ Bi-Directional State Updates (Humidifier → ESPHome)

Tank present / removed

Out-of-water status

Display brightness state

Target humidity

Current humidity

Cool mist level

Warm mist level

Operating mode (manual, sleep, humidity)

Raw frame debugging output

✔ Home Assistant Auto-Discovery

Everything appears as native HA entities:

Switches

Sensors

Numbers

Text sensors


✔ No Custom Component Needed

Everything is implemented using:

uart:

debug:

lambda: code

Template entities

No external ESPHome components required.

🛠 Hardware Requirements

ESP32 (recommended: WROOM or WROVER)

Level-safe direct UART connection (LV600S uses 3.3V logic)

A spare USB port for ESP flashing

🔌 Wiring
LV600S Pin	ESP32 Pin	Description
Device TX	GPIO16	LV600S → ESP32
Device RX	GPIO17	ESP32 → LV600S
GND	GND	Common ground

Baud Rate: 9800 8N1

Make sure wires are secure — intermittent serial connections can cause strange behavior.

📨 UART Protocol Summary
Frame Format

All frames begin with:

A5 <origin> <id> <opcode> 00 <checksum> ...

Checksum Rule

A frame is valid if:

(sum of all bytes) & 0xFF == 0xFF

Origins
Byte	Meaning
0x02	Device → ESP32 (status)
0x12	Device → ESP32 (command ACK)
0x22	ESP32 → Device (command)
Key Commands (ESP → LV600S)
Feature	Command
Power	01 00 A0
Display	01 05 A1
Target Humidity	01 14 41
Cool Level	01 13 41
Warm Level	01 12 41
Sleep Mode	01 82 40
Mode Group	01 29 A1
Manual Mode Details	01 60 A2
Timer (seconds LE)	01 64 A2
Status Frame (Device → ESP32)

All begin with:

A5 02 <id> 18 00 …


Confirmed byte positions:

Index	Meaning
14	Tank (00=present, 01=removed)
15	Out-of-water (01=empty)
16	Display on/off
18	Target humidity
19	Current humidity
21	Mode (01=manual, 02=sleep, 03=humidity)
22	Cool mist level (1–9)
23	Warm mist level (0–3)

Full protocol documentation is in /docs/protocol.md.

🧠 ESPHome Firmware Features (Technical Detail)
Parsing Logic

All raw bytes captured via uart.debug

Static buffer accumulates

Frame boundary detection using checksum

Decodes useful fields

Publishes to:

template sensors

template switches

template numbers

Smart Behavior

Sleep mode forces cool level 9, but original level is saved & restored

Manual mode hides target humidity (LV600S behavior)

On exiting Manual → previous target restored

On exiting Sleep → previous cool level restored

All incoming changes (panel buttons or cloud/app) sync back to HA immediately



🙌 Credits

Protocol reverse-engineering and firmware design: With the help of ChatGPT (OpenAI)

Large portions of the protocol were mapped by iteratively capturing live UART traffic using a logic analyser and recreating command structures manually.

ESPHome inspiration:

Based on concepts used in:
tsquared96/ESPHome-Navien
acvigue/esphome-levoit-air-purifier

ESPHome community & documentation

If you use this work in your own projects, please consider linking back to this repository — it helps the community grow!
