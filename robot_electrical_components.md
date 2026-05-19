# Robot Electrical Components

## Purpose

Concise inventory of the robot's electrical and electronic components, focused on compatibility, power budget, wiring, signal levels, and Linorobot2/ROS integration.

---

## Components

### Raspberry Pi 4 Model B

**Power input:**

- 5 V via USB-C from UPS module
- Requires stable high-current 5 V supply

**Interfaces:**

| Connection | Purpose | Notes |
|---|---|---|
| USB | Sensors, microcontroller, camera, lidar | Devices TBD |
| GPIO | Encoder or control signals | Must be 3.3 V safe |
| I2C | Displays, sensors, power monitor | Check address conflicts |
| Ethernet/Wi-Fi | Development and ROS networking | TBD |

### Raspberry Pi I2C Shim

**Electrical characteristics:**

| Parameter | Value |
|---|---|
| Logic level | 3.3 V |
| Bus signals | SDA, SCL |
| Pi pins used | GPIO2 (SDA1), GPIO3 (SCL1) |

**Compatibility notes:**

- Shares the Pi's 3.3 V I2C bus.
- Verify connected devices use 3.3 V logic or have level shifting.
- Track I2C addresses as devices are added.

### Waveshare UPS Module (3S)

**Electrical characteristics:**

| Parameter | Value |
|---|---|
| Battery type | 3S Li-ion / LiPo |
| Battery voltage range | 9.0–12.6 V |
| 5 V output | Up to 5 A / 25 W |
| Charging | Yes |
| Charge and discharge simultaneously | Yes |

**Connections:**

| Connection | Purpose |
|---|---|
| BAT terminals | 3-cell battery pack |
| 5 V output | Raspberry Pi and USB peripherals |
| Charge input | External power adapter |
| Optional I2C | Battery monitoring |

**Compatibility notes:**

- 5 V rail must cover Raspberry Pi plus USB peripherals.
- Motor power should not be drawn from the 5 V rail.
- Optional I2C monitoring may be useful for ROS battery state.

### Panasonic NCR18650GA Cells

**Quantity:** 4 cells (3 installed, 1 spare)

**Key specifications (per cell):**

| Parameter | Value |
|---|---|
| Chemistry | Lithium-ion |
| Form factor | 18650 |
| Capacity | 3450 mAh |
| Nominal voltage | 3.6–3.7 V |
| Full charge voltage | 4.2 V |
| Continuous discharge | 10 A |

**3S Pack Characteristics:**

| Parameter | Value |
|---|---|
| Nominal voltage | 11.1 V |
| Full charge voltage | 12.6 V |
| Pack capacity | 3450 mAh |
| Stored energy | ~38 Wh |

**Estimated runtime:**

| Average Power | Estimated Runtime |
|---|---|
| 10 W | ~3.8 h |
| 20 W | ~1.9 h |
| 30 W | ~1.3 h |

### DFRobot FIT0186 Gearmotor with Encoder

**Part identification:**

| Field | Value |
|---|---|
| DigiKey part number | 1738-1106-ND |
| Manufacturer | DFRobot |
| Manufacturer part number | FIT0186 |
| Motor model | GB37Y3530-12V-251R |
| Description | 12 V DC gearmotor, 251 RPM, incremental encoder |

**Electrical and mechanical characteristics:**

| Parameter | Value |
|---|---:|
| Rated voltage | 12 VDC |
| Gear ratio | 43.8:1 |
| No-load speed | 251 RPM +/- 10% |
| No-load current | 350 mA |
| No-load electrical power | ~4.2 W |
| Stall current | 7 A |
| Stall electrical power | ~84 W |
| Stall torque | 18 kg.cm / ~1.77 N.m |
| Encoder operating voltage | 5 V |
| Encoder type | Hall / incremental |
| Encoder resolution | 16 CPR motor shaft / 700 CPR gearbox shaft |
| Output shaft | 6 mm D-shaft |
| Weight | ~205 g |

**Power budget notes:**

