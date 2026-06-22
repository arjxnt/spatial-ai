# RescueSense

WiFi-based through-wall sensing for fire and rescue operations. Detects survivors, monitors breathing and heart rate, and alerts on firefighter falls — using $9 ESP32 nodes and existing WiFi signals. No cameras, no contact, no line of sight.

---

## What it does

When people breathe or move, they disturb the radio waves already filling a building. RescueSense captures those disturbances using Channel State Information (CSI) from ESP32 nodes and runs them through a trained model to answer the questions that matter at an incident:

- Is there someone alive in that room?
- Where are they?
- Are they breathing?
- Did a firefighter just go down?

It works through drywall, wood framing, furniture, and smoke. It does not work through reinforced concrete or metal-clad walls.

---

## Hardware

| Device | Role | Cost |
|--------|------|------|
| ESP32-S3 (8MB flash) | CSI sensing node | ~$9 |
| ESP32-S3 SuperMini (4MB) | Compact sensing node | ~$6 |
| ESP32-C6 + Seeed MR60BHA2 | Node with onboard 60 GHz mmWave fallback | ~$15 |
| Raspberry Pi 4/5 or any Linux box | Runs the sensing server | — |

You need at minimum two nodes per room (one transmitting, one receiving). Three nodes in a triangle gives better position resolution. Nodes can be tossed through a window or door before entry.

---

## What the sensors can actually detect

| Capability | Range | Notes |
|---|---|---|
| Breathing rate | 6–30 BPM | Reliable when subject is still; degrades with heavy motion |
| Heart rate | 40–120 BPM | Requires 10–15 s of stillness to stabilize |
| Presence (through wall) | Up to ~10 m | Drops with wall thickness and material |
| Fall detection | <200 ms latency | Trained on collapse signatures, not stumbles |
| Person count | 1–4 people | Accuracy degrades past 3 in small rooms |
| Pose / body position | 17 keypoints | Usable for downed vs. standing, not fine motor |

Detection accuracy: **87.2% AP@50** on held-out test data (same layout). Accuracy drops ~15 percentage points when layout changes without re-calibration. Calibration takes one empty-room sweep (~30 seconds).

---

## Deployment

### Quick start

```bash
# Clone
git clone https://github.com/arjxnt/spatial-ai.git
cd spatial-ai

# Install Python dependencies (sensing server)
pip install -r requirements.txt

# Flash an ESP32-S3 node (requires ESP-IDF v5.4)
cd firmware/esp32-csi-node
idf.py -p /dev/ttyUSB0 flash

# Provision the node with your network
python provision.py --port /dev/ttyUSB0 --ssid "YourNet" --password "secret" --target-ip 192.168.1.20

# Start the sensing server
python -m wifi_densepose.server

# Open the dashboard
open ui/index.html
```

### At a structure fire

1. Drop 2–3 nodes inside doorways or through windows before entry — they boot in ~4 seconds.
2. Open the dashboard on a tablet at incident command.
3. Run a 30-second calibration sweep (empty hallway if possible, otherwise skip — the model has a no-calibration fallback with ~10% accuracy penalty).
4. The dashboard shows room occupancy, breathing status per detected person, and a MAYDAY alert if any node detects a stationary person for >15 seconds.

### HAZMAT / confined space

Nodes run fully passive after boot — no crew entry required to maintain coverage. Set the fall-detection threshold lower (`--fall-sensitivity high`) for SCBA-limited mobility. Vital sign logs are written locally to the server and timestamped for incident reports.

---

## Architecture

```
ESP32 nodes  →  CSI frames (100 Hz)  →  sensing server (Rust)
                                              │
                               ┌──────────────┼──────────────┐
                          signal DSP      NN inference    vitals
                          (phase clean,   (17-keypoint    (bandpass
                           multistatic)    pose model)     BPM/HR)
                                              │
                                         dashboard (JS)
                                         MQTT / Home Assistant
```

The Rust sensing server handles CSI ingestion, signal processing, and inference. The web dashboard polls the server over a local websocket — no internet required. MQTT output is available for integration with existing dispatch or monitoring systems.

Source layout:
- `v2/` — Rust workspace (15 crates: core, signal processing, NN inference, vitals, hardware, CLI)
- `firmware/esp32-csi-node/` — ESP32 C firmware
- `ui/` — Web dashboard
- `python/` / `archive/v1/` — Python reference implementation

---

## Building from source

```bash
# Rust (requires Rust 1.85+)
cd v2
cargo build --workspace --no-default-features

# Run tests (1,463 tests)
cargo test --workspace --no-default-features

# Python proof verification
python archive/v1/data/proof/verify.py
# Should print: VERDICT: PASS
```

---

## Limitations

- **Requires WiFi infrastructure or a hotspot.** Nodes need a network to send CSI data to the server. In a no-infrastructure scenario, one node can act as an AP for the rest.
- **Concrete and metal walls block signals.** Reinforced concrete, elevator shafts, and metal-clad structures significantly reduce or eliminate detection.
- **Accuracy drops in new environments without calibration.** The model was trained on specific room layouts. An empty-room baseline sweep before incident deployment improves accuracy by ~15 percentage points.
- **Vital signs require stillness.** Breathing and heart rate extraction is unreliable when the subject is moving. Fall detection works on the transition to stillness.
- **Person count tops out around 3–4.** Signal overlap between people in small rooms degrades per-person isolation.
- **Not a medical device.** Vital sign readings are for situational awareness. Do not use as a primary diagnostic.

---

## License

MIT. Based on [RuView](https://github.com/ruvnet/RuView) by ruvnet.
