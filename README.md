# Pi-Floppy

## Preface

This tutorial is a minor adaptation of the "Make a Pi Zero W Smart USB flash drive" article on the MagPi magazine. \
Original article here: https://magpi.raspberrypi.com/articles/pi-zero-w-smart-usb-flash-drive

Note that the original MagPi tutorial has a typo in Step 14: \
`ExecStart=/usr/local/share/usb_share.py` should be `ExecStart=/usr/local/share/usbshare.py`.

Also, for the USB device to show up properly on windows and be writable (if that is desired), \
you need to do `sudo modprobe g_mass_storage file=/piusb.bin stall=0 ro=0 removable=1` in Step 10.

Otherwise, the MagPi tutorial still works as of 2022-02-07 on a Pi Zero 2 W.

## Intro

This tutorial helps you make a Raspberry Pi (tested on a Pi Zero 2 W) into a remotely accessible floppy USB emulatior drive, for use with floppy disk to USB emulators (commonly sold on eBay). These devices can be used in any old machine that has a 3.5" floppy drive reader, I tested this on an ABB S4C+ controller.

You will need a Pi and a PC with Windows.

## Step 1: Set Up Your Pi

I prefer to use the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to set up Raspbian. I use the extra settings in the Raspberry Pi Imager to preload my Wi-Fi, account, locale and SSH settings onto the Pi Zero 2 W.

This means I can directly SSH into my Pi after flashing the OS onto the SD card.

I turn off the desktop environment with \
`sudo raspi-config`

## Step 2: USB Driver

`sudo nano /boot/config.txt` \
Then append to a new line at the end of the file: \
`dtoverlay=dwc2` \
Save and exit (crtl+x, y) \
`sudo nano /etc/modules` \
Again, append to a new line at the end of the file: \
`dwc2` 

## Step 3: Container File

For our floppy, we need at least 1.44MB * 100 = 144MB of space. The next line makes a 256MiB file: \
`sudo dd bs=1M if=/dev/zero of=/piflop.bin count=256` \
Format with FAT32: \
`sudo mkdosfs /piflop.bin -F 32 -I` 

## Step 4: Set up USB Floppy Device

Now plug your Pi Zero 2 W into a Windows PC. Use the micro USB port in the middle, not the PWR IN port.

Enable mass storage mode: \
`sudo modprobe g_mass_storage file=/piflop.bin stall=0 ro=0 removable=1` \
Your drive should show up in Windows. It might ask you to format it, if so then format with FAT32, quick format. \
Now format the drive with the [USB_Floppy_Manager_v1.40i](https://drive.google.com/file/d/1C-k-ev3rfInV3rtoDJusstr8cn3QsRBf/view) tool. The link is to a Google Drive share that works as of 2022-02-07. \
Now we need to enable mass storage mode on boot: \
`sudo nano /etc/rc.local` \
Append to a new line **right above** `exit 0`: \
`sudo modprobe g_mass_storage file=/piflop.bin stall=0 ro=0 removable=1` 

## Step 5: Mount Container File

Make the directory to mount to: \
`sudo mkdir /mnt/floppy` \
And add it to fstab: \
`sudo nano /etc/fstab` \
Append to a new line at the end of the file: \
`/piflop.bin	/mnt/floppy	auto	users,umask=000 0 2` \
Mount: \
`sudo mount -a`

## Step 6: Automate Device Reconnect

The USB device needs to be unmounted and remounted on the Windows PC (or floppy to USB emulator device) every time a file is changed for it to show up on the PC/emulator. We'll use (and slightly modify) the great script from the MagPi tutorial. \
`sudo pip3 install watchdog` \
`cd /usr/local/share` \
`sudo wget http://rpf.io/usbzw -O usbshare.py` \
`sudo chmod +x usbshare.py` \
Now edit the script: \
`sudo nano usbshare.py` \
On the line that starts with `CMD_MOUNT` (line 7), change to this: \
`CMD_MOUNT = "modprobe g_mass_storage file=/piflop.bin stall=0 ro=0 removable=1"` \
On the line that starts with `WATCH_PATH` (line 11), change to this: \
`WATCH_PATH = "/mnt/floppy"` \
On the line that starts with `ACT_TIME_OUT` (line 13), change to this: (this decreases the idle wait time before the code unmounts and remounts the drive from 30 seconds to 2 seconds, you can experiment with different values) \
`ACT_TIME_OUT = 2` \
Now save the file.

Next, let's create the systemd service unit file:\
`cd /etc/systemd/system`\
`sudo nano usbshare.service`\
Paste this into the file:
```
[Unit]
Description=USB Share Watchdog

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/share/usbshare.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Start the service: \
`sudo systemctl daemon-reload`\
`sudo systemctl enable usbshare.service`\
`sudo systemctl start usbshare.service`

You can check the status of the service: \
`sudo systemctl status usbshare.service`\
It should be green.

## Step 7: Set up Your File Sharing of Choice

The MagPi tutorial referenced in the beginning has a step on setting up Samba (Step 11), or you can also use NFS (nfs-kernel-server). Many good tutorials out there!
