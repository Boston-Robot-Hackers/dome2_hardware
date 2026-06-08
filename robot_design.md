# Robot Design Document

Living design document: mechanical, electrical, software, controls, integration, and open decisions.

---

## 1. Bill of Materials

| Item | Link | Role | Status |
|---|---|---|---|
| Raspberry Pi 4 Model B | [Manufacturer](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) | Main ROS computer | Known |
| Waveshare UPS Module 3S | [Manufacturer](https://www.waveshare.com/product/ups-module-3s.htm) | Battery UPS, 5 V supply | Known |
| Panasonic NCR18650GA cells ×3 (+1 spare) | [Manufacturer](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA) | 3S battery pack | Known |
| Pololu 25D MP 34:1 gearmotor w/encoder ×4 | [Pololu #4864](https://www.pololu.com/product/4864) | Drive motors | **Chosen** — ~$200 total ⚠️ cost; see JGA25-370 alternative (~$35) |
| Cytron MDD10A ×2 | [Cytron](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels) | Motor driver (2 per robot, 2 motors each) | **Chosen** |
| ESP32-S3 DevKitC-1 | [Espressif](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/hw-reference/esp32s3/user-guide-devkitc-1.html) | micro-ROS motor controller node | **Chosen** |
| RC Airplane Wheels 4" ×4 | TBD | Drive wheels | **Chosen** — 4" (~102 mm), 4 mm set-screw hub, foam/rubber |
| Luxonis OAK-D Lite | [Luxonis](https://shop.luxonis.com/products/oak-d-lite-1) | Vision sensor (stereo depth + RGB + VPU) | **Chosen** |
| LDROBOT LD19 | [LDROBOT](https://www.ldrobot.com/product/en/98) | 2D navigation lidar | **Chosen** |

---

## 2. Component Specifications

### [Raspberry Pi 4 Model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
Power: 5 V USB-C from UPS module. Interfaces: USB (sensors/camera/lidar), GPIO (3.3 V safe only), I2C (check address conflicts), Ethernet/Wi-Fi.

### [Waveshare UPS Module (3S)](https://www.waveshare.com/product/ups-module-3s.htm)
- Input: 3S Li-ion, 9.0–12.6 V
- 5 V output: 5 A / 25 W (Pi + USB peripherals)
- 3.3 V output: 300 mA
- XH2.54 battery-voltage output: unregulated pack voltage (9.0–12.6 V) — **motor power supply**
- Optional I2C: battery voltage, current, power status
- Charge and discharge simultaneously: yes
- **Single switch controls both 5 V and XH2.54 outputs** — both rails go down together
- Charge input: 12.6 V / 2 A external power adapter
- Verify XH2.54 connector and internal trace current rating for motor use (XH connectors typically rated ~3 A; 4-motor stall ~7.2 A chosen MP)

### [Panasonic NCR18650GA Cells](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA)
4 cells (3 installed, 1 spare). Per cell: 18650, Li-ion, 3450 mAh, 3.6–3.7 V nominal, 4.2 V full, 10 A continuous.
3S pack: 11.1 V nominal / 12.6 V full, 3450 mAh, ~38 Wh. Runtime: ~3.8 h @ 10 W · ~1.9 h @ 20 W · ~1.3 h @ 30 W.

### [Pololu 34:1 Metal Gearmotor 25Dx67L mm MP 12V](https://www.pololu.com/product/4864) — Chosen
Pololu #4864. ~$50/ea × 4 = ~$200 total. ⚠️ High unit cost — see JGA25-370 below for cheaper alternative.
12 VDC, 230 RPM no-load, 4 mm D-shaft, 25 mm body.
- No-load: 100 mA, ~1.2 W
- Stall: 1.8 A, 4.7 kg·cm (0.46 N·m)
- Encoder: quadrature Hall effect, 48 CPR motor shaft, 1632 CPR output shaft
- Paired channel stall: ~3.6 A — well within MDD10A 10 A continuous ✓
- 4-motor stall: 7.2 A — within cell 10 A continuous ✓
- Pololu continuous torque limit: 4 kg·cm — stall torque (4.7 kg·cm) above limit ⚠️ avoid sustained high-load
- Verify encoder signal voltage before connecting to ESP32 GPIO.

**MP vs HP:** MP chosen for lower current; HP #4844 fallback if torque insufficient. Full comparison in Section 9.

### JGA25-370 1:32 — Cheaper Alternative (not yet bought)

[Amazon B0GV7J5DBY](https://www.amazon.com/dp/B0GV7J5DBY) — ~$15–20 per 2-pack × 2 = **~$30–40 total** vs $200 for Pololu. Same 25 mm body, 4 mm D-shaft.
- 12 VDC, 260 RPM → 1.38 m/s ✓
- Stall: 6.2 A, 8.7 kg·cm — higher torque than MP ✓
- Encoder: 1664 CPR, 3.3–5 V (safe if encoder Vcc wired to 3.3 V ⚠️)
- Paired channel stall: 12.4 A — **exceeds MDD10A 10 A continuous** ⚠️ within 30 A peak; needs firmware current limit + no sustained stall
- 4-motor stall: 24.8 A ≫ cell 10 A continuous ⚠️ — sustained stall kills battery
- **Verdict:** $160 cheaper, adequate speed + better torque, but tighter current margins. Viable with care; MP is safer.

### Wheels — Chosen: 4" RC Airplane Wheels

**Chosen:** 4-inch (~102 mm) RC airplane wheels with 4 mm set-screw hub. Motor output shaft 4 mm D-shaft engages set screw directly — no adapter.

- **Hub:** 4 mm set-screw hub — D-shaft flat engages directly ✓
- **Diameter:** 4" ≈ 102 mm — close enough to 100 mm target; speed calc uses 102 mm
- **Material:** foam or soft rubber — compliant, good grip on varied surfaces
- **Width:** ~25–35 mm
- **Weight:** very light (~30–60 g each)
- **Cost:** $5–15 each from hobby shops
- **Availability:** RC hobby suppliers (HobbyKing, Amazon, local shops)
- ⚠️ Foam compresses under load — actual diameter and speed may shift slightly; measure under real load
- ⚠️ Less durable than polyurethane skate wheels under sustained outdoor use

---

### LDROBOT LD19 — Chosen

360° 2D lidar, triangulation, indoor use. ROS 2 driver: `ldrobot_lidar_ros2` (official).
- Interface: USB (CDC serial, no driver install needed on Linux) — plug into Pi USB port
- Power: 5 V USB bus-powered, ~180 mA / ~0.9 W typical
- Range: 0.02–12 m · scan rate: 10 Hz · angular resolution: ~1°
- ⚠️ Indoor-optimized — strong sunlight or highly reflective surfaces degrade accuracy
- Verify `ldrobot_lidar_ros2` package compatibility with target ROS 2 distro before integration

---

### [Luxonis OAK-D Lite](https://shop.luxonis.com/products/oak-d-lite-1) — Chosen

Stereo depth + RGB + onboard MyriadX VPU. ROS 2 driver: `depthai-ros`.
- Interface: USB 3.0 (USB-C), bus-powered — **requires USB 3.0 port** (Pi 4B has 2×USB 3.0 ✓; Pi 5 preferred for bandwidth headroom)
- Typical: ~400 mA / ~2 W · peak (active inference + streaming): up to **1.0 A / ~5 W**
- ⚠️ Pi 4B + OAK-D Lite peak 5V rail: Pi 4B 2.5 A + OAK-D 1.0 A + LD19 0.18 A = **3.68 A — within 5 A UPS limit ✓**
- ⚠️ `depthai-ros` stability on ROS 2 Humble/Jazzy — verify before integration

---

### [Cytron MDD10A](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels) ×2 — Chosen

2-channel brushed DC driver. 7–30 V, **10 A continuous / 30 A peak per channel**. One MDD10A per motor pair (L and R); two units for 4-motor SKID_STEER.
- Interface: PWM + DIR per channel — matches `USE_GENERIC_1_IN_MOTOR_DRIVER` in linorobot2_hardware ✓
- MP motor stall 1.8 A/channel — well within 10 A continuous ✓
- Logic input: 3.3 V compatible ✓ — direct ESP32 GPIO connection, no level shifting needed
- Power: motor rail (9–12.6 V) direct from battery

### ESP32-S3 DevKitC-1 — Chosen

Runs [linorobot2_hardware](https://github.com/hippo5329/linorobot2_hardware) micro-ROS firmware. Subscribes to `/cmd_vel`, runs PID, drives MDD10A via GPIO, reads encoders directly, publishes `/odom/unfiltered` to Pi.
- **Firmware base:** `LINO_BASE SKID_STEER` (4WD paired L/R)
- **Motor driver firmware:** `USE_GENERIC_1_IN_MOTOR_DRIVER`
- **PlatformIO board:** `esp32-s3-devkitc-1`
- **Interface to Pi:** USB serial at 921600 baud; Pi runs `micro_ros_agent serial --dev /dev/ttyUSB0`
- **Encoders:** wire directly to ESP32 GPIO (8 pins: A+B × 4 motors)
- **Motor GPIO per channel:** PWM pin + DIR pin × 4 motors = 8 GPIO pins
- **Power:** 5 V via UART USB port from Pi
- ⚠️ DevKitC-1 has two USB-C ports — use **UART port** (not USB-OTG) for Pi serial and flashing
- **Mounting:** no screw holes — 3D-printed snap-fit bracket (board 69.4 × 26.0 mm). Search Printables/Thingiverse "ESP32-S3 DevKitC mount" before modeling.

### Key firmware config (lino_base_config.h)

```c
#define LINO_BASE SKID_STEER
#define USE_GENERIC_1_IN_MOTOR_DRIVER
#define MOTOR_MAX_RPM       230
#define MOTOR_OPERATING_VOLTAGE  12
#define MOTOR_POWER_MAX_VOLTAGE  12
#define COUNTS_PER_REV1     1632   // MP 34:1 output shaft (48 CPR × 34)
#define COUNTS_PER_REV2     1632
#define COUNTS_PER_REV3     1632
#define COUNTS_PER_REV4     1632
#define WHEEL_DIAMETER      0.102  // 4" RC wheels
// LR_WHEELS_DISTANCE: measure from physical chassis
```

---

## 3. Motor Sizing

**Use case: indoor + outdoor.** Outdoor terrain, surface transitions, and small obstacles raise torque demand significantly vs. flat indoor floor.

Chosen MP #4864, 4" RC airplane wheels (~102 mm), ~2 kg robot:
- **Speed:** 1.0 m/s needs ~188 RPM; 230 RPM no-load → ~1.23 m/s, ~23% margin ✓
- **Torque:** 4.7 kg·cm stall, Pololu limit 4 kg·cm continuous / 8 kg·cm intermittent — adequacy + HP fallback in Section 2. Avoid sustained high-torque operation.
- **Odometry:** 1632 CPR output (48 × 34) — good resolution.

---

## 4. Electrical Design

### Power Budget — 5 V Rail (25 W max)
- Raspberry Pi 4B: 600 mA idle / 1.2 A typical / 2.5 A heavy — source: RPi foundation
- OAK-D Lite: ~400 mA typical / 1.0 A peak · LD19 lidar: ~180 mA · ESP32-S3: ~100 mA typical
- **Typical total: ~2.3 A / ~11.5 W — 54% headroom ✓** · peak (OAK-D + heavy CPU): ~4.2 A ✓

### Power Budget — Battery Rail (9.0–12.6 V)
Per-motor current: 100 mA no-load, ~200 mA typical driving, 1.8 A stall.
5 V rail draws ~1.1 A from battery at 85% efficiency (10 W / (11 V × 0.85)).

| Condition | Motor current | Battery current | Power @ 11 V | Notes |
|---|---|---|---|---|
| Idle | 4 × 100 mA = 0.4 A | ~1.5 A | ~16.5 W | includes 5V rail draw |
| Typical driving | 4 × 200 mA = 0.8 A | ~1.9 A | ~21 W | ~200 mA/motor flat surface estimate |
| Heavy load | 4 × 600 mA = 2.4 A | ~3.5 A | ~38.5 W | slopes, acceleration bursts |
| Stall (worst case) | 4 × 1.8 A = 7.2 A | ~8.3 A | ~91 W | within cell 10 A rating ✓ |

**Runtime (38 Wh pack):** typical driving ~108 min · heavy load ~59 min · idle ~138 min

**Key limits:**
- 5 V rail: ~2.2 A of 5 A used ✓
- MDD10A: 1.8 A stall/channel — well within 10 A continuous ✓ (ample margin)
- Stall: 7.2 A total — within cell 10 A continuous rating ✓
- Fuse: 10 A for bench test; motor stall budget peaks at ~8.3 A total ✓

### Switching and Protection

**UPS switch** cuts both 5 V and XH2.54 12 V simultaneously — single switch controls everything.
- Wire gauge: minimum 16 AWG for motor rail; 14 AWG preferred.
- Verify XH2.54 connector and UPS trace can handle motor operating current.

### Electrical Risks
- 5 V rail overload (Pi + USB near UPS 5 A limit, especially Pi 5 config)
- 4-motor stall (MP): 7.2 A total — within cell 10 A continuous ✓
- I2C / GPIO at wrong voltage level
- Noise coupling between motor power and logic wiring

### Compatibility Notes
- Motor power from XH2.54 12 V output — not 5 V rail.
- All control electronics share common ground.
- GPIO and I2C signals must be 3.3 V safe.
- Encoder power 5 V; verify signal voltage before connecting to Pi GPIO.
- Avoid sustained motor stall.

---

## 5. Integration Plan


1. Confirm mechanical layout and component placement.
2. Validate battery and UPS wiring without motors.
3. Bring up Pi on regulated 5 V.
4. Add RoboClaw, test motor power separately via USB.
5. Bring up ESP32 with linorobot2_hardware firmware; confirm USB serial link to Pi (`micro_ros_agent`).
6. Wire ESP32 GPIO → motor driver (per RoboClaw interface decision); confirm motors respond to `/cmd_vel`.
7. Validate odometry — encoder reads through ESP32 to ROS.
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
| GPIO signal check | Confirm 3.3 V-safe signals | TBD |
| ROS bringup test | Confirm software stack starts cleanly | TBD |
| Drive test | Confirm motor direction and control | TBD |
| Odometry test | Confirm encoder direction and scale | TBD |

---

## 7. Alternate Configuration: Pi 5

OAK-D Lite chosen for both configs. Swap-in upgrade is compute only. Motors, motor controller, chassis, battery, camera unchanged.

| Component | Base | Alternate |
|---|---|---|
| Compute | Raspberry Pi 4B | Raspberry Pi 5 |
| Vision | Luxonis OAK-D Lite | Luxonis OAK-D Lite (same) |

### [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)
Replaces Pi 4B. Higher performance, native USB 3.0. GPIO/I2C pinout compatible with Pi 4B.
- Idle: ~1.0 A / ~5 W · typical: ~1.8 A / ~9 W · heavy: ~3.5 A / ~17.5 W (RPi foundation)
- Requires 5V/5A supply — same Waveshare UPS, near limit under heavy load

### 5 V Rail — Pi 5 + OAK-D Lite

| Load | Typical | Peak | Notes |
|---|---|---|---|
| Raspberry Pi 5 | 1.8 A / 9 W | 3.5 A / 17.5 W | RPi foundation |
| OAK-D Lite | 0.4 A / 2 W | 1.0 A / 5 W | peak under active inference |
| LD19 lidar | 0.18 A / 0.9 W | 0.18 A / 0.9 W | |
| **Total** | **2.38 A / 11.9 W ✓** | **4.68 A / 23.4 W ✓** | within 5 A UPS limit |

Peak (heavy CPU + active OAK-D + lidar) = 4.68 A — within UPS 5 A limit ✓ (~320 mA margin). Avoid simultaneous sustained heavy CPU and active inference to preserve margin.

**Runtime:** 5 V rail ~16 W typical from battery · typical total ~40 W → **~57 min** · heavy ~53 W → **~43 min**

### Open Questions — Pi 5 config
- Linorobot2 compatibility with Pi 5 / Ubuntu confirmed?
- UPS 5 V output stable at 4.5–5 A sustained — measure under real load.

---

## 8. Open Questions

- Final chassis design?
- Does OAK-D Lite depth replace lidar or supplement it?
- `depthai-ros` stable on ROS 2 Humble/Jazzy — verify before integration.
- Motor variant (MP 67L chosen): verify torque adequate under real outdoor load — switch to HP #4844 if insufficient.
- Outdoor use: what surface types, max slope angle, curb height? Affects torque margin calculation.
- Verify `ldrobot_lidar_ros2` ROS 2 distro compatibility.
- LD19 mounting position — confirm clearance from chassis frame.
- What payload margin beyond ~2 kg robot mass?
- Initial ROS / Linorobot2 bringup path?
- Raspberry Pi RAM size?
- Charger/power adapter model?
- Wire gauge and connector for motor rail — verify against real load current.
- Measure actual motor current under real load at 1.0 m/s.
- Encoder output signal voltage — 3.3 V safe or level shifting needed? Verify MP #4864 encoder signal level before connecting to Pi GPIO.
- `LR_WHEELS_DISTANCE` — measure from physical chassis once built.
- IMU: add one? linorobot2_hardware supports MPU6050 and others for EKF fusion.
- Battery monitoring integrated into ROS?
- Are three installed cells new and matched?

---

## 9. Superseded Components

### [RoboClaw 2x7A Motor Controller (V6B)](https://www.pololu.com/product/3682)
Superseded by 2× Cytron MDD10A. RoboClaw incompatible with linorobot2_hardware GPIO motor driver interface (no PWM+DIRA+DIRB input mode). MDD10A is directly supported and simpler.

### [DFRobot FIT0186 Gearmotor](https://www.dfrobot.com/product-634.html)
Superseded by Pololu 34:1. DigiKey 1738-1106-ND. 12 V, 251 RPM, 7 A stall (14 A/channel paired — just under RoboClaw 2x7A 15 A peak, far above 7.5 A continuous ⚠️), 700 CPR output encoder.

### Motor Comparison: FIT0186 vs Pololu HP 34:1 vs Pololu MP 34:1

| Parameter | FIT0186 | Pololu HP #4844 | Pololu MP #4864 (chosen) |
|---|---|---|---|
| No-load speed | 251 RPM | 300 RPM | 230 RPM |
| No-load current | 350 mA | 300 mA | 100 mA |
| Stall current | 7.0 A | 5.0 A | 1.8 A |
| Stall torque | 18 kg·cm | 11 kg·cm | 4.7 kg·cm |
| Encoder CPR (output) | 700 | 1632 | 1632 |
| 4-motor stall current | 28 A | 20 A | 7.2 A |
| Paired channel stall | 14 A ⚠️ | 10 A | 3.6 A — within MDD10A 10 A cont ✓ |
| No-load speed (102 mm wheel) | ~1.34 m/s | ~1.60 m/s | ~1.23 m/s |
| Price (each) | — | ~$50 | ~$50 |

---

## 11. Shop Motor Inventory

Physical motors on hand in the shop. Catalog of what's available, independent of the chosen design. Use to evaluate substitutes against the limits in [Section 2 / Section 10](#10-superseded-components).

| # | Motor | Qty | Voltage | No-load RPM | Stall current | Encoder | Condition | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | [TSINY TS-25GA370H-45](https://makerselectronics.com/product/dc-motor-ga25-370-with-encoder-4-4kg-130rpm-12v-with-bracket/) (25 mm dia, ~46.8:1) | ≥2 (confirm) | DC 12 V | 130 | 1.8 A stall | 6-wire, hall (CPR TBD — measure) | Installed on chassis | Stall torque 4.4 kg·cm, nearly identical to MP #4864. Encoder CPR not on datasheet — count pulses to determine. Encoder voltage TBD — verify 3.3 V safe before connecting to ESP32 ⚠️ |
| 2 | SGM25-370 (25 mm dia) | TBD | DC 6 V | 280 | TBD | Magnetic, rear PCB + JST (CPR TBD) | Loose, on test wheel | ⚠️ 6 V motor — under-driven on 12 V rail or needs voltage limit. Encoder voltage TBD — verify 3.3 V safe |
| 3 | [JGA25-370 1:32](https://www.amazon.com/dp/B0GV7J5DBY) (model MC370P34_V12_R13, 25 mm dia, 4 mm D-shaft offset, 96 g) | 2-pack ×2 = 4 | DC 12 V | 260 ±10% (rated) | **6.2 A** (rated 1.1 A) | AB hall, 13 lines → 1664 CPR, **3.3–5 V** ✓, integrated pull-ups, 6-pin PH2.0 | Candidate — not bought | 1.38 m/s @ 4" wheel ✓. Rated torque 1.5 kg·cm, stall torque 8.7 kg·cm. Paired stall 12.4 A — exceeds MDD10A 10 A continuous ⚠️ within 30 A peak → firmware current cap + avoid sustained stall. 4-motor 24.8 A ≫ cell 10 A continuous ⚠️. Encoder Vcc must wire to ESP32 3.3 V — pull-ups follow Vcc, 5 V power → 5 V signals → fries ESP32 GPIO ⚠️ |

> Fill in each row as motors are identified. Flag any with paired-channel stall > 15 A (exceeds RoboClaw 2x7A peak), or sustained draw > 7.5 A continuous, with ⚠️.
