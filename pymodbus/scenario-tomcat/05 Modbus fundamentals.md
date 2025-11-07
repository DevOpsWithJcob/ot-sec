Below is a **teaching-ready document** — you can directly use this to explain **Modbus fundamentals, architecture, and operation** to students, engineers, or junior cybersecurity/OT professionals.

It’s written in clear, structured form — suitable for slides, classroom training, or internal documentation.

---

# 📘 **Understanding Modbus: The Industrial Communication Protocol**

---

## 🔹 1. What is Modbus?

**Modbus** is a _communication protocol_ used in **industrial automation systems** to allow devices to exchange data.

- Developed by **Modicon (Schneider Electric)** in **1979**
    
- **Open, royalty-free**, and extremely simple
    
- Still used worldwide in **PLC–HMI**, **SCADA**, **sensors**, and **industrial control systems (ICS)**
    

Modbus defines **how data is structured, addressed, and transmitted** between industrial devices such as:

- **PLCs (Programmable Logic Controllers)**
    
- **Sensors and actuators**
    
- **HMIs (Human-Machine Interfaces)**
    
- **SCADA systems**
    

---

## 🔹 2. Why Modbus Still Matters

Despite its age, Modbus remains the **de facto standard** for data exchange in automation because:

|Feature|Benefit|
|---|---|
|Simplicity|Easy to implement in embedded systems|
|Open standard|No licensing required|
|Multi-vendor support|Works across devices from different manufacturers|
|Ubiquity|Supported by almost all industrial control software|
|Compatibility|Works over serial (RS-485) and Ethernet networks|

---

## 🔹 3. Modbus Roles: **Client and Server**

Like many communication protocols, Modbus defines two primary roles:

|Role|Other Name|Typical Device|Function|
|---|---|---|---|
|**Client**|_Master_|SCADA, HMI, data logger|Initiates communication (sends requests)|
|**Server**|_Slave_|PLC, RTU, sensor, actuator|Responds to requests|

### Example

```
SCADA (Client)  →  PLC (Server)
"Give me temperature (register 40001)"
PLC  →  SCADA
"Temperature = 73.4°C"
```

The **client always starts the conversation**, and the **server responds**.

---

## 🔹 4. Variants of Modbus

|Type|Transport Layer|Medium|Typical Use|
|---|---|---|---|
|**Modbus RTU**|Serial (RS-485/RS-232)|Wired|PLC to field sensors (legacy systems)|
|**Modbus ASCII**|Serial (ASCII characters)|Wired|Older human-readable form|
|**Modbus/TCP**|TCP/IP (Ethernet)|Network|Modern SCADA, HMI, and PLCs|

✅ **Modbus/TCP** (what you’re simulating in Python) is the Ethernet-based version of Modbus RTU, encapsulated in TCP packets.

---

## 🔹 5. How Modbus/TCP Works

Each message (called an **ADU – Application Data Unit**) consists of:

|Field|Description|
|---|---|
|**Transaction ID**|Matches requests/responses|
|**Protocol ID**|Always `0` for Modbus|
|**Length**|Number of bytes in message|
|**Unit ID**|Identifies device (important for gateways)|
|**Function Code**|What operation to perform (read/write)|
|**Data**|The actual payload (addresses, values, etc.)|

**Protocol stack example:**

```
Ethernet Frame
└── IP Header
    └── TCP Header (Port 502)
        └── Modbus/TCP ADU
             ├─ MBAP Header
             └─ Modbus PDU (Function Code + Data)
```

---

## 🔹 6. Function Codes — The Core of Modbus

Function Codes define the **action**.  
Each request uses one function code; the server responds with data or an exception.

|Code|Operation|Data Type|Access|Description|
|---|---|---|---|---|
|**01**|Read Coils|Discrete Output (1-bit)|R/W|Read status of ON/OFF devices|
|**02**|Read Discrete Inputs|Discrete Input (1-bit)|R|Read external digital input|
|**03**|Read Holding Registers|16-bit Registers|R/W|Read analog/output data|
|**04**|Read Input Registers|16-bit Registers|R|Read sensor inputs|
|**05**|Write Single Coil|1-bit|W|Turn one output ON/OFF|
|**06**|Write Single Register|16-bit|W|Write to one register|
|**15 (0x0F)**|Write Multiple Coils|1-bit|W|Write group of coils|
|**16 (0x10)**|Write Multiple Registers|16-bit|W|Write group of registers|

