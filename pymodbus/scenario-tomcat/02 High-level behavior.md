Below is a **concise, implementation-ready roadmap** for your Modbus/OT mini-system using **pymodbus 3.11.3**. You asked for structure and file list (no code). I’ve optimized the design so it matches real ICS behavior while staying simple to run and capture in Wireshark.

---

# 0) High-level behavior you want

- A **Sensor Generator** produces _temperature_ and _pressure_ values and **sends** them to a **PLC**.
    
- The **PLC** holds the plant state in registers, runs **logic** (e.g., thresholds), and **commands** the **Valves & Doors**.
    
- “PLC sends to valves/doors” = PLC acts as a **Modbus client** toward a **Valves/Doors device** (a second Modbus server) and writes its commands there.
    

This mirrors the real world:

- Sensors → PLC (ingest path)
    
- PLC logic → Actuators (command path)
    
- Optional HMI/SCADA just reads everything.
    

---

# 1) Components & roles

1. **sensor_sim.py**
    
    - Role: **Producer** of temperature/pressure.
        
    - Protocol: **Modbus client → PLC server** (writes _ingress_ holding registers with FC06/FC16).
        
    - Why HR? In Modbus, **Input Registers are read-only** to clients; so the PLC must expose **Holding Registers** as the _ingress buffer_ for external writers.
        
2. **plc_server.py**
    
    - Role: **Modbus server** for inbound writes/reads (from sensor_sim and HMI), **plus PLC logic**, **plus Modbus client** to actuators.
        
    - Responsibilities:
        
        - Maintain a **datastore** (co/di/hr/ir).
            
        - **Ingest** sensor values from HR (written by `sensor_sim`).
            
        - **Compute logic** (thresholds, interlocks, timing).
            
        - **Expose state** (mirrors into IR/HR for readback).
            
        - **Issue commands** to actuators by acting as a **client** and writing to the **valve/door server** (FC05/06/0F/10 as needed).
            
    - This file runs asyncio server and a periodic task for logic + actuator writes.
        
3. **actuator_server.py**
    
    - Role: **Valves & Doors device**, a separate **Modbus server** that the PLC writes to.
        
    - Datapoints:
        
        - **Coils**: command targets (e.g., `VALVE1_OPEN`, `DOOR3_LOCK`).
            
        - **Discrete Inputs**: feedback flags (e.g., limit switch reached) — set internally to simulate movement/latency.
            
        - **Holding Registers**: optional command/setpoint values, states, timestamps.
            
    - May simulate **actuation delay** and **feedback** (e.g., coil goes 1 → after 500 ms DI shows “open”).
        
4. **hmi_monitor.py** (optional but useful)
    
    - Role: **Read-only dashboard** (Modbus client) to poll the PLC & Actuator servers, print current temps/pressures/door & valve states.
        
    - Generates additional clean traffic for Wireshark and helps validate logic.
        
5. **addresses.py**
    
    - Role: **Central address map & constants** (no code logic here).
        
    - Keeps all register/coil addresses consistent across files.
        
    - Example sections:
        
        - PLC ingress HR map (e.g., `HR_TEMP = 0`, `HR_PRESS = 1`)
            
        - PLC coils/registers for derived state/alarms
            
        - Actuator server coil/register map (e.g., `CO_VALVE1_CMD = 0`, `CO_DOOR1_LOCK = 10`)
            
        - Threshold constants
            
6. **config.yaml**
    
    - Role: **Runtime configuration** (IPs/ports, thresholds, polling rates, deadbands, actuator latencies).
        
    - Allows you to tweak behavior without code edits.
        
    - Keys you’ll want:
        
        - `plc_server: { host: 127.0.0.1, port: 5020 }`
            
        - `actuator_server: { host: 127.0.0.1, port: 5021 }`
            
        - `sensor_sim: { write_interval_ms: 1000, temp_profile: ..., pressure_profile: ... }`
            
        - `logic: { temp_high: 60.0, pressure_high: 10.0, debounce_ms: 300 }`
            
        - `actuator: { motion_delay_ms: 500 }`
            
7. **logging.ini** (or simple Python logging setup)
    
    - Role: uniform, timestamped logs across all processes (INFO for state, DEBUG for payloads).
        
8. **requirements.txt**
    
    - Pin `pymodbus==3.11.3` and any small dependencies (PyYAML for config).
        
9. **README.md**
    
    - Role: quick run steps, Wireshark filters, address map table.
        
10. _(optional)_ **docker-compose.yml**
    

- Role: bring up the three processes in one command; convenient for repeatable labs/captures.
    

---

# 2) Address map (proposed)

Keep addressing **0-based** in code; list below shows logical groupings.

## PLC (Server on 127.0.0.1:5020)

- **Holding Registers (writeable ingress from sensors)**
    
    - `HR_TEMP = 0` (float16/float32 via two 16-bit regs if you want real decimals; else scaled int)
        
    - `HR_PRESS = 2` (same note as above)
        
    - `HR_FLAGS = 10` (bitfield for sensor health, optional)
        
- **Input Registers (read-only derived state mirrored by PLC logic)**
    
    - `IR_TEMP_FILTERED = 0`
        
    - `IR_PRESS_FILTERED = 2`
        
    - `IR_ALARMS = 10` (bitfield summary)
        
- **Coils (PLC outputs for visualization / HMI)**
    
    - `CO_ALARM_HIGH_TEMP = 0`
        
    - `CO_ALARM_HIGH_PRESS = 1`
        
    - `CO_PERMISSIVE_OK = 2`
        
- **Holding Registers (PLC logic/commands to actuators, also mirrored here for HMI)**
    
    - `HR_CMD_VALVE1 = 100` (0=close,1=open)
        
    - `HR_CMD_VALVE2 = 101`
        
    - `HR_CMD_DOOR1 = 110` (0=locked,1=unlocked)
        
    - `HR_CMD_DOOR2 = 111`
        
    - `HR_CMD_DOOR3 = 112`
        

