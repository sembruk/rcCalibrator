16/6/2017. Allow some days to finish the github page before download.

# OsccalCalibrator
Calibration of the internal RC oscillator of atmega328p chip, and OSCCAL aware UART bootloader(ATmegaBOOT).

### The problem
Most of the projects using AVR have a crystal or resontator connected to pins XTAL1 and XTAL2.
Using FUSE settings the MCU can use its internal RC oscillator. The problem is however that
the RC oscillator is not very well calibrated. At least for atmega328p can deviate up to
10% from the 8Mhz standard frequency(usually 0-3%). This is unacceptable for example for
UART communications which tolerate up to ~2-3% error.
The AVR microcontrollers have a register called OSCCAL (Oscillator Calibration) which can
be used to drift the RC frequency and reduse the factory erron.

### The purposeof this project
- to find the optimal OSCCAL value
- to provide a mechanism via the "osccal" utility to automatically build bootloader and
application code capable of fixing the RC frequency. The prinary purpose of this is to
allow UART communications

### UART complications
As if the RC high error margin wasnt enough. The use of UART communications introduces
additional error, because the 8MHz frequency is not divided exactly with the standard
Serial bitrates. See [wormfood tables](http://wormfood.net/avrbaudcalc.php)

**@8MHz**
38400 +0.2%
57600 +2.1%   The proMinis are marginally capable of this speed
115200 -3.5%  This is the reason ProMini@8Mhz cannot do 115.2K


This line is also begins a separate paragraph, but...
This line is only separated by a single newline, so it's a separate line in the same paragraph.

### UART bootloader even more complications
The use of a UART bootloader and at the same time using the internal RC
oscillator, is a "chicken and egg" problem. Suppose we know that the optimal
OSCCAL value for a spacific atmega chip is 139 : We write an arduino application
and right after setup() we write
OSCCAL=139;
Seems good but it will not work (reliably) :
The bootloader starts before the application. If the chip happens to be
baddly factory calibrated, we will not be able to upload any code to the chip.
The solution provided here is simple and I believe very robust. "osccal"
utility finds the  correct OSCCAL value and then the (modified)ATmegaBOOT is compiled
aginst this specific OSCCAL value. Then it is uploaded to the chip.

### Reasons to use an external crystal
Generally whenever you need better accuracy than the
When the question arises **"How to calibrate the internal avr oscillator"**

the usual answer is **"Use a crystal with 2 (22pF)caps and avoid all the trouble."**

In many cases this is true.

### Why use the internal oscillator

However there are some (many in my opinion) cases, where the Internal RC
oscillator should/must be used.
- Fewer parts on the breadboard/PCB. Less clutter and more reliable. This
is usually the first reason that comes in mind, but it is also the least importand.
A crystal is usually a tiny part of the complexity and the cost of
a project. For simple projects is fine however, if we can do without a crystal.
- Ability to change the frequency at runtime. For example we can drift
the 8Mhz frequency -2.1% for extremely reliable 57600 UART communication and
drift it +3.5% for 115200. Of course we can calibrate the RC oscillator to
the UART friendly 7.37(28) Mhz frequency. Note however that if you write
Arduino code, better use 8Mhz (and 16Mhz if use a crystal) only. A lot of
useful Arduino functions like millis() work correctly only for 8Mhz and 16Mhz
- You have 2 additional GPIO pins. The XTAL1 and XTAL2 can be used for any purpose.
A high voltage programmer for example, needs a lot of GPIO pins and 2 more pins
can make the difference.
- A lot of projects don't need any accuracy of RC oscillator. If you upload your code
using a UART bootloader however you need it.
- **Much faster startup from sleep mode.** This is the reason this project exists.
I have another project where the MCU is in sleep, and wakes up from an incoming
 SMS. Here is the message it receives when it uses a crystal

**"*%&%^$^&%*&%&*%^*&^456 ...."**

The crystal needs a lot of time to stabilize it's freequency. As you can
see were able to get only the last 3 digits of the incoming phone number.
Here is the message when the (calibrated) RC oscillator is in use.

**"CMT+ ...."**

This time we did't lose a single character.

### How this project can be used

There are multiple strategies:
- To spot the "good" atmegas and use them on UART applications. This of
course works only if you have a lot of atmegas and only some of them
need to be calibrated. This method has the advandage that is simple and no
modification of existing code is needed. If you need 57600 speed this
method is unreliable however : see wormfood table. 57.6k at 8Mhz introduces a +2.1% error,
even on a perfect 8Mhz crystal. The 115200 is even worse : -3.5% error. The
maximum reliable speed is 38400 (introduces only 0.2% error by itself)
- To be used with a custom bootloader who sets the OSCCAL register at
startup. I have slighty modified the ATmegaBOOT (used in
Arduino proMini) to do exactly this. It sets the speed at about -2% (Reduces OSCCAL register by 4) of
the optimal 8Mhz value, to make 57600 upload very reliable, and just
before the application code starts, sets the OSCCAL to the optimal
value for 8Mhz frequency, Although it can easily support 115200bps
by drifting the speed to +3.5% of the optimal (see note above), I
wanted it to be interchangeable with ProMini 3.3V  so you dont need to
define/use a new board in the Arduino build system. "Standard is better than
better"

### Build Instructions
- You need a usbasp ISP programmer **Picture** .I recommend to use a module with 3.3V/5V option and turn
it to the voltage you are going to run the atmega328 after the calibration.
Probably the voltage will be 3.3V as we talk about a 8Mhz system.
- a ZIF developer board and
- a DS3231 RTC module. You
- also need some female-female dupont 2.54 cables, better to be sort. If
you need visual feedback (I recommend it at least for the first steps)
you need also a 16x2 LCD and a i2c adapter, and to solder the secondary
i2c header of the DS3231 module. A complication here is that the LCD modules
come as 5V and 3.3V versions. Probably you need the 3.3V as mentioned above.

Developer Board | Rtc | LCD (if you use it)
---|---|---
+/VCC  | VCC | VCC
-/GND  | GND | GND
PDX(Arduino A4)  |   SDA | SDA
PDX(Arduino A5)  |   SCL | SCL
A3   |   SQW |   -

WARNING: if you are unfamiliar with ISP programmers and how to reprogram
a atmega328 chip do not go further.

How it works. The following instructions are for the linux command line.
I suppose they can be adapted for windows, but i didn't test it.
When you download the github page
git clone .......
there is a python scrip "osscal".
Make it executable chmod +x OsccalCalibrator/osccal. Optionally put it in your PATH.
connect the usbasp programmer (with the RTC) to a USB port
attach a atmega chip (WARNING whatever it has in the FLASH, it will be erased, including the
bootloader) and do a

```sh
> path/osccal
```

After a few seconds you will see the optimal osccal value in the command
line. If you have the LCD "model" you will also see the results there.
So from the perspective of the software, "osccal" is a command witch
 gives a number as a result (The best OSCCAL value = the closer to 8Mhz)
If your intention is to just find some "good" MCUs then this is the end of
the story.

### Why to use the MCU with a modified UART bootloader.

There are some pages around, that give instructions to use
an uncalibrated atmega328p with a 38400 bootloader. This is VERY
unreliable, as a lot of chips come from the factory with clock errors
far worse than 2%. It is also non standard and require a custom board definition.
The ATmegaBOOT Makefile included here, uses the "osccal" utility
to find the correct OSCCAL value. Then it compiles
the ATmegaBOOT with this value and then writes it to the atmega328p chip. The chip
can be used then just like a proMini to upload code with 57600bps.
According to my tests the upload process is bulletproof. This is to be expected because the
frequency is calibrated.

### Modified AtmegaBOOT installation
The process is quite automatic. Go to the folder where you downloaded OsccalCalibrator

```sh
> cd atmegaBOOT
# The Makefile uses "osccal" utility to find the optimal OSCCAL value
> make isp
```

wait a few seconds ... ready !

It is important to note that the bootloader does NOT use any predefined
EEPROM location to read the OSCCAL. This avoids the danger to accidentally erase
the EEPROM by the application (to store some data), with probably catastrophic results
for the project. The bootloader is recompiled for every
new chip and the OSCCAL value is saved in the bootloader
area and is unique for this chip. This is the reason there is no precompiled HEX for this
bootloader.

### rfboot
I have written the bootloader rfboot(link) which can (optionally) set the optimal OSCCAL before
jump to the application. The bootloader does not need any OSCCAL calibration to work (it uses SPI),
but any UART device the project is using, might need it. As usual the 38400bps is the safest option.

### applications without bootloader
You have to modify the Makefile of your project to use the "osscal" as a shell command and
get the best OSCCAL value before burn the application code. Somewhere inside the application
just after main() or setup() and before Serial setup put a

```C++
OSCCAL = CALIBRATED_OSCAL_VALUE;
```

CALIBRATED_OSCAL_VALUE must be passed to the gcc by the Makefile.
See the Makefile of the ATmegaBOOT bootloader, included in this site

### Alternative method. Read the OSCCAL value from the EEPROM
When "osscal" utility runs, it installs the calibrator.hex file to the MCU
In turn this utility calculates the OSCCAL using the RTC module as clock reference, and
saves the OSCCAL value as:

EEPROM byte 0: 0x05

EEPROM byte 1: OSCCAL

EEPROM byte 2: 255-OSCCAL

EEPROM byte 3: 0x05

So it is possible that the application reads the EEPROM and uses the OSCCAL value provided.
However I find the method quite fragile. A programming mistace can overwrite the contents
of the EEPROM.
