python recommended version: *3.11* 

install python lib: 
```bash
pip install pymodbus
```


### Examples
- [async client](https://github.com/pymodbus-dev/pymodbus/blob/dev/examples/simple_async_client.py)
- [sync client](https://github.com/pymodbus-dev/pymodbus/blob/dev/examples/simple_sync_client.py)
- [Advanced examples](https://pymodbus.readthedocs.io/en/dev/source/examples.html)

### Serial (RS-485)[](https://pymodbus.readthedocs.io/en/latest/source/client.html#serial-rs-485 "Link to this heading")

Pymodbus do not connect to the device (server) but connects to a comm port or usb port on the local computer.

RS-485 is a half duplex protocol, meaning the servers do nothing until the client sends a request then the server being addressed responds. The client controls the traffic and as a consequence one RS-485 line can only have 1 client but upto 254 servers (physical devices).

RS-485 is a simple 2 wire cabling with a pullup resistor. It is important to note that many USB converters do not have a builtin resistor, this must be added manually. When experiencing many faulty packets and retries this is often the problem.