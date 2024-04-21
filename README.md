# Installing Arch Linux - A "Simple" Guide
*Get started using Arch Linux and familiarize with the intricacies of its use. This guide includes a tutorial on how to install arch linux with full disk encryption onto any computer.*

 
## Introduction
 Installing Arch Linux as a first-time Linux distribution is not recommended. Arch Linux has less protection against breaking your own system, rather it is better to start with a simple more user friendly Linux distribution such as Ubuntu.

 This guide assumes you have prior Linux experience from distributions such as Ubuntu. The goal of this tutorial is to make the transition as smooth as possible, and take the next step in your journey as a Linux user. 

## Installation Guide

### 1. Burn an Arch Linux image
Before you can install Arch Linux on a new machine, you will need to put an Arch Linux disk image on a USB stick. You can download the latest Arch Linux disk image from: https://archlinux.org/download/. Just scroll down until you see the different mirrors for your country and pick one from there.

Once you have downloaded the disk image it is time to restore it. This process varies depending on which operating system you have.

#### Linux Systems with GNOME Desktop

1. Search for "Disks" and open the disks tool by GNOME.
2. Select the USB stick from the list of disks on the left side of the window.
3. Once you have selected the disk click the three dots on the top right, and select "Restore Disk Image".
4. Select your Arch Linux disk image and burn it onto the disk.

#### Windows systems

1. Download https://win32diskimager.org/
2. Use the Win32DiskImager tool to burn the downloaded Arch Linux image onto your USB.

### 2. Boot onto your USB flash drive.

Now you will need to boot onto your USB flash drive by restarting your computer and accessing the BIOS menu. This is usually done by repeatedly pressing the `DEL` or `F12` key during boot. However, the exact key might vary for your system therefore it is best if you look this up for yourself. 

Once you have accessed the BIOS select the USB drive from your boot menu/boot priority and boot from your USB drive.

### 3. Install Arch Linux

*This part of the guide was largely inspired by https://github.com/huntrar and their https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268. This part of the tutorial is merely a copy-paste with some clarifications that I found necessary to get a better understanding of what we are doing.*

#### Connect to internet (wireless)
*If you are connected via Ethernet you should already be connected, and therfore you can skip this section*.


**List Available Wi-Fi Devices**:
   Start by identifying your wireless device.
   ```bash
   iwctl device list
   ```
   This will display the wireless devices available on your system. Note the device identifier, usually something like `wlan0`.

**Turn on the Wi-Fi device**:
   If the output shows your wireless device is "off", you need to turn it on using:
   ```bash
   iwctl device wlan0 set-property Powered on
   ```
**Scan for Wi-Fi Networks**:
   Begin scanning for available networks by using the following command:
   ```bash
   iwctl station wlan0 scan
   ```
   You might need to wait a few seconds for the scan to complete.

**List Detected Networks**:
   To see a list of available Wi-Fi networks:
   ```bash
   iwctl station wlan0 get-networks
   ```
   Identify the SSID of the network you wish to connect to from this list.

**Connect to a Wi-Fi Network**:
   To connect to the network:
   ```bash
   iwctl station wlan0 connect "SSID"
   ```
   Replace “SSID” with the actual SSID of your network. If the network is protected by a password, you will be prompted to enter it. Enter the password when prompted.

**Verify Connection**:
   Once connected, check the IP address assigned to your device with:
   ```bash
   ip a
   ```
   Look for the `wlan0` interface and ensure it has an IP address.

**Test Internet Accessibility**:
   Confirm you have internet access:
   ```bash
   ping -c 4 archlinux.org
   ```
   If you see responses, your internet connection is set up correctly.


#### Set the system clock

```
timedatectl set-ntp true
```

#### Use GDisk to create new partitions

**Find out what the name of your disk is**
```bash
lsblk
```
You should see your main disk come up. In my case it is called `/dev/sda`. It is important that you select the correct disk. Usually you can enumerate the correct disk by the process of elimination by looking at the disk size. I.e. if your only disk is 1TB, look for a 1TB disk.

**Create BIOS boot, EFI and LUKS partitions**

Use the command and replace the `<your_drive_here>` with the disk name that you found before:
```bash
gdisk <your_drive_here>
```

In my case this becomes: `gdisk /dev/sda`. This command allows us to make changes to our disk partitions.

Now its time to create the new partitions (the command might look strange but this is because gdisk uses single key commands to perform tasks):

