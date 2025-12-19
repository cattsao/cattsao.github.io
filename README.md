# Project 1- Arch Linux Installation

<a href="archlinux.html">Project 1- Arch Linux Installation</a>

<a href="wireguard.html">Project 3- WireGuard Lab</a>

**Step 1: Acquire Image**
I acquired an installation image from https://release.archboot.com/aarch64/latest/iso/ since I have a Mac.

To verify the image signature, I downloaded the b2sum signature for the ISO and installed b2sum by typing the command `brew install b2sum` into the terminal. I then used the `b2sum path/to/image` command to verify the signature. 

The signature did not match.

Step 1 (update):
I realized the signature was mismatched because I loaded the image a month ago and was only able to verify the signature now. The signatures listed on the website are only from the most current ISO. 

I downloaded the most updated installation image and verified that one instead.

The signature matched!

**Step 2: Bootloading**
I went onto UTM, created a custom operating system, pointed the current boot device to CD/DVD Image, booted the acquired ISO image, and selected Legacy Hardware. For hardware, I gave it 2048 MiB of memory and 2 cores to support a desktop environment. For storage I gave it 4 GiB. I did not select any directory to be made accessible within the VM.

Once I opened the VM, the bootloader did not appear. The only words displayed were "Display output is not active." I realized that legacy hardware would probably not work for Apple Silicon, so I deleted the VM and made a new one using the above configuration but without selecting Legacy Hardware. This worked!

However, once I opened the virtual console, "Display output is not active" appeared again. To try fixing this, I went back to the bootloader, pressed `e` to edit the boot entry, and changed each console argument pointing to serial to point to the graphical console `tty0`. This did NOT work. It still displayed "Display output is not active." I then checked the System information of the VM and noticed it was using `QEMU 9.1 ARM Virtual Machine (alias of virt-9.1) (virt)` which meant it was running in emulation instead of virtualize mode. The VM needs to be run in virtualize mode in order for output to be displayed. To fix this, I had to delete the VM and create a new one with the correct settings. This time, I chose Linux instead of Custom for the Operating System. I selected Use Apple Virtualization, booted my ISO image, gave it 2048 MiB of memory, gave it 2 CPU cores, and gave it 10 GiB of storage. THIS WORKED- MY VIRTUAL CONSOLE CORRECTLY DISPLAYED!!!!

I finally got a chance to verify the boot mode by typing `cat /sys/firmware/efi/fw_platform_size` into the virtual console. It returned `64`, indicating the system has a 64-bit x64 UEFI.

**Step 3: Connecting to Internet**
To ensure my network interface is listed and enabled, I ran the `ip link` command. Then I ran the commands `systemctl restart systemd-networkd` and `systemctl status systemd-networkd` to request an IP address with DHCP. I verified the connection using the `ping ping.archlinux.org` command.

To ensure the system clock was synchronized, I used the command `timedatectl`. 

**Step 4: Partition Disks**
To identify the block devices, I ran the command `fdisk -l`. To create EFI system and root partitions for the disk, I used the command `fdisk /dev/vda`.
I typed `g` to create a new GPT disklabel, typed `n` to create each partition, and gave the EFI system partition `+512M` of space.

I then formatted each partition to their appropriate file systems. I formatted the EFI system partition using the command `mkfs.fat -F /dev/vda1` and the root partition using `mkfs.ext4 /dev/vda2`.

Finally, I mounted the file systems using the `mount /dev/vda2 /mnt` to mount the root partition to `/mnt` and `mount --mkdir /dev/vda1 /mnt/boot` to mount the EFI system partition to `/mnt/boot`.

**Step 5: Installing Essential Packages**
I checked my mirrorlist using `cat /etc/pacman.d/mirrorlist` and edited it to include all US servers using `nano /etc/pacman.d/mirrorlist`.

