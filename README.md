# OpenFL
OpenFL is the Formlabs an API for interfacing with the Formlabs Form 1 and Form 1+ 3D printers, giving full control over printing and laser moves. We also include a version of [PreForm](http://formlabs.com/products/preform/) that allows custom material files for Form 1 and Form 1+.

[2016 Symposium on Computational Fabrication presentation](https://www.mattkeeter.com/research/openfl.pdf)

Contact: ben@formlabs.com matt@formlabs.com

# Quickstart
## PreForm
In order to use all of the firmware features and to set custom material files for Form 1/1+, you need a special version of PreForm, available here:
* https://s3.amazonaws.com/FormlabsReleases/Release/2.3.3/PreForm_2.3.3_release_OpenFL_build_1.dmg
* https://s3.amazonaws.com/FormlabsReleases/Release/2.3.3/PreForm_setup_2.3.3_release_OpenFL_build_1.exe

Next, you can load the custom material file, [Form_1+_FLGPCL02_100.ini](Form_1+_FLGPCL02_100.ini) from the PreForm UI and print with it.

To use all of the Python features described below, you need to update the firmware using this PreForm build.

## Python tools
To install dependencies, run
```
pip install -r requirements.txt
```

Then, have a look through the `examples` subfolder.


# Examples
## Inspecting a print
To inspect a print that has been sent to the printer, close PreForm and open a Python shell:
```
>>> from OpenFL import Printer
>>> p = Printer.Printer()
>>> flp = p.read_block_flp(0) # Read layer 0
>>> len(flp) # How many packets are there in the layer?
18796
>>> for packet in flp[:10]: print packet
... 
0x12 TimeRemaining 9546
0x02 XYMoveClockRate 60000
0x01 LaserPowerLevel 0
0x05 ZCurrent 80
0x04 ZFeedRate 6000
0x03 ZMove 78347
0x0a WaitForMovesToComplete 
0x04 ZFeedRate 6000
0x03 ZMove -400
0x0a WaitForMovesToComplete 
>>> for packet in flp[1000:1010]: print repr(packet) # Look further into the layer where it's filling:
... 
<XYMove(12 points) at 0x10615f440>
<LaserPowerLevel(0) at 0x105e291d0>
<XYMove(1 points) at 0x10615f488>
<LaserPowerLevel(48282) at 0x105e29290>
<XYMove(12 points) at 0x10615f4d0>
<LaserPowerLevel(0) at 0x105e29310>
<XYMove(1 points) at 0x10615f518>
<LaserPowerLevel(48282) at 0x105e293d0>
<XYMove(12 points) at 0x10615f560>
<LaserPowerLevel(0) at 0x105e29410>
>>> for packet in flp[-5:]: print repr(packet) # The last five packets of the layer.
... 
<LaserPowerLevel(41857) at 0x10703b750>
<XYMove(17 points) at 0x107034f38>
<LaserPowerLevel(0) at 0x10703b790>
<XYMove(2 points) at 0x107034f80>
<Dwell(1000 ms) at 0x10703b7d0>
>>> 
```

## Embedding things in a print
You can have a print stop part-way through, lift up, and wait for button press.
In this case, we uploaded a 16-layer 200 µm/lyaer print from PreForm to the printer, then closed PreForm and ran thefollowing Python code:

```
from OpenFL import FLP
from OpenFL import Printer
from examples.insert_material_swaps import insert_pause_before

p = Printer.Printer() # Connect to the printer
layer_i = 8
flp = p.read_block_flp(layer_i)
flp = insert_pause_before(flp, zJog_mm=150.0 - 0.2*i)
# Overexpose the next layer w/ 6 more copies of the laser moves:
flp += [laser for laser in flp 
        if isinstance(laser, FLP.LaserCommand)] * 6
p.write_block_flp(layer_i, x) # Send it back to the printer
p.start_printing(0, 16) # Print!
```


# Serial Output Commands
OpenFL provides commands for bidirectional communication with the printer while it is printing.

> **Caution**: With any covers removed, a Formlabs printer is no longer a Class 1 laser device. See [NOTICE.md](NOTICE.md).

The J22 header of the board, next to the USB plug, can be connected to an FTDI TTL-232R-3V3 six-pin cable. The end labeled "G" on the board is ground (black), not to be confused with the green wire that's at the other end of the plug. This serial port is at 115200 baud, so can be listened to with, e.g.,

<img src="Form_1+_pinout.png" width="500" alt="Serial and wait-on-pin headers">

This serial port is at 115200 baud, so can be listened to with, e.g., 
```
$ python -m serial.tools.miniterm /dev/tty.usbserial-AL009TJ4 115200
```
The serial port will show a number of status messages as the machine boots, for example. Most interesting, though, are the two FLP commands which allow programmed output. One writes strings to the serial port; the other writes the current Form 1/1+ system clock time (in ms). For example:
```
>>> from OpenFL import Printer, FLP
>>> p=Printer.Printer()
>>> p.write_block(0, FLP.Packets([FLP.SerialPrintClockCommand(),
                                  FLP.SerialPrintCommand('test\n'),
                                  FLP.SerialPrintClockCommand(),
                                  FLP.Dwell(s=1),
                                  FLP.SerialPrintClockCommand()]))
>>> p.start_printing(0, 1)
```
results in the following output on the serial line:
```
clock: 190845
test
clock: 190846
clock: 191848
```
Note that printing `'test\n'` took about 1 ms whereas the `FLP.Dwell(ms=1000)` command took 1002 ms.

Along with `FLP.SerialPrintClockCommand` there is `FLP.NopCommand`, which also holds a string; it does nothing but can be used to put markers or other metadata in FLP files.

# Mid-print input
## `FLP.WaitButtonPress`
`FLP.WaitButtonPress` displays a message and pauses the print until the user presses the button.

## `FLP.WaitOnPin`
`FLP.WaitOnPin` allows the printer to synchronize with other electronics: When the printer gets to a `FLP.WaitOnPin` packet, it pauses until a pin is pulled to ground. With this and `FLP.SerialPrintCommand`, a Form 1/1+ can be synchronized with another computer. For example, `FLP.SerialPrintCommand` could tell a computer to execute motor moves with additional motors, then when they complete, the computer can signal the printer to continue.

> **Caution**: With any covers removed, a Formlabs printer is no longer a Class 1 laser device. See [NOTICE.md](NOTICE.md).

The physical pin is the center pin of J13, the three-pin header labeled CAL next to the serial CONSOLE header. Shorting that center pin of J13 to ground (e.g., the J13 pin toward the back of the printer, toward the CONSOLE header) triggers `FLP.WaitOnPin` to continue.

# LEGAL DISCLAIMER
SEE [NOTICE FILE](NOTICE.md).

# Copyright
Copyright 2016 Formlabs

Released under the [Apache License](https://github.com/formlabs/openfl/blob/master/COPYING).
