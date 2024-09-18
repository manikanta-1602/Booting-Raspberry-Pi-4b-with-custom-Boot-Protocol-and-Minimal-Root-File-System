# Booting-Raspberry-Pi-4b-with-custom-Boot-Protocol-and-Minimal-Root-File-System
Table of Contents
Hardware Requirements
Preparation
Create the Bootable SD Card
Create the Toolchain
Build the Bootloader
Build the Kernel
Build the Root Filesystem
Preparing the Root Partition
Boot
Hardware Requirements
A Linux Desktop machine running Ubuntu to act as a host to cross compile source code.
SD card and reader.
Preparation
Install required dependencies: 

```bash
sudo apt install bc bison flex libssl-dev make libc6-dev libncurses5-dev
```
Create the Bootable SD Card
Insert the SD card and find its device name:
```bash
lsblk
'''

In this case, the device name is sdb and it already has two partitions sdb1 and sdb2.

Deleting existing partitions, just to be sure:

$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): d
Partition number (1,2, default 2):

Partition 2 has been deleted.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): p
Disk /dev/sdb: 1.85 GiB, 1967128576 bytes, 3842048 sectors
Disk model: Storage Device
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
Add two partitions:

Create a 100MB primary partition of type W95 FAT32 (LBA)
Create another primary partition with the remaining space of type Linux
$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.34).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
p   primary (0 primary, 0 extended, 4 free)
e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-3842047, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-3842047, default 3842047): +100M

Created a new partition 1 of type 'Linux' and of size 100 MiB.
Partition #1 contains a vfat signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): n
Partition type
p   primary (1 primary, 0 extended, 3 free)
e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (2-4, default 2):
First sector (206848-3842047, default 206848):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (206848-3842047, default 3842047):

Created a new partition 2 of type 'Linux' and of size 1.8 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): t
Partition number (1,2, default 2): 1
Hex code (type L to list all codes): b

Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help): p
Disk /dev/sdb: 1.85 GiB, 1967128576 bytes, 3842048 sectors
Disk model: Storage Device
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot  Start     End Sectors  Size Id Type
/dev/sdb1         2048  206847  204800  100M  b W95 FAT32
/dev/sdb2       206848 3842047 3635200  1.8G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
Fromat the partitions:

# FAT32 for boot partition
$ sudo mkfs.vfat -F 32 -n boot /dev/sdb1

# ext4 for root partition
$ sudo mkfs.ext4 -L root /dev/sdb2
Mount the partitions:

sudo mount /dev/sdb1 /mnt/boot
sudo mount /dev/sdb2 /mnt/root
Create the Toolchain
The toolchain will consist of:

A cross compiler
Binary Utilities like the assembler and the linker
some runtime libraries
Toolchain is built using toolchain generator called crosstool-ng.

Download crosstool-ng source:

git clone https://github.com/crosstool-ng/crosstool-ng
cd crosstool-ng/
# Switching to 1.24.0 version is necessary. This is because the RPi fork of the Linux Kernel always trails behind the original Linux Kernel, but it is more stable. The kernel version must be higher than the kernel version configured for the toolchain.
git checkout crosstool-ng-1.24.0 -b 1.24.0
Build and install crosstool-ng

./bootstrap
./configure --prefix=${PWD} 
make 
make install
export PATH="${PWD}/bin:${PATH}"
Configure crosstool-ng

By default, crosstool-ng comes with a few ready-to-use-configurations. You can see the full list by typing

ct-ng list-samples
For this version of crosstool-ng, there are no configurations for Rpi4. So an existing configuration for rpi3 is used and is modified to match Rpi4.

The one from the list that is closest to the target is aarch64-rpi3-linux-gnu This command is run to get the details of this configuration:

ct-ng show-aarch64-rpi3-linux-gnu
Output:

[L...]   aarch64-rpi3-linux-gnu
Languages       : C,C++
OS              : linux-4.20.8
Binutils        : binutils-2.32
Compiler        : gcc-8.3.0
C library       : glibc-2.29
Debug tools     : gdb-8.2.1
Companion libs  : expat-2.2.6 gettext-0.19.8.1 gmp-6.1.2 isl-0.20 libiconv-1.15 mpc-1.1.0 mpfr-4.0.2 ncurses-6.1 zlib-1.2.11
Companion tools :
The OS version is to be noticed. The OS is linux-4.20.8 meaning binaries compiled by the toolchain will run on any kernel version >=4.20.8

Selecting aarch64-rpi3-linux-gnu as a base-line configuration:

ct-ng aarch64-rpi3-linux-gnu
Customization for Rpi4:

Open menuconfig:

ct-ng menuconfig
Three changes are to be made:

Paths and misc options -> Render the toolchain read-only -> false
This is for allow extending the toolchain after it is created (by default it is created as read-only)

Target Options -> Emit Assembly for CPU -> change cortex-a53 to cortex-a72

Toolchain Options -> Tuple's Vendor String -> change rpi3 to rpi4

Building the toolchain

Before building the toolchain, it is to be made sure that gcc version is 8 and g++ version is 8. Otherwise error will arise while building the toolchain in binutils step.

If the host system is Ubuntu 20.04 or lower, it is possible to downgrade the gcc and g++ version easily:

sudo apt-get install -y gcc-8 g++-8
Set gcc 8 as default:

sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-8 80
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-8 80
Verify the Installation:

gcc --version
g++ --version
If the host system is Ubuntu 22.04 (or higher):

The gcc-8 package has been discontinued in Ubuntu 22.04 and later repositories, but there is a workaround.

Workaround for Ubuntu 22.04 or higher

Install isl-0.20 and libexpat and other tools:

wget https://libisl.sourceforge.io/isl-0.20.tar.gz
mv isl-0.20.tar.gz /home/sprak/crosstool-ng/.build/tarballs/
wget https://github.com/libexpat/libexpat/releases/download/R_2_2_6/expat-2.2.6.tar.bz2
mv expat-2.2.6.tar.bz2 /home/surya/crosstool-ng/.build/tarballs/
sudo apt-get install -y python3-dev
Finally, build the configured crosstool-ng:

 ```bash
 ct-ng build
 ```
Build the Bootloader
Because bootloader is device specific, it is to be configured before building like crosstool-ng.

Here, U-boot is used for the proprietory ROM code.

Clone U-Boot repository:

git clone git://git.denx.de/u-boot.git 
cd u-boot
git checkout v2021.10 -b v2021.10
Configure U-Boot:

export PATH=${HOME}/x-tools/aarch64-rpi4-linux-gnu/bin/:$PATH
export CROSS_COMPILE=aarch64-rpi4-linux-gnu-
make rpi_4_defconfig
Build U-boot:

make
