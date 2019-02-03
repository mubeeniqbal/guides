# Install Arch Linux on VMware Workstation (UEFI/GPT)

## Drives

sda is 256 GB HDD

## Filesystem tree

```
[s] = subvolume
[d] = directory

/dev/sda1 [esp] (HDD) 1024M for EFI System Partition

/dev/sda2 [arch] (HDD)
|
+-- [s] rootvol      /
+-- [s] boot         /boot
+-- [s] opt          /opt
+-- [s] srv          /srv
+-- [s] home         /home
+-- [s] data         /data
+-- [s] var          /var
+-- [s] pacmanpkg    /var/cache/pacman/pkg
+-- [s] abs          /var/abs
+-- [s] var-cache    /var/cache
+-- [s] var-log      /var/log
+-- [s] var-spool    /var/spool
+-- [s] var-tmp      /var/tmp
+-- [d] snapshots
    |
    +-- [s] rootvol    /.snapshots
    +-- [s] home       /home/.snapshots
    +-- [s] var        /var/.snapshots
```

## OS Installation

**Verify the boot mode**

```shell
ls /sys/firmware/efi/efivars
```

If the directory does not exist, the system may be booted in BIOS or CSM mode.

**Connect to the internet**

```shell
ping -c 3 archlinux.org
```

Since this machine is on a wired connection so it might just work right away. Configuration is needed for wireless connections.

**Update the system clock**

```shell
timedatectl set-ntp true
timedatectl set-timezone Asia/Karachi
timedatectl status
```

**View disks and partitions**

```shell
lsblk
```

**Erase the partition table**

```shell
sgdisk --zap-all /dev/sda
# Zero out only the first 10000 blocks.
dd if=/dev/zero of=/dev/sda bs=1M count=10000 status=progress
```

**View disks and partitions again to ensure udev has given them the names (i.e. /dev/sdX where X is a, b, c, etc.) you'd want them to have**

```shell
lsblk
```

