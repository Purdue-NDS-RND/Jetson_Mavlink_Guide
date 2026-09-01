# Jetson ↔ Pixhawk MAVLink Guide

A practical guide to connecting an NVIDIA Jetson companion computer to a Pixhawk running ArduPilot. It covers two ways to work with the same MAVLink telemetry and commands:

- **MAVROS** when your application is built with ROS 2
- **pymavlink** when you want direct, low-level access from Python

> [!CAUTION]
> Remove the propellers and keep the vehicle disarmed while testing. A working MAVLink connection can carry arm, mode-change, and actuator commands.

## What this connection provides

```text
ROS 2 nodes ↔ MAVROS ↔ MAVLink ↔ USB serial ↔ Pixhawk ↔ sensors and actuators
Python app   ↔ pymavlink ↔ MAVLink ↔ USB serial ↔ Pixhawk ↔ sensors and actuators
```

MAVROS translates MAVLink messages into ROS topics, services, and parameters. pymavlink exposes the MAVLink messages directly to Python. Both can receive telemetry and send commands, but they are different programming interfaces.

## Before you begin

This guide assumes:

- the Pixhawk is running ArduPilot;
- ROS 2 and MAVROS are already installed on the Jetson;
- the current shell has sourced the correct ROS 2 environment; and
- the USB connection is configured for MAVLink at the baud rate used below.

The examples use `/dev/ttyACM0` at `115200` baud. Your device name or configured baud rate may be different.

## 1. Connect the hardware

Connect the Jetson to the Pixhawk with a data-capable USB cable:

```text
Jetson USB-A/USB-C ── USB cable ── Pixhawk USB-C
```

Linux will normally create a device such as `/dev/ttyACM0` or `/dev/ttyUSB0`.

## 2. Identify the Pixhawk serial device

List stable serial-device links before and after plugging in the Pixhawk:

```bash
ls -l /dev/serial/by-id/
```

If that directory does not exist, compare the following output before and after connecting the cable:

```bash
ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

The newly appearing device is usually the Pixhawk. For example:

| Device | Typical meaning |
| --- | --- |
| `/dev/ttyACM0` | Pixhawk USB MAVLink interface |
| `/dev/ttyAMA0` | Jetson internal UART; usually not the USB Pixhawk |

Unplugging the Pixhawk should make its device disappear. A path under `/dev/serial/by-id/` is preferable in scripts because it is less likely to change between reboots.

## 3. Fix serial permissions

Check the device owner and group:

```bash
ls -l /dev/ttyACM0
```

If MAVROS reports `Permission denied`, a temporary test is:

```bash
sudo chmod 666 /dev/ttyACM0
```

This permission is lost when the device is unplugged or the Jetson reboots. For a persistent fix, add your user to the serial-device group:

```bash
sudo usermod -aG dialout "$USER"
```

Log out and back in, or reboot, then confirm that `dialout` appears in:

```bash
groups
```

## 4. Launch MAVROS

Keep this terminal open so you can see connection and plugin messages:

```bash
ros2 launch mavros apm.launch fcu_url:=/dev/ttyACM0:115200
```

MAVROS also accepts the explicit serial URL form:

```bash
ros2 launch mavros apm.launch fcu_url:=serial:///dev/ttyACM0:115200
```

If the node exits with a GeographicLib/geoid dataset error, install the dataset required by your MAVROS package before continuing. This is separate from serial connectivity.

## 5. Verify the MAVROS connection

In a second terminal, source ROS 2 again and inspect the vehicle state:

```bash
ros2 topic echo /mavros/state --once
```

A healthy link includes:

```yaml
connected: true
armed: false
mode: STABILIZE
```

The exact mode can differ. The important first signal is `connected: true`, which means MAVROS is receiving the flight controller's MAVLink heartbeat.

Next, confirm that MAVROS interfaces are available:

```bash
ros2 topic list | grep '^/mavros/'
ros2 service list | grep '^/mavros/'
```

You can safely observe telemetry without commanding the vehicle:

```bash
ros2 topic echo /mavros/battery --once
ros2 topic echo /mavros/global_position/global --once
```

Some topics will not publish until the associated sensor is healthy. For example, global position normally requires a GPS fix.

## What to expect after MAVROS connects

`connected: true` confirms the communications link; it does **not** mean the vehicle is ready to fly.

After connection, expect the following:

1. MAVROS begins converting incoming MAVLink telemetry into ROS 2 messages.
2. `/mavros/state` updates the armed state, flight mode, and connection status.
3. Sensor plugins publish data only when the flight controller is producing the corresponding messages.
4. MAVROS services become available for operations such as changing mode, arming, and mission handling.
5. ArduPilot still enforces its own pre-arm checks, failsafes, EKF health, GPS requirements, and mode rules.

Before attempting autonomous control, verify all of the following:

- the link remains connected for several minutes;
- the reported mode and sensor values are reasonable;
- the vehicle is physically secured with propellers removed;
- ArduPilot reports no unresolved pre-arm failures; and
- your control node handles disconnects, stale telemetry, rejected commands, and shutdown safely.

Changing to `GUIDED` mode or calling an arm service can be rejected even with a healthy connection. Treat that rejection as a flight-controller safety result to investigate, not as proof that MAVROS is broken.

## Using pymavlink instead

[pymavlink](https://mavlink.io/en/mavgen_python/) is a low-level Python MAVLink library. Unlike MAVROS, it does not create ROS topics or services. Your program receives MAVLink messages, reads their fields, sends commands, waits for acknowledgements, and manages timeouts itself.

Install it in a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip pymavlink
```

