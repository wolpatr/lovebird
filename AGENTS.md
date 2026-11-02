# AGENTS.md

## Project facts

- Dev machine: Windows 11, git-bash shell, Python 3.11/3.12. `gh` CLI is NOT installed. GitHub auth: token in `~/.git-credentials`, user wolpatr.
- Firmware target: Waveshare ESP32-S3-Touch-LCD-1.28 (round GC9A01 touch display, CST816S touch, five-way joystick), MicroPython.
- Broker target: Mosquitto in Docker on hermes-ubuntu-4gb-nbg1-1 (Hetzner cloud, Ubuntu, 4GB RAM, 40GB disk, Nuremberg).
- Budget: about 50 euro total for two devices, stretchable to 100.
- Writing rule: no em dashes anywhere (files, issues, commits, messages). Use plain hyphens or restructure the sentence.
- The repo is public: never put credentials, server IPs or personal details in anything pushed or posted here.

## Agent skills

### Issue tracker

GitHub issues at github.com/wolpatr/lovebird. Driven via REST API (no gh CLI). See `docs/agents/issue-tracker.md`.

### Triage labels

Defaults kept as-is: needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `GLOSSARY.md` at repo root (created lazily), ADRs in `docs/adr/`. See `docs/agents/domain.md`.