I tried to install essential packages using `pacstrap -K /mnt base linux linux-firmware xfsprogs btrfs-progs e2fsprogs dosfstools lvm2 dhcpcd networkmanager network-manager-applet nano man-db man-pages texinfo` but it failed... In the process of trying to figure out what went wrong the console started throwing a wall of kernel messages and I had to restart and remount my filesystems. For some reason, `/dev/vda1` was mounted to `/boot` and `/dev/vda2` was mounted to `/`. I used `umount /boot` to unmount `/dev/vda1` from `/boot` and tried using `umount /` to unmount `/dev/vda2` from `/` but that didn't work so I had to restart the VM to restore the correct root filesystem. I remounted `/dev/vda2` and `/dev/vda1` using the commands from Step 4. Then I updated the database with `pacman -Sy`. I tried installing the base system with `pacstrap -K /mnt base linux linux-firmware` but it failed. To check if `/mnt` contains files from the initial failed install, I ran `ls -R /mnt | head`. Since it did end up containing files from the failed install, I ran `umount -R /mnt`
`mkfs.ext4 /dev/vda2`
`mount /dev/vda2 /mnt`
`mkdir /mnt/boot`
`mount /dev/vda1 /mnt/boot`
Then I reran `pacstrap -K /mnt base linux linux-firmware`. It still failed.
I checked the system clock again to make sure it was not the issue, and it was fine. At this point, the issue was either with the pacman keyring being broken or the live ISO session being unstable. To reinitialize the pacman keyring, I ran `pacman-key --init`
`pacman-key --populate archlinux`
`pacman-key --refresh-keys`
The refresh failed, so I ran `pacman -Sy archlinux-keyring` and tried `pacman-key --populate archlinux again`. I then checked if my mirrorlist was usable with `pacman -Syy`. It was usable. Now it was time to check pacstrap again. This time, I used the smallest possible command to do so- `pacstrap -K /mnt base`. This worked, meaning base was not the issue. Then I ran `pacstrap -K /mnt linux linux-firmware`. This failed.
Since the `linux` package was failing, this meant `/mnt/dev`, `/mnt/proc`, or `/mnt/sys` could be missing. To check if they were, I ran 
`mount --bind /dev /mnt/dev`
`mount --bind /proc /mnt/proc`
`mount --bind /sys /mnt/sys`
These worked. So I ran `pacstrap -K /mnt linux linux-firmware` again. It still failed. Another cause for `linux` to fail is if `pacman-key`and `dkms`are stuck waiting for entropy. So I force loaded the RNG using `modprobe virtio_rng 2>/dev/null` and tried running `pacstrap -K /mnt linux linux-firmware` again. It failed again. Finally, I tried running `touch /mnt/etc/vconsole.conf` to try fixing `mkinitcpio` since archboot sometimes breaks it for fun. Then I retried `pacstrap -K /mnt linux linux-firmware`. It failed yet again. Perhaps the issue was that `/mnt/boot` already contains kernel files from the previous failed archboot installation. So now I decided to unmount boot using `umount /mnt/boot`, reformat the EFI partition using `mkfs.fat -F32 /dev/vda1`, remount `/mnt/boot` using `mount /dev/vda1 /mnt/boot`, and verify `/mnt/boot` was empty using `ls /mnt/boot`. Once again I tried running `pacstrap -K /mnt linux linux-firmware`. IT WORKKKKKKKEEEEDDD!!!!!! FINALLLLYYYYYY

At this point, I was ready to move on from installing essential packages. I decided that it was okay to not install anything other than `base` and `linux`, and that I could always install other packages if needed later.

**Step 6: Configure the System**
To get necessary file systems mounted on startup, I generated an `fstab` file using `genfstab -U /mnt >> /mnt/etc/fstab`.

Then I changed root into the new system using `arch-chroot /mnt`.

I set the time zone using `ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime`. I ran `hwclock --systohc` to generate `/etc/adjtime`.

