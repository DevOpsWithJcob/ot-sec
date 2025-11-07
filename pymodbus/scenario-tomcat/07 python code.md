Awesome—this is a clean architecture. Here’s a straight-to-implementation guide that turns your pseudocode into a runnable, async Modbus playground using `pymodbus==3.11.3`. I’ll give you a minimal-but-complete code shape, exact APIs to use, and the order to bring it up.

---

# 1) Project layout

```
modbus-lab/
├─ addresses.py
├─ config.yaml
├─ logging.ini
├─ common/
│  ├─ __init__.py
│  ├─ encoding.py
│  └─ utils.py
├─ plc/
│  ├─ __init__.py
│  ├─ logic.py
│  ├─ state.py
│  ├─ actuator_client.py
│  └─ plc_server.py
├─ actuators/
│  ├─ __init__.py
│  ├─ actuator_server.py
│  └─ simulator.py
├─ sensors/
│  ├─ __init__.py
│  ├─ sensor_sim.py
│  └─ profiles.py
└─ hmi/
   ├─ __init__.py
   └─ hmi_monitor.py
```

---

# 2) Dependencies & setup

```bash
python -m venv .venv
. .venv/bin/activate
pip install "pymodbus==3.11.3" pyyaml
```

---

# 3) Config & logging

## `config.yaml` (example)

```yaml
plc_server:
  host: "127.0.0.1"
  port: 5020

actuator_server:
  host: "127.0.0.1"
  port: 5021

encoding:
  type: "float32"        # or "scaled_int"
  endianness: "big"      # "big"|"little" (affects word order)
  scale: 100             # used if type == scaled_int
  width_words: 2         # float32=2 words; scaled_int could be 1 or 2

logic:
  scan_ms: 100
  temp_high: 60.0
  pressure_high: 2.5
  temp_window: 5
  press_window: 5

actuator:
  sim_step_ms: 100
  motion_delay_ms: 800

sensor_sim:
  interval_ms: 200
  temp_profile:
    type: sine
    ampl: 15
    offset: 45
    period_ms: 8000
  pressure_profile:
    type: random_walk
    start: 1.2
    step_max: 0.05
    min: 0.5
    max: 3.5

hmi:
  interval_ms: 300
```

## `logging.ini` (very simple)

```ini
[loggers]
keys=root

[handlers]
keys=console

[formatters]
keys=std

[logger_root]
level=INFO
handlers=console

[handler_console]
class=StreamHandler
level=DEBUG
formatter=std
args=(sys.stdout,)

[formatter_std]
format=%(asctime)s %(levelname)s %(name)s: %(message)s
```

---

# 4) Addresses

Use your constants as-is in `addresses.py` (pure integers). No code changes needed.

---

# 5) Common plumbing

## `common/utils.py`

```python
import asyncio
import logging
import logging.config
import time
import yaml

def load_yaml(path: str):
    with open(path, "r") as f:
        return yaml.safe_load(f)

def setup_logging(ini_path: str):
    logging.config.fileConfig(ini_path, disable_existing_loggers=False)
    return logging.getLogger("modbus")

def schedule_periodic(interval_ms: int, task_fn):
    async def _runner():
        period = interval_ms / 1000.0
        while True:
            t0 = time.perf_counter()
            try:
                await task_fn()
            except Exception as e:
                logging.getLogger("modbus").exception("periodic task error: %s", e)
            dt = time.perf_counter() - t0
            await asyncio.sleep(max(0.0, period - dt))
    return asyncio.create_task(_runner())

# Optional guard wrappers
async def safe_read(fn, *args, **kwargs):
    try:
        return await fn(*args, **kwargs)
    except Exception as e:
        logging.getLogger("modbus").warning("safe_read: %s", e)
        return None

async def safe_write(fn, *args, **kwargs):
    try:
        return await fn(*args, **kwargs)
    except Exception as e:
        logging.getLogger("modbus").warning("safe_write: %s", e)
        return False
```

## `common/encoding.py`