**Check UEFI mode (if UEFI variable print you're in UEFI mode)**

```shell
efivar -l
```

**Use cgdisk to create gpt partitions (UEFI and BIOS are going to have different partition layouts)**

```shell
cgdisk /dev/sda
```

- 1024 MB fat32 "esp" partition [ef00]
- Rest for btrfs "arch" partition [8300]


**Check if the disk partitions are set as desired**

```shell
lsblk
```

**Create filesystems**

```shell
mkfs.fat -F32 -n esp /dev/sda1
mkfs.btrfs -L arch /dev/sda2
```

**Mount main btrfs partitions**

```shell
mkdir -p /mnt/btrfs-root/arch
mount /dev/sda2 /mnt/btrfs-root/arch
```

**Create btrfs subvolumes for system**

```shell
btrfs subvolume create /mnt/btrfs-root/arch/rootvol
btrfs subvolume create /mnt/btrfs-root/arch/boot
btrfs subvolume create /mnt/btrfs-root/arch/opt
btrfs subvolume create /mnt/btrfs-root/arch/srv
btrfs subvolume create /mnt/btrfs-root/arch/home
btrfs subvolume create /mnt/btrfs-root/arch/data
btrfs subvolume create /mnt/btrfs-root/arch/var
btrfs subvolume create /mnt/btrfs-root/arch/pacmanpkg
# abs is Arch Build System. It's created inside var/abs but subvol is called abs; not var-abs.
btrfs subvolume create /mnt/btrfs-root/arch/abs
btrfs subvolume create /mnt/btrfs-root/arch/var-cache
btrfs subvolume create /mnt/btrfs-root/arch/var-log
btrfs subvolume create /mnt/btrfs-root/arch/var-spool
btrfs subvolume create /mnt/btrfs-root/arch/var-tmp
```

**Create btrfs subvolumes for snapshots**

```shell
mkdir -p /mnt/btrfs-root/arch/snapshots
btrfs subvolume create /mnt/btrfs-root/arch/snapshots/rootvol
btrfs subvolume create /mnt/btrfs-root/arch/snapshots/home
btrfs subvolume create /mnt/btrfs-root/arch/snapshots/var
```

**List all subvolumes for /mnt/btrfs-root/arch**

```shell
btrfs subvolume list -a /mnt/btrfs-root/arch
```

**Create directory to mount root subvolume and mount it**

```shell
mkdir -p /mnt/btrfs-active
mount -o subvol=rootvol /dev/sda2 /mnt/btrfs-active
```

**Create directories in this newly mounted subvolume and mount other subvolumes**

Don't change the order of the commands since to create directories inside another subvolume it should be mounted first.

```shell
mkdir -p /mnt/btrfs-active/{boot,home,data,var,opt,srv,.snapshots}
mount -o subvol=boot /dev/sda2 /mnt/btrfs-active/boot
mount -o subvol=home /dev/sda2 /mnt/btrfs-active/home
mount -o subvol=data /dev/sda2 /mnt/btrfs-active/data
mount -o subvol=var /dev/sda2 /mnt/btrfs-active/var
mount -o subvol=opt /dev/sda2 /mnt/btrfs-active/opt
mount -o subvol=srv /dev/sda2 /mnt/btrfs-active/srv
mount -o subvol=snapshots/rootvol /dev/sda2 /mnt/btrfs-active/.snapshots

mkdir -p /mnt/btrfs-active/var/{abs,cache,log,spool,tmp,.snapshots}
mkdir -p /mnt/btrfs-active/home/.snapshots
mount -o subvol=abs /dev/sda2 /mnt/btrfs-active/var/abs
mount -o subvol=var-cache /dev/sda2 /mnt/btrfs-active/var/cache
mount -o subvol=var-log /dev/sda2 /mnt/btrfs-active/var/log
mount -o subvol=var-spool /dev/sda2 /mnt/btrfs-active/var/spool
mount -o subvol=var-tmp /dev/sda2 /mnt/btrfs-active/var/tmp
mount -o subvol=snapshots/home /dev/sda2 /mnt/btrfs-active/home/.snapshots
mount -o subvol=snapshots/var /dev/sda2 /mnt/btrfs-active/var/.snapshots

mkdir -p /mnt/btrfs-active/var/cache/pacman/pkg
mount -o subvol=pacmanpkg /dev/sda2 /mnt/btrfs-active/var/cache/pacman/pkg
```

**List all subvolumes for `/mnt/btrfs-active`**

```shell
btrfs subvolume list -a /mnt/btrfs-active
```

**Mount EFI System Partition**

```shell
mkdir -p /mnt/btrfs-active/boot/efi
mount /dev/sda1 /mnt/btrfs-active/boot/efi
```

**Select a mirror**

```shell
nano /etc/pacman.d/mirrorlist
```

**Check internet connection**

```shell
ping -c 3 www.google.com
```

It should already be connected since this machine has a wired connection.

**Install the base system**

```shell
pacstrap /mnt/btrfs-active base base-devel btrfs-progs
```

In case pacstrap errors out:

```shell
# Try running this command first
pacman -Sy archlinux-keyring && pacman-key --refresh-keys
# and then run pacstrap again.
pacstrap /mnt/btrfs-active base base-devel btrfs-progs
```

**Generate an `fstab`**

```shell
genfstab -U -p /mnt/btrfs-active >> /mnt/btrfs-active/etc/fstab
nano /mnt/btrfs-active/etc/fstab
```

```
# Make /tmp a ramdisk (not needed since Arch Linux mounts /tmp on tmpfs)
# tmpfs is used for /tmp by the default systemd setup and does not require an entry in fstab unless a specific configuration is needed.
tmpfs    /tmp    tmpfs    rw    0    0

# /dev/sda2 LABEL=arch
UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX    /    btrfs    rw,relatime,discard,ssd,space_cache,subvol=rootvol    0    0

# /dev/sdb1 LABEL=data
UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX    /home    btrfs    rw,relatime,space_cache,subvol=home    0    0

# /dev/sdb1 LABEL=data
UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX    /var    btrfs    rw,relatime,space_cache,subvol=var    0    0

# /dev/sda1 LABEL=bios
UUID=XXXX-XXXX    /boot    vfat    rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro    0    2
```

**Chroot into the base system**

```shell
arch-chroot /mnt/btrfs-active /bin/bash
```

**Set timezone**

```shell
ln -s /usr/share/zoneinfo/Asia/Karachi /etc/localtime
```

**Set hardware clock**

```shell
hwclock --systohc --utc
```

**Setup locale to `en_US` (uncomment `en_US.UTF-8 UTF-8`)**

```shell
nano /etc/locale.gen
```

```
...
#en_SG ISO-8859-1
en_US.UTF-8 UTF-8
#en_US ISO-8859-1
...
```

**Generate the locale(s) specified in `/etc/locale.gen`**

```
locale-gen
```

**Create the `/etc/locale.conf` file substituting your chosen locale**

```shell
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

**Set console font and keymap to default i.e. leave entries empty**

```shell
nano /etc/vconsole.conf
```

```
KEYMAP=
FONT=
```

**Set Hostname**

```shell
echo servo > /etc/hostname
```

**Add the same hostname to `/etc/hosts`**

```shell
nano /etc/hosts
```

```
#
# /etc/hosts: static lookup table for host names
#

#<ip-address>	<hostname.domain.org>	<hostname>
127.0.0.1	localhost.localdomain	localhost
::1		localhost.localdomain	localhost
127.0.1.1	servo.localdomain	servo

# End of file
```

**Configure the network (this time for your newly installed environment)**

```shell
ip link

# For wireless network run the commands below
pacman -S iw wpa_supplicant
# Install dialog which is needed by wifi-menu
pacman -S dialog
```

Do not run wifi-menu at this point into your chrooted environment as it will conflict with the one running outside the chrooted environment.

**Connect automatically to known networks**

```shell
# Enable netctl-auto service for automatically connecting to wireless connections.
pacman -S wpa_actiond
systemctl enable netctl-auto@<interface_name>.service
  
# No need to use ifplugd since dhcpcd provides the same feature out of the box.
> # enable netctl-ifplugd service for automatically connecting to wired connections.
> pacman -S ifplugd
> systemctl enable netctl-ifplugd@<interface_name>.service

# Enable dhcpcd to connect to known networks.
systemctl enable dhcpcd.service

# dhcpcd.service can be enabled without specifying an interface.
# This may, however, create a race condition at boot with systemd-udevd trying to apply a predictable network interface name:
systemctl enable dhcpcd@enp2s0.service
```

**Create an initial ramdisk environment**

```shell
# Remove "fsck" (maybe "fsck" works now so don't remove it) and add "btrfs" in HOOKS.
nano /etc/mkinitcpio.conf
mkinitcpio -p linux
```

**Set root password**

```shell
passwd
```

**Install and configure bootloader (GRUB)**

```shell
pacman -S grub efibootmgr
```

**Install boot files**

```shell
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch-grub
grub-mkconfig -o /boot/grub/grub.cfg
```

**Install intel microcode package for microprocessor updates at boot**

```shell
pacman -S intel-ucode
```

**Regenerate grub config to activate loading microcode updates**

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

**Optionally set grub timeout to zero**

```shell
cp /etc/default/grub /etc/default/grub.backup
nano /etc/default/grub
```

```
...
GRUB_TIMEOUT=0
...
```

Remember that `grub.cfg` has to be re-generated after any change to `/etc/default/grub` or files in `/etc/grub.d/`.

```shell
grub-mkconfig -o /boot/grub/grub.cfg
```

**Unmount the partitions and reboot**

```shell
exit

umount -R /mnt/btrfs-active/boot/efi
umount -R /mnt/btrfs-active
umount -R /mnt/btrfs-root/arch

# Reboot into your newly installed Arch Linux system.
reboot
```

---

**Milestone reached: Base system installed!**

---

**Install snapper to create and maintain btrfs subvolume snapshots**

```shell
pacman -S snapper
```

**Create snapper configs for the subvolumes (`rootvol`, `home` and `var`). This will generate a subvolume named `.snapshots` in the root of (i.e. directly under) each of the subvolumes.**

```shell
snapper -c root create-config /
snapper -c home create-config /home
snapper -c var create-config /var
```

**The above `create-config` commands will fail since we have already created the `.snapshots` subvolume for each of the configs because we wanted the snapshots to reside on `subvolid=0` and not be a child of the root subvolume. Hence, we will have to create the configs manually.**

```shell
cp -vf /etc/snapper/config-templates/default /etc/snapper/configs/root
cp -vf /etc/snapper/config-templates/default /etc/snapper/configs/home
cp -vf /etc/snapper/config-templates/default /etc/snapper/configs/var
```

**Check that the subvolume is set to the mount point of the subvolume you want to snapshot for that config.**

```shell
nano /etc/snapper/configs/root
```

```
...
SUBVOLUME="/"
...
```

```shell
nano /etc/snapper/configs/home
```

```
...
SUBVOLUME="/home"
...
```

```shell
nano /etc/snapper/configs/var
```

```
...
SUBVOLUME="/var"
...
```

**Add the config name to `/etc/conf.d/snapper`**

```shell
nano /etc/conf.d/snapper
```

```
...
SNAPPER_CONFIGS="root home var"
...
```

**Install `mlocate` to get the `locate` and `updatedb` utilities**

```shell
pacman -S mlocate
```

**Run updatedb to create the file `/etc/updatedb.conf`**

```shell
updatedb
```

**By default updatedb will also index the .snapshots directory to save snapshots, which can cause serious slowdown and excessive memory usage if you have many snapshots. You need to prune .snapshots from updatedb so the updatedb ignores the .snapshots directories. For that add .snapshots to the whitespace-separate-list PRUNENAMES in /etc/updatedb.conf. Try adding the entries in alphabetical order.**

```shell
nano /etc/updatedb.conf
```

```
...
PRUNENAMES="... .snapshots ..."
...
```

---

**Milestone reached: Snapper installed and configured!**

---

**Now you can take a snapshot of your system for `root`, `home` and `var` subvolumes.**

```shell
# Reboot right before taking the first system snapshot so that we have a fresh system image
reboot
snapper -c root create --description "Initial system snapshot" && snapper -c home create --description "Initial system snapshot" && snapper -c var create --description "Initial system snapshot"
```

**List all the snapshots**

```shell
snapper list-configs
snapper -c root list && snapper -c home list && snapper -c var list
```

---

**Milestone reached: System snapshot taken

---

- Install zsh shell
  # pacman -S zsh zsh-completions

- Check that zsh is added to /etc/shells
  # nano /etc/shells
  
  > #
  > # /etc/shells
  > #
  > 
  > /bin/sh
  > /bin/bash
  > /bin/zsh
  > /usr/bin/zsh
  > 
  > # End of file

- Set zsh as default shell for current user
  # echo $SHELL
  # zsh
  # echo $SHELL
  # chsh -s $(which zsh)
  # reboot
  # echo $SHELL

- Configure zsh to be the same as the Arch Linux monthly ISO releases
  The configuration is globally changed by the addition of /etc/zsh/zshrc from the package installation in this step
  # pacman -S grml-zsh-config

- Copy skeleton shell configuration files to the home directory
  (doing this just to keep the files in the user's home directory. we'll change these files in the next steps so that the changes are not global and affect the user only.)
  # cp -vf /etc/skel/.bash_logout ~/
  # cp -vf /etc/skel/.bash_profile ~/
  # cp -vf /etc/skel/.bashrc ~/
  # cp -vf /etc/skel/.zshrc ~/
  # reboot

- Add Fish like syntax highlighting
  # pacman -S zsh-syntax-highlighting
  # echo '# Fish like syntax highlighting' >> ~/.zshrc
  # echo 'source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh' >> ~/.zshrc
  (Make the "END OF FILE" comment the last line in ~/.zshrc)
  # nano ~/.zshrc
  # reboot

- Install pkgfile package to add command not found hook
  # pacman -S pkgfile
  # pkgfile --update

- Check if the hook is working
  # pkgfile makepkg
  # pkgfile -l archlinux-keyring

- Add command not found hook
  # abiword (shouldn't show the hook right now)
  # echo '# Command not found hook' >> ~/.bashrc
  # echo 'source /usr/share/doc/pkgfile/command-not-found.bash' >> ~/.bashrc
  (fix formatting)
  # nano ~/.bashrc
  # echo '# Command not found hook' >> ~/.zshrc
  # echo 'source /usr/share/doc/pkgfile/command-not-found.zsh' >> ~/.zshrc
  (Make the "END OF FILE" comment the last line in ~/.zshrc)
  # nano ~/.zshrc
  # reboot
  # abiword (should show the command not found hook now)

--- configured zsh -------------------

- Enable color outputs for pacman
  # sudo nano /etc/pacman.conf
  
  > #Color (uncomment this line)
  > Color

- Add ILoveCandy to the end of Miscellaneous section in /etc/pacman.conf
  This makes the progress bars to look like Pacman from the game eating powerpills.
  # nano /etc/pacman.conf
  
  > # Misc options
  > ...
  > Color
  > ...
  > ILoveCandy

- Backup pacman mirrors list (/etc/pacman.d/mirrorlist)
  # sudo cp -vf /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup

- Install and use Reflector to sort mirror list
  # sudo pacman -S reflector
  # sudo reflector --verbose --protocol http --latest 100 --fastest 50 --sort rate --save /etc/pacman.d/mirrorlist

--- installed reflector and updated pacman mirrors list --------------

- Install the openssh package
  # pacman -S openssh

- To add a nice welcome message (e.g. from the /etc/issue file), configure the Banner option:
  # cp -vf /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
  # nano /etc/ssh/sshd_config

  > ...
  > Banner /etc/issue
  > ...

- Enable and start the sshd daemon
  (sshd.service can also be used but sshd.socket is the recommended way to run sshd in almost all cases.)
  # systemctl enable sshd.socket
  # systemctl start sshd.socket

- If the server is to be exposed to the WAN, it is recommended to change the default port from 22 to a random higher one
  # cat /etc/services | grep 39901 (check if port 39901 is already being used)

- If using the socket service, you will need to edit the unit file if you want it to listen on a port other than the default 22.
  # systemctl edit sshd.socket
  
  > ...
  > [Socket]
  > ListenStream=39901
  > ...

  # cat /etc/systemd/system/sshd.socket.d/override.conf

- Although we are not using sshd.service but still change the port for sshd.service as well to be consistent.
  # nano /etc/ssh/sshd_config

  > ...
  > Port 39901
  > ...

- SSH Protection: Generate SSH key pair on the SSH client machine and NOT on the SSH server machine. Client machine is the one to remotely connect to the SSH server machine.
  # ssh-keygen -t ed25519 -o -C "$(whoami)@$(hostname)-$(date -I)"
  OR (for macOS)
  # ssh-keygen -t ed25519 -o -C "$(whoami)@$(hostname)-$(date +%Y-%m-%d)"
  (when the key is being generated on the client name the file as id_ed25519_servo rather than it just being id_ed25519. Also provide the full path to where the file will reside and not just the name of the file. Then provide the passphrase for the file)

  (later if you ever need to change the passphrase for the private key without changing the key itself use this command: # ssh-keygen -f ~/.ssh/id_ed25519_servo -p)

- SSH Protection: Edit SSH client config file for managing SSH host keys on the SSH client machine and NOT on the SSH server machine.
  # nano ~/.ssh/config

  > ...
  > # global options
  > #User mubeen
  > 
  > # host-specific options
  > Host servo
  >     HostName 192.168.x.x
  >     Port 39901
  >     IdentitiesOnly yes
  >     IdentityFile ~/.ssh/id_ed25519_servo
  > ...

  (provide the HostName for your SSH server machine by looking up it’s ip address over LAN)

- SSH Protection: Copying the public key to the remote SSH server.
  Copy the ~/.ssh/id_ed25519_servo.pub file to a usb storage device and then plug the usb storage device into the remote SSH server machine.
  # mkdir -p /mnt/pny (let’s say the device label is pny so we just call the directory that for simplicity’s sake)
  # mount /dev/sdb1 /mnt/pny
  # mkdir ~/.ssh
  # chmod 700 ~/.ssh
  # cat /mnt/pny/id_ed25519_servo.pub >> ~/.ssh/authorized_keys
  (By default, for OpenSSH, the public key needs to be concatenated with ~/.ssh/authorized_keys)
  # chmod 600 ~/.ssh/authorized_keys
  # cat ~/.ssh/authorized_keys
  # umount /mnt/pny
  # rm -r /mnt/pny

- SSH Protection: Force public key authentication
  # nano /etc/ssh/sshd_config

  > ...
  > PasswordAuthentication no
  > ChallengeResponseAuthentication no
  > ...

- Now reboot the system
  # reboot

TODO: Configure ssh-agent (or better yet keychain) on the SSH client machine to remember ssh keypass.
TODO: User Dynamic DNS to have a static domain name against IP address.

--- ssh configured ----------------

- Install sudo package
  # pacman -S sudo

- Uncomment the wheel group
  # visudo
  
  > %wheel ALL=(ALL) ALL

- Create new user
  (Add all root like admin users to the wheel group rather than adding them to root. The wheel group has the root access and so will any user inside the wheel group. We don’t want to keep changing the sudoers file.)
  # useradd -m -G wheel -s /usr/bin/zsh -c "Mubeen Iqbal" mubeen
  # passwd mubeen
  # reboot

- Login with new user and set finger data for user.
  (check if the name is already set or not)
  # sudo chfn -f "Mubeen Iqbal" mubeen

- Remove trailing commas (if there are any) manually from user finger data
  # sudo nano /etc/passwd
  # cat /etc/passwd

--- added new user ---------------------

- Copy .bashrc and .zshrc from root home to new user home since we made some changes in them back in the root account and they are now different than the skeleton versions (ones provided by /etc/skel)
  # sudo cp -vf /root/.bash_logout ~/
  # sudo cp -vf /root/.bash_profile ~/
  # sudo cp -vf /root/.bashrc ~/
  # sudo cp -vf /root/.zshrc ~/
  # sudo cp -vfR /root/.ssh ~/

  (configure ssh for the new user. We will use the same settings as root)
  # sudo chown mubeen:mubeen -R ~/.ssh
  # chmod 700 ~/.ssh
  # chmod 600 ~/.ssh/authorized_keys

- Configure the zsh shell for new user (I don’t need this since the grml configuration for zsh which comes from the grml package installation is enough for me)
  # zsh /usr/share/zsh/functions/Newuser/zsh-newuser-install -f

--- configured zsh for new user ----------------

- Install git to pull any scripts that we might need later on
  # sudo pacman -S git
  # git config --global user.name  "Mubeen Iqbal"
  # git config --global user.email "mubeen.ace@gmail.com"

TODO: configure git properly (https://wiki.archlinux.org/index.php/git)

--- git installed -----------------------------------

- Create a git repositories folder to keep all your repositories (do this for user mubeen and not root)
  # mkdir -vp ~/git-repos

- Clone snp and copy to /usr/local/bin to wrap commands with pre-post snapshots
  (This snp script was forked and improved by me. This script is only for running commands wrapped inside pre-post snapshots. It’s not meant for single snapshots.)
  # cd ~/git-repos
  # git clone https://gist.github.com/81fd74d5e8cf522c780f.git snp
  # sudo cp -vf ~/git-repos/snp/snp /usr/local/bin/

TODO: add snapper helper script "snp" from my gists with aliases (https://wiki.archlinux.org/index.php/Snapper)
TODO: add snapper helper script "rollback" from my gists with aliases (https://wiki.archlinux.org/index.php/Snapper)
TODO: add pacman aliases (https://wiki.archlinux.org/index.php/Pacman_tips#Shortcuts)
TODO: configure bash properly (https://wiki.archlinux.org/index.php/bash)
TODO: configure nano, vi, emacs, etc. for syntax highlighting

--- snapper helper scripts configured ----------------------

- Install Yaourt to seamlessly install packages from AUR but to do that since Yaourt itself is in AUR you we will have to install it from AUR the traditional way. Create a builds directory to download tarballs
  # mkdir -vp ~/builds

- First install Yaourt’s dependency
  # cd ~/builds
  # git clone https://aur.archlinux.org/package-query.git
  # cd package-query
  # makepkg -si
  # cd ..
  
- Then install Yaourt itself
  # cd ~/builds
  # git clone https://aur.archlinux.org/yaourt.git
  # cd yaourt
  # makepkg -si
  # cd ..

- Check if yaourt is installed, then reboot
  # cd ~
  # yaourt -h
  # sudo reboot

- Remove the builds folder since we no longer need it (and the tarballs+builds)
  # rm -vrf ~/builds

--- installed Yaourt for AUR package management -----------------------

- Take system snapshot before installing plex media server
  # sudo snapper -c root create --description "Before installing plex-media-server" && sudo snapper -c home create --description "Before installing plex-media-server" && sudo snapper -c var create --description "Before installing plex-media-server"

- Install plex media server from AUR
  # yaourt -S plex-media-server
  # sudo reboot
  # sudo systemctl enable plexmediaserver.service
  # sudo systemctl start plexmediaserver.service
  # sudo reboot

- Make sure the all the media has permissions set to 755. This also allows the user plex and group plex to read and execute files because others (the last 5 in 755) is given the permission 5 i.e. r-x
  # sudo chmod 755 -R /data/media

- Add a plex-versions directory. This is where all optimized versions of files for Plex will go so that we can keep them all in one place. This makes it very convenient to perform a clean up (deleting optimized versions files).
  # mkdir -p /data/media/plex-versions
  (We add this directory as the library folder in Plex for all the libraries so that it appears under the list to store optimized version files all in one place)

- Change the owner and group of the plex-versions directory recursively to be plex and plex respectively so that Plex can write to these directories since Plex runs as the user plex in the group plex.
  # sudo chown -R plex:plex /data/media/plex-versions

- Make sure that the permissions are set to 755 recursively. The 7 permissions will allow user plex to read, write and execute files in the directory plex-versions and it's sub-directories.
  # sudo chmod 755 -R /data/media/plex-versions

- Access plex media server using any browser on any machine. Visit http://192.168.10.2:32400/web/index.html
  (assuming 192.168.10.2 is the ip address of the media server machine)

--- installed Plex Media Server -----------------------

- Install htop to list processes and system resource usage
  # sudo pacman -S htop



Keep your system up to date:

- Update mirrorlist using reflector to get the fastest mirrors
  # sudo reflector --verbose --protocol http --latest 100 --fastest 50 --sort rate --save /etc/pacman.d/mirrorlist

- Full system official package upgrade
  # sudo pacman -Syu

- Full system official package plus AUR package upgrade
  # yaourt -Syua

- Update pkgfile DB
  # sudo pkgfile -u

- Clean package cache
  # sudo pacman -Sc
  # sudo paccache -r
  # sudo paccache -ruk0

- Update 'locate' DB
  # sudo updatedb



TODO: Read up on https://wiki.archlinux.org/index.php/General_recommendations to further improve this guide.

Notes:
- At the end of installation check that the timedate synchronization service is running
  # timedatectl status
  # systemctl status systemd-timesyncd.service
  (if it’s not running then turn it on using
  # timedatectl set-ntp true)

#!/bin/bash

# Filesystem tree.
# [s] = subvolume
# [d] = directory

# /dev/sda1 [esp] (HDD) 1024M for EFI System Partition

# /dev/sda2 [arch] (HDD)
# |
# +-- [s] rootvol      /
# +-- [s] boot         /boot
# +-- [s] opt          /opt
# +-- [s] srv          /srv
# +-- [s] home         /home
# +-- [s] data         /data
# +-- [s] var          /var
# +-- [s] pacmanpkg    /var/cache/pacman/pkg
# +-- [s] abs          /var/abs
# +-- [s] var-cache    /var/cache
# +-- [s] var-log      /var/log
# +-- [s] var-spool    /var/spool
# +-- [s] var-tmp      /var/tmp
# +-- [d] snapshots
#     |
#     +-- [s] rootvol    /.snapshots
#     +-- [s] home       /home/.snapshots
#     +-- [s] var        /var/.snapshots


# Create filesystems
mkfs.fat -F32 -n esp /dev/sda1
mkfs.btrfs -L arch /dev/sda2

# Mount main btrfs partition
mkdir -p /mnt/btrfs-root
mount /dev/sda2 /mnt/btrfs-root

# Create btrfs subvolumes for system
btrfs subvolume create /mnt/btrfs-root/rootvol
btrfs subvolume create /mnt/btrfs-root/boot
btrfs subvolume create /mnt/btrfs-root/opt
btrfs subvolume create /mnt/btrfs-root/srv
btrfs subvolume create /mnt/btrfs-root/home
btrfs subvolume create /mnt/btrfs-root/data
btrfs subvolume create /mnt/btrfs-root/var
btrfs subvolume create /mnt/btrfs-root/pacmanpkg
btrfs subvolume create /mnt/btrfs-root/abs
# abs is Arch Build System. It's created inside var/abs but subvol is
# called abs; not var-abs
btrfs subvolume create /mnt/btrfs-root/var-cache
btrfs subvolume create /mnt/btrfs-root/var-log
btrfs subvolume create /mnt/btrfs-root/var-spool
btrfs subvolume create /mnt/btrfs-root/var-tmp

# Create btrfs subvolumes for snapshots
mkdir -p /mnt/btrfs-root/snapshots
btrfs subvolume create /mnt/btrfs-root/snapshots/rootvol
btrfs subvolume create /mnt/btrfs-root/snapshots/home
btrfs subvolume create /mnt/btrfs-root/snapshots/var

# List all subvolumes for /mnt/btrfs-root
btrfs subvolume list -a /mnt/btrfs-root




# Unmount
umount -R /mnt/btrfs-root

# Create directory to mount root subvolume and mount it
mount -o subvol=rootvol /dev/sda2 /mnt/btrfs-root

# Create directories in this newly mounted subvolume and mount other subvolumes
# don't change the order of the commands since to create directories inside
# another subvolume it should be mounted first
mkdir -p /mnt/btrfs-root/{boot,home,data,var,opt,srv,.snapshots}

mount -o subvol=boot /dev/sda2 /mnt/btrfs-root/boot
mount -o subvol=home /dev/sda2 /mnt/btrfs-root/home
mount -o subvol=data /dev/sda2 /mnt/btrfs-root/data
mount -o subvol=var /dev/sda2 /mnt/btrfs-root/var
mount -o subvol=opt /dev/sda2 /mnt/btrfs-root/opt
mount -o subvol=srv /dev/sda2 /mnt/btrfs-root/srv
mount -o subvol=snapshots/rootvol /dev/sda2 /mnt/btrfs-root/.snapshots

mkdir -p /mnt/btrfs-root/var/{abs,cache,log,spool,tmp,.snapshots}
mkdir -p /mnt/btrfs-root/home/.snapshots

mount -o subvol=abs /dev/sda2 /mnt/btrfs-root/var/abs
mount -o subvol=var-cache /dev/sda2 /mnt/btrfs-root/var/cache
mount -o subvol=var-log /dev/sda2 /mnt/btrfs-root/var/log
mount -o subvol=var-spool /dev/sda2 /mnt/btrfs-root/var/spool
mount -o subvol=var-tmp /dev/sda2 /mnt/btrfs-root/var/tmp
mount -o subvol=snapshots/home /dev/sda2 /mnt/btrfs-root/home/.snapshots
mount -o subvol=snapshots/var /dev/sda2 /mnt/btrfs-root/var/.snapshots

mkdir -p /mnt/btrfs-root/var/cache/pacman/pkg

mount -o subvol=pacmanpkg /dev/sda2 /mnt/btrfs-root/var/cache/pacman/pkg

# List all subvolumes for /mnt/btrfs-root
btrfs subvolume list -a /mnt/btrfs-root

# Mount EFI System Partition
mkdir -p /mnt/btrfs-root/boot/efi
mount /dev/sda1 /mnt/btrfs-root/boot/efi










Btrfs note: COW can negatively affect performance with large files that have small random writes. Disable COW for databases and virtual machine images.
    To disable CoW for single files/directories do:
    $ chattr +C /dir/file
