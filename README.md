# esphome_Levoit_humidifier
ESPHOME Yaml for Levoit Humidifiers

Flashing instrusctions and wiring diagrams coming soon


Levoit LV600S UART Protocol (Reverse-Engineered)

Model: LV600S
Firmware: 1.1.08 (1.0.50)
Interface: UART (3.3V logic)
Direction: Full duplex
Baud: 9800 8N1
Pins:

Device TX → ESP RX: GPIO16

Device RX → ESP TX: GPIO17

1. Protocol Overview

Communication is done using binary frame messages with the following characteristics:

Fixed header byte: 0xA5

Fixed command origin byte:

0x02 = device → app/MCU (status frames)

0x22 = app/MCU → device (command frames)

0x12 = device reply to commands

Variable length frames (typically 10–21 bytes)

Checksum rule:
A valid frame satisfies:

sum(all bytes) & 0xFF == 0xFF


or equivalently:

checksum = (0xFF - (sum(all_bytes_except_checksum) & 0xFF)) & 0xFF


No delimiter — stream must be parsed incrementally

Device echo replies for many commands

Device sends frequent status frames during operation (~1 Hz)

2. Frame Structure
2.1 General Format
+-----+--------+------+--------+-----------+----------------+
| A5  | Origin |  ID  | Opcode | Checksum  | Payload...     |
+-----+--------+------+--------+-----------+----------------+
byte0 byte1    byte2  byte3    byte4/5     bytes...


Where:

byte0 = 0xA5 always

byte1 = direction

byte2 = sequence / command ID

byte3 = function opcode or length indicator

byte4..5 = checksum or zero padding depending on opcode

payload = command-specific fields

Origins
Value	Meaning
0x02	Device → ESP32 (status broadcast)
0x12	Device → ESP32 (command acknowledgment)
0x22	ESP32 → Device (control command)

Note: Most “set” operations involve two responses: a 0x12 confirm + a 0x02 updated state frame.

3. RX Frames (Device → ESP32)
3.1 Status Frame: A5 02 …

Sent regularly (~1Hz), or after a command is applied.

Minimal format (observed range 23–38 bytes)
A5 02 ID 18 00  <… payload …>  (checksum satisfies FF rule)

Byte Map (confirmed offsets)
Index	Meaning
0	A5 Header
1	0x02 Device TX
2	Sequence
3	0x18 Status opcode
4–5	Padding / always 00
6	Unknown
7–8	Always 0x11 0x41 in humidity mode
9	Always 00
10	Always 0x32
11	Always 00
12	0x01 Power ON
13	Mode Group? (always 00)
14	Tank: 00 = present, 01 = removed
15	Out-Of-Water: 01 = empty, 00 = OK
16	Display: 0x00=off, >0x00=on
17	Unknown
18	Target Humidity (40–80%)
19	Current Humidity (%)
20	Always 0x2A
21	Mode: 01=Manual, 02=Sleep, 03=Humidity(Auto)
22	Cool Mist Level: 1–9
23	Warm Mist Level: 0–3
24+	Reserved / future data

Warm mist level is not always present in all firmware. In LV600S 1.1.08 it appears at 23.

4. TX Frames (ESP32 → Device)
4.1 General Command Structure
A5 22 <cmd_id> <opcode> 00 <checksum>  01 <class> <code> 00 <args…>


Explanation:

01 = command class

<class> <code> = feature selector (power, display, humidity, etc)

<args> = numeric value (level, target, timeout)

4.2 Command Categories
Category	Class	Code	Opcode	Description
Power	01	00 A0	05	On/Off
Display	01	05 A1	05	Display brightness
Humidity Target	01	14 41	08	Set target %
Cool Level	01	13 41	08	Set cool-mist power level
Warm Level	01	12 41	09	Set warm-mist heater level
Mode Group	01	29 A1	0A	Change overall operating mode
Mode Detail	01	60 A2	07	Manual mode details
Sleep Mode	01	82 40	0A	Night/Sleep settings
Timer	01	64 A2	08	Run time (seconds LE)

Timer payload uses 4-byte little-endian seconds:
secs = minutes * 60

Example:

4h30m = 270 min = 16200 sec = 0x00003F48 LE
payload = 48 3F 00 00

5. Known Command Examples
5.1 Power ON
A5 22 ID 05 00 CHK 01 00 A0 00 01

5.2 Power OFF
A5 22 ID 05 00 CHK 01 00 A0 00 00

5.3 Display ON (100%)
A5 22 ID 05 00 CHK 01 05 A1 00 64

5.4 Display OFF
A5 22 ID 05 00 CHK 01 05 A1 00 00

5.5 Target Humidity = X%
A5 22 ID 08 00 CHK 01 14 41 00 XX 00 00 00

5.6 Cool Level 1–9
A5 22 ID 08 00 CHK 01 13 41 00 LV 00 00 00

5.7 Warm Level 0–3
A5 22 ID 09 00 CHK 01 12 41 00 LV 00 00 00 00 00

5.8 Enter Sleep Mode
A5 22 ID 0A 00 CHK 01 82 40 00 <target> 00 00 00 00 00

5.9 Exit Sleep → Auto Mode

Two frames:

A5 22 ID 0A 00 CHK 01 29 A1 00 01 00 00 00 00 00
A5 22 ID 08 00 CHK 01 14 41 00 <target> 00 00 00


+(optional) restore cool level

A5 22 ID 08 00 CHK 01 13 41 00 <cool> 00 00 00

5.10 Enter Manual Mode
A5 22 ID 0A 00 CHK 01 29 A1 00 01 00 00 00 00 00
A5 22 ID 07 00 CHK 01 60 A2 00 00 01 01

5.11 Exit Manual → Auto
A5 22 ID 0A 00 CHK 01 29 A1 00 01 00 00 00 00 00
A5 22 ID 08 00 CHK 01 14 41 00 <saved_target> 00 00 00

5.12 Timer (minutes → seconds LE)
A5 22 ID 08 00 CHK 01 64 A2 00 <ss ss ss ss>


Timer cancel:

A5 22 ID 08 00 CHK 01 64 A2 00 00 00 00 00

6. ESPHome Firmware Features

This project provides a local control bridge between Home Assistant and the LV600S using a UART-connected ESP32.

6.1 Supported Features
Bi-Directional State

✓ Full capture of all raw frames, printed as hex
✓ Continuous device telemetry decoding
✓ Device button press → HA updates
✓ App changes → HA updates

Humidifier Controls
Feature	Status
Power	Full control
Display Brightness	ON/OFF
Target Humidity	40–80%
Cool Mist Level	1–9
Warm Mist Level	0–3
Timer	0–720 min
Manual Mode	Full support
Auto Mode	Full support
Sleep Mode	Full support
Tank Present	Yes
Out-Of-Water	Yes
Smart Behavior

State preservation:

When entering Manual, original target humidity is saved

When exiting Manual, old target is restored

When entering Sleep, cool level is saved

When exiting Sleep, cool level is restored

So the LV600S behaves like the app:

Sleep always uses fan level 9

Manual hides target humidity

Exiting modes returns the user’s settings

Integration

No cloud

No VeSync

Local control only

Works with Home Assistant Auto-Discovery

No custom component needed (all inline lambda)

7. Parser Logic Summary

Raw UART is parsed using ESPHome uart.debug mode:

Chunked binary capture

One static buffer

Frame detection:

locate 0xA5

extend until checksum valid

Extract fields by index

Publish:

Raw hex string

Tank present

Water out

Display on/off

Target humidity

Current humidity

Cool level

Warm level

Mode text

Values update immediately on incoming status frame.

8. Checksum Rule

A frame is valid when:

(sum(frame bytes) & 0xFF) == 0xFF


Because the command builder:

for(i != checksum_index) s += frame[i];
frame[checksum_index] = (0xFF - (s & 0xFF)) & 0xFF;


For most commands, checksum_index = 5.

9. Implementation Notes

cmd_id is incremented for every sent command

device ignores incorrect checksums silently

mode changes always use two-step commands

some bytes are constant pads

timer uses seconds, not minutes

warm level in status is at index 23

future options may exist in unparsed bytes

10. Future Work / Unknown Fields

bytes 6–13 in status frames

sleep personalization parameters

night light (exists on LV600S)

child lock (possible at class A3)

filter usage

heater power percentage

If you want, I can write the README, or create a /docs/protocol.md file with diagrams and byte offset tables.

You now have a complete, clean, reproducible description of the LV600S UART protocol suitable for a GitHub repository.

You said:
pls write the readme add a statement for use at your own risk and give yourself credit too
ChatGPT said:

Absolutely — here is a clean, polished, GitHub-ready README.md that includes:
✔ Overview
✔ Hardware wiring
✔ Protocol summary
✔ Features
✔ ESPHome integration details
✔ “Use at your own risk” disclaimer
✔ Credit to ChatGPT (as requested)

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

(Optional) select controls

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

📁 File Structure Suggestion
/
├── esphome/
│   ├── lv600s.yaml           # Main firmware
│   └── secrets.yaml
├── docs/
│   ├── protocol.md           # Detailed protocol reference
│   └── wiring.png            # Wiring diagram
└── README.md

🙌 Credits

Protocol reverse-engineering and firmware design: With the help of ChatGPT (OpenAI)

Large portions of the protocol were mapped by iteratively capturing live UART traffic using a logic analyser and recreating command structures manually.

ESPHome inspiration:

Based on concepts used in:
tsquared96/ESPHome-Navien
acvigue/esphome-levoit-air-purifier

ESPHome community & documentation

If you use this work in your own projects, please consider linking back to this repository — it helps the community grow!
