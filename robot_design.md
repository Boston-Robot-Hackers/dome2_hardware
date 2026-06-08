# Robot Design Document

Living design document: mechanical, electrical, software, controls, integration, and open decisions.

---

## 0. Performance Requirements

Requirements used to validate component selection. Update before ordering.

| Requirement | Value | Status | Notes |
|---|---|---|---|
| Max speed | 1.0 m/s | Known | Minimum; 1.39 m/s achievable with JGA25-370 + 4" wheel |
| Robot total mass | ~3 kg | **Verify** | Chassis + electronics + battery — weigh or CAD before ordering |
| Max slope | 5° (1:12 ADA ramp) | Known | Minimal — covers accessible wheelchair ramps; torque check below |
| Obstacle clearance | TBD | TBD | Ground clearance and wheel diameter must exceed |
| Min runtime | 2 hr | Known | ⚠️ 38 Wh pack → ~108 min typical driving — marginally short; see runtime note below |
| Use case | Indoor + outdoor | Known | Outdoor raises torque demand vs. flat floor |
| Drive type | Differential drive, 4-wheel SKID_STEER | Known | L/R pairs driven together |
| Payload | None | Known | Base BOM sensors only |
| Turn radius | TBD | **Verify** | ⚠️ Skid steer turn requires wheel scrubbing — zero-radius spin depends on surface friction and motor torque; high-friction surfaces (carpet, asphalt) may resist; test before assuming |
| Operating voltage | 3S Li-ion, 9.0–12.6 V motor rail | Known | |

### Derived Motor Requirements

| Parameter | Required | JGA25-370 1:32 | Pololu MP 34:1 | Pass? |
|---|---|---|---|---|
| No-load RPM @ 12 V | ≥ 188 RPM for 1.0 m/s | 260 RPM ✓ | 230 RPM ✓ | Both ✓ |
| Torque @ 5° slope, 3 kg | ≥ 0.34 kg·cm/wheel continuous | 1.5 kg·cm rated ✓ (4× margin) | 4 kg·cm rated ✓ (11× margin) | Both ✓ |
| Encoder CPR | ≥ ~500 CPR output shaft | 1664 CPR ✓ | 1632 CPR ✓ | Both ✓ |
| Encoder voltage | 3.3 V safe for ESP32 GPIO | 3.3–5 V ✓ (wire to 3.3 V) | Unverified ⚠️ | Verify Pololu |
| Single-motor stall vs. driver | ≤ 10 A continuous (MDD10A, 1 motor/channel) | 6.2 A ✓ | 1.8 A ✓ | Both ✓ — no driver risk |
| 4-motor stall vs. cell | ≤ 10 A continuous (cell) | 24.8 A ⚠️ | 7.2 A ✓ | JGA needs firmware cap to protect battery |

⚠️ Fill in TBD rows before finalizing motor order — slope angle especially drives torque requirement.

**Runtime note:** 38 Wh pack at ~21 W typical driving = ~108 min — falls ~12 min short of 2 hr target. Options: (1) accept margin (close enough at lighter loads), (2) add 4th cell to make 4S pack (changes voltage — motor and UPS must support), (3) reduce idle/5V draw, (4) lower speed target. Recheck after measuring actual chassis mass at 3 kg.

---

## 1. Bill of Materials

