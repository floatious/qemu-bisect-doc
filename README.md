# Automatic git bisect using QEMU and PCI passthrough

Doing a manual git bisect on server hardware is lifedraining.
It can take a really long time to reboot certain servers.

Luckily, many problems can be reproduced in QEMU, and therefore
don't need to run on real hardware. Rebooting a QEMU machine is
quick, especially with KVM, and we can use "git bisect run" to
automate it.

Unfortunately, certain problems can only be reproduced on real hardware.
However, we still don't want to wait for server hardware to reboot.
The solution is to use QEMU with the real device supplied via PCI
passthrough.

This is not fool proof though, as you are limited by the iommu groups
defined by your hardware. You will need to pass through all PCI devices
in the same iommu group (except for PCI bridges). This is not a problem
if the device you want to pass through is alone in the iommu group, or if
the other devices in the iommu group are not system critical for the host.
Don't worry if you have PCI bridges in your iommu group, bridges will be
ignored by the scripts.

The first step in this guide is therefore to visualize the iommu groups:
```
for g in $(find /sys/kernel/iommu_groups/ -maxdepth 1 -mindepth 1 -type d | sort -V); do
	echo "IOMMU group $(basename "$g"):"
	for d in $(\ls -1 "$g"/devices/); do
		echo -n $'\t'
		lspci -Dnns "$d"
	done
done
```

If you have some system critical devices in the same iommu group, you can
stop reading now. (For example, the device that contains your root file system
is in the same iommu group as the device that you want to pass through.)
(If you really need to perform an automatic bisect, you could disable the iommu
and modprobe vfio with the module parameter enable_unsafe_noiommu_mode=1.)

If the device you want to pass through is alone in the iommu group or only
shared with devices that are not system critical, continue with the guide.
Note that all devices, in the same iommu group as the device that you want to
pass through, will be unavailable to the host while one device in the group is
passed through.

Install QEMU.
On Ubuntu/Debian:
```
sudo apt install qemu-system-x86 qemu-utils
```

or on Fedora/RHEL:
```
sudo dnf install qemu-system-x86 qemu-img
```

Create a new directory were you will keep the new code, e.g.:
```
mkdir ~/src
```

Clone and build the kernel that you will run inside QEMU:
```
cd ~/src
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
make defconfig
./scripts/config --enable CONFIG_BTRFS_FS
./scripts/config --enable CONFIG_BTRFS_FS_POSIX_ACL
./scripts/config --enable CONFIG_PSI
./scripts/config --enable CONFIG_MEMCG
./scripts/config --enable CONFIG_CRYPTO_LZO
./scripts/config --enable CONFIG_ZRAM
./scripts/config --enable CONFIG_ZRAM_DEF_COMP_LZORLE
make olddefconfig
make -j$(nproc)
```

Create a new directory were you will keep the QEMU scripts and data, e.g.:
```
mkdir ~/qemu-data
```

Get a QEMU friendly rootfs image (this guide uses Fedora) using:
```
cd ~/qemu-data
wget https://download.fedoraproject.org/pub/fedora/linux/releases/36/Cloud/x86_64/images/Fedora-Cloud-Base-36-1.5.x86_64.qcow2
```

Create a new image that will be used as our rootfs, based on the Fedora image that we just downloaded.
It will not modify the Fedora image we just downloaded. Differences will be saved in rootfs-overlay.img.
It will not take up 128 GB, it will simply allow it to grow up to that size, since this new file will automatically
increase in size when we install new packages.
```
qemu-img create -f qcow2 -b Fedora-Cloud-Base-36-1.5.x86_64.qcow2 -F qcow2 rootfs-overlay.img 128G
```

Install cloud-localds on Ubuntu/Debian:
```
sudo apt install cloud-image-utils genisoimage
```

or on Fedora/RHEL:
```
sudo dnf install cloud-utils genisoimage
```

Cloud images these days are shipped without any default password for security reasons.
Therefore we need to set a root password inside your new image. This is done using cloud-config.
Create the user-data file by pasting the following into a terminal and press enter:
```
cat >user-data <<EOF
#cloud-config
chpasswd:
  list: |
    root:your_password
  expire: False
EOF
```

Open the file and change your_password to whatever you prefer.

Generate the binary file user-data.img using:
```
cloud-localds user-data.img user-data
```

Add the following to /etc/security/limits.conf
```
your_username             hard    memlock         unlimited
your_username             soft    memlock         unlimited
```

Replace your_username with your username.

You need to logout and login again for changes to take effect.
You can run:
```
ulimit -l
```

To verify that the changes have taken place.

