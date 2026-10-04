# The Lampster - Bluetooth Control Protocol

I bought a so called Lampster a couple of years ago in a Kickstarter campaign and it finally arrived. The Lampster is a small scultpture with a lamp as its head.

![The Lampster](lampster.jpg)

The lamp itself if great, but the app to control the lampster is really bad in terms of usability. 

I reverse engineered the protocol so I can control the lampster using a Raspberry Pi and probably write my own smart home integration later.

## Prerequirement

To use these commands you need a linux computer with bluetooth and bluez (linux bluetooth stack) installed. If you use a raspberry pi 3 or raspberry pi 4 this should work out of the box.

## Find the device id of your lampster

```bash
bluetoothctl
scan on
exit
```

## Connect to to your lampster

```
gatttool -I -b XX:XX:XX:XX:XX:XX
connect
```

## Control your lampster

Once connected use the following commands.

### Mode selection

command | mode description
--- | ---
char-write-req 0x0021 c0 | turn it on
char-write-req 0x0021 40 | turn it off
char-write-req 0x0021 a8 | activate RGB mode
char-write-req 0x0021 c8 | activate WHITE mode
char-write-req 0x0021 28 | activate OFF mode?!?

### White LEDs

Once you are in WHITE mode you can adjust the warm white and cold white LEDs with this command:

command | description
--- | ---
char-write-cmd 0x0025 WWCC | WW for warm white LEDs, CC for cold white LEDs

The values seem to be Hex and range from 00 to 64 (0 to 100%)

command | description
--- | ---
char-write-cmd 0x0025 0100 | lowest warm white intensity
char-write-cmd 0x0025 6400 | highest warm white intensity
char-write-cmd 0x0025 0001 | lowest cold white intensity
char-write-cmd 0x0025 0064 | highest cold white intensity
char-write-cmd 0x0025 1E1E | both 30% intensity
char-write-cmd 0x0025 6464 | both full intensity

### RGB LEDs

Once you are in RGB mode you can adjust the colors with this command:

command | description
--- | ---
char-write-cmd 0x002a RRGGBB | RR for red LEDs, GG for green LEDs, BB for blue LEDs

The values seem to be Hex and range from 00 to 64 (0 to 100%)

command | description
--- | ---
char-write-cmd 0x002a 010000 | RED lowest intensity
char-write-cmd 0x002a 640000 | RED highest intensity
char-write-cmd 0x002a 000100 | GREEN lowest intensity
char-write-cmd 0x002a 006400 | GREEN highest intensity
char-write-cmd 0x002a 000001 | BLUE lowest intensity
char-write-cmd 0x002a 000064 | BLUE highest intensity
char-write-cmd 0x002a 1E1E1E | all 30% intensity
char-write-cmd 0x002a 646464 | all full intensity

## More details

Found while building a Home Assistant integration, by reading the values and notifications the lamp sends. Observed on a Lampster with model "LA-2017B", firmware "10" and hardware "100B".

### The mode value is a bit field

Bit 0x80 is power and bit 0x20 selects RGB (otherwise white), so 0x28 is "RGB mode, switched off".

value | meaning
--- | ---
c8 | white mode, on
a8 | RGB mode, on
48 | white mode, off
28 | RGB mode, off
c0 / 40 | on / off without a mode bit

- Writing c0 while the lamp is already on is rejected (Write Not Permitted).
- Mode writes are also rejected while a previous mode change is still being processed; trying again after about half a second works.

### Reading and notifications

- The mode, white and RGB values can all be read (char-read-hnd), including while the lamp is off.
- All three also support notifications. The lamp notifies changes made with the touch button, but not changes written over Bluetooth.
- When the touch button changes the mode, only the mode is notified, not the white or RGB values.

### Touch button

- A short tap turns the lamp off by clearing only the power bit (c8 to 48, a8 to 28). The next tap sets it again, so the lamp comes back in the same mode with the same values.
- A slightly longer press is the start of a hold. In RGB mode a hold first switches to white (c8).
- Holding the button changes both white values by one step at a time, about ten steps per second, and each hold goes the other way from the last. Dimming stops when the lower of the two values reaches 1; brightening stops when the higher one reaches 100. Both values move by the same amount, so the color temperature shifts during a hold.

### Other characteristics

UUIDs as listed by bleak on Windows. Its handles are one lower than the ones gatttool uses above (0x0020, 0x0024 and 0x0029 for mode, white and RGB).

UUID | properties | content
--- | --- | ---
01ff5554-ba5e-f4ee-5ca1-eb1e5e4b1ce0 | read, write, notify | mode
01ff5556-ba5e-f4ee-5ca1-eb1e5e4b1ce0 | read, write, notify | white (WWCC)
01ff5559-ba5e-f4ee-5ca1-eb1e5e4b1ce0 | read, write, notify | RGB (RRGGBB)
00002a6e-0000-1000-8000-00805f9b34fb | read, notify | temperature, signed 16 bit, 0.01 °C, notified about every 5 seconds
00002a26 / 00002a27 | read | firmware and hardware revision
01ff5560-ba5e-f4ee-5ca1-eb1e5e4b1ce0 | read, notify | unknown, reads about 3300 (perhaps a voltage in mV)
01ff5561 / 01ff5562 | read, write, notify | unknown, 4 bytes, zero
01ff5563 | read, write | unknown, 1 byte
01ff5551 / 01ff5552 (service 01ff5550) | indicate and write / write without response | unknown