1. Create a new empty GUID partition table: `o`. A warning might appear saying "This option deletes all partitions and creates a new protective MBR". In this case press `Y`. Do note that your data will be very hard to recover after this point. Make sure your important files are backed up.
2. Create a new BIOS boot partition: `n`. The partition number should be **1**, first sector **0**, last sector **+1M**, hex code or GUID **ef02**.  
3. Create a new EFI partition: `n`. The partition number should be **2**, first sector should be the default value, last sector should be **+550M**, hex code or GUID should be **ef00**.
4. Create a new LUKS encrypted partition: `n`. The partition number should be **3**, first sector should be the default value, last sector should be the default value, hex code or GUID should be **8309**.
5. Print your partition table and verify the details: `p`. You should have **three** partitions: 
    - BIOS boot partition 1024 KiB
    - EFI system partition 550 MiB
    - Linux LUKS (the size depends on your disk size).
6. If your partition table matches what I explained in the previous step you can write your partition table to the disk using `w`. If a warning comes up, again make sure your data is backed up and press `Y`.

Create the LUKS1 encrypted container on the Linux LUKS partition (GRUB does not support LUKS2 as of May 2019)
cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p3

#### Create the LUKS encrypted container
The LUKS encrypted container is the logical container that will store all of our files. Replace `<your_luks_partition>` with the appropriate partition name. In my case it is `/dev/sda3`.
```
cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 <your_luks_partition>
```

1. **`-S` (or `--key-slot`)**: 
   - This option specifies the key slot to be used for the new key. LUKS can support multiple key slots. In this case, `-S 1` specifies that key slot 1 should be used.

2. **`-s` (or `--key-size`)**:
   - This sets the size of the key used in bits. `-s 512` means a 512-bit key is used for the encryption. The size of the encryption key is crucial for determining the strength of the encryption.

3. **`-h` (or `--hash`)**:
   - This specifies the hash algorithm used to hash the passphrase and other data. In your command, `-h sha512` indicates that the SHA-512 (Secure Hash Algorithm 512-bit) is used. Hash algorithms are used in LUKS for various purposes such as deriving the key from the passphrase.

4. **`-i` (or `--iter-time`)**:
   - This sets the maximum number of milliseconds to spend with passphrase processing (PBKDF2 passphrase processing time). `-i 50000` means that the process will take about 5000 milliseconds (or 5 seconds). This is a measure to defend against brute force attacks, making each guess slower.

You will be prompted to type in yes in capital letters hence do: `YES`.

Now you will be prompted to enter an encryption passphrase, this is a crucial step to ensure security and privacy of your data. Use a randomly generated password, **do not come up with one yourself** as humans are not good at generating entropy.

#### Open the LUKS container
Replace `<your_luks_partition>` with the appropriate partition name. In my case it is `/dev/sda3`.

```
cryptsetup open <your_luks_partition> cryptlvm
```

#### Create the physical volume on top of the opened LUKS container
This step is irrelevant of your device/partition names, it should work for everyone.
```
pvcreate /dev/mapper/cryptlvm
```

#### Create the volume group and add physical volume to it.
This step is irrelevant of your device/partition names, it should work for everyone.
```
vgcreate vg /dev/mapper/cryptlvm
```

#### Create logical volumes on the volume group for swap, root, and home.
Your swap partition is for your memory, the root partition will store system files and your home directory will store your personal files. It is important to not be too greedy with disk size and give your root partitiona a good size, if you install many packages, programs, libraries your root partition will get eaten up quickly. Therefore, I choose to set my root partition at `120GB`. If you run out of disk space, you will have to perform cumbersome operations to extend it later.

**Create the swap partition**
```
lvcreate -L 8G vg -n swap
```

**Create the root partition**
```
lvcreate -L 120G vg -n root
```

**Create the home partition**
Here we use a keyword 100%FREE which will take up the remaining space in the logical volume group.
```
lvcreate -l 100%FREE vg -n home
```

#### Format filesystems on each logical volume
We set the root and home volumes to have filesystem formats EXT4, a common linux filesystem format.
```
mkfs.ext4 /dev/vg/root
mkfs.ext4 /dev/vg/home
mkswap /dev/vg/swap
```

#### Mount filesystems
At the moment, we can still not access these volumes that we created, we must place them onto a folder using the mount command. The swapon command activates our swap volume.

**Mount the root volume**
```
mount /dev/vg/root /mnt
```