```python
import struct

def _float_to_words(f: float, endianness: str):
    b = struct.pack(">f" if endianness == "big" else "<f", f)
    hi, lo = b[0:2], b[2:4]
    if endianness == "big":
        return [int.from_bytes(hi, "big"), int.from_bytes(lo, "big")]
    else:
        return [int.from_bytes(hi, "little"), int.from_bytes(lo, "little")]

def _words_to_float(words, endianness: str):
    w0, w1 = words[0], words[1]
    if endianness == "big":
        b = w0.to_bytes(2, "big") + w1.to_bytes(2, "big")
        return struct.unpack(">f", b)[0]
    else:
        b = w0.to_bytes(2, "little") + w1.to_bytes(2, "little")
        return struct.unpack("<f", b)[0]

def encode_value(value, enc_cfg):
    t = enc_cfg["type"]
    if t == "scaled_int":
        scale = enc_cfg.get("scale", 1)
        width = enc_cfg.get("width_words", 1)
        val = int(round(value * scale))
        if width == 1:
            return [val & 0xFFFF]
        elif width == 2:
            return [(val >> 16) & 0xFFFF, val & 0xFFFF]
        else:
            raise ValueError("unsupported width_words")
    elif t == "float32":
        return _float_to_words(float(value), enc_cfg.get("endianness", "big"))
    else:
        raise ValueError("unknown encoding type")

def decode_value(words, enc_cfg):
    t = enc_cfg["type"]
    if t == "scaled_int":
        scale = enc_cfg.get("scale", 1)
        width = enc_cfg.get("width_words", 1)
        if width == 1:
            raw = words[0]
            # naive unsigned -> signed fix (optional)
            if raw & 0x8000: raw = raw - 0x10000
        else:
            raw = (words[0] << 16) | words[1]
            if raw & 0x80000000: raw -= 0x100000000
        return raw / scale
    elif t == "float32":
        return _words_to_float(words, enc_cfg.get("endianness", "big"))
    else:
        raise ValueError("unknown encoding type")
```

---

# 6) PLC logic & state

## `plc/logic.py`

```python
from dataclasses import dataclass

@dataclass
class Alarms:
    high_temp: bool
    high_press: bool

@dataclass
class Cmds:
    valve1_open: bool
    valve2_open: bool
    door1_unlock: bool
    door2_unlock: bool
    door3_unlock: bool

def compute_alarms(temp, press, logic_cfg) -> Alarms:
    return Alarms(
        high_temp = temp > logic_cfg["temp_high"],
        high_press = press > logic_cfg["pressure_high"],
    )

def compute_permissive(temp, press, logic_cfg) -> bool:
    return (temp < logic_cfg["temp_high"]) and (press < logic_cfg["pressure_high"])

def decide_commands(temp, press, alarms: Alarms, permissive_ok: bool, logic_cfg) -> Cmds:
    return Cmds(
        valve1_open = alarms.high_temp and (not alarms.high_press),
        valve2_open = (not alarms.high_temp) and (not alarms.high_press),
        door1_unlock = permissive_ok,
        door2_unlock = permissive_ok,
        door3_unlock = permissive_ok,
    )
```

## `plc/state.py`

```python
from collections import deque

class MovingAverage:
    def __init__(self, window: int):
        self.buf = deque(maxlen=window)
    def update(self, v: float) -> float:
        self.buf.append(float(v))
        return sum(self.buf) / len(self.buf)

class State:
    def __init__(self, cfg_logic):
        self.temp_filter = MovingAverage(cfg_logic["temp_window"])
        self.press_filter = MovingAverage(cfg_logic["press_window"])
        self.last_cmds = None
        self.timers = {}
    def filter_temp(self, v): return self.temp_filter.update(v)
    def filter_press(self, v): return self.press_filter.update(v)
```

---

# 7) PLC server (datastore + periodic logic)

Key points with `pymodbus 3.11.3`:

- Async server: `from pymodbus.server.async_io import StartAsyncTcpServer`
    
- Datastore: `ModbusServerContext(ModbusSlaveContext(...), single=True)`
    
- Blocks: `ModbusSequentialDataBlock(0, [0]*N)` for `co`, `di`, `hr`, `ir`
    
- Access: read/write via the datastore (`context[device_id].getValues(table, address, count)`).
    

## `plc/plc_server.py`

