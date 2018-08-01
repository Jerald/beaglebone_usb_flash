# beaglebone_usb_flasher
#### A full system to flash a BeagleBone over usb.
---

This system is forked from the [BBBlfs](https://github.com/ungureanuvladvictor/BBBlfs) project. Without his work this likely wouldn't be here. 
Albeit a lot of it was broken and has to be re-engineered... But still his work was the base.


## Build

You need to build the project to create a small executable used to put the BeagleBone into the correct mode to be flashed over usb.

It requires `libusb` and `automake` as dependencies. Download them on your favourite package manager. After that, run the below commands to make the executable:
```bash
./autogen.sh
./configure
make
```

## Usage

Press the S2 button on the BeagleBone and apply power to the board. That's the small button on the opposite side as the ethernet port. The board should now start into USB boot mode.

Connect your BeagleBone to the host PC, the kernel will identify the board as an RNDIS interface. Which is to say, a bare-bones ethernet interface. 

<!-- Be sure you do not have any BOOTP servers on your network. -->

Navigate to the repo's `bin/` folder and execute `flash_script.sh` **as root**. The first argument should be the image you want to flash. For now only .xz compressed images are supported.

For example: `sudo ./flash_script.sh  image.img.xz`


## How to build the binary blobs

The full system works as follow:

* The AM335x ROM when in USB boot mode exposes a RNDIS interface to the PC and waits to TFTP a binary blob that it executes. That binary blob is the SPL
* Once the SPL runs it applies the same proceure as the ROM and TFTPs U-Boot
* Again U-Boot applies the same thinking and TFTPs a FIT(flatten image tree) which includes a Kernel, Ramdisk and the DTB
* U-Boot runs this FIT and boots the Kernel
* When the kernel starts the init script exports the eMMC using the g_mass_storage kernel module as an USB stick to the Linux so it can be flashed


## Building U-Boot for USB booting
* Grab the latest U-Boot sources from [git://git.denx.de/u-boot.git](git://git.denx.de/u-boot.git)
* Checkout commit id 524123a70761110c5cf3ccc5f52f6d4da071b959
* Install your favourite cross-compiler, I am using arm-linux-gnueabihf-
* Apply this patch to U-Boot sources [https://raw.githubusercontent.com/ungureanuvladvictor/BBBlfs/master/tools/USB_FLash.patch](https://raw.githubusercontent.com/ungureanuvladvictor/BBBlfs/master/tools/USB_FLash.patch )

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- am335x_evm_usbspl_config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi-
```
Now you have u-boot.img which is the uboot binary and spl/u-boot-spl.bin which is the spl binary


## Building the Kernel
* Grab the latest from [https://github.com/beagleboard/kernel](https://github.com/beagleboard/kernel)
```bash
git checkout 3.14
./patch.sh
cp configs/beaglebone kernel/arch/arm/configs/beaglebone_defconfig
wget http://arago-project.org/git/projects/?p=am33x-cm3.git\;a=blob_plain\;f=bin/am335x-pm-firmware.bin\;hb=HEAD -O kernel/firmware/am335x-pm-firmware.bin
cd kernel
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- beaglebone_defconfig -j4
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- zImage dtbs modules -j4
```

* After compilation you have in arch/arm/boot/ the zImage


## Building the ramdisk

* Our initramfs will be built around BusyBox. First we create the basic folder structure.
```bash
mkdir initramfs
mkdir -p initramfs/{bin,sbin,etc,proc,sys}
cd initramfs
wget -O init https://raw.githubusercontent.com/ungureanuvladvictor/BBBlfs/master/tools/init
chmod +x init
```
* Now we need to cross-compile BusyBox for our ARM architecture
```bash
git clone git://git.busybox.net/busybox
cd busybox
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig
```
* Now here we need to compile busybox as a static binary: Busybox Settings --> Build Options --> Build Busybox as a static binary (no shared libs)  -  Enable this option by pressing "Y"
```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j4
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- install CONFIG_PREFIX=/path/to/initramfs
```
* Now we need to install the kernel modules in our initramfs
```bash
cd /path/to/kernel/sources
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- modules_install INSTALL_MOD_PATH=/path/to/initramfs
```


## Packing things up

* Now we need to put our initramfs in a .gz archive that the kernel knows how to process
```bash
mkdir maker
cd /path/to/initramfs
find . | cpio -H newc -o > ../initramfs.cpio
cd .. && cat initramfs.cpio | gzip > initramfs.gz
mv initramfs.gz /path/to/maker/folder/ramdisk.cpio.gz
```
* Now we need to pack all things in a FIT image. In order to do so we need some additional packages installed, namely the mkimage and dtc compiler.
```bash
sudo apt-get update
sudo apt-get install u-boot-tools device-tree-compiler
cd /path/to/maker/folder
wget -O maker.its https://raw.githubusercontent.com/ungureanuvladvictor/BBBlfs/master/tools/maker.its
cp /path/to/kernel/arch/arm/boot/zImage .
cp /path/to/kernel/arch/arm/boot/dts/am335x-boneblack.dtb .
mkimage -f maker.its FIT
```
* At this point we have all things put into place. You need to copy the binary blobs in the bin/ folder and run ```flash_script.sh```