**Create a folder for our home directory**
```
mkdir /mnt/home
```

**Mount our home volume to the home directory**
```
mount /dev/vg/home /mnt/home
```

**Activate swap volume**
```
swapon /dev/vg/swap
```

#### Prepare the EFI partition
Now we create a fat32 filesystem for the EFI system partition. Replace <your_efi_partition_name> with your efi partition name, in my case it is `/dev/sda2`.
```
mkfs.fat -F32 <your_efi_partition_name>
```

#### Mount the efi partition

**Create a directory for the EFI partition**
```
mkdir /mnt/efi
```

**Mount the EFI partition to the created directory**
Replace <your_efi_partition_name> with your efi partition name, in my case it is `/dev/sda2`.
```
mount <your_efi_parition_name> /mnt/efi
```

#### Install necessary packages
Pacstrap is a tool that helps us install linux firmware, tools and other things onto our system. 
```
pacstrap /mnt base linux linux-firmware mkinitcpio lvm2 dhcpcd wpa_supplicant
```

#### Generate an fstab file
An fstab file is a type of file that allows the system to mount our disks for us automatically. Otherwise we would have to mount them every time we turn on the computer.
```
genfstab -U /mnt >> /mnt/etc/fstab
```

#### Enter new system chroot
chroot is a short for "change root". This basically emulates as if we are inside of the newly installed system.
```
arch-chroot /mnt
```


#### Set the time zone
Replace `<your_time_zone>` with your respective timezone found in `/usr/share/zoneinfo`. You can view the different zone files by doing `cd /usr/share/zoneinfo`, followed by `ls`. 
Example: `/usr/share/zoneinfo/America/Los_Angeles`
```
ln -sf <your_time_zone> /etc/localtime
```

#### Sync the hardware clock
Ensures that the time remains consistent between reboots. This is crucial because many system processes depend on accurate timekeeping.
```
hwclock --systohc
```

#### Install nano
Since none of us know how to exit vim/vi my go-to editor is `nano`. Sadly, this cute tiny little editor is not included which is why we will add it using:
```
pacman -S nano
```
**pacman is the default package manager in Arch Linux**

#### Set your locale
I am going to set the en_US UTF8 locale. To set your locale simply do `nano /etc/locale.gen`. Use your arrow keys to traverse the text file. Go down until you see your wanted locale and remove the first "#" that all locales are prepended with.

Once you have uncommented the locale do `ctrl + x`, `y` and then press enter to save the file. 

#### Generate the locale
Now we need to generate the locale. You should see the name of your locale show up during generation.
```
locale-gen
```

#### Configure the locale config file.
We need to create a configuration file for the locale.
We can do this with nano.

```
nano /etc/locale.conf
```

In there type the following replacing <your_locale_name> with your locale name. In my case it is `en_US.UTF-8`.
```
LANG=<your_locale_name>
```

Once you have done that do `ctrl + x`, `y` and then press enter to create & save the file. 

#### Configure the network identifiers
When you connect to a network your machine will be identified, using a name. This name is set in the `/etc/hostname` file. We can set it using nano.

Do:
```
nano /etc/hostname
```

In there enter your hostname name. In my case it will be `starkbamse`.

Once you have done that do `ctrl + x`, `y` and then press enter to create & save the file. 

#### Modify hosts files
Arch linux does not map the word localhost to the default loopback address **127.0.0.1**, we've got to do that ourselves. **::1** is the IPv6 version of the 127.0.0.1 loopback address. 

The **localdomain** is primarily about providing a default, non-routable domain context that works out of the box for most typical network software requirements and ensuring that internal name resolution is consistent and controlled without depending on external DNS services which may vary or not be available in every environment.

Do:
```
nano /etc/hosts
```

