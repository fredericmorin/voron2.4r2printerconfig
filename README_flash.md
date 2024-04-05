
Printer hardware:
- Voron V2.4r2
- BTT octopus v1.1
- BTT EBB SB2209 v1.0



# Firmware strategy

Use Octopus as can bridge to can network

Firmware:
- Katapult serial on octopus
- Katapult can on ebb
- Klipper usb to can on octopus
- Klipper can on ebb


# Install
```sh
git clone https://github.com/Klipper3d/klipper klipper-ebb
ln -s ~/printer_data/config/config/klipper_config      ~/klipper/.config
ln -s ~/printer_data/config/config/klipper-ebb_config  ~/klipper-ebb/.config
```



# First Flash

## Katapult on Octopus v1.1

```sh
sudo systemctl stop klipper

git clone https://github.com/Arksine/katapult ~/katapult
cd ~/katapult
make menuconfig
#               Katapult Configuration v0.0.1-62-g6a7ca81
#      Micro-controller Architecture (STMicroelectronics STM32)  --->
#      Processor model (STM32F446)  --->
#      Build Katapult deployment application (Do not build)  --->
#      Clock Reference (12 MHz crystal)  --->
#      Communication interface (USB (on PA11/PA12))  --->
#      Application start offset (32KiB offset)  --->
#      USB ids  --->
#  ()  GPIO pins to set on bootloader entry
#  [*] Support bootloader entry on rapid double click of reset button
#  [ ] Enable bootloader entry on button (or gpio) state
#  [ ] Enable Status LED
make -j4
```

Flash to octopus. You have to put jumper on boot0 and press the reset button that is right next to the USB-C connector.

Once you get DFU to work:
```sh
sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin  --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
```

## Klipper on Octopus v1.1

```sh
sudo systemctl stop klipper

# skip git clone on mainsailOS
cd ~/klipper
~/klippy-env/bin/python scripts/make_version.py `hostname`
# v0.11.0-143-gc54d83c9-mainsailos
make menuconfig
#                 Klipper Firmware Configuration
#  [*] Enable extra low-level configuration options
#      Micro-controller Architecture (STMicroelectronics STM32)  --->
#      Processor model (STM32F446)  --->
#      Bootloader offset (32KiB bootloader)  --->
#      Clock Reference (12 MHz crystal)  --->
#      Communication interface (USB to CAN bus bridge (USB on PA11/PA12))  --->
#      CAN bus interface (CAN bus (on PD0/PD1))  --->
#      USB ids  --->
#  (500000) CAN bus speed
#  ()  GPIO pins to set at micro-controller startup
make -j4
python3 ~/katapult/scripts/flash_can.py -d /dev/serial/by-id/usb-katapult_stm32f446xx* -f ~/klipper/out/klipper.bin
sudo apt install ifupdown net-tools
sudo mkdir -p /etc/network/interfaces.d/
sudo tee /etc/network/interfaces.d/can0 <<EOF
allow-hotplug can0
iface can0 can static
    bitrate 500000
    up ifconfig \$IFACE txqueuelen 1024
    pre-up ip link set can0 type can bitrate 500000
    pre-up ip link set can0 txqueuelen 1024
EOF
sudo ifup can0
python3 ~/katapult/scripts/flash_can.py -i can0 -q
#  Resetting all bootloader node IDs...
#  Checking for katapult nodes...
#  Detected UUID: xxxxxxxxxxxx, Application: Klipper
#  Query Complete
```

## Katapult on EBB SB2209

```sh
sudo systemctl stop klipper

git clone https://github.com/Arksine/katapult ~/katapult-ebb
cd ~/katapult-ebb
make menuconfig
#               Katapult Configuration v0.0.1-62-g6a7ca81
#      Micro-controller Architecture (STMicroelectronics STM32)  --->
#      Processor model (STM32G0B1)  --->
#      Build Katapult deployment application (Do not build)  --->
#      Clock Reference (8 MHz crystal)  --->
#      Communication interface (CAN bus (on PB0/PB1))  --->
#      Application start offset (8KiB offset)  --->
#  (500000) CAN bus speed
#  ()  GPIO pins to set on bootloader entry
#  [*] Support bootloader entry on rapid double click of reset button
#  [ ] Enable bootloader entry on button (or gpio) state
#  [*] Enable Status LED
#  (PA13)  Status LED GPIO Pin
make -j4
```