| Item | Link | Role | Status |
|---|---|---|---|
| Raspberry Pi 4 Model B | [Manufacturer](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) | Main ROS computer | Known |
| Waveshare UPS Module 3S | [Manufacturer](https://www.waveshare.com/product/ups-module-3s.htm) | Battery UPS, 5 V supply | Known |
| Panasonic NCR18650GA cells ×3 (+1 spare) | [Manufacturer](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA) | 3S battery pack | Known |
| JGA25-370 1:32 gearmotor w/encoder ×4 | [Amazon B0GV7J5DBY](https://www.amazon.com/dp/B0GV7J5DBY) | Drive motors | **Chosen** — ~$30–40 total; firmware current limit required |
| Pololu 25D MP 34:1 gearmotor w/encoder ×4 | [Pololu #4864](https://www.pololu.com/product/4864) | Drive motors | Backup — higher cost (~$200), lower stall current |
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
- ⚠️ Verify XH2.54 connector and internal trace current rating for motor use (XH connectors typically rated ~3 A; JGA25-370 typical 4-motor operating current ~0.8 A, but stall far exceeds this — firmware must prevent stall)

### [Panasonic NCR18650GA Cells](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA)
4 cells (3 installed, 1 spare). Per cell: 18650, Li-ion, 3450 mAh, 3.6–3.7 V nominal, 4.2 V full, 10 A continuous.
3S pack: 11.1 V nominal / 12.6 V full, 3450 mAh, ~38 Wh. Runtime: ~3.8 h @ 10 W · ~1.9 h @ 20 W · ~1.3 h @ 30 W.

### JGA25-370 1:32 — Chosen

[Amazon B0GV7J5DBY](https://www.amazon.com/dp/B0GV7J5DBY) — ~$15–20 per 2-pack × 2 = **~$30–40 total**. Model MC370P34_V12_R13. 25 mm body, 4 mm D-shaft, 96 g.
- 12 VDC, 260 RPM ±10% no-load → 1.39 m/s @ 4" wheel ✓
- Stall: 6.2 A, 8.7 kg·cm — better torque than Pololu MP ✓
- Rated torque: 1.5 kg·cm continuous
- Encoder: AB Hall, 13 lines → **1664 CPR** output shaft, **3.3–5 V** ✓, integrated pull-ups, 6-pin PH2.0
- ⚠️ Encoder Vcc **must** wire to ESP32 3.3 V — pull-ups follow Vcc; 5 V power → 5 V signals → fries ESP32 GPIO
- Paired channel stall: 12.4 A — **exceeds MDD10A 10 A continuous** ⚠️ within 30 A peak → **firmware current limit required, no sustained stall**
- 4-motor stall: 24.8 A ≫ cell 10 A continuous ⚠️ — sustained stall kills battery

### [Pololu 34:1 Metal Gearmotor 25Dx67L mm MP 12V](https://www.pololu.com/product/4864) — Backup

Pololu #4864. ~$50/ea × 4 = ~$200 total. Safer current margins but high cost.
12 VDC, 230 RPM no-load, 4 mm D-shaft, 25 mm body.
- No-load: 100 mA, ~1.2 W
- Stall: 1.8 A, 4.7 kg·cm (0.46 N·m)
- Encoder: quadrature Hall effect, 48 CPR motor shaft, 1632 CPR output shaft
- Paired channel stall: ~3.6 A — well within MDD10A 10 A continuous ✓
- 4-motor stall: 7.2 A — within cell 10 A continuous ✓
- Pololu continuous torque limit: 4 kg·cm — stall torque (4.7 kg·cm) above limit ⚠️ avoid sustained high-load
- Verify encoder signal voltage before connecting to ESP32 GPIO.

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

2-channel brushed DC driver. 7–30 V, **10 A continuous / 30 A peak per channel**. **1 motor per channel; 2 boards for 4 motors.**
- Wiring: board A ch1=FL, ch2=RL; board B ch1=FR, ch2=RR. Encoders go to ESP32 GPIO, not MDD10A.
- Interface: PWM + DIR per channel — matches `USE_GENERIC_1_IN_MOTOR_DRIVER` in linorobot2_hardware ✓
- JGA25-370 single-motor stall 6.2 A — within 10 A continuous ✓ (no driver risk from stall)
- ⚠️ 4-motor stall 24.8 A still kills battery cells — firmware current limit required to protect battery, not driver
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

---

## 3. Motor Sizing

**Use case: indoor + outdoor.** Outdoor terrain, surface transitions, and small obstacles raise torque demand significantly vs. flat indoor floor.

Chosen JGA25-370 1:32, 4" RC airplane wheels (~102 mm), ~2 kg robot:
- **Speed:** 1.0 m/s needs ~188 RPM; 260 RPM no-load → ~1.39 m/s, ~39% margin ✓
- **Torque:** 8.7 kg·cm stall, 1.5 kg·cm rated continuous — adequate for outdoor use ✓. ⚠️ Avoid sustained stall; firmware current limit required.
- **Odometry:** 1664 CPR output — good resolution.

---

## 4. Electrical Design

### Power Budget — 5 V Rail (25 W max)
- Raspberry Pi 4B: 600 mA idle / 1.2 A typical / 2.5 A heavy — source: RPi foundation
- OAK-D Lite: ~400 mA typical / 1.0 A peak · LD19 lidar: ~180 mA · ESP32-S3: ~100 mA typical
- **Typical total: ~2.3 A / ~11.5 W — 54% headroom ✓** · peak (OAK-D + heavy CPU): ~4.2 A ✓

### Power Budget — Battery Rail (9.0–12.6 V)
Per-motor current: ~200 mA typical driving, 6.2 A stall (JGA25-370).
5 V rail draws ~1.1 A from battery at 85% efficiency (10 W / (11 V × 0.85)).

| Condition | Motor current | Battery current | Power @ 11 V | Notes |
|---|---|---|---|---|
| Idle | 4 × 100 mA = 0.4 A | ~1.5 A | ~16.5 W | includes 5V rail draw |
| Typical driving | 4 × 200 mA = 0.8 A | ~1.9 A | ~21 W | ~200 mA/motor flat surface estimate |
| Heavy load | 4 × 600 mA = 2.4 A | ~3.5 A | ~38.5 W | slopes, acceleration bursts |
| Stall (worst case) | 4 × 6.2 A = 24.8 A | ~25.9 A | ~285 W | ⚠️ far exceeds cell 10 A continuous — sustained stall kills battery; firmware must prevent |

**Runtime (38 Wh pack):** typical driving ~108 min · heavy load ~59 min · idle ~138 min

**Key limits:**
- 5 V rail: ~2.2 A of 5 A used ✓
- MDD10A: 1 motor/channel → 6.2 A single-motor stall — within 10 A continuous ✓
- Stall: 24.8 A total — far exceeds cell 10 A continuous ⚠️ — sustained stall must be prevented
- Fuse: size for operating current (~4–5 A typical); stall cutoff via firmware, not fuse

### Switching and Protection

**UPS switch** cuts both 5 V and XH2.54 12 V simultaneously — single switch controls everything.
- Wire gauge: minimum 16 AWG for motor rail; 14 AWG preferred.
- Verify XH2.54 connector and UPS trace can handle motor operating current.

### Electrical Risks
- 5 V rail overload (Pi + USB near UPS 5 A limit, especially Pi 5 config)
- 4-motor stall (JGA25-370): 24.8 A total — far exceeds cell 10 A continuous ⚠️ firmware must prevent
- I2C / GPIO at wrong voltage level
- Noise coupling between motor power and logic wiring

### Compatibility Notes
- Motor power from XH2.54 12 V output — not 5 V rail.
- All control electronics share common ground.
- GPIO and I2C signals must be 3.3 V safe.
- JGA25-370 encoder: 3.3–5 V ✓ — wire encoder Vcc to 3.3 V to keep signals 3.3 V safe.
- Avoid sustained motor stall.

---

## 5. Alternate Configuration: Pi 5

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


## 6. Shop Motor Inventory

Physical motors on hand in the shop. Catalog of what's available, independent of the chosen design. Use to evaluate substitutes against the limits in Section 2.

| # | Motor | Qty | Voltage | No-load RPM | Stall current | Encoder | Condition | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | [TSINY TS-25GA370H-45](https://makerselectronics.com/product/dc-motor-ga25-370-with-encoder-4-4kg-130rpm-12v-with-bracket/) (25 mm dia, ~46.8:1) | ≥2 (confirm) | DC 12 V | 130 | 1.8 A stall | 6-wire, hall (CPR TBD — measure) | Installed on chassis | Stall torque 4.4 kg·cm, nearly identical to MP #4864. Encoder CPR not on datasheet — count pulses to determine. Encoder voltage TBD — verify 3.3 V safe before connecting to ESP32 ⚠️ |
| 2 | SGM25-370 (25 mm dia) | TBD | DC 6 V | 280 | TBD | Magnetic, rear PCB + JST (CPR TBD) | Loose, on test wheel | ⚠️ 6 V motor — under-driven on 12 V rail or needs voltage limit. Encoder voltage TBD — verify 3.3 V safe |
| 3 | [JGA25-370 1:32](https://www.amazon.com/dp/B0GV7J5DBY) (model MC370P34_V12_R13, 25 mm dia, 4 mm D-shaft offset, 96 g) | 2-pack ×2 = 4 | DC 12 V | 260 ±10% (rated) | **6.2 A** (rated 1.1 A) | AB hall, 13 lines → 1664 CPR, **3.3–5 V** ✓, integrated pull-ups, 6-pin PH2.0 | **Chosen — buy** | 1.38 m/s @ 4" wheel ✓. Rated torque 1.5 kg·cm, stall torque 8.7 kg·cm. Paired stall 12.4 A — exceeds MDD10A 10 A continuous ⚠️ within 30 A peak → firmware current cap + avoid sustained stall. 4-motor 24.8 A ≫ cell 10 A continuous ⚠️. Encoder Vcc must wire to ESP32 3.3 V — pull-ups follow Vcc, 5 V power → 5 V signals → fries ESP32 GPIO ⚠️ |

> Fill in each row as motors are identified. Flag any with paired-channel stall > 10 A (exceeds MDD10A continuous) or 4-motor stall > 10 A (exceeds cell continuous) with ⚠️.