In there you should enter the following while replacing <your_hostname> with your hostname:
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 <your_hostname>.localdomain <your_hostname>
```

Once you have done that do `ctrl + x`, `y` and then press enter to create & save the file. 

#### Modify the mkinitcpio.conf/initramfs
`initramfs` (initial RAM filesystem) is a temporary root file system that is loaded into memory during the system boot process. Its primary function is to mount the real root file system and initialize the necessary drivers and configurations required to boot the system. In simpler terms, initramfs is what boots your kernel.

When the Linux kernel boots, it first loads this in-memory filesystem, which contains all necessary executables and drivers to mount the actual filesystem on which the system resides. This is essential for supporting systems where the root partition is on a file system or a device that requires special handling (like LVM, RAID, or encryption).

Use nano to modify the mkinitcpio.conf:
```
nano /etc/mkinitcpio.conf
```

In the file find the line that starts with `HOOKS` and change it so that it matches the following order of hooks:
```
HOOKS=(base udev autodetect keyboard modconf block encrypt lvm2 filesystems fsck)
```

Once you have done that do `ctrl + x`, `y` and then press enter to create & save the file. 

1. **base**: Contains the minimal set of directories and files to support a basic functioning Bash shell. This is necessary for any further system operations.

2. **udev**: Initializes userspace /dev using udev and waits for the completion of device and event handling. This permits dynamic creation and management of device nodes.

3. **autodetect**: Tries to minimize the initramfs by only including modules needed for booting the system on the actual hardware (detected during the build process). This speeds up boot time by avoiding the loading of unnecessary drivers.

4. **keyboard**: Includes keymap and console font hook modules to ensure the correct keyboard layout and font are used, which is especially essential for entering passwords in encrypted setups.

5. **modconf**: Allows for custom module options to be specified in `/etc/modprobe.d`. This can be used to configure hardware drivers as needed before the root filesystem is mounted.

6. **block**: Includes block device modules, necessary for accessing the hard drives, SSDs or other block devices on the system.

7. **encrypt**: Adds support for encrypted block devices, such as those encrypted with LUKS. Essential if the root file system resides on an encrypted volume.

8. **lvm2**: Adds support for Logical Volume Manager, allowing the system to organize data into logical volumes rather than physical partitions. This is used when your logical volumes are not located on the root filesystem by default.

9. **filesystems**: Includes the necessary filesystem drivers needed to mount the root filesystem. This must cover whatever filesystem type your root partition uses (e.g., ext4, btrfs).

10. **fsck**: Includes tools to check the integrity of filesystems before they are mounted. This can potentially prevent booting from a corrupted file system, reducing risk of data loss.


#### Recreate the initramfs image
Since we have modified the configuration of our initramfs image we need to recreate it.
```
mkinitcpio -p linux
```

#### Set root password
We need to set a root password, we do this through and follow the instructions prompted to us:
```
passwd
```

#### Install the GRUB bootloader
The GRUB bootloader is a well-known bootloader that allows us to boot into our arch linux system. Without it, our system will not boot.
```
pacman -S grub
```


#### Configure GRUB to allow booting from /boot on a LUKS1 encrypted partition
Use nano to edit the grub configuration file
```
nano /etc/default/grub
```

Find the line: `GRUB_ENABLE_CRYPTODISK=y`and remove the `#` preceding it, uncommenting it.

Once you have done that do `ctrl + x`, `y` and then press enter to create & save the file. 

#### Allow unlocking the encrypted LUKS partition 
GRUB needs to be able to unlock our encrypted volume. For this we need to specify the id of our LUKS partition.

Do:
```
blkid
```

In the output you need to find the device that is your LUKS partition. In my case it is: `/dev/sda3`. Once you have found it, take a picture/copy/write down the UUID next to your LUKS partition name.

Use nano to edit the grub configuration file:


```
nano /etc/default/grub
```