Hook up EB2209 to your printer:
- 24v power (WITH A FUSE PLEASE)
- Octopus can connector
- Close 120R jumper on EB2209 board
- Power up
- Connect EBB usb-C to printer RPi
- Hold both reset and boot0 button, release reset button, wait 1 second, release boot0
- See DFU device show up in `lsusb`

```sh
sudo dfu-util -a 0 -D ~/katapult-ebb/out/katapult.bin  --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11

python3 ~/katapult-ebb/scripts/flash_can.py -i can0 -q
#  Resetting all bootloader node IDs...
#  Checking for katapult nodes...
#  Detected UUID: yyyyyyyyyyyy, Application: Katapult
#  Detected UUID: xxxxxxxxxxxx, Application: Klipper
#  Query Complete
```


## Klipper on EBB SB2209

```sh
sudo systemctl stop klipper

git clone https://github.com/Klipper3d/klipper.git ~/klipper-ebb
cd ~/klipper-ebb
~/klippy-env/bin/python scripts/make_version.py `hostname`
# v0.11.0-146-ge2d7c598-mainsailos
make menuconfig
#                 Klipper Firmware Configuration
#  [*] Enable extra low-level configuration options
#      Micro-controller Architecture (STMicroelectronics STM32)  --->
#      Processor model (STM32G0B1)  --->
#      Bootloader offset (8KiB bootloader)  --->
#      Clock Reference (8 MHz crystal)  --->
#      Communication interface (CAN bus (on PB0/PB1))  --->
#  (500000) CAN bus speed
#  ()  GPIO pins to set at micro-controller startup
make -j4
python3 ~/katapult-ebb/scripts/flash_can.py -i can0 -u yyyyyyyyyyyy -f ~/klipper-ebb/out/klipper.bin
python3 ~/katapult-ebb/scripts/flash_can.py -i can0 -q
#  Resetting all bootloader node IDs...
#  Checking for katapult nodes...
#  Detected UUID: yyyyyyyyyyyy, Application: Klipper
#  Detected UUID: xxxxxxxxxxxx, Application: Klipper
#  Query Complete
```

## References and Userful Links

- https://github.com/akhamar/voron_canbus_octopus_sb2040#useful-tricks-to-be-able-to-update-an-octopus-11-in-usb-to-can-bridge
- https://www.teamfdm.com/forums/topic/851-install-canboot-on-sb2040/#comment-5785
- https://gist.github.com/jfryman/0c3827079e23d7bc55f9677d2c6b8bec
- https://github.com/Arksine/katapult?tab=readme-ov-file#katapult-deployer


# Update Firmware

## Klipper on EBB SB2209

```sh
cd ~/klipper-ebb/
git pull
make olddefconfig
make -j4

sudo systemctl stop klipper
~/klippy-env/bin/python3 ~/katapult-ebb/scripts/flash_can.py -i can0 -u c5f03e47651b -r
~/klippy-env/bin/python3 ~/katapult-ebb/scripts/flash_can.py -i can0 -u c5f03e47651b -f ~/klipper-ebb/out/klipper.bin
sudo systemctl start klipper

~/klippy-env/bin/python3 ~/katapult-ebb/scripts/flash_can.py -i can0 -q
```

## Klipper on Octopus v1.1

```sh
cd ~/klipper/
git pull
make olddefconfig
make -j4

sudo systemctl stop klipper
~/klippy-env/bin/python3 ~/katapult/scripts/flash_can.py -i can0 -u 3c57b3d9e9aa -r
~/klippy-env/bin/python3 ~/katapult/scripts/flash_can.py -d /dev/serial/by-id/usb-*stm32f446xx* -f ~/klipper/out/klipper.bin
sudo systemctl start klipper

~/klippy-env/bin/python3 ~/katapult/scripts/flash_can.py -i can0 -q
```


