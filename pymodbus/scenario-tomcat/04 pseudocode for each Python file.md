Below is **pseudocode for each Python file** in the tree we defined. It’s implementation-oriented (pymodbus 3.11.3 flow), but still library-agnostic enough to keep you flexible.

---

# `addresses.py` (constants only)

```pseudo
# PLC ingress (written by sensors)
HR_TEMP = 0          # 2 regs if float32
HR_PRESS = 2

# PLC derived/readback
IR_TEMP_FILTERED = 0
IR_PRESS_FILTERED = 2
IR_ALARMS = 10       # bitfield

# PLC alarms/permissives (coils)
CO_ALARM_HIGH_TEMP = 0
CO_ALARM_HIGH_PRESS = 1
CO_PERMISSIVE_OK = 2

# PLC command mirrors (for HMI readback)
HR_CMD_VALVE1 = 100
HR_CMD_VALVE2 = 101
HR_CMD_DOOR1  = 110
HR_CMD_DOOR2  = 111
HR_CMD_DOOR3  = 112

# Actuator server (commands)
CO_VALVE1_OPEN = 0
CO_VALVE2_OPEN = 1
CO_DOOR1_UNLOCK = 10
CO_DOOR2_UNLOCK = 11
CO_DOOR3_UNLOCK = 12

# Actuator server (feedback)
DI_VALVE1_OPENED = 0
DI_VALVE2_OPENED = 1
DI_DOOR1_OPEN    = 10
DI_DOOR2_OPEN    = 11
DI_DOOR3_OPEN    = 12
```

---

# `plc/plc_server.py` (Modbus server + scheduler)

```pseudo
main():
  cfg = load_yaml("config.yaml")
  log = setup_logging("logging.ini")

  # Build datastore blocks with initial values (co/di/hr/ir)
  device = make_device_context(
    co = zeros(size=...),
    di = zeros(size=...),
    hr = initial_hr_values(),   # includes ingress HRs for temp/press
    ir = zeros(size=...)
  )

  server_ctx = make_server_context(devices=device, single=True)

  # Start PLC logic task (periodic)
  plc_state = State()  # from state.py
  actuator_cli = ActuatorClient(cfg.actuator_server)  # from actuator_client.py
  schedule_periodic( interval=cfg.logic.scan_ms, task=logic_tick(server_ctx, plc_state, actuator_cli, cfg) )

  # Start Modbus TCP server on cfg.plc_server.host:port (async)
  start_async_tcp_server(context=server_ctx, address=(cfg.plc_server.host, cfg.plc_server.port))

logic_tick(ctx, state, actuator_cli, cfg):
  # 1) Read ingress HRs (temp, press)
  temp_raw  = hr_read(ctx, HR_TEMP, width=cfg.encoding.width)
  press_raw = hr_read(ctx, HR_PRESS, width=cfg.encoding.width)
  temp = decode_value(temp_raw, cfg.encoding)     # use common/encoding.py
  press = decode_value(press_raw, cfg.encoding)

  # 2) Filter/debounce (state keeps filters/timers)
  temp_f = state.filter_temp(temp)
  press_f = state.filter_press(press)

  # 3) Evaluate logic: thresholds, interlocks
  alarms = compute_alarms(temp_f, press_f, cfg.logic)
  permissive_ok = compute_permissive(temp_f, press_f, cfg.logic)

  # 4) Decide actuator commands
  cmds = decide_commands(temp_f, press_f, alarms, permissive_ok, cfg.logic)
  # cmds example: { valve1_open: True/False, door1_unlock: True/False, ... }

  # 5) Mirror for HMI: write IR/CO/HR in PLC server
  ir_write(ctx, IR_TEMP_FILTERED, encode_value(temp_f, cfg.encoding))
  ir_write(ctx, IR_PRESS_FILTERED, encode_value(press_f, cfg.encoding))
  co_write(ctx, CO_ALARM_HIGH_TEMP, alarms.high_temp)
  co_write(ctx, CO_ALARM_HIGH_PRESS, alarms.high_press)
  co_write(ctx, CO_PERMISSIVE_OK, permissive_ok)
  hr_write(ctx, HR_CMD_VALVE1, int(cmds.valve1_open))
  hr_write(ctx, HR_CMD_VALVE2, int(cmds.valve2_open))
  hr_write(ctx, HR_CMD_DOOR1,  int(cmds.door1_unlock))
  hr_write(ctx, HR_CMD_DOOR2,  int(cmds.door2_unlock))
  hr_write(ctx, HR_CMD_DOOR3,  int(cmds.door3_unlock))

  # 6) Command actuators (PLC acts as Modbus client toward actuator server)
  actuator_cli.apply_commands(cmds)  # writes coils on actuator server

  # 7) Log summary for traceability
  log.debug("PLC tick", temp_f, press_f, alarms, cmds)
```

