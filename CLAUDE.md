# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## What This Repo Is

Hardware design documentation for **Dome3** — a differential-drive, 4-wheeled indoor robot built on ROS 2 / Linorobot2. No source code lives here. Work is editing Markdown design docs and keeping specs consistent.

---

## Document Structure

| File | Purpose |
|---|---|
| `robot_design.md` | **Primary living document.** BOM, component specs, power budget, integration plan, test plan, open questions. Always authoritative. |
| `archive/` | Superseded drafts. Do not treat as current — specs here may contradict `robot_design.md`. |
| `pics/` | Photos of physical hardware. |

---

## Key Design Facts

**Platform:** Raspberry Pi 4B (base) or Pi 5 (alternate) · ROS 2 / Linorobot2 · differential drive, 4 motors paired L/R.

**Power architecture — two rails:**
- **5 V rail:** Waveshare UPS Module 3S → Pi + USB peripherals. 5 A / 25 W max.
- **Motor/battery rail:** 3S Li-ion pack (11.1 V nom / 12.6 V full, 38 Wh) → fuse → RoboClaw VIN.
- UPS single switch cuts both rails simultaneously. Motor rail requires separate fuse + switch.

**Motor controller:** RoboClaw 2x5A (Pololu #2394). 5 A continuous / 10 A peak per channel. Has onboard quadrature encoder PID — no separate MCU needed. Interfaces: USB or TTL serial to Pi.

**Backup controller:** Cytron MDD10A — no onboard PID; encoder PID would run on Pi.

**Critical limits to preserve in all edits:**
- Paired motor stall = ~10 A/channel → exactly at RoboClaw 2x5A peak. Zero margin. Acceleration limiting is essential.
- 4-motor stall (20 A total) exceeds cell 10 A continuous rating. Sustained stall must be avoided.
- 5 V rail: Pi 5 + OAK-D Lite at peak = 5.0 A exactly — no margin.
- All GPIO/I2C signals must be 3.3 V safe.

**Encoder signal voltage:** unverified — must confirm 3.3 V safe before connecting to Pi GPIO.

---

## Document Conventions

- `robot_design.md` is the single source of truth. Superseded components go in **Section 10 (Superseded Components)** with a comparison table.
- Status column values: `Known`, `Known — exact model TBD`, `**Chosen**`, `Backup`, `Required — [constraint]`, `TBD`.
- Power budget tables include battery rail current (motor current + UPS 5V draw at 85% efficiency).
- Flag risks inline with ⚠️ and a short reason.
- Open questions live in **Section 8**; answer them in-place and remove when resolved.

---

## Software Stack (not in this repo)

Target: ROS 2 on Raspberry Pi, Linorobot2 framework. RoboClaw interfaces via USB or TTL serial. OAK-D Lite uses `depthai-ros` driver (needs USB 3.0 — Pi 5 preferred).
