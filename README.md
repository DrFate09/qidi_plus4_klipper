# QIDI Plus 4 - Kalico (or mainline Klipper) wtih Raspberry Pi host

# !!! DANGER - WARNING !!!
This is **work in progress**. Do not use any of these configs or instructions unless you know what you're doing!
These modifications are for experienced users. If you are not comfortable with a command line, linux, and electronics, please stop here!

**ALSO NOTE: YOUR OEM SCREEN WILL NOT WORK AFTER FOLLOWING THESE STEPS**

You could move to a KlipperScreen setup to get a functioning screen.

---

# Introduction
Flashing Kalico (or mainline Klipper) on the QIDI Plus 4 is relatively easy. It does require some effort to flash the toolhead MCU. You will need an ST-Link programmer or clone to flash the toolhead.
Alongside flashing the mainboard and toolhead MCUs, you will also need to be able to precicesly solder onto small PCB pads on the mainboard.

# Backup
Before proceeding, backup any and all data in your printer configs, particularly your `printer.cfg`, `gcode_macros.cfg` and any other files
you may want to save. This can be done via the Fluidd interface or by using the plugin [Klipper-Backup](https://klipperbackup.xyz/)

---

## Installing Kalico (or mainline Klipper), Moonraker, Fluidd/Mainsail and others

Starting with a Raspberry Pi 4 (or another other Linux box), we can start installing the software needed to run the printer. To begin, lets install KIAUH.

## Installing KIAUH
KIAUH is a helper script to install Klipper/Kalico, Mainsail, Fluidd, Crowsnest, Moonraker, and many other things you may need or want.
The following steps will get KIAUH installed (using SSH):
```
  cd ~
  git clone https://github.com/dw-0/kiauh.git
  ./kiauh/kiauh.sh
```
You will now be entered into the KIAUH main menu where you can install the software needed. At a minimum install:
* Klipper (or Kalico)
* Moonraker
* Fluidd (or Mainsail)

KIAUH should automatically install all the required modules and start the services for you.

---

# Flashing the main MCU

## Flashing Katapult Deployer
These steps will build the Katapult deployer application. Make sure that you already have your ST-LINK programmer (or clone) on hand. If something goes wrong,
you will have to recover with the programmer. [Further details of Katapult deployer can be found here.](https://github.com/Arksine/katapult?tab=readme-ov-file#katapult-deployer)

1. SSH into your printer
2. Download Katapult and configure for the main board:
    ```
    cd ~/
    git clone https://github.com/Arksine/katapult
    cd katapult
    make menuconfig
    ```
    Once you are in `make menuconfig`, you want to build katapult with the following options:
    <img width="960" height="540" alt="katapult" src="https://github.com/user-attachments/assets/aaeb641d-18ff-4294-94c4-5ff8477b9edf" />

    After you are sure that your menuconfig matches the above settings, you can quit menuconfig (press q then y to save)
3. run `make -j4` to build katapult
4. Copy `~/katapult/out/deployer.bin` to your computer via your favorite method (scp, sftp, or using the fluidd interface)
5. On your computer, format a microSD card as FAT32
6. Copy the `deployer.bin` file to the microSD card and rename it `qd_mcu.bin`
7. Eject the microSD card and plug it into the microSD card slot on the printer.
8. Once the card is inserted, find the button labeled "RESET" on the main board and press it
9. Wait 30-60 seconds
    - You can verify that the flash worked by checking the file on the microSD card - it should be renamed to `qd_mcu.CUR`
10. Eject the microSD card. Katapult should now be flashed to the main MCU.

## Flashing Klipper on the main MCU

1. SSH into your printer
2. Install the `pyserial` Python package. This package is needed to run the katapult flashtool script.
   ```
   python -m pip install pyserial
   ```
4. Execute the following to build Klipper
    ```
    cd ~/klipper
    make menuconfig
    ```
    Configure the make arguments in `menuconfig` to match these settings:
    <img width="960" height="540" alt="klipper" src="https://github.com/user-attachments/assets/2ba1dc97-5d31-4b86-82ac-5377cb8c026f" />

5. Save and quit by pressing `q` then `y` to save
6. Build klipper by running `make clean; make -j4`
7. Prepare to flash. Double-click the "RESET" button on the board to load katapult. The button must be pressed twice within 500ms.
8. Flash via katapult by doing the following:
    ```
    cd ~/katapult/scripts
    python3 flashtool.py -b 500000 -d /dev/ttyS0 -f ~/klipper/out/klipper.bin
    ```
9. Klipper is now flashed to the main MCU. We can now move on to the toolhead.

---

# Flashing the toolhead
The toolhead takes quite a bit more work to flash. You will need to solder some pins on the board and you will also need an ST-Link programmer (or clone).
A clone ST-LINK programmer is what I used. You can purchase one from Amazon (or any other source). [This is the one I used](https://www.amazon.com/dp/B07SQV6VLZ)

## Preparing the toolhead
Begin by unsliding the back cover of the toolhead, disconnecting all connectors, and unscrewing the board from the toolhead.

Next, you need to solder a 6 pin header on the board like the below image:
![toolhead_with_pins](https://github.com/user-attachments/assets/14d5968c-32c2-4e1f-9a2e-761f0ec4c12c)

Once the header is soldered, we need to wire it to the ST-LINK (or clone):
![stlink-wiring-1](https://github.com/user-attachments/assets/dde1173e-9f4d-4b27-a64b-e16d1406ec65)

Use the following wiring diagram:

<img width="670" height="607" alt="wiring-diagram" src="https://github.com/user-attachments/assets/36824e27-57cc-4ce8-be83-ac72ccabf5a1" />

```
This ASCII diagram is courtesy @transmutated
+-------- J1 6-Pin Connector --------+            +----- STLink V2 20-Pin Connector ------+
|                                    |            |                                       |
| Pin 1: DIO   ----------------------|------------| Pin 7: SWDIO / TMS7                   |
| Pin 3: CLK   ----------------------|------------| Pin 9: SWCLK / TCK9                   |
| Pin 5: RST   ----------------------|------------| Pin 15: MCU RST                       |
| Pin 6: GND   ----------------------|------------| Pin 12: GND                           |
| Pin 4: GND                         |            |                                       |
| Pin 2: 3V3   ----------------------|---+--------| Pin 1: VDD                            |
|                                    |   |        |                                       |
|                                    |   |--------| Pin 19: VDD                           |
+------------------------------------+            +---------------------------------------+
```

## Flashing Katapult on the toolhead
I used the printer itself to flash the toolhead but you can use any other computer available to build and flash.

1. Plug your ST-Link into the top usb port of the printer
2. Install the ST-Link tools: `sudo apt install stlink-tools`
3. Check that your ST-Link is recognized (and recognizing the toolhead): `st-info --probe`. You should see output similar to this:

    <img width="283" height="189" alt="st-info-1" src="https://github.com/user-attachments/assets/9d3c2e20-7363-4f50-a6c0-b91edcd1e510" />

   Note that your output might not match exactly. What you are looking for is that it found an stlink programmer, and it detects a chip (above we see `chipid: 0x0423` and `descr: F4xx`).
   If you don't see these things, double check your wiring.

4. Next, we need to build katapult for the toolhead:
    ```
    cd ~/katapult
    make menuconfig
    ```
    Make sure your menuconfig matches this:

    <img width="960" height="540" alt="katapult" src="https://github.com/user-attachments/assets/b033631b-aee4-4ec2-b318-2b63b7a5c848" />

    ```
    make clean
    make -j4
    ```

5. Now we can flash katapult via the ST-Link:

    ```
    st-flash write ~/katapult/out/katapult.bin 0x8000000
    ```

    You should see a message like this:

    <img width="813" height="241" alt="st-flash" src="https://github.com/user-attachments/assets/05e07ff0-7ae0-4fd0-b19f-69a6c145c138" />

6. Unhook your ST-Link, and put the toolhead board back into the printer. Reconnect all wires and reboot the printer.

## Flashing Klipper on the toolhead
This is the same basic process that we used to flash klipper on the main MCU
1. Build klipper
    ```
    cd ~/klipper
    make menuconfig
    ```
    Your menuconfig should match this for the toolhead:

    <img width="960" height="540" alt="klipper" src="https://github.com/user-attachments/assets/969d96d4-2f6a-4121-b085-b710e4fdfe02" />

    ```
    make clean
    make -j4
    ```
2. Double click the reset button on the toolhead within 500ms to enter Katapult
3. Flash klipper:
    ```
    cd ~/katapult/scripts
    python3 flashtool.py -b 500000 -d /dev/ttyS2 -f ~/klipper/out/klipper.bin
    ```
    You should see a successful flash like this:

    <img width="745" height="323" alt="flash_success-toolhead" src="https://github.com/user-attachments/assets/e0192953-8906-4f90-82b3-e0576ff41ace" />

4. Reboot the printer and you're done - you can now start configuring!

---

# Special thanks
- @Phrac; using your [QIDI Plus 4 flashing guide](https://github.com/phrac/plus4_kalico?tab=readme-ov-file) as the initial basis for this guide.
- @53Aries; helping me troubleshoot the problems I encountered while setting up this modification to begin with.
- @transmutated; referring to your [QIDI Plus 4 flashing guide](https://github.com/cgarwood82/plus4MainlineKlipperConfig) to help guide me through my initial modification.