```python
import asyncio
import logging
from dataclasses import asdict

from pymodbus.datastore import ModbusServerContext, ModbusSlaveContext, ModbusSequentialDataBlock
from pymodbus.server.async_io import StartAsyncTcpServer

from addresses import *
from common.utils import load_yaml, setup_logging, schedule_periodic
from common.encoding import encode_value, decode_value
from plc.state import State
from plc.logic import compute_alarms, compute_permissive, decide_commands
from plc.actuator_client import ActuatorClient

log = logging.getLogger("plc")

def _make_device_context(cfg):
    # Size your blocks safely above highest address you use.
    CO_SIZE = 128
    DI_SIZE = 128
    HR_SIZE = 256
    IR_SIZE = 256
    store = ModbusSlaveContext(
        di=ModbusSequentialDataBlock(0, [0]*DI_SIZE),
        co=ModbusSequentialDataBlock(0, [0]*CO_SIZE),
        hr=ModbusSequentialDataBlock(0, [0]*HR_SIZE),
        ir=ModbusSequentialDataBlock(0, [0]*IR_SIZE),
        zero_mode=True,
    )
    # Optionally seed ingress HRs (temp/press) here
    return ModbusServerContext(slaves=store, single=True)

def _read_words(ctx, table: str, address: int, count: int, device_id: int = 0x01):
    return ctx[device_id].getValues(table, address, count)

def _write_words(ctx, table: str, address: int, values, device_id: int = 0x01):
    ctx[device_id].setValues(table, address, list(values))

def hr_read(ctx, addr, width): return _read_words(ctx, "hr", addr, width)
def hr_write(ctx, addr, words): _write_words(ctx, "hr", addr, words)
def ir_write(ctx, addr, words): _write_words(ctx, "ir", addr, words)
def co_write(ctx, addr, bit): _write_words(ctx, "co", addr, [int(bool(bit))])

async def logic_tick(ctx, state: State, actuator_cli: "ActuatorClient", cfg):
    enc = cfg["encoding"]
    width = enc["width_words"]

    temp_raw  = hr_read(ctx, HR_TEMP, width)
    press_raw = hr_read(ctx, HR_PRESS, width)
    temp  = decode_value(temp_raw, enc)
    press = decode_value(press_raw, enc)

    temp_f  = state.filter_temp(temp)
    press_f = state.filter_press(press)

    alarms = compute_alarms(temp_f, press_f, cfg["logic"])
    permissive_ok = compute_permissive(temp_f, press_f, cfg["logic"])
    cmds = decide_commands(temp_f, press_f, alarms, permissive_ok, cfg["logic"])

    ir_write(ctx, IR_TEMP_FILTERED, encode_value(temp_f, enc))
    ir_write(ctx, IR_PRESS_FILTERED, encode_value(press_f, enc))
    co_write(ctx, CO_ALARM_HIGH_TEMP, alarms.high_temp)
    co_write(ctx, CO_ALARM_HIGH_PRESS, alarms.high_press)
    co_write(ctx, CO_PERMISSIVE_OK, permissive_ok)
    hr_write(ctx, HR_CMD_VALVE1, [int(cmds.valve1_open)])
    hr_write(ctx, HR_CMD_VALVE2, [int(cmds.valve2_open)])
    hr_write(ctx, HR_CMD_DOOR1,  [int(cmds.door1_unlock)])
    hr_write(ctx, HR_CMD_DOOR2,  [int(cmds.door2_unlock)])
    hr_write(ctx, HR_CMD_DOOR3,  [int(cmds.door3_unlock)])

    await actuator_cli.apply_commands(cmds)

    log.debug("PLC tick: temp_f=%.2f press_f=%.2f alarms=%s cmds=%s",
              temp_f, press_f, alarms, cmds)

async def main():
    cfg = load_yaml("config.yaml")
    setup_logging("logging.ini")
    device_ctx = _make_device_context(cfg)
    plc_state = State(cfg["logic"])
    actuator_cli = ActuatorClient(cfg["actuator_server"])

    # schedule periodic logic
    schedule_periodic(cfg["logic"]["scan_ms"], lambda: logic_tick(device_ctx, plc_state, actuator_cli, cfg))

    # start async server
    await StartAsyncTcpServer(context=device_ctx,
                              address=(cfg["plc_server"]["host"], cfg["plc_server"]["port"]))

if __name__ == "__main__":
    asyncio.run(main())
```

---

# 8) PLC → Actuator client

## `plc/actuator_client.py`