Create a file named setup_dev.sh containing the following:
Modify the variable to match your setup.
```
#!/bin/sh

# Change this to contain the PCI address of the device you want to pass through
# to QEMU. In this example, we will use:
# 0000:00:14.3 Network controller: Intel Corporation Cannon Point-LP CNVi [Wireless-AC] (rev 11)
# Therefore, the variable will be set to: 0000:00:14.3
PCI_BDF=0000:00:14.3

sudo modprobe vfio-pci

function bind_driver() {
	local drv="$1"
	for DEV in $(ls /sys/bus/pci/devices/$PCI_BDF/iommu_group/devices) ; do
		if [[ $(( 0x$(setpci -s $DEV HEADER_TYPE) & 0x7f )) -eq 0 ]]; then
			sudo sh -c "echo $drv > /sys/bus/pci/devices/$DEV/driver_override"
			sudo sh -c "echo $DEV > /sys/bus/pci/devices/$DEV/driver/unbind"
			sudo sh -c "echo $DEV > /sys/bus/pci/drivers_probe"
		fi
	done
}

if [ $# -eq 1 ] && [ $1 = reset ]; then
	bind_driver ""
else
	bind_driver "vfio-pci"
	group=$(readlink /sys/bus/pci/devices/$PCI_BDF/iommu_group)
	group_nbr=$(printf '%s' $group | sed 's,.*/iommu_groups/,,')
	sudo chown $USER:$USER /dev/vfio/$group_nbr
fi
```

Create a file named launch_qemu.sh containing the following:
Modify the variables to match your setup.
```
#!/bin/sh

# Change this to contain the PCI address of the device you want to pass through
# to QEMU. It has to be the same PCI address as you specified in setup_dev.sh
PCI_BDF=0000:00:14.3
KERNEL=~/src/linux/arch/x86/boot/bzImage
ROOTFS=~/qemu-data/rootfs-overlay.img
USER_DATA=~/qemu-data/user-data.img
QEMU_GUEST_SSH_FWD_PORT=10222
RAM=4G

qemu-system-x86_64 -m $RAM -cpu host -smp $(nproc) -enable-kvm -nographic \
             -drive file=$ROOTFS,format=qcow2,if=virtio \
             -drive file=$USER_DATA,format=raw,if=virtio \
             -kernel $KERNEL \
             -append "console=ttyS0 root=/dev/vda5 rootflags=subvol=root net.ifnames=0" \
             -device virtio-net-pci,netdev=usernet \
             -netdev user,id=usernet,hostfwd=tcp::$QEMU_GUEST_SSH_FWD_PORT-:22 \
             -device vfio-pci,host=$PCI_BDF
```

Make the scripts executable and run them:
```
chmod +x setup_dev.sh launch_qemu.sh
./setup_dev.sh
./launch_qemu.sh
```

You can kill your QEMU machine by typing Ctrl-A and then X.

At the end, when you are done with the automated bisection,
you can reset the PCI device to the original driver by running:
```
./setup_dev.sh reset
```

Login to your QEMU machine with user root and the password you chose previously.

The first thing you should do after logging on to the QEMU machine is to update the package lists
and make sure that we are running the latest security and bug fixes:
```
dnf upgrade --refresh
```

If you prefer to type in a real terminal rather than the limited QEMU console,
you can run:
```
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
systemctl restart sshd
```

and then you will have the option to log on to your QEMU machine using SSH:
```
ssh -A -p 10222 root@localhost
```

Now it is time to install fio and nvme-cli, or any other tools that are needed
in order to trigger the kernel bug inside your QEMU machine:
```
dnf install fio nvme-cli
```

Create a new init file, /myinit, on the root file system of your QEMU machine.
This is basically a test that should trigger the kernel bug on a faulty kernel,
and return "Success" on a non-faulty kernel.
Copy and modify to match your reproducer:
```
#!/bin/sh

export PATH=/usr/sbin:/usr/bin
mount -t sysfs -o nodev,noexec,nosuid sysfs /sys
mount -t proc -o nodev,noexec,nosuid proc /proc
mount -t tmpfs -o nodev,nosuid tmpfs /tmp

## enable network
## reprobe wifi, since firmware on rootfs wasn't mounted at probe time
#echo 0000:00:04.0 > /sys/bus/pci/drivers_probe
#nmcli device wifi connect MYSSID password mysecretpass
#ip route del default

## could e.g. connect to a NVMe-oF
#nvme list

## could e.g. run some fio workload
#fio --name=test --filename=/dev/null --ioengine=io_uring --rw=randread --size=10G

## could e.g. run some python script
#printf "%s" "print('Hello {}'.format('Python'))" | python

## could e.g. disconnect to a NVMe-oF
#nvme list

## could e.g. sleep after disconnect so that the warning has time to print
#sleep 5

## could e.g. print Success if no kernel warning in dmesg during disconnect
#warning=$(dmesg | grep WARNING)
#if [ -z "$warning" ]; then
#	echo Success
#fi


# my reproducer example:
# bisect to find the commit that increased boot time by more than 10 seconds
time_to_ready=$(awk '{print int($1)}' /proc/uptime)
echo "time to boot was: $time_to_ready seconds"
if [ $time_to_ready -lt 5 ]; then
	echo Success
fi


if [ $$ -eq 1 ]; then
	poweroff -f
fi
```

Once you have installed all tools and have created a /myinit script that can
reproduce the kernel bug that we want to bisect, make the script executable
and turn off your QEMU machine:
```
chmod +x /myinit
poweroff
```

Now open your launch_qemu.sh script again, and change this line:
```
-append "console=ttyS0 root=/dev/sda5 rootflags=subvol=root net.ifnames=0" \
```
to:
```
-append "console=ttyS0 root=/dev/sda5 rootflags=subvol=root net.ifnames=0 init=/myinit" \
```

So that your init script will run automatically when starting the QEMU machine,
instead of the regular systemd boot.

Create a file named bisect.sh containing the following:
```
#!/bin/sh

#EXTRAS="CC=clang"
EXTRAS=

# you can tweak the working tree by cherry-picking hot-fixes
#if git cherry-pick -n 52a9dab6d892763b2a8334a568bd4e2c1a6fde66 &&
#make $EXTRA olddefconfig && make $EXTRA -j$(nproc)
if make $EXTRA olddefconfig && make $EXTRA -j$(nproc)
then
	# run project specific test and report its status
	~/qemu-data/launch_qemu.sh | grep ^Success
	status=$?
else
	# tell the caller this is untestable
	status=125
fi

# undo the tweak to allow clean flipping to the next commit
git reset --hard

# return control
exit $status
```

Make the script executable:
```
chmod +x bisect.sh
```

Now it is time to start the bisect.
You need to specify a known good tag and a known bad tag.

Here is an example when I bisected the boot time regression:
```
git bisect start
git bisect good linux-next/stable
git bisect bad next-20220602
git bisect run ~/qemu-data/bisect.sh
```

Which eventually gives the resulting output:
```
2b28a1a84a0eb3412bad1a2d5cce2bb4addec626 is the first bad commit
commit 2b28a1a84a0eb3412bad1a2d5cce2bb4addec626
Author: Saravana Kannan <saravanak@google.com>
Date:   Fri Apr 29 15:09:32 2022 -0700

    driver core: Extend deferred probe timeout on driver registration
    
    The deferred probe timer that's used for this currently starts at
    late_initcall and runs for driver_deferred_probe_timeout seconds. The
    assumption being that all available drivers would be loaded and
    registered before the timer expires. This means, the
    driver_deferred_probe_timeout has to be pretty large for it to cover the
    worst case. But if we set the default value for it to cover the worst
    case, it would significantly slow down the average case. For this
    reason, the default value is set to 0.
    
    Also, with CONFIG_MODULES=y and the current default values of
    driver_deferred_probe_timeout=0 and fw_devlink=on, devices with missing
    drivers will cause their consumer devices to always defer their probes.
    This is because device links created by fw_devlink defer the probe even
    before the consumer driver's probe() is called.
    
    Instead of a fixed timeout, if we extend an unexpired deferred probe
    timer on every successful driver registration, with the expectation more
    modules would be loaded in the near future, then the default value of
    driver_deferred_probe_timeout only needs to be as long as the worst case
    time difference between two consecutive module loads.
    
    So let's implement that and set the default value to 10 seconds when
    CONFIG_MODULES=y.
    
    Cc: Greg Kroah-Hartman <gregkh@linuxfoundation.org>
    Cc: "Rafael J. Wysocki" <rjw@rjwysocki.net>
    Cc: Rob Herring <robh@kernel.org>
    Cc: Linus Walleij <linus.walleij@linaro.org>
    Cc: Will Deacon <will@kernel.org>
    Cc: Ulf Hansson <ulf.hansson@linaro.org>
    Cc: Kevin Hilman <khilman@kernel.org>
    Cc: Thierry Reding <treding@nvidia.com>
    Cc: Mark Brown <broonie@kernel.org>
    Cc: Pavel Machek <pavel@ucw.cz>
    Cc: Geert Uytterhoeven <geert@linux-m68k.org>
    Cc: Yoshihiro Shimoda <yoshihiro.shimoda.uh@renesas.com>
    Cc: Paul Kocialkowski <paul.kocialkowski@bootlin.com>
    Cc: linux-gpio@vger.kernel.org
    Cc: linux-pm@vger.kernel.org
    Cc: iommu@lists.linux-foundation.org
    Reviewed-by: Mark Brown <broonie@kernel.org>
    Acked-by: Rob Herring <robh@kernel.org>
    Signed-off-by: Saravana Kannan <saravanak@google.com>
    Link: https://lore.kernel.org/r/20220429220933.1350374-1-saravanak@google.com
    Signed-off-by: Greg Kroah-Hartman <gregkh@linuxfoundation.org>

 Documentation/admin-guide/kernel-parameters.txt |  6 ++++--
 drivers/base/base.h                             |  1 +
 drivers/base/dd.c                               | 19 +++++++++++++++++++
 drivers/base/driver.c                           |  1 +
 4 files changed, 25 insertions(+), 2 deletions(-)
bisect found first bad commit
```
