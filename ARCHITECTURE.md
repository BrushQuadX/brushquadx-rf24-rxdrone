# Architecture

## Scope

This firmware is a MultiWii-derived flight controller for a brushed Quad-X. The active build selects the nRF24 receiver path, four motor outputs, an MPU6050 IMU, and a BMP085 barometer. Many other MultiWii drivers remain in the repository, but they are compile-time optional and are not active unless enabled in `config.h`.

## Runtime pipeline

```text
setup()
	|-- initialize serial, pins, PWM, EEPROM configuration
	|-- configure nRF24 receiver
	|-- initialize sensors and calibration timers
	`-- initialize optional display, GPS, alarms, and logging features

loop()
	|-- read nRF24 packet and update RC channels
	|-- every 20 ms: compute RC values and process stick commands
	|-- wait for the configured 2.8 ms control period
	|-- compute IMU attitude from gyro and accelerometer
	|-- calculate roll, pitch, and yaw PID corrections
	|-- mix corrections into four Quad-X motor commands
	`-- write motor PWM outputs
```

The 20 ms RC command processing cadence is independent of the faster control loop. `LOOP_TIME` is set to 2,800 microseconds, although sensor, PID, and output work can affect the measured cycle time.

## Startup and configuration

`MultiWii.cpp::setup()` configures the four motor pins, serial port, status pins, output timers, and EEPROM-backed settings. It then initializes the receiver and sensors and starts gyro and barometer calibration. The active compile-time configuration is spread across:

- `config.h`: frame, sensor, receiver, timing, tuning defaults, pins, and optional features
- `def.h`: processor detection, motor count, board-specific definitions, and derived feature macros
- `types.h`: RC channel indexes, flight flags, sensor data, attitude, altitude, and configuration structures

The active selections are `QUADX`, `MPU6050`, `BMP085`, `NRF24_RX`, `PID_CONTROLLER 1`, `ONLYARMWHENFLAT`, and `BUZZER`. The ATmega328P is detected as `PROMINI`; the four motor outputs are D9, D6, D5, and D3.

## Receiver boundary

`NRF24_RX.cpp` owns the RF24 object and listens on pipe `0xE8E8F0F0E1` using CE D7 and CSN D10. It disables auto-acknowledgement and reads the following seven-byte `RF24Data` structure:

| Field | Meaning |
| --- | --- |
| `throttle` | Throttle byte, inverted into the throttle RC channel |
| `yaw` | Current transmitter byte used for receiver roll |
| `pitch` | Pitch byte |
| `roll` | Current transmitter byte used for receiver yaw |
| `AUX1`, `AUX2` | Two binary auxiliary switches |
| `switches` | Reserved in the current receiver conversion |

Values are mapped from 0-255 to 2,000-1,000 microseconds. If no packet arrives for 1,000 ms, `MyData` is reset to centered sticks and inactive auxiliary switches. This prevents stale commands from remaining active, but it is not the timed MultiWii landing failsafe: `FAILSAFE` is disabled in the active `config.h`.

For each received packet, the receiver prepares an `RF24AckPayload` containing attitude angles, heading, estimated altitude, and packed flight flags. Latitude and longitude are currently hard-coded in `NRF24_Read_RC()` and must not be treated as live GPS data.

## Estimation and control

`Sensors.cpp` selects the MPU6050 and BMP085 drivers through the active sensor macros. `IMU.cpp` filters the raw sensor data and computes attitude angles. The configured gyro low-pass filter is 42 Hz, with gyro smoothing enabled as `{20, 20, 3}` for roll, pitch, and yaw.

The main loop uses PID controller 1. Roll and pitch use gyro rate control, with accelerometer-based angle correction when ANGLE or HORIZON mode is active. Yaw is gyro controlled, with magnetic heading correction only when magnetometer support is enabled. The resulting `axisPID` values are passed to `mixTable()`.

`Output.cpp` applies the Quad-X mixer, constrains motor commands, and writes them as direct PWM values. Motor commands use the 1,000-2,000 range, with `MINTHROTTLE` set to 1,050 for brushed motors and `MINCOMMAND` set to 1,000 while disarmed.

## Arming and safety

Arming is controlled by the configured AUX arm box and the optional yaw stick command. `ONLYARMWHENFLAT` requires accelerometer calibration before arming. Disarming clears the armed flag and causes the motor output path to use the disarmed command. The buzzer and status LED report selected alarm and state conditions.

The source contains the standard MultiWii `FAILSAFE` implementation, but the active build does not define `FAILSAFE`. A production configuration should enable and test it if signal-loss landing and automatic motor shutdown are required; otherwise the nRF24 one-second reset only supplies neutral RC values.

## Serial and persistence interfaces

Serial port 0 runs at 115,200 baud and is used by the serial/MSP protocol implemented in `Protocol.cpp`. The protocol can expose sensor, RC, attitude, motor, and configuration data and accepts supported calibration/configuration commands.

`EEPROM.cpp` stores global settings and the active tuning profile. Startup validates the stored profile and refreshes constants when the firmware image changes. Changes to PID values, RC rates, trims, and other EEPROM-backed settings therefore survive reset, subject to the existing MultiWii checksum and profile logic.

## Module map

| Module | Responsibility |
| --- | --- |
| `MultiWii.cpp` | Scheduler, setup, RC command handling, flight modes, PID loop, arming |
| `NRF24_RX.cpp` | RF24 setup, packet receive, RC conversion, acknowledgement telemetry |
| `Sensors.cpp` / `IMU.cpp` | Sensor drivers, calibration, filtering, attitude estimation |
| `Output.cpp` | PWM timer setup, Quad-X motor mixing, motor output |
| `EEPROM.cpp` | Persistent global and profile settings |
| `Protocol.cpp` | MultiWii Serial Protocol messages |
| `Alarms.cpp` | Buzzer and alarm state handling |
| `GPS.cpp` / `LCD.cpp` | Optional navigation and display support retained from MultiWii |

The compile-time macros are the primary integration boundary. Adding hardware should begin by updating `config.h` and the corresponding driver selection, then validating the derived definitions in `def.h` and the output/channel mappings before flight testing.
