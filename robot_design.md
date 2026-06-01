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
| Pololu 25D MP 34:1 gearmotor w/encoder ×4 | [Pololu #4864](https://www.pololu.com/product/4864) | Drive motors | **Chosen** — ~$50/ea × 4 = ~$200 ⚠️ |
| RoboClaw 2x7A (V6B) | [Pololu](https://www.pololu.com/product/3682) | Motor controller | **Chosen** |
| Cytron MDD10A | [Cytron](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels) | Backup motor controller | Backup |
| Wheels ×4 | TBD | Drive wheels | Required — ~100 mm diameter, 4 mm D-shaft, decision pending |
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
- Verify XH2.54 connector and internal trace current rating for motor use (XH connectors typically rated ~3 A; 4-motor stall ~7 A chosen MP, up to ~25 A high-stall candidates)

### [Panasonic NCR18650GA Cells](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA)
4 cells (3 installed, 1 spare). Per cell: 18650, Li-ion, 3450 mAh, 3.6–3.7 V nominal, 4.2 V full, 10 A continuous.
3S pack: 11.1 V nominal / 12.6 V full, 3450 mAh, ~38 Wh. Runtime: ~3.8 h @ 10 W · ~1.9 h @ 20 W · ~1.3 h @ 30 W.

### [Pololu 34:1 Metal Gearmotor 25Dx67L mm MP 12V](https://www.pololu.com/product/4864) — Chosen
Pololu #4864. ~$50/ea × 4 = ~$200 total. ⚠️ High unit cost — verify before ordering all 4.
12 VDC, 230 RPM no-load, 4 mm D-shaft, 25 mm body.
- No-load: 100 mA, ~1.2 W
- Stall: 1.8 A, ~21.6 W, 4.7 kg·cm (0.46 N·m)
- Encoder: quadrature Hall effect, 48 CPR motor shaft, 1632 CPR output shaft
- 4 motors no-load: 0.4 A, ~4.8 W · paired channel stall: ~3.6 A — **well within RoboClaw 2x7A 7.5 A continuous ✓**
- 4-motor stall: 7.2 A — within cell 10 A continuous rating ✓
- Pololu recommended continuous torque limit: 4 kg·cm; intermittent: 8 kg·cm — stall torque (4.7 kg·cm) is above continuous limit ⚠️ avoid sustained high-load operation
- Verify encoder output signal voltage before connecting to Pi GPIO.

### MP 67L (chosen) vs HP 67L alternative

HP 67L = [Pololu #4844](https://www.pololu.com/product/4844). Full 3-way comparison (incl. superseded FIT0186) in **Section 10**.

- **MP chosen:** paired stall 3.6 A / 4-motor stall 7.2 A — within RoboClaw continuous + cell ratings (HP: 10 A / 20 A — at controller peak, over cell rating). 3× lower idle current → longer runtime. 230 RPM → ~1.20 m/s, clears 1.0 m/s target.
- **MP risk:** stall torque 4.7 kg·cm vs HP 11 kg·cm — adequate for flat indoor + light outdoor, marginal on steep slopes/curbs. Switch to HP #4844 if real-load torque insufficient.

### Wheels — Decision Pending

Motor output shaft: 4 mm D-shaft. Target diameter: ~100 mm (verified against 1.0 m/s speed budget).
Two candidate types:

#### Inline Skate / Scooter Wheels
Examples: inline skate wheels (72–110 mm), kick scooter wheels (100–125 mm).
- **Hub:** designed for 608 bearings (8 mm ID bore) — **not direct fit on 4 mm D-shaft**; needs hub adapter or press-fit insert
- **Diameter:** 100 mm readily available (inline skate = 90–110 mm, scooter = 100–125 mm)
- **Material:** polyurethane, hardness 78A–88A (harder = faster/less grip, softer = more grip/wear)
- **Tread:** smooth or lightly textured — good on hard floors, less grip on loose outdoor surfaces
- **Width:** ~24 mm (inline) — adequate for stability
- **Weight:** moderate (~100–150 g each)
- **Cost:** cheap — $10–30 for a set of 4
- **Availability:** hardware/sports stores, Amazon
- ⚠️ Hub adapter required for 4 mm D-shaft — adds complexity, potential slop

#### RC Airplane Wheels
Examples: Du-Bro, Dubro, generic foam/rubber RC wheels (2.5"–4" = 63–100 mm).
- **Hub:** typically 3–6 mm shaft hole with set screw — **4 mm versions exist**, D-shaft flat engages set screw directly ✓
- **Diameter:** 100 mm = ~4" — available but less common; 3" (76 mm) and 4" (102 mm) standard sizes
- **Material:** foam or soft rubber — more compliant, better grip on varied surfaces
- **Tread:** foam (smooth, light grip) or rubber tread (better outdoor grip)
- **Width:** ~25–35 mm — similar to skate
- **Weight:** very light (~30–60 g each)
- **Cost:** $5–15 each from hobby shops
- **Availability:** RC hobby suppliers (HobbyKing, Amazon, local shops)
- ✓ D-shaft compatible without adapter if set screw hub
- ⚠️ Foam wheels compress under load — diameter and speed estimates may shift slightly
- ⚠️ Less durable than PU skate wheels under sustained outdoor use

#### Wheel Comparison

| Parameter | Skate/Scooter | RC Airplane |
|---|---|---|
| Diameter (100 mm) | ✓ easy to find | ✓ 4" ≈ 102 mm close enough |
| D-shaft fit | ⚠️ adapter needed | ✓ set screw hub (4 mm) |
| Grip — hard floor | ✓ good | ✓ good |
| Grip — outdoor/rough | ⚠️ marginal | ✓ better (compliant) |
| Weight | moderate | light |
| Durability | ✓ high | ⚠️ foam wears faster |
| Cost (×4) | ✓ $10–30 total | ~$20–60 total |
| Availability | ✓ local stores | hobby shops / online |

**Recommendation:** RC airplane wheels if 4 mm D-shaft set-screw hub confirmed — simplest fit, lighter, good grip. Skate wheels viable if hub adapter sourced and lower cost matters more than grip.

---

### [RoboClaw 2x7A Motor Controller (V6B)](https://www.pololu.com/product/3682) — Chosen
2-channel brushed DC, 6–34 V, **7.5 A continuous / 15 A peak per channel**. (Replaces discontinued 2x5A, Pololu #2394.)
Interfaces: USB, TTL serial, RC, analog. Dual quadrature encoder inputs. Built-in speed/position PID.
Paired MP 34:1 motors stall at ~3.6 A/channel — well within 7.5 A continuous limit ✓. Higher 15 A peak also brings the high-stall JGA25-370 (paired 12.4 A) inside peak — see Section 11.

---

## 3. Motor Sizing

**Use case: indoor + outdoor.** Outdoor terrain, surface transitions, and small obstacles raise torque demand significantly vs. flat indoor floor.

Chosen MP #4864, 100 mm wheels, ~2 kg robot:
- **Speed:** 1.0 m/s needs ~191 RPM; 230 RPM no-load → ~1.20 m/s, ~20% margin ✓
- **Torque:** 4.7 kg·cm stall, Pololu limit 4 kg·cm continuous / 8 kg·cm intermittent — adequacy + HP fallback in Section 2. Avoid sustained high-torque operation.
- **Odometry:** 1632 CPR output (48 × 34) — good resolution.

---

## 4. Electrical Design

### Power Budget — 5 V Rail (25 W max)
- Raspberry Pi 4B: 600 mA idle / 1.2 A typical / 2.5 A heavy — source: RPi foundation
- USB camera: ~300 mA · USB lidar: ~500 mA · I2C: negligible
- **Typical total: ~2.0 A / ~10 W — 60% headroom ✓**

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
- 5 V rail: ~2 A of 5 A used ✓
- RoboClaw 2x7A: paired stall 3.6 A/channel — within 7.5 A continuous ✓ (ample margin)
- Stall: 7.2 A total — within cell 10 A continuous rating ✓
- Fuse: 10 A for bench test; motor stall budget peaks at ~8.3 A total ✓

### Switching and Protection

**UPS switch** cuts both 5 V and XH2.54 12 V simultaneously — single switch controls everything.
- Wire gauge: minimum 16 AWG for motor rail; 14 AWG preferred.
- Verify XH2.54 connector and UPS trace can handle motor operating current.

### Electrical Risks
- 5 V rail overload (Pi + USB near UPS 5 A limit, especially Pi 5 config)
- 4-motor stall exceeds cell 10 A continuous rating for high-stall motors (JGA25-370 ~25 A); chosen MP 7.2 A within ✓
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
- Wheel decision: RC airplane (4 mm set-screw hub) vs skate/scooter (needs hub adapter)? Confirm 4 mm D-shaft hub adapter exists for skate route before deciding.
- Motor variant (MP 67L chosen): verify torque adequate under real outdoor load — switch to HP #4844 if insufficient.
- Outdoor use: what surface types, max slope angle, curb height? Affects torque margin calculation.
- Configure RoboClaw 2x7A acceleration and current limits to stay within 7.5 A continuous per channel.
- Which sensors required for first working version?
- What payload margin beyond ~2 kg robot mass?
- Initial ROS / Linorobot2 bringup path?
- Raspberry Pi RAM size?
- Exact I2C shim model?
- Charger/power adapter model?
- Wire gauge and connector for motor rail — verify against real load current.
- Measure actual motor current under real load at 1.0 m/s.
- Encoder output signal voltage — 3.3 V safe or level shifting needed?
- RoboClaw 2x7A to Pi: USB or TTL serial?
- Battery monitoring integrated into ROS?
- Are three installed cells new and matched?
- Cytron MDD10A backup: which component handles encoder PID if used?
- Yahboom 4-ch driver: per-channel continuous + peak current rating? (email support@yahboom.com) Must clear JGA25-370 6.2 A stall.
- Yahboom 4-ch driver: ROS 2 / Linorobot2 driver exists, or custom I2C/serial node needed?

---

## 9. Backup Components

### [Cytron MDD10A](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels)
Use if RoboClaw 2x7A unavailable. **No onboard encoder PID** — PID must run on Pi or separate microcontroller.
7–30 V, 10 A continuous / 30 A peak per channel. Interfaces: PWM, RC, analog only.

### [Yahboom 4-Channel Encoder Motor Drive Module](https://category.yahboom.net/products/quad-md-module) — Candidate
4 **independent** channels (one per motor) → eliminates RoboClaw's paired-motor stall doubling. Onboard STM32F103RCT6 co-processor handles motor drive + encoder PID (like RoboClaw, offloads Pi). I2C + UART serial control; Pi/Jetson/STM32 targets. 4 encoder inputs. VIN "12.6 V recommended for 520 motors" — matches 3S full-charge rail ✓.
- ⚠️ **Per-channel current rating not published** (chip + amps absent from spec page & USART manual). Deciding number — email support@yahboom.com. Need ≥ ~3 A cont / 6+ A peak to survive JGA25-370 6.2 A stall on its own channel.
- ⚠️ **No known ROS 2 / Linorobot2 driver** — proprietary I2C/serial protocol; needs custom node. RoboClaw has mature driver; this is integration risk.
- 520-motor VIN hint suggests higher-current driver than TB6612-class (1.2 A) — unconfirmed.

---

## 10. Superseded Components

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
| Paired channel stall | 14 A — near 2x7A 15 A peak ⚠️ | 10 A — within 2x7A 15 A peak ✓ | 3.6 A — within continuous ✓ |
| No-load speed (100 mm wheel) | ~1.31 m/s | ~1.57 m/s | ~1.20 m/s |
| Price (each) | — | ~$50 | ~$50 |

---

## 11. Shop Motor Inventory

Physical motors on hand in the shop. Catalog of what's available, independent of the chosen design. Use to evaluate substitutes against the limits in [Section 2 / Section 10](#10-superseded-components).

| # | Motor | Qty | Voltage | No-load RPM | Stall current | Encoder | Condition | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | [TSINY TS-25GA370H-45](https://www.tsinymotor.com) (25 mm dia, 1:45) | ≥2 (confirm) | DC 12 V | 130 | TBD | 6-wire, hall (CPR TBD) | Installed on chassis | Gearbox "45" = 45:1 ratio. Encoder voltage TBD — verify 3.3 V safe ⚠️ |
| 2 | SGM25-370 (25 mm dia) | TBD | DC 6 V | 280 | TBD | Magnetic, rear PCB + JST (CPR TBD) | Loose, on test wheel | ⚠️ 6 V motor — under-driven on 12 V rail or needs voltage limit. Encoder voltage TBD — verify 3.3 V safe |
| 3 | [JGA25-370 1:32](https://www.amazon.com/dp/B0GV7J5DBY) (25 mm dia, 4 mm D-shaft) | 2-pack ×2 = 4 | DC 12 V | 260 (rated) | **6.2 A** | AB hall, 13 lines → 1664 CPR, **3.3–5 V** ✓, 6-pin PH2.0 | Candidate — not bought | 1.38 m/s @ 4" wheel ✓. Stall torque 8.7 kg·cm. Paired stall 12.4 A — within RoboClaw 2x7A 15 A peak ✓ but above 7.5 A continuous → accel limit + firmware current cap, avoid sustained stall. 4-motor 24.8 A ≫ cell 10 A continuous ⚠️. Encoder 3.3 V safe ✓ |

> Fill in each row as motors are identified. Flag any with paired-channel stall > 15 A (exceeds RoboClaw 2x7A peak), or sustained draw > 7.5 A continuous, with ⚠️.