_(Note: The PLC will also **write these commands to the actuator server** as coils/registers — these HRs just mirror what PLC decided for easy HMI reads.)_

## Actuator Server (Server on 127.0.0.1:5021)

- **Coils (command targets written by PLC client)**
    
    - `CO_VALVE1_OPEN = 0`
        
    - `CO_VALVE2_OPEN = 1`
        
    - `CO_DOOR1_UNLOCK = 10`
        
    - `CO_DOOR2_UNLOCK = 11`
        
    - `CO_DOOR3_UNLOCK = 12`
        
- **Discrete Inputs (feedback set by actuator server’s internal simulation)**
    
    - `DI_VALVE1_OPENED = 0`
        
    - `DI_VALVE2_OPENED = 1`
        
    - `DI_DOOR1_OPEN = 10`
        
    - `DI_DOOR2_OPEN = 11`
        
    - `DI_DOOR3_OPEN = 12`
        
- **Holding Registers (optional telemetry/state)**
    
    - `HR_LAST_CMD_TS = 200`
        
    - `HR_FAULT_CODE = 201`
        

---

# 3) Data types & encoding

- **Choose one encoding and stick to it**:
    
    - **Scaled integers** (e.g., `temp_celsius * 10` as an int) → simple, portable.
        
    - OR **32-bit floats** across **two HRs** (define endianness: big-endian or little-endian word order).
        
- Document your choice in `README.md` and `addresses.py`.
    

---

# 4) Message flows

### A) Sensor → PLC ingest

1. `sensor_sim.py` computes next `temp`, `press`.
    
2. Writes **HR_TEMP/HR_PRESS** on **PLC** via FC06/FC16 at **5020**.
    
3. PLC logic task reads ingress HRs, filters/debounces, updates IRs/Coils.
    

### B) PLC logic → Actuators

1. PLC evaluates conditions (e.g., `temp > T_HIGH and press < P_HIGH`).
    
2. Sets internal HR command mirrors (e.g., `HR_CMD_VALVE1=1`).
    
3. As **Modbus client**, PLC **writes** actuator coils on **actuator_server (5021)**:
    
    - `CO_VALVE1_OPEN = 1`, etc.
        
4. Actuator server simulates motion delay and sets DI feedback bits accordingly.
    

### C) HMI monitor (optional)

- Polls both servers, prints a compact live view.
    

---

# 5) Files and what each contains (summary)

- `sensor_sim.py`
    
    - Reads `config.yaml` profiles (sawtooth, random walk, fixed set).
        
    - Writes HRs to PLC every N ms.
        
    - Handles reconnects; logs values written.
        
- `plc_server.py`
    
    - Modbus **server** on 5020 with datastore (co/di/hr/ir).
        
    - Periodic **logic task**:
        
        - ingest from HR_TEMP/HR_PRESS
            
        - filter/debounce
            
        - set alarms/permissives (coils, IR/HR)
            
        - as **client**, write actuator commands to `actuator_server`.
            
    - Clear logging of decisions (“VALVE1 OPEN due to temp=… pressure=…”).
        
- `actuator_server.py`
    
    - Modbus **server** on 5021 with its own datastore.
        
    - Periodic **simulation task**: update DI feedback with delay; optionally inject faults.
        
- `hmi_monitor.py` _(optional)_
    
    - Read-only poller of key PLC/Actuator points. One-screen text output.
        
- `addresses.py`
    
    - All constants: HR/IR/CO/DI for PLC + Actuator; bitfields; enum values.
        
- `config.yaml`
    
    - IPs/ports; thresholds; intervals; encoding; actuator delays; fault rates.
        
- `logging.ini` or built-in logging setup
    
    - Consistent, timestamped logs.
        
- `requirements.txt`
    
    - `pymodbus==3.11.3`, `PyYAML`, etc.
        
- `README.md`
    
    - Run order, Wireshark filter (`tcp.port == 5020 || tcp.port == 5021`), “Decode As → Modbus/TCP”, address map tables.
        
- _(optional)_ `docker-compose.yml`
    
    - Three services: plc, actuator, sensor (+ hmi). Healthchecks.
        

---

# 6) Test & capture plan (brief)

1. Start **actuator_server.py** (5021) → confirm listening.
    
2. Start **plc_server.py** (5020) → confirm listening.
    
3. Start **sensor_sim.py** → verify PLC HRs changing; PLC logs decisions.
    
4. Verify PLC writes to actuator coils; actuator DI flips after delay.
    
5. Start **hmi_monitor.py** (optional) to see live status.
    
6. Wireshark:
    
    - Capture on loopback.
        
    - Display filter: `tcp.port == 5020 || tcp.port == 5021` (or `modbus`).
        
    - Inspect FC 0x06/0x10 (sensor → PLC), FC 0x05/0x06 (PLC → actuator).
        

---

# 7) Notes that prevent common pitfalls

- **Clients cannot write Input Registers**. Use **Holding Registers** as the “ingress” from sensors into the PLC.
    
- Define **endianness** if you use 32-bit floats (and keep it the same everywhere).
    
- Keep **scan/poll intervals** modest (e.g., 250–1000 ms) for readable captures.
    
- Separate **ingress**, **logic**, and **command** clearly in the PLC; don’t mix roles.
    
- Add **bounds checking** & **deadbands** in logic to avoid output chatter.
    
- For realism, add **actuator motion delays** and **feedback DI** updates.
    

---

If you want, I can turn this into a ready-to-run folder skeleton (files with stubs, comments, and TODO blocks) so you can start filling in code where needed.