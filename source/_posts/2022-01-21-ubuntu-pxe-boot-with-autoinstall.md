---
title: Ubuntu (Focal) PXE Boot with autoinstall
date: 2022-01-21 12:37:44
tags:
  - homelab
  - linux
---

After `N` VMs deep in my homelab and delays upon delays (read: procrastination), I've finally gotten around to setting up my compute provisioning workflow to do [PXE boot](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) and [autoinstall](https://ubuntu.com/server/docs/install/autoinstall). Goodbye install UI, you will not be missed.

Note: I'm running on [Proxmox](https://www.proxmox.com/en/) and (virtual) [pfSense](https://www.pfsense.org/) but that shouldn't affect much of the setup.

## Setup

! Most, if not all, of the commands to be run must be root

### TFTP and HTTP Server

* Install the TFTP and HTTP server via:

```bash
apt-get -y install tftpd-hpa apache2
mkdir /tftp
chown tftp:tftp /tftp
```

* Edit `/etc/default/tftpd-hpa` with the following:

```
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

then

```bash
systemctl restart tftpd-hpa
```

* Create `/etc/apache2/conf-available/tftp.conf` with:

```
<Directory /tftp>
    Options +FollowSymLinks +Indexes
    Require all granted
</Directory>
Alias /tftp /tftp
```

then

```bash
a2enconf tftp
systemctl restart apache2
```

### BIOS Boot

* Download the latest .iso and extract the kernel and initrd by:

```bash
wget https://releases.ubuntu.com/20.04.3/ubuntu-20.04.3-live-server-amd64.iso -O /tftp/ubuntu-20.04.3-live-server-amd64.iso
mount /tftp/ubuntu-20.04.3-live-server-amd64.iso /mnt/
cp /mnt/casper/{vmlinuz,initrd} /tftp/
umount  /mnt
```

* Copy over the bootloader to the tftp directory by:

```bash
wget http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/current/legacy-images/netboot/ubuntu-installer/amd64/pxelinux.0 -O /tftp/pxelinux.0
```

* Install syslinux via:

```bash
apt-get -y install syslinux-common
```
then

```bash
cp /usr/lib/syslinux/modules/bios/{ldlinux.c32,libcom32.c32,libutil.c32,vesamenu.c32} /tftp
```

* Create `/tftp/pxelinux.cfg/default` file with:

```
DEFAULT vesamenu.c32
TIMEOUT 200
ONTIMEOUT ubuntu-focal-autoinstall
PROMPT 0
NOESCAPE 1

LABEL ubuntu-focal-autoinstall
        MENU label Install Ubuntu Focal - autoinstall
        KERNEL vmlinuz
        INITRD initrd
        APPEND root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://<SERVER-IP>/tftp/ubuntu-20.04.3-live-server-amd64.iso autoinstall ds=nocloud-net;s=http://<SERVER-IP>/tftp/cloud-init-bios/ cloud-config-url=http://<SERVER-IP>/tftp/cloud-init-bios/meta-data

LABEL ubuntu-focal-install
        MENU label Install Ubuntu Focal
        KERNEL vmlinuz
        INITRD initrd
        APPEND root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://<SERVER-IP>/tftp/ubuntu-20.04.3-live-server-amd64.iso
```

Where:

`<SERVER-IP>` - IP of the TFTP server

### "autoinstall" Config

* Create `/tftp/cloud-init-bios/user-data` with (password is `ubuntu`):

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: ubuntu-server
    password: "$6$exDY1mhS4KUYCE/2$zmn9ToZwTKLhCw.b4/b.ZRTIZM30JZ4QrOQ2aOXJ8yk96xpcCof0kxKwuX1kqLG/ygbJ1f8wxED22bTL4F46P0"
    username: ubuntu
```
More options can be found in the [reference guide](https://ubuntu.com/server/docs/install/autoinstall-reference).

* Run `touch /tftp/cloud-init-bios/meta-data`

### DHCP Update

In my case, I'm running pfSense so I had to update the DHCP server with:

{% asset_img pfsense_dhcp.jpg REEEEEE %}

## Testing in Proxmox

To test if everything checks out, simply create a VM with no OS media. Network boot is already included in the boot order (last).

## Gotchas

Minimum VM memory should be atleast 2GB in order for the installation to succeed.

## References

* https://ubuntu.com/server/docs/install/netboot-amd64
* https://askubuntu.com/questions/1238070/deploy-ubuntu-20-04-on-bare-metal-or-virtualbox-vm-by-pxelinux-cloud-init-doesn/1240068#1240068
