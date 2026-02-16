# Install **Proxmox ve** on **ISCSI** disk for **Bios** boot
To make Proxmox boot from mbr with bios you need to first install debian and then install pve on top.

🧪Tested on a HP dl380 g7

> [!Note]
> I assume you already have an iscsi drive somewhere like in truenas with Logical Block Size of 512 Bytes

### Debian installation
### Preparazione sistema (che può essere anche una live avviata da un pc o da una vm, non per forza l'hardware finale)
1. Installazione pacchetti necessari:
    ```bash
    sudo apt update
    sudo apt install -y debootstrap open-iscsi
    ```
2. connessione disco iscsi:
    ```bash
    sudo iscsistart -i <initiator iqn:name> -t <target iqn:disk> -g N -a <ip of target>
    ```
    Example
3. Creazione della partizione root(`/`) (In questo caso a me basta una partizione con tutto dentro)
    ```bash
    fdisk /dev/sdX
    o # create new empty MBR (DOS) partition table
    n # create primary partition starting from 2048
    ...
    a # make partition bootable\
    w # write data to disk
    ```
4. Formattazione partizione root(`/`)
    ```bash
    mkfs.ext4 /dev/sdX1
    ```
5. Mount della partizione
    ```bash
    mount /dev/sdX1 /mnt
    mount -o bind /dev /mnt/dev
    mount -o bind /sys /mnt/sys
    mount -o bind /proc /mnt/proc
    cp /etc/resolv.conf /mnt/etc/resolv.conf
    ```
6. Installazione sistema di base con debootstrap
    ```bash
    debootstrap <debian version name> /mnt/debian http://ftp.us.debian.org/debian
    ```
    Example:
    ```bash
    debootstrap trixie /mnt/debian http://ftp.us.debian.org/debian
    ```
7. Entering the new created system
    ```bash
    chroot /mnt/debian
    ```
8. Automatically mount root partition at boot
    1. Get partition UUID and copy it:
        ```bash
        ls -l /dev/disk/by-uuid
        ```
        Example output:
        ```
        total 0
        lrwxrwxrwx 1 root root 10 Feb 16 15:30 2303c60d-f44c-4db2-ad84-31ce13364fe4 -> sdb1
        ```
    2. Edit fstab file to tell wich disk to mount:
        ```bash
        nano /etc/fstab
        ```
        It should look like this, make sure you put the uuid copied before:
        ```
        # /etc/fstab: static file system information.
        #
        # file system                                   mount point     type    options                                        dump     pass
        UUID=2303c60d-f44c-4db2-ad84-31ce13364fe4       /               ext4    defaults,_netdev,x-systemd.requires=iscsid.service      0       1
        ```
9. Setting up the network:
    ```bash
    nano /etc/network/interfaces
    ```
    Add at the end of the file this lines:
    ```
    auto lo
    iface lo inet loopback
    ```
    Optionally add to the configuration of the network interfaces if you don't have dhcp based on your network like the example below:
    ```
    auto eth0
    iface eth0 inet static
        address 192.168.0.42
        network 192.168.0.0
        netmask 255.255.255.0
        broadcast 192.168.0.255
        gateway 192.168.0.1
    ```
10. Setting the hostname:
    1. Adding it to `hostname` file:
        ```bash
        echo <hostname> > /etc/hostname
        ```
        Example:
        ```bash
        echo pve > /etc/hostname
        ```
    2. Adding it to `hosts` file:
        ```bash
        nano /etc/hosts
        ```
        This file should look like this:
        ```
        127.0.0.1 localhost
        127.0.1.1 <hostname>

        # The following lines are desirable for IPv6 capable hosts
        ::1     ip6-localhost ip6-loopback
        fe00::0 ip6-localnet
        ff00::0 ip6-mcastprefix
        ff02::1 ip6-allnodes
        ff02::2 ip6-allrouters
        ff02::3 ip6-allhosts
        ```
11. Adding the proxmox repostitory and gpg key and updating the apt lists:
    ```bash
    echo "deb http://download.proxmox.com/debian/pve stretch pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
    wget https://enterprise.proxmox.com/debian/proxmox-archive-keyring-trixie.gpg -O /etc/apt/trusted.gpg.d/proxmox-archive-keyring.gpg
    apt update
    ```
13. Finding the correct proxmox kernel:
    ```bash
    apt-cache search proxmox-kernel
    ```
    There should be a long list of kernels, pick the one marked like `Latest Proxmox Kernel Image`, for me is the `proxmox-kernel-6.17 - Latest Proxmox Kernel Image`
14. Installing required packets, use the kernel from above command:
    ```bash
    apt install -y iscsi-boot grub2-common pve-headers proxmox-kernel-6.17 ssh initramfs-tools
    ```
15. Allow iscsi to manage iBFT information (iBFT are information made by iSCSI boot firmware like iPXE for the system)
    ```bash 
    echo "ISCSI_AUTO=true" > /etc/iscsi/iscsi.initramfs
    ```
16. Updating Initramfs:
    ```bash 
    update-initramfs -u -k all
    ```
17. Installing and updating Grub:
    ```bash
    grub-install /dev/sdX
    update-grub
    ```
18. Setting root password and enabling ssh:
    ```bash
    passwd
    ```
    Enablig root login with password by removing `#` from `PermitRootLogin yes`:
    ```bash
    nano /etc/ssh/sshd_config
    ```
    Or adding ssh key to login:
    ```bash
    mkdir /root/.ssh
    cat << EOF > /root/.ssh/authorized_keys
    <your ssh key>
    EOF
    ```
20. Exiting and unmounting all partitions and devices:
    ```bash
    exit
    umount -R /mnt
    ```
22. Detaching and removing Iscsi disk:
    ```bash
    sudo iscsiadm -m node -T <target iqn:disk> -p <ip of target> --logout
    sudo iscsiadm -m node -T <target iqn:disk> -p <ip of target> -o delete
    ```
### First boot
> [!Warning]
> 🚧🚧🏗️🚧🚧
> Work in progress


nano /etc/resolv.conf
setting up network config /etc/network/config
apt remove os-prober


## References
- [Debian manual installation guide](https://www.debian.org/releases/stable/amd64/apds03.en.html)
- [Proxmox manual installation guide for iscsi](https://pve.proxmox.com/wiki/Proxmox_ISCSI_installation)