In there find the line that starts with `GRUB_CMDLINE_LINUX` and enter the following replacing x..x sequence with your UUID that you copied earlier:
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:cryptlvm root=/dev/vg/root"
```

Once you have done that do `ctrl + x`, `y` and then press enter to save the file. 

#### Install GRUB to the mounted ESP for UEFI booting
First we need to install the efi boot manager:
```
pacman -S efibootmgr
```

Then install the boot loader on the EFI partition
```
grub-install --target=x86_64-efi --efi-directory=/efi
```

#### Enable microcode updates
Microcrode updates are sort of like firmware updates for your computer's processor (CPU). Microcode is a layer of low-level code embedded in many modern CPUs; it helps control the processor's behavior and provides a mechanism to fix or mitigate known hardware bugs and security vulnerabilities post-manufacturing.

**Intel CPUs**
```
pacman -S intel-ucode
```

**AMD CPUs**
```
pacman -S amd-ucode
```

#### Embed another keyfile in initramfs
To prevent having to unlock the keyring twice (grub, initramfs) we can add another key to our LUKS container so that it will auto-decrypt once we've unlocked the container.

**Create a folder that will store our key and set its permissions**
```
mkdir /root/secrets && chmod 700 /root/secrets
```

**Generate a random key using /dev/urandom as entropy, and set permissions**
```
head -c 64 /dev/urandom > /root/secrets/crypto_keyfile.bin && chmod 600 /root/secrets/crypto_keyfile.bin
```

**Add the newly created key as a valid unlock key in our luks container while replacing <your_luks_partition> with your luks partition name. In my case it is `/dev/sda3`.

```
cryptsetup -v luksAddKey -i 1 <your_luks_partition> /root/secrets/crypto_keyfile.bin
```

#### Add the keyfile to the initramfs image
We need to include the newly generated key as a file attachment in our initramfs so that it can be used to decrypt the system. 

Use nano:
```
nano /etc/mkinitcpio.conf
```

Find the line that starts with `FILES` and set it to:
```
FILES=(/root/secrets/crypto_keyfile.bin)
```

Once you have done that do `ctrl + x`, `y` and then press enter to save the file. 

#### Recreate the initramfs image
Since we have modified the mkinitcpio.conf configuration file we need to recreate the initramfs image:
```
mkinitcpio -p linux
```

#### Instruct GRUB to use our newly created key to decrypt the system
Use nano:
```
nano /etc/default/grub
```

Find the line that start with `GRUB_CMDLINE_LINUX` and append the following to the end of the line separated by a **single space within the quotes**, but do **not** remove the previous contents.
```
cryptkey=rootfs:/root/secrets/crypto_keyfile.bin
```

Once you have done that do `ctrl + x`, `y` and then press enter to save the file. 

#### Regenerate GRUB's configuration file
Since we have modified the GRUB config we need to regenerate the actual configuration file.
```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Restrict ```/boot``` permissions
Set appropriate permissions for the boot folder.
```
chmod 700 /boot
```

#### Install sudo
Sudo allows us to perform admin commands protected by a password, making you think twice about your action.

```
pacman -S sudo
```

#### Add another user
Using the root user by default is a bad idea from a security perspective, adding another user will prevent you from accidentally breaking your system. And trust me, it does happen. Replace <username> with your desired username. This will be the username you login with every time. In my case it will be `starkbamse`.
```
useradd -m <username>
```

#### Set the password of your user
You need to set the password of your newly created user. Replace <username> with your username Do it by:
```
passwd <username>
```

#### Give your new user admin permissions
We use the command usermod to give the newly created user admin permissions. Replace <username> with your username.
```
usermod -aG wheel,audio,video,storage <username>
```

#### Allow members of wheel group to use admin commands
Since we still do not know how to exit vim/vi we need to force `visudo` to use nano instead.
```
EDITOR=nano visudo
```

In here you will need to remove the % sign on the line that starts with `wheel ALL=(ALL:ALL) ALL`.

This allows users of group wheel to perform admin command upon successful password entry.

Once you have done that do `ctrl + x`, `y` and then press enter to save the file. 

#### Install desktop environment
Unless you are a madman you will need a desktop environment. The first step is to install a display server. We will use Xorg for this.

```
pacman -S xorg networkmanager
```

Then you install the GNOME desktop environment
```
pacman -S gnome
```

Enable the display manager
```
systemctl enable gdm.service
```

Enable the network manager
```
systemctl enable NetworkManager.service
```

#### Use local mirror links
Arch linux uses default mirror links however you can improve download speed by switching to local mirror links. To do this you will need to install reflector:

```
pacman -S reflector
```

Use reflector to update the mirrorlist replacing <country> with your full country name:
```
reflector --country <country> --protocol http --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```

#### Reboot
Now you need to reboot your system and login to your user that you created.

Before rebooting you need to exit chroot
```
exit
```

Then reboot your system using
```
reboot
```

When you have rebooted you will be prompted to enter the passphrase to decrypt the LUKS partition. This will take some time, but be patient.

#### Completed
Congratulations! You are now an Arch Linux user! Below you will find an installation guide for installing Nvidia Graphics drivers.

#### Install graphic drivers (optional)
To install Nvidia graphics drivers in compatibility with this tutorial visit: https://github.com/korvahannu/arch-nvidia-drivers-installation-guide

#### Install CUDA

After installing the drivers, install the appropriate CUDA version. 

CUDA 12
```
sudo pacman -S cuda
```

CUDA 12 toolkit
```
sudo pacman -S cuda-tools
```

****

This installation guide was last tested April 21st 2024.

