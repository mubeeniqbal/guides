# Install Remix OS on VMware Workstation

## Download Remix OS 64-bit

First, go to Remix OS's website to download Remix OS 64-bit: http://www.jide.com/remixos-for-pc#downloadNow

---

**UPDATE:** Since Jide has discontinued the development of Remix OS the download links on their website don't work any longer. They take you to a 404 page instead.

You can download the latest Remix OS 64-bit from here instead: https://osdn.net/projects/remixos/downloads/66775/Remix_OS_for_PC_Android_M_64bit_B2016112101.zip/

---

## Create New Virtual Machine

Open VMware Workstation. I am using VMware Workstion 14 Pro as of this writing.

Then go to **File > New Virtual Machine...**

A new virtual machine wizard will pop up.

Set it up as follows:

1. What type of configuration do you want? **Custom (advanced)**
2. Select the latest Hardware (**Workstation 14.x** as of this writing).
3. Select **I will install the operating system later.**
4. Guest operating system: **Linux**
    - Version: **Other Linux 4.x or later kernel 64-bit**
5. Virtual machine name: **remix-os**
    - Select a location to store the VM files.
6. Processor Configuration
    - Number of processors: **1**
    - Number of core per processor: **2**
7. Memory for virtual machine: **4096 MB**
8. Network Type: **Use network address translation (NAT)**
9. SCSI Controller: **LSI Logic (Recommended)**

Create a VM with these settings:

- File name: remix-os

HARDWARE
  
  Memory: 4 GB
  Processors: 1, Cores: 2, Virtualize Intel VT-x, Virtualize IOMMU
  Hard Disk: (SCSI 0:0) 100 GB
  CD/DVD: (SATA 0:1)
  Network Adapter: NAT
  Display: Accelerate 3D graphics, Use host settings for monitor, Graphics memory: 1 GB
  
  OPTIONS
  
  General: VM name: Remix OS, OS: Other Linux 4.x or later kernel 64-bit
  Power: Report battery information to guest
  Advanced: UEFI selected

Drives: sda is 100 GB HDD

Partitions
```
/dev/sda1 [esp] (HDD) 1024M for EFI System Partition
/dev/sda2 [arch] (HDD)
```

- Start by booting Arch Linux live CD. We are going to use it to partition the drive for GPT+UEFI.

- Verify the boot mode
  # ls /sys/firmware/efi/efivars
  (If the directory does not exist, the system may be booted in BIOS or CSM mode)

- View disks and partitions
  # lsblk

- Erase the partition table
  # sgdisk --zap-all /dev/sda
  # dd if=/dev/zero of=/dev/sda bs=1M count=10000 status=progress
  (zeroes out only the first 10000 blocks)

- View disks and partitions again to ensure udev has given them the names (i.e. /dev/sdX where X is a, b, c, etc.) you'd want them to have
  # lsblk

- Check UEFI mode (if UEFI variable print you're in UEFI mode)
  # efivar -l

- Use cgdisk to create gpt partitions (UEFI and BIOS are going to have different partition layouts)
  # cgdisk /dev/sda (1024 MB fat32 "esp" partition [ef00], rest for btrfs "arch" partition [8300])

- Check if the disk partitions are set as desired.
  # lsblk

- Create filesystems
  # mkfs.fat -F32 -n esp /dev/sda1
  # mkfs.ext4 -L remix /dev/sda2

- View partitions
  # blkid

- Shutdown VM
  # poweroff

- Change ISO image file in VM CD/DVD drive to Remix OS ISO and boot VM.
- Select Resident Mode and press e to edit the boot command.
- Replace
  DATA= USB_DATA_PATITION=1
  With
  INSTALL=1 DEBUG=
- Then press F10 to boot.
- Select sda2 for installation.
- Select do not format since we have already formatted using Arch Linux live CD.
- Do you want to install EFI GRUB2? Select Yes. (You can also use Arch Linux to install the latest and greatest Grub and then write the grub.cfg file manually.)
- Do you want to format the boot partition sda1? Select No.
- Do you want to install /system directory as read/write? Select Yes.
- After this installation will start.
- When the installation ends select Reboot.
- When the VM reboots you are presented with Remix OS first time setup.
- Select language: English (United States)
- Agreement: Agree
- Wi-Fi Setup: Skip
- When presented to install any apps install no apps.
- Select “I would like to activate Google Play services.”
- After that you enter the desktop.
- The first thing you do is go to display settings and change Sleep settings to NEVER SLEEP.
- Then shutdown the machine.
- Then in VM settings change CD/DVD drive to “Use physical drive”.
- Then boot the machine again.
- Then the next time when you are at the desktop open “Play activator” and hit YES, LET’S ACTIVATE.
- Then reboot.
- Then open Play Store and sign in.
- The wait some time for the Play Store to sync. Then reboot again.
- All done.
