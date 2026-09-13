# STM32F411/ICM-20948 v2 Self-Balancing Cube

Firmware and integration work for a three-axis reaction-wheel balancing cube based on the open-source [remrc/Self-Balancing-Cube](https://github.com/remrc/Self-Balancing-Cube) project.

This version replaces the original ESP32 and MPU6050 with:

- **STM32F411CEU6** microcontroller
- **ICM-20948 v2 breakout** intelligent IMU
- Three **Nidec 24H** reaction-wheel motors with integrated drivers and quadrature encoders
- **3S LiPo** power system

> [!WARNING]
> This project contains high-speed reaction wheels and a LiPo battery. Incorrect PWM polarity, wiring, firmware, or mechanical assembly can cause an unexpected full-power motor start. Read the safety section and complete every diagnostic stage before enabling closed-loop balancing.

## Project status

**Current phase: transitioning into firmware development and software integration.** CAD and the first custom PCB design are complete; electrical bring-up and balancing validation remain ahead.

| Area | Current progress |
|---|---|
| Mechanical design / CAD | Complete for the current build |
| Mechanical assembly | Frame and reaction-wheel assembly completed; final wiring/integration remains |
| Custom four-layer PCB | First design complete and sent for fabrication; known issues require rework and validation |
| STM32 connection | Header spacing mismatch identified; plan to connect the board through wires |
| IMU connection | J7 connector unavailable; plan to solder the IMU wiring directly |
| Firmware | Next active development phase: STM32 project setup, peripheral bring-up, and integration |
| Closed-loop balancing | Not yet demonstrated or validated |

### Current hardware issues

- **Power protection:** an unintended VBAT connection was identified that may bypass the protection stage. A trace-isolation/bodge-wire repair is planned. The repair and its effectiveness have not yet been verified.
- **STM32 header spacing:** the PCB's two long header rows do not match the STM32 board. A wired connection is planned, with a pin-by-pin wiring map and continuity checks before power-up.
- **IMU J7 connection:** direct-soldered wiring is planned because the connector is unavailable. Pin order, logic compatibility, and strain relief still need verification.

These are open integration tasks, not completed fixes. The first PCB revision should not be treated as a validated design for replication.

### Next milestones

1. Document the as-built wiring and verify the PCB rework, power rails, and pin allocation.
2. Create the STM32CubeIDE project and establish basic diagnostics.
3. Bring up safe motor PWM/brake control and validate the encoders.
4. Integrate and characterize the ICM-20948 v2.
5. Implement attitude estimation, calibration, and fault handling.
6. Perform output-limited control tests, then tune vertex balancing.
7. Add edge balancing and usability features after the initial controller is reliable.

See [JOURNAL.md](JOURNAL.md) for retrospective CAD/PCB notes and future session logs, and [PROJECT_PLAN.md](PROJECT_PLAN.md) for the planned architecture and acceptance criteria.

### Documentation and project files

The repository currently contains project documentation, not a buildable firmware release. CAD, KiCad source files, manufacturing outputs, photos, and firmware still need to be added here. The existing [project media folder](https://drive.google.com/drive/folders/1Wyz73s2HkP9Nr1Q7drfzrUGMNGJb-3j-?usp=drive_link) is linked for reference.

Earlier CAD/PCB work is documented retrospectively without reconstructed work hours. Future sessions should record what changed, evidence, test results, and actual tracked time.

## Why this is a firmware port, not a board substitution

The original Arduino/ESP32 sketch cannot be compiled unchanged for this hardware. Several subsystems are fundamentally different:

| Original design | This design |
|---|---|
| ESP32 Arduino | STM32CubeIDE and STM32 HAL |
| MPU6050 raw accelerometer/gyro | ICM-20948 v2 raw accelerometer/gyro with STM32 attitude estimation |
| Original complementary filter | STM32 complementary/Mahony filter using calibrated ICM-20948 data |
| GPIO encoder interrupts | STM32 hardware timer encoder mode |
| ESP32 LEDC PWM | STM32 timer PWM |
| EEPROM API | Versioned internal Flash record with CRC |
| BluetoothSerial | USB CDC or UART initially |
| 15 ms `millis()` loop | Deterministic 200 Hz hardware-timed loop |

The original three-wheel coordinate transformations and controller concept remain useful, but controller gains, timing, calibration, drivers, and safety behavior must be implemented and tested again.

## Hardware

### Required core components

- STM32F411CEU6 board, typically a Black Pill-style board
- ICM-20948 v2 breakout
- Three Nidec 24H motors with integrated motor electronics and encoder outputs
- 3S LiPo battery and suitable connector
- Regulated logic supply appropriate for the selected STM32 board and peripherals
- Motor brake, direction, PWM, and encoder wiring
- Battery-voltage divider
- Active buzzer or another fault indicator
- USB-to-UART adapter if USB CDC is not used
- Physical power disconnect

### Items that must be verified before connection

ICM-20948 v2 breakout and STM32 breakout boards vary. Confirm from the actual boards and schematics:

- STM32 board HSE frequency; this project currently assumes 25 MHz
- ICM-20948 v2 regulator and permitted supply voltage
- ICM-20948 v2 logic voltage, pull-ups, level shifters, and interface-selection pins
- Motor encoder output voltage and whether it is push-pull or open-collector
- Need for external encoder pull-ups
- Maximum battery-divider voltage at a fully charged 3S pack
- Ground continuity between logic, IMU, motors, and battery measurement

Do not assume that a module accepting 5 V power also has 5 V-safe signal pins.

## Planned firmware architecture

The architecture, interfaces, safety behavior, and timing values below are implementation targets. Firmware bring-up, execution timing, and successful balancing have not been verified.

```text
ICM-20948 v2 interrupt
    -> SPI or I2C transfer
    -> register/FIFO driver and sample validation
    -> calibrated accelerometer and gyro sample
    -> STM32 attitude estimator
    -> latest timestamped quaternion and angular rate

200 Hz control timer
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

A 25 MHz external crystal is assumed for the selected board and still needs confirmation; it is not the intended CPU speed. The STM32F411 will be configured through its PLL for a target system clock of **100 MHz**.

Initial timing targets:

| Function | Target |
|---|---:|
| ICM-20948 accelerometer and gyro sampling | 500 Hz initially |
| STM32 attitude estimator | 500 Hz initially |
| Balance controller | 200 Hz initially |
| Motor PWM | 20 kHz |
| Maximum accepted IMU age | 15 ms initially |

The final CubeMX clock tree must also produce a valid peripheral clock if USB CDC is enabled.

## ICM-20948 v2 approach

The ICM-20948 is a 9-axis IMU containing a 3-axis gyroscope, 3-axis accelerometer, AK09916 3-axis magnetometer, FIFO, programmable filters, and an embedded DMP. It supports SPI up to 7 MHz or I2C up to 400 kHz. The initial controller will read raw accelerometer and gyroscope data and perform attitude estimation on the STM32.

Initial sensing strategy:

- Gyroscope and accelerometer sampled at 500 Hz
- Configurable digital low-pass filtering
- Startup stationary gyro-bias calibration
- Six-axis complementary or Mahony quaternion estimator on the STM32
- Magnetometer disabled for balancing initially because motor currents can disturb it
- DMP integration deferred until the raw-data controller is validated

SPI is preferred for deterministic high-rate acquisition. I2C is acceptable for initial testing if the breakout exposes it more conveniently. Connect and use the data-ready interrupt. Every accepted sample must be timestamped locally, checked for plausibility, and monitored for staleness.

The exact v2 breakout must be inspected for its regulator, level shifting, pull-ups, address selection, and exposed interrupt/chip-select pins. The bare ICM-20948 has lower VDDIO limits than the STM32's 3.3 V logic, so compatibility depends on the breakout circuitry.

## Coordinate frames

All sensor measurements must be transformed into a documented mechanical cube frame before they reach the controller.

The mapping will be bench-tested so that positive physical rotation around cube X, Y, and Z produces positive cube-frame gyro X, Y, and Z respectively. Axis swaps and sign changes will be centralized in one module rather than distributed through the controller.

The wheel numbering, positive rotation direction, encoder sign, and cube-frame force direction will also be documented before closed-loop testing.

## Calibration

The MPU6050 offset procedure will be replaced with two layers: stationary accelerometer/gyro calibration, followed by quaternion reference calibration for the mechanical vertex and edge poses.

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
JOURNAL.md               Retrospective hardware notes and future session logs
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

### 4. ICM-20948 v2

- Verify WHO_AM_I and configure the register banks, ranges, filters, FIFO, and data-ready interrupt.
- Read calibrated accelerometer and gyro samples continuously.
- Run the STM32 attitude estimator and measure sample interval, age, bias, saturation, and communication errors.
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
- IMU data is missing, invalid, saturated, or stale
- ICM-20948 v2 identity/configuration check fails or a reset is detected
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
- Timestamped and fault-monitored ICM-20948 v2 samples and STM32 attitude estimates
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

This repository is an independent STM32F411/ICM-20948 v2 port and will require new firmware, calibration, validation, and gain tuning.

## License

No license has been selected yet. Until a license file is added, normal copyright restrictions apply to original work in this repository. Any reused or adapted material must continue to comply with its original licensing terms.