```python
from pymodbus.client import AsyncModbusTcpClient
from addresses import *
import asyncio

class ActuatorClient:
    def __init__(self, server_addr):
        self.host = server_addr["host"]
        self.port = server_addr["port"]
        self.cli = AsyncModbusTcpClient(self.host, port=self.port)

    async def ensure_connected(self):
        if not self.cli.connected:
            await self.cli.connect()

    async def _write_coil(self, address: int, value: bool, device_id: int = 0x01):
        await self.ensure_connected()
        # device_id (a.k.a unit_id) is keyword in 3.x
        await self.cli.write_coil(address=address, value=bool(value), device_id=device_id)

    async def apply_commands(self, cmds):
        await asyncio.gather(
            self._write_coil(CO_VALVE1_OPEN, cmds.valve1_open),
            self._write_coil(CO_VALVE2_OPEN, cmds.valve2_open),
            self._write_coil(CO_DOOR1_UNLOCK, cmds.door1_unlock),
            self._write_coil(CO_DOOR2_UNLOCK, cmds.door2_unlock),
            self._write_coil(CO_DOOR3_UNLOCK, cmds.door3_unlock),
        )

    async def close(self):
        await self.cli.close()
```

---

# 9) Actuator server + simulator

## `actuators/actuator_server.py`

```python
import asyncio
import logging
from pymodbus.datastore import ModbusServerContext, ModbusSlaveContext, ModbusSequentialDataBlock
from pymodbus.server.async_io import StartAsyncTcpServer
from common.utils import load_yaml, setup_logging, schedule_periodic
from actuators.simulator import simulate_motion

log = logging.getLogger("actuators")

def _make_device_context():
    CO_SIZE = 128
    DI_SIZE = 128
    HR_SIZE = 64
    store = ModbusSlaveContext(
        di=ModbusSequentialDataBlock(0, [0]*DI_SIZE),
        co=ModbusSequentialDataBlock(0, [0]*CO_SIZE),
        hr=ModbusSequentialDataBlock(0, [0]*HR_SIZE),
        ir=None,
        zero_mode=True,
    )
    return ModbusServerContext(slaves=store, single=True)

async def main():
    cfg = load_yaml("config.yaml")
    setup_logging("logging.ini")
    ctx = _make_device_context()
    schedule_periodic(cfg["actuator"]["sim_step_ms"], lambda: simulate_motion(ctx, cfg))
    await StartAsyncTcpServer(context=ctx,
                              address=(cfg["actuator_server"]["host"], cfg["actuator_server"]["port"]))

if __name__ == "__main__":
    asyncio.run(main())
```

## `actuators/simulator.py`

```python
import time
from addresses import *

# map command coil -> feedback DI
MAPPING = [
    (CO_VALVE1_OPEN, DI_VALVE1_OPENED),
    (CO_VALVE2_OPEN, DI_VALVE2_OPENED),
    (CO_DOOR1_UNLOCK, DI_DOOR1_OPEN),
    (CO_DOOR2_UNLOCK, DI_DOOR2_OPEN),
    (CO_DOOR3_UNLOCK, DI_DOOR3_OPEN),
]

_state = {
    # cmd_addr -> {"last": 0/1, "t_start": secs, "target": 0/1}
}

def _read_co(ctx, addr, device_id=0x01):
    return ctx[device_id].getValues("co", addr, 1)[0]

def _write_di(ctx, addr, bit, device_id=0x01):
    ctx[device_id].setValues("di", addr, [int(bool(bit))])

def simulate_motion(ctx, cfg):
    delay_s = cfg["actuator"]["motion_delay_ms"] / 1000.0
    now = time.perf_counter()

    for cmd_addr, di_addr in MAPPING:
        cur = _read_co(ctx, cmd_addr)
        st = _state.setdefault(cmd_addr, {"last": cur, "t_start": None, "target": cur})

        if cur != st["last"]:
            st["t_start"] = now
            st["target"] = cur
            st["last"] = cur

        if st["t_start"] is not None and (now - st["t_start"]) >= delay_s:
            _write_di(ctx, di_addr, st["target"])
            st["t_start"] = None
```

---

# 10) Sensor writer

## `sensors/profiles.py`

