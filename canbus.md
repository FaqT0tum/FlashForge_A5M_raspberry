### Now we will proceed to the configuration and installation of the Canbus U2C and EBB36 boards
The guide is based on Erik Zimmerman's, not because it is wrong, but to optimize it based on my experience and to minimize the problems I encountered.
The original guide can be found [here](https://github.com/EricZimmerman/VoronTools/blob/main/EBB_CAN.md) 

The components I used in this guide are:

1. Raspbery Pi 5
2. BTT U2C Version (2.1)
3. BTT EBB36 Version (1.2)
4. Some wire in the 22awg range
5. USB cables (I recommend that you make sure that the Type-C cable is not only power but also DATA)
6. A little patience :)

Before proceeding with the configuration of the various components, 
I recommend doing everything outside of the printer to make the various steps that will come easier for you.

# Raspberry setup

1. Use mkdir to create a new folder with the command `sudo mkdir /etc/network/interfaces.d`
2. Use nano to create a new file with the command `sudo nano /etc/network/interfaces.d/can0`
3. Add the following code
    ```bash
    allow-hotplug can0
    iface can0 can static
     bitrate 1000000
     up ifconfig $IFACE txqueuelen 1024
     pre-up ip link set can0 type can bitrate 1000000
     pre-up ip link set can0 txqueuelen 1024
     ```

4. Use nano to create a new file with the command `sudo nano /etc/systemd/network/10-can.link`
5. Add the following code then save and exit nano
    ```bash
    [Match]
    Type=can
    
    [Link]
    TransmitQueueLength=1024
    ```

6. Use nano to create a new file with the command `sudo nano /etc/systemd/network/25-can.network`
7. Add the following code then save and exit nano
    ```bash
    [Match]
    Name=can*
    
    [CAN]
    BitRate=1M
    ```

8. Now you need to add a line to bottom in crontab for enable canbus at startup, with the command `sudo crontab -e`
9. Add the following code then save and exit nano. If asked for the editor type, select (1 - nano)
    ```bash

    @reboot sudo ip link set up can0 type can bitrate 1000000 &

    ```
10. Reboot the pi with `sudo reboot`

# U2C Flashing

Now to make sure that U2C has the right firmware we need to flash it with the latest version available [here](https://github.com/Arksine/CanBoot/issues/44#issuecomment-1381555466)

1. The steps to flash this firmware are (taken from above link):
    - Download the STM32 CubePROG [here](https://www.st.com/en/development-tools/stm32cubeprog.html) 
    - Unzip the firmware file, previously downloaded
    - Press and hold the boot button on the u2c and plug the usb cable on PC, then release the boot button.
    - Launch STM32 CubePROG
    - Press the blue button that says (UART) and change it to (USB), and connect
    - Press on the green download symbol on the left
    - Browse and select a G0B1_U2C_V2.bin
    - Press (Start Programming)
    - unplug and reconnect the U2C to the Raspberry
    - Success
3. ADD THE JUMPER for the 120 ohm resistor to the U2C and set the U2C aside
    
    ![image](img/ebb/U2C120.png)

# EBB36 Flashing

### Building Katapult (formerly known as CanBoot)

1. Clone the Katapult repository to your pi
    ```bash
    cd ~
    git clone https://github.com/Arksine/katapult
    ```

2. Run the following commands to bring up the menu to configure the firmware
    ```bash
    cd katapult
    make menuconfig
    ```
    
3. Starting from the top, make your firmware selections look exactly like the image below
    ![image](img/ebb/CanBootConfig.png)

    NOTE: For Status LED GPIO Pin, be sure to enter **PA13**
    
4. Exit using `ESC` or `Q`, then confirm with yes (`Y`)
5. Build the firmware using the following commands:
    ```
    make clean
    make
    ```
    ![image](img/ebb/CanBootFirmware.png)

### Flashing Katapult

1. Add a jumper as shown in the image below so the board can be powered via a USB connection
    ![image](img/ebb/EBBButtons.png)

2. Connect your device to your Pi via USB
3. Press and hold the `RESET` and `BOOT` buttons down (button locations shown in step 1)
    1. Release `RESET` button
    2. Release `BOOT` button
4. The device should now be in DFU mode. Verify this via the `lsusb` command, which should look something like this:
    ```
    Bus 001 Device 005: ID 0483:df11 STMicroelectronics STM Device in DFU Mode
    ```
5. Record the device ID (the part after `ID` above, but yours may be different)
6. Run the following command to erase and flash the EBB with Katapult (again, VERIFY your device ID. The ID is at the end of the command below):
    ```
    sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
    ```
7. The board will now be flashed. It will look similar to what is shown below. NOTE: If you see any mention of an error after the `File downloaded successfully` message, it can be ignored.

    ![image](img/ebb/CanFlashOk.png)
    
8. Unplug the board from USB and remove the USB jumper you installed for step 1
9. ADD THE JUMPER for the 120 ohm resistor to the EBB

    ![image](img/ebb/EBB120.png)

# Verifying you see Katapult and flashing Klipper to the EBB

At this point, after checking twice that everything is connected correctly, power up the printer and SSH into the Pi.

1. Change directories into the klipper directory via `cd ~/klipper`
2. type `make menuconfig` and adjust things so it looks like the screenshot below

    ![image](img/ebb/EBBKlipper.png)

3. Exit, saving the changes, then build klipper by typing `make`
4. Find your UUID by running this command:
    ```bash
    sudo service klipper stop
    python3 ~/katapult/scripts/flashtool.py -i can0 -q
    ```
5. It should return something like:

    ```
    "Detected UUID: XXXXXXXXXX, Application: Katapult"
    ```

6. Record the UUID. This is YOUR uuid and will be used for the next step and in your cfg files.
7. Now its time to flash klipper via CAN! Run the following command, substituting your uuid:

   ```bash
   python3 ~/katapult/scripts/flashtool.py -i can0 -u b6d9de35f24f -f ~/klipper/out/klipper.bin
   ```
8. The EBB will be flashed and you should see a message about success, etc. Request for uuids again via:

   ```bash
    python3 ~/katapult/scripts/flashtool.py -i can0 -q
   ```

   but now, notice how it shows **Klipper** for the application! Yay!

9. Restart klipper via

   ```bash
   sudo service klipper start
   ```
