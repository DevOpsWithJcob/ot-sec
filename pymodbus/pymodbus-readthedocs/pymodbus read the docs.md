## client protocols:
Pymodbus offers clients with transport different protocols and different framers

| protocol        | ASCCII | RTU | RTU_Over_TCP | Socket | TLS |
| --------------- | ------ | --- | ------------ | ------ | --- |
| Serial (RS-485) | Yes    | Yes | No           | No     | No  |
| TCP             | Yes    | No  | Yes          | Yes    | No  |
| TLS             | No     | No  | No           | No     | Yes |
| UDP             | Yes    | No  | Yes          | Yes    | No  |

### Serial (RS-485)
Pymodbus do not connect to the device (server) but connects to a comm port or usb port on the local computer. RS-485 is a half duplex protocol, meaning the servers do nothing until the client sends a request then the server being addressed responds. The client controls the traffic and as a consequence one RS-485 line can only have 1 client but upto 254 servers (physical devices). RS-485 is a simple 2 wire cabling with a pullup resistor. It is important to note that many USB converters do not have a builtin resistor, this must be added manually. When experiencing many faulty packets and retries this is often the problem.

### TCP
Pymodbus connects directly to the device using a standard socket and have a one-to-one connection with the device. In case of multiple TCP devices the application must instantiate multiple client objects one for each connection.

> **NOTE:** a TCP device often represent multiple physical devices (e.g Ethernet-RS485 converter), each of these devices can be addressed normally

### TLS
A variant of TCP that uses encryption and certificates. TLS is mostly used when the devices are connected to the internet. 
### UDP
A broadcast variant of TCP. UDP allows addressing of many devices with a single request, however there are no control that a device have received the packet.

---
####  Client usage
Using pymodbus client to set/get information from a device (server) is done in a few simple steps.

#### Asynchronous example
```python
import asyncio  
from pymodbus.client import AsyncModbusTcpClient  
async def main():  
    # Create the client  
    client = AsyncModbusTcpClient("MyDevice.lan")  
    # Connect (async)  
    await client.connect()  # returns True/False for connection success  
    # Write a single coil (turn something ON)    await client.write_coil(1, True, unit=1)  # (note: use 'unit', not 'device_id')  
    # Read three coils starting at address 2    result = await client.read_coils(2, 3, unit=1)  
    if not result.isError():  
        print(f"First bit read: {result.bits[0]}")  
    else:  
        print(f"Error reading coils: {result}")  
    # Close connection cleanly  
    await client.close()  
  
asyncio.run(main())
```

#### Retry logic for async clients
If 3 consequitve requests (calls) do not receive a response, the connection is terminated.

## Development Nots
Large parts of the implementation are shared between the different classes, to ensure high stability and efficient maintenance. 
The synchronous clients are not thread safe nor is a single client intended to be used from multiple threads. Due to the nature of the modbus protocol, it makes little sense to have client calls split over different threads, however the application can do it with proper locking implemented.

The asynchronous client only runs in the thread where the asyncio loop is created, it does not provide mechanisms to prevent (semi)parallel calls, that must be prevented at application level.

## Client device addressing
With **TCP**, **TLS** and **UDP**, the tcp/ip address of the physical device is defined when creating the object. Logical devices represented by the device is addressed with the `device_id=parameter`.

With **Serial**, the comm port is defined when creating the object. The physical devices are addressed with the device_id= parameter

>device_id=0 is defined as broadcast in the modbus standard, but pymodbus treats it as a normal device. please note device_id=0 can only be used to address devices that truly have id=0 ! Using device_id=0 to address a single device with id not 0 is against the protocol.

If an application is expecting multiple responses to a broadcast request, it must call client.execute and deal with the responses.

If no response is expected to a request, the no_response_expected=True argument can be used in the normal API calls, this will cause the call to return immediately with ExceptionResponse(0xff).

## Client response handling

