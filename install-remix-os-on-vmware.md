# Install Remix OS on VMware Workstation (UEFI/GPT)

## The Goal

1. Partition and format disk using Arch Linux.
2. Install Remix OS.
3. Install GRUB bootloader using Arch Linux.

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
    - Select **SATA** option and from the drop down select **SATA 0:1**.
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

_**Device** filenames_ under Linux typically take the form `/dev/sdx`, where `x` is a lowercase letter.

- `/dev/sda` is the first disk, `/dev/sdb` is the second disk, and so on.

_Device **partition** filenames_ take the form `/dev/sdxY`, where `Y` is a number.

- `/dev/sda1` is the first partition on device `/dev/sda`, `/dev/sdb2` is the second partition, and so on.

---

We will partition the disk as follows.

Disk | Size
-----|-------
sda  | 100 GB

Partition | Size                    | Partition Type ID       | Label | Format
----------|-------------------------|-------------------------|-------|-------
sda1      | 1024 MB                 | ef00 _EFI System_       | esp   | FAT32
sda2      | Remainder of the device | 8300 _Linux filesystem_ | remix | ext4

### Device Partitioning Commands

Type the commands listed below to partition the disk.

**Verify the boot mode**

If UEFI mode is enabled on a UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:

```
# ls /sys/firmware/efi/efivars
```

If the directory does not exist, the system may be booted in BIOS or CSM mode.

You can also check UEFI boot mode via the following command:

```
# efivar -l
```

If UEFI variables print you're in UEFI mode.

**View disks and partitions**

```
# lsblk
```

**Erase the partition table**

```
# sgdisk --zap-all /dev/sda
# dd if=/dev/zero of=/dev/sda bs=1M count=10000 status=progress && sync
```

This erases the GPT and MBR partition tables and zeroes out the first 10000 blocks of the disk.

**Use cgdisk to create GPT partitions**

```
# cgdisk /dev/sda
```

When cgdisk starts it will give you a warning message about non-GPT or damaged disk detected. This is because our disk has nothing on it and is in a completely wiped out state. Press enter to continue.

1. Select `[ New ]` (to create a new partition) using the left and right arrow keys and then press enter.
2. It will ask for the first sector. This is for partition alignment purposes which is important for disk performance. The default is already set to `2048`. Just press enter to accept the default.
    - We want to leave (the recommended) `1 MiB` at the start of the disk for proper partition alignment. Each sector on the disk is typically `512 bytes` in size (although there is a lot more detail to it including emulated and pysical sector sizes). `1 MiB = 1024 * 1024 = 1048576 bytes => 1048576 / 512 = 2048 sectors`. That is why the default of `2048 sectors (1 MiB)` is what we want.
    - Read more on partition alignment here: https://www.thomas-krenn.com/en/wiki/Partition_Alignment
3. Enter size for first partition: `1024M`
    - This is going to be the EFI system partition (ESP) which is required for GPT partitioning scheme on UEFI systems.
4. Enter hexcode for EFI system partition: `ef00`
5. Enter partition name: `esp`
6. Use up and down arrow keys to select the rest of the free space. Don't select the free space of `1007 KiB`.That unused space is at the start of the disk and is a result of the default alignment value of 2048 sectors.
    - An interesting thing to note here though is that we left a space of `1 MiB` at the start of the disk earlier and now all we see is `1007 KiB` free at the start of the disk. Where did `17 KiB` go? This is because in a GPT partitioning scheme LBA 0 - 33 (34 sectors in simple words) are occupied by the GUID Partition Table (GPT). `34 * 512 = 17408 bytes => 17408 / 1024 = 17 KiB => 1 MiB = 1024 KiB => 1024 - 17 = 1007 KiB`.
7. Select `[ New ]` again (to create another partition) using the left and right arrow keys and then press enter.
8. It will ask for the first sector for this new partition. Simply press enter to accept the default.
9. Next, it will ask to enter size for the partition. Since we want the second partition to occupy the remainder of the device just press enter to accept the default size (which will already be set to the remainder of the device).
    - We will install Remix OS on this partition.