- One motor draws about 4.2 W at no-load and can draw about 84 W at stall.
- Two drive motors would draw about 8.4 W no-load and up to about 168 W at stall.
- The motor driver should tolerate at least 7 A peak per motor, with margin.
- The 3S battery rail can power this motor directly, but speed will vary with battery voltage.
- Approximate no-load speed range from the 3S pack is ~188 RPM at 9.0 V to ~264 RPM at 12.6 V, assuming speed scales roughly with voltage.
- Encoder power is 5 V; confirm encoder output signal level before connecting directly to Raspberry Pi GPIO.

---

## Component Inventory

| Component | Voltage | Current | Interface | Status |
|---|---:|---:|---|---|
| Raspberry Pi 4 Model B | 5 V | TBD | USB, GPIO, I2C, Wi-Fi/Ethernet | Known |
| Raspberry Pi I2C Shim | 3.3 V logic | Negligible | I2C | Known |
| Waveshare UPS Module (3S) | 9–12.6 V in, 5 V out | 5 A max | Power, optional I2C | Known |
| Panasonic NCR18650GA (3 used + 1 spare) | 3.6–4.2 V each | 10 A each | Battery pack | Known |
| DFRobot FIT0186 gearmotor w/encoder | 12 V motor, 5 V encoder | 350 mA no-load, 7 A stall | Motor power, encoder | Known |
| Motor driver | 9–12.6 V battery input | TBD | Motor power, control signals | TBD |

---

## Power Architecture

```text
3S Battery Pack
   ├──> Waveshare UPS Module ──> 5 V rail ──> Raspberry Pi 4B + USB peripherals
   └──> Motor driver VIN direct from battery voltage
```

## Power Budget

### 5 V Rail

The UPS 5 V output is limited to 5 A / 25 W. This rail should power only the Raspberry Pi and low-voltage peripherals.

| Load | Voltage | Current | Power | Status |
|---|---:|---:|---:|---|
| Raspberry Pi 4 Model B | 5 V | TBD | TBD | Required |
| USB peripherals | 5 V | TBD | TBD | TBD |
| I2C peripherals | 3.3/5 V | TBD | TBD | TBD |
| 5 V rail total | 5 V | TBD | Must be <=25 W | Open |

### Battery Voltage Rail

The 3S pack provides 9.0-12.6 V to motor power electronics. Motor loads should be budgeted separately from the 5 V rail.

| Load | Voltage | Current | Power | Status |
|---|---:|---:|---:|---|
| Motor driver | 9.0-12.6 V | TBD | TBD | Required |
| DFRobot FIT0186 drive motor, each | 12 V nominal | 0.35 A no-load / 7 A stall | ~4.2 W no-load / ~84 W stall | Known |
| Two drive motors | 12 V nominal | 0.7 A no-load / 14 A stall | ~8.4 W no-load / ~168 W stall | Planned |
| Battery rail total | 9.0-12.6 V | TBD | TBD | Open |

## Compatibility Notes

- Motor power comes directly from the 3S battery.
- Raspberry Pi power comes from the regulated 5 V UPS output.
- All control electronics should share a common ground.
- GPIO and I2C signals connected to the Pi must be 3.3 V safe.
- Motor encoder power is 5 V; verify encoder signal voltage before connecting to Pi GPIO.
- I2C address conflicts need to be checked once devices are selected.
- Stall current for two FIT0186 motors can exceed the 10 A continuous rating of a single-series 3S NCR18650GA pack, so current limiting, fusing, and mechanical stall avoidance matter.

## Open Questions

- Exact Raspberry Pi RAM size?
- Which USB devices will be attached?
- Which GPIO pins are reserved?
- Exact I2C shim model?
- Which I2C devices and addresses will be used?
- Which charger/power adapter will be used?
- Which peripherals will draw from the 5 V rail?
- What motor driver will be used for the DFRobot FIT0186 motors?
- What continuous motor current should be assumed under real robot load?
- How many FIT0186 motors will be installed?
- Are encoder output signals 3.3 V safe, or is level shifting required?
- Will battery monitoring be integrated into ROS?
- Are the three installed cells new and matched?
- What is the expected average system power draw?