---

## 🔹 7. The Four Modbus Memory Areas

Modbus defines _logical tables_ of data:

|Data Type|Typical Address Range|Access|Description|
|---|---|---|---|
|**Coils (0x)**|00001–09999|Read/Write|Output bits (e.g., valve ON/OFF)|
|**Discrete Inputs (1x)**|10001–19999|Read-only|Digital inputs (limit switches)|
|**Input Registers (3x)**|30001–39999|Read-only|Sensor readings (temperature, pressure)|
|**Holding Registers (4x)**|40001–49999|Read/Write|Internal variables or setpoints|

> Note: The “1x / 3x / 4x” numbering is logical. In software, we often use **0-based addresses** (i.e., 0 instead of 40001).

---

## 🔹 8. How Communication Happens (Step-by-Step)

### Example: SCADA Reads Pressure from PLC

1. **SCADA (Client)** sends request:  
    → Function code `0x04` (Read Input Registers)  
    → Address: 30001 (pressure sensor)  
    → Count: 1
    
2. **PLC (Server)** reads the value from its memory (e.g., 8.52 bar).
    
3. **PLC responds** with:  
    → Function code `0x04`  
    → Value bytes `[0x08, 0x52]`
    
4. **SCADA decodes** bytes into engineering units and displays the pressure.
    

---

## 🔹 9. Example Data Flow in a Plant

```
      ┌──────────────────────────────────────┐
      │              SCADA (Client)          │
      │  - Reads sensors                     │
      │  - Writes control commands           │
      └───────────────┬──────────────────────┘
                      │ Modbus/TCP (Port 502)
                      │
           ┌──────────┴──────────┐
           │                     │
     ┌────────────┐        ┌────────────┐
     │  PLC #1    │        │  PLC #2    │
     │  (Server)  │        │  (Server)  │
     │ Reads sensors,       │ Controls motors, valves │
     │ runs logic, etc.     │                         │
     └────────────┘        └────────────┘
```

Each PLC acts as a **server**, exposing its process data via Modbus registers.

---

## 🔹 10. Modbus in ICS Networks (Purdue Model)

| Level                     | Description             | Protocols             | Example                          |
| ------------------------- | ----------------------- | --------------------- | -------------------------------- |
| **Level 3 (Control)**     | SCADA / Historians      | Modbus/TCP, OPC, DNP3 | SCADA collecting process data    |
| **Level 2 (Supervisory)** | HMI / Operator stations | Modbus/TCP            | Operator panel controlling lines |
| **Level 1 (Control)**     | PLCs, RTUs, IEDs        | Modbus RTU/TCP        | PLC controlling valves           |
| **Level 0 (Field)**       | Sensors, actuators      | Electrical signals    | 4–20mA, discrete inputs          |

Modbus connects Levels 0–2, where **real-time control and monitoring** occur.

---

## 🔹 11. Security Limitations

Modbus was designed **long before cybersecurity was a concern**.  
It assumes a **trusted network**.

|Weakness|Explanation|
|---|---|
|❌ No encryption|Data sent in clear text|
|❌ No authentication|Any client can talk to any server|
|❌ No session integrity|Easy to spoof or replay commands|
|❌ No authorization|Cannot restrict which client writes values|

That’s why industrial networks use **network segmentation**, **firewalls**, and **monitoring tools** (e.g., Zeek, Suricata) to detect anomalies.

---

## 🔹 12. Example Use Case

Let’s revisit your earlier example:

> Two valves and three doors that open/close based on temperature and pressure.

|Component|Role|Function|
|---|---|---|
|**PLC**|Modbus Server|Reads temperature/pressure, decides control actions|
|**Sensor**|Modbus Client|Sends readings to PLC (writes holding registers)|
|**Valves & Doors**|Modbus Server|Receive open/close commands from PLC|
|**HMI**|Modbus Client|Reads system status and alarms|

**Flow:**