10. Enter hexcode for Linux filesystem partition: `8300`
11. Enter partition name: `remix`
12. Select `[ Verify ]` (to verify the integrity of the disk's data structures) using the left and right arrow keys and then press enter.
13. If everything went well the system will report you with no problems found. Press enter to continue.
14. Select `[ Write ]` (to write the partition table to disk) using the left and right arrow keys and then press enter.
    - So far everything that we did happened in-memory and not on the actual disk. This is to prevent from destroying data on disk in case of a mistake happening. The write operation writes our in-memory operations on the disk itself.
15. Type in `yes` to confirm the write operation.
16. We are done partitioning the disk at this point. Select `[ Quit ]` (to exit cgdisk) using the left and right arrow keys and then press enter.

**Check if the partitions are created as desired**

```
# lsblk
```

### Partition Formatting Commands

**Format the partitions**

Once the partitions have been created, each must be formatted with an appropriate file system.

```
# mkfs.fat -F32 -n esp /dev/sda1
# mkfs.ext4 -L remix /dev/sda2
```

**Check if the partitions are formatted as desired**

```
# lsblk -f
```

**Shutdown virtual machine**

```
# poweroff
```

## Install Remix OS

Now that we have properly partitioned and formatted the disk we can install Remix OS onto it. In order to do that we are going to boot the VM with Remix OS installer.

1. Click on **Edit virtual machine settings**.
2. Click on **CD/DVD**.
3. Select **Use ISO image file** option.
4. Browse the **Remix OS installer ISO** file.
5. Click **OK**.

Then click on **Power on this virtual machine** to boot it.

1. Select **"Resident mode - All your data and apps are saved"** from the boot menu and press the `E` key on your keyboard to edit the boot command.
    - The installer does not start using the default boot command so we have to edit it to start the installation process.
2. Replace text after `quiet` i.e. `DATA= USB_DATA_PATITION=1` with `INSTALL=1 DEBUG=`.
3. Press `Ctrl-X` or `F10` to boot using the editted boot command.

Remix OS installer will boot up and you will be greeted by a menu to choose partition to install Remix OS.

1. Select `sda2` using the up and down arrow keys and press enter.
    - `sda2` is the partition we formatted earlier using ext4 filesystem for Remix OS installation.
2. Select `OK` using the left and right arrow keys and press enter.
3. Select `Do not format` (since we have already formatted using Arch Linux) and press enter.
4. Do you want to install EFI GRUB2? Select `Skip`.
    - We are going to install the latest and greatest GRUB later using Arch Linux.
5. Do you want to install /system directory as read/write? Select `Yes`.

After this the installation will begin. When the system prompts you of successful installation select `Reboot`.

## Install GRUB Bootloader

As soon as the VM reboots keep pressing `F2` for a few seconds (until you see the VMware logo) to enter the **Boot Manager**. You won't be able to boot into the newly installed Remix OS yet because we don't have a bootloader installed to boot into Remix OS.

Once the system displays the boot manager select `Shut down the system` and press enter.

In order to install GRUB we are going to boot from the Arch Linux installer ISO once again.

1. Click on **Edit virtual machine settings**.
2. Click on **CD/DVD**.
3. Select **Use ISO image file** option.
4. Browse the **Arch Linux installer ISO** file.
5. Click **OK**.

Then click on **Power on this virtual machine** to boot it.

Select **Arch Linux archiso x86_64 UEFI CD** from the boot menu.

Arch Linux installer will boot up. Type in the commands below to install GRUB.

**List block devices to check partition names**

```
# lsblk -f
```

**Mount the Remix OS partition**

```
# mount /dev/sda2 /mnt
```

**List files in the Remix OS partition**

```
# ls -la /mnt
```

You will notice that there is no `/boot` directory in the Remix OS partition. In Linux (Remix OS is Android which is based on the Linux kernel), and other Unix-like operating systems, the `/boot` directory contains files used for booting the operating system as well as the bootloader configuration file and bootloader stages.

**Mount EFI system partition (ESP)**

The ESP is going to be mounted at `/boot/efi`. We are going to create the needed directories and mount ESP.

```
# mkdir -vp /mnt/boot/efi
# mount /dev/sda1 /mnt/boot/efi
```

If you list files under ESP you will notice that ESP is indeed empty.

```
# ls -la /mnt/boot/efi
```

**Install GRUB**

Now that we have mounted the partitions at the correct mount points it's time to install GRUB.

```
# grub-install -v --target=x86_64-efi --efi-directory=/mnt/boot/efi --bootloader-id=arch-grub --boot-directory=/mnt/boot
```

If you list files under `/boot` and ESP now you will notice that grub files have been installed there.

```
# ls -la /mnt/boot
# ls -la /mnt/boot/efi
```

**Configure `grub.cfg` to boot Remix OS**

We are not done yet. We need to create a `grub.cfg` file and add entries to it so that we can boot Remix OS from GRUB.

```
# nano /mnt/boot/grub/grub.cfg
```

Add the following content to `grub.cfg`:

```shell
### Begin /etc/grub.d/00_header ###

insmod part_gpt
insmod part_msdos

if [ -s $prefix/grubenv ]; then
    load_env
fi

if [ "${next_entry}" ]; then
    set default="${next_entry}"
    set next_entry=
    save_env next_entry
    set boot_once=true
else
    set default="0"
fi

if [ x"${feature_menuentry_id}" = xy ]; then
    menuentry_id_option="--id"
else
    menuentry_id_option=""
fi

export menuentry_id_option

if [ "${prev_saved_entry}" ]; then
    set saved_entry="${prev_saved_entry}"
    save_env saved_entry
    set prev_saved_entry=
    save_env prev_saved_entry
    set boot_once=true
fi

function savedefault {
    if [ -z "${boot_once}" ]; then
        saved_entry="${chosen}"
        save_env saved_entry
    fi
}

function load_video {
    if [ x$feature_all_video_module = xy ]; then
        insmod all_video
    else
        insmod efi_gop
        insmod efi_uga
        insmod ieee1275_fb
        insmod vbe
        insmod vga
        insmod video_bochs
        insmod video_cirrus
    fi
}

if [ x$feature_default_font_path = xy ]; then
    font=unicode
else
    insmod part_gpt
    insmod ext2
    set root='hd0,gpt2'
    
    if [ x$feature_platform_search_hint = xy ]; then
        search --no-floppy --fs-uuid --set=root --hint-ieee1275='ieee1275//sas/disk@0,gpt2' --hint-baremetal=ahci0,gpt2 e0f8cb1e-2089-4475-813b-5d7008fad287
    else
        search --no-floppy --fs-uuid --set=root e0f8cb1e-2089-4475-813b-5d7008fad287
    fi
    
    font="/usr/share/grub/unicode.pf2"
fi

if loadfont $font; then
    set gfxmode=auto
    load_video
    insmod gfxterm
    set locale_dir=$prefix/locale
    set lang=en_US
    insmod gettext
fi

terminal_input console
terminal_output gfxterm

if [ x$feature_timeout_style = xy ]; then
    set timeout_style=menu
    set timeout=5

# Fallback normal timeout code in case timeout_style feature is unavailable
else
    set timeout=5
fi

### End /etc/grub.d/00_header ###

### Begin /etc/grub.d/40_custom ###

menuentry "Remix OS 2016-11-21" {
    load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_gpt
    insmod ext2
    
    search --set=root --file /RemixOS/kernel
    
    echo 'Loading Linux linux ...'
    linux /RemixOS/kernel quiet root=/dev/ram0 SERIAL=random logo.showlogo=1 androidboot.selinux=permissive
    echo 'Loading initial ramdisk ...'
    initrd /RemixOS/initrd.img
}

menuentry "Remix OS 2016-11-21 (DEBUG mode)" {
    load_video
    set gfxpayload=keep
    insmod gzio
    insmod part_gpt
    insmod ext2
    
    search --set=root --file /RemixOS/kernel
    
    echo 'Loading Linux linux ...'
    linux /RemixOS/kernel root=/dev/ram0 SERIAL=random logo.showlogo=1 androidboot.selinux=permissive DEBUG=2
    echo 'Loading initial ramdisk ...'
    initrd /RemixOS/initrd.img
}

menuentry "Firmware setup" {
    fwsetup
}

menuentry "System restart" {
    echo 'System rebooting...'
    reboot
}

menuentry "System shutdown" {
    echo 'System shutting down...'
    halt
}

### End /etc/grub.d/40_custom ###
```












<br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br><br>
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



**Check the EFI system partition (ESP)**

We will mount the ESP temporarily only to ensure that it is empty indeed. We are going to install the bootloader in ESP.

```
# mount /dev/sda1 /mnt
```

List all files in ESP. It should be empty.

```
# ls -la /mnt
```

Unmount ESP.

```
# umount -R /mnt
```

