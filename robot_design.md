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

## 3. Mechanical Design

### Chassis

| Item | Current Decision | Notes |
|---|---|---|
| Chassis type | TBD | Differential drive assumed until confirmed |
| Frame material | TBD | Consider stiffness, weight, and ease of modification |
| Wheel layout | TBD | Must match odometry assumptions and FIT0186 motor mounting |
| Mounting system | TBD | Leave room for Pi, batteries, motor driver, wiring, and sensors |

### Mechanical Requirements

- Provide secure battery mounting.
- Keep electronics accessible for debugging.
- Route high-current motor wiring away from low-level signal wiring where practical.
- Leave room for strain relief on battery, motor, and USB connections.
- Provide a stable sensor mounting location.

## 4. Electrical Design

Electrical details are tracked in `robot_electrical_components.md`.

### Current Electrical Architecture

- 3S Li-ion battery pack powers the robot.
- Waveshare 3S UPS module provides regulated 5 V power.
- Raspberry Pi 4B runs from the UPS 5 V output.
- Motor driver should be powered directly from the 3S battery rail.
- Control electronics should share a common ground.

### Power Rails

| Rail | Voltage | Loads | Notes |
|---|---:|---|---|
| Battery rail | 9.0-12.6 V | Motor driver, drive motors | High-current rail |
| 5 V rail | 5 V | Raspberry Pi, USB peripherals | Limited by UPS output |
| 3.3 V logic | 3.3 V | Pi GPIO, I2C logic | Must remain Pi-safe |

### Electrical Risks

- 5 V rail overload from Raspberry Pi plus USB devices.
- Motor current exceeding battery, wiring, connector, or driver limits.
- I2C or GPIO devices using unsafe signal voltage.
- Grounding or noise problems between motor power and logic electronics.
- Battery pack safety if cells are not matched or protected correctly.

## 5. Compute and Control

### Main Computer

| Component | Role | Status |
|---|---|---|
| Raspberry Pi 4 Model B | Main ROS computer | Known |
| DFRobot FIT0186 gearmotors | Drive motors with encoders | Known |

### Control Path

```text
ROS / Linorobot2
   └──> Motor control interface
          └──> Motor driver
                 └──> Drive motors
```

### Open Control Decisions

- Whether motor control is handled by a separate microcontroller.
- Motor driver model.
- FIT0186 encoder interface and signal level.
- ROS topics and message flow.
- Odometry source and calibration workflow.

## 6. Sensors and Peripherals

| Device | Interface | Power | Status | Notes |
|---|---|---:|---|---|
| Camera | USB or CSI | TBD | TBD | Exact model TBD |
| Lidar | USB or serial | TBD | TBD | Exact model TBD |
| DFRobot FIT0186 encoders | GPIO or microcontroller | 5 V encoder power | Known motor, interface TBD | Must be 3.3 V safe if connected to Pi |
| I2C devices | I2C | TBD | TBD | Track addresses |

## 7. Software Design

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

## 8. Integration Plan

1. Confirm mechanical layout and component placement.
2. Validate battery and UPS wiring without motors connected.
3. Bring up Raspberry Pi on regulated 5 V power.
4. Confirm I2C bus and any power-monitoring interface.
5. Add motor driver and test motor power separately.
6. Integrate motor control with ROS / Linorobot2.
7. Add encoders and validate odometry.
8. Add sensors and confirm full power budget.
9. Run controlled driving tests.

## 9. Test Plan

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

## 10. Documentation Index

| Document | Purpose |
|---|---|
| `robot_design.md` | Overall system design |
| `robot_electrical_components.md` | Electrical/electronic components, power, and compatibility |
| `2026-05-18-robot-electronics-summary.md` | Summary of earlier electronics documentation session |

## 11. Open Questions

- What is the final chassis design?
- What drive configuration will be used?
- How many DFRobot FIT0186 motors will be used?
- Which motor driver will be used?
- What is the expected continuous motor current under real robot load?
- Are the FIT0186 encoder output signals safe for Raspberry Pi GPIO?
- Which sensors are required for the first working version?
- Which microcontroller, if any, will sit between ROS and the motor driver?
- What is the target runtime?
- What is the target payload weight?
- What is the initial ROS / Linorobot2 bringup path?
