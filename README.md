---
title: "Wildlife Soundscape Logger"
date: "2026-04-05"
team_size: 6
scope: "ESP32-based outdoor audio recording system for behavioral ecology research"
skills: ["I2S Protocol Implementation", "SPI File System Management", "Embedded C++", "Audio Signal Analysis (FFT)", "Hardware Integration", "Power Management", "Serial Communication & Logging"]
---

# Wildlife Soundscape Logger

## Table of Contents

- [Project Overview](#project-overview)
- [Skills Demonstrated](#skills-demonstrated)
- [Technical Summary](#technical-summary)
- [Design Challenges](#design-challenges)
- [Repository & Building](#repository--building)

---

## Project Overview

**Context:** Semester-long embedded systems capstone project for a Western University biology professor studying how urban noise affects wildlife behavior on campus.

**Team:** 6 people (mechanical, electrical, firmware subteams). I served as **firmware lead** and **primary technical advisor**, guiding protocol selection, hardware integration, and system validation while supporting skill development across mechanical and electrical subteams.

**Outcome:** **1st place** in our category (~20 teams). Delivered a **$77.50/unit audio logger** that imitates commercial alternatives costing $125–200 and solves the professor's top constraint: **low maintenance** (single person deployment/retrieval, 4+ week unattended operation).

The system records 5-minute stereo WAV files at 44.1 kHz to microSD card on a 60-minute duty cycle, enabling month-long field deployments without battery swaps or manual data retrieval.

---

## Highlights

| Achievement | Impact |
|---|---|
| **1st place** in design category | Won against ~20 competing teams in ES1050 capstone competition |
| **$77.50/unit cost** | 38% cheaper than AudioMoth ($125); 61% cheaper than Wildlife Acoustics ($200) |
| **4+ week autonomy** | Solves client's #1 priority: low maintenance vs. 2-week manual retrieval on commercial systems |
| **Real-world validation** | Biology professor in-person demo and feedback on our presented proof of concept |
| **Technical leadership** | Firmware lead + technical advisor; enabled mechanical and electrical teams to solve integration challenges independently |
| **Pragmatic engineering** | Delivered working system in 1-week hardware sprint through signal analysis, test signal generation, and iterative subsystem validation |

---

## Skills Demonstrated

### Firmware Leadership & Technical Advisory
- **System architect**: selected I2S + SPI interface strategy, established protocol parameters (44.1 kHz, master RX mode, 1 MHz SD clock), and owned integration validation across three subteams
- **Mentorship & skill development**: guided mechanical team through CAD-to-3D-print workflow and electrical team through schematic review and microcontroller GPIO mapping; enabled independent problem-solving rather than imposing solutions

### Hardware Integration & Protocol Design
- **I2S receiver configuration** in master mode with APLL clock, accounting for 32-bit stereo at 44.1 kHz and DMA buffer contention with SPI operations
- **SPI file system management**: initialized SD card at 1 MHz (not 10+ MHz default) to reduce electromagnetic interference with I2S data lines; validated through spectral analysis
- **Electrical noise diagnosis**: mapped real I2S captures into frequency domain using FFT in Audacity to identify harmonic peaks; validated solution against known-frequency tone recordings

### Embedded Firmware Development
- **Audio stream integration**: wrapped `arduino-audio-tools` WAV encoder and configured it for real-time I2S-to-file encoding with proper header finalization
- **Multi-module architecture**: designed clean separation between I2S driver, WAV writer, SD card init, and session management; enabled rapid isolation and testing of failures
- **Power-aware scheduling**: implemented low-power sleep mode between recording cycles, holding I2S and clock active while reducing system current draw

### Pragmatic Engineering Under Constraint
- **Received hardware 1 week before deadline**: prioritized working baseline over feature completeness; started with minimal reference implementation (`Atomic` repo) instead of feature-rich codebase that failed to compile
- **Signal validation without live microphone**: designed I2S dummy data generator (sine wave test signal) to decouple microphone hardware from SD write pipeline; accelerated subsystem debugging
- **Serial logging & automation**: wrote scripts to stream raw I2S samples over UART (921600 baud) for post-hoc analysis, enabling offline debugging when hardware was unavailable

---

## Technical Summary

The logger boots into a simple state machine: initialize I2S and SD, create a timestamped session folder, then loop on a 60-minute cycle (5 min record → 55 min sleep). All audio is encoded as PCM WAV with standard headers, compatible with ecology analysis tools like Cornell's Raven Pro and MATLAB.

### I2S Audio Capture
- **Configuration**: 44.1 kHz stereo, 32-bit samples, standard I2S format, master receiver mode
- **Clock**: APLL enabled for precise sample rate generation
- **Buffering**: 4 KB × 4 buffers to handle DMA bursts without sample loss during concurrent SPI writes

### SD Card & File System
- **Interface**: VSPI (GPIO 18/19/23 clock/MISO/MOSI, GPIO 5 chip select)
- **Clock speed**: 1 MHz (conservative to minimize crosstalk)
- **File structure**: `AUDIO_LOGGER_SESSION_N/recording_M_HH-MM-DD.wav` for organized multi-deployment retrieval
- **Encoding**: WAV header written once per file, PCM data streamed, header finalized on close

### Power & Duty Cycle
- **Target**: 4–8 week runtime on AA batteries
- **Strategy**: 5-minute active recording + 55-minute deep sleep, clock/I2S held in sleep mode
- **Current draw during sleep**: <100 µA (ESP32 spec); dominant load is microphone quiescent current (~4 mA)

---

## Design Challenges

### Challenge 1: Electrical Noise in Parallel I2S & SPI

**Problem:** I2S data lines carry MHz-rate clock and serial data; simultaneously toggling SPI clock (SD write) at the default 10+ MHz caused bit errors in WAV files and clicking artifacts in audio.

**Tradeoffs Considered:**
- Increase I2S buffer sizes (reduces timing sensitivity, increases RAM and latency)
- Reduce SPI clock speed (trades throughput for noise immunity)
- Shield signal lines (adds cost and complexity; unavailable mid-project)

**Rationale & Solution:** Reduced SPI clock to **1 MHz**, well below I2S data rates. Validated by capturing real I2S waveforms and analyzing the frequency spectrum in Audacity: confirmed peak spectral energy shifted away from I2S clock harmonics. Further validated by recording a **1 kHz reference tone** through the system and confirming accurate frequency content in FFT output. This solution cost nothing and required no hardware changes.

### Challenge 2: Firmware Base Selection Under Time Pressure

**Problem:** Searched GitHub for ESP32 I2S-to-SD examples. Found a feature-rich repo (`Beastbroak`) with many capabilities, but it failed to compile, had unclear architecture, and took hours to debug dependency issues.

**Tradeoffs Considered:**
- Invest time in fixing complex repo (risky; unfamiliar codebase)
- Start from scratch using only Arduino I2S/SD APIs (slow; more boilerplate)
- Find a minimal working example and extend it (unknown if available)

**Rationale & Solution:** Pivoted to a simpler reference (`Atomic`), which compiled immediately and achieved serial I2S communication in under an hour. Built up incrementally: serial output → WAV file generation → SD writes → session management. This reduced velocity impact from hardware delays and left time for audio validation and integration testing.

### Challenge 3: Signal Validation Without Hardware

**Problem:** Firmware arrived late; microphone hardware not available for testing until 3 days before submission.

**Tradeoffs Considered:**
- Write firmware blind and hope it works (high risk of failure)
- Use Wokwi online simulator (limited I2S support)
- Generate test signal locally and validate the full pipeline

**Rationale & Solution:** Implemented **I2S dummy data generator** (sine wave at 44.1 kHz) in a separate ESP32 node to simulate a real microphone. Routed this into the logger's RX pin and validated that WAV files were generated correctly before the real microphone arrived. This decoupling caught bugs in the WAV encoder and SD mount logic weeks before final integration, reducing last-minute surprises.

### Challenge 4: Duty Cycle Firmware Complexity

**Problem:** Final spec requires 5-minute record + 55-minute sleep; prototype was built on a 60-second cycle for rapid iteration. Transitioning to real timing risked destabilizing tested code.

**Tradeoffs Considered:**
- Rebuild timing logic from scratch (risky)
- Keep prototype timing and manually adjust recordings (defeats power savings goal)
- Parameterize sleep duration and test both modes

**Rationale & Solution:** Made sleep duration and record length **compile-time constants**, allowing the same codebase to run both 60-second cycles (for testing) and production 60-minute cycles. Validated 50+ consecutive recordings in prototype mode before setting production parameters. This approach kept the critical state machine logic unchanged and reduced regression risk.

---

## Repository & Building

**GitHub:** [github.com/NEhnes/mic_sd_logger](https://github.com/NEhnes/mic_sd_logger) (MIT license)

**Setup (Linux):**

```bash
sudo apt install platformio git
git clone https://github.com/NEhnes/mic_sd_logger
cd mic_sd_logger/src/final_submission
pio run -t upload
pio device monitor -b 115200
```

**Dependencies:**
- PlatformIO + ESP32 board support
- Arduino framework (built-in)
- `arduino-audio-tools` v1.2.2 (Phil Schatzmann)
- `SD` and `SPI` libraries (Arduino built-in)

**Hardware (Bill of Materials):**

| Component | Unit Cost | Notes |
|-----------|-----------|-------|
| ICS-43434 MEMS Microphone | $11.63 | I2S output, <100 µA quiescent |
| ESP32 WROOM MCU | $13.99 | 240 MHz dual-core, integrated WiFi |
| MicroSD Card Adapter | $2.00 | SPI interface |
| LM2596 Buck Converter | $6.00 | Battery to 3.3V regulation |
| AA Battery Holder (×2) | $9.00 | 3V source for buck input |
| **Total** | **$42.62** | (excludes housing, mesh, mounting) |

**Key Configurations:**

I2S: 44.1 kHz, 2 channels (stereo), 32-bit samples, master RX mode, APLL enabled  
SD: 1 MHz SPI clock, GPIO 5 CS, VSPI pins (18/19/23), FAT32 filesystem  
Power: 55-minute sleep with RTC/I2S active, <100 µA quiescent

**Testing & Validation:**

- Electrical noise analysis: FFT in Audacity of real I2S captures
- Audio accuracy: 1 kHz reference tone recording vs. spectral analysis
- Extended runtime: 600+ second operation validated (600+ recordings at 1-second cycle)
- Portability: hand-carry deployment time <30 seconds, <5 lbs
- Durability: drop tested (clips are weak point; production redesign recommended)

---

## Project Structure

```
src/final_submission/          # Production logger
├── platformio.ini
├── include/
│   ├── sd_card.h              # SPI mount, 1 MHz clock init
│   └── wav_writer.h           # WAV encoder wrapper
└── src/
    ├── main.cpp               # Boot, I2S config, session loop
    ├── sd_card.cpp
    └── wav_writer.cpp

src/prototype_logger/          # Multi-node exploration
├── sender_node/               # I2S sine wave generator
├── receiver_node/             # I2S→Serial CSV streamer
└── sd_logger_node/

src/test_signal_generation/    # Standalone I2S test signal

docs/
├── documentation-submission.pdf
├── program-flow-diagram-submission.pdf
├── wiring-schematic-submission.png
└── i2s-dummy-test-setup.png
```

---

## Notes for Future Work

**Strengths:** Low cost, real-world client validation, demonstrated 1st-place result, pragmatic audio diagnostics (FFT analysis), clean separation of concerns in firmware.

**Known Limitations:** Prototype under time pressure; clip arm durability insufficient (45″ drop test failure); extreme temperature testing (-20°C) not completed; 40 dB noise gate not implemented; scheduled recording firmware uses 1-minute prototype cycle instead of production 60-minute cycle in demo.

**For Production:** Redesign mechanical clips for robustness, extend environmental testing, implement adaptive noise gating, add telemetry (battery voltage, microSD health checks) over serial.
