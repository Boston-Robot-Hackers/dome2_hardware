# Robot Electrical Components

## Purpose

Electrical specification focused on power rails, power budget, wiring constraints, signal levels, and compatibility risks. The full robot BOM and mechanical sizing notes live in `robot_design.md`.

---

## Components

### [Raspberry Pi 4 Model B](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)

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
- Exact product link is TBD until the shim model is identified.

### [Waveshare UPS Module (3S)](https://www.waveshare.com/product/ups-module-3s.htm)

**Electrical characteristics:**

| Parameter | Value |
|---|---|
| Battery type | 3S Li-ion / LiPo |
| Battery voltage range | 9.0–12.6 V |
| 5 V output | Up to 5 A / 25 W |
| Charging | Yes |
| Charge and discharge simultaneously | Yes |

**Connections:**

| Connection | Purpose | Electrical notes |
|---|---|---|
| BAT terminals | 3-cell battery pack | 3S Li-ion pack input |
| 5 V output | Raspberry Pi and USB peripherals | Regulated 5 V, up to 5 A / 25 W |
| 3.3 V output | Low-power logic peripherals | Regulated 3.3 V, up to 300 mA |
| XH2.54 battery-voltage output | Battery-voltage output | Unregulated pack voltage, about 9.0-12.6 V, documented as 12.6 V / 2 A |
| Charge input | External power adapter | 12.6 V / 2 A charger input |
| Optional I2C | Battery monitoring | Reports battery voltage, current, power, and related status |

**Compatibility notes:**

- 5 V rail must cover Raspberry Pi plus USB peripherals.
- Motor power should not be drawn from the 5 V rail.
- The XH2.54 battery-voltage output follows the 3S pack voltage and is not regulated.
- The XH2.54 battery-voltage output is documented as 12.6 V / 2 A, so it is not suitable as the main drive-motor supply.
- Optional I2C monitoring may be useful for ROS battery state.

### [Panasonic NCR18650GA Cells](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA)

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

### [DFRobot FIT0186 Gearmotor with Encoder](https://www.dfrobot.com/product-634.html)

**Part identification:**

| Field | Value |
|---|---|
| DigiKey part number | 1738-1106-ND |
| Manufacturer | DFRobot |
| Manufacturer part number | FIT0186 |
| Motor model | GB37Y3530-12V-251R |
| Description | 12 V DC gearmotor, 251 RPM, incremental encoder |
| DigiKey link | https://www.digikey.com/en/products/detail/dfrobot/FIT0186/6588528 |

**Electrical and mechanical characteristics:**

| Parameter | Value |
|---|---:|
| Rated voltage | 12 VDC |
| No-load current | 350 mA |
| No-load electrical power | ~4.2 W |
| Stall current | 7 A |
| Stall electrical power | ~84 W |
| Encoder operating voltage | 5 V |
| Encoder type | Hall / incremental |

**Power budget notes:**

- One motor draws about 4.2 W at no-load and can draw about 84 W at stall.
- Four drive motors would draw about 16.8 W no-load and up to about 336 W at stall.
- The motor driver should tolerate at least 7 A peak per motor, with margin.
- The 3S battery rail can power this motor directly, but speed will vary with battery voltage.
- Encoder power is 5 V; confirm encoder output signal level before connecting directly to Raspberry Pi GPIO.



### [RoboClaw 2x5A Motor Controller](https://www.pololu.com/product/2394)

**Status:** Alternative motor controller under consideration

| Parameter | Value |
|---|---:|
| Motor channels | 2 brushed DC motors |
| Motor supply voltage | 6-34 V |
| Continuous current | 5 A per channel |
| Peak current | 10 A per channel |
| Control interfaces | USB, TTL serial, RC, analog |
| Encoder support | Dual quadrature encoders |
| Closed-loop control | Built-in speed/position PID |

**Fit for this robot:**

- Voltage range matches the 3S battery rail.
- Encoder support is a good match for the DFRobot FIT0186 motors.
- Current rating is marginal but plausible for a light robot: each FIT0186 motor is 350 mA no-load and 7 A stall.
- RoboClaw current capacity is not the main system limit if motor power is routed through the Waveshare UPS module's battery-voltage output.


---

## Electrical Power Components

