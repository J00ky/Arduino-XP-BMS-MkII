Arduino-XP-BMS-MkII 
Rev. 1.0 - 2022-01-14
-------------------------------
A BMS for Valence XP batteries, designed to run on Arduino or similar hardware.
Merger of original Seb code and portions of Crelex's Valence Battery Reader code by Daren T.
https://github.com/J00ky/Arduino-XP-BMS-MkII

Original code by Seb Francis -> https://diysolarforum.com/members/seb303.13166/
https://github.com/seb303/Arduino-XP-BMS

Crelex's Valence Battery Reader code
https://github.com/Crelex/Valance-Battery-Reader

Overview
--------
The MkII version of the Arduino XP BMS has the following features:
* Automatically searches for battery modules and identifies their ID numbers.
* Identified modules are listed by ID#, model type, and serial number during initial communications.
* Compatible for Rev. 1 (black) and Rev. 2 (green) models.
* Any comms failure, including BMS disconnection, will result in BMS sending the wakeup command but battery system config info is retained for faster system recovery.
* "Poor man's balancing" - send a digital signal on pin 17 to the battery charger to switch into constant voltage mode. Signal is triggered by
  the following logic:
  1. monitor all module voltages and look for the module with the lowest voltage (this is the minimum module voltage)
  2. check for modules with cell voltages >3.28V AND total voltage is >100mV higher than the minimum module voltage
  2. trigger balancing on any module where voltage is >100mV higher than the minimum module voltage AND 
     the identified module has a min cell voltage >3.28V
  3. maintain balancing until minimum module voltage is within 100mV of the identified balancing module
  This method of balancing (setting CV mode) relies on the voltage relaxation effect, this is expected to be very slow.
  While this balancing mode is on, the "over voltage" LED will flash. 

Still designed to provide monitoring of Valence XP batteries in order to:
* Keep the Valence internal BMS awake so the intra-module balancing is active.
* Provide a signal to a charge controller to disable charging in case of individual cell over-voltage or over-temperature.
* Provide a signal to a load disconnect relay in case of individual cell under-voltage or over-temperature.
* Provide warning and shutdown status outputs for over-temperature, over-voltage, under-voltage and communication error.
* Provide basic event logging to EEPROM.
* Have a mode for long term storage / not in use, where it will let the batteries rest at a lower SOC.

Current Limitations
-------------------
* Does not handle true inter-battery balancing, so only suitable for parallel installations. Balancing code still needs debugging.
* Only up to 6 cell / 19V batteries.
* Low temperature checking code added but not implemented or validated.

Hardware
--------
This sketch has been primarily written for and tested on Teensy 3.2 hardware, but should run on any Arduino or similar board that
has a dedicated hardware serial port. In order to view the console output you'll need either native-USB support (e.g. Teensy) or
in the case of Arduino a board with multiple serial ports, such as the Mega or Due (since Arduino USB uses one of the serial ports).

The clock speed will need to be adequate for good serial timing at 115200 baud. Unless using a specific crystal which is an exact
multiple of 115200, a good rule of thumb would be to have a clock speed around 20Mhz or more to ensure good enough timing. This
does also depend on how the hardware is implemented - for example, the Teensy 3.2 has a high resolution baud rate for the hardware
UART, and so is particularly accurate. The exact serial timing error depends on the clock rate:  
At 24 MHz: -0.08%  <- plenty accurate enough, and uses the least power  
At 48 Mhz: +0.04%  
At 96 MHz: -0.02%  

Requires the following additional components:
* A 5V voltage regulator
* An external RS485 transceiver such as the MAX485 (if the MCU inputs are not 5V tolerant run the RO/RX through a potential divider)
* Some LEDs/resistors for the status display
* Something to convert the logic level Enable outputs to the voltage/current levels required for the charging & load control

Installation & Configuration
----------------------------
* Install Arduino IDE, and if using Teensy hardware: Teensyduino - https://www.pjrc.com/teensy/teensyduino.html
* Select board and clock rate (e.g. 24 MHz is plenty fast enough).
* Define the pin numbers where the outputs and RS485 driver are connected.
* Define other board-specific parameters, such as port for Serial Monitor, EEPROM size, etc.
* Configure the desired thresholds for voltage, temperature and SOC.  
  The default settings are quite conservative, chosen to maximise battery life rather than squeeze out every last Ah of
  capacity. The values used by the official Valence U-BMS are much less conservative, and are shown in the comments.
* This version will automatically populate the battery modules with their unique IDs.

Console interface
-----------------
```
Commands can be entered via the Serial Monitor.
help         - show available commands
debug 0      - turn off debugging output
debug 1      - debugging output shows errors, status changes and other occasional info
debug 2      - in addition to the above, debugging output shows continuous status and readings from batteries
debug 21     - show status and readings from batteries once, then switch to debug level 1
debug 2 <n>  - show status and readings from batteries every <n> seconds, otherwise as debug level 1
mode normal  - enter normal mode
mode storage - enter long term storage mode
log read     - read events log from EEPROM
log clear    - clear events log
reset cw     - resets CommsWarning status (otherwise this stays on once triggered)
```

EEPROM data
-----------
```
The top 32 bytes are reserved for storing settings persistently:
Byte 0: debug level (0, 1, 2)
Byte 1: mode (0 = normal, 1 = storage)

The rest of the EEPROM stores a 32 byte data packet whenever the status changes:

0  PO   0   0   ST  STC  EC  EL  OTW OTS OVW OVS UVW UVS CW  CS (uint16_t bitmap)
     PO is set for the first event after power on
     ST is set when storage mode is active
     STC is set when storage mode is active and charging (i.e. storageMinSOC has been reached)

Timestamp of event = number of seconds since power-on (uint32_t)

Values from the battery, only if the status change was triggered by a value from a specific battery:
Battery id (uint8_t)  (0 if no battery values)
V1 (int16_t)
V2 (int16_t)
V3 (int16_t)
V4 (int16_t)
T1 (int16_t)
T2 (int16_t)
T3 (int16_t)
T4 (int16_t)
PCBA (int16_t)
SOC (uint16_t)
CURRENT (int16_t)

3 bytes unused
```
License
-------
This sketch is released under GPLv3 and comes with no warranty or guarantees. Use at your own risk!
Libraries used in this sketch may have licenses that differ from the one governing this sketch.
Please consult the repositories of those libraries for information.

