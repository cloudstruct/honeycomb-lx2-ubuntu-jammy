# honeycomb-lx2-ubuntu-jammy
Installing Ubuntu Jammy (22.04) on the SolidRun HoneyComb LX2 board

## Installation methods

There are two different methods for installing Ubuntu 20.04, using a pre-built Ubuntu image that includes the bootloader and using a generic UEFI image that allows booting the Ubuntu installer ISO image. Unfortunately, neither of these methods currently work if you want to use Ubuntu 22.04. Fortunately, we can fix this!

### Creating an Ubuntu image with embedded bootloader

The [official images](https://images.solid-run.com/LX2k/lx2160a_build) are built from the [`SolidRun/lx2160a_build`](https://github.com/SolidRun/lx2160a_build) Github repo. While the scripts in this repo will allow you to easily build your own images, they don't currently support Ubuntu 22.04, but only a few minor changes are needed.

The following changes are needed to the `runme.sh` script:

```diff
diff --git a/runme.sh b/runme.sh
index bb8881f..c8dcfee 100755
--- a/runme.sh
+++ b/runme.sh
@@ -237,8 +237,8 @@ case "\$1" in
                mount /dev/vda /mnt
                cd /mnt/
                udhcpc -i eth0
-               wget -c -P /tmp/ http://cdimage.ubuntu.com/ubuntu-base/releases/20.04/release/ubuntu-base-20.04.1-base-arm64.tar.gz
-               tar zxf /tmp/ubuntu-base-20.04.1-base-arm64.tar.gz -C /mnt
+               wget -c -P /tmp/ http://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04-base-arm64.tar.gz
+               tar zxf /tmp/ubuntu-base-22.04-base-arm64.tar.gz -C /mnt
                mount -o bind /proc /mnt/proc/
                mount -o bind /sys/ /mnt/sys/
                mount -o bind /dev/ /mnt/dev/
@@ -250,7 +250,7 @@ case "\$1" in
                echo "127.0.0.1 localhost" > /mnt/etc/hosts
                export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true LC_ALL=C LANGUAGE=C LANG=C
                chroot /mnt apt update
-               chroot /mnt apt install --no-install-recommends -y systemd-sysv apt locales less wget procps openssh-server ifupdown net-tools isc-dhcp-client ntpdate lm-sensors i2c-tools psmisc less sudo htop iproute2 iputils-ping kmod network-manager iptables rng-tools apt-utils
+               chroot /mnt apt install --no-install-recommends -y systemd-sysv apt locales less wget procps openssh-server ifupdown net-tools isc-dhcp-client ntpdate lm-sensors i2c-tools psmisc less sudo htop iproute2 iputils-ping kmod network-manager iptables rng-tools apt-utils fdisk
                echo -e "root\nroot" | chroot /mnt passwd
                umount /mnt/var/lib/apt/
                umount /mnt/var/cache/apt
```

After these changes, you can follow the instructions in that repo to build your image and then follow the [instructions in the Quickstart Guide](https://solidrun.atlassian.net/wiki/spaces/developer/pages/197494288/HoneyComb+LX2+ClearFog+CX+LX2+Quick+Start+Guide#Booting-from-an-SD-card).

### Use UEFI to boot installer ISO image

The [official UEFI images](https://images.solid-run.com/LX2k/lx2160a_uefi) are built from the [`SolidRun/lx2160a_uefi`](https://github.com/SolidRun/lx2160a_uefi) Github repo. Fortunately, we don't need to rebuild these images. You can write this image to a SD card and set it aside for later.

Unfortunately, the kernel used on the Ubuntu 22.04 installer ISO image has a "bug" that causes a bunch of errors to spew to the console and the machine to not finish booting. There are some kernel commandline parameters you can pass to work around this, but the kernel still doesn't support the eMMC/SD or the onboard NIC. We can rebuild the ISO image with a custom kernel, but it's not exactly straightforward.

#### Download and extract the official installer ISO

Starting by downloading the Ubuntu Server 22.04 ARM64 installer ISO image.

```bash
$ curl -O https://cdimage.ubuntu.com/releases/22.04/release/ubuntu-22.04-live-server-arm64.iso
```

Mount it and extract the contents.

```bash
$ mkdir mnt
$ sudo mount -o loop ubuntu-22.04-live-server-arm64.iso mnt
$ mkdir extract-cd
$ cp -a mnt/* extract-cd/
$ sudo umount mnt
```

#### Build the custom kernel

This procedure is based on [Carlos Eduardo's excellent post](https://carlosedp.medium.com/solidrun-honeycomb-arm-up-and-running-56b3de896143), but I've made some changes based on my experience.

Clone the `SolidRun/linux-stable` repo and checkout the branch corresponding to the `5.15.x` kernel (the same version used for Ubuntu 22.04).

```bash
$ git clone https://github.com/SolidRun/linux-stable.git -b linux-5.15.y-cex7
```

Mount one of the squashfs files from ISO image contents and extract the kernel config.

```bash
$ sudo mount -o loop extract-cd/casper/ubuntu-server-minimal.ubuntu-server.installer.generic.squashfs mnt
$ cp mnt/boot/config-5.15.0-25-generic linux-stable/.config
```

Update the kernel config and build the DEB packages. This can be run on an AMD64 host if you have the cross-toolchain installed, but I personally did the build on a Raspberry Pi 4.

```bash
$ cd linux-stable
$ sed -i 's/CONFIG_SYSTEM_TRUSTED_KEYS.*/CONFIG_SYSTEM_TRUSTED_KEYS=""/' .config
$ sed -i 's/CONFIG_SYSTEM_REVOCATION_KEYS.*/CONFIG_SYSTEM_REVOCATION_KEYS=""/' .config
$ scripts/config --disable DEBUG_INFO
$ scripts/config --disable MODULE_SIG
$ ./scripts/config --enable CONFIG_FIXED_PHY
$ ./scripts/config --enable CONFIG_OF_MDIO
$ ./scripts/config --enable CONFIG_ACPI_MDIO
$ ./scripts/config --enable CONFIG_PHYLIB
$ ./scripts/config --enable CONFIG_ARM_SMMU_DISABLE_BYPASS_BY_DEFAULT
$ ./scripts/config --enable CONFIG_FSL_MC_UAPI_SUPPORT
$ make olddefconfig
$ make CROSS_COMPILE=aarch64-linux-gnu- ARCH=arm64 -j`nproc` INSTALL_MOD_STRIP=1 bindeb-pkg
```

#### Rebuild ISO image with custom kernel

TODO
