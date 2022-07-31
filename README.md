# Sunny's Arch Linux installation guide

💡***Remember, you can click the icon to the left of the `README.md` title for a table of contents.***

This noob-friendly guide aims to assist users in setting up a fresh Arch Linux system equipped with a minimal installation of the KDE Plasma desktop environment.

I have personally had a lot of troubles getting Arch to connect to Wi-Fi during the install process. For maximum convenience, consider connecting your machine using Ethernet.

Throughout this guide, if you see square brackets '[ ]' in code blocks, that means the word of code (square brackets included) should be replaced with something else. This is detailed in the instructions before or after the code block.

This guide makes a few assumptions:

- The keyboard layout used is English US.

- The ISO used is the Arch Linux ISO from the [official site](https://archlinux.org/download/).

- The system boot mode is UEFI.

- The disk to install Arch on uses GPT.

## Setup

### Boot mode verification

To verify that the boot mode is indeed UEFI, the following command shows an output without error.

```
ls /sys/firmware/efi/efivars
```

If the directory does not exist, the system is not in UEFI mode. Refer to your motherboard's manual for troubleshooting.

### Internet connection

If you are connected via Ethernet, you should have Internet access already. If you must use Wi-Fi, you will have to use *iwd* (iNet wireless daemon). Get an interactive prompt for it using:

```
iwctl
```

List Wi-Fi devices.

```
device list
```

Take note of the name of the Wi-Fi device you are using (usually `wlan0`). For the next three commands, replace `[device_name]` with this name.

Scan for networks (this command does not output anything).

```
station [device_name] scan
```

List all available networks.

```
station [device_name] get-networks
```

Take note of the SSID of the network you want to connect to. Then connect to it using

```
station [device_name] connect [ssid]
```

where `[ssid]` is the SSID you took note of. You will be prompted to enter a password if required by the network.

Exit *iwd*.

```
exit
```

Verify your Internet connection.

```
ping archlinux.org
```

You should see lines continuosly being printed in the terminal. This indicates a working connection. 

Press Ctrl+C to stop pinging.

### Update system clock

*timedatectl* is the service that controls the system time and date. Update the system clock with:

```
timedatectl set-ntp true
```

The service's status can be checked with:

```
timedatectl status
```

## Partitioning

### Disk identification

List available disks and their partitions in your system.

```
lsblk
```

Take note of the name of the disk you want to install Arch on. Usually, it should be `sda`, `nvme0n1` or similar.

To modify our disks and partitions, we will use a command-line tool called *fdisk*. *fdisk* supports GPT and is used in the official Arch Linux installation guide. However, it is fully dialog-based and may not be intuitive for new users. You can refer to the [full list](https://wiki.archlinux.org/title/Partitioning#Partitioning_tools) of partitioning tools for alternatives. In this guide, we will be sticking to *fdisk*.

### Partitioning with *fdisk*

Launch *fdisk* on the target disk.

```
fdisk /dev/[target_disk_name]
```

`[target_disk_name]` should be the name you took note of earlier (`sda`, `nvme0n1` or similar).

From this point onward, all changes you make inside *fdisk* are saved inside memory only. You can still discard changes and exit. **Writing the changes to the disk is irreversible**, so be careful and double check your changes before writing.

*fdisk* is quite simple. You type a one-letter command, press Enter, and a dialog prompts you to enter further values to modify your disk. The `m` command shows a list of all available commands.

For UEFI systems, a bare minimum of 2 partitions are required for a working installation of Arch: a root partition and an EFI partition. For this installation, we will create these 2 partitions as well as a [swap partition](https://www.makeuseof.com/tag/swap-partition/).

To completely wipe the target disk and create these 3 partitions, follow the steps below:

1. Enter `g`. This **deletes all current partition data** and creates a new GPT.

2. Enter `n` (creates a new partition). 
   
   Partition number prompt: Enter `1` (denoting the first partition). 
   
   First sector prompt: Enter without typing anything (this takes the default value for first sector).
   
   Last sector prompt: Enter `+512M` (so our EFI partition is 512 MiB in size).

3. Enter `n`.
   
   Enter `2`.
   
   Enter without typing anything.
   
   Enter `+[swap_size]`, where `[swap_size]` is the desired size of our swap partition. For example, `8G` indicates 8 GiB and `512M` indicates 512 MiB. Swap size is a personal choice. [This article](https://help.ubuntu.com/community/SwapFaq#How_much_swap_do_I_need.3F) provides a guideline on how much swap you should need.

4. Enter `n`.
   
   Enter `3`.
   
   Enter without typing anything.
   
   Enter without typing anything (this allocates the remainder of the entire disk to the root partition).

5. Enter `t` (changes the type of a specified partition).
   
   Enter `1` (selects the first partition).
   
   Enter `1` (sets the partition type to *EFI System*; you can view all partition types by entering `L` and return to the prompt by pressing `q`).

6. Enter `t`.
   
   Enter `2`.
   
   Enter `19` (sets the partition type to *Linux swap*).

7. Enter `p` to view and check the current partition table. You should have a 512 MiB EFI partition, a swap partition of your specified size, and a root partition that takes up the remaining disk space. You can add new partitions with `n` and delete partitions with `d` if you wish.

8. Enter `w` to write changes and exit *fdisk*. This action is **irreversible**.

### Formatting

List available disks and their partitions in your system again.

```
lsblk
```

Identify the disk you just partitioned and take note of the names of the partitions. They should be called `sda1`, `sda2` and `sda3`, or `nvme0n1p1`, `nvme0n1p2` and `nvme0n1p3`, or similar.

Format the EFI partition.

```
mkfs.fat -F32 /dev/[EFI_partition]
```

`[EFI_partition]` should be the name of the first partition you took note of earlier (with size 512M).

Using the same logic, format the swap partition

```
mkswap /dev/[swap_partition]
```

and the root partition.

```
mkfs.ext4 /dev/[root_partition]
```

### Mounting

Mount the root volume to `/mnt`.

```
mount /dev/[root_partition] /mnt
```

Enable the swap volume.

```
swapon /dev/[swap_partition]
```

Create a directory called `boot` inside `/mnt` and a directory called `efi` inside `boot`.

```
mkdir /mnt/boot && mkdir /mnt/boot/efi
```

Mount the EFI partition to the `efi` directory.

```
mount /dev/[EFI_partition] /mnt/boot/efi
```

## Base and kernel installation

Before we start installing packages, it might be helpful to update the mirror list so the packages are retrieved from sources closer to our geographical location. This will increase download speeds. 

You can view the current mirror list with

```
cat /etc/pacman.d/mirrorlist
```

Make a backup of this default mirror list in the same directory.

```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Update the mirror list using the *reflector* module.

```
reflector -c [country1] -c [country2] -a 12 -p https --sort rate --save /etc/pacman.d/mirrorlist
```

`[country1]` and `[country2]` should be names of countries where you are currently or which are close to you. Available country names can be viewed [here](https://archlinux.org/mirrorlist/).

Install the base system.

```
pacstrap /mnt base base-devel linux linux-firmware linux-headers sof-firmware dosfstools mtools [ucode] [text_editor]
```

`[ucode]` should be replaced with `intel-ucode` if you have an Intel CPU, or `amd-ucode` if you have an AMD CPU.

`[text_editor]` should be replaced with the name of a command-line text editor of your choice. For new users, `nano` is recommended. In the rest of this guide, any step that requires editing a text file is done with `nano`. 

Generate the file systems table (fstab) file.

```
genfstab -U /mnt >> /mnt/etc/fstab
```

Change root into the new system.

```
arch-chroot /mnt
```

## Localisation

### Setting time zone

To view the list of available time zones, run

```
ls /usr/share/zoneinfo
```

Some of these zone names are directories that contain more zone names. We can view their contents with

```
ls /usr/share/zoneinfo/[region]
```

where `[region]` is the name of a directory inside `/usr/share/zoneinfo`.

Take note of the name of the time zone you are in. Then run

```
ln -sf /usr/share/zoneinfo/Asia/[time_zone] /etc/localtime
```

replacing `[time_zone]` with the name.

Generate `/etc/adjtime`.

```
hwclock --systohc
```

### Setting locale

Edit `locale.gen`.

```
nano /etc/locale.gen
```

If this is your first time using `nano`, it is a very simple command-line text editor. Use the arrow keys to scroll through the file.

Search for your desired locale and uncomment (delete the `#` at the start of the line) it. Take note of the name of the locale. For example, if the locale we wish to use is `en_US.UTF-8`, we uncomment the line `#en_US.UTF-8 UTF-8`. We have to take note the name of the locale, which in this case is `en_US.UTF-8`.

In `nano`, press Ctrl+X to exit the file, press Y to save changes, and press enter to retain the file name.

Generate the uncommented locale.

```
locale-gen
```

Create `locale.conf` and add the `LANG` variable.

```
echo LANG=[locale_name] > /etc/locale.conf
```

`[locale_name]` should be replaced with the name of the locale we took note of earlier (for the previous example, the full command is `echo LANG=en_US.UTF-8 > /etc/locale.conf`).

## Network configuration

Create the `hostname` file and set the hostname.

```
echo [hostname] > /etc/hostname
```

`[hostname]` should be your desired hostname (ie. the name of your device).

Edit the `hosts` file (in this case using `nano`).

```
nano /etc/hosts
```

This file already has two lines of comments. Go to the third line and type the following:

```
127.0.0.1        localhost
::1              localhost
127.0.1.1        [hostname].localdomain [hostname]
```

Again, replace both instances of `[hostname]` with the hostname you chose earlier. The `hosts` file should now have 5 lines in total. Save and exit the file.

Install `networkmanager`, a program and service that allows us to manage network connections.

```
pacman -S networkmanager
```

Enable the service.

```
systemctl enable NetworkManager
```

## Permission settings

Set the root (administrator) password for your system.

```
passwd
```

Follow the prompt to set the password.

Create a user (for yourself).

```
useradd -m [username]
```

`[username]` should be the desired name for the user. Replace it with the same name for the next two commands.

Set a password for this user.

```
passwd [username]
```

Add this user to essential groups.

```
usermod -aG wheel,audio,video,optical,storage [username]
```

Edit `/etc/sudoers.tmp`.

```
EDITOR=nano visudo
```

Find the following two lines.

```
## Uncomment to allow members of group wheel to execute any command
# %wheel ALL=(ALL:ALL) ALL
```

Uncomment the second line so it becomes like this:

```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
```

Save and exit the file.

## Bootloader setup

For this installation, we will install a very popular bootloader called GRUB. A list of alternatives can be found [here](https://wiki.archlinux.org/title/Arch_boot_process#Boot_loader).

Install GRUB and `efibootmgr`, a UEFI management tool.

```
pacman -S grub efibootmgr
```

Install GRUB to the drive.

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
```

Generate GRUB config file.

```
grub-mkconfig -o /boot/grub/grub.cfg
```

At this point, we already have a working Arch Linux system, albeit without a GUI. It is good practice to reboot the machine first before continuing to see if everything works well.

Exit `chroot`.

```
exit
```

Unmount `/mnt`.

```
umount -R /mnt
```

Reboot.

```
reboot
```

If everything works as intended, you should be able to log in as the user you created before by entering prompts for a username and password.

## GUI installation

### Display driver installation

From this point onward, the root password has to be entered every time a `sudo` command is run.

Check for package updates.

```
sudo pacman -Syu
```

If you do not know what GPU your machine has, identify it using

```
lspci -k | grep -A 2 -E "(VGA|3D)"
```

Look for keywords like ***Nvidia***, ***AMD*** and ***Intel***. If you have an Nvidia or AMD GPU, make sure you know the name of the GPU's microarchitecture (Ampere, RDNA2, etc.).

Install the display driver.

```
sudo pacman -S xorg [display_driver]
```

Replace `[display_driver]` with

- `nvidia` if you have an Nvidia GPU of the Maxwell series or newer.
  
  - If you have an Nvidia GPU of the Turing series or newer, you can choose to install `nvidia-open` instead, which are open-source Nvidia drivers.

- `xf86-video-nouveau` if you have an Nvidia GPU of the Tesla series or older.

- `xf86-video-amdgpu` if you have an AMD GPU with GCN 3 architecture or newer.

- `xf86-video-ati` if you have an AMD GPU with GCN 2 architecture or older.

- `xf86-video-intel` if you have an Intel GPU.

If you have an Nvidia GPU of the Kepler or Fermi series, leave `[display_driver]` blank for now and just install `xorg`.

Press Enter without typing anything when prompted to select members of the `xorg` group.

### KDE Plasma installation

There are multiple ways to install KDE Plasma in Arch Linux minimally.

The bare minimum is to install `plasma-desktop`. This is basically just the Plasma desktop without commonly used plugins and applets. If you know exactly what additional packages you want to install for your Plasma desktop, this is a good place to start. This method, however, is not recommended for beginners. You might end up with an overly minimal desktop that is unintuitive and missing basic features.

The other 2 ways are to install `plasma` or `plasma-meta`. Both include `plasma-desktop` as well as other essential packages for a minimal yet usable out-of-the-box Plasma desktop.

`plasma` is a package group. All packages within this group are explicitly installed. You can uninstall any package individually later on if you want. `plasma-meta` is a meta package. All packages within it are treated as a single package. You cannot uninstall individual packages without installing all other packages that came with the meta package.

Whether you want to install `plasma` or `plasma-meta` is up to you. `plasma` offers more flexibility for users who may want to trim down their installation later on. `plasma-meta` offers stability for users who are confident that they require all packages in the meta package and wish to treat them all as a single unit, preventing an accidental uninstallation of a core Plasma package. Note that [`plasma-meta`](https://archlinux.org/packages/extra/any/plasma-meta/) has fewer packages than [`plasma`](https://archlinux.org/groups/x86_64/plasma/).

In this guide, we will install `plasma-meta` (which you can replace with `plasma` if you wish), as well as a number of other commonly used KDE applications excluded from the minimal installation.

```
sudo pacman -S plasma-meta ark dolphin firefox git gwenview kcalc kdeconnect kinit konsole krunner kvantum kwrite okular packagekit-qt5 partitionmanager print-manager spectacle vlc xsettingsd
```

This is a table explaining what each package is. These are just the packages that I usually install personally and should be used as a guideline. Feel free to omit packages and add others as you wish.

| Package          | Description                                                        |
| ---------------- | ------------------------------------------------------------------ |
| ark              | Archive manager                                                    |
| dolphin          | File manager                                                       |
| firefox          | Browser                                                            |
| git              | Version control system                                             |
| gwenview         | Image viewer                                                       |
| kcalc            | Scientific calculator                                              |
| kdeconnect       | Connects KDE to your smartphone to do various inter-device actions |
| kinit            | KDE process launcher                                               |
| konsole          | Terminal                                                           |
| krunner          | KDE desktop launcher                                               |
| kvantum          | Theme engine                                                       |
| kwrite           | GUI text editor                                                    |
| okular           | Document viewer                                                    |
| packagekit-qt5   | Ensures Discover works on Arch                                     |
| partitionmanager | Disk utility                                                       |
| print-manager    | Printer and print job management tool                              |
| spectacle        | Screenshot utility                                                 |
| vlc              | Media player                                                       |
| xsettingsd       | Provides settings to X11 applications                              |

Some applications above require the JACK API and you will be prompted to install one of `jack2` and `pipewire-jack`. The differences between the two are illustrated [here](https://wiki.archlinux.org/title/JACK_Audio_Connection_Kit#Comparison_of_JACK_implementations). For me personally, I go with `pipewire-jack`.

Some applications above require you to install a `ttf-font` package. You can choose any. For me personally, I go with `noto-fonts`, a popular font family that is open-source.

Some applications above require a backend for `phonon-qt5`, KDE's multimedia framework. You will be prompted to choose one of `phonon-qt5-gstreamer` and `phonon-qt5-vlc`. The dfferences between the two are illustrated [here](https://community.kde.org/Phonon/FeatureMatrix). For me personally, I go with `phonon-qt5-vlc`. Note that `phonon-qt5-vlc` automatically installs the `vlc` package, which is the VLC media player application. If you do not intend to use VLC media player, you might prefer `phonon-qt5-gstreamer`.

Enable the SDDM display manager.

```
sudo systemctl enable sddm
```

### Legacy Nvidia drivers

If you have an Nvidia GPU of the Kepler or Fermi series, you will have to install legacy drivers from the Arch User Repository (AUR). Otherwise, skip this section. For beginners, the process is different from installing with the `pacman -S` command.

Change the working directory to your `Downloads` folder (it can be any folder, but this should be most convenient).

```
cd ~/Downloads
```

Acquire build files using Git.

```
git clone [url]
```

Replace `[url]` with `https://aur.archlinux.org/nvidia-470xx-utils.git` if you have a Kepler series GPU or `https://aur.archlinux.org/nvidia-390xx-utils.git` if you have a Fermi series GPU.

Change the working directory to the folder containing the build files.

```
cd [folder]
```

Replace `[folder]` with `nvidia-470xx-utils` or `nvidia-390xx-utils`, depending on the one you downloaded.

Then, package and install the driver.

```
makepkg -si
```

This is generally how an AUR package is installed. These 3 steps

1. `git clone [url]`

2. `cd [folder]`

3. `makepkg -si`

can be applied to install any package from the AUR. You can search for AUR packages [here](https://aur.archlinux.org/) and get their Git clone URLs.

## Audio and bluetooth installation

### Bluetooth setup

Install audio and Bluetooth utilities.

```
sudo pacman -S alsa-utils bluez bluez-utils
```

Enable Bluetooth.

```
sudo systemctl enable bluetooth.service
```

### Pipewire installation

By default, our system uses the [PulseAudio](https://wiki.archlinux.org/title/PulseAudio) sound server. [PipeWire](https://wiki.archlinux.org/title/PipeWire) is a newer alternative that has performed better on nearly every device I tried. I recommend installing it.

```
sudo pacman -S pipewire pipewire-alsa pipewire-jack pipewire-pulse wireplumber
```

Press Y and Enter when asked whether to replace `pulseaudio` with `pipewire-pulse`.

If you have the `jack2` package installed, it will conflict with `pipewire-jack`. You can safely replace `jack2` with `pipewire-jack`.

## Post-installation

This is where you can install any other packages or services you want before you boot into the GUI for the first time. Usually, I will further install the following packages:

```
sudo pacman -S acpid cronie cups neofetch ntp openssh pacman-contrib wget xf86-input-synaptics
```

| Package              | Description                                        |
| -------------------- | -------------------------------------------------- |
| acpid                | ACPI power management event daemon                 |
| cronie               | Program scheduling daemon                          |
| cups                 | Printing daemon                                    |
| neofetch             | Shows system information in the terminal           |
| ntp                  | Network Time Protocol implementation               |
| openssh              | SSH connectivity tool                              |
| pacman-contrib       | Contributed scripts for pacman                     |
| tlp                  | Power management tool (if you have a laptop)       |
| wget                 | Tool to retrieve files from the web                |
| xf86-input-synaptics | Driver for laptop touchpads (if you have a laptop) |

Enable any services you installed.

```
sudo systemctl enable acpid.service  
sudo systemctl enable cups.service
sudo systemctl enable ntpd.service
sudo systemctl enable sshd.service
sudo systemctl enable tlp.service
```

Clear pacman cache (this requires `pacman-contrib`).

```
sudo paccache -r
```

Reboot!

```
reboot
```

After you reboot, you should be greeted by a GUI login prompt. And that is it - A minimal KDE Plasma desktop on Arch Linux!

## What's next?

### Yay

As an Arch Linux user, you may find yourself installing packages from the AUR a lot. It may be more convenient to use an AUR helper, such as Yay. Yay has to be manually installed from the AUR first.

Change the working directory.

```
cd ~/Downloads
```

Acquire build files.

```
git clone https://aur.archlinux.org/yay.git
```

Change the working directory to the Yay folder.

```
cd yay
```

Package and install Yay.

```
makepkg -si
```

From this point onwards, you can search for packages from the AUR (and even the Arch Linux repositories) using `yay -Ss [package_name]`. Packages from the AUR can now be installed easily with `yay -S [package_name]`. Instead of running `sudo pacman -Syu`, you can now run `yay` to update all packages in your system, including those installed from the AUR.

### Multilib repository

In addition to the *core*, *community* and *extra* repositories, Arch Linux also has a repository called *multilib* that contains 32-bit software and libraries. Software commonly used for gaming on Linux (such as WINE and Steam) is usually found in this repository.

The *multilib* repository is disabled by default. To enable it, edit `/etc/pacman.conf`.

```
sudo nano /etc/pacman.conf
```

Find these 2 lines

```
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

and uncomment them.

### Input methods

To change input methods in KDE Plasma, I recommend using the Fcitx5 input method framework. Unfortunately, input methods are not done very well in Plasma and I have experienced a lot of troubles getting it to work. By far the best guide I have come across thus far is on [Jeffrey Tse's blog](https://jeffreytse.net/computer/2020/11/19/how-to-use-fcitx5-elegantly-on-arch-linux.html). The process to set Fcitx5 up is quite long so I will not repeat it here. By following the instructions in the article closely, you should be able to get Fcitx5 up and running.

### Hibernation

If you have set up a sufficiently large swap partition during the *Partitioning* step, you can now put it to good use and enable hibernation on your system.

List the UUIDs of your partitions.

```
sudo blkid | grep UUID=
```

Look for your swap partition. It should be the line that contains the string `TYPE="swap"`. In that same line, you should see strings for `UUID` and `PARTUUID`. The field we should pay attention to is `UUID`. Its value is a string of hexadecimal numbers. Copy the value of `UUID` (excluding the double quotation marks).

We need to tell the kernel where our swap partition is, and this is done via configuring the bootloader. This guide will only provide instructions for GRUB.

For GRUB, edit `/etc/default/grub`.

```
sudo nano /etc/default/grub
```

You should see the field `GRUB_CMDLINE_LINUX_DEFAULT`. Usually, for a fresh installation, the entire line looks like this:

```
GRUB_CMDLINE_LINUX_DEFAULT='quiet'
```

It is fine if you see anything other than `quiet`. We need to add the parameter `resume=UUID=[your_uuid]`, where `[your_uuid]` is the UUID value of your swap partition which you copied earlier. Append this parameter within the quotation marks of the field. There should be a space between parameters. 

The line should now look like this:

```
GRUB_CMDLINE_LINUX_DEFAULT='quiet resume=UUID=[your_uuid]'
```

Remember, you may have other parameters between `quiet` and `resume`. That is fine. Save and exit the file.

Regenerate the GRUB config.

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Edit `/etc/mkinitcpio.conf`.

```
sudo nano /etc/mkinitcpio.conf
```

Look for the `HOOKS` field. It should be underneath all the comments in the `# HOOKS` section, and look something like this:

```
HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
```

It is fine if yours looks a little different. We need to add the `resume` hook to this field. It can be anywhere within the parentheses, **as long as it is positioned after** `udev`. 

The line should now look like this:

```
HOOKS=(base udev resume autodetect modconf block filesystems keyboard fsck)
```

Save and exit the file.

Regenerate the initramfs.

```
mkinitcpio -p linux
```

Reboot the system. Your system should now be able to hibernate via the GUI button or the following terminal command.

```
systemctl hibernate
```
