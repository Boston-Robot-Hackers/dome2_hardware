# Robot Electrical Components

Electrical specification: power rails, power budget, wiring constraints, signal levels, compatibility risks. Full BOM and mechanical sizing in `robot_design.md`.

---

## Active Components

### [Raspberry Pi 4 Model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
Power: 5 V USB-C from UPS module. Interfaces: USB (sensors/camera/lidar), GPIO (3.3 V safe only), I2C (check address conflicts), Ethernet/Wi-Fi.

### Raspberry Pi I2C Shim
3.3 V logic, SDA/SCL on GPIO2/GPIO3. Verify all connected devices use 3.3 V logic or have level shifting. Exact product TBD.

### [Waveshare UPS Module (3S)](https://www.waveshare.com/product/ups-module-3s.htm)
- Input: 3S Li-ion, 9.0–12.6 V
- 5 V output: 5 A / 25 W (Pi + USB peripherals)
- 3.3 V output: 300 mA
- XH2.54 battery-voltage output: unregulated, 12.6 V / 2 A max — **not a motor supply**
- Optional I2C: battery voltage, current, power status
- Charge and discharge simultaneously: yes

### [Panasonic NCR18650GA Cells](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA)
4 cells (3 installed, 1 spare). Per cell: 18650, Li-ion, 3450 mAh, 3.6–3.7 V nominal, 4.2 V full, 10 A continuous.
3S pack: 11.1 V nominal / 12.6 V full, 3450 mAh, ~38 Wh. Runtime: ~3.8 h @ 10 W · ~1.9 h @ 20 W · ~1.3 h @ 30 W.

### [Pololu 34:1 Metal Gearmotor 25Dx67L mm HP 12V](https://www.pololu.com/product/4844) — Chosen
Pololu #4844. 12 VDC, 300 RPM no-load, 4 mm D-shaft, 25 mm body.
- No-load: 300 mA, ~3.6 W
- Stall: 5.0 A, ~60 W, 11 kg·cm (1.08 N·m)
- Encoder: quadrature Hall effect, 48 CPR motor shaft, 1632 CPR output shaft
- 4 motors no-load: 1.2 A, ~14.4 W · paired channel stall: ~10 A (within RoboClaw 2x7A ~15 A peak)
- Verify encoder output signal voltage before connecting to Pi GPIO.

### [RoboClaw 2x7A Motor Controller](https://www.basicmicro.com/RoboClaw-2x7A-Motor-Controller_p_55.html) — Chosen
2-channel brushed DC, 6–34 V, 7 A continuous / ~15 A peak per channel (verify datasheet).
Interfaces: USB, TTL serial, RC, analog. Dual quadrature encoder inputs. Built-in speed/position PID.
Paired 34:1 motors stall at ~10 A/channel → ~5 A margin. Onboard PID eliminates separate microcontroller.

---

## Power Architecture

```text
3S Battery Pack
   ├──> Waveshare UPS switch ──> 5 V rail ──> Raspberry Pi 4B + USB peripherals
   └──> Fuse ──> motor switch / e-stop ──> RoboClaw 2x7A VIN ──> Drive motors
```

## Power Budget

### 5 V Rail (25 W max)
- Raspberry Pi 4B: 600 mA idle / 1.2 A typical / 2.5 A heavy — source: RPi foundation
- USB camera: ~300 mA
- USB lidar (e.g. RPLidar A1): ~500 mA
- I2C peripherals: negligible
- **Typical total: ~2.0 A / ~10 W — 60% headroom within 5 A / 25 W limit ✓**

### Battery Rail (9.0–12.6 V)
Per-motor current (Pololu 34:1 datasheet): 300 mA no-load, ~500 mA typical indoor driving, 5.0 A stall.

| Condition | Motor current | Battery current | Power @ 11 V | Notes |
|---|---|---|---|---|
| No-load / idle | 4 × 300 mA = 1.2 A | ~2.3 A | ~25 W | includes ~12 W for UPS 5V rail (85% eff.) |
| Typical driving | 4 × 500 mA = 2.0 A | ~3.1 A | ~36 W | estimated 500 mA/motor indoor flat surface |
| Heavy load | 4 × 1.5 A = 6.0 A | ~7.1 A | ~78 W | slopes, acceleration bursts |
| Stall (worst case) | 4 × 5.0 A = 20 A | ~21 A | ~231 W | exceeds cell 10 A rating ⚠️ avoid |

**Runtime estimate (38 Wh pack):**
- Typical driving (~36 W): **~60 min**
- Heavy load (~78 W): **~29 min**
- Idle only (~25 W): **~91 min**

