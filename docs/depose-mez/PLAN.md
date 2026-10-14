# depose-mez — Plan

Fork of [ruvnet/RuView](https://github.com/ruvnet/RuView). This doc covers the hardware architecture, the RF bands, and a Stage 2 research plan. It's a planning doc only, and no code has changed.

## 1. What upstream already gives us (per the repo)
- ESP32 CSI node firmware in `firmware/esp32-csi-node` (ESP-IDF). It has build targets for ESP32-S3 and ESP32-C6 (`sdkconfig.defaults.esp32c6`), plus an optional mmWave UART on the C6. Its README also lists a `5ghz-channel` config (channel 36).
- A Raspberry Pi 5 path in `docs/tutorials/pi5-cluster-cognitive-rf-observer.md`. That path runs nexmon_csi on the Pi 5's onboard bcm43455c0 radio at 5 GHz (ch.149/80 MHz) and uses Hailo-8 accelerators.
- A Rust v2 workspace (`v2/crates`) with fusion, HAL, and Home Assistant/Matter integration, plus pretrained CSI models on Hugging Face.
- Caveats:
  - The "LoRA" in the README means MicroLoRA ML adapters. It has nothing to do with LoRa radio, and upstream has no LoRa radio code.
  - The README says UWB and radar fusion accuracy is "still synthetic until real-data validation".
  - ADR-073 claims "ESP32-S3 supports 5 GHz CSI". That's wrong, because the S3 is 2.4 GHz only. Treat upstream's 5 GHz ESP config as needing an ESP32-C5.

## 2. Architecture
- **Master: Raspberry Pi 4 or 5.** It runs the RuView server, gathers CSI frames from the nodes over Wi-Fi/UDP, and runs inference and the Home Assistant/MQTT bridge. A Pi 5 is preferred because upstream already documents it.
- **Sensors: ESP32 CSI nodes.** These run the upstream firmware, with two or more nodes per room for multistatic sensing.
- **5 GHz: ESP32-C5 nodes** (see Bands). These need a firmware port and testing, since upstream doesn't target the C5.
- **6 GHz: Intel AX210 node.** PicoScenes is x86-64 only, so this needs a small x86 PC (see Bands). It can't go on the Pi.
- **LoRa: backhaul and telemetry only.** This means things like node health, alerts, and low-rate presence events from far nodes. LoRa is far too slow to carry raw CSI. This is new work, because upstream has no LoRa radio code.

## 3. Bands
| Radio | Band | Notes |
|---|---|---|
| ESP32 classic / S3 / C6 | 2.4 GHz only | C6 is Wi-Fi 6 at 2.4 GHz |
| ESP32-C5 | 2.4 + 5 GHz | CSI is confirmed at 5 GHz by Espressif (esp-csi README; ESP-IDF v6.0 C5 docs; esp-idf #18565). Needs IDF newer than v5.5 (release/6.0 or master). Hardware limits: 40 MHz max (HT40), 20 MHz for HE/VHT, no 80/160 MHz. One RF chain, so no simultaneous dual-antenna CSI (esp-csi #242). |
| Any ESP32 at 6 GHz | none | No Espressif part found with 6 GHz Wi-Fi |
| Intel AX210/AX211 | 2.4/5/6 GHz (5945–7125 MHz), up to 160 MHz, 2x2 | CSI via PicoScenes. PicoScenes "doesn't support ARM CPU" (ps.zpj.io install page), so it can't run on a Pi 5 even though the Pi 5's PCIe slot could physically hold the card. Use an x86 mini-PC. 320 MHz channels are Wi-Fi 7, which the AX210 doesn't support. |
| LoRa | US915: 902–928 MHz. EU868: 863–870 MHz | The "898 MHz" figure isn't a standard EU LoRa band |
| UWB (e.g. Qorvo DW3000) | UWB overall 3.1–10.6 GHz. Ch5 ≈ 6.5 GHz, Ch9 ≈ 7.99 GHz | Ranging to about 10 cm. Good for anchor and tag ground truth. |

## 4. Other transceiver ICs on other frequencies
| IC | Band | Good for |
|---|---|---|
| TI CC1101 | sub-GHz, ~300–348 / 387–464 / 779–928 MHz | Cheap sub-GHz RSSI links, best wall penetration, coarse motion |
| Semtech SX1280 | 2.4 GHz LoRa/FLRC, with time-of-flight ranging | Ranging and long-range 2.4 GHz telemetry |
| Nordic nRF52 / nRF24L01+ | 2.4 GHz | BLE RSSI/AoA (nRF52 direction finding), cheap links |
| Hi-Link HLK-LD2410 | 24 GHz FMCW radar | Presence and stationary-human detection. Upstream already has mmWave UART hooks. |
| TI IWR6843 | 60–64 GHz FMCW | Point clouds and micro-Doppler. Doesn't go through walls. |
| Infineon BGT60 | ~58–63.5 GHz FMCW | Vitals and micro-Doppler at short range |
| Qorvo DW3000 | UWB ch5 and ch9 | Ranging and CIR-based sensing |

The values in this table are standard datasheet ranges written from knowledge. Only the ESP32-C5, AX210 and PicoScenes facts were checked against sources in this pass, so re-check the datasheets before buying.

## 5. Stage 2 research (hypotheses)
**H1, beamforming and phased CSI arrays.** Phase-synchronized multi-antenna CSI can steer and estimate angle of arrival. Single-chain ESP32s can't phase-align across boards without a shared clock, so start with the AX210 (2x2) or SDR.
- Kotaru et al., *SpotFi: Decimeter Level Localization Using WiFi*, SIGCOMM 2015.
- Adib & Katabi, *See Through Walls with WiFi!*, SIGCOMM 2013.

**H2, counting the walls penetrated.** Each wall adds a roughly discrete attenuation step and an extra delay or multipath component.
- The trade-off: higher frequency and wider bandwidth give finer delay resolution (160 MHz gives about 1.9 m path-length bins, from c/B). They also lose more power per wall. Lower frequency (sub-GHz, 2.4 GHz) penetrates better but resolves less.
- Method: calibrate per-wall attenuation at 0.9, 2.4, 5 and 6 GHz across known walls, then fit a classifier on the multi-band RSSI/CSI-delay features. ruview ADR-073 (multi-frequency mesh scan) is a starting point.
- Xie et al., *Precise Power Delay Profiling with Commodity WiFi* (Splicer), MobiCom 2015.

**H3, human vs dog/cat.** Gait micro-Doppler (cadence, limb count, body height) and breathing rate differ by species. A typical adult rests at about 12–20 breaths/min, while cats and dogs breathe faster and have a smaller chest displacement.
- Wang et al., *Understanding and Modeling of WiFi Signal Based Human Activity Recognition* (CARM), MobiCom 2015.
- Zhao et al., *Through-Wall Human Pose Estimation Using Radio Signals*, CVPR 2018.
- Adib et al., *Smart Homes that Monitor Breathing and Heart Rate*, CHI 2015.
- Geng, Huang & De la Torre, *DensePose From WiFi*, arXiv:2301.00250.
- A specific WiFi pet-vs-human paper hasn't been verified yet, so treat this as an open gap.

## 6. Ethics and legal
- Sense only spaces you own or have explicit consent for. Through-wall sensing of neighbors or of others without consent may break privacy and wiretap laws.
- Stay within FCC Part 15 limits (15.247 for 2.4/5.8 GHz and 902–928 MHz, 15.407 for U-NII including 6 GHz LPI/VLP, 15.519 for UWB). Don't modify radios to exceed certified power. Transmitting at 6 GHz requires certified devices and indoor-only operation for LPI.
