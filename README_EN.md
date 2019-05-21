# Set ThinkPad T560 HAP bit

![Banner](/Images/banner.jpg)
<p align="right">
<a href="https://github.com/patbec/ThinkPad-T560-HAP-Bit/blob/master/README.md">Diese Seite auf Deutsch lesen</a>
</p>

## Description

Here a small tutorial how to completely deactivate the Intel Management Engine (ME)
on a ThinkPad with Skylake CPU. The whole thing is made possible by setting the
HAP bit and requires flashing the EEPROM with a modified firmware.
This tutorial explains how to use a ThinkPad T560 (`20FH0023GE`) with the **ME 11.6** firmware.

### Preparation

We need an external device to read the EEPROM and rewrite it with a different
firmware. The laptop CPU itself has neither read nor write access to the required flash
areas

Hardware
- WINGONEER EEPROM Routing USB Programmer **CH341A Writer** LCD Flash for 25 SPI Serie 24 I2C
- WINGONEER **SOIC8** SOP8 Test Clip For EEPROM 93CXX / 25CXX / 24Cxx

To use the software you need a Linux based operating system *like Ubuntu*. It is recommended to do this on a 2nd laptop, if you have Windows installed on your
device there are the following alternatives:
- Create a Ubuntu VM with VirtualBox and pass the CH341A Writer through
- Using a Live Linux
- Buy a RaspberryPi

## The execution

The execution is divided into 3 steps:
1. Updating the Laptop Firmware
2. Install the required software
3. Create the modified firmware and describe the EEPROM

### Step 1

First the laptop must be brought up to date:

Download ME firmware files, available at the official Lenovo website:
https://pcsupport.lenovo.com/de/de/downloads/ds112240


This package also contains the `MEInfoWin Tool` to display the current status of the Management Engine.
After downloading, start the application on your laptop and **perform the update**.

This command can be used to display the current status of the Management Engine:
```
cmd /k "C:\DRIVERS\WIN\ME\MEInfoWin.exe -verbose"
```
Example output:
```
Intel(R) MEInfo Version: 11.6.29.3287
Copyright(C) 2005 - 2017, Intel Corporation. All rights reserved.




Windows OS Version : 7.0

...

  CurrentState:                               Normal
  ManufacturingMode:                          Disabled
  FlashPartition:                             Valid
  OperationalState:                           CM0 with UMA
  InitComplete:                               Complete
  BUPLoadState:                               Success
  ErrorCode:                                  No Error
  ModeOfOperation:                            Normal
  SPI Flash Log:                              Not Present
  Phase:                                      ROM/Preboot
  ICC:                                        Valid OEM data, ICC programmed
  ME File System Corrupted:                   No
  PhaseStatus:                                AFTER_SRAM_INIT
  FPF and ME Config Status:                   Match
```

As expected, the state `Normal` is  output under `CurrentState`.

Now boot into the BIOS and ** activate the service mode**,  which temporarily deactivates the internal battery. It will be deactivated the next time it is started up.
```
F1 -> Config -> Power -> Disable Built-in Battery
```

Unscrew the laptop and search for the EEPROM.

![ThinkPad BIOS](/Images/thinkpad-01.jpg)
At https://www.lenovoservicetraining.com find instructions and videos for the respective model.

### Step 2
Once the EEPROM has been found, you are ready to go: Connect the CH341A Writer
to your laptop / desktop PC and install flashrom:
```
sudo apt-get install flashrom
```

Create a folder we will work with
```
# Create working directory and change to it
mkdir ThinkPad-T560 && cd ThinkPad-T560
```

Download the me_cleaner repository
```
git clone https://github.com/corna/me_cleaner
```

Python is **already included**, in a standard Ubuntu installation, if not it can be installed with the following command:
```
sudo apt-get install python
sudo apt-get install python-pip
```

### Step 3

Connect the SOIC8 clip to the EEPROM.

![SOIC8 Clip](/Images/thinkpad-02.jpg)
![ThinkPad Overview](/Images/thinkpad-03.jpg)

The command `flashrom -p ch341a_spi` can be used to check whether the clip is correctly seated and the EEPROM is recognized.
```
sudo flashrom -p ch341a_spi
flashrom v0.9.9-rc1-r1942 on Linux 4.10.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.
No operations were specified.
```

Read out the EEPROM and export the original firmware into the working directory:
```
sudo flashrom -p ch341a_spi -r factory_ME_11.6_Consumer_T560.bin
flashrom v0.9.9-rc1-r1942 on Linux 4.10.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.
Reading flash... done.
```

