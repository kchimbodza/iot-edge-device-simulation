# IoT Edge Device Simulation

A Python-based Constrained Device Application (CDA) simulating real-world IoT 
edge device behavior including sensor data generation, actuator control, and 
secure cloud communication.

It is the edge half of a two-part system. The gateway half, the Java Gateway Device Application (GDA), is
[iot-gateway-cloud-bridge](https://github.com/kchimbodza/iot-gateway-cloud-bridge).
Edge device → gateway → cloud: this CDA reads (emulated) sensors and publishes them over MQTT, the GDA checks the data
and forwards it to the Ubidots cloud, and commands come back down to this device.

## At a glance

| | |
|---|---|
| **What it does** | Reads temperature, humidity, pressure and pitch from a Sense HAT emulator, sends them and its own CPU/memory use to the gateway, and runs the commands it gets back (humidifier, LED matrix). The joystick and tilt move a dot on the 8×8 LED matrix. |
| **Protocols** | MQTT over TLS (port 8883), CoAP client and server |
| **Tech** | Python 3, paho-mqtt, aiocoap / CoAPthon3, APScheduler, psutil, Sense HAT emulator (sense-emu, pisense) |
| **Tests** | 122 tests: 52 unit, 70 integration |

## Architecture

<p align="center"><img src="docs/architecture.svg" width="100%" alt="Full edge-to-cloud system with this repo, the edge device (CDA, Python), highlighted. Telemetry path: a Sense HAT emulator produces temperature, humidity, pressure and pitch readings; sensor and actuator managers poll them with APScheduler; the CDA DeviceDataManager publishes JSON over MQTT with TLS to a local broker on port 8883; the GDA DeviceDataManager receives it and checks the humidity rule (outside 30 to 50 percent for 10 seconds sends a humidifier command); the GDA CloudClientConnector publishes to Ubidots over TLS. Commands go back the other way: humidifier commands and the Ubidots LED switch travel from the GDA through the broker to the CDA, which drives the emulated humidifier, HVAC and LED matrix. The GDA also runs a CoAP server, tested on its own."></p>

1. **Sense:** `SensorAdapterManager` polls the Sense HAT emulator (or the built-in simulators) every 2 s using APScheduler.
2. **Send:** `DeviceDataManager` turns each reading into JSON and publishes it to its own MQTT topic on the local broker, over TLS. CPU and memory use are sent the same way.
3. **Act:** it subscribes to the actuator-command topic. Commands from the gateway (humidifier on/off, LED on/off) go to `ActuatorAdapterManager`, which drives the emulated device and sends a response back.
4. **LED demo (lab 12):** the joystick moves a dot on the LED matrix; the middle button switches to tilt mode, where the pitch sensor moves it. Each new position is published to the gateway, which shows it on Ubidots.

**Why this architecture**

| Choice | Why |
|---|---|
| The device only talks to the gateway | No cloud credentials on the device; the gateway decides what goes to the cloud. |
| MQTT publish/subscribe | The device just publishes to a topic; it does not need to know who listens or the gateway's address. |
| TLS with a CA certificate | Sensor data and commands are encrypted on the wire (checked with Wireshark, GDA commit `005524b`). |
| Sense HAT emulator and simulators | The whole system can be built and tested on a laptop, without a Raspberry Pi. |
| Managers behind one `DeviceDataManager` | Sensors, actuators and network clients are separate parts, each switched on or off in `config/PiotConfig.props`. |

## Project Structure

| Module | Description |
|--------|-------------|
| Sensor Simulation | Generates temperature, humidity, and pressure data |
| Actuator Control | Controls simulated devices based on thresholds |
| MQTT Client | Pub/sub messaging to gateway device |
| TLS/SSL | Certificate-based encrypted communication |
| System Monitor | Tracks CPU, memory, and device performance |

## Results

**Unit tests** (2026-11-13, Linux, Python 3.14.8, run from the repo root: `python -m unittest discover -s tests/unit -t .`):

| Run | Passed | Failed | Errors | Skipped |
|---|---|---|---|---|
| Before the fixes | 43 / 52 | 3 | 3 | 3 |
| **Now** | **46 / 52** | **0** | **0** | 6 |

What was fixed:

- **Hard-coded paths:** the default config and a test credential file pointed to absolute paths on my old laptop. The config path is now worked out from the code's own location, and the test credential path is relative to the repo.
- **Test order:** `ConfigUtil` is a Singleton, so the custom-config tests left their test config loaded for every later test. They now drop it when they finish. This also exposed a stale test that expected the plain MQTT port (1883) after the config moved to TLS (8883); it now accepts either.
- **Gateway files:** the 3 `test_DataIntegration` read tests need files the GDA writes to `/tmp/gda-data`. They now skip with a clear message when those files are missing, instead of erroring.

The 6 skips are those 3 tests plus 3 marked `@unittest.skip` in the code (one config test and two actuator simulator tests).

The 70 integration tests need an MQTT broker, the gateway or the Sense HAT emulator, so they were not run here.
There is no CI for this repo.

**Lab milestones** (from the commit history): sensor and actuator simulation (lab 3, "All unit and integration tests
passing"), Sense HAT emulation (4), JSON data (5), MQTT client (6), CoAP server (8) and client (9), commands from the
gateway with responses (10), end-to-end with the cloud (11), LED and pitch demo with new tests (12).

## What worked and what didn't

| | What happened | Notes |
|---|---|---|
| ✅ | Edge to cloud and back | Sensor data reaches the gateway over MQTT/TLS; humidifier and LED commands come back and are run on the emulated device (labs 10–12). |
| ✅ | Runs on a laptop | The Sense HAT emulator and simulators replace real hardware. |
| ✅ fixed | Portable unit tests | Hard-coded laptop paths and a Singleton leaking test config between tests made 5 unit tests fail on another machine; both are fixed, and all non-skipped unit tests now pass (see Results). |
| ❌ | Shared config between tests | `ConfigUtil` is shared, so 2 tests depend on which tests ran before them. |
| ⚠️ | CoAP built but not on the live path | The CoAP client and server were built and tested in labs 8–9, but `DeviceDataManager` sends data over MQTT only. |
| ⚠️ | Local temperature control is off | An on-device HVAC rule exists (`handleTempChangeOnDevice`), but it is switched off in the config; decisions are made by the gateway. |

## Technologies Used
`Python` `MQTT` `TLS/SSL` `CoAP` `Wireshark`

## Skills Demonstrated
- IoT sensor data simulation and actuator control
- Secure MQTT pub/sub messaging with TLS encryption
- Edge device performance monitoring
- Protocol analysis and verification with Wireshark

<details>
<summary><b>Run it yourself</b></summary>

Needs Python 3, an MQTT broker with TLS on port 8883, and the Sense HAT emulator (`sense-emu`) for emulator mode.
First set the certificate path (`certFile`) in `config/PiotConfig.props`.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m unittest discover -s tests/unit -t .     # unit tests, no broker needed
python -m programmingtheiot.cda.app.ConstrainedDeviceApp
```

Run the [GDA](https://github.com/kchimbodza/iot-gateway-cloud-bridge) as well for the full system.

</details>

## Built with

Built for a Connected Devices course, starting from the course's
[Programming the IoT](https://github.com/programming-the-iot) Python template by Andrew D. King (MIT licence, see `LICENSE`),
which supplies the interfaces, config utilities and test shells. I implemented the device logic in lab modules 2–12.

## Author

**Kudzayi Chimbodza** · [GitHub @kchimbodza](https://github.com/kchimbodza)
