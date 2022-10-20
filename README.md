# bootFromUSB
Boot NVIDIA Jetson Nano Developer Kit from a mass storage USB device. This includes Jetson Nano 2GB, and may also work with the TX1. This setup is done from the Jetson Development Kit itself.

<em><b>WARNING: </b>This is a low level system change. You may have issues which are not easily solved. You should do this working on a freshly flashed micro SD card, and certainly do not attempt this with valuable data on the card itself. A serial debug console is useful if things go wrong. </em>

A new feature of JetPack 4.5 (L4T 32.5) is the ability to boot from a USB device with mass storage class and bulk only protocol. This includes devices such as a USB HDD, USB SSD and flash drives.

In order to setup a Jetson to boot from a USB device, there are several steps.


## Step 1: Setup and Boot the Jetson
You will need to do an initial setup of the Jetson with JetPack 4.5+ in order to load updated firmware into the Jetson Module QSPI-NOR flash memory. Follow the 'Getting Started' instructions on the JetPack site: https://developer.nvidia.com/embedded/jetpack

The JetPack archives are located here: https://developer.nvidia.com/embedded/jetpack-archive

During the initial setup of L4T 32.5+, the firmware for the Jetson Nano developer kits relocates the boot firmware from the micro SD card to the Jetson module integrated QSPI-NOR flash memory. This also changes the layout of the SD card. This layout is now analagous to the BIOS in a PC.

## Step 2: Prepare the USB Drive
Plug in the USB drive.

Prepare the USB drive (USB 3.0+, SSD, HDD, or flash drive) by formatting the disk as GPT with an Ext4 partition. Formatting the disk will erase any data that is on that disk. When finished, the disk should show as /dev/sda1 or similar. Note: Make sure that the partition is Ext4, as other formats will appear to copy correctly but cause issues later on. You may set the volume label during this process.

You can prepare the USB drive by using the Disks app. Formatting the disk using Format and creating a partition are two different tasks. Format the disk as GPT, then add a partition formatted as Ext4. Name the newly created partition APP so that Jetson system apps recognize it correctly.

Once the disk is ready, mount the disk. If you are using a desktop, you can do this by clicking on the USB disk icon and opening a file browser on its contents.

## **Formatting the USB Drive using Linux Command**

To simplify the steps for formatting the USB drive using command line, make sure that you only have one USB drive being connected to the Jetson Nano. Firstly, we need to search for the device path of the USB drive by entering the command in the terminal:

```
$ sudo parted -l
```

We have to obtain the optimal IO size and physical block size of the USB drive by entering the 2 commands below (you have to change the term “sda” into the device path as obtained from the previous step):

```
$ cat /sys/block/sda/queue/optimal_io_size
$ cat /sys/block/sda/queue/physical_block_size
```

After obtaining the number of sector, it’s time to format the USB drive. Make sure that there are no important files inside before proceeding to the next step!

```
$ sudo parted /dev/sda
$ mklabel gpt
```

You might be prompted to confirm the formatting, just enter “y” for confirmation. After formatting, we need to create the partition in the USB drive. Replace the  variable with the number of sector which we calculated earlier.

```
$ mkpart primary ext4 s 100%
```

Press ctrl + D to exit the parted command line. Then, we have to properly format the partition we created earlier to ext4 file system. Make sure that you change the term “sda” into the device path being obtained previously.

```
$ mkfs.ext4 /dev/sda1
$ sudo parted -l
```

Look for the device path of the USB drive, in this case it is “/dev/sda”. Search for the USB drive again. If all of the steps are taken correctly, you will see that the Partition Table is set to “gpt” and the file system for the partition is “ext4” as shown in the image above.

```
ajeetraina@ajeetraina-desktop:~$ sudo mkdir /mnt/usbboot
ajeetraina@ajeetraina-desktop:~$ sudo mount /dev/sda1 /mnt/usbboot
[sudo] password for ajeetraina:
ajeetraina@ajeetraina-desktop:~$ sudo df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p1   15G   12G  2.6G  82% /
none            947M     0  947M   0% /dev
tmpfs           986M  4.0K  986M   1% /dev/shm
tmpfs           986M   27M  960M   3% /run
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           986M     0  986M   0% /sys/fs/cgroup
tmpfs           198M  4.0K  198M   1% /run/user/121
tmpfs           198M     0  198M   0% /run/user/1000
/dev/sda1        29G   12G   16G  43% /mnt/usbboot
```

## **Cloning the repository**

```
git clone https://github.com/PHONGANSCENTER/boot_from_usb.git
```

```
ajeetraina@ajeetraina-desktop:~/boot_from_usb$ ./copyRootToUSB.sh -p /dev/sda1
Device Path: /dev/sda1
Target: /mnt/usbboot
..
Setting up rsync (3.1.2-2.1ubuntu1.2) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
Processing triggers for systemd (237-3ubuntu10.50) ...
 11,511,169,692  95%   17.93MB/s    0:10:12 (xfr#142716, to-chk=0/205797)
ajeetraina@ajeetraina-desktop:~/bootFromUSB$
```

Copy the below content and put it in extlinux.conf file as shown below:

```
ajeetraina@ajeetraina-desktop:/mnt/usbboot/boot/extlinux$ cat extlinux.conf
TIMEOUT 30
DEFAULT sdcard

MENU TITLE L4T boot options
# Sample extlinux.conf file for booting from USB
# You will need to set the root environment variable to match your system
#
# You can set the root to various levels of specificity
# If you have multiple USB storage devices, the PARTUUID approach is very useful
# PARTUUID is the PARTUUID of the USB device; most exact - recommended
# LABELNAME is a little more specific; root=LABEL=jetson_drive
# /dev/sda1 is the most general; root=/dev/sda1
# Note: The UUID options seems to have some issues on the Jetson
LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} quiet root=PARTUUID=32a76e0a-9aa7-4744-9954-dfe6f353c6a7 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0

# When testing a custom kernel, it is recommended that you create a backup of
# the original kernel and add a new entry to this file so that the device can
# fallback to the original kernel. To do this:
#
# 1, Make a backup of the original kernel
#      sudo cp /boot/Image /boot/Image.backup
#
# 2, Copy your custom kernel into /boot/Image
#
# 3, Uncomment below menu setting lines for the original kernel
#
# 4, Reboot

LABEL sdcard
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} quiet root=/dev/sda1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0
```

That’s it. Now remove the SD card and let your Jetson nano boot from USB drive

```
ajeetraina@ajeetraina-desktop:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        29G   12G   16G  43% /
none            947M     0  947M   0% /dev
tmpfs           986M   40K  986M   1% /dev/shm
tmpfs           986M   27M  960M   3% /run
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           986M     0  986M   0% /sys/fs/cgroup
tmpfs           198M  8.0K  198M   1% /run/user/1000
ajeetraina@ajeetraina-desktop:~$
```

## Step 5 Try It Out!

Remove the micro SD card, and boot the system.


<h2>Release Notes</h2>
<h3>March, 2021</h3>

* JetPack 4.5.1
* L4T 32.5.1
* Remove payload updater, the JetPack update addresses this issue
* Slightly more generous readme
* Tested on Jetson Nano


<h3>February, 2021</h3>

* JetPack 4.5
* L4T 32.5
* Initial Release
* Tested on Jetson Nano





