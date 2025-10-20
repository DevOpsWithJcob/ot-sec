Here’s a clean project tree you can start from. It separates **roles** (sensor → PLC → actuators) and keeps addresses/config centralized. No code included—just structure.

```txt
modbus-ics-lab/
├─ README.md                          # run steps, Wireshark filters, address map table
├─ requirements.txt                   # pymodbus==3.11.3, pyyaml, etc.
├─ config.yaml                        # IPs/ports, thresholds, polling rates, delays, encoding
├─ logging.ini                        # unified logging format/levels
├─ addresses.py                       # single source of truth for all Modbus addresses/constants
│
├─ plc/                               # PLC role: Modbus server + control logic + client to actuators
│  ├─ __init__.py
│  ├─ plc_server.py                   # Modbus/TCP server (5020) exposing ingress HR/IR/CO/DI
│  ├─ logic.py                        # evaluates temp/pressure, sets alarms & commands
│  ├─ actuator_client.py              # Modbus client used by PLC to write actuator server coils/regs
│  └─ state.py                        # optional: in-memory model (debounce, timers, latches)
│
├─ actuators/                         # Actuator device: Modbus server with coils/DI + motion simulation
│  ├─ __init__.py
│  ├─ actuator_server.py              # Modbus/TCP server (e.g., 5021) exposing coils/DI/HR
│  └─ simulator.py                    # simulates motion delay & feedback DI updates
│
├─ sensors/                           # Sensor generator: “writes into PLC ingress HRs”
│  ├─ __init__.py
│  ├─ sensor_sim.py                   # Modbus client that writes HR_TEMP/HR_PRESS to PLC
│  └─ profiles.py                     # value generators (random walk, ramps, sine, fixed)
│
├─ hmi/                               # Optional read-only monitor
│  ├─ __init__.py
│  └─ hmi_monitor.py                  # polls PLC & actuator server, prints compact status
│
├─ common/                            # Shared helpers (no business logic)
│  ├─ __init__.py
│  ├─ encoding.py                     # number encoding/endianness helpers (scaled int vs 32-bit float)
│  └─ utils.py                        # retry/backoff, time utils, logging bootstrap
│
├─ scripts/                           # Convenience launchers (optional)
│  ├─ run_all.sh                      # start actuators → plc → sensors → hmi (with sleeps)
│  └─ stop_all.sh
│
├─ docker/                            # Optional containerization
│  ├─ docker-compose.yml              # brings up actuator, plc, sensors, hmi
│  ├─ Dockerfile.plc
│  ├─ Dockerfile.actuator
│  ├─ Dockerfile.sensor
│  └─ Dockerfile.hmi
│
└─ tests/                             # Optional smoke tests (no heavy frameworks needed)
   ├─ test_addresses.py               # assert address constants don’t overlap, ranges valid
   └─ test_encoding.py                # verify float/int packing & endianness round-trips

```
### Notes on what maps where (to keep you consistent)

- **`addresses.py`**: define _all_ PLC ingress/derived/command points and actuator coils/DI in one place. Example groups you’ll want:
    
    - PLC ingress HR: `HR_TEMP`, `HR_PRESS`
        
    - PLC derived IR/CO: `IR_TEMP_FILTERED`, `CO_ALARM_HIGH_TEMP`, etc.
        
    - PLC command mirrors HR: `HR_CMD_VALVE1`, `HR_CMD_DOOR1`, …
        
    - Actuator coils/DI: `CO_VALVE1_OPEN`, `DI_VALVE1_OPENED`, …
        
- **`config.yaml`**:
    
    - `plc_server: { host: 127.0.0.1, port: 5020 }`
        
    - `actuator_server: { host: 127.0.0.1, port: 5021 }`
        
    - `sensor_sim: { interval_ms: 1000, profile: "ramp", encoding: "scaled_int" }`
        
    - `logic: { temp_high: 60.0, pressure_high: 10.0, debounce_ms: 300 }`
        
    - `actuator: { motion_delay_ms: 500 }`
        
    - `encoding: { type: "scaled_int", scale: 10, endianness: "big" }`
        
- **`encoding.py`**: one place to pack/unpack floats/ints consistently across all roles.
    

This layout keeps responsibilities clear:

- **sensors** write to PLC **holding registers** (ingress).
    
- **plc** exposes a **server**, runs **logic**, and uses a **client** to command **actuators**.
    
- **actuators** expose a **server** that reflects commands and simulates feedback.
    
- **hmi** is optional but useful to observe state and generate read traffic.