```
Sensor → PLC → Valves/Doors
   ↑         ↓
   └─── HMI reads everything
```

---

## 🔹 13. Capturing Modbus Traffic (Wireshark Example)

To analyze Modbus/TCP packets:

1. Start Wireshark → Capture on loopback or Ethernet.
    
2. Filter:
    
    ```
    tcp.port == 502
    ```
    
    or for lab:
    
    ```
    tcp.port == 5020 || tcp.port == 5021
    ```
    
3. Right-click → _Decode As → Modbus/TCP_
    
4. You’ll see:
    
    - Function codes (0x01, 0x03, 0x04, 0x06, etc.)
        
    - Register addresses
        
    - Data values
        

---

## 🔹 14. Example Packet (Read Holding Register)

**Client Request**

|Field|Value|
|---|---|
|Function Code|0x03|
|Start Address|0x006B (register 107)|
|Quantity|3|

**Server Response**

| Function Code | 0x03 |  
| Byte Count | 6 |  
| Data | 0x000A, 0x0102, 0x0001 |

→ Meaning: Registers 107–109 = [10, 258, 1]

---

## 🔹 15. Advantages & Drawbacks

|Advantages|Drawbacks|
|---|---|
|Simple & open|No security|
|Lightweight|Limited data size|
|Widely supported|No real-time sync|
|Easy to debug|No discovery (static addressing)|

---

## 🔹 16. Modern Alternatives

|Protocol|Key Feature|Security|
|---|---|---|
|**OPC UA**|Object-oriented, encrypted|✅ TLS|
|**DNP3**|Used in power/energy systems|Partial|
|**EtherNet/IP / PROFINET**|Real-time control|Medium|
|**MQTT / AMQP**|SCADA/IIoT data layer|With TLS|

Modbus still dominates **Level 1–2** devices for simplicity, but modern plants often **gateway** it into OPC UA or MQTT for secure higher-layer integration.

---

## 🔹 17. Quick Recap

|Concept|Definition|
|---|---|
|**Modbus Server**|Device exposing process data (e.g., PLC)|
|**Modbus Client**|Device reading/writing that data (e.g., SCADA)|
|**Registers**|Data storage (Coils, Discrete Inputs, Input & Holding Registers)|
|**Function Codes**|Define read/write operations|
|**Modbus RTU/TCP**|Serial vs. Ethernet transports|
|**Security**|None — must be isolated or monitored|

---

## 🔹 18. Simple Teaching Diagram

```
         ┌────────────────────────────┐
         │         SCADA (Client)     │
         │   FC=03 Read HR            │
         └────────────┬───────────────┘
                      │ TCP 502
         ┌────────────┴───────────────┐
         │          PLC (Server)      │
         │  HR40001 = Temp, HR40002=Pressure │
         └─────┬─────────────┬────────┘
               │              │
      ┌────────┘              └────────┐
 ┌────────────┐              ┌────────────┐
 │  Sensor    │              │  Valve     │
 │ (writes HR)│              │ (reads Coil)│
 └────────────┘              └────────────┘
```

---

## 🔹 19. Summary for Teaching

✅ **Key takeaway:**

> Modbus is a simple, deterministic protocol that underpins a huge portion of industrial communication.  
> Understanding it helps you analyze, simulate, and secure real OT systems.

**When teaching:**

- Start from the physical (sensor → PLC → HMI)
    
- Explain Modbus’ role as a _messenger_ carrying data
    
- Use Wireshark or Python simulations for live demonstration
    
- Emphasize its lack of security and the need for segmentation
    

---

## 🔹 20. Suggested Practical Demo (for Class)

|Step|Description|Tool|
|---|---|---|
|1|Run `modbus_server.py` (PLC simulation)|Python/pymodbus|
|2|Run `modbus_client.py` (HMI)|Python|
|3|Observe packets in Wireshark|Filter `tcp.port == 5020`|
|4|Change register values in client|See server response|
|5|Decode packets manually|MBAP header + PDU structure|
|6|Discuss attack vectors (replay, unauthorized write)|Zeek or Suricata rules|

---

Would you like me to format this as a **Word or PDF document** (with section titles, diagrams, and example screenshots placeholders) so you can distribute it as training material?