Set the HAP bit, here there are 2 different possibilities:
> me_cleaner sets this HAP/AltMeDisable bit when the -s (enable only the kill-switch, but don't remove the extra code from the firmware) or the -S (enable the kill-switch and remove the extra code from the firmware) are passed.

We will ** only activate the kill-switch** and save the edited firmware after `patched_ME_11.6_Consumer_T560.bin`.
```
sudo me_cleaner/me_cleaner.py -s factory_ME_11.6_Consumer_T560.bin -O patched_ME_11.6_Consumer_T560.bin
Full image detected
The ME/TXE region goes from 0x3000 to 0x700000
Found FPT header at 0x3010
Found 13 partition(s)
Found FTPR header: FTPR partition spans from 0x4000 to 0x133000
Found FTPR manifest at 0x4448
ME/TXE firmware version 11.6.29.3287
Setting the HAP bit in PCHSTRP0 to disable Intel ME...
Checking the FTPR RSA signature... VALID
Done! Good luck!
```

Install the modified firmware:
```
sudo flashrom -p ch341a_spi -w patched_ME_11.6_Consumer_T560.bin
flashrom v0.9.9-rc1-r1942 on Linux 4.10.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```

Finally, adjust the permissions of the two firmware files using `chown`:
```
# Example
chown -R USERNAME:GROUPNAME /PATH/TO/FILE

# The users group should be present on most Linux distributions.
chown -R $USER:users factory_ME_11.6_Consumer_T560.bin
chown -R $USER:users patched_ME_11.6_Consumer_T560.bin
```

At the first start an error message can occur that the BIOS settings cannot be loaded.

To check if ME is disabled boot into BIOS:
```
F1 -> Config -> Intel(R) AMT
```
The menu item `Intel (R) AMT Control` l should now be displayed with `[Permanently Disabled]` and should **no** longer be selectable.


`MEInfoWin` can also be used to check whether the Management Engine is deactivated:
```
cmd /k "C:\DRIVERS\WIN\ME\MEInfoWin.exe -verbose"
```

The output should now look like this:
```
  CurrentState:                               Disabled
  ManufacturingMode:                          Disabled
  FlashPartition:                             Valid
  OperationalState:                           Transitioning
  InitComplete:                               Initializing
  BUPLoadState:                               Success
  ErrorCode:                                  Disabled
  ModeOfOperation:                            Alt Disable Mode
  SPI Flash Log:                              Not Present
  Phase:                                      BringUp
  ICC:                                        Valid OEM data, ICC programmed
  ME File System Corrupted:                   No
  PhaseStatus:                                UNKNOWN
  FPF and ME Config Status:                   Match
```

**Congratulations!**


## Quellen
- [HAP-Bit](http://blog.ptsecurity.com/2017/08/disabling-intel-me.html)
- [me_cleaner](https://github.com/corna/me_cleaner)
- [Intel ME 11.x Firmware Images Unpacker](https://github.com/ptresearch/unME11)
- [Intel Banner](https://www.bleepingcomputer.com/news/hardware/intel-fixes-9-year-old-cpu-flaw-that-allows-remote-code-execution)

## Autor

* **Patrick Becker** - [GitHub](https://github.com/patbec)

E-Mail: [github@bec-wolke.de](mailto:github@bec-wolke.de)

## tl;dr
```
# Install flashrom
sudo apt-get install flashrom

# Create working directory
mkdir ThinkPad-T560 && cd ThinkPad-T560

# Download me_cleaner
git clone https://github.com/corna/me_cleaner

# Connect SOIC8 clip to the EEPROM and check if flashrom recognizes the EEPROM
flashrom -p ch341a_spi

# Read out EEPROM and save the firmware
sudo flashrom -p ch341a_spi -r factory_ME_11.6_Consumer_T560.bin

# Set HAP bit
sudo me_cleaner/me_cleaner.py -s factory_ME_11.6_Consumer_T560.bin -O patched_ME_11.6_Consumer_T560.bin

# Install the modified firmware
sudo flashrom -p ch341a_spi -w patched_ME_11.6_Consumer_T560.bin

# Adapt rights
chown -R $USER:users factory_ME_11.6_Consumer_T560.bin
chown -R $USER:users patched_ME_11.6_Consumer_T560.bin
```

Check if ME is deactivated, boot into BIOS:
```
F1 -> Config -> Intel(R) AMT
```
`AMT Control` should now be displayed with `[Permanently Disabled]` and should **no** longer be selectable.

**Congratulations!**
