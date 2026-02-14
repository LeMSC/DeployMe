# 🚀 Déploiement industriel de postes de travail Linux via PXE

> **Documentation complète – de l’architecture PXE à l’installation manuelle et automatisée**

---

## 📌 Introduction

Ce document constitue la **version complète et détaillée** du projet de déploiement de postes de travail Linux via **PXE**.

L’objectif est d’**industrialiser l’installation de postes utilisateurs Linux**, en couvrant non seulement l’installation du système d’exploitation, mais **l’ensemble du cycle de mise en service initial** :

- démarrage réseau,
- installation de l’OS,
- configuration du poste,
- intégration à un outil de gestion de configuration et de conformité,
- maintien en conditions opérationnelles.

À l’issue du processus, le poste est :

- ✅ installé et à jour
- ✅ configuré (système et applications)
- ✅ intégré à un dispositif de conformité et de supervision (Rudder)

🎯 L’ambition est de proposer, pour Linux, une approche **fonctionnellement comparable à SCCM / MECM** pour les environnements Windows, tout en restant adaptée à la réalité des postes de travail Linux (diversité matérielle, flexibilité, simplicité).

📎 La procédure décrite fonctionne aussi bien sur **machines physiques** que sur **machines virtuelles**.

---

## 🧑‍💻 Expérience utilisateur – Vue globale

```text
1. Branchement d’un nouveau poste
2. Démarrage réseau (PXE)
   - Attribution d’une adresse IP
   - Affichage du menu PXE
3. Sélection du profil d’installation

4. Installation automatique :
   - Partitionnement du disque
   - Installation du système
   - Installation de l’environnement graphique
   - Installation de packages additionnels et applications. 
   - Patching
   - Installation et enrôlement de l’agent Rudder

5. Redémarrage
6. Poste prêt à l’emploi
7. Configuration et conformité maintenues en continu
```

👉 **Aucune intervention humaine n’est requise pendant la phase d’installation.**

---

## 🧠 Hypothèses et choix d’architecture

Dans la suite de ce document, les hypothèses suivantes sont retenues :

- Les serveurs **PXE** et **DHCP** sont situés sur le même réseau que les postes clients (`192.168.1.0/24`).
- Le serveur **DHCP** et le serveur **PXE** sont deux machines distinctes (choix d’architecture, non contrainte technique).
- L’outil de gestion de configuration et de conformité utilisé est **Rudder**.

📎 Pour une excellente introduction à Rudder :
👉 https://blog.stephane-robert.info/docs/infra-as-code/gestion-de-configuration/rudder/

### 🔄 Variantes d’architecture possibles

**1️⃣ DHCP centralisé et PXE sur VLAN séparé**
- DHCP centralisé
- Serveur PXE dans un autre VLAN
- Relais DHCP via IP Helper

**2️⃣ Réseau de déploiement dédié**
- VLAN ou sous-réseau spécifique au déploiement
- Serveur PXE assurant DHCP + TFTP
- Les postes sont temporairement connectés à ce réseau

---

# 🧩 Partie I – Mise en place de l’infrastructure PXE

Cette première partie couvre la **mise en place complète de l’infrastructure PXE**, jusqu’à l’affichage d’un menu permettant de lancer une **installation manuelle de l’OS**.

La **Partie II** (à venir) traitera de l’industrialisation complète de l’installation (mode automatique et silencieux).

---

## 🛠️ Étape 0 – Installation du serveur Ubuntu 24.04 LTS

### 📋 Prérequis matériels minimaux

- Processeur : 1 GHz ou plus
- Mémoire : 1 Go
- Stockage : 5 Go disponibles

> ℹ️ L’installation d’Ubuntu Server n’est pas détaillée ici.
> De nombreux tutoriels existent sur le sujet.

📎 Exemple : https://www.linux-fra.com/?p=15124

### 📥 Téléchargement de l’ISO

- Ubuntu 24.04 Server LTS  
  https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso

### ⚙️ Paramètres d’installation retenus

- Type : installation par défaut
- Nom du serveur (exemple) : `lxpxe01`
- Utilisateur créé : `user01`

### 🔄 Mise à jour post-installation

```bash
sudo apt update
sudo apt upgrade
```
Et maintenant… on peut commencer à jouer 😄

---

## 🔐 Étape 1 – Installation et configuration de SSH

Cette étape n’est pas strictement obligatoire, mais **fortement recommandée**, notamment pour l’administration à distance d’un serveur physique.

### 1️⃣ Installation du service SSH

```bash
sudo apt install openssh-server
```

### 2️⃣ Vérification de l’état du service

```bash
sudo systemctl status ssh
```

Le service doit apparaître comme **active (running)** et **enabled**.

Si le service SSH d'Ubuntu est bien en cours d'exécution, vous verrez en sortie :
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

Comme le SSH doit aussi être disponible à chaque nouveau démarrage du système, l'entrée **`preset: enabled`** doit également apparaître dans la ligne **`Loaded`**.
Si le service SSH reste inactif et que son lancement automatique après redémarrage n'est pas activé, vous pouvez alors renseigner deux commandes supplémentaires :

Si nécessaire :

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### 3️⃣ Ouverture du port SSH dans le pare-feu
UFW est le programme de configuration d'Ubuntu dédié au pare-feu propriétaire du système.  
Avec ce programme, configurez une règle appropriée pour la communication depuis le SSH, afin d'ouvrir le port aux données entrantes comme sortantes :
```bash
sudo ufw allow ssh
```

### 4️⃣ Configuration avancée (optionnelle)

Le fichier de configuration principal est : **/etc/ssh/sshd_config**.  
Si vous souhaitez le modifier, ouvrez-le depuis l'éditeur de texte de votre choix (ici, il s'agit de nano) avec cette commande :  

```bash
sudo nano /etc/ssh/sshd_config
```

Vous pouvez notamment :
- changer le port SSH
- désactiver la redirection TCP
- etc.

Après modification, un petit redémarrage de SSH est nécessaire.

```bash
sudo service ssh restart
```

### 5️⃣ Test de connexion

```bash
ssh user01@lxpxe01
```

📌 **Une dernière chose**. 
Le serveur PXE doit disposer d’une **adresse IP statique** (réservation DHCP recommandée).
Pour cette procédure, j'ai simplement reservé une IP pour ce serveur sur le DHCP puis renouvellé le bail.

```bash
sudo dhclient -r
sudo dhclient
```
Si les outils dhcp client ne sont pas installés, il faudra précéder ces deux commandes de:  

```bash
sudo apt install isc-dhcp-client
```

🎉 **Nous voilà prêt pour la suite !**

---

## 🛠️ Étape 2 – Installation des services nécessaires au PXE

Le PXE repose sur les composants suivants :

- **dnsmasq** : TFTP / PXE (et éventuellement DHCP)
- **Apache2** : serveur de stockage
- **NFS** : Les paquets NFS fournissent des fonctionnalités de services de fichiers.
- **syslinux / grub** : bootloaders BIOS et UEFI

Le setup repose sur dnsmasq, petit serveur léger qui va être notre TFTP, apache2, NFS et syslinux pour les bootloaders.  
Pour information, dnsmasq (https://fr.wikipedia.org/wiki/Dnsmasq) peut aussi être utilisé comme serveur DHCP et DNS.
Rappelez-vous notre DCHP existe déjà, vous devrez configurer votre DHCP externe pour pointer vers ce serveur PXE. 
(Ou utiliser dnsmasq comme DHCP, on y reviendra :))

> ℹ️ Configurez votre DHCP existant : ajoutez dans la subnet du VLAN bootloader: "pxelinux.0"; next-server <IP_du_serveur_PXE>; et redémarrez-le.

### Installation des paquets

```bash
sudo apt-get install apache2
sudo apt-get install nfs-kernel-server 
sudo apt-get install dnsmasq
```

### Download des packages pxelinux
```bash
cd ~/tmp/
wget https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/syslinux-6.03.zip
unzip syslinux-6.03.zip
```

### Download des packages UEFI
```bash
cd ~/tmp/
apt-get download shim.signed
dpkg -x <%nom du package%> shim

apt-get download grub-efi-amd64-signed
dpkg -x <%nom du package%> grub
```

---

## 📁 Étape 3 – Création des arborescences TFTP et WEB

### 3.1 Arborescence TFTP

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

### 3.2 Arborescence WEB (Apache)
Comme nous utilisons le serveur Web Apache, nous allons copier tous les fichiers sources dans le répertoire `/var/www/html`.  
Nous allons copier le contenu des fichiers iso d'Ubuntu 25.10 Desktop à cet emplacement.

Notre structure ressemblera à la représentation suivante.   
Vous pouvez bien sûr créer votre propre structure.


```text
/var/www/html
 └─ desktop
     └─ u2510
```

```bash
sudo mkdir -p /var/www/html/desktop/u2510
```

---

## 📦 Étape 4 – Peuplement du répertoire WEB

### Montage de l’ISO
L'OS que nous voulons déployer dans notre exemple est Ubuntu Desktop 25.10, il va falloir le télécharger et monter l'ISO.  
Ca va ressembler à ça:  

```bash
cd ~/Téléchargements
wget -c "https://releases.ubuntu.com/25.10/ubuntu-25.10-desktop-amd64.iso"

sudo mount ~/Téléchargements/ubuntu-25.10-desktop-amd64.iso /media
```

### Copie des fichiers

```bash
sudo cp -rf /media/* /var/www/html/desktop/u2510
sudo cp -rf /media/.disk /var/www/html/desktop/u2510
```
La 1ère commande ci-dessus copie tout le contenu de l'iso source, à l'exception d'un dossier caché nécessaire au bon fonctionnement du processus pxe.  
Vous devez donc exécuter une commande supplémentaire afin de vous assurer que tous les fichiers dont vous avez besoin ont été copiés correctement. (seconde ligne de commande ci-dessus)

### Démontage de l’ISO

```bash
sudo umount /media
```

---

## 📡 Étape 5 – Configuration du serveur NFS
Notre structure de dossiers étant prête, nous pouvons commencer à configurer les différents services utilisés par le serveur PXE.   Pour nous assurer que notre structure de répertoires est accessible via le réseau et via le protocole nfs, nous devons modifier le fichier suivant en exécutant la commande suivante.

Éditer le fichier `/etc/exports` :

```bash
sudo nano /etc/exports
```

Insérez à la fin du fichier le chemin d'accès où vous avez stocké vos fichiers d'installation, le sous-réseau qui peut y accéder et le type de droit que vous souhaitez accorder.  Dans notre  scénario, nous voulons accorder l'accès au répertoire suivant /var/www/html/desktop  via le sous-réseau 192.168.1.0/24 et nous accordons un accès en lecture seule (ro).  
Ainsi, à la fin du fichier, nous ajouterions la ligne suivante 

```text
/var/www/html/desktop 192.168.1.0/24(ro)
```

Redémarrer le service :

```bash
sudo systemctl restart nfs-kernel-server
```

---

## 🧩 Étape 6 – Configuration de dnsmasq

### Configuration minimale (PXE uniquement)

Nous devons maintenant configurer le service dnsmasq qui assurera la liaison entre les différents services.  
Le fichier de configuration dnsmasq sera utilisé pour fournir les informations nécessaires au client pxe lors de son démarrage.  
Ce fichier indiquera où rechercher le chargeur d'amorçage pxe en fonction de l'architecture du client (uefi ou bios).  
Modifions donc le fichier /etc/dnsmasq.conf et ajoutons les informations suivantes à la fin

Pour modifier le fichier de configuration, exécutez la commande suivante  

```bash
sudo nano /etc/dnsmasq.conf 
```
Le contenu du fichier doit ressembler à ça:
```cfg
enable-tftp
tftp-root=/tftp
```

### Exemple avec DHCP intégré (optionnel)
Comme dit précédemment dans notre architecture, nous disposons déjà d'un serveur DCHP, mais si vous voulez utiliser dnsmasq pour gérer les deux rôles, c'est possible!
Dans ce cas le fichier pourrait ressembler plutôt à ça:
(C'est un exemple)

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

**On reprend,**

redemarrage du service :

```bash
sudo systemctl restart dnsmasq
```

Pour vérifier que dnsmasq démarre correctement et sans erreurs:

```bash
sudo systemctl status dnsmasq
```


##  Étape 7 – Contenu des répertoires TFTP & Boot folders

Il est temps d'alimenter notre arboresnce TFTP 

### 7.1 – Le repertoire bios
Nous l'avons créé à une étape précedente. Ce repertoire contiendra les fichiers pxelinux nécessaire pour démarrer depuis le réseau.   

```bash
sudo cp /tmp/bios/com32/elflink/ldlinux/ldlinux.c32  /tftp/bios
sudo cp /tmp/bios/com32/libutil/libutil.c32  /tftp/bios  
sudo cp /tmp/bios/com32/menu/menu.c32  /tftp/bios
sudo cp /tmp/bios/com32/menu/vesamenu.c32  /tftp/bios 
sudo cp /tmp/bios/core/pxelinux.0  /tftp/bios
sudo cp /tmp/bios/core/lpxelinux.0  /tftp/bios
```

### 7.2 – Le repertoire grub
Même principe, le repertoire grub contient les fichiers necessaires pour les machines UEFI.
Nous utiliserons la version signée de grub.

```bash
sudo cp ~/tmp/grub/usr/lib/grub/x86_64-efi-signed/grubnetx64.efi.signed  /tftp/grub/grubx64.efi
sudo cp ~/tmp/shim/usr/lib/shim/shimx64.efi.signed  /tftp/grub/bootx64.efi
```

**remarques :** 
Dans certains cas, pas de "shimx64.efi.signed" mais un "shimx64.efi.signed.latest", ça fonctionne aussi !

Pour finir, on copie 2 fichiers de plus depuis l'iso.
```bash
sudo cp /var/www/html/desktop/u2510/boot/grub/grub.cfg  /tftp/grub/
sudo cp /var/www/html/desktop/u2510/boot/grub/font.pf2 /tftp/grub/
```

### 7.3 – Le repertoire boot
Au cours de cette étape, nous devons placer le chargeur d'amorçage approprié afin que le processus d'installation puisse démarrer correctement.  Nous allons copier les fichiers nécessaires à partir de l'emplacement /var/www/html.   
Exécutez les commandes suivantes pour copier les fichiers nécessaires à l'emplacement approprié

Remarque : assurez-vous que le dossier /tftp/boot/casper a été créé et existe...
```bash
sudo cp /var/www/html/desktop/u2510/casper/vmlinuz      /tftp/boot/casper
sudo cp /var/www/html/desktop/u2510/casper/initrd       /tftp/boot/casper
```

### 7.4 – Lien symbolique vers le répertoire de boot
Création d'un lien symbolique vers /tftp/boot 
```bash
sudo ln -s /tftp/boot  /tftp/bios/boot
```

---

## 🧭 Étape 8 – Menus PXE BIOS et UEFI
Ce sont les fichiers les plus importants de la configuration.  Ces fichiers indiquent à la machine cible où se connecter et où se trouvent les fichiers source nécessaires pour effectuer l'installation réseau.  Créons-les donc...

### 8.1 fichier de configuration pxelinux
Nous allond créer un repertoire pxelinux.cfg sous /tftp/bios

```bash
sudo mkdir /tftp/bios/pxelinux.cfg
```
Dans ce repertoire, nous allons créer un fichier vide 'default'. 
C'est ce fichier qui va controler le comportement de pxelinux. 

pour le modifier:

```bash
sudo nano /tftp/bios/pxelinux.cfg/default 
```

### Exemple pxelinux.cfg (BIOS)

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

Ca va fonctionner, mais on peut faire plus sexy !

Déjà ajoutons ce package et copions quelques librairies. 

```bash
sudo apt install syslinux-common
sudo cp *.c32 /tftp/bios
sudo systemctl restart dnsmasq
``` 

Nouvel exemple de /tftp/bios/pxelinux.cfg/default
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

### 8.1 fichier de configuration grub
Maintenant, nous devons également créer un menu de démarrage grub et nous assurer que l'option appropriée est disponible et fonctionne.  
Le chargeur d'amorçage grub lit les informations contenues dans le fichier grub.cfg.  

Pour le modifier:

```bash
sudo nano /tftp/grub/grub.cfg
```
### Exemple grub.cfg (UEFI)
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

**Version un peu plus sexy:**
(exemple sur le même principe que précédemment)

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

## 🧪 Tests et validation

Il est fortement recommandé de tester l’infrastructure PXE via une **machine virtuelle** avant toute utilisation en production.

### Recommandations pour VirtualBox

- UEFI : adaptateur réseau **virtio-net**
- BIOS legacy : **Intel PRO/1000 MT 82540EM**

Le menu PXE doit apparaître et permettre :
- une installation réseau manuelle,
- ou un démarrage automatique sur disque après expiration du timer.

---
Fin de la partie I

## 🔜 Prochaine étape

👉 **Partie II** :
- installation totalement automatisée (autoinstall)
- post-installation silencieuse
- intégration complète avec Rudder

---
 *Living easy, living free.*  
**MSC**

> 💡 Suggestions et retours d’expérience sont les bienvenus !