---

# `plc/logic.py` (pure decision logic; side-effect free)

```pseudo
compute_alarms(temp, press, logic_cfg):
  high_temp = temp > logic_cfg.temp_high
  high_press = press > logic_cfg.pressure_high
  return { high_temp, high_press }

compute_permissive(temp, press, logic_cfg):
  return (temp < logic_cfg.temp_high) AND (press < logic_cfg.pressure_high)

decide_commands(temp, press, alarms, permissive_ok, logic_cfg):
  cmds = {}
  # Example policy: open valve1 if temp high and pressure not high; else close
  cmds.valve1_open = alarms.high_temp AND NOT alarms.high_press
  cmds.valve2_open = NOT alarms.high_temp AND NOT alarms.high_press

  # Doors: unlock only when permissive is true
  cmds.door1_unlock = permissive_ok
  cmds.door2_unlock = permissive_ok
  cmds.door3_unlock = permissive_ok

  # Add hysteresis/deadband if needed (handled via state filters or here)
  return cmds
```

---

# `plc/actuator_client.py` (PLC → Actuator server Modbus client)

```pseudo
class ActuatorClient:
  init(server_addr):
    self.addr = server_addr
    self.cli = create_modbus_tcp_client(server_addr.host, server_addr.port)

  ensure_connected():
    if not self.cli.connected:
      self.cli.connect()

  apply_commands(cmds):
    ensure_connected()
    # Write coils on actuator server (device_id=1 unless you use multi-device)
    write_coil(CO_VALVE1_OPEN, cmds.valve1_open)
    write_coil(CO_VALVE2_OPEN, cmds.valve2_open)
    write_coil(CO_DOOR1_UNLOCK, cmds.door1_unlock)
    write_coil(CO_DOOR2_UNLOCK, cmds.door2_unlock)
    write_coil(CO_DOOR3_UNLOCK, cmds.door3_unlock)

  close():
    self.cli.close()
```

---

# `plc/state.py` (filters, timers, latches)

```pseudo
class State:
  init():
    self.temp_filter = MovingAverage(window=cfg.logic.temp_window)
    self.press_filter = MovingAverage(window=cfg.logic.press_window)
    self.last_cmds = None
    self.timers = {}  # e.g., debounce timers

  filter_temp(v):
    return self.temp_filter.update(v)

  filter_press(v):
    return self.press_filter.update(v)

class MovingAverage:
  init(window):
    self.buf = ring_buffer(size=window)
  update(v):
    buf.push(v)
    return avg(buf)
```

---

# `actuators/actuator_server.py` (Modbus server for valves/doors)

```pseudo
main():
  cfg = load_yaml("config.yaml")
  log = setup_logging("logging.ini")

  # Build device context:
  #   coils = command inputs from PLC (default 0)
  #   di    = feedback bits (default 0)
  #   hr    = optional telemetry (timestamps, faults)
  device = make_device_context(
    co = zeros(size=...),
    di = zeros(size=...),
    hr = zeros(size=...)
  )
  server_ctx = make_server_context(devices=device, single=True)

  # Periodic simulator task to update DI from CO with delay/faults
  schedule_periodic(interval=cfg.actuator.sim_step_ms, task=simulate_motion(server_ctx, cfg))

  start_async_tcp_server(context=server_ctx, address=(cfg.actuator_server.host, cfg.actuator_server.port))
```

---

# `actuators/simulator.py` (motion delay & feedback)

```pseudo
simulate_motion(ctx, cfg):
  # Read command coils, update DI after configured delay
  # Maintain internal per-device state: when a command changed, start a timer
  static state = { timers_by_point }

  # For each actuator point:
  for (cmd_coil, fb_di) in mapping_list:
    cmd = co_read(ctx, cmd_coil)

    if command_changed_since_last_tick(cmd):
       start_timer(state, cmd_coil, cfg.actuator.motion_delay_ms)

    if timer_expired(state, cmd_coil):
       # reflect motion completion
       di_write(ctx, fb_di, cmd)  # DI shows opened if coil==1; closed if 0

  # Optionally set HR_FAULT_CODE with low probability to simulate faults
```

---

# `sensors/sensor_sim.py` (sensor → PLC HR writer)