```python
import math
import random
import time

class Fixed:
    def __init__(self, value): self.value=float(value)
    def next_value(self): return self.value

class Ramp:
    def __init__(self, start, step, min, max, wrap=True):
        self.cur=float(start); self.step=float(step); self.min=min; self.max=max; self.wrap=wrap
    def next_value(self):
        v=self.cur
        self.cur += self.step
        if self.wrap:
            span = self.max - self.min
            while self.cur > self.max: self.cur -= span
            while self.cur < self.min: self.cur += span
        else:
            self.cur = max(self.min, min(self.max, self.cur))
        return v

class Sine:
    def __init__(self, ampl, offset, period_ms):
        self.ampl=float(ampl); self.offset=float(offset)
        self.period=float(period_ms)/1000.0; self.t0=time.perf_counter()
    def next_value(self):
        t=time.perf_counter()-self.t0
        return self.offset + self.ampl*math.sin(2*math.pi*t/self.period)

class RandomWalk:
    def __init__(self, start, step_max, min, max):
        self.cur=float(start); self.step_max=float(step_max); self.min=min; self.max=max
    def next_value(self):
        self.cur += random.uniform(-self.step_max, self.step_max)
        self.cur = max(self.min, min(self.max, self.cur))
        return self.cur

def build_profile(spec: dict):
    t = spec["type"]
    if t == "fixed": return Fixed(spec["value"])
    if t == "ramp":  return Ramp(spec["start"], spec["step"], spec["min"], spec["max"], True)
    if t == "sine":  return Sine(spec["ampl"], spec["offset"], spec["period_ms"])
    if t == "random_walk": return RandomWalk(spec["start"], spec["step_max"], spec["min"], spec["max"])
    raise ValueError(f"unknown profile type: {t}")
```

## `sensors/sensor_sim.py`

```python
import asyncio
from pymodbus.client import AsyncModbusTcpClient
from addresses import *
from common.utils import load_yaml, setup_logging
from common.encoding import encode_value
from sensors.profiles import build_profile

async def main():
    cfg = load_yaml("config.yaml")
    setup_logging("logging.ini")
    enc = cfg["encoding"]
    width = enc["width_words"]

    cli = AsyncModbusTcpClient(cfg["plc_server"]["host"], port=cfg["plc_server"]["port"])
    await cli.connect()

    temp_p = build_profile(cfg["sensor_sim"]["temp_profile"])
    press_p = build_profile(cfg["sensor_sim"]["pressure_profile"])
    period = cfg["sensor_sim"]["interval_ms"] / 1000.0

    while True:
        temp = temp_p.next_value()
        press = press_p.next_value()
        await cli.write_registers(address=HR_TEMP, values=encode_value(temp, enc), device_id=0x01)
        await cli.write_registers(address=HR_PRESS, values=encode_value(press, enc), device_id=0x01)
        await asyncio.sleep(period)

if __name__ == "__main__":
    asyncio.run(main())
```

> Note: `write_registers` is FC16 (multiple). If you ever switch to single-word HRs, you can still call `write_registers` with a one-element list.

---

# 11) HMI reader

## `hmi/hmi_monitor.py`

