# Flashing Coreboot on the ThinkPad T420

I have a more detailed explanation on the process of flashing coreboot on [my website](https://nenadstoisavljevic.xyz/guides/coreboot). This repository is mostly just plain commands for what you would run, I find this easier to follow anyhow.

Please keep in mind that there is always a possibility that you could brick your ThinkPad T420 when trying to flash coreboot. By connecting the Pomona 5250 clip to the BIOS chip, you are essentially connecting an external power supply which could cause a lot of things to go wrong, although it is highly unlikely.

*Be sure to back up your old BIOS so that you can re-flash it if something were to go wrong.*

## Setup Raspberry Pi

#### Enable SPI Interface

In order to read/write your BIOS chip, you will need to enable certain kernel modules.

```sh
sudo raspi-config

# Select "[5] Interfacing Options"
# Then select "P4 SPI" and "Yes" to enable it
# Optionally enable "P2 SSH" for running the pi headless
# Do the same for "P5 I2C" and enable it
# Also enable "P8 Remote GPIO" if you are using SSH
sudo reboot
```

## Prepare Computer for Building Coreboot

#### Update packages index

```sh
sudo apt update
```

#### Install dependencies needed to build coreboot

```sh
# For a Debian-based system
sudo apt install git build-essential gnat flex bison libncurses5-dev wget zlib1g-dev

# For an Arch-based system
sudo pacman -S base-devel gcc-ada flex bison ncurses wget zlib git
```

#### Create a directory to work in

```sh
mkdir -p ~/work/roms
cd ~/work
```

#### Clone the latest version of me_cleaner repo

```sh
git clone https://github.com/corna/me_cleaner
```

#### Clone coreboot repo and update submodules

```sh
git clone https://review.coreboot.org/coreboot
cd ~/work/coreboot
git submodule update --init --checkout
```

#### Create directory for binary blobs

```sh
mkdir -p ~/work/coreboot/3rdparty/blobs/mainboard/lenovo/t420
```

#### Build and compile ifdtool

```sh
cd ~/work/coreboot/utils/ifdtool
make
```

## Pinout

Carefully wire up the Pomona 5250 clip to the Raspberry Pi by using female to female jumper wires. Here is a [pinout image](pinout.png).

You will need six female to female jumpers wires for /CS, DO, GND, DI, CLK, and VCC.

*Note: VCC should be 3.3 volts.*

## Read the Flash Chip

#### Create a roms directory and install flashrom

```sh
mkdir -p ~/work/roms
cd ~/work/roms

sudo apt install flashrom
```

#### Set SPI speeds and find flash chip name

```sh
# Lower spispeeds will provide you with better reads. However,
# you can increase `spispeed=128` to a higher value such as
# 512 for faster reads. An spispeed of 128 takes roughly
# 10 min per read.

# This will probe the chip and multiple chips may be found.
# You can choose any one of them as long as their checksums
# match later on, when you go to compare them.
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=128
```

#### Read the factory BIOS off the chip

```sh
# Flashrom may have detected a Winbond chip or a Macronix chip, depending on the chip.
# Use the -c option to specify which chip you would like to read off of.

sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=128 -c <chip name> -r factory1.rom
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=128 -c <chip name> -r factory2.rom
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=128 -c <chip name> -r factory3.rom
```

#### Compare checksums

```sh
# Very important, make sure all the checksums are the same!
# If for whatever reason they are not, head on over to
# https://www.coreboot.org/Board:lenovo/t420 for troubleshooting.
# DO NOT CONTINUE until they all match.

sha512sum factory*.rom
```

## Binary Blobs

#### Clean the Intel ME by running me_cleaner

Be sure to run `me_cleaner` with the `-S` option as this will enable the kill-switch and the HAP/AltMeDisable bit set, it will also remove unnecessary code from the firmware.

```sh
cd ~/work/roms
cp factory1.rom cleaned.rom

~/work/me_cleaner/me_cleaner.py -S cleaned.rom
```

The `factory1.rom` should not be edited, make sure that you back it up to **MULTIPLE** safe locations.

#### Extract binary blobs with ifdtool

```sh
~/work/coreboot/utils/ifdtool/ifdtool -x cleaned.rom
```

#### Rename/remove binary blobs

```sh
mv flashregion_0_flashdescriptor.bin descriptor.bin
rm flashregion_1_bios.bin
mv flashregion_2_intel_me.bin me.bin
mv flashregion_3_gbe.bin gbe.bin
```

#### Copy blobs to coreboot directory

```sh
cp descriptor.bin me.bin gbe.bin ~/work/coreboot/3rdparty/blobs/mainboard/lenovo/t420/
```

## Configure the Coreboot Rom

Compiling coreboot on the Raspberry Pi will take many hours, so consider copying the `~/work/coreboot` directory to a faster computer if you are not already using one. A faster machine will significantly reduce the amount of time it will take to build coreboot.

#### Configure coreboot

There are ways to compile coreboot without the VGA BIOS firmware, but if you would like to run Windows, SeaBIOS will require it. If you would like to extract it yourself, you can follow [my guide](https://nenadstoisavljevic.xyz/guides/coreboot) or you have the option to use a pre-extracted one instead.

```sh
# Run the following command to download the pre-exracted VGA BIOS firmware.
wget https://github.com/NenadStoisavljevic/t420-coreboot-guide/raw/master/snb_vbios_2170.rom

cd ~/work/coreboot
make nconfig

# Copy the coreboot configuration down below
General
  -*- Use CMOS for configuration values
  [*] Compress ramstage with LZMA
  [*] Include coreboot .config file into the ROM image
  [*] Allow use of binary-only repository
  [*] Add a bootsplash image
      (<PATH TO BOOTSPLASH IMAGE>) Bootsplash path and filename

Mainboard
  Mainboard vendor (Lenovo) --->
  Mainboard model (ThinkPad T420) --->
  ROM chip size (8192 KB (8 MB)) --->
  (0x100000) Size of CBFS filesystem in ROM

Chipset
  [*] Enable VMX for virtualization
  [*] Set IA32_FEATURE_CONTROL lock bit
      Include CPU microcode in CBFS (Generate from tree) --->
  [*] Use native raminit
  [*] Ignore vendor programmed fuses that limit max. DRAM frequency
  [*] Add Intel descriptor.bin file
      (3rdparty/blobs/mainboard/$(MAINBOARDDIR)/descriptor.bin) Path and filename
  [*] Add Intel ME/TXE firmware
      (3rdparty/blobs/mainboard/$(MAINBOARDDIR)/me.bin) Path to management engine firmware
  [*] Add gigabit ethernet firmware
      (3rdparty/blobs/mainboard/$(MAINBOARDDIR)/gbe.bin) Path to gigabit ethernet

Devices
      Graphics initialization (Run VGA Option ROMs) --->
  -*- Use onboard VGA as primary video device
  [*] Re-run VGA Option ROMs on S3 resume
      Option ROM execution type (Native mode) --->
  -*- Enable PCIe Common Clock
  -*- Enable PCIe ASPM
  [*] Enable PCIe Clock Power Management
  [*] Enable PCIe ASPM L1 SubState
  [*] Add a VGA BIOS image
      (<PATH TO VGA BIOS IMAGE>) VGA BIOS path and filename
      (8086,0126) VGA device PCI IDs
  [*] Add a Video Bios Table (VBT) binary to CBFS
      (src/mainboard/$(MAINBOARDDIR)/data.vbt) VBT binary path and filename

Generic Drivers
  [*] Support Intel PCI-e WiFi adapters
  [*] PS/2 keyboard init

Console
  [*] Squelch AP CPUs from early console
  [*] Show POST codes on the debug console

System Tables
  [*] Generate SMBIOS tables

Payload
      Add a payload (SeaBIOS) --->
      SeaBIOS version (master) --->
      (3000) PS/2 keyboard controller initialization timeout (milliseconds)
  [*] Hardware init during option ROM execution
      Payload compression algorithm (Use LZMA compression for payloads) --->
  [*] Use LZMA compression for secondary payloads
      Secondary Payloads --->
  [*] Load coreinfo as a secondary payload
  [*] Load Memtest86+ as a secondary payloada
  [*] Load nvramcui as a secondary payload
  [*] Load tint as a secondary payload
      Memtest86+ version (Stable) --->
```

#### Build coreboot

```sh
# Replace X with the number of threads your CPU has.
make crossgcc-i386 CPUS=X
make iasl
make

# The coreboot rom will be saved to ~/work/coreboot/build/coreboot.rom.
```

## Write Coreboot to Flash Chip

#### Change to roms directory

```sh
cd ~/work/roms
```

#### Probe flash chip to make sure it is detected

```sh
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=128
```

#### Write the coreboot rom to flash chip

```sh
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=128 -c <chip name> -w coreboot.rom

# If you get "Erasing and writing flash chip... FAILED" once, that is
# supposed to happen, nothing to worry about.
```

#### Conclusion

- After the write has been verified, you can proceed to power off the Raspberry Pi by running `sudo poweroff`.
- If you have not been successful, be sure to check your connections and attempt a re-flash.
- If that doesn't work, then try re-compiling coreboot and attempt another re-flash.
- Lastly, if all else FAILS you can always re-flash `factory1.rom` by using the same write command.

## Other Useful Information
- coreboot's [wiki page](https://www.coreboot.org/Board:lenovo/t420) for the official documentation on the process
- jkaye99's [guide](https://www.instructables.com/Lenovo-T420-Coreboot-WRaspberry-Pi/) has great instructions overall and teardown images
- tripcode!Q/7's [video](https://www.youtube.com/watch?v=znS-2umyBJs) for how to install IVYBRIDGE CPUs

## FAQ
- If you're getting "No EEPROM/flash device found" when trying to read from the flash chip, be sure that you've enabled SPI under the interface section. You can confirm this by running `ls /dev | grep spi` and that will list out the SPI devices to ensure they are properly connected. This could also occur if you didn't properly wire up the female to female jumper wires with the Raspberry Pi, so double-check your connections for that. Sometimes the kernel can give you issues as well. So download the latest version of Raspbian Lite and try following the tutorial from the beginning again.
- When you get `Microcode error: 3rdparty/blobs/cpu/intel/model_206ax/microcode.bin does not exist`, all you have to do is clone the 3rd party blobs. If `git submodule update --init --checkout` did not populate the `3rdparty/blobs` directory (which it should have), then you have to manually clone it yourself by running:

```sh
cd ~/work/coreboot/3rdparty
git clone https://review.coreboot.org/blobs
```
