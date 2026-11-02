# lovebird

Two internet-bonded tamagotchi pets. Each pet is a Waveshare ESP32-S3-Touch-LCD-1.28 round touch screen running MicroPython firmware: a face you can stroke, stats you keep alive, and a small vocabulary of gestures (hug, feed, pet, short messages) that travel over MQTT to the other pet, wherever it lives.

## Status

The way is being charted with wayfinder. The map lives in this repo's GitHub issues. See [docs/agents/issue-tracker.md](docs/agents/issue-tracker.md) for how the tracker is driven (note: no `gh` CLI on the dev machine, REST API + token is used instead).

## Hardware (v0)

- 2x Waveshare ESP32-S3-Touch-LCD-1.28, round 240x240 GC9A01 touch display, CST816S touch, QMI8658 IMU (accelerometer + gyro), 16MB flash, USB-C, LiPo charging on board. Input is touch and motion only: there is no joystick on this board (verified against the Waveshare wiki)
  - Official product page (verified 2026-11-02): https://www.waveshare.com/esp32-s3-touch-lcd-1.28.htm (about $21.99 each)
  - Waveshare EU store (verified 2026-11-02): https://www.waveshare.eu/en/ESP32-S3-Touch-LCD-1.28.htm
  - Also stocked by EU hobby-electronics shops (berrybase.de, exp-tech.de); their shops are JS-rendered, so check stock on the site directly
- v0 runs USB powered. Batteries, buzzers and shells come later.

## Architecture

```
  [ pet one: ESP32-S3 board ]        [ pet two: ESP32-S3 board ]
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