To correct region and language specific formatting, I tried to run `nano locale-gen`. This did not work since `nano` is not installed. I tried installing `nano` using `pacman -S nano` but this didn't work because `pacman` couldn't verify the package signature. To work around this, I cleared the package cache using `pacman -Scc`. Then I force refreshed all package databases `pacman -Syy`. Next, I tried to update the `archlinux-keyring` using `pacman -S archlinux-keyring --noconfirm`. That failed, so I tried `pacman-key --init` and `pacman-key -- populate`. This worked. Now I tried installing nano again using `pacman -S nano`. It still resulted in PGP signature errors, so I tried `pacman -S nano --overwrite '*'`. It still failed. My keyring likely broke. To fix this, I deleted all existing keys with `rm -rf /etc/pacman.d/gnupg`. Then I recreated the keyring with `pacman-key --init` and populated the Arch Linux keys with `pacman-key --populate archlinux`. Then I force refreshed package databases with `pacman -Syy`. I tried to cleanly reinstall the keyring package with `pacman -S archlinux-keyring --noconfirm`. This failed. The keyring was still broken. To fix it, I tried force-refreshing the package databases using `pacman -Syy --noconfirm`. Then I removed the broken keyring directory using `rm -rf /etc/pacman.d/gnupg`. Then I reinitialized the keyring with `pacman-key --init` and `pacman-key --populate archlinux`. I tried installing the keyring without signature checking using `pacman -S archlinux-keyring --noconfirm --disable-download-timeout --noscriptlet --overwrite '*'` This too failed. Finally, I tried installing the keyring using `pacman -S archlinux-keyring --noconfirm --config <(echo -e '[options]\nSigLevel = Never')`. It continued to fail. I realized this command accidentally removed the repository definitions. I force refreshed the pacman databases using `pacman -Syy --noconfirm`. I deleted the broken keyring using `rm -rf /etc/pacman.d/gnupg`. I reinitialized the keyring using `pacman-key --init` and `pacman-key --populate archlinux`. I installed the keyring with signature checking disabled only for the package file using `pacman -S archlinux-keyring --noconfirm --overwrite '*' --disable-download-timeout --assume-installed archlinux-keyring`.  It failed so I tried `pacman -U /var/cache/pacman/pkg/archlinux-keyring-*.pkg.tar.* --noconfirm --overwrite '*'`. This didn't work. Once again, the cycle restarts. Force refresh, delete broken keyring, reinitialize keyring, populate Linux keys. This time, I tried installing the keyring package without any special flags, using `pacman -S archlinux-keyring`. Unsurprisingly, it failed. Perhaps the new keys weren't actually imported. Again, I destroyed the broken keyring with `rm -rf /etc/pacman.d/gnupg`. Then I recreated the keyring directory using `install -dm755 /etc/pacman.d/gnupg`. Next I initialized the keyring and populated the keys. Finally I attempted to install the keyring package again, using `pacman -Sy --noconfirm --config <(printf "[options]\nSigLevel = Never") archlinux-keyring`. This command also lacked repository definitions like earlier, resulting in failure. So again I deleted the broken keyring and reinitialized the keyring. Then I attempted to force `pacman` to treat `archlinux-keyring` as installed using `pacman -Sy --noconfirm --overwrite="*" --assume-installed archlinux-keyring archlinux-keyring`. That failed so I tried `pacman -Sy --noconfirm --overwrite="*" archlinux-keyring --assume-installed archlinux-keyring --disable-download-timeout`. It failed. Maybe my mirror has an issue. I switched up my approach and backed up the old mirrorlist using `cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup`. I replaced the mirrorlist with the official master repo using `printf "Server = https://geo.mirror.pkgbuild.com/\$repo/os/\$arch\n" > /etc/pacman.d/mirrorlist`. I force refreshed databases from this good mirror using `pacman -Syy --noconfirm`. This command failed. Perhaps its the mirrorlist that's failing. So I replaced my mirrorlist using `cat > /etc/pacman.d/mirrorlist << 'EOF'` 
`Server = https://mirror.archlinuxarm.org/$arch/$repo`
`EOF`. Then I refreshed using `pacman -Syy --noconfirm`. It didn't work. I then ran `rm -f /var/lib/pacman/sync/*.db*` to wipe corrupted database files. I pulled fresh databases from the new mirror with `pacman -Syy`. It failed, so I tried rewriting the mirrorlist using http instead of https, using `cat > /etc/pacman.d/mirrorlist << 'EOF'`
`Server = http://mirror.archlinuxarm.org/$arch/$repo`
`EOF`. Then I refreshed the databases with `rm -f /var/lib/pacman/sync/*.db*` and `pacman -Syy`. Next, I typed `pacman -S archlinux-keyring --noconfirm`. It failed. So I completely disabled signature checking inside chroot using `sed -i 's/^SigLevel.*/SigLevel = Never/g' /etc/pacman.conf` and confirmed it worked using `grep SigLevel /etc/pacman.conf`. Then I force refreshed the package databases with `pacman -Syy`. Next, I installed the keyring without PGP checks using `pacman -S archlinux-keyring --noconfirm`. The keyring installation succeeded! Then I restored the correct pacman setting with `sed -i 's/^SigLevel.*/SigLevel = Required DatabaseOptional/g' /etc/pacman.conf` and verified it with `grep SigLevel /etc/pacman.conf`. Finally, I fully synced and upgraded to confirm everything works using `pacman -Syu`. That failed. So I cleared the entire pacman cache with `rm -rf /var/cache/pacman/pkg/*`. I force refreshed the databases and upgraded again using `pacman -Syyu --noconfirm`. It still failed. So I started to redo the keyring with `pacman-key --init`, `pacman-key --populate armlinuxarm` , and `pacman-key --populate archlinux` but then the second command failed. This means that chroot probably doesn't actually contain the keyring files. So I downloaded the keyring package manually with `curl -LO https://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. This failed so I tried again using `curl -kLO https://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz` to allow insecure SSL. I confirmed the download with `ls -l archlinuxarm-keyring-*.pkg.tar.xz`. I manually installed the keyring with `pacman -U archlinuxarm-keyring-*.pkg.tar.xz --noconfirm --overwrite '*' --config <(printf "[options]\nSigLevel = Never")`. It failed so I went to check whether the file actually exists and is valid. I ran `ls -lh`. Then I ran `file archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. It failed so I redownloaded the keyring package over http using `curl -LO http://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. Then I verified its size using `ls -lh archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. The size was too small so I downloaded it from a different mirror using `curl -LO http://de.mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. I checked the size again with `ls -lh archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. It was still tiny. So I tried downloading the official ARM mirror with `wget http://os.archlinuxarm.org/arm/extra/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. My system does not have `wget` so I tried `curl -LO http://os.archlinuxarm.org/arm/extra/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. I verified again and the file is still tiny so I tried `curl -LO https://mirror.archlinuxarm.org/arm/extra/archlinuxarm-keyring-20240124-1-any.pkg.tar.xz`. The file was still tiny. At this point I was questioning my life choices. What if this headache was all for nothing? Pehaps I could simply use `vim` instead of `nano` to edit `/etc/locale.gen`.

I typed `vim /etc/locale-gen` into the console. The file opened. I uncommented out the line saying `en_US.UTF-8 UTF-8`. I generated the locale with `locale-gen`. I set the system locale by opening the locale configuration file using `vim /etc/locale.conf` and setting the LANG variable to `en_US.UTF-8` with `LANG=en_US.UTF-8`.

At this point, I also realized I was in the live environment instead of the installed system. So I remounted my system with 
`mount /dev/vda2 /mnt`
`mount /dev/vda1 /mnt/boot`
Then I entered the installed system with `arch-chroot /mnt`. Next, I changed my timezone (since I have changed my timezone ever since starting this project) using `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime` and `timedatectl set-ntp true`, and ran `hwclock --systohc`. I tried re-editing `locale.gen` to uncomment `en_US.UTF-8 UTF-8` but `vim` was not installed. In this moment everything clicked. 

I was supposed to install the other essential packages in the previous step.

*facepalm*

argghhhhhhhhhhhh

all that time...wastedddddddd

Anyway,

**Step 5 part 2**
I exited the chroot and installed `vim` using `pacstrap -K /mnt vim`. I also installed `man-db`, `man-pages`, and `texinfo` using `pacstrap -K /mnt man-db man-pages texinfo`.

**Step 6 continued**
I re-entered chroot with `arch-chroot /mnt`. I typed `vim /etc/locale.gen` and uncommented `en_US.UTF-8 UTF-8`. Then I generated the locale by running `locale-gen`. Then I created `locale.conf` and set LANG to `en_US.UTF-8` using `echo "LANG=en_US.UTF-8" > /etc/locale.conf`.

Then I created the hostname file using `echo "Sphinx" > /etc/hostname`.

To complete network setup, I identified my network device using `ip link`. Then I started and enabled systemd-networkd with the commands
`systemctl enable systemd-networkd`
`systemctl enable systemd-resolved`
`systemctl start systemd-networkd`
`systemctl start systemd-resolved`
Next, I configured DHCP with `mkdir -p /etc/systemd/network` and edited `/etc/systemd/network/20-wired.network` to include my interface name `eth0`. I added 
`[Match]`
`Name=eth0`
`[Network]`
`DHCP=yes`
to the configuration file.
I then attempted enabled the `systemd`resolver symlink with `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`. At first it failed since the file `/etc/resol.conf` probably already exists. So I ran `rm -f /etc/resolv.conf` to remove the preexisting file and then created the symlink. This didn't work, indicating that the file could be bind-mounted. So I unmounted the file with `umount /etc/resolv.conf` and then redid the symlink. Next, I tested the network using the commands
`systemctl status systemd-networkd`
`ip a`
`ping archlinux.org`

Next, I set a password for the root user with `passwd`.

I then installed a bootloader by first running `bootctl install`. Since the output stated that the random seed file and its mount point was world accessible, I edited `/etc/fstab` hide world access by changing `fmask` and `dmask` from `0022` to `0077`. Then I tried to remount `/boot` using `mount -o remount /boot` but that didn't work. It could be an issue with the EFI partition. I ran `lsblk -f`to find the correct UUID listed for my `vfat /boot` partition. Then I opened `/etc/fstab` with `vim` to check whether the UUID under `/boot` matched. It did not match so I updated it. Then I mounted `/boot` with `mount /boot` and ran `mount -o remount /boot`. I adjusted permissions with `chmod 700 /boot` and `chmod 600 /boot/loader/random-seed*`. Then I reran `bootctl install`. I opened `/boot/loader/entries/arch.conf` and added the main entry
`title   Arch Linux
`linux   /Image`
`initrd  /initramfs-linux.img`
`devicetree /dtbs`
`options root=/dev/vda2 rw`
I checked my loader configuration with `cat /boot/loader/loader.conf` and edited it to include 
`default arch.conf`
`timeout 3`
`editor no`

Then I left chroot with `exit`, ran `umount -R /mnt`, and ran `reboot`.

**Step 7: Installing a DE**
I logged in and tried updating the package database  with `pacman -Syu` to no avail. I checked that I had an ip address with `ip addr`. Then I checked if DNS works with `ping -c 3 8.8.8.8`. This failed, meaning my installed system doesn't have a working network interface. So I checked what network devices I had with `ip link` and brought the Ethernet interface I had up with `ip link set enp0s1 up`. I checked if systemd-networkd was running with `systemctl status systemd-networkd` and enabled it for future boots with `systemctl enable systemd-networkd`. I created a DHCP configuration for `enp0s1` by making the file `/etc/systemd/network/20-wired.network` and putting `[Match]
`Name=enp0s1`
`[Network]`
`DHCP=yes`
in it. I reloaded `networkd` with `systemctl restart systemd-networkd`. Then I started the DNS resolver by inputting `systemctl enable systemd-resolved` and `systemctl start systemd-resolved`. I linked `resolv.conf` with `ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`. Now that my network interface has been set up, I attempted to update my system again with `pacman -Syu`. It still didn't work. So I checked if the system clock was correct with `timeadatectl set-ntp true` and `timedatectl status`. The system clock was not synchronized. To force synchronize I ran `timedatectl set-ntp false` and `timedatectl set-ntp true`. Once again, it was time to try `pacman -Syu`. It failed, so I tried `pacman -Syyu`. It was still throwing errors so I attempted to refresh the keyring with `pacman -S archlinux-keyring`. It failed meaning my keyring is broken or empty so I reinitialized the keyring with `rm -r /etc/pacman.d/gnupg`
`pacman-key --init`
`pacman-key --populate archlinux`
I tried to refresh the keys with `pacman-key --keyserver keyserver.ubuntu.com --refresh-keys`. It failed, indicating that the keyserver connections are failing. So I tried the secure keyserver with the command `pacman-key --keyserver hkps://keyserver.ubuntu.com:443 --refresh-keys`. Once it finished, I ran `pacman -Syyu` to no avail. So I ran `rm -rf /etc/pacman.d/gnupg`, reinitialized the keyring with `pacman-key --init`, populated the keyring with `pacman-key --populate archlinux`, and updated the keyring package with `pacman -Syy archlinux-keyring`. It did not work. 

Perhaps the issue was that I was using commands for Arch Linux rather than Arch Linux ARM. So I ran `rm -r /etc/pacman.d/gnupg` and recreated the keyring with `pacman-key --init` and `pacman-key --populate archlinuxarm` this time. FAIL

Maybe I'm in some generic ARM Arch bootstrap? I ran `rm -rf /etc/pacman.d/gnupg`, `pacman-key --init`, and `pacman-key --populate` this time. Then I ran `pacman -Sy archlinux-keyring --noconfirm`. FAIL

I ran `curl -O http://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240130-1-any.pkg.tar.xz`. Then `pacman --config /dev/null -U archlinuxarm-keyring-*.pkg.tar.xz`. FAIL


so apparently I still need to REDO THE ARCH INSTALL because ~~I'm~~ it is ~~a~~ ~~failure~~ broken

what the actual

**Step 2: repair mode**
So now I have to create a new VM I guess???

I created a new VM, selecting Virtualize and Linux. I did not select Use Apple Virtualization or Boot from kernel image. I enabled Hardware OpenGL acceleration. I gave it the same amount of memory, cores and storage as the previous one. Then I went to its settings and imported the disk from the previous VM.

I mounted the root and boot partitions of the old system with `mount /dev/vdb2 /mnt` and `mount /dev/vdb1 /mnt/boot`. I chrooted in with `arch-chroot /mnt`. Things failed and kept failing

Ok so now I'm thinking the issue is that I've been using commands for x86 when the system is ARM.

**Step 2/3/4/6 for real: NEW VM TIME**
I created a new VM again, selecting Emulate and Linux. I used the same ISO, selected ARM64 (aarch64) for Architecture, and selected QEMU 9.1 ARM Virtual Machine (alias of virt-9.1) (virt). I gave 2048 MiB of memory, 2 CPU cores, 10 GiB of storage, and enabled hardware OpenGL acceleration. I imported the disk from the oldest VM, using VirtIO as the interface and turning off removability.

I mounted the old root and boot. Then I bound system mounts using
`mount --bind /dev /mnt/dev`
`mount --bind /proc /mnt/proc
`mount --bind /sys /mnt/sys
`mount --bind /run /mnt/run`
Then I chrooted.

Inside chroot I attempted to ensure the clock was synced using `timedatectl set-ntp true`. It failed so I deleted the x86 keyring using `rm -rf /etc/pacman.d/gnupg` and `pacman-key --init`. I tried to download the correct ARM keyring using `curl -O https://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240130-1-any.pkg.tar.xz`. Maybe the date is wrong. I tried using `date -u "$(curl -sI https://google.com | grep -i '^date:' | cut -d' ' -f2-)"` to fix the time.

It did not work. I checked 
`printf '%q\n' "$srv"`
`printf '%q\n' "$iso"`
`date --version` to try to understand why the date was not valid since it looked right. Since `$srv` was empty I ran 
`curl -sI http://example.com`
`curl -v http://example.com`
`ping -c1 142.250.72.206`
`ping -c1 google.com`
to figure what network connectivity was and was not working. None of them worked. So I shut down the VM and edited its settings. I set the CPU to cortex-a72 and I increased the memory to 4096 MiB. I changed my time zone to Central with `timedatectl set-timezone America/Chicago` and `ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime`. 

The ARM keyring is missing. So I used `curl --insecure -O https://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240130-1-any.pkg.tar.xz` to download the keyring without SSL verification. I then tried installing it using `pacman --config /dev/null -U archlinuxarm-keyring-20240130-1-any.pkg.tar.xz` but it failed. So I removed it, and tried different mirrors. First I used `curl --insecure -O https://mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240130-1-any.pkg.tar.xz`. Wanna guess what happened? It failed :D!! Next I tried `curl --insecure -O https://de.mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240130-1-any.pkg.tar.xz`. Fail again. Then I tried `curl --insecure -O https://us.mirror.archlinuxarm.org/aarch64/core/archlinuxarm-keyring-20240130-1-any.pkg.tar.xz` and the host could not be resolved. This means my issue is DNS. I overwrote `/etc/resolv.conf` with `echo "nameserver 8.8.8.8" | tee /etc/resolv.conf` and `echo "nameserver 1.1.1.1" | tee -a /etc/resolv.conf`. Then I installed CA certs with `curl --insecure -O https://pkg.cloudflare.com/pub/ca-certificates.tar.gz`. 


WHOOPSIES I MEANT TO CHROOT INTO THE ARCH SYSTEM AFTER FIXING THE CPU AND MEMORY

I swapped the old disk to be the first option in my Drives for the VM. Then I ran `pacman -Sy --noconfirm --needed --overwrite="*" --config <(echo -e "[options]\nSigLevel = Never") ca-certificates ca-certificates-mozilla
` to fix the CA certs`. Didn't work. So I removed the keyring with `rm -r /etc/pacman.d/gnupg`, recreated it with `pacman-key --init`, and repopulated it with `pacman-key --populate archlinux`. I force refreshed the package databases with `pacman -Syy`, and refreshed keys from the Ubuntu server using `pacman-key --refresh-keys --keyserver hkps://keyserver.ubuntu.com`. Then I tried installing the CA certificates with `pacman -S ca-certificates ca-certificates-utils ca-certificates-mozilla`. I guess system just hates all Arch Linux ARM PGP keys. Rude. So I reinitialized the `pacman` keyring with `rm -rf /etc/pacman.d/gnupg` and `pacman-key --init`. Then I manually imported the key using `pacman-key --add /usr/share/pacman/keyrings/archlinuxarm.gpg`. It didn't work. I booted  back into my ISO rescue environment.

I mounted the root filesystem using `mount /dev/vda2 /mnt` and  `mount /dev/vda1 /mnt/boot`. I downloaded the correct Arch Linux ARM root filesystem tarball with `curl -L --insecure -o ArchLinuxARM-aarch64-latest.tar.gz \` 
`http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz`. I made a temporary directory to unpack the keyring files. Then I extracted the keyring directory from the tarball using `tar -xzf ArchLinuxARM-aarch64-latest.tar.gz -C alaroot ./usr/share/pacman/keyrings`. I copied the keyring files into the installed system using `cp /usr/share/pacman/keyrings/* /mnt/usr/share/pacman/keyrings/
 Then I chrooted. Using `pacman-key --populate archlinuxarm` I repopulated the newly installed keyring. I verified the keyring is trusted using `pacman-key --list-keys | grep -i arch`. I updated with `pacman -Syy` and `pacman -S archlinuxarm-keyring`. Then I overwrote them with `pacman -S archlinuxarm-keyring --overwrite="*"`. I fully updated the system with `pacman -Syu`. SUCCESS! FINALLY!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

**Step 7: Installing a DE Successfully?!**
In the conflagration of my joy I powered off my VM, so I had to boot in to the live system, remount my partitions with `mount /dev/vda2 /mnt` and `mount /dev/vda1 /mnt/boot`. Then I bind mounted the system directories with 
`mount --bind /dev /mnt/dev
`mount --bind /proc /mnt/proc`
`mount --bind /sys /mnt/sys`
and `mount --bind /run /mnt/run` for networking inside chroot. I chrooted back in with `arch-chroot /mnt`.

I decided to install LXQt as my desktop environment. Since LXQt runs on top of X11, I installed the Xorg packages using `pacman -S xorg-server xorg-apps xorg-xinit`. I also ran `pacman -S xterm xorg-twm` to install the terminal emulator and tab window manager for LXQt. Then I installed LXQt alongside some useful components to come with it using 
`pacman -S lxqt openbox`
`pacman -S lxqt-session lxqt-panel lxqt-config lxqt-policykit`
`pacman -S lxqt-qtplugin lxqt-themes oxygen-icons`.
I installed LightDM as the display manager using `pacman -S lightdm lightdm-gtk-greeter` and `systemctl enable lightdm`. I installed NetworkManager with `pacman -S networkmanager network-manager-applet` and enabled it with `systemctl enable NetworkManager`.

I then exited chroot, tried unmounting all partitions with `umount -R /mnt`, and attempted restarting the machine with `reboot`. Both `umount` and `reboot` said `/mnt/sys: target is busy`. I typed `exit` again to make sure I was actually out of `chroot` but it caused to exit all the way out the the terminal session. So I force shut down the VM, placed my disk with the installed system as the first entry under Drives in my VM settings, and restarted the machine.

I logged into the root account and opened QTerminal.

**Step 8: Creating User Accounts**
I used `useradd -m -G wheel catherine` and `useradd -m -G wheel codi` to add myself and codi as sudo users.

**Step 9: Installing a Different Shell**
I installed `fish` with `pacman -S fish`.

**Step 10: Installing ssh**
I installed `ssh` with `pacman -S openssh`. I started the `ssh` daemon using `systemctl start sshd` and enabled `ssh` to start on boot with `systemctl enable sshd`.

**Step 11: Add Color Coding to Terminal**
To get similar colors to the live ISO, I installed `bash-completion` and `coreutils` with `pacman -S bash-completion coreutils`. I copied the Arch ISO default bash configuration using `cp /etc/skel/.bashrc ~/.bashrc` and reloaded it with `source ~/.bashrc`. No color appeared.
I ran `pacman -S qtermwidget qterminal lxqt-themes`. That didn't do anything so I tried to add
`[General]`
`background=#1e1e1e`
`foreground=#d0d0d0`
`palette=#000000,#cd0000,#00cd00,#cdcd00,#1e90ff,#cd00cd,#00cdcd,#e5e5e5,#4d4d4d,#ff0000,#00ff00,#ffff00,#4682b4,#ff00ff,#00ffff,#ffffff`
to `/usr/share/qtermwidget5/color-schemes/ArchLinux.colorscheme`. 
`vim` was not letting me write to the file, and since I'm in `root` it's probably because the filesystem is mounted read-only. I tried remounting the filesystem read-write, using `mount -o remount,rw /`. It failed since the UUID likely did not match. I ran `blkid /dev/vda2` to check the UUID. I ran `vim /etc/fstab` and replaced the UUID with the one from `blkid /dev/vda2`. Then I reran `mount -o remount,rw /` and then `systemctl daemon-reload`.

WHAT THE HECK I'M STUPID I DIDN'T NEED TO MESS WITH ANYTHING FROM THE ISO ANYMORE JUST TO CHANGE DUMB COLORS

No but actually the real way to color the terminal is to run `vim ~/.bashrc` and add `PS1='\[[\e[31m\]\u\[\e[0m\]@\[\e[32m\]\h\[\e[0m\] \w]\$ '`. I did that. Then I reloaded using `source ~/.bashrc`.

Also since I changed this disk to be the first drive in the list of Drives in the VM configuration, it already automatically boots into the GUI desktop environment.

**Step 12: Add Some Aliases**
I added `alias fresh='pacman -Syu'`. I also added `alias c='clear'` and `alias h='history` . I added these alias to the `~/.bashrc` file to make them permanent.

**Step 13: Install Browser**
I forgot to install a browser earlier, so I did it now using `pacman -S firefox`.

**Step 14: Install AUR package**
I did not install an AUR package yet either. I chose to install the `hollywood` package. To do so, I first installed `base-devel`. Then I tried cloning its repository with `git clone https://aur.archlinux.org/hollywood.git`. I did not have git, so I installed it using `pacman -S git`. It failed so I tried updating the mirrorlist so that it includes
`Server = http://mirror.archlinuxarm.org/$arch/$repo`
`Server = http://de3.mirror.archlinuxarm.org/$arch/$repo`
`Server = http://sg.mirror.archlinuxarm.org/$arch/$repo`
`Server = http://tw1.mirror.archlinuxarm.org/$arch/$repo`
and force-refreshed the `pacman` databases with our trusty command `pacman -Syyu`. Installing git worked after this. Then I cloned the `hollywood` repository. I changed directories to `hollywood`. Then tried to make and install the package using `makepkg -si`. It failed since I tried doing it as `root`. Oops. I switched users to `catherine` and tried again but it still failed, likely because the files had been cloned in and therefore owned by `root`. So I attemepted to change the ownership of the files using `chown -R catherine:catherine .`. I forgot to switch back to root so I did so and then tried changing the ownership again. I then switched back to the `catherine` user and tried making and installing the package again. It still failed. I realized that maybe I had the wrong build directory. So I tried uncommenting out the `BUILDDIR` variable in `/etc/makepkg.conf` but was unable to because I was doing it as the `catherine` user. So I switched back to `root` and tried again. Then I tried running `makepkg -si` again but it still failed. Perhaps the cloned directory still has artifacts leftover from its previous root ownership. So I tried clearing them out with `sudo rm -rf pkg src`. This didn't work because I forgot to set passwords for `catherine` and `codi`. So I changed to `root` and gave them passwords. I cleared the leftover build directories. I tried running `makepkg -si` again and it failed again. I suspected that the parent directory could possibly be owned by root still, so I ran `pwd` and confirmed it. So I tried moving the directory out of `/root` using `sudo mv /root/hollywood /home/catherine`. Apparently `catherine` does not have sudo permission even though I added `catherine` to the wheel group ? So I opened `/etc/sudoers` and tried to uncomment out the `%wheel ALL=(ALL:ALL) ALL` line. This didn't work because I was using `vim` instead of `visudo`. I tried using `visudo` and it still failed. So I ran `EDITOR=vim visudo` and commented out the line. As `catherine`, I ran `sudo mv /root/hollywood /home/catherine` again. Then I ran `sudo chown -R catherine:catherine /home/catherine/hollywood`. After changing to the `hollywood` directory, I was finally able to run `makepkg -si`. However it failed to install and resolve missing dependencies. So I ran `sudo pacman -S apg bmon byobu ccze cmatrix htop jp2a mlocate moreutils tree speedometer` to install the missing dependencies. A few could not be found, so `makepkg -si` still failed, even though those dependencies were optional for `hollywood`. So I just ran `makepkg`. It didn't work either so I resorted to commenting out the unfound dependencies from the `PKGBUILD`. I ran `makepkg -si` once again, and guess what? IT WORKED :D Now I can look like a cool hacker even though this documentation may suggest otherwise.

PS: If you made it this far, thanks for taking this journey with me..this is where my holiday went ;-;
