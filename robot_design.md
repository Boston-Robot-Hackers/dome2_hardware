# Robot Design Document

## Purpose

Living design document for the new robot. This file tracks the overall system design across mechanical structure, electrical power, electronics, software, controls, integration, and open engineering decisions.

Detailed electrical component notes are maintained separately in `robot_electrical_components.md`.

---

## 1. Design Goals

- Build a reliable mobile robot platform suitable for ROS / Linorobot2 experimentation.
- Keep the design modular enough to change sensors, motor drivers, and compute hardware.
- Separate motor power from regulated logic power.
- Track compatibility constraints before committing to wiring or mounting decisions.
- Maintain enough documentation to support debugging, upgrades, and rebuilds.

## 2. System Overview

```text
Battery Pack
   ├──> Motor Power Rail ──> Motor Driver ──> Drive Motors
   └──> UPS / Regulator ──> 5 V Rail ──> Raspberry Pi + Sensors + Peripherals

Raspberry Pi
   ├──> ROS / Linorobot2
   ├──> Microcontroller or Motor Controller
   ├──> Sensors
   └──> Network / Development Interface
```

## 3. Bill of Materials

| Item | Link | Role | Status | Notes |
|---|---|---|---|---|
| Raspberry Pi 4 Model B | [Manufacturer](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) | Main ROS computer | Known | RAM size TBD |
| Raspberry Pi I2C Shim | TBD | I2C expansion / wiring convenience | Known | Exact model TBD |
| Waveshare UPS Module 3S | [Manufacturer](https://www.waveshare.com/product/ups-module-3s.htm) | 3S battery UPS and 5 V supply | Known | 5 V / 5 A output; XH battery-voltage output is not motor-current rated |
| Panasonic NCR18650GA cells | [Manufacturer](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA) | 3S battery pack cells | Known | 3 installed, 1 spare |
| Pololu 25D HP 34:1 gearmotors with encoders, quantity 4 | [Pololu #4844](https://www.pololu.com/product/4844) | Drive motors and encoder feedback | **Chosen** | 12 V, 300 RPM no-load, 34:1, 11 kg·cm stall torque, 1632 CPR output encoder, 5.0 A stall each |
| RoboClaw 2x7A | [BasicMicro](https://www.basicmicro.com/RoboClaw-2x7A-Motor-Controller_p_55.html) | Dual motor controller | **Chosen** | 7 A continuous / ~15 A peak per channel; USB, TTL serial, RC, analog; onboard encoder PID; paired 34:1 motors = ~10 A stall per channel — within peak rating |
| Cytron MDD10A | [Cytron](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels) | Backup dual motor controller | Backup | 10 A continuous / 30 A peak per channel, 7–30 V; PWM/RC/analog only — no onboard encoder PID; needs external PID loop |
| DFRobot FIT0186 gearmotors with encoders, quantity 4 | [Manufacturer](https://www.dfrobot.com/product-634.html) | Drive motors and encoder feedback | Superseded | 12 V, 251 RPM, 7 A stall each |
| RoboClaw 2x5A | [Pololu](https://www.pololu.com/product/2394) | Dual motor controller | Superseded | 5 A continuous / 10 A peak per channel |
| Motor power fuse | TBD | Motor rail protection | Required | Initial bench-test target: 7.5 A or 10 A automotive blade fuse |
| Motor power switch / e-stop | TBD | Disable motor power while Pi stays on | Required | DC-rated switch required |
| Camera | TBD | Vision sensor | TBD | USB or CSI TBD |
| Lidar | TBD | 2D navigation sensor | TBD | Exact model TBD |
| I2C peripherals | TBD | Battery/status/sensor peripherals | TBD | Track addresses and voltage levels |

## 4. Mechanical Design

### Chassis

- Chassis type: TBD (differential drive assumed)
- Frame material: TBD
- Wheel layout: four powered wheels, left/right paired — front+rear left on left channel, front+rear right on right channel
- Mounting: TBD — must fit Pi, batteries, RoboClaw, wiring, sensors

### Mechanical Requirements

- Provide secure battery mounting.
- Keep electronics accessible for debugging.
- Route high-current motor wiring away from low-level signal wiring where practical.
- Leave room for strain relief on battery, motor, and USB connections.
- Provide a stable sensor mounting location.

## 5. Motor Sizing

Pololu 25D HP 34:1, 100 mm wheels, ~2 kg robot:
- Robot: 2 kg, 100 mm wheel (0.314 m circumference), 1.0 m/s target → ~191 RPM required
- Motor: 300 RPM no-load at 12 V → ~1.57 m/s no-load, ~1.17 m/s at 9 V, ~36% margin at 12 V
- Current: 5.0 A stall per motor, ~10 A per paired channel, RoboClaw 2x7A peak ~15 A → ~5 A headroom
- Encoder: 1632 CPR output (48 × 34) — good odometry resolution
- Torque: 11 kg·cm stall — more than adequate for indoor ~2 kg robot; limit acceleration in software


## 6. Electrical Design

Electrical details are tracked in `robot_electrical_components.md`.

### Current Electrical Architecture

- 3S Li-ion battery pack powers the robot.
- Waveshare 3S UPS module provides regulated 5 V power.
- Raspberry Pi 4B runs from the UPS 5 V output.
- Motor driver should be powered directly from the 3S battery rail.
- Control electronics should share a common ground.

### Power Rails

- Battery rail: 9.0–12.6 V → RoboClaw 2x7A + drive motors (separately fused/switched)
- 5 V rail: 5 V → Raspberry Pi, USB peripherals (limited to 25 W by UPS)
- 3.3 V logic: Pi GPIO, I2C (Pi-safe only)

### Electrical Risks

- 5 V rail overload from Raspberry Pi plus USB devices.
- Motor current exceeding battery, wiring, connector, or driver limits.
- I2C or GPIO devices using unsafe signal voltage.
- Grounding or noise problems between motor power and logic electronics.
- Battery pack safety if cells are not matched or protected correctly.

## 7. Compute and Control

### Main Computer

- [Raspberry Pi 4 Model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) — main ROS computer
- [Pololu 25D HP 34:1 gearmotors](https://www.pololu.com/product/4844) ×4 — chosen drive motors
- RoboClaw 2x7A — chosen motor controller (onboard encoder PID)
- Cytron MDD10A — backup motor controller (no onboard PID)

### Control Path

```text
ROS / Linorobot2
   └──> Motor control interface
          └──> Motor driver
                 └──> Drive motors
```

### Open Control Decisions

- Whether RoboClaw 2x7A connects directly to Pi via USB/TTL or through a microcontroller.
- ROS topics and message flow for RoboClaw integration.
- Odometry source and calibration workflow.
- Pololu 34:1 encoder signal voltage — 3.3 V safe for Pi GPIO, or level shifting required.

## 8. Sensors and Peripherals

| Device | Interface | Power | Status | Notes |
|---|---|---:|---|---|
| Camera | USB or CSI | TBD | TBD | Exact model TBD |
| Lidar | USB or serial | TBD | TBD | Exact model TBD |
| Pololu 34:1 encoders | RoboClaw 2x7A encoder inputs | 5 V encoder power | Chosen motor, signal level TBD | Verify 3.3 V safe before connecting to Pi GPIO |
| I2C devices | I2C | TBD | TBD | Track addresses |

## 9. Software Design

### Target Stack

- Raspberry Pi OS or compatible Linux distribution.
- ROS / Linorobot2.
- Network access over Ethernet or Wi-Fi for development.

### Software Responsibilities

- Motor control integration.
- Odometry publishing.
- Sensor drivers.
- Battery monitoring if UPS I2C is used.
- Teleoperation and bringup scripts.
- Configuration files for robot dimensions, wheel parameters, and control tuning.

## 10. Integration Plan

1. Confirm mechanical layout and component placement.
2. Validate battery and UPS wiring without motors connected.
3. Bring up Raspberry Pi on regulated 5 V power.
4. Confirm I2C bus and any power-monitoring interface.
5. Add motor driver and test motor power separately.
6. Integrate motor control with ROS / Linorobot2.
7. Add encoders and validate odometry.
8. Add sensors and confirm full power budget.
9. Run controlled driving tests.

## 11. Test Plan

| Test | Purpose | Status |
|---|---|---|
| 5 V rail voltage test | Confirm stable Pi power | TBD |
| Battery rail voltage test | Confirm 3S voltage range | TBD |
| Idle current test | Estimate baseline runtime | TBD |
| Motor current test | Confirm driver and battery margin | TBD |
| I2C scan | Detect devices and address conflicts | TBD |
| GPIO signal check | Confirm 3.3 V-safe signals | TBD |
| ROS bringup test | Confirm software stack starts cleanly | TBD |
| Drive test | Confirm motor direction and control | TBD |
| Odometry test | Confirm encoder direction and scale | TBD |

## 12. Documentation Index

| Document | Purpose |
|---|---|
| `robot_design.md` | Overall system design |
| `robot_electrical_components.md` | Electrical/electronic components, power, and compatibility |
| `2026-05-18-robot-electronics-summary.md` | Summary of earlier electronics documentation session |

## 13. Alternate Configuration: Pi 5 + OAK-D Lite

Swap-in upgrade path. Motors, motor controller, chassis, and battery unchanged.

| Component | Base Config | Alternate Config |
|---|---|---|
| Compute | Raspberry Pi 4B | Raspberry Pi 5 |
| Vision | TBD USB/CSI camera | [Luxonis OAK-D Lite](https://shop.luxonis.com/products/oak-d-lite-1) |
| USB | USB 2.0 (adequate) | USB 3.0 required for OAK-D Lite bandwidth |

### Raspberry Pi 5
- 5V USB-C, requires 5A/27W supply — same UPS module but at its limit under heavy load
- Idle: ~1A / ~5W · typical: ~1.8A / ~9W · heavy: ~3.5A / ~17.5W
- Better CPU performance for ROS 2 nav stack, point cloud processing
- Native USB 3.0 — required for OAK-D Lite

### OAK-D Lite
- USB 3.0 (USB-C), bus-powered from Pi 5
- Typical: ~400 mA / ~2W · active inference: ~500 mA / ~2.5W
- Provides: stereo depth (3D), RGB camera, onboard MyriadX VPU for neural net inference
- ROS 2 driver: `depthai-ros` package
- Replaces separate RGB camera + avoids separate depth sensor

### 5V Rail — Pi 5 config

| Load | Typical | Heavy | Notes |
|---|---|---|---|
| Raspberry Pi 5 | 1.8 A / 9 W | 3.5 A / 17.5 W | RPi foundation data |
| OAK-D Lite | 0.4 A / 2 W | 0.5 A / 2.5 W | USB bus powered |
| USB lidar | 0.5 A / 2.5 W | 0.5 A / 2.5 W | |
| I2C | negligible | negligible | |
| **Total** | **2.7 A / 13.5 W ✓** | **4.5 A / 22.5 W ⚠️** | 5 A / 25 W limit |

Heavy load approaches UPS 5A limit — avoid sustained heavy CPU + active OAK-D inference simultaneously, or upgrade to a higher-current 5V supply.

### Runtime — Pi 5 config
Battery rail unchanged (~22W motors + ~2W RoboClaw). 5V rail draws ~15W typical from battery (13.5W / 85% UPS efficiency).
- Typical total: ~39W → **~58 min** (vs ~60 min Pi 4B config — negligible difference)
- Heavy load total: ~50W → **~46 min**

### Open questions — Pi 5 config
- Linorobot2 compatibility with Pi 5 / Ubuntu for Pi 5 confirmed?
- `depthai-ros` package stable on ROS 2 Humble/Jazzy?
- UPS module 5V output stable at 4.5A sustained — measure under real load before relying on it.
- Does OAK-D Lite depth replace lidar, supplement it, or is lidar still needed for nav?

---

## 14. Open Questions

- What is the final chassis design?
- What drive configuration will be used?
- Verify Pololu 34:1 exact no-load speed, stall current, and stall torque from datasheet.
- Verify RoboClaw 2x7A exact peak current rating from BasicMicro datasheet.
- Cytron MDD10A backup: if used, which component handles encoder PID — Pi or separate microcontroller?
- Motor power: confirm separate fused/switched battery rail bypassing Waveshare UPS module.
- What continuous motor current under real load at 1.0 m/s, ~2 kg robot?
- What acceleration limit to configure in RoboClaw to keep current reasonable?
- Are Pololu 34:1 encoder output signals 3.3 V safe for Pi GPIO?
- Which sensors required for first working version?
- RoboClaw 2x7A to Pi: USB or TTL serial?
- What is target runtime?
- What payload margin beyond ~2 kg robot mass?
- What is initial ROS / Linorobot2 bringup path?
