# Arduino DIY Mini Oscilloscope

A low-cost digital oscilloscope built from an Arduino-compatible board, a 128x64 SSD1306 I2C OLED display and a rotary encoder. It is meant as a simple, educational tool for looking at slow-to-moderate analog signals.

The sketch uses the memory-friendly `U8g2` display library, a hardware interrupt for the encoder, and a faster ADC clock for quicker sampling.

---

## Features

- **Live waveform display** of an analog signal on a 128x64 OLED
- **Adjustable timebase** with nine settings, controlled by turning the encoder
- **Freeze mode:** press the encoder button to hold the current capture, press again to resume
- **Auto vertical scaling:** the waveform is stretched to fit the screen using the min/max of each capture
- **Faster ADC:** prescaler changed from 128 to 16 for roughly 8x faster conversions
- **On-screen info:** current time-per-division and a `FROZEN` indicator
- **Low RAM use:** uses the paged `U8G2_SSD1306_128X64_NONAME_1` buffer mode

---

## Hardware

| Part | Notes |
| --- | --- |
| Arduino-compatible board | Uno, Nano or Pro Mini (ATmega328P). Needs hardware interrupt on D2 |
| 0.96" SSD1306 OLED, 128x64 | I2C version |
| Rotary encoder | KY-040 module, or a bare encoder with pull-ups |
| Breadboard and jumper wires | |

### Wiring

The encoder's **CLK** pin must go to an interrupt-capable pin (D2 in this sketch).

| Component | Pin | Arduino |
| --- | --- | --- |
| OLED | VCC | 5V |
| OLED | GND | GND |
| OLED | SCL | A5 |
| OLED | SDA | A4 |
| Encoder | + | 5V |
| Encoder | GND | GND |
| Encoder | CLK | D2 |
| Encoder | DT | D3 |
| Encoder | SW | D4 |
| Signal input | Probe tip | A0 |
| Signal input | Probe ground | GND |

> **Warning:** The analog input accepts **0 to 5 V only**. Anything outside that range can damage the board. Do not connect mains or other high-voltage signals directly.

---

## Software Setup

1. Install the Arduino IDE.
2. Open **Tools > Manage Libraries...**, search for **U8g2** (by olikraus) and install it.
3. `Wire.h` ships with the IDE, so nothing else is needed.
4. Open `arduino-oscilloscope.ino`, select your board and port, and upload.

---

## Usage

1. Wire everything as shown above and upload the sketch.
2. On power-up a splash screen appears for two seconds.
3. Connect your signal to **A0** and its ground to **GND**.
4. Turn the encoder to change the timebase:
   - **Clockwise:** zoom in (less time per division)
   - **Counter-clockwise:** zoom out (more time per division)
5. Press the encoder button to freeze the display. Press again to resume.

### Timebase settings

The sketch inserts a delay (in microseconds) between samples. The on-screen label is calculated as `32 x delay`, because each of the four horizontal divisions spans 32 samples.

| Delay between samples | Label shown |
| --- | --- |
| 0 us | 0 us/div |
| 10 us | 320 us/div |
| 25 us | 800 us/div |
| 50 us | 1600 us/div |
| 100 us | 3200 us/div |
| 250 us | 8000 us/div |
| 500 us | 16000 us/div |
| 1000 us | 32000 us/div |
| 2500 us | 80000 us/div |

The default on boot is the 50 us delay setting.

**Accuracy note:** each ADC conversion itself takes about 13 to 14 us at the 16 prescaler, and that time is not included in the label. At small delay values the real time per division is therefore noticeably longer than shown. For accurate frequency measurements, calibrate against a known signal.

---

## How the Code Works

### `setup()`
- Starts I2C at 400 kHz and initializes the display.
- Configures the encoder pins as inputs with internal pull-ups.
- Attaches an interrupt on the CLK pin (falling edge) that calls `updateEncoder()`.
- Sets the ADC prescaler to 16 (`ADCSRA |= (1 << ADPS2)` after clearing the prescaler bits), which cuts conversion time from about 112 us to about 13 us.
- Shows the boot splash for two seconds.

### `loop()`
1. **Button handling:** polls the encoder switch with a 50 ms debounce and toggles `isFrozen`.
2. **Capture:** when not frozen, reads 128 samples from A0, waiting the selected delay between each.
3. **Auto-scaling:** finds the minimum and maximum readings, adds a small margin, and avoids a zero range.
4. **Drawing:** renders a 4x4 grid, connects the samples with lines mapped to screen height, and draws the timebase label (with a black background box for readability) and the `FROZEN` tag.

### `updateEncoder()`
An interrupt service routine that runs on each falling edge of CLK. It compares DT with CLK to determine the rotation direction and moves `timebaseIndex` up or down, clamped to the valid range.

---

## Limitations

- **No trigger:** capture starts at an arbitrary point, so repeating waveforms will drift or jitter on screen.
- **Limited bandwidth:** the Arduino's ADC is limited to roughly 70 kS/s at best here, so only low-frequency signals (up to a few kHz) display cleanly.
- **Positive voltages only:** the input cannot measure negative voltages.
- **Relative scale:** auto-scaling means the vertical axis has no fixed volts-per-division, so absolute voltage cannot be read from the screen.
- **Blocking button check:** the button wait loop pauses sampling while the button is held down.

---

## Possible Improvements

- Add rising or falling edge triggering to stabilize repeating signals
- Calculate and display peak-to-peak voltage and frequency
- Add input protection, for example a series resistor with clamp diodes or a Zener
- Add an AC coupling option with a switch and capacitor
- Include ADC conversion time in the timebase label for accurate readings

---

## Troubleshooting: Upload Fails on a Pro Mini

Some cheap CH340G USB-to-serial adapters have no DTR pin broken out, so the board never auto-resets for uploading. A workaround:

1. Find the DTR pin on the CH340G chip using its datasheet pinout.
2. Connect a 0.1 uF ceramic capacitor (marked `104`) between the chip's DTR pin and the Arduino's RST pin.
3. Upload again.

---

## License

This project is open source. You are free to use, modify and distribute the code. A credit or link back to the repository is appreciated but not required.

*Enjoy the process, not just the results.*
