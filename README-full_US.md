# 🚀 Industrial deployment of Linux workstations via PXE

> **Complete documentation – from PXE architecture to manual and automated installation**

---

## 📌 Introduction

This document is the **complete and detailed version** of the Linux workstation deployment project based on **PXE**.

The objective is to **industrialize the provisioning of Linux workstations**, covering not only the operating system installation but the **entire initial lifecycle of a workstation**:

- network boot,
- OS installation,
- system configuration,
- integration with a configuration and compliance management tool,
- continuous operational maintenance.

At the end of the process, the workstation is:

- ✅ installed and up to date
- ✅ configured (system and applications)
- ✅ enrolled in a compliance and supervision platform (Rudder)

🎯 The ambition is to provide, for Linux environments, an approach **functionally comparable to SCCM / MECM** in Windows ecosystems, while remaining adapted to the realities of Linux workstations (hardware diversity, flexibility, simplicity).

📎 The procedure applies to both **physical machines** and **virtual machines**.

---

## 🧑‍💻 User experience – High-level overview

```text
1. A new workstation is connected
2. Network boot (PXE)
   - IP address assignment
   - PXE menu display
3. Installation profile selection

4. Automated installation:
   - Disk partitioning
   - Operating system installation
   - Desktop environment installation
   - Additionnals applications and packages installation
   - Additionnals configurations
   - Rudder agent installation and enrollment

5. Reboot
6. Workstation ready for use
7. Configuration and compliance continuously enforced
```

👉 **No human interaction is required during the installation phase.**

---

## 🧠 Assumptions and architectural choices

The following assumptions are used throughout this document:

- **PXE** and **DHCP** servers are located on the same network as client machines (`192.168.1.0/24`).
- The **DHCP** server and the **PXE** server are two separate systems (architectural choice, not a technical constraint).
- The configuration and compliance management tool used is **Rudder**.

📎 Recommended introduction to Rudder:
👉 https://blog.stephane-robert.info/docs/infra-as-code/gestion-de-configuration/rudder/

### 🔄 Possible architectural variants

**1️⃣ Centralized DHCP with PXE on a separate VLAN**
- Centralized DHCP
- PXE server located in another VLAN
- DHCP relay via IP Helper

**2️⃣ Dedicated deployment network**
- VLAN or subnet dedicated to deployment
- PXE server providing DHCP + TFTP
- Workstations temporarily connected to this network

---

# 🧩 Part I – PXE infrastructure setup

This first part covers the **complete setup of the PXE infrastructure**, up to the point where a menu allows launching a **manual operating system installation**.

**Part II** will focus on fully industrialized deployment (unattended and silent installation).

---

## 🛠️ Step 0 – Ubuntu Server 24.04 LTS installation

### 📋 Minimum hardware requirements

- CPU: 1 GHz or higher
- RAM: 1 GB
- Storage: 5 GB available

> ℹ️ Ubuntu Server installation is not detailed here. Numerous tutorials are available online.

📎 Example: https://www.linux-fra.com/?p=15124

### 📥 ISO download

- Ubuntu Server 24.04 LTS  
  https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso

### ⚙️ Installation parameters

- Type: default installation
- Server hostname (example): `lxpxe01`
- Created user: `user01`

### 🔄 Post-installation update

```bash
sudo apt update
sudo apt upgrade
```

---
## 🔐 Step 1 – SSH Installation and Configuration

This step is not strictly mandatory, but **strongly recommended**, especially for remote administration of a physical server.

### 1️⃣ Installing the SSH service

```bash
sudo apt install openssh-server
```

### 2️⃣ Checking the service status

```bash
sudo systemctl status ssh
```

The service should appear as **active (running)** and **enabled**.

If the Ubuntu SSH service is running, you will see output:
```bash
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-02-05 22:12:35 UTC; 5min ago
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
   Main PID: 1265 (sshd)
      Tasks: 1 (limit: 9336)
     Memory: 5.5M (peak: 6.2M)
        CPU: 145ms
     CGroup: /system.slice/ssh.service
             └─1265 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

As SSH must also be available each time the system starts, the **`preset: enabled`** entry must also appear in the **`Loaded`** line.
If the SSH service remains inactive and its automatic launch after restart is not activated, you can then enter two additional commands:

If necessary:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### 3️⃣ Opening SSH port in firewall
UFW is Ubuntu's configuration program dedicated to the system's proprietary firewall.  
With this program, configure an appropriate rule for communication from SSH, in order to open the port for both incoming and outgoing data:
```bash
sudo ufw allow ssh
```

### 4️⃣ Advanced configuration (optional)

The main configuration file is: **/etc/ssh/sshd_config**.  
If you want to modify it, open it from the text editor of your choice (here, it's nano) with this command:  

```bash
sudo nano /etc/ssh/sshd_config
```

In particular, you can:
- change the SSH port
- disable TCP redirection
- etc.

After modification, a small restart of SSH is necessary.

```bash
sudo service ssh restart
```

### 5️⃣ Connection test

```bash
ssh user01@lxpxe01
```

📌 **One last thing**. 
The PXE server must have a **static IP address** (DHCP reservation recommended).
For this procedure, I simply reserved an IP for this server on DHCP then renewed the lease.

```bash
sudo dhclient -r
sudo dhclient
```
If the dhcp client tools are not installed, you will need to precede these two commands with:  

```bash
sudo apt install isc-dhcp-client
```

🎉 **Here we are ready for the rest!**

---

## 🛠️ Step 2 – Installing the necessary PXE services

PXE is based on the following components:

- **dnsmasq**: TFTP / PXE (and possibly DHCP)
- **Apache2**: storage server
- **NFS**: NFS packages provide file services functionality.
- **syslinux / grub**: BIOS and UEFI bootloaders

The setup is based on dnsmasq, a small lightweight server which will be our TFTP, Apache2, NFS and syslinux for the bootloaders.  
For information, dnsmasq (https://fr.wikipedia.org/wiki/Dnsmasq) can also be used as a DHCP and DNS server.
Remember our DCHP already exists, you will need to configure your external DHCP to point to this PXE server. 
(Or use dnsmasq like DHCP, we'll come back to that :))

> ℹ️ Configure your existing DHCP: add in the bootloader VLAN subnet: "pxelinux.0"; next-server <PXE_server_IP>; and restart it.

### Installing packages

```bash
sudo apt-get install apache2
sudo apt-get install nfs-kernel-server 
sudo apt-get install dnsmasq
```

### Download pxelinux packages
```bash
cd ~/tmp/
wget https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.zip
unzip syslinux-6.03.zip
```

### Download UEFI packages
```bash
cd ~/tmp/
apt-get download shim.signed
dpkg -x <%nom du package%> shim

apt-get download grub-efi-amd64-signed
dpkg -x <%nom du package%> grub
```

---
## 📁 Step 3 – Creation of TFTP and WEB trees

### 3.1 TFTP tree

```text
/tftp
 ├─ bios
 ├─ boot
 └─ grub
```

```bash
sudo mkdir /tftp
sudo mkdir /tftp/bios
sudo mkdir /tftp/boot
sudo mkdir /tftp/grub
```

### 3.2 WEB tree (Apache)
As we are using the Apache web server, we will copy all the source files into the `/var/www/html` directory.  
We will copy the contents of the Ubuntu 25.10 Desktop iso files to this location.

Our structure will look like the following representation.   
You can of course create your own structure.


```text
/var/www/html
 └─ desktop
     └─ u2510
```

```bash
sudo mkdir -p /var/www/html/desktop/u2510
```

---

## 📦 Step 4 – Populating the WEB directory

### Mounting the ISO
The OS that we want to deploy in our example is Ubuntu Desktop 25.10, you will have to download it and mount the ISO.  
It will look like this:  

```bash
cd ~/Téléchargements
wget -c "https://releases.ubuntu.com/25.10/ubuntu-25.10-desktop-amd64.iso"

sudo mount ~/Téléchargements/ubuntu-25.10-desktop-amd64.iso /media
```

### Copying files

```bash
sudo cp -rf /media/* /var/www/html/desktop/u2510
sudo cp -rf /media/.disk /var/www/html/desktop/u2510
```
The 1st command above copies all the contents of the source iso, except for a hidden folder necessary for the pxe process to work properly.  
So you need to run an additional command to make sure that all the files you need have been copied correctly. (second command line above)

### Disassembly of the ISO

```bash
sudo umount /media
```

---

## 📡 Step 5 – NFS Server Configuration
With our folder structure ready, we can start configuring the different services used by the PXE server.   To ensure that our directory structure is accessible over the network and through the nfs protocol, we need to modify the following file by running the following command.

Edit the `/etc/exports` file:

```bash
sudo nano /etc/exports
```

Insert at the end of the file the path where you stored your installation files, the subnet that can access them, and the type of permission you want to grant.  In our scenario, we want to grant access to the following directory /var/www/html/desktop via subnet 192.168.1.0/24 and we grant read-only (ro) access.  
So at the end of the file we would add the following line 

```text
/var/www/html/desktop 192.168.1.0/24(ro)
```

Restart the service:

```bash
sudo systemctl restart nfs-kernel-server
```

---

## 🧩 Step 6 – Configuring dnsmasq

### Minimum requirements (PXE only)

We must now configure the dnsmasq service which will ensure the connection between the different services.  
The dnsmasq configuration file will be used to provide the necessary information to the pxe client when it starts.  
This file will indicate where to look for the pxe bootloader depending on the client architecture (uefi or bios).  
So let's edit the /etc/dnsmasq.conf file and add the following information at the end

To modify the configuration file, run the following command  

```bash
sudo nano /etc/dnsmasq.conf 
```
The contents of the file should look like this:
```cfg
enable-tftp
tftp-root=/tftp
```

### Example with integrated DHCP (optional)
As said before in our architecture, we already have a DCHP server, but if you want to use dnsmasq to manage both roles, it's possible!
In this case the file could look more like this:
(This is an example)

```ini
#Interface 
#-- Vous trouverez le nom de votre interface avec un ip a
interface=eth0,lo
bind-interfaces
domain=mondomain.local

#--------------------------
#-- DHCP
#--------------------------
#-- Range DHCP
dhcp-range=192.168.1.10,192.168.1.150,255.255.255.0,2h

#-- Passerelle
dhcp-option=3,192.168.1.1

#-- DNS
dhcp-option=6,192.168.1.160
server=8.8.8.8

#----------------------#
#-- TFTP 
#----------------------#

#--Chemin vers le pxeboot (à adapter)
dhcp-boot=/bios/pxelinux.0,pxeserver,192.168.1.160


enable-tftp
tftp-root=/tftp

#-- Detection de l'architecture et lancement du bon bootloader
dhcp-match=set:efi-x86_64,option:client-arch,7 
dhcp-boot=tag:efi-x86_64,grub/bootx64.efi
```

**Let's start again,**

restarting the service:

```bash
sudo systemctl restart dnsmasq
```

To verify that dnsmasq starts correctly and without errors:

```bash
sudo systemctl status dnsmasq
```
---
## Step 7 – Contents of TFTP & Boot folders directories

It's time to populate our TFTP tree 

### 7.1 – The bios directory
We created it in a previous step. This directory will contain the pxelinux files needed to boot from the network.   

```bash
sudo cp /tmp/bios/com32/elflink/ldlinux/ldlinux.c32  /tftp/bios
sudo cp /tmp/bios/com32/libutil/libutil.c32  /tftp/bios  
sudo cp /tmp/bios/com32/menu/menu.c32  /tftp/bios
sudo cp /tmp/bios/com32/menu/vesamenu.c32  /tftp/bios 
sudo cp /tmp/bios/core/pxelinux.0  /tftp/bios
sudo cp /tmp/bios/core/lpxelinux.0  /tftp/bios
```

### 7.2 – The grub directory
Same principle, the grub directory contains the necessary files for UEFI machines.
We will use the signed version of grub.

```bash
sudo cp ~/tmp/grub/usr/lib/grub/x86_64-efi-signed/grubnetx64.efi.signed  /tftp/grub/grubx64.efi
sudo cp ~/tmp/shim/usr/lib/shim/shimx64.efi.signed  /tftp/grub/bootx64.efi
```

**notes:** 
In some cases, no "shimx64.efi.signed" but a "shimx64.efi.signed.latest", that works too!

Finally, we copy 2 more files from the ISO.
```bash
sudo cp /var/www/html/desktop/u2510/boot/grub/grub.cfg  /tftp/grub/
sudo cp /var/www/html/desktop/u2510/boot/grub/font.pf2 /tftp/grub/
```

### 7.3 – The boot directory
During this step, we need to place the correct bootloader so that the installation process can start properly.  We will copy the necessary files from the /var/www/html location.   
Run the following commands to copy the necessary files to the correct location

Note: Make sure the /tftp/boot/casper folder has been created and exists...
```bash
sudo cp /var/www/html/desktop/u2510/casper/vmlinuz      /tftp/boot/casper
sudo cp /var/www/html/desktop/u2510/casper/initrd       /tftp/boot/casper
```

### 7.4 – Symbolic link to the boot directory
Creating a symbolic link to /tftp/boot 
```bash
sudo ln -s /tftp/boot  /tftp/bios/boot
```

---

## 🧭 Step 8 – PXE BIOS and UEFI Menus
These are the most important files in the configuration.  These files tell the target machine where to connect and where the source files needed to perform the network installation are located.  So let's create them...

### 8.1 pxelinux configuration file
We will create a pxelinux.cfg directory under /tftp/bios

```bash
sudo mkdir /tftp/bios/pxelinux.cfg
```
In this directory, we will create an empty 'default' file. 
It is this file which will control the behavior of pxelinux. 

to modify it:

```bash
sudo nano /tftp/bios/pxelinux.cfg/default 
```

### Example pxelinux.cfg (BIOS)

```cfg
DEFAULT menu.c32
MENU TITLE PXE BOOT MENU
PROMPT 0 
TIMEOUT 0

MENU COLOR TABMSG  37;40  #ffffffff #00000000
MENU COLOR TITLE   37;40  #ffffffff #00000000 
MENU COLOR SEL      7     #ffffffff #00000000
MENU COLOR UNSEL    37;40 #ffffffff #00000000
MENU COLOR BORDER   37;40 #ffffffff #00000000

LABEL Ubuntu Desktop 25.10
    kernel /boot/casper/vmlinuz
    append nfsroot=<PXE_IP_ADDRESS>:/var/www/html/desktop/u2510 netboot=nfs ip=dhcp boot=casper initrd=/boot/casper/initrd
```

It will work, but we can make it sexier!

Let's add this package and copy some libraries. 

```bash
sudo apt install syslinux-common
sudo cp *.c32 /tftp/bios
sudo systemctl restart dnsmasq
``` 

New example of /tftp/bios/pxelinux.cfg/default
```cfg
#DEFAULT menu.c32
UI vesamenu.c32
MENU TITLE PXE BOOT MENU
#l'image de fond devra être dans /tftp/bios/
MENU BACKGROUND fond.png
PROMPT 0
#On defini un timer de 10 sec 
TIMEOUT 100
#A l'échéance du TIMER le système démarre sur le disque local 
ONTIMEOUT LOCAL

MENU COLOR TABMSG  37;40  #ffffffff #00000000
MENU COLOR TITLE   37;40  #ffffffff #00000000
MENU COLOR SEL      7     #ffffffff #00000000
MENU COLOR UNSEL    37;40 #ffffffff #00000000
MENU COLOR BORDER   37;40 #ffffffff #00000000

LABEL UBUNTU_DESKTOP_25_10
        MENU LABEL ^1 UBUNTU DESKTOP 25.10
        kernel /boot/casper/vmlinuz
        append nfsroot=1<PXE_IP_ADDRESS>:/var/www/html/desktop/u2510 netboot=nfs ip=dhcp boot=casper initrd=/boot/casper/initrd

LABEL LOCAL
        MENU LABEL ^2 BOOT FROM LOCAL DISK
        MENU DEFAULT
        kernel chain.c32
        append hd0
```

### 8.1 grub config file
Now we also need to create a grub boot menu and make sure the correct option is available and working.  
The grub bootloader reads information from the grub.cfg file.  

To modify it:

```bash
sudo nano /tftp/grub/grub.cfg
```
### Example grub.cfg (UEFI)
```cfg
set timeout=30

loadfont unicode

set menu_color_normal=white/black
set menu_color_highlight=black/light-gray

menuentry "Install Ubuntu 25.10" {
	set gfxpayload=keep
	linux	/boot/casper/vmlinuz ip=dhcp nfsroot=192.168.1.5:/var/www/html/desktop/u2510/ netboot=nfs boot=casper
	initrd	/boot/casper/initrd
}
```

**Slightly sexier version:**
(example on the same principle as previously)

```cfg
set timeout=10 
loadfont unicode 
set default=disk

loadfont /tftp/grub/unicode.pf2

GRUB_TERMINAL="gfxterm"
GRUB_GFXMODE="auto"

set color_normal=yellow/black  
set color_highlight=black/yellow  
set menu_color_normal=black/light-gray  
set menu_color_highlight=yellow/dark-gray

menuentry "UBUNTU DESKTOP 25.10" --id ubuntu { 
	set gfxpayload=keep 
	linux /boot/casper/vmlinuz ip=dhcp nfsroot=192.168.1.5:/var/www/html/desktop/u2510/ netboot=nfs boot=casper 
	initrd /boot/casper/initrd
}

menuentry "BOOT FROM LOCAL DISK" --id disk {
    insmod gzio
    insmod part_gpt
    insmod ext2
    set root='hd0,gpt2'
    linux /boot/vmlinuz-6.17.0-5-generic root=/dev/sda2 ro quiet splash
    initrd /boot/initrd.img-6.17.0-5-generic
}
```

---

## 🧪 Tests and validation

It is strongly recommended to test the PXE infrastructure via a **virtual machine** before any use in production.

### Recommendations for VirtualBox

- UEFI: network adapter **virtio-net**
- Legacy BIOS: **Intel PRO/1000 MT 82540EM**

The PXE menu should appear and allow:
- a manual network installation,
- or automatic startup on disk after the timer expires.

---
End of part I

## 🔜 Next step

👉 **Part II**:
- fully automated installation (autoinstall)
- silent post-installation
- full integration with Rudder

---
 *Living easy, living free.*  
**MSC**

> 💡 Suggestions and feedback are welcome!