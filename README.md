# esphome_Levoit_humidifier
ESPHOME Yaml for Levoit Humidifiers

To use you either need a new esp32 like a D1 mini esp32 or to reflash the onboard esp32. 

I chose to remove the esp32 from the device and replace it with my own esp32 so i can always go back if i want to.

wiring is simple, we just need to connect our esp32 to the GND VCC and RX/TX lines on the humidifier board.

it is possible to just add this on top and use it with the VeSync app at the same time but i chose not to.



Support for the 600s and the 450s models via differnt yaml (they were similar but the 450s series has a differnt set of frames for control) pls use the correct YAML for your device.

📘 Levoit LV600S Local UART Control (ESPHome Bridge)

Local, cloud-free control of the Levoit LV600S humidifier using an ESP32 over UART.
This project reverse-engineers the LV600S serial protocol and exposes full control through ESPHome + Home Assistant.

No VeSync cloud.
No app dependency.
Fast, reliable, local control.

⚠️ Disclaimer (Use at Your Own Risk)

This project is not affiliated with Levoit or VeSync.
It is based on reverse-engineered UART communication and is provided “AS IS”, with NO WARRANTY of any kind.

You are solely responsible for any damage to your humidifier, ESP32, or other equipment.

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


High-level protocol differences (README text)

451S has more frame families on the wire. In addition to the status frames, you commonly see short control/ack style frames like A5 22 … and A5 12 …. Importantly, the 451S “status layout” can arrive as both A5 02 … and A5 12 …, so the parser must accept both.

Field offsets differ significantly between models. For example, byte 14 is a tank flag on 600S, but power on/off on 451S. The target/current humidity shift by +1 byte on 451S compared to 600S (600S: target@18/current@19; 451S: target@19/current@20).

451S “problem/status” is ambiguous by itself. The 451S status_raw bit indicates a generic fault condition that can be triggered by tank removed OR true out-of-water. To produce a meaningful “Water Out” signal, we derive a logical Water Status: water_problem = status_problem && tank_present.

Frame length varies more on 451S. You’ll see frames that are shorter (≈31 bytes) and “full” frames that include an extra trailer (often 00 00 09, bringing it to ≈34 bytes). The parser should not rely on a single fixed length.


Protocol map for both:


| Byte idx | Field                                   | 600S (A5 02 …)                                                                                                     | 451S (A5 02 … and also A5 12 …)                                                               | Notes                                                            |
| -------: | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
|        0 | SOF                                     | `0xA5`                                                                                                             | `0xA5`                                                                                        | Start-of-frame                                                   |
|        1 | Frame type                              | `0x02`                                                                                                             | `0x02` **or** `0x12`                                                                          | **451S** emits “status layout” as both `0x02` and `0x12`         |
|        2 | Sequence / counter                      | Varies (ex: `0x61`, `0xA5`, `0xB3`…)                                                                               | Varies (ex: `0x03`, `0x14`, `0x5A`…)                                                          | Looks like a rolling counter                                     |
|        3 | Length-ish / constant                   | Often `0x18` in your 600S samples                                                                                  | Often `0x20` in your 451S samples                                                             | Not treated as true length in our parser                         |
|        4 | Unknown                                 | `0x00` (seen)                                                                                                      | `0x00` (seen)                                                                                 | Reserved/unknown                                                 |
|        5 | Checksum-ish / part of sum              | Varies                                                                                                             | Varies                                                                                        | Not “checksum byte” alone; checksum is frame-sum rule            |
|        6 | Unknown constant                        | Often `0x01`                                                                                                       | Often `0x01`                                                                                  | Likely fixed                                                     |
|        7 | Magic                                   | `0x11`                                                                                                             | `0x11`                                                                                        | Fixed in your captures                                           |
|        8 | Magic                                   | `0x41`                                                                                                             | `0x41`                                                                                        | Fixed in your captures                                           |
|        9 | Unknown                                 | `0x00`                                                                                                             | `0x00`                                                                                        | Fixed in your captures                                           |
|       10 | Device/model-ish                        | `0x32` (seen in all 600S rows you pasted)                                                                          | Varies (seen as `0x01` / etc depending on frame)                                              | **600S** very consistent here in your sample rows                |
|       11 | Unknown                                 | `0x00`                                                                                                             | Varies                                                                                        | Unknown                                                          |
|       12 | Unknown                                 | `0x01`                                                                                                             | `0x2D` / etc (varies by 451S frame family)                                                    | Diverges between models                                          |
|       13 | Unknown                                 | `0x01`                                                                                                             | Often `0x00`/`0x01`                                                                           | Diverges                                                         |
|       14 | Power (451S) / Tank (600S)              | **Tank flag** (from your sheet): `1 = tank removed`, `0 = tank present`                                            | **Power flag:** `0 = off`, `1 = on/running`                                                   | This is a *big* offset difference                                |
|       15 | Water (600S) / Tank (451S)              | **Water flag:** `1 = water out`, `0 = OK`                                                                          | **Tank flag:** `0 = tank present`, `1 = removed`                                              | Another key difference                                           |
|       16 | Display / brightness (600S)             | `0x64` in your sample rows (looks like brightness / always-on in those rows)                                       | Often `0x00` (varies)                                                                         | **451S uses byte 17 for display**, not 16                        |
|       17 | Display (451S) / (600S misc)            | `0x00/0x01` (varies in 600S rows)                                                                                  | **Display brightness:** `0x00 = off`, `0x64 = on`                                             | Proven by your app toggle trace                                  |
|       18 | Target humidity (600S) / Status (451S)  | **Target RH** (ex: `0x41` = 65)                                                                                    | **Status/problem flag:** `0 = OK`, `1 = problem` (tank removed OR water-out)                  | In 451S we derived **Water Status** logically using tank_present |
|       19 | Current humidity (600S) / Target (451S) | **Current RH** (ex: `0x2A` = 42)                                                                                   | **Target RH**                                                                                 | Off-by-one shift vs 600S                                         |
|       20 | Unknown (600S) / Current (451S)         | Often `0x16` (22) in your 600S rows                                                                                | **Current RH**                                                                                | 451S proven from many logs                                       |
|       21 | raw_flags                               | `0x03` (constant in your pasted 600S sample rows)                                                                  | `raw_flags` byte (we have it captured but not fully decoded)                                  | We should document bit meanings once proven                      |
|       22 | Cool level / Mode                       | Appears to encode **cool mist level** (but mapping not fully proven; values looked non-linear in your pasted rows) | **Mode family:** `0=auto`, `1=manual`, `2=sleep` (mostly works; manual reporting still flaky) | 451S mode is here; 600S likely uses this for fan/cool            |
|       23 | Unknown / Cool (451S)                   | Unknown                                                                                                            | **Cool mist level raw** (1–8 manual; `0x09` auto; `0x01` sleep observed)                      | 451S cool level is here                                          |
|       24 | Unknown / Warm (451S)                   | Unknown                                                                                                            | **Warm level** (0–3)                                                                          | 451S warm level proven                                           |
|   25..30 | Padding / reserved                      | Mostly zeros in your 600S rows                                                                                     | Varies; can be shorter/partial in logs                                                        | Both have trailing padding-like bytes sometimes                  |
|   31..33 | Trailer / mirror bytes                  | (not seen in your 600S snippet)                                                                                    | Sometimes ends with `00 00 09` and byte **33** mirrors cool (`fan_mirr`)                      | 451S frames can be **31 bytes or 34 bytes**                      |


🙌 Credits

Protocol reverse-engineering and firmware design: With the help of ChatGPT (OpenAI)

Large portions of the protocol were mapped by iteratively capturing live UART traffic using a logic analyser and recreating command structures manually.

ESPHome inspiration:

Based on concepts used in:
tsquared96/ESPHome-Navien
acvigue/esphome-levoit-air-purifier

ESPHome community & documentation

If you use this work in your own projects, please consider linking back to this repository — it helps the community grow!
