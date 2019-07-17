---
layout: post
title:  "Configuring GPU Passthrough with VFIO on Fedora 30 notes"
date:   2019-07-16 19:38:21 -0700
categories: Virtualization KVM VFIO
---

GPU Passthrough is all the rage.  The ArchLinux Wiki does an amazing job explaining all the different ways to setup VFIO/GPU passthrough.  These are my notes for setting up GPU Passthrough in Fedora 30 with two identical cards.  This does not aim to be a comprehensive guide.

## Setup KVM
 
Install KVM

```bash
sudo dnf -y install bridge-utils libvirt virt-install qemu-kvm virt-top libguestfs-tools virt-manager
```

Configure bridge networking using NetworkManager (this is largely due to laziness when setting the OS up, NM is the default, so I kinda just deal with it...)

```bash
# make up a bridge name, I like br0
export BR_NAME=br0
# this will vary depending on the mobo/eth card.  see `ip addr` to find device name
export BR_INTERFACE=enp5s0

# Get our device UUID
BR_INT_OG_UUID=$(nmcli -g GENERAL.CON-UUID device show "${BR_INTERFACE}")

# Create bridge
nmcli connection add type bridge autoconnect yes con-name "{BR_NAME}" ifname "${BR_NAME}"

# Disable STP
nmcli connection modify "${BR_NAME}" bridge.stp no

# Add our interface to our bridge
nmcli connection add type bridge-slave autoconnect yes con-name ${BR_INTERFACE} ifname ${BR_INTERFACE} master ${BR_NAME}

# Turn off our old interface
nmcli con down "${BR_INTERFACE}"

# Turn on our new bridge
nmcli con up "${BR_NAME}"

# delete our old interface config
nmcli con delete "${BR_INT_OG_UUID}"
```

## Update Kernel CLI parameters

Append to `GRUB_CMDLINE_LINUX` in `/etc/default/grub` to enable iommu and load our `vfio-pci` driver.  We do this early to ensure `vfio` is able to claim the devices before the kernel does.

```bash
GRUB_CMDLINE_LINUX="amd_iommu=on iommu=pt rd.driver.pre=vfio-pci"
```

## Modprobe Config

Create `/etc/modprobe.d/vfio.conf` to tell modprobe how to load `vfio-pci`

```
install vfio-pci /sbin/vfio-pci-override.sh
```

## Dracut Config

Create `/etc/dracut.conf.d/vfio.conf` to load our drivers and setup script into our initrd.

```
add_drivers+="vfio vfio_iommu_type1 vfio_pci vfio_virqfd"
install_items+="/sbin/vfio-pci-override.sh /usr/bin/dirname"
```

## VGA Passthrough Script

Create `/sbin/vfio-pci-override.sh`, this will passthrough all non-boot VGA devices, including audio and USB controllers if they exist.

{% gist 2ff70cc63346a09c2374262e1075564c vfio-pci-override.sh %}

```bash
!/bin/sh

set -u

for boot_vga in /sys/bus/pci/devices/*/boot_vga; do
  echo "Found vga device: ${boot_vga}"
  if [ $(<"${boot_vga}") -eq 0 ]; then
    echo "Found Boot VGA Device - false: ${boot_vga}"
    
    dir=$(dirname -- "${boot_vga}")
    for dev in "${dir::-1}"*; do
      echo "Registering Devices: ${dev}"
      echo 'vfio-pci' > "${dev}/driver_override"
    done
  else
    echo "Found Boot VGA Device - true: ${boot_vga}"
  fi
done

modprobe -i vfio-pci
```

## Update our boot system

```bash
# update initrd
# or dracut -f --regenerate-all
dracut -f --kver $(uname -r)

# update grub
grub2-mkconfig > /etc/grub2-efi.cfg 
```

## Post setup Check

{% gist 2ff70cc63346a09c2374262e1075564c vfio-setup-check.sh  %}

(this could be improved)

## Links

* [Cybercity `nmcli` guide](https://www.cyberciti.biz/faq/how-to-add-network-bridge-with-nmcli-networkmanager-on-linux/)
* [Arch Linux VFIO Wiki](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)
* [VFIO Blogspot](http://vfio.blogspot.com/2015/05/vfio-gpu-how-to-series-part-3-host.html)
