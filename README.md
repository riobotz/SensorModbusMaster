# SensorModbusMaster

This library is designed to use an Arduino as a modbus master to communicate with a sensor/slave via [modbus RTU](https://en.wikipedia.org/wiki/Modbus).  It's specifically written with lots of "higher-level" functions to help out users who are largely unfamiliar with the modbus protocol and want an easy way to get information from a modbus device.
_____

## Using the library

To communicate with a modbus sensor or other modbus slave, first create a stream instance (ie, Serial1, SoftwareSerial, AltSoftSerial) and then an instance of the modbusMaster.

```cpp
// Create the stream instance
HardwareSerial modbusSerial = Serial1;  // ALWAYS use HardwareSerial if it's an option
// OR
// AltSoftSerial modbusSerial;  // AltSoftSerial should be your second choice, if your board is supported
// OR
// SoftwareSerial modbusSerial(txPin, rxPin);  // SoftwareSerial should be your last choice.

// Create the modbus instance
modbusMaster modbus;
```

Within the setup function begin both the serial instance and the modbusMaster instance.  The enable pin allows you to use an RS485 to TTL adapter that is half-duplex with a pin that enables data sending.

```cpp
// Start the stream
modbusSerial.begin(baudRate);

// start the modbus
modbus.begin(modbusSlaveID, modbusSerial, enablePin);
```

Once you've created and begun these, getting data from or adding data to a register is very simple:

```cpp
// Retrieve a 32-bit big endian float from input register 15 (input registers are called with 0x04)
modbus.float32FromRegister(0x04, 15, bigEndian);

// Write the value "56" to holding register 20 as a little-endian unsigned 16-bit integer
modbus.uint16ToRegister(20, 56, littleEndian);
```

There are also mid-level functions available to help to reduce serial traffic by calling many registers at once and low level functions to make raw modbus calls.  See SensorModbusMaster.h for all the avaialable functions.

The following data types are supported:
- uint16 (16-bit unsigned integer)
- int16 (16-bit signed integer)
- float32 (32-bit float)
- uint32 (32-bit unsigned integer)
- int32 (32-bit signed integer)
- TAI64 (64-bit timestamp, https://www.tai64.com/)
    - This is supported as if it were a 32-bit unix timestamp because the first 16-bits of the TAI64 timestamp will be 0x40000000 until the year 2106.
- byte (raw bytes of data)
- char (c++ style characters)
- String (Arduino Strings)
- pointer (pointers to other registers)
_____


## Notes on modbus maps
While modbus RTU specifications define the format of a data frame and a very simple data structure for a master and slave, there are no specification for what types of data a slave stores, where it is stored, or in what format it is stored.  You **MUST** get this information from the manufacture/programmer of your modbus device.  Typically this information is shared in what is called a modbus map.

You need the following data from your modbus map:
- the baud rate the device communicaes at (modbus allows any baud rate)
- the parity the device uses on the serial line (modbus technically allows 8O1, 8E1, and 8N2, though some devices may use 8N1)
- the type of register or coils the data you are interested is stored in (ie, holding register, input register, coil, or discrete input)
    - Note - This library does not currently support getting or setting values in coils.
- the register or coil number the data is stored in
- the format of data within the registers/coils (ie, float, integer, bitmask, ascii text)
- whether multi-register numeric data is stored as ["big-endian" or "little-endian"](https://en.wikipedia.org/wiki/Endianness) values (That is, is it high _word_ first or low _word_ first.  There are no specifications for this.)
- whether single-register data is stored "big-endian" or "little-endian" (That is, is it high _byte_ first or low _byte_ first.  Modbus specifies that it should be high byte first (big-endian), but devices vary.)
    - Note - This library only supports data that is "fully" big or little endian.  That is, data must be both high byte and high word first or both low byte and low word first.

Without this information, you have little hope of being able to communicate properly with the device.  You can use programs like [CAS modbus scanner](http://www.chipkin.com/cas-modbus-scanner/) to find a device if its address, baud rate, and parity are unknown, but it may take some time to make a connection.  You can also use the "scanRegisters" utility in this library to get a view of all the registers, but if you don't have a pretty good idea of what you are looking for that will not be as helpful as you might hope.
_____


## TTL and RS485/RS322
Again, while modbus RTU specifications define the format of a data frame transfered over a serial line, the type of serial signal is not defined.  Many modbus sensors communicate over [RS-485](https://en.wikipedia.org/wiki/RS-485).  To interface with them, you will need an RS485-to-TLL adapter. There are a number of RS485-to-TLL adapters available.  When shopping for one, be mindful of the logic level of the TLL output by the adapter.  The MAX485, one of the most popular adapters, has a 5V logic level in the TLL signal.  This will _fry_ any board that can only use on 3.3V logic.  You would need a voltage shifter in between the Mayfly and the MAX485 to make it work.  You will also need an interface board to communicate between an Arduino and any modbus sensor that communicates over [RS422](https://en.wikipedia.org/wiki/RS-422) or [RS232](https://en.wikipedia.org/wiki/RS-232).  Again, mind your voltages.