| Component | Link | Voltage | Current | Interface | Status |
|---|---|---:|---:|---|---|
| Raspberry Pi 4 Model B | [Manufacturer](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/) | 5 V | TBD | USB, GPIO, I2C, Wi-Fi/Ethernet | Known |
| Raspberry Pi I2C Shim | TBD | 3.3 V logic | Negligible | I2C | Known |
| Waveshare UPS Module (3S) | [Manufacturer](https://www.waveshare.com/product/ups-module-3s.htm) | 9–12.6 V in, 5 V out | 5 A max | Power, optional I2C | Known |
| Panasonic NCR18650GA (3 used + 1 spare) | [Manufacturer](https://energy.panasonic.com/eu/business/products/lithium-ion/models/NCR18650GA) | 3.6–4.2 V each | 10 A each | Battery pack | Known |
| DFRobot FIT0186 gearmotor w/encoder | [Manufacturer](https://www.dfrobot.com/product-634.html), [DigiKey](https://www.digikey.com/en/products/detail/dfrobot/FIT0186/6588528) | 12 V motor, 5 V encoder | 350 mA no-load, 7 A stall | Motor power, encoder | Known |
| Motor driver | TBD | 9–12.6 V battery input | TBD | Motor power, control signals | TBD |
| RoboClaw 2x5A motor controller | [Pololu](https://www.pololu.com/product/2394) | 6-34 V motor input | 5 A continuous / 10 A peak per channel | USB, TTL serial, RC, analog, encoders | Alternative; one controller if motors are paired left/right |

---

## Power Architecture

```text
3S Battery Pack
   ├──> Waveshare UPS switch ──> 5 V rail ──> Raspberry Pi 4B + USB peripherals
   └──> Fuse ──> motor power switch / e-stop ──> Motor driver VIN ──> Drive motors
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
| Waveshare UPS battery-voltage output | Up to 12.6 V | 2 A documented output | ~25 W at 12.6 V | Not enough for drive motors |
| Motor driver | 9.0-12.6 V | TBD | TBD | Required |
| RoboClaw 2x5A, if used | 6-34 V | 5 A continuous / 10 A peak per channel | Each channel drives two motors in parallel; possible 14 A stall per channel | Alternative, marginal |
| DFRobot FIT0186 drive motor, each | 12 V nominal | 0.35 A no-load / 7 A stall | ~4.2 W no-load / ~84 W stall | Known |
| Four FIT0186 drive motors | 12 V nominal | 1.4 A no-load / 28 A stall total | ~16.8 W no-load / ~336 W stall total | Planned |
| Left motor pair | 12 V nominal | 0.7 A no-load / 14 A stall | ~8.4 W no-load / ~168 W stall | Front+rear left on one controller channel |
| Right motor pair | 12 V nominal | 0.7 A no-load / 14 A stall | ~8.4 W no-load / ~168 W stall | Front+rear right on one controller channel |
| Ideal lower-power replacement motor, each | 12 V nominal | <0.25 A no-load / 2-3 A stall | Lower motor-rail stress | Target |
| Four ideal replacement motors | 12 V nominal | <1.0 A no-load / 8-12 A stall | Better match to compact robot power path | Target |
| Battery rail total | 9.0-12.6 V | TBD | TBD | Open |

## Switching and Protection

The robot intentionally has two power switches:

| Switch | Controls | Purpose |
|---|---|---|
| UPS switch | Raspberry Pi, 5 V logic rail, low-voltage peripherals | Allows clean Pi boot/shutdown and software debugging |
| Motor power switch / e-stop | Motor driver and drive motors | Allows motion to be disabled while keeping the Pi and ROS alive |

Recommended motor power path:

```text
3S battery positive
   -> fuse close to battery
   -> motor power switch or e-stop
   -> motor driver VIN+

3S battery negative
   -> motor driver VIN-
   -> common ground reference
```

First-pass protection target: use a DC-rated fuse, switch, wiring, and connector sized for the expected motor current. A 7.5 A or 10 A automotive blade fuse is a reasonable starting point for controlled bench testing, but the final value should be chosen after measuring real motor current under load.

## Compatibility Notes

- Motor power should come from a separately fused/switched battery rail.
- Raspberry Pi power comes from the regulated 5 V UPS output.
- The UPS switch and motor switch should remain separate so motion can be disabled while the Pi stays powered.
- All control electronics should share a common ground.
- GPIO and I2C signals connected to the Pi must be 3.3 V safe.
- Motor encoder power is 5 V; verify encoder signal voltage before connecting to Pi GPIO.
- I2C address conflicts need to be checked once devices are selected.
- The Waveshare UPS 3S XH battery-voltage output is documented as 12.6 V / 2 A, so it should not be treated as a high-current motor rail.
- Stall current for four FIT0186 motors can reach 28 A total, far above the 10 A continuous rating of a single-series 3S NCR18650GA pack, so current limiting, fusing, and mechanical stall avoidance matter.
- A lower-power motor target would keep four-motor stall current around 8-12 A total, reducing stress on the battery rail and protection components.
- With paired left/right drive, one RoboClaw 2x5A could command four motors using two channels, but each channel would drive two motors in parallel. A FIT0186 pair can theoretically reach 14 A stall per channel, above the RoboClaw 2x5A peak rating. The battery path must be rated for motor current independently of the controller rating.

## Open Questions

- Exact Raspberry Pi RAM size?
- Which USB devices will be attached?
- Which GPIO pins are reserved?
- Exact I2C shim model?
- Which I2C devices and addresses will be used?
- Which charger/power adapter will be used?
- Which peripherals will draw from the 5 V rail?
- Can one RoboClaw 2x5A safely drive paired front/rear motors on each left/right channel, or is a higher-current left/right controller needed?
- What exact fuse rating, switch rating, wire gauge, and connector will be used for the motor rail?
- Will the motor switch be a simple power switch, a large accessible e-stop, or both?
- What continuous motor current should be assumed under real robot load?
- Four FIT0186 motors are currently assumed; confirm whether all four are powered drive motors.
- Are encoder output signals 3.3 V safe, or is level shifting required?
- Will battery monitoring be integrated into ROS?
- Are the three installed cells new and matched?
- What is the expected average system power draw?
