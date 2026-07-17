# STM32F411/BNO080 Self-Balancing Cube

Firmware and integration work for a three-axis reaction-wheel balancing cube based on the open-source [remrc/Self-Balancing-Cube](https://github.com/remrc/Self-Balancing-Cube) project.

This version replaces the original ESP32 and MPU6050 with:

- **STM32F411CEU6** microcontroller
- **GY-BNO080** intelligent IMU
- Three **Nidec 24H** reaction-wheel motors with integrated drivers and quadrature encoders
- **3S LiPo** power system

> [!WARNING]
> This project contains high-speed reaction wheels and a LiPo battery. Incorrect PWM polarity, wiring, firmware, or mechanical assembly can cause an unexpected full-power motor start. Read the safety section and complete every diagnostic stage before enabling closed-loop balancing.

## Project status

**Current phase: planning and hardware bring-up.**

The physical assembly and most electrical components are available. Firmware is being redesigned for the STM32F411 and BNO080; stable balancing firmware has not yet been released.

Development priorities:

1. Verify power, signal voltages, and pin allocation.
2. Bring up safe motor PWM and brake control.
3. Validate all three quadrature encoders.
4. Integrate and characterize the BNO080.
5. Implement calibration and fault handling.
6. Tune vertex balancing.
7. Add edge balancing and usability features.

See [PROJECT_PLAN.md](PROJECT_PLAN.md) for the complete architecture, milestones, risks, and acceptance criteria.

## Why this is a firmware port, not a board substitution

The original Arduino/ESP32 sketch cannot be compiled unchanged for this hardware. Several subsystems are fundamentally different:

| Original design | This design |
|---|---|
| ESP32 Arduino | STM32CubeIDE and STM32 HAL |
| MPU6050 raw accelerometer/gyro | BNO080 SH-2/SHTP quaternion and gyro reports |
| Software complementary filter | BNO080 onboard sensor fusion |
| GPIO encoder interrupts | STM32 hardware timer encoder mode |
| ESP32 LEDC PWM | STM32 timer PWM |
| EEPROM API | Versioned internal Flash record with CRC |
| BluetoothSerial | USB CDC or UART initially |
| 15 ms `millis()` loop | Deterministic 100 Hz hardware-timed loop |

The original three-wheel coordinate transformations and controller concept remain useful, but controller gains, timing, calibration, drivers, and safety behavior must be implemented and tested again.

## Hardware

### Required core components

- STM32F411CEU6 board, typically a Black Pill-style board
- GY-BNO080 breakout
- Three Nidec 24H motors with integrated motor electronics and encoder outputs
- 3S LiPo battery and suitable connector
- Regulated logic supply appropriate for the selected STM32 board and peripherals
- Motor brake, direction, PWM, and encoder wiring
- Battery-voltage divider
- Active buzzer or another fault indicator
- USB-to-UART adapter if USB CDC is not used
- Physical power disconnect

### Items that must be verified before connection

GY-BNO080 and STM32 breakout boards vary. Confirm from the actual boards and schematics:

- STM32 board HSE frequency; this project currently assumes 25 MHz
- BNO080 regulator and permitted supply voltage
- BNO080 logic voltage, pull-ups, level shifters, and interface-selection pins
- Motor encoder output voltage and whether it is push-pull or open-collector
- Need for external encoder pull-ups
- Maximum battery-divider voltage at a fully charged 3S pack
- Ground continuity between logic, IMU, motors, and battery measurement

Do not assume that a module accepting 5 V power also has 5 V-safe signal pins.

## Planned firmware architecture

```text
BNO080 interrupt
    -> SPI or I2C transfer
    -> SH-2/SHTP packet parser
    -> latest timestamped quaternion and gyro sample

100 Hz control timer
    -> validate IMU freshness
    -> read three encoder counters
    -> calculate quaternion balance error
    -> determine vertex, edge, fallen, or fault state
    -> run state-feedback controller
    -> mix X/Y/Z corrections into three wheel commands
    -> update PWM, direction, and brake

Background processing
    -> command interface and telemetry
    -> calibration workflow
    -> Flash parameter storage
    -> battery monitoring
    -> diagnostics and fault reporting
```

Serial output, Flash writes, and blocking IMU transfers must never run inside the control interrupt.

## Clock and timing

A 25 MHz crystal is the external clock source, not the intended CPU speed. The STM32F411 will be configured through its PLL for a target system clock of **100 MHz**.

Initial timing targets:

| Function | Target |
|---|---:|
| BNO080 Game Rotation Vector | 200 Hz |
| BNO080 calibrated gyro | 200 Hz |
| Balance controller | 100 Hz |
| Motor PWM | 20 kHz |
| Maximum accepted IMU age | 30 ms initially |

The final CubeMX clock tree must also produce a valid peripheral clock if USB CDC is enabled.

## BNO080 approach

The BNO080 is a sensor hub, not a register-compatible replacement for the MPU6050. It uses the SH-2/SHTP protocol and produces fused sensor reports.

The initial controller will use:

- **Game Rotation Vector** for orientation
- **Calibrated Gyroscope** for angular velocity

Game Rotation Vector is preferred over a magnetometer-dependent rotation vector because the motors and their currents may disturb the local magnetic field.

SPI is preferred when the breakout exposes it correctly. I2C at 400 kHz is the fallback. In either case, the BNO080 interrupt and reset signals should be connected, and every sample should be timestamped locally.

## Coordinate frames

All sensor measurements must be transformed into a documented mechanical cube frame before they reach the controller.

The mapping will be bench-tested so that positive physical rotation around cube X, Y, and Z produces positive cube-frame gyro X, Y, and Z respectively. Axis swaps and sign changes will be centralized in one module rather than distributed through the controller.

The wheel numbering, positive rotation direction, encoder sign, and cube-frame force direction will also be documented before closed-loop testing.

## Calibration

The MPU6050 accelerometer-offset procedure from the reference firmware will be replaced with quaternion reference calibration.

The firmware will store:

- Vertex reference quaternion
- Edge reference quaternion
- Sensor-to-cube mapping
- Controller gains
- Battery calibration
- Version, record length, magic value, and CRC32

At runtime the orientation error is calculated from:

```text
q_error = inverse(q_reference) * q_current
```

Calibration will only be accepted when IMU data is fresh, sensor accuracy is acceptable, gyro magnitude is low, and the pose remains stable for a defined interval.

Planned commands:

```text
cal vertex
cal edge
cal status
save
```

An absent, corrupt, or incompatible calibration record must prevent balancing.

## Controller

The controller will retain the reference project's general state-feedback structure while using explicit units:

```text
wheel command =
      angle gain * angle error
    + gyro gain * angular velocity
    + wheel velocity gain * wheel velocity
    + wheel position gain * accumulated wheel position
```

Yaw damping uses vertical-axis gyro and combined wheel motion.

The reference three-wheel transformation will be ported and verified against the actual wheel layout:

```text
m1 = (0.5*x - 0.866*y) / 1.37 + z
m2 = (0.5*x + 0.866*y) / 1.37 + z
m3 = -x / 1.37 + z
```

Original numeric gains will not be copied blindly. The new firmware uses different units, sample timing, sensor latency, encoder scaling, and motor-command scaling.

## Development environment

Planned tools:

- STM32CubeIDE
- STM32CubeMX configuration integrated with CubeIDE
- ST-LINK programmer/debugger
- Git
- Serial terminal
- Multimeter
- Oscilloscope or logic analyzer strongly recommended

The repository will eventually contain the complete CubeIDE project and generated configuration needed for a clean build.

## Planned source layout

```text
Core/Inc/                Public module headers
Core/Src/                Application and driver implementation
Drivers/                 STM32 HAL and device support
Docs/                    Wiring, coordinate frames, calibration, and tuning
Tests/                   Host-side and hardware test support
PROJECT_PLAN.md          Detailed engineering plan
README.md                 Project overview and bring-up guide
```

## Bring-up order

Do not skip ahead to balance tuning. Each stage is a gate for the next.

### 1. Power and MCU

- Verify every regulated rail without motor output enabled.
- Confirm the STM32 HSE and generated 100 MHz clock.
- Establish UART or USB diagnostics.
- Report firmware version and reset cause at boot.

### 2. Brake and PWM

- Assert brake before PWM initialization.
- Test one motor at a time at a low command limit.
- Verify direction control.
- Measure the real zero-torque PWM polarity.
- Confirm reset and debugger attachment never start a motor.

The reference Nidec interface uses active-low PWM. Treat that as a hypothesis to verify electrically, not an assumption.

### 3. Encoders

- Configure timer encoder mode where possible.
- Verify both channels of each encoder.
- Test direction, stationary noise, wraparound, and maximum speed.
- Document positive directions and counts per revolution.

### 4. BNO080

- Detect sensor reset and boot completion.
- Receive quaternion and gyro reports continuously.
- Measure report interval, age, and communication errors.
- Verify all cube-frame axes.
- Confirm individual motor operation does not interrupt reports.

### 5. Calibration and storage

- Capture stable vertex and edge references.
- Save a versioned record with CRC.
- Verify save, reboot, load, and corrupt-record behavior.

### 6. Output-limited closed-loop testing

- Use a support or tether.
- Apply a low motor-command limit.
- Verify by hand that every correction opposes the fall.
- Test stale-IMU and excessive-tilt shutdown.

### 7. Vertex tuning

- Tune angle feedback.
- Add gyro damping.
- Add wheel-velocity feedback.
- Add a small wheel-position term to prevent saturation.
- Add yaw damping.

### 8. Edge balancing and refinements

Only after vertex balance is reliable:

- Implement edge state detection and reference
- Tune an independent edge controller
- Add wireless tuning
- Add status LEDs and improved telemetry

## Serial interface

Initial transport will be UART or USB CDC. Planned commands include:

```text
help
status
faults
imu
encoders
battery
motor <1|2|3> <command>
motor stop
brake <on|off>
cal vertex
cal edge
cal status
save
set <parameter> <value>
get <parameter>
telemetry <on|off>
```

Manual motor commands will only work in a diagnostic state with the balance controller disabled.

## Safety behavior

The firmware must disable motor output if any of these conditions occurs:

- Boot or calibration is in progress
- IMU data is missing, invalid, or stale
- BNO080 reset is detected
- Tilt exceeds the allowed range
- Battery voltage is below the safe threshold
- Encoder plausibility fails
- Control-loop overrun occurs
- A NaN or infinity reaches the controller
- A stop command is received

Physical precautions:

- Wear eye protection during powered tests.
- Use a tether or fixture for initial control tests.
- Keep hands, hair, tools, and loose wiring away from the wheels.
- Secure the LiPo and all connectors mechanically.
- Keep an accessible power disconnect.
- Do not charge the LiPo inside the cube.
- Use independent undervoltage and charging protection appropriate for the pack.
- Never write Flash while the cube is balancing.

## Testing and telemetry

Host-side tests are planned for:

- Quaternion normalization, inversion, and sign handling
- Reference-orientation error
- Coordinate transformations
- Three-wheel mixing and inverse mixing
- Saturation and accumulated-state limits
- Flash record CRC and version checks
- Fault-state transitions

Runtime telemetry will include:

- Control-loop execution time
- IMU sample age and accuracy
- Quaternion and angle errors
- Gyro values
- Encoder deltas and wheel velocities
- Individual controller terms
- Motor commands before and after saturation
- Battery voltage
- Active state and fault flags

## Definition of the first release

Version `v0.1.0` will require:

- A clean build from documented STM32CubeIDE prerequisites
- Documented pin map and wiring
- Passing motor and encoder diagnostics
- Timestamped and fault-monitored BNO080 reports
- CRC-protected vertex calibration
- Repeatable vertex balance without wheel-speed saturation
- Reliable stale-IMU, over-tilt, low-battery, and software-fault shutdown
- Documented controller units, gains, calibration, and bring-up procedure

Edge balancing may ship in a later release if it is not reliable enough for `v0.1.0`.

## Contributing and development notes

Until the first hardware bring-up is complete, interface definitions, pin assignments, and controller parameters should be considered provisional.

When making changes:

- Keep hardware access separate from control mathematics.
- State physical units in names or comments.
- Do not mix sensor-frame and cube-frame values.
- Include safe defaults and failure behavior.
- Update documentation whenever pin assignments, coordinate signs, or calibration data change.
- Record test conditions and telemetry when changing controller gains.

## Acknowledgements

This project is derived from and inspired by Remigijus's [Self-Balancing-Cube](https://github.com/remrc/Self-Balancing-Cube). The original repository established the mechanical concept, Nidec motor interface, wheel mixing, balance-control structure, and practical build approach.

This repository is an independent STM32F411/BNO080 port and will require new firmware, calibration, validation, and gain tuning.

## License

No license has been selected yet. Until a license file is added, normal copyright restrictions apply to original work in this repository. Any reused or adapted material must continue to comply with its original licensing terms.
