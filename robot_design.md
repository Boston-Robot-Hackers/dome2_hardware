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
| DFRobot FIT0186 gearmotors with encoders, quantity 4 | [Manufacturer](https://www.dfrobot.com/product-634.html), [DigiKey](https://www.digikey.com/en/products/detail/dfrobot/FIT0186/6588528) | Drive motors and encoder feedback | Known | 12 V, 251 RPM, 43.8:1, 18 kg.cm stall torque, 700 CPR output encoder, 7 A stall each |
| RoboClaw 2x5A | [Pololu](https://www.pololu.com/product/2394) | Candidate dual motor controller | Alternative | Two channels: front+rear left motors on left channel, front+rear right motors on right channel |
| Motor power fuse | TBD | Motor rail protection | Required | Initial bench-test target: 7.5 A or 10 A automotive blade fuse |
| Motor power switch / e-stop | TBD | Disable motor power while Pi stays on | Required | DC-rated switch required |
| Camera | TBD | Vision sensor | TBD | USB or CSI TBD |
| Lidar | TBD | 2D navigation sensor | TBD | Exact model TBD |
| I2C peripherals | TBD | Battery/status/sensor peripherals | TBD | Track addresses and voltage levels |

## 4. Mechanical Design

### Chassis

| Item | Current Decision | Notes |
|---|---|---|
| Chassis type | TBD | Differential drive assumed until confirmed |
| Frame material | TBD | Consider stiffness, weight, and ease of modification |
| Wheel layout | Four powered wheels, left/right paired drive | Front+rear left motors share left controller channel; front+rear right motors share right channel |
| Mounting system | TBD | Leave room for Pi, batteries, motor driver, wiring, and sensors |

### Mechanical Requirements

- Provide secure battery mounting.
- Keep electronics accessible for debugging.
- Route high-current motor wiring away from low-level signal wiring where practical.
- Leave room for strain relief on battery, motor, and USB connections.
- Provide a stable sensor mounting location.

## 5. Motor Sizing

Current assumptions:

| Parameter | Value |
|---|---:|
| Robot mass | ~2 kg |
| Wheel diameter | 100 mm / 0.10 m |
| Wheel circumference | ~0.314 m |
| Target top speed | 1.0 m/s |
| Wheel RPM required for 1.0 m/s | ~191 RPM |
| FIT0186 no-load speed at 12 V | 251 RPM |
| No-load vehicle speed at 12 V | ~1.31 m/s |
| Approx no-load vehicle speed at 9.0 V | ~0.99 m/s |
| Approx no-load vehicle speed at 12.6 V | ~1.38 m/s |
| RPM margin at 12 V for 1.0 m/s target | ~24% |
| Stall wheel force, one motor | ~35 N |
| Stall wheel force, four motors | ~142 N |
| Ideal acceleration at full stall force, before traction losses | ~71 m/s^2 |

Sizing conclusion:

- With 100 mm wheels, the FIT0186 is not excessive on speed. A 1.0 m/s target requires about 191 RPM, below the 251 RPM no-load rating.
- For a roughly 2 kg robot, the motors are very strong on torque. The theoretical four-motor stall force is about 142 N, far more than needed for normal indoor driving and likely above available wheel traction.
- Practical acceleration should be limited in software or motor-controller configuration. A target acceleration around 0.5-1.0 m/s^2 would need only a small fraction of available stall torque.
- The design risk is mainly electrical: stall current is 7 A per motor, so acceleration limiting, current limiting, fusing, and avoiding mechanical stalls are important. Four FIT0186 motors raise the stall-current case to 28 A total, or 14 A stall per controller channel when front/rear motors are paired left and right.


### Ideal Replacement Motor Target

If replacing the FIT0186 motors with a better-matched, lower-power option, keep roughly the same speed class but reduce torque and stall current.

| Spec | Target |
|---|---:|
| Motor type | Brushed DC gearmotor with quadrature encoder |
| Rated voltage | 12 V nominal |
| Gearbox output speed | ~250-350 RPM no-load at 12 V |
| Minimum useful loaded speed | ~200 RPM |
| No-load current | <150-250 mA per motor |
| Typical running current | ~300-800 mA per motor |
| Stall current | ~2-3 A per motor |
| Stall torque | ~0.4-0.8 N.m |
| Encoder | Quadrature, preferably 3.3 V-safe or open-collector |
| Encoder resolution | ~300-1000 CPR at gearbox output |
| Shaft | Match wheel hub, ideally 4-6 mm D-shaft |
| Comfortable motor driver class | 2x3A to 2x5A |

The current FIT0186 motors are acceptable on speed, but high on torque and stall current. A better match would be around 250-350 RPM with 2-3 A stall current and 0.4-0.8 N.m stall torque per motor. That would keep the 1.0 m/s design target while reducing stress on the battery, fuse, switch, wiring, and motor controller.

## 6. Electrical Design

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
| Battery rail | 9.0-12.6 V | Motor driver, drive motors | Must be separately rated/fused for motor current |
| 5 V rail | 5 V | Raspberry Pi, USB peripherals | Limited by UPS output |
| 3.3 V logic | 3.3 V | Pi GPIO, I2C logic | Must remain Pi-safe |

### Electrical Risks

- 5 V rail overload from Raspberry Pi plus USB devices.
- Motor current exceeding battery, wiring, connector, or driver limits.
- I2C or GPIO devices using unsafe signal voltage.
- Grounding or noise problems between motor power and logic electronics.
- Battery pack safety if cells are not matched or protected correctly.

## 7. Compute and Control

### Main Computer

| Component | Role | Status |
|---|---|---|
| [Raspberry Pi 4 Model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) | Main ROS computer | Known |
| [DFRobot FIT0186 gearmotors](https://www.dfrobot.com/product-634.html) | Drive motors with encoders | Known |
| [RoboClaw 2x5A](https://www.pololu.com/product/2394) | Alternative dual motor controller | Candidate |

### Control Path

```text
ROS / Linorobot2
   └──> Motor control interface
          └──> Motor driver
                 └──> Drive motors
```

### Open Control Decisions

- Whether motor control is handled by a separate microcontroller.
- Motor driver model: RoboClaw 2x5A is a candidate for left/right paired drive, but each channel would see two motors in parallel and battery-current path must be resolved.
- FIT0186 encoder interface and signal level.
- ROS topics and message flow.
- Odometry source and calibration workflow.

## 8. Sensors and Peripherals

| Device | Interface | Power | Status | Notes |
|---|---|---:|---|---|
| Camera | USB or CSI | TBD | TBD | Exact model TBD |
| Lidar | USB or serial | TBD | TBD | Exact model TBD |
| [DFRobot FIT0186 encoders](https://www.dfrobot.com/product-634.html) | GPIO or microcontroller | 5 V encoder power | Known motor, interface TBD | Must be 3.3 V safe if connected to Pi |
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

## 13. Open Questions

- What is the final chassis design?
- What drive configuration will be used?
- Four DFRobot FIT0186 motors are currently assumed; confirm whether all four are driven independently.
- Can one RoboClaw 2x5A safely drive paired front/rear motors on each left/right channel, or is a higher-current controller needed?
- Will motor power bypass the Waveshare UPS module through a separate protected/fused battery rail?
- What continuous motor current should be assumed under real robot load at the 1.0 m/s target speed and ~2 kg robot mass?
- What acceleration limit should be configured to keep motor current reasonable?
- Should the FIT0186 motors be kept, or replaced with lower-current 250-350 RPM motors closer to the ideal target?
- Are the FIT0186 encoder output signals safe for Raspberry Pi GPIO?
- Which sensors are required for the first working version?
- Which microcontroller, if any, will sit between ROS and the motor driver?
- What is the target runtime?
- What payload margin is needed beyond the current ~2 kg robot mass?
- What is the initial ROS / Linorobot2 bringup path?
