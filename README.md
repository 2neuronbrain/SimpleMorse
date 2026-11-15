# SimpleMorse

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Framework: Arduino](https://img.shields.io/badge/Framework-Arduino-blue.svg)](https://www.arduino.cc/)
[![Platforms](https://img.shields.io/badge/Platforms-AVR%20%7C%20ESP32%20%7C%20ESP8266%20%7C%20ARM-brightgreen.svg)]()

A lightweight Arduino library that decodes simplified, push-button Morse code input into alphanumeric text. 

Unlike traditional Morse code implementations that rely on strict duration timing (distinguishing dits and dahs by how long a single key is held down), **SimpleMorse** assigns distinct functions to dedicated physical buttons: **Dot (`.`)**, **Dash (`-`)**, **Space (Submit / Word Space)**, and an optional **Backspace**. This eliminates timing sensitivity, making Morse input intuitive, reliable, and accessible for embedded systems, assistive devices, and educational projects.

---

## Table of Contents

- [Features](#features)
- [How It Works](#how-it-works)
- [Hardware Wiring](#hardware-wiring)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [Special Commands & Control Macros](#special-commands--control-macros)
- [Morse Code Lookup Reference](#morse-code-lookup-reference)
- [Examples](#examples)
- [Hardware Compatibility](#hardware-compatibility)
- [License](#license)

---

## Features

- **Dedicated Multi-Button Input:** Separate buttons for Dot, Dash, Space, and Backspace.
- **Timing-Independent:** No precision holding or speed synchronization required; users can enter symbols at any pace.
- **Internal Pull-Up Support:** Configures pins as `INPUT_PULLUP` by default—connect buttons directly between GPIO pins and Ground with zero external resistors.
- **Built-In Debounce & Edge Detection:** Transitions are captured on button press (falling edge) with integral debounce handling.
- **State Change Detection:** `stateChange()` method returns `true` only when new input arrives, allowing flicker-free display updates.
- **Control Macros:** Supports sequence commands (e.g., delete and clear) via Morse sequences directly from 3-button setups.
- **Zero External Dependencies:** Built strictly on the core Arduino framework (`Arduino.h`).

---

## How It Works

SimpleMorse maintains two internal buffers:
1. **`symbolBuffer`** (Max 6 symbols): Stores the incoming sequence of dots (`.`) and dashes (`-`) for the current character being composed.
2. **`textBuffer`** (Max 16 characters): Stores the committed decoded alphanumeric text string.

### Input Workflow

```
[Dot Pin Press]  ──> Appends '.' to symbolBuffer
[Dash Pin Press] ──> Appends '-' to symbolBuffer
[Space Pin Press]──> Matches symbolBuffer against internal lookup table:
                       ├─ Match found: Appends character to textBuffer
                       ├─ Macro matched: Executes control action (delete / clear)
                       └─ symbolBuffer empty: Appends ' ' (word space) to textBuffer
                     Clears symbolBuffer for the next character
[Back Pin Press] ──> Removes the last character from textBuffer
```

- **Composing a Character:** Press the Dot and Dash buttons in sequence. The library builds the `symbolBuffer`.
- **Submitting a Character:** Press the Space button. The library looks up the sequence, appends the decoded character to `textBuffer`, and resets `symbolBuffer`.
- **Inserting a Word Space:** Press the Space button when `symbolBuffer` is empty. A space (`' '`) is appended to `textBuffer`.
- **Backspace / Correction:** Press the hardware Back button, or enter the Morse backspace macro (`......` + Space) to delete the trailing character from `textBuffer`.
- **Buffer Rollover:** `symbolBuffer` holds up to 6 symbols, and `textBuffer` holds up to 16 characters (tailored for standard 16×2 character displays). Exceeding these limits rolls over cleanly to prevent overflow.

---

## Hardware Wiring

Buttons operate in **active-LOW** mode. Each push button connects directly between the assigned Arduino digital pin and Ground (`GND`). The library's `begin()` method initializes the microcontroller's internal pull-up resistors (`INPUT_PULLUP`).

### Pinout Table (Standard Setup)

| Button | Arduino Pin (Default Example) | Other Pin Terminal | Logic Level When Pressed |
| :--- | :--- | :--- | :--- |
| **Dash (`-`)** | Digital Pin 11 | `GND` | `LOW` |
| **Dot (`.`)** | Digital Pin 10 | `GND` | `LOW` |
| **Space** | Digital Pin 9 | `GND` | `LOW` |
| **Backspace** *(Optional)* | Digital Pin 8 | `GND` | `LOW` |

### Wiring Diagram

```
                 Arduino / Microcontroller
               ┌───────────────────────────┐
               │                           │
               │  Pin 11 (Dash)  ──────┐   │
               │  Pin 10 (Dot)   ───┐  │   │
               │  Pin 9  (Space) ─┐ │  │   │
               │  Pin 8  (Back)  ││ │  │   │
               │                 ││ │  │   │
               │  GND ───────────┼┼─┼──┼───┘
               └─────────────────┼┼─┼──┼───┐
                                 ││ │  │   │
                     [Back] ─────┘│ │  │   │
                     [Space] ─────┘ │  │   │
                     [Dot] ─────────┘  │   │
                     [Dash] ───────────┘   │
                       │                   │
                       └───────────────────┴──> Common GND
```

---

## Installation

### Method 1: Arduino Library Manager (ZIP)
1. Download the latest release `.zip` from the [repository releases](https://github.com/2neuronbrain/SimpleMorse).
2. Open the Arduino IDE.
3. Navigate to **Sketch** → **Include Library** → **Add .ZIP Library...**.
4. Select the downloaded `.zip` file.

### Method 2: Manual Installation via Git
Clone this repository directly into your local Arduino libraries directory:

```bash
# On Linux / macOS
cd ~/Arduino/libraries/
git clone https://github.com/2neuronbrain/SimpleMorse.git

# On Windows (PowerShell)
cd Documents/Arduino/libraries/
git clone https://github.com/2neuronbrain/SimpleMorse.git
```

### Method 3: PlatformIO
Add the repository directly to your `platformio.ini` dependencies:

```ini
lib_deps =
    https://github.com/2neuronbrain/SimpleMorse.git
```

---

## Quick Start

The following minimal sketch demonstrates initializing the library, reading button presses, and printing both the in-progress Morse symbol sequence and the completed text to the Serial Monitor:

```cpp
#include <SimpleMorse.h>

// Define button pin connections
const int DASH_PIN  = 11;
const int DOT_PIN   = 10;
const int SPACE_PIN = 9;
const int BACK_PIN  = 8; // Optional hardware backspace pin

// Instantiate the SimpleMorse decoder
SimpleMorse morse(DASH_PIN, DOT_PIN, SPACE_PIN, BACK_PIN);

void setup() {
  Serial.begin(9600);
  morse.begin(); // Configures pins as INPUT_PULLUP
  Serial.println("SimpleMorse initialized. Ready for input...");
}

void loop() {
  // Poll inputs and process debounce logic
  morse.update();

  // Print output only when a state change occurs
  if (morse.stateChange()) {
    Serial.print("Current Symbol : ");
    Serial.println(morse.getSymbol());

    Serial.print("Decoded Text   : ");
    Serial.println(morse.getText());
    Serial.println("-------------------------");
  }
}
```

---

## API Reference

### Constructors

```cpp
SimpleMorse(int dashPin, int dotPin, int spacePin, int backPin);
```
Initializes a 4-button configuration including a dedicated hardware backspace pin.
- `dashPin`: Digital pin connected to the Dash button.
- `dotPin`: Digital pin connected to the Dot button.
- `spacePin`: Digital pin connected to the Space button.
- `backPin`: Digital pin connected to the Backspace button.

```cpp
SimpleMorse(int dashPin, int dotPin, int spacePin);
```
Initializes a 3-button configuration without a dedicated backspace pin. Deletion and clearing can still be performed using Morse control macros.

---

### Core Methods

#### `void begin()`
Configures all assigned button pins as `INPUT_PULLUP`. Must be called inside the Arduino `setup()` function.

#### `void update()`
Reads pin states, handles 50 ms debounce delays, detects falling edges, updates `symbolBuffer`, and commits decoded characters to `textBuffer`. Must be called continuously in the Arduino `loop()` function.

#### `bool stateChange()`
Returns `true` if the most recent `update()` cycle processed a valid button press, symbol entry, character decode, or backspace. Use this to trigger selective display or serial refreshes without screen flickering.

---

### Buffer Accessors

#### `String getText()`
Returns the accumulated decoded string currently held in `textBuffer` (up to 16 characters).

#### `String getSymbol()`
Returns the current sequence of dots and dashes being formed in `symbolBuffer` before the Space button commits it.

---

### Buffer Editing & Reset

#### `void clear()`
Clears both `textBuffer` and `symbolBuffer`, resetting input back to an empty state.

#### `void backspace()`
Removes the most recent character from `textBuffer` and prints the updated buffer status to Serial.

---

### Diagnostics & Debugging

#### `void print_details()`
Prints the current contents of `textBuffer` and `symbolBuffer` directly to `Serial`.

#### `void spit_info()`
Prints the configured pin assignments (`dashPin`, `dotPin`, `spacePin`, `backPin`) to `Serial`.

---

## Special Commands & Control Macros

For hardware setups without a physical backspace button (or for users who prefer keeping their hands on the Morse keys), SimpleMorse provides built-in macro sequences:

| Command Sequence | Action | Description |
| :--- | :--- | :--- |
| `......` + **[Space]** | **Backspace** | 6 consecutive dots followed by Space deletes the last character from `textBuffer`. |
| `------` + **[Space]** | **Clear All** | 6 consecutive dashes followed by Space wipes both `textBuffer` and `symbolBuffer`. |
| *[Empty]* + **[Space]** | **Word Space** | Pressing Space when no dots or dashes are buffered appends an ASCII space (`' '`) to `textBuffer`. |

---

## Morse Code Lookup Reference

SimpleMorse supports standard International Morse representations for English letters `A–Z` and numbers `0–9`:

| Character | Morse | Character | Morse | Character | Morse | Character | Morse |
| :---: | :--- | :---: | :--- | :---: | :--- | :---: | :--- |
| **A** | `.-` | **K** | `-.-` | **U** | `..-` | **1** | `.----` |
| **B** | `-...` | **L** | `.-..` | **V** | `...-` | **2** | `..---` |
| **C** | `-.-.` | **M** | `--` | **W** | `.--` | **3** | `...--` |
| **D** | `-..` | **N** | `-.` | **X** | `-..-` | **4** | `....-` |
| **E** | `.` | **O** | `---` | **Y** | `-.--` | **5** | `.....` |
| **F** | `..-.` | **P** | `.--.` | **Z** | `--..` | **6** | `-....` |
| **G** | `--.` | **Q** | `--.-` | | | **7** | `--...` |
| **H** | `....` | **R** | `.-.` | **0** | `0`* | **8** | `---..` |
| **I** | `..` | **S** | `...` | | | **9** | `----.` |
| **J** | `.---` | **T** | `-` | | | | |

*\*Note: In the current lookup table definition, digit `0` is mapped to the single character `"0"`. Standard International Morse digit `0` (`-----`) can be adapted if required by editing `symbolsAlphabet` in `SimpleMorse.h`.*

---

## Examples

The library comes with pre-configured examples under the [`examples/`](examples/) folder:

1. **[`bare.ino`](examples/bare/bare.ino)**
   - Demonstrates core 4-button Morse decoding output to the Serial Monitor at 9600 baud.
   - Ideal for initial hardware verification and testing.

2. **[`bare_with_lcd.ino`](examples/bare_with_lcd/bare_with_lcd.ino)**
   - Demonstrates a standalone decoder displaying real-time text and symbol streams on a 16×2 I2C LiquidCrystal display (`LiquidCrystal_I2C` library required).
   - Shows row 0 as `Txt:` (accumulated message) and row 1 as `In :` (active symbol input).

---

## Hardware Compatibility

SimpleMorse is architecture-independent (`platforms=*`) and compatible with any microcontroller supported by the Arduino core:

- **AVR Boards:** Arduino Uno, Nano, Mega 2560, Pro Mini
- **ATtiny:** ATtiny85, ATtiny84
- **Espressif:** ESP32, ESP32-S2/S3/C3, ESP8266 (NodeMCU, Wemos D1 Mini)
- **ARM Boards:** Raspberry Pi Pico (RP2040), STM32 (BluePill)

---

## Contributing

Contributions, bug reports, and pull requests are welcome:

1. Fork the repository.
2. Create a dedicated feature branch (`git checkout -b feature/improvement`).
3. Commit your changes with clear, descriptive commit messages (`git commit -m 'Add support for punctuation lookup'`).
4. Push to your branch (`git push origin feature/improvement`).
5. Open a Pull Request.

---

## License & Authors

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for complete details.

**Author:** Himanshu Chauragade ([@2neuronbrain](https://github.com/2neuronbrain))  
**Email:** [himanshu.chauragade@gmail.com](mailto:himanshu.chauragade@gmail.com)