```python
import asyncio
from pymodbus.client import AsyncModbusTcpClient
from addresses import *
from common.utils import load_yaml, setup_logging
from common.encoding import decode_value

async def read_words(cli, table, addr, count, device_id=0x01):
    if table == "ir":
        resp = await cli.read_input_registers(address=addr, count=count, device_id=device_id)
        return resp.registers
    elif table == "hr":
        resp = await cli.read_holding_registers(address=addr, count=count, device_id=device_id)
        return resp.registers
    elif table == "co":
        resp = await cli.read_coils(address=addr, count=1, device_id=device_id)
        return [int(resp.bits[0])]
    elif table == "di":
        resp = await cli.read_discrete_inputs(address=addr, count=1, device_id=device_id)
        return [int(resp.bits[0])]
    else:
        raise ValueError("bad table")

def print_dashboard(t, p, alarms, cmds, fb):
    print(f"T={t:6.2f}  P={p:5.2f} | "
          f"ALM[T]={int(alarms['high_temp'])} ALM[P]={int(alarms['high_press'])} PERM={int(alarms['perm_ok'])} | "
          f"CMD V1={cmds['valve1']} V2={cmds['valve2']} D1={cmds['door1']} D2={cmds['door2']} D3={cmds['door3']} | "
          f"FB V1={fb['v1_opened']} V2={fb['v2_opened']} D1={fb['d1_open']} D2={fb['d2_open']} D3={fb['d3_open']}")

async def main():
    cfg = load_yaml("config.yaml")
    setup_logging("logging.ini")
    enc = cfg["encoding"]
    width = enc["width_words"]

    plc = AsyncModbusTcpClient(cfg["plc_server"]["host"], port=cfg["plc_server"]["port"])
    act = AsyncModbusTcpClient(cfg["actuator_server"]["host"], port=cfg["actuator_server"]["port"])
    await plc.connect(); await act.connect()

    period = cfg["hmi"]["interval_ms"]/1000.0
    while True:
        t = decode_value(await read_words(plc, "ir", IR_TEMP_FILTERED, width), enc)
        p = decode_value(await read_words(plc, "ir", IR_PRESS_FILTERED, width), enc)

        alarms = {
          "high_temp": (await read_words(plc, "co", CO_ALARM_HIGH_TEMP, 1))[0],
          "high_press": (await read_words(plc, "co", CO_ALARM_HIGH_PRESS, 1))[0],
          "perm_ok": (await read_words(plc, "co", CO_PERMISSIVE_OK, 1))[0],
        }
        cmds = {
          "valve1": (await read_words(plc, "hr", HR_CMD_VALVE1, 1))[0],
          "valve2": (await read_words(plc, "hr", HR_CMD_VALVE2, 1))[0],
          "door1":  (await read_words(plc, "hr", HR_CMD_DOOR1, 1))[0],
          "door2":  (await read_words(plc, "hr", HR_CMD_DOOR2, 1))[0],
          "door3":  (await read_words(plc, "hr", HR_CMD_DOOR3, 1))[0],
        }
        fb = {
          "v1_opened": (await read_words(act, "di", DI_VALVE1_OPENED, 1))[0],
          "v2_opened": (await read_words(act, "di", DI_VALVE2_OPENED, 1))[0],
          "d1_open":   (await read_words(act, "di", DI_DOOR1_OPEN, 1))[0],
          "d2_open":   (await read_words(act, "di", DI_DOOR2_OPEN, 1))[0],
          "d3_open":   (await read_words(act, "di", DI_DOOR3_OPEN, 1))[0],
        }
        print_dashboard(t, p, alarms, cmds, fb)
        await asyncio.sleep(period)

if __name__ == "__main__":
    asyncio.run(main())
```

---

# 12) Tests (lightweight)

## `tests/test_encoding.py` (pytest style)

```python
from common.encoding import encode_value, decode_value

def roundtrip(enc):
    for v in [0, 12.3, -5.5, 99.99]:
        w = encode_value(v, enc)
        v2 = decode_value(w, enc)
        assert abs(v - v2) < 1e-3

def test_float32_big():
    roundtrip({"type":"float32","endianness":"big","width_words":2})

def test_scaled_int_1w():
    roundtrip({"type":"scaled_int","scale":100,"width_words":1})
```

(You can add the address overlap checks if you keep ranges.)

---

# 13) Bring-up order & run

Use separate terminals (or background processes):

```bash
# 1) Actuator server
python -m actuators.actuator_server

# 2) PLC server
python -m plc.plc_server

# 3) Sensors
python -m sensors.sensor_sim

# 4) HMI
python -m hmi.hmi_monitor
```

You should see the HMI printing filtered T/P, alarms toggling as the profiles wander, commands mirroring on PLC HRs, and actuator DI feedback settling ~800 ms after command changes.

---

# 14) Gotchas & tips

- **Device/unit id**: In 3.11.x, client calls accept `device_id=` (a.k.a. unit id). I used `device_id=0x01` everywhere to align with your note.
    
- **Tables vs. rights**: Sensors write **Holding Registers** (HR) to PLC; HMI reads **Input Registers** (IR) for filtered values and **Coils** (CO) for alarms/permissives.
    
- **Endianness**: Keep it only in `encoding.py`. Wireshark will thank you.
    
- **Server context access**: Inside your own server, you use the **context API** (`getValues/setValues`) instead of “client” reads.
    
- **Async**: All servers and clients are async; avoid blocking in periodic tasks. If you later mix threads, protect the server context.
    
- **Sizing blocks**: Allocate blocks larger than your highest address (I used 128/256). If an address is out of range, writes silently fail.
    
- **FC06 vs FC16**: Using `write_registers()` (FC16) with a single word is fine. It simplifies width handling.
    

---

If you want, I can turn any single module above into a fully copy-pasteable file tailored to your exact `config.yaml` (e.g., switch to `scaled_int`, change motion model, add hysteresis).