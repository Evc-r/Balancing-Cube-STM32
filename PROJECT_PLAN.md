# STM32F411/ICM-20948 v2 Self-Balancing Cube Project Plan

## Current progress

CAD and mechanical assembly are completed. The first custom four-layer PCB design is completed and was sent for fabrication. The project is moving into firmware development/software integration. PCB rework and electrical validation are still pending; firmware bring-up and successful balancing have not been verified.

The custom PCB is part of the current build. A VBAT connection may bypass the protection circuit; a bodge repair is planned, but its completion and effectiveness are unverified. The STM32 header spacing is incorrect, so a wired STM32 connection is planned. The J7 IMU connector is unavailable, so directly soldering its wiring is planned. These repairs and connections have not been verified.

The architecture, rates, and interfaces below are **implementation targets**, not measured performance or completed firmware. M0 remains an open validation gate; M1–M9 are upcoming milestones. See [README.md](README.md) for the status summary and [JOURNAL.md](JOURNAL.md) for progress notes.

## 1. Goal

Port and improve the control system from [remrc/Self-Balancing-Cube](https://github.com/remrc/Self-Balancing-Cube) for this hardware:

- STM32F411CEU6 microcontroller, with an assumed 25 MHz HSE to confirm and a target 100 MHz system clock
- ICM-20948 v2 breakout IMU
- Three Nidec 24H reaction-wheel motors with direction, active-low PWM, shared brake, and quadrature encoders
- 3S LiPo power system

The first release target is safe, repeatable vertex balancing. Edge balancing and wireless tuning come afterward.

## 2. Scope

### Included

- Integration and validation of the existing custom four-layer PCB
- Native STM32CubeIDE/HAL firmware
- Three 20 kHz motor PWM outputs
- Three direction outputs and one shared brake output
- Three hardware quadrature encoder interfaces where pin mapping permits
- ICM-20948 v2 quaternion and calibrated gyro acquisition
- Sensor-to-cube coordinate transformation
- Quaternion-based vertex and edge calibration
- State-feedback balance controller
- Battery monitoring and fault shutdown
- USB CDC or UART diagnostics
- Flash-backed calibration and parameter storage

### Deferred until balancing works

- Bluetooth/BLE tuning
- WS2812 status LEDs
- Automated gain tuning
- Jump-up maneuver

## 3. Main changes from the reference firmware

| Reference project | This project |
|---|---|
| ESP32 Arduino | STM32CubeIDE with STM32 HAL |
| ESP32 LEDC PWM | STM32 timer PWM |
| GPIO encoder interrupts | Hardware timer encoder mode |
| MPU6050 raw readings | ICM-20948 v2 register/FIFO accelerometer and gyro samples |
| Original complementary filter | STM32 complementary/Mahony quaternion estimator |
| Accelerometer offset calibration | Quaternion reference calibration |
| EEPROM API | Internal Flash record with version and CRC |
| BluetoothSerial | USB CDC or UART first; external wireless later |
| 15 ms millis loop | Deterministic 200 Hz hardware-timed loop |

The original three-wheel mixing and controller structure will be retained only after signs, units, and scaling are verified.

## 4. Architecture

```text
ICM-20948 data-ready INT
    -> SPI/I2C register or FIFO transfer
    -> calibrated accelerometer + gyro sample
    -> STM32 attitude estimator
    -> latest timestamped quaternion + angular rate

200 Hz control timer
    -> copy latest valid IMU sample
    -> read encoder counters
    -> calculate quaternion error from balance reference
    -> run vertex/edge state machine
    -> run state-feedback controller
    -> convert X/Y/Z command to three wheel commands
    -> update PWM, direction, and brake

Background tasks
    -> serial commands and telemetry
    -> calibration workflow
    -> Flash save/load
    -> battery measurement
    -> fault reporting
```

The control interrupt must never print serial data, write Flash, or perform a blocking ICM-20948 v2 transaction.

## 5. Proposed repository structure

```text
Core/Inc/
  app_config.h
  app_types.h
  balance_controller.h
  icm20948_driver.h
  calibration.h
  command_interface.h
  coordinate_transform.h
  encoder.h
  fault_manager.h
  motor_control.h
  parameter_store.h
  telemetry.h
Core/Src/
  balance_controller.c
  icm20948_driver.c
  calibration.c
  command_interface.c
  coordinate_transform.c
  encoder.c
  fault_manager.c
  motor_control.c
  parameter_store.c
  telemetry.c
  main.c
Docs/
Tests/
PROJECT_PLAN.md
README.md
JOURNAL.md
```

Keep hardware access outside the controller mathematics so transforms, mixing, safety logic, and quaternion calculations can be unit tested.

## 6. Units and core data

Use explicit units everywhere:

- Angle: radians
- Angular velocity: radians/second
- Wheel velocity: counts/second initially, later radians/second
- Control period: seconds
- Motor command: normalized -1.0 to +1.0
- Battery voltage: volts

```c
typedef struct { float w, x, y, z; } Quaternion;
typedef struct { float x, y, z; } Vector3f;

typedef struct {
    Quaternion orientation;
    Vector3f accel_m_s2;
    Vector3f gyro_rad_s;
    uint32_t timestamp_us;
    uint8_t validity and saturation status;
    bool valid;
} ImuSample;

typedef struct {
    uint32_t version;
    Quaternion vertex_reference;
    Quaternion edge_reference;
    float sensor_to_cube_rotation[3][3];
    uint32_t crc32;
} CalibrationRecord;
```

## 7. Hardware and CubeMX plan

### Clock

- Confirm the board actually uses a 25 MHz HSE.
- Configure the PLL for a 100 MHz system clock.
- Verify APB timer clocks in CubeMX.
- If USB CDC is used, ensure a valid 48 MHz USB clock.
- Measure a timed GPIO output to confirm the generated clock configuration.

### Timer allocation

Conceptual allocation only; final assignments depend on exposed pins and alternate-function conflicts:

```text
TIM1 CH1-CH3: motor PWM 1-3
TIM2 CH1/CH2: encoder 1
TIM3 CH1/CH2: encoder 2
TIM4 CH1/CH2: encoder 3
TIM5: control scheduler or microsecond time base
```

Complete the CubeMX pin allocation before permanent harness or PCB wiring. If all three encoder timers cannot be mapped, use hardware encoder mode for the fastest wheels and a validated interrupt decoder for the remaining wheel.

### Motor interface

- PWM frequency: 20 kHz.
- Preserve active-low PWM until verified with a meter or oscilloscope.
- Initialize every PWM compare register to zero-torque before releasing brake.
- Keep the shared brake asserted during boot, faults, Flash writes, and debugger halts.
- Test one motor at a time with a low command limit.

### Encoders

- Confirm output voltage and whether outputs are push-pull or open-collector.
- Add external pull-ups if required.
- Do not exceed the selected STM32 pin voltage limits.
- Configure timer input filtering if noise creates false counts.
- Verify the direction sign of every wheel independently.

### ICM-20948 v2

Prefer SPI if the exact ICM-20948 v2 breakout board exposes it correctly. Otherwise use 400 kHz I2C with INT and RESET connected.

Inspect the exact breakout for:

- Regulator configuration
- Logic voltage
- I2C pull-ups
- Level shifters
- Interface-selection pins

Do not assume every board sold as ICM-20948 v2 breakout has the same circuit.

### Battery ADC

- Use a divider safe at maximum charged 3S voltage.
- Ensure the ADC input cannot exceed its limit.
- Add RC filtering near the MCU.
- Calibrate against a multimeter.
- Retain independent LiPo undervoltage protection.

## 8. IMU strategy

The ICM-20948 contains a 3-axis gyroscope, 3-axis accelerometer, AK09916 3-axis magnetometer, FIFO, programmable digital filters, and an embedded DMP. It supports SPI up to 7 MHz or I2C up to 400 kHz.

Initial implementation:

- Read raw accelerometer and gyroscope data through SPI where possible
- Configure suitable full-scale ranges and digital low-pass filters
- Use the data-ready interrupt and optionally the FIFO
- Sample accelerometer and gyro at 500 Hz initially
- Calibrate stationary gyro bias at startup
- Apply stored accelerometer scale/offset correction if characterization shows it is required
- Run a six-axis complementary or Mahony quaternion estimator on the STM32 at 500 Hz
- Run the balance controller at 200 Hz using the freshest complete estimate
- Leave the magnetometer disabled for balancing initially because motor currents can disturb it
- Defer DMP integration until the raw-data controller is validated

SPI is preferred for predictable high-rate acquisition. I2C at 400 kHz is acceptable for early tests. Every sample must be timestamped locally and checked for stale data, clipping, impossible jumps, communication errors, and estimator validity.

The exact v2 breakout must be inspected for regulator, voltage translation, pull-ups, address selection, and exposed INT/CS pins. The bare ICM-20948 has stricter VDDIO limits than 3.3 V, so breakout-level compatibility must be verified rather than assumed.

Initial timing targets:

```text
Accelerometer/gyro sampling: 2 ms (500 Hz)
STM32 attitude estimation:   2 ms (500 Hz)
Control-loop interval:       5 ms (200 Hz)
Maximum accepted sample age: 15 ms initially
```

The controller must enter a safe fault state if samples stop, WHO_AM_I/configuration checks fail, the sensor resets, readings saturate unexpectedly, or the attitude estimator becomes invalid.

## 9. Coordinate frames and calibration

### Sensor-to-cube transform

Implement one central sensor-to-cube transformation. Do not scatter sign swaps through controller code.

Verify on the bench:

- Positive cube rotation around X produces positive cube-frame gyro X.
- Positive cube rotation around Y produces positive cube-frame gyro Y.
- Positive cube rotation around Z produces positive cube-frame gyro Z.

Document the mapping in a future `Docs/COORDINATE_FRAMES.md`.

### Quaternion references

Store separate vertex and edge reference quaternions. At runtime:

```text
q_error = inverse(q_reference) * q_current
```

For small errors:

```text
angle_x ~= 2 * q_error.x
angle_y ~= 2 * q_error.y
angle_z ~= 2 * q_error.z
```

Normalize quaternions and enforce a consistent quaternion hemisphere so equivalent q and -q representations do not cause discontinuities.

Calibration commands:

```text
cal vertex
cal edge
cal status
save
```

Calibration capture requires fresh data, acceptable sensor accuracy, low gyro magnitude, and a stable orientation for a defined duration.

## 10. Controller plan

### Vertex controller

```text
u_x = K_angle_x * angle_error_x
    + K_gyro_x * gyro_x
    + K_wheel_v_x * wheel_velocity_x
    + K_wheel_p_x * wheel_position_x

u_y = K_angle_y * angle_error_y
    + K_gyro_y * gyro_y
    + K_wheel_v_y * wheel_velocity_y
    + K_wheel_p_y * wheel_position_y

u_z = K_yaw_rate * gyro_z
    + K_yaw_wheel * wheel_velocity_z
```

Use saturation and limit accumulated wheel-position terms.

### Three-wheel mixing

Port and verify the reference equations:

```text
m1 = (0.5*x - 0.866*y) / 1.37 + z
m2 = (0.5*x + 0.866*y) / 1.37 + z
m3 = -x / 1.37 + z
```

Verify the inverse transformation against actual wheel locations and encoder signs.

### Tuning order

1. Verify signs through hand-held, output-limited tests.
2. Add low-gain angle feedback.
3. Add gyro damping.
4. Tune angle and gyro terms for bounded behavior.
5. Add wheel-velocity feedback.
6. Add a small wheel-position term to prevent saturation.
7. Add yaw damping.
8. Implement and tune edge balancing separately.

Do not copy the original numeric gains: the units, loop period, IMU latency, encoder scaling, and motor-command scaling have changed.

## 11. Safety requirements

Disable motor output when:

- Firmware is booting or calibrating.
- IMU data is missing, stale, or invalid.
- A ICM-20948 v2 reset is detected.
- Tilt exceeds the configured fall threshold.
- Battery voltage is below the safe threshold.
- Encoder plausibility fails.
- The control loop overruns.
- A NaN or infinity enters the controller.
- An explicit stop command is received.

Additional requirements:

- Begin with a low motor-command limit.
- Use a physical support, tether, eye protection, and an accessible power disconnect.
- Secure the battery and wiring.
- Never write Flash while balancing.
- Ensure debugger halt cannot leave motor PWM active.
- Use independent 3S LiPo protection.

## 12. Serial interface

Initial transport: UART or USB CDC.

Planned commands:

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

Manual motor commands are allowed only in a diagnostic state with the balance controller disabled.

## 13. Flash storage

Store a versioned record containing:

- Magic value
- Structure version and length
- Vertex and edge reference quaternions
- Axis mapping
- Controller gains
- Battery calibration
- CRC32

Invalid magic, version, length, or CRC must load safe defaults and require calibration. Save only with motor output disabled and brake asserted.

## 14. Milestones

### M0 - Hardware confirmation

- Document and verify the planned VBAT/protection repair before powering the integrated system; confirm the intended power path and protection behavior.
- Map and continuity-check the planned wired STM32 connection and direct-soldered IMU connection.
- Record exact STM32 board and ICM-20948 v2 breakout module.
- Confirm motor connector, encoder type, and logic voltage.
- Confirm battery, regulator, divider, and brake wiring.
- Complete a conflict-free CubeMX draft pin map.

Exit: planned repairs and connections are completed and verified, the intended power path and protection behavior are confirmed, and no voltage-level or pin-allocation questions remain.

### M1 - STM32 bring-up

- Generate CubeMX project.
- Verify intended system clock.
- Enable UART or USB logging.
- Add microsecond timing and fault indication.

Exit: measured clock and stable diagnostics.

### M2 - Motor and brake diagnostics

- Configure PWM, direction, and shared brake.
- Reproduce the reference motor test at low command.
- Confirm true zero-torque PWM polarity.

Exit: each motor runs independently in both directions, and resets never cause motion.

### M3 - Encoder bring-up

- Configure three encoder interfaces.
- Measure signed deltas at a fixed interval.
- Test direction, stationary noise, wraparound, and maximum speed.

Exit: all encoders count reliably with documented signs.

### M4 - ICM-20948 v2 bring-up

Tasks:

- Verify WHO_AM_I and banked-register access.
- Configure accelerometer and gyro ranges, sample rates, filters, interrupt, and FIFO if used.
- Calibrate stationary gyro bias.
- Timestamp samples and monitor data age, saturation, and communication faults.
- Implement sensor-to-cube mapping.
- Implement and validate the STM32 attitude estimator.

Exit criteria:

- Raw accelerometer and gyro samples remain continuous while motors run individually.
- Sample interval, bias, clipping, and estimator health are monitored.
- All three cube-frame gyro signs are verified.
- Static orientation estimates are stable and return correctly after slow tilts.
- A disconnected, reset, or misconfigured IMU generates a safe fault.

### M5 - Calibration and persistence

- Capture stable vertex and edge references.
- Implement quaternion error.
- Add versioned Flash storage with CRC.

Exit: calibration survives reboot; absent or corrupt calibration cannot enable balancing.

### M6 - Low-power closed-loop tests

- Port wheel mixing.
- Add output-limited angle and gyro control.
- Verify every correction sign while holding the cube.
- Add stale-data and over-angle shutdown.

Exit: corrections oppose disturbances on both axes, with reliable fault shutdown.

### M7 - Vertex balancing

- Tune angle and gyro terms.
- Add wheel-velocity and wheel-position feedback.
- Add yaw damping and telemetry.

Exit: repeatable balance without wheel saturation and with predictable fall shutdown.

### M8 - Edge balancing

- Determine correct edge axis and wheel combination.
- Implement edge detection and independent gains.

Exit: repeatable edge balance without degrading vertex behavior.

### M9 - Release preparation

- Add optional wireless tuning and status indicators.
- Complete build, wiring, calibration, and tuning documentation.
- Tag the first stable release.

Exit: another person can build, flash, calibrate, and test the cube from the repository instructions.

## 15. Testing and telemetry

Host-side tests should cover:

- Quaternion normalization, inverse, and hemisphere handling
- Reference error calculation
- Coordinate transforms
- Wheel mixing and inverse mixing
- Saturation and accumulated-state limiting
- Flash CRC and version validation
- Fault-state transitions

Telemetry should include:

- Timestamp and loop execution time
- IMU age and accuracy
- Quaternion and angle errors
- Gyro X/Y/Z
- Encoder deltas and wheel velocities
- Individual controller terms
- Saturated and unsaturated motor commands
- Battery voltage
- Active state and fault flags

## 16. Principal risks

| Risk | Mitigation |
|---|---|
| CEU6 timer pin conflicts | Finish CubeMX allocation before wiring |
| Active-low PWM causes startup torque | Measure idle polarity and initialize PWM before brake release |
| Encoder voltage exceeds GPIO limit | Verify electrically and level-shift if required |
| ICM-20948 v2 breakout variants differ | Inspect and document the exact module |
| Fusion latency destabilizes control | Timestamp samples and tune controlled bandwidth |
| Motor magnetic interference | Keep the magnetometer out of the initial balance estimator |
| Quaternion sign discontinuity | Enforce a consistent hemisphere |
| Coordinate signs are incorrect | Centralize and bench-verify transforms |
| Original gains have incompatible units | Retune from low output limits using explicit units |
| Flash stalls execution | Save only while motors are disabled |
| LiPo over-discharge | Conservative thresholds plus independent protection |
| Debug halt leaves motors active | Safe debug configuration and physical disconnect |

## 17. Definition of v0.1.0

The first successful release requires:

- Clean STM32CubeIDE build from documented prerequisites
- Documented wiring and pin map
- Passing motor and encoder diagnostics
- Timestamped and fault-monitored ICM-20948 v2 reports
- CRC-protected vertex calibration
- Repeatable vertex balance without wheel saturation
- Reliable stale-IMU, over-tilt, low-battery, and software-fault shutdown
- Documented controller units and gains
- Complete bring-up and calibration instructions

Edge balancing may be released later if it is not reliable enough for v0.1.0.

## 18. Immediate next actions

Before powered bring-up: document the PCB rework, verify the intended power path and protection behavior after the planned repair, and check the STM32/IMU wiring against the schematic. Add CAD, KiCad sources, and photos to the repository as they become available. Start session-based journaling and time tracking with firmware work.

1. Document both sides and pin labels of the exact STM32F411 and ICM-20948 v2 breakout boards.
2. Confirm the 25 MHz HSE and decide whether USB CDC is required.
3. Build the complete pin inventory and CubeMX timer allocation.
4. Measure motor encoder logic voltage and determine pull-up requirements.
5. Create the STM32CubeIDE project and UART diagnostics.
6. Implement brake-safe boot behavior before applying motor power.
7. Bring up one PWM channel and motor at low command.
8. Bring up its encoder in timer encoder mode.
9. Repeat for all motors and encoders.
10. Implement ICM-20948 v2 communication and log quaternion, gyro, interval, age, and accuracy before writing balance-control code.