With MAVROS stopped, try this read-only connection test:

```python
from pymavlink import mavutil

connection = mavutil.mavlink_connection(
    "/dev/ttyACM0",
    baud=115200,
    source_system=255,
)

print("Waiting for a flight-controller heartbeat...")
heartbeat = connection.wait_heartbeat(timeout=10)
if heartbeat is None:
    raise TimeoutError("No flight-controller heartbeat received within 10 seconds")

print(
    f"Connected to system {connection.target_system}, "
    f"component {connection.target_component}"
)

while True:
    message = connection.recv_match(
        type=["HEARTBEAT", "SYS_STATUS", "GPS_RAW_INT"],
        blocking=True,
        timeout=2,
    )
    if message is None:
        print("No matching telemetry received in the last 2 seconds")
        continue
    print(message)
```

When it works, you should first see the detected system and component IDs, followed by decoded MAVLink messages. Message rates vary, and a message such as `GPS_RAW_INT` may be missing or show an invalid fix until GPS is available.

### What pymavlink requires from your application

Compared with MAVROS, expect to write more communication logic yourself:

- wait for and monitor heartbeats;
- select the correct target system and component;
- filter and validate incoming messages;
- request message intervals when the default stream is insufficient;
- send commands with the correct units and coordinate frames;
- check `COMMAND_ACK` results instead of assuming a command succeeded; and
- handle timeouts, reconnects, duplicate messages, and safe shutdown.

> [!IMPORTANT]
> Do not run MAVROS and a pymavlink script against `/dev/ttyACM0` at the same time. They will compete for the same serial stream. Stop one before starting the other, or configure a MAVLink router/UDP forwarding setup and give each client its own endpoint.

## SITL versus real hardware

| Environment | MAVROS FCU URL | pymavlink connection |
| --- | --- | --- |
| ArduPilot SITL | `udp://:14550@` | `udpin:localhost:14550` |
| Pixhawk over USB | `/dev/ttyACM0:115200` | `/dev/ttyACM0`, `baud=115200` |

Example MAVROS SITL launch:

```bash
ros2 launch mavros apm.launch fcu_url:=udp://:14550@
```

Only one program can bind a particular UDP port. If SITL, MAVROS, a ground station, and pymavlink must run together, configure separate forwarded endpoints with MAVProxy or a MAVLink router.

## Troubleshooting

| Symptom | Likely cause | Check |
| --- | --- | --- |
| Serial device never appears | power-only cable, bad port, or USB issue | Try another data cable and inspect `dmesg --follow` while reconnecting |
| `Permission denied` | user lacks serial-device access | Check `ls -l` and membership in `dialout` |
| `Device or resource busy` | another process owns the port | Stop MAVROS, pymavlink, a serial console, or modem-manager client using it |
| MAVROS runs but `connected: false` | wrong port, baud, or no heartbeat | Confirm the device, ArduPilot serial settings, and MAVROS console output |
| Connected but no GPS topic data | GPS has no fix or stream is absent | Check GPS status in the ground station and outdoors with clear sky |
| Arm/mode command is rejected | pre-arm, EKF, safety-switch, or mode requirement | Read the ArduPilot status text; do not bypass the safety check blindly |
| pymavlink waits forever | no heartbeat or wrong endpoint | Use a finite timeout and confirm that no other program owns the link |

## Recommended next steps

1. Keep the vehicle disarmed and confirm stable telemetry.
2. Record which topics or MAVLink messages your application actually needs.
3. Test command logic in ArduPilot SITL before connecting actuators.
4. Add explicit timeout, acknowledgement, and failsafe behavior.
5. Perform restrained, propeller-free hardware tests before any flight test.

## References

- [MAVROS ROS 2 documentation](https://docs.ros.org/en/jazzy/p/mavros/)
- [pymavlink usage guide](https://mavlink.io/en/mavgen_python/)
- [ArduPilot MAVLink basics](https://ardupilot.org/dev/docs/mavlink-basics.html)
- [MAVLink heartbeat protocol](https://mavlink.io/en/services/heartbeat.html)
