# BrushQuadX RF24 RX Drone

Arduino firmware for a brushed Quad-X flight controller with an nRF24L01+ radio receiver. The project is based on the MultiWii control loop and is configured for an ATmega328P/Pro Mini-class board.

![Drone](/assets/images/drone.png)

## Current configuration

- Frame: Quad-X, four brushed motors
- Sensors: MPU6050 IMU and BMP085 barometer over I2C
- Receiver: nRF24L01+ at 250 kbps, with automatic acknowledgements disabled
- Control loop target: 2,800 microseconds
- Motor throttle range: 1,000-2,000; minimum running throttle is 1,050 and stop command is 1,000
- Serial/MSP port: 115,200 baud
- Safety and status: arm only when level, buzzer on D8, status LED on D13

## Hardware connections

| Function | Arduino pin |
| --- | --- |
| Motor 1 | D9 |
| Motor 2 | D6 |
| Motor 3 | D5 |
| Motor 4 | D3 |
| nRF24 CE | D7 |
| nRF24 CSN | D10 |
| nRF24 MOSI / MISO / SCK | D11 / D12 / D13 |
| I2C SDA / SCL | A4 / A5 |
| Buzzer | D8 |

The motor order and Quad-X mixing are defined in `config.h` and `Output.cpp`. The nRF24 CE and CSN pins are constructed in `NRF24_RX.cpp`.

## Radio protocol

The receiver listens on pipe address `0xE8E8F0F0E1` and accepts this seven-byte payload:

```text
throttle, yaw, pitch, roll, AUX1, AUX2, switches
```

The first four values are bytes in the range 0-255 and are converted to 1,000-2,000 microsecond RC values using an inverted mapping. In the current implementation, packet `yaw` feeds the roll channel and packet `roll` feeds the yaw channel; this is intentional for the present transmitter wiring and should be changed in `NRF24_RX.cpp` if the transmitter protocol changes.

When no packet has been received for one second, the receiver resets the packet to centered sticks with zero auxiliary switches. The generic MultiWii timed landing failsafe is present in the source, but `FAILSAFE` is commented out in the active configuration, so this one-second neutral-input behavior is the radio-loss behavior currently enabled.

The receiver sends an acknowledgement payload containing attitude, estimated altitude, status flags, and currently hard-coded latitude/longitude values. The latitude and longitude fields are placeholders; GPS is not enabled in the active build.

## Build and use

Open `BrushQuadX_RF24_RXDrone/BrushQuadX_RF24_RXDrone.ino` in the Arduino IDE, select the target ATmega328P board and its serial port.

![Arduino Settings](/assets/images/arduino-settings.png)

Compile and upload the sketch. Verify the motor direction/order without propellers before flight.

Configuration and tuning are primarily in `BrushQuadX_RF24_RXDrone/config.h`. Persistent MultiWii settings are loaded from EEPROM during startup. The main control path is:

```text
nRF24 packet -> RC channel conversion -> arming/failsafe checks
			 -> IMU update -> PID controller -> Quad-X mixer -> motor PWM
```

## Repository layout

- `BrushQuadX_RF24_RXDrone.ino`: Arduino entry translation unit and project notes
- `MultiWii.cpp`: setup, scheduler, RC handling, IMU/PID control loop, arming, and mode logic
- `NRF24_RX.cpp/.h`: nRF24 transport and packet conversion
- `Sensors.cpp`, `IMU.cpp`, `GPS.cpp`: sensor drivers and estimation/navigation support
- `Output.cpp`: motor PWM setup, output scaling, and Quad-X mixing
- `EEPROM.cpp`: persistent configuration storage
- `Protocol.cpp`: serial MultiWii Serial Protocol (MSP) handling
- `config.h`, `def.h`, `types.h`: compile-time options, board definitions, and shared data types

See [ARCHITECTURE.md](ARCHITECTURE.md) for the detailed runtime and module design.

## Credits

- Original brushed-drone software and project lineage: [iforce2d](https://www.youtube.com/@iforce2d)
- Reference implementation and video: [Max Imagination](https://youtu.be/Sa6EslOHsI0?si=lMlFbbdl9P2VlySP)
- Source files and related project materials: [Max Imagination Google Drive folder](https://drive.google.com/drive/folders/1RpVh6NTzbtIL2lJVP0EqVGJy4QKX_Syh)
- Earlier brushed-drone adaptation and tutorial material: [ELECTRONOOBS tutorial](http://www.electronoobs.com/eng_arduino_tut117.php)
- Flight-control foundation: [MultiWii](https://www.multiwii.com/)
- Current repository adaptation: JohnMVSantos, 2026