```python
import asyncio
import logging
from pymodbus.client import AsyncModbusTcpClient
from pymodbus.exceptions import ModbusException

# ---------------------------------------------------------------------------
# Setup logging
# ---------------------------------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
_logger = logging.getLogger(__name__)

# ---------------------------------------------------------------------------
# Async main logic
# ---------------------------------------------------------------------------
async def main():
    client = AsyncModbusTcpClient("127.0.0.1", port=5020)
    connected = await client.connect()

    if not connected:
        _logger.error("Failed to connect to Modbus server.")
        return

    try:
        # Example: write coil 1 ON
        await client.write_coil(1, True, unit=1)

        # Example: safely read coil 1
        try:
            rr = await client.read_coils(1, 1, unit=1)  # 'unit' preferred over device_id
        except ModbusException as exc:
            _logger.error(f"ERROR: exception in pymodbus {exc}")
            raise

        if rr.isError():
            _logger.error(f"ERROR: pymodbus returned an error! Response={rr}")
            raise ModbusException("Read coils failed.")

        # Success — handle response
        _logger.info(f"Coil 1 state: {rr.bits[0]}")

    except Exception as e:
        _logger.exception(f"Unexpected error during Modbus operation: {e}")

    finally:
        await client.close()
        _logger.info("Modbus client closed.")

# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------
if __name__ == "__main__":
    asyncio.run(main())

```


### **3 connection retries** (2 seconds apart)

```python
import asyncio
import logging
from pymodbus.client import AsyncModbusTcpClient
from pymodbus.exceptions import ModbusException

# ---------------------------------------------------------------------------
# Setup Logging
# ---------------------------------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
_logger = logging.getLogger(__name__)

# ---------------------------------------------------------------------------
# Async main logic
# ---------------------------------------------------------------------------
async def main():
    host = "127.0.0.1"
    port = 5020
    max_retries = 3
    retry_delay = 2  # seconds

    client = AsyncModbusTcpClient(host, port=port)

    connected = False
    for attempt in range(1, max_retries + 1):
        _logger.info(f"Attempt {attempt}/{max_retries}: Connecting to {host}:{port} ...")
        connected = await client.connect()
        if connected:
            _logger.info("Connected to Modbus server.")
            break
        else:
            _logger.warning(f"Connection attempt {attempt} failed. Retrying in {retry_delay}s...")
            await asyncio.sleep(retry_delay)

    if not connected:
        _logger.error(f"Failed to connect to Modbus server after {max_retries} attempts.")
        return

    try:
        # Example: write coil 1 ON
        await client.write_coil(1, True, unit=1)

        # Example: safely read coil 1
        try:
            rr = await client.read_coils(1, 1, unit=1)
        except ModbusException as exc:
            _logger.error(f"Error: exception in pymodbus {exc}")
            raise

        if rr.isError():
            _logger.error(f"ERROR: pymodbus returned an error! Response={rr}")
        else:
            _logger.info(f"Coil 1 state: {rr.bits[0]}")

    except Exception as e:
        _logger.exception(f"Unexpected error during Modbus operation: {e}")

    finally:
        await client.close()
        _logger.info("Modbus client closed.")

# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------
if __name__ == "__main__":
    asyncio.run(main())

```


---
## Client interface classes
There are a client class for each type of communication and for asynchronous/synchronous

| Protocol | Async                   | sync               |
| -------- | ----------------------- | ------------------ |
| Serial   | AsyncModbusSerialClient | ModbusSerialClient |
| TCP      | AsyncModbusTcpClient    | ModbusTcpClient    |
| TLS      | AsyncModbusTlsClient    | ModbusTlsClient    |
| UDP      | AsyncModbusUdpClient    | ModbusUdpClient    |


## Client common
Some methods are common to all clients: 
```python 
class pymodbus.client.base.ModbusBaseClient(framer: FramerType, retries: int, comm_params: CommParams, trace_packet: Callable[[bool, bytes], bytes] | None, trace_pdu: Callable[[bool, ModbusPDU], ModbusPDU] | None, trace_connect: Callable[[bool], None] | None)

```

refrence https://app.readthedocs.org/projects/pymodbus/downloads/pdf/latest/ page 20


---
refrence https://app.readthedocs.org/projects/pymodbus/downloads/pdf/latest/