**Key limits:**
- 5 V rail: fine — 2 A of 5 A used ✓
- Battery current typical: ~3 A of 10 A cell rating ✓
- Stall: 20 A motors alone exceeds 10 A cell continuous rating — avoid sustained stall ⚠️
- Fuse sizing: 10 A suitable for bench test; raise to 15 A for normal driving after measuring real load

## Switching and Protection

Two independent switches:
- **UPS switch** — controls Pi, 5 V rail, logic peripherals. Allows Pi debug without motor power.
- **Motor switch / e-stop** — controls RoboClaw + motors. Disables motion while Pi/ROS stay live.

```text
Battery+ → fuse (near battery) → motor switch/e-stop → RoboClaw VIN+
Battery− → RoboClaw VIN− → common ground
```

First-pass fuse: 7.5 A or 10 A automotive blade for bench testing. Final value after measuring real load.

## Compatibility Notes
- Motor power must come from separately fused/switched battery rail — not UPS 5 V output.
- All control electronics share common ground.
- GPIO and I2C signals must be 3.3 V safe.
- Encoder power 5 V; verify encoder output signal voltage before connecting to Pi GPIO.
- I2C address conflicts must be checked once all devices are selected.
- Avoid sustained motor stall conditions: paired 34:1 stall at ~10 A/channel, within RoboClaw 2x7A peak but not indefinitely safe.

## Open Questions
- Raspberry Pi RAM size?
- USB devices to be attached?
- GPIO pins reserved?
- Exact I2C shim model?
- I2C devices and addresses?
- Charger/power adapter model?
- Peripherals on 5 V rail?
- Fuse rating, switch rating, wire gauge, connector for motor rail?
- Motor switch: DC switch, e-stop button, or both?
- Measure actual motor current under real load at 1.0 m/s to validate ~500 mA/motor estimate.
- Encoder output signal voltage — 3.3 V safe or level shifting needed?
- Battery monitoring integrated into ROS?
- Are three installed cells new and matched?

---

## Alternate Configuration Components

### [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)
Replaces Pi 4B. Higher performance, native USB 3.0. Requires 5V/5A (27W) supply — same Waveshare UPS but near its limit under heavy load.
- Idle: ~1.0 A / ~5 W · typical: ~1.8 A / ~9 W · heavy: ~3.5 A / ~17.5 W
- USB 3.0 required for OAK-D Lite bandwidth
- GPIO and I2C pinout compatible with Pi 4B

### [Luxonis OAK-D Lite](https://shop.luxonis.com/products/oak-d-lite-1)
Replaces generic USB camera. Stereo depth + RGB + onboard MyriadX VPU for neural net inference.
- Interface: USB 3.0 (USB-C), bus-powered
- Typical: ~400 mA / ~2 W · active inference: ~500 mA / ~2.5 W
- ROS 2 driver: `depthai-ros`
- Note: USB 3.0 bandwidth required — Pi 4B USB 3.0 ports exist but Pi 5 preferred for headroom

---

## Backup

### [Cytron MDD10A](https://www.cytron.io/p-10amp-5v-30v-dc-motor-driver-2-channels)
Use if RoboClaw 2x7A unavailable. **No onboard encoder PID** — PID must run on Pi or separate microcontroller.
7–30 V, 10 A continuous / 30 A peak per channel. Interfaces: PWM, RC, analog only.

---

## Superseded

### [DFRobot FIT0186 Gearmotor](https://www.dfrobot.com/product-634.html)
Superseded by Pololu 34:1. DigiKey 1738-1106-ND. 12 V, 251 RPM, 7 A stall (14 A/channel paired — exceeds RoboClaw 2x7A peak), 700 CPR output encoder.

### [RoboClaw 2x5A](https://www.pololu.com/product/2394)
Superseded by RoboClaw 2x7A. 6–34 V, 5 A continuous / 10 A peak per channel — insufficient for paired motor stall. USB, TTL serial, RC, analog, built-in PID.

### Motor Comparison: FIT0186 vs Pololu 34:1

| Parameter | FIT0186 | Pololu 34:1 |
|---|---|---|
| No-load speed | 251 RPM | 300 RPM |
| No-load current | 350 mA / 4.2 W | 300 mA / 3.6 W |
| Stall current | 7.0 A / 84 W | 5.0 A / 60 W |
| Stall torque | 18 kg·cm | 11 kg·cm |
| Encoder CPR (output) | 700 | 1632 |
| 4-motor stall current | 28 A | 20 A |
| Paired channel stall | 14 A — exceeds RoboClaw 2x7A | 10 A — within peak |
| No-load speed (100 mm wheel) | ~1.31 m/s | ~1.57 m/s |
