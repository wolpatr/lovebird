# lovebird

Two internet-bonded tamagotchi pets. Each pet is a Waveshare ESP32-C6-Touch-LCD-1.28 round touch screen running MicroPython firmware: a face you can stroke, stats you keep alive, and a small vocabulary of gestures (hug, feed, pet, short messages) that travel over MQTT to the other pet, wherever it lives.

## Status

The way is being charted with wayfinder. The map lives in this repo's GitHub issues. See [docs/agents/issue-tracker.md](docs/agents/issue-tracker.md) for how the tracker is driven (note: no `gh` CLI on the dev machine, REST API + token is used instead).

## Hardware (v0)

- 2x Waveshare ESP32-C6-Touch-LCD-1.28 (metal case variant, ordered 2026-11-02, delivery 15-20 days): round 240x240 GC9A01 touch display, CST816S touch, QMI8658 IMU (accelerometer + gyro), 16MB flash, USB-C, RTC chip, TF slot, LiPo charging on board. Single-core RISC-V ESP32-C6 at 160MHz, 512KB SRAM, no PSRAM. Input is touch and motion only.
  - Official product page (verified 2026-11-02): https://www.waveshare.com/esp32-c6-touch-lcd-1.28.htm ($19.99 bare / $24.99 metal case)
  - Note: originally charted on the ESP32-S3-Touch-LCD-1.28-B; re-targeted after the order landed on the C6 (same display, touch, IMU and flash; different chip)
- v0 runs USB powered. Batteries, buzzers come later; the metal case covers the shell question.

## Architecture

```
  [ pet one: ESP32-C6 board ]         [ pet two: ESP32-C6 board ]
        MicroPython                        MicroPython
      face / stats / touch               face / stats / touch
             |                                  |
             +------- TLS MQTT over the ------+
                            internet
                                |
                 [ Mosquitto broker in Docker ]
                   [ hermes-ubuntu-4gb-nbg1-1 ]
                     Hetzner cloud, Nuremberg
              QoS 1 queues hugs while a pet sleeps
```

## Software

- Firmware: MicroPython. The personality (faces, stats, moods) is Python data, editable live.
- Broker: Mosquitto 2.x in Docker, per-device accounts, offline queueing via QoS 1.
- Verbs: HUG, FEED, PET, TEXT, plus presence and battery reporting.

## Conventions

- No em dashes in any file, commit, issue or message. Use plain hyphens or restructure.
- Domain language lives in GLOSSARY.md (created when the first terms are resolved) and decisions in docs/adr/.