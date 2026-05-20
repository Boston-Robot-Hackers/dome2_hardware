# Robot Design Document

Living design document: mechanical, electrical, software, controls, integration, and open decisions.

---

## 1. Bill of Materials

| Item | Link | Role | Status |
|---|---|---|---|
| Raspberry Pi 4 Model B | [Manufacturer](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) | Main ROS computer | Known |
| Raspberry Pi I2C Shim | TBD | I2C expansion | Known — exact model TBD |
| Waveshare UPS Module 3S | [Manufacturer](https://www.waveshare.com/product/ups-module-3s.htm) | Battery UPS, 5 V supply | Known |
| Panasonic NCR18650GA cells ×3 (+1 spare) | [Manufacturer](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA) | 3S battery pack | Known |
| Pololu 25D HP 34:1 gearmotor w/encoder ×4 | [Pololu #4844](https://www.pololu.com/product/4844) | Drive motors | **Chosen** |
| RoboClaw 2x5A | [Pololu](https://www.pololu.com/product/2394) | Motor controller | **Chosen** |
| Cytron MDD10A | [Cytron](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels) | Backup motor controller | Backup |
| Wheels ×4 | TBD | Drive wheels | Required — 100 mm diameter |
| Camera | TBD | Vision sensor | TBD |
| Lidar | TBD | 2D navigation sensor | TBD |
| I2C peripherals | TBD | Battery/status/sensor peripherals | TBD |

---

## 2. Component Specifications

### [Raspberry Pi 4 Model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
Power: 5 V USB-C from UPS module. Interfaces: USB (sensors/camera/lidar), GPIO (3.3 V safe only), I2C (check address conflicts), Ethernet/Wi-Fi.

### Raspberry Pi I2C Shim
3.3 V logic, SDA/SCL on GPIO2/GPIO3. Verify all connected devices use 3.3 V logic or have level shifting. Exact product TBD.

### [Waveshare UPS Module (3S)](https://www.waveshare.com/product/ups-module-3s.htm)
- Input: 3S Li-ion, 9.0–12.6 V
- 5 V output: 5 A / 25 W (Pi + USB peripherals)
- 3.3 V output: 300 mA
- XH2.54 battery-voltage output: unregulated pack voltage (9.0–12.6 V) — **motor power supply**
- Optional I2C: battery voltage, current, power status
- Charge and discharge simultaneously: yes
- **Single switch controls both 5 V and XH2.54 outputs** — both rails go down together
- Charge input: 12.6 V / 2 A external power adapter
- Verify XH2.54 connector and internal trace current rating for motor use (XH connectors typically rated ~3 A; motor stall can reach 20 A)

### [Panasonic NCR18650GA Cells](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA)
4 cells (3 installed, 1 spare). Per cell: 18650, Li-ion, 3450 mAh, 3.6–3.7 V nominal, 4.2 V full, 10 A continuous.
3S pack: 11.1 V nominal / 12.6 V full, 3450 mAh, ~38 Wh. Runtime: ~3.8 h @ 10 W · ~1.9 h @ 20 W · ~1.3 h @ 30 W.

### [Pololu 34:1 Metal Gearmotor 25Dx67L mm HP 12V](https://www.pololu.com/product/4844) — Chosen
Pololu #4844. 12 VDC, 300 RPM no-load, 4 mm D-shaft, 25 mm body.
- No-load: 300 mA, ~3.6 W
- Stall: 5.0 A, ~60 W, 11 kg·cm (1.08 N·m)
- Encoder: quadrature Hall effect, 48 CPR motor shaft, 1632 CPR output shaft
- 4 motors no-load: 1.2 A, ~14.4 W · paired channel stall: ~10 A — **at RoboClaw 2x5A 10 A peak limit ⚠️**
- Verify encoder output signal voltage before connecting to Pi GPIO.

### [RoboClaw 2x5A Motor Controller](https://www.pololu.com/product/2394) — Chosen
2-channel brushed DC, 6–34 V, 5 A continuous / 10 A peak per channel.
Interfaces: USB, TTL serial, RC, analog. Dual quadrature encoder inputs. Built-in speed/position PID.
Paired 34:1 motors stall at ~10 A/channel — exactly at peak limit, **zero margin**. Acceleration limiting and stall avoidance are essential.

---

## 3. Motor Sizing

Pololu 25D HP 34:1, 100 mm wheels, ~2 kg robot:
- 1.0 m/s target requires ~191 RPM; motor gives 300 RPM no-load → ~1.57 m/s no-load, ~36% margin
- Torque: 11 kg·cm stall — more than adequate; limit acceleration in software
- Encoder: 1632 CPR output (48 × 34) — good odometry resolution

---

## 4. Electrical Design

### Power Budget — 5 V Rail (25 W max)
- Raspberry Pi 4B: 600 mA idle / 1.2 A typical / 2.5 A heavy — source: RPi foundation
- USB camera: ~300 mA · USB lidar: ~500 mA · I2C: negligible
- **Typical total: ~2.0 A / ~10 W — 60% headroom ✓**

### Power Budget — Battery Rail (9.0–12.6 V)
Per-motor current: 300 mA no-load, ~500 mA typical driving, 5.0 A stall.

| Condition | Motor current | Battery current | Power @ 11 V | Notes |
|---|---|---|---|---|
| Idle | 4 × 300 mA = 1.2 A | ~2.3 A | ~25 W | includes ~12 W for 5V rail (85% eff.) |
| Typical driving | 4 × 500 mA = 2.0 A | ~3.1 A | ~36 W | 500 mA/motor indoor flat surface |
| Heavy load | 4 × 1.5 A = 6.0 A | ~7.1 A | ~78 W | slopes, acceleration bursts |
| Stall (worst case) | 4 × 5.0 A = 20 A | ~21 A | ~231 W | exceeds cell 10 A rating ⚠️ avoid |

**Runtime (38 Wh pack):** typical driving ~60 min · heavy load ~29 min · idle ~91 min

**Key limits:**
- 5 V rail: 2 A of 5 A used ✓ · battery typical: 3 A of 10 A cell rating ✓
- RoboClaw 2x5A: 5 A continuous / 10 A peak per channel — paired stall hits peak exactly ⚠️
- Stall: 20 A total exceeds cell continuous rating — avoid sustained stall ⚠️
- Keep typical per-channel motor current ≤5 A (≤2.5 A per motor) via acceleration limits and speed cap
- Fuse: 10 A for bench test; do not raise above 10 A (controller peak limit)

### Switching and Protection

**UPS switch** cuts both 5 V and XH2.54 12 V simultaneously — single switch controls everything.
- Wire gauge: minimum 16 AWG for motor rail; 14 AWG preferred.
- Verify XH2.54 connector and UPS trace can handle motor operating current.

### Electrical Risks
- 5 V rail overload (Pi + USB near UPS 5 A limit, especially Pi 5 config)
- Motor stall current (20 A) exceeds cell continuous rating
- I2C / GPIO at wrong voltage level
- Noise coupling between motor power and logic wiring

### Compatibility Notes
- Motor power from XH2.54 12 V output — not 5 V rail.
- All control electronics share common ground.
- GPIO and I2C signals must be 3.3 V safe.
- Encoder power 5 V; verify signal voltage before connecting to Pi GPIO.
- I2C address conflicts must be checked once all devices are selected.
- Avoid sustained motor stall.

---

## 5. Integration Plan


1. Confirm mechanical layout and component placement.
2. Validate battery and UPS wiring without motors.
3. Bring up Pi on regulated 5 V.
4. Confirm I2C bus and power-monitoring interface.
5. Add RoboClaw, test motor power separately.
6. Integrate motor control with ROS / Linorobot2.
7. Add encoders, validate odometry.
8. Add sensors, confirm full power budget.
9. Run controlled driving tests.

---

## 6. Test Plan

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

---

## 7. Alternate Configuration: Pi 5 + OAK-D Lite

Swap-in upgrade. Motors, motor controller, chassis, battery unchanged.

| Component | Base | Alternate |
|---|---|---|
| Compute | Raspberry Pi 4B | Raspberry Pi 5 |
| Vision | TBD USB/CSI camera | [Luxonis OAK-D Lite](https://shop.luxonis.com/products/oak-d-lite-1) |

### [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)
Replaces Pi 4B. Higher performance, native USB 3.0. GPIO/I2C pinout compatible with Pi 4B.
- Idle: ~1.0 A / ~5 W · typical: ~1.8 A / ~9 W · heavy: ~3.5 A / ~17.5 W (RPi foundation)
- Requires 5V/5A supply — same Waveshare UPS, near limit under heavy load

### [Luxonis OAK-D Lite](https://shop.luxonis.com/products/oak-d-lite-1)
Replaces generic USB camera. Stereo depth + RGB + onboard MyriadX VPU.
- Interface: USB 3.0 (USB-C), bus-powered
- Typical: ~400 mA / ~2 W · peak (active inference + streaming): up to **1.0 A / ~5 W**
- ROS 2 driver: `depthai-ros`

### 5 V Rail — Pi 5 + OAK-D Lite

| Load | Typical | Peak | Notes |
|---|---|---|---|
| Raspberry Pi 5 | 1.8 A / 9 W | 3.5 A / 17.5 W | RPi foundation |
| OAK-D Lite | 0.4 A / 2 W | 1.0 A / 5 W | peak under active inference |
| USB lidar | 0.5 A / 2.5 W | 0.5 A / 2.5 W | |
| **Total** | **2.7 A / 13.5 W ✓** | **5.0 A / 25 W ⚠️** | at UPS hard limit |

Peak (heavy CPU + active OAK-D + lidar) hits UPS 5 A limit exactly — no margin. Avoid simultaneous sustained heavy CPU and active inference, or upgrade 5 V supply.

**Runtime:** 5 V rail ~16 W typical from battery · typical total ~40 W → **~57 min** · heavy ~53 W → **~43 min**

### Open Questions — Pi 5 config
- Linorobot2 compatibility with Pi 5 / Ubuntu confirmed?
- `depthai-ros` stable on ROS 2 Humble/Jazzy?
- Does OAK-D Lite depth replace lidar or supplement it?
- UPS 5 V output stable at 4.5–5 A sustained — measure under real load.

---

## 8. Open Questions

- Final chassis design?
- Drive configuration confirmed (differential, 4-wheel paired)?
- Verify Pololu 34:1 specs from datasheet (no-load speed, stall current, stall torque).
- Configure RoboClaw 2x5A acceleration and current limits to stay within 5 A continuous per channel.
- Which sensors required for first working version?
- What payload margin beyond ~2 kg robot mass?
- Initial ROS / Linorobot2 bringup path?
- Raspberry Pi RAM size?
- Exact I2C shim model?
- Charger/power adapter model?
- Wire gauge and connector for motor rail — verify against real load current.
- Measure actual motor current under real load at 1.0 m/s.
- Encoder output signal voltage — 3.3 V safe or level shifting needed?
- RoboClaw 2x5A to Pi: USB or TTL serial?
- Battery monitoring integrated into ROS?
- Are three installed cells new and matched?
- Cytron MDD10A backup: which component handles encoder PID if used?

---

## 9. Backup Components

### [Cytron MDD10A](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels)
Use if RoboClaw 2x5A unavailable. **No onboard encoder PID** — PID must run on Pi or separate microcontroller.
7–30 V, 10 A continuous / 30 A peak per channel. Interfaces: PWM, RC, analog only.

---

## 10. Superseded Components

### [DFRobot FIT0186 Gearmotor](https://www.dfrobot.com/product-634.html)
Superseded by Pololu 34:1. DigiKey 1738-1106-ND. 12 V, 251 RPM, 7 A stall (14 A/channel paired — exceeds RoboClaw 2x5A peak), 700 CPR output encoder.

### Motor Comparison: FIT0186 vs Pololu 34:1

| Parameter | FIT0186 | Pololu 34:1 |
|---|---|---|
| No-load speed | 251 RPM | 300 RPM |
| No-load current | 350 mA / 4.2 W | 300 mA / 3.6 W |
| Stall current | 7.0 A / 84 W | 5.0 A / 60 W |
| Stall torque | 18 kg·cm | 11 kg·cm |
| Encoder CPR (output) | 700 | 1632 |
| 4-motor stall current | 28 A | 20 A |
| Paired channel stall | 14 A — exceeds RoboClaw 2x5A | 10 A — within peak |
| No-load speed (100 mm wheel) | ~1.31 m/s | ~1.57 m/s |
