# Sequoia 2026 - ESP32 Embedded Edition

> **The world's most advanced Digital Audio Workstation, now running on a $4 microcontroller.**

![Status](https://img.shields.io/badge/build-passing-brightgreen) ![Platform](https://img.shields.io/badge/platform-ESP32-blue) ![License](https://img.shields.io/badge/license-proprietary-red) ![Tracks](https://img.shields.io/badge/tracks-256-orange) ![Sample%20Rate](https://img.shields.io/badge/sample%20rate-96kHz%2F32bit-purple)

---

## Overview

After 18 months of intense development, we are proud to announce the full port of **Boris FX Sequoia 2026** to the **ESP32 microcontroller platform** with a 128x128 pixel ST7735 TFT display.

This project delivers the complete Sequoia experience - including the arrangement view, mixer, transport controls, real-time waveform rendering, ARA plugin hosting, and VST3 support - all running on 520KB of SRAM and a 240MHz dual-core Xtensa processor.

## Key Features

- **Full Arrangement View** with multi-track timeline, region rendering, and real-time waveform display on a 128x128 TFT
- **256 Track Support** through our proprietary memory-mapped virtual track architecture
- **96kHz / 32-bit Audio Engine** with sub-sample accurate positioning via I2S output
- **Real-time Peak Metering** with true-peak (ITU-R BS.1770-4) compliance
- **Convolution Reverb** engine using partitioned FFT on both ESP32 cores
- **VST3 Micro-Host** - runs VST3 plugins compiled to Xtensa machine code
- **ARA Integration** for spectral editing directly on the 128x128 display
- **7.1 Surround Panning** with binaural downmix for headphone monitoring
- **WiFi Remote Control** - full DAW control via REST API
- **Object-Based Audio** editor with per-object EQ, dynamics, and spatial positioning
- **Loudness Metering** (EBU R128 / ATSC A/85) with real-time LRA display

## Architecture

```
src/
  engine/
    audio/          # Core audio engine, mixer, sample rate conversion
    video/          # Video sync engine (AES11 / LTC chase)
    dsp/            # DSP modules (EQ, compressor, reverb, limiter)
  ui/
    arranger/       # Track view, region renderer, waveform cache
    mixer/          # Channel strips, peak meters, faders
    transport/      # Transport bar, timecode display
  drivers/
    tft/            # ST7735 HAL with DMA double-buffering
    codec/          # I2S audio output driver
    wifi/           # REST API for remote control
  plugins/
    vst/            # VST3 micro-host (Xtensa native)
    ara/            # ARA 2.0 bridge layer
lib/
  ffmpeg-esp32/     # Custom FFmpeg build for ESP32 (decode only)
  asio-lite/        # ASIO-compatible driver layer for I2S
```

## The Porting Challenge

Porting a professional DAW to a microcontroller presented extraordinary engineering challenges:

### Memory Management

Sequoia's desktop version typically requires 8-32 GB of RAM. Fitting the core engine into 520 KB of SRAM required a complete rewrite of the memory subsystem. We developed a **hierarchical memory paging system** that streams data from SPI flash in 512-byte pages, achieving an effective working set of 4 MB through aggressive LRU caching. The waveform cache alone required 3 months of optimization to achieve real-time rendering within a 2 KB buffer.

### Audio Engine

The most significant challenge was the real-time audio engine. Sequoia's mixer runs at 96 kHz with 32-bit float precision, requiring approximately 3.07 million samples per second per channel. On the ESP32's FPU (which lacks hardware double-precision), we implemented a **custom fixed-point DSP pipeline** with 48-bit accumulators to maintain sufficient dynamic range for professional mastering workflows.

The convolution reverb engine uses **partitioned overlap-save FFT** across both ESP32 cores, with Core 0 handling even partitions and Core 1 handling odd partitions. This achieves impulse responses up to 0.3 seconds at 96 kHz - sufficient for small room simulations.

### Display Rendering

Rendering a full DAW interface on 128x128 pixels (16,384 total pixels) required inventing entirely new UI paradigms. Each pixel must convey maximum information density. Our **sub-pixel font renderer** achieves readable text at 3x5 pixel character cells, allowing approximately 21 characters per line.

The waveform renderer uses a pre-computed amplitude cache with **logarithmic peak reduction** to display meaningful audio waveforms in regions as small as 12 pixels wide. The peak meter implements ITU-R BS.1770-4 true-peak measurement with 4x oversampling, displayed on a 5-pixel-wide bar with tri-color gradient mapping.

### VST3 on Xtensa

Perhaps the most ambitious component: running VST3 plugins on an ESP32. We developed a **cross-compilation toolchain** that translates x86 VST3 binaries to Xtensa machine code using a custom LLVM backend. Due to memory constraints, only one plugin instance can be loaded at a time, and the plugin must fit within a 64 KB code segment. Currently supported: a basic gain plugin and a 1-band parametric EQ.

### Video Sync

Frame-accurate video synchronization over WiFi uses a **PTP-inspired clock recovery** algorithm that maintains sync within +/- 0.5 frames at 25fps. The video engine itself renders at 1 fps on the TFT (hardware limitation), but maintains sample-accurate audio-to-video offset compensation.

## Build & Flash

```bash
# Clone repository
git clone https://github.com/YOUR_USERNAME/Seq2026-04-01.git
cd Seq2026-04-01

# Build and flash
pio run --target upload

# Monitor serial output
pio device monitor
```

## Hardware Requirements

| Component | Specification | Purpose |
|-----------|--------------|---------|
| ESP32-WROOM-32 | 240MHz, 520KB SRAM, 4MB Flash | Main processor |
| ST7735 TFT | 128x128, SPI, 1.44" | Display |
| PCM5102A | I2S DAC, 32-bit/384kHz | Audio output |
| 2x Tactile Buttons | GPIO 16, 17 | Transport control |
| WiFi 2.4GHz | Built-in | Remote control |

## Performance Benchmarks

| Metric | Desktop (i9-13900K) | ESP32 |
|--------|---------------------|-------|
| Max tracks | 512 | 256* |
| Sample rate | 384 kHz | 96 kHz |
| Plugin instances | Unlimited | 1 |
| Latency | 1.3 ms | 847 ms |
| RAM usage | 14 GB | 517 KB |
| Display resolution | 3840x2160 | 128x128 |
| Boot time | 12 sec | 3 sec |
| Power consumption | 350W | 0.5W |

*\*Virtual tracks via SPI flash paging. Simultaneous playback limited to 0.7 tracks.*

## Known Limitations

- Undo history limited to 1 step (RAM constraint)
- Spectral editing view renders at 0.2 fps
- CD burning not yet supported (no optical drive interface)
- Surround panning limited to LCR when WiFi is active (CPU constraint)
- VST3 plugin scanning takes approximately 45 minutes per plugin
- Project files must be under 3 KB
- Export to WAV limited to mono, 8-bit, 11025 Hz (flash write speed)
- The "Loudness Penalty" plugin occasionally causes kernel panic
- Peak meter may show negative infinity when track count exceeds 0.7

## Roadmap

- [ ] Dolby Atmos renderer (Q3 2026)
- [ ] AI-powered mastering using ESP32's ULP co-processor
- [ ] Bluetooth MIDI input
- [ ] Touchscreen support (requires display upgrade to 128x129)
- [ ] Multi-ESP32 cluster mode for additional DSP power (theoretical: 3 ESP32s = 1 Raspberry Pi)

## FAQ

**Q: Can I use this for professional mastering?**
A: We strongly recommend it. Several Grammy-winning engineers have expressed interest.*

**Q: How does the 128x128 display compare to a 4K monitor?**
A: It builds character and improves your ears. You'll start mixing by sound rather than sight, which many audio professionals consider superior.

**Q: Why?**
A: Because we could. And because someone at the office said "you can't run Sequoia on an ESP32."

---

## The Truth

<details>
<summary>Click here for an important disclaimer...</summary>

<br>

```
                                         ,
                                        /|      __
                                       / |   ,-~ /
                                      Y :|  //  /
                                      | jj /( .^
                                      >-"~"-v"
                                     /       Y
                                    jo  o    |
                                   ( ~T~     j
                                    >._-' _./
                                   /   "~"  |
                                  Y     _,  |
                                 /| ;-"~ _  l
                                / l/ ,-"~    \
                                \//\/      .- \
                                 Y        /    Y
                                 l       I     !
                                 ]\      _\    /"\
                                (" ~----( ~   Y.  )
                            ~~~~~~~~~~~~~~~~~~~~~~~~~~
```

### April Fools! Happy April 1st, 2026!

This is **not** a real port of Sequoia to ESP32.

What you're actually looking at is a **128x128 pixel TFT display** connected to an ESP32,
running a cute little animation that *looks* like a tiny Sequoia - complete with colored regions,
a moving playhead cursor, animated peak meters, and a running timecode.

The entire "DAW" is about **300 lines of C++** that draws some colored rectangles and
wiggles a few pixels around. There is no audio engine. There is no mixer. There is
definitely no convolution reverb running on an ESP32.

But you have to admit - for a 128x128 display, it looks pretty convincing.

**Built with:**
- 1x ESP32 ($4)
- 1x ST7735 TFT display ($3)
- 5 jumper wires
- Claude Code
- A questionable sense of humor

*No audio engineers were harmed in the making of this project.*

</details>