```pseudo
main():
  cfg = load_yaml("config.yaml")
  log = setup_logging("logging.ini")
  enc = cfg.encoding

  plc_cli = create_modbus_tcp_client(cfg.plc_server.host, cfg.plc_server.port)

  profile_temp = build_profile(cfg.sensor_sim.temp_profile)    # from profiles.py
  profile_press = build_profile(cfg.sensor_sim.pressure_profile)

  loop every cfg.sensor_sim.interval_ms:
    temp = profile_temp.next_value()
    press = profile_press.next_value()

    temp_words  = encode_value(temp, enc)   # scaled int OR float32 words
    press_words = encode_value(press, enc)

    ensure plc_cli connected
    write_holding_registers(HR_TEMP, temp_words)   # FC06/16 depending on width
    write_holding_registers(HR_PRESS, press_words)

    log.info("sensor→plc", temp, press)
```

---

# `sensors/profiles.py` (value generators)

```pseudo
build_profile(spec):
  switch spec.type:
    case "fixed":
      return Fixed(spec.value)
    case "ramp":
      return Ramp(start=spec.start, step=spec.step, min=spec.min, max=spec.max, wrap=True)
    case "sine":
      return Sine(ampl=..., offset=..., period=...)
    case "random_walk":
      return RandomWalk(start=..., step_max=..., bounds=(min,max))
    default:
      raise config_error

class Ramp:
  next_value():
    v = current
    current = wrap_or_clamp(current + step)
    return v
# (implement others similarly)
```

---

# `hmi/hmi_monitor.py` (read-only poller)

```pseudo
main():
  cfg = load_yaml("config.yaml")
  plc_cli = create_modbus_tcp_client(cfg.plc_server.host, cfg.plc_server.port)
  act_cli = create_modbus_tcp_client(cfg.actuator_server.host, cfg.actuator_server.port)

  loop every cfg.hmi.interval_ms:
    # From PLC
    t = decode_value(read_input_registers(IR_TEMP_FILTERED, width), enc)
    p = decode_value(read_input_registers(IR_PRESS_FILTERED, width), enc)
    alarms = {
      "high_temp": read_coil(CO_ALARM_HIGH_TEMP),
      "high_press": read_coil(CO_ALARM_HIGH_PRESS),
      "perm_ok": read_coil(CO_PERMISSIVE_OK)
    }
    cmds = {
      "valve1": read_holding_register(HR_CMD_VALVE1),
      "valve2": read_holding_register(HR_CMD_VALVE2),
      "door1":  read_holding_register(HR_CMD_DOOR1),
      ...
    }

    # From Actuator server (feedback)
    fb = {
      "v1_opened": read_discrete(DI_VALVE1_OPENED),
      "v2_opened": read_discrete(DI_VALVE2_OPENED),
      "d1_open":   read_discrete(DI_DOOR1_OPEN),
      ...
    }

    print_compact_dashboard(t, p, alarms, cmds, fb)
```

---

# `common/encoding.py` (packing/unpacking)

```pseudo
encode_value(value, enc_cfg):
  if enc_cfg.type == "scaled_int":
    return int(round(value * enc_cfg.scale))  # single 16-bit or 32-bit as needed
  if enc_cfg.type == "float32":
    return float_to_two_words(value, endianness=enc_cfg.endianness)
  else: error

decode_value(words, enc_cfg):
  if enc_cfg.type == "scaled_int":
    return words_to_int(words) / enc_cfg.scale
  if enc_cfg.type == "float32":
    return two_words_to_float(words, endianness=enc_cfg.endianness)
```

---

# `common/utils.py` (plumbing)

```pseudo
load_yaml(path): return parsed_yaml
setup_logging(ini_path): configure logging; return logger

retry_connect(client, backoff): try connect; sleep & retry as needed
schedule_periodic(interval_ms, task_fn): create async task with sleep loop

safe_read(fn, *args): try fn; on error log & return None
safe_write(fn, *args): try fn; on error log & return False
```

---

# Test stubs (optional)

## `tests/test_addresses.py`

```pseudo
assert_no_overlap(PLC_HR_INGRESS_RANGE, PLC_HR_COMMAND_RANGE, PLC_IR_RANGE, PLC_CO_RANGE)
assert_in_bounds(each_constant, MAX_RANGE)
```

## `tests/test_encoding.py`

```pseudo
for v in [0, 12.3, -5.5, 99.99]:
  words = encode_value(v, cfg.encoding)
  v2 = decode_value(words, cfg.encoding)
  assert approx_equal(v, v2)
```

---

## Notes to keep implementation consistent

- **Client methods** in pymodbus 3.11.3 use **keyword args** and **`device_id=`**; server is **async**.
    
- Use **Holding Registers** for sensor ingress (clients can’t write Input Registers).
    
- Keep **endianness and scaling** centralized in `encoding.py`.
    
- For **Wireshark**, capture both ports (PLC 5020, Actuator 5021) and “Decode As → Modbus/TCP”.
    

If you want, I can convert any of these pseudocode blocks into actual Python next.