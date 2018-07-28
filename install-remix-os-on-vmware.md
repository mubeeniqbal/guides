# Install Remix OS on VMware Workstation (UEFI/GPT)

## What You Need

1. VMware Workstation (installed on your machine)
2. Arch Linux installer (to partition the virtual machine disk)
3. Remix OS 64-bit installer

## Download Arch Linux Installer

Visit the Arch Linux website to download the latest Arch Linux installer: https://www.archlinux.org/download/

## Download Remix OS 64-bit Installer

Visit the Remix OS website to download the latest Remix OS 64-bit installer: http://www.jide.com/remixos-for-pc#downloadNow

---

**UPDATE:** Since Jide has discontinued the development of Remix OS the download links on their website don't work any longer. They take you to a 404 page instead.

You can download the latest Remix OS 64-bit installer from here instead: https://osdn.net/projects/remixos/downloads/66775/Remix_OS_for_PC_Android_M_64bit_B2016112101.zip/

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
    - Number of cores per processor: **2**
7. Memory for virtual machine: **4096 MB**
8. Network Type: **Use bridged networking**
9. SCSI Controller: **LSI Logic (Recommended)**
10. Virtual disk type: **SCSI (Recommended)**
11. Select a Disk: **Create a new virtual disk**
12. Specify Disk Capacity
    - Maximum disk size (GB): **100**
    - Select **Store virtual disk as a single file**
13. Disk file: **remix-os.vmdk**
14. Click **Finish**

After that you will see the Remix OS virtual machine tab created in VMware Workstation but we are not done yet configuring the VM. On that tab click on **Edit virtual machine settings** and then configure the VM as follows.

**Hardware** tab:

1. Processors (based on your hardware support)
    - Check **Virtualize Intel VT-x/EPT or AMD-V-RVI**
    - Check **Virtualize IOMMU (IO Memory management unit)**
2. CD/DVD
    - Click on **Advanced...**
    - Select **SATA** option and from the drop down select **SATA 0:0**.
3. Network Adapter
    - Check **Replicate physical network connection state** under **Bridged** option.
4. Display
    - Check **Accelerate 3D graphics**.
    - From **Graphics memory** dropdown select **1 GB**.

**Options** tab:

1. General
    - Virtual machine name: **Remix OS**
2. Power
    - Check **Report battery information to guest**
3. Advanced
    - Firmware type: **UEFI**

After that click **OK** to apply your changes.

## Partition Virtual Machine Disk

Before installing Remix OS we first have to partition the virtual machine disk using GPT patitioning scheme for UEFI. In order to do that we are going to boot the VM with Arch Linux installer.

1. Click on **Edit virtual machine settings**.
2. Click on **CD/DVD**.
3. Select **Use ISO image file** option.
4. Browse the **Arch Linux installer ISO** file.
5. Click **OK**.

Then click on **Power on this virtual machine** to boot it.

Select **Arch Linux archiso x86_64 UEFI CD** from the boot menu.

Arch Linux installer will boot up and you will be greeted by a simple command line interface:

```
root@archiso ` #
```

We are going to create two GPT partitions:

1. **EFI system partition (ESP)**
    - This partition is an OS independent partition that acts as the storage place for the EFI bootloaders, applications and drivers to be launched by the UEFI firmware. It is mandatory for UEFI boot.
2. **Linux filesystem partition**.
    - Remix OS will be installed in this partition.

---

**NOTE**

- Linux sees disks as sd<em>**x**</em> where _**x**_ is **a**, **b**, **c**, ... for disks **1**, **2**, **3**, ... respectively.
- Linux sees partitions in each disk as sd<em>x**Y**</em> where _**Y**_ is **1**, **2**, **3**, ... for partitions **1**, **2**, **3**, ... respectively.

---

We will partition the disk as follows.

Disk | Size
-----|-------
sda  | 100 GB

Partition | Size                    | Partition Type ID | Label | Format
----------|-------------------------|-------------------|-------|-------
sda1      | 1024 MB                 | ef00              | esp   | FAT32
sda2      | Remainder of the device | 8300              | remix | ext4

Type the commands listed below to partition the disk.

**Verify the boot mode**

If UEFI mode is enabled on a UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:

```
# ls /sys/firmware/efi/efivars
```

If the directory does not exist, the system may be booted in BIOS or CSM mode.

**View disks and partitions**

```
# lsblk
```

**Erase the partition table**

```
# sgdisk --zap-all /dev/sda
# dd if=/dev/zero of=/dev/sda bs=1M count=10000 status=progress
```

This erases the partition table and zeroes out the first 10000 blocks.

- View disks and partitions again to ensure udev has given them the names (i.e. /dev/sdX where X is a, b, c, etc.) you'd want them to have
  # lsblk

- Check UEFI mode (if UEFI variable print you're in UEFI mode)
  # efivar -l

- Use cgdisk to create gpt partitions (UEFI and BIOS are going to have different partition layouts)
  # cgdisk /dev/sda (1024 MB fat32 "esp" partition [ef00], rest for ext4 "remix" partition [8300])

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
