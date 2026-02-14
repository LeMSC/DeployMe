# 🚀 Déploiement industriel de postes Linux via PXE

> **Provisioning automatisé de postes Linux, de l’allumage à la conformité continue**

---

## 🇫🇷 Présentation (FR)

Ce projet propose une **méthode industrielle de déploiement de postes de travail Linux** reposant sur une infrastructure **PXE**.

L’objectif n’est pas seulement d’installer un système d’exploitation, mais de **livrer un poste immédiatement opérationnel**, intégré à un outil de **gestion de configuration et de conformité (Rudder)**.

### Fonctionnalités clés

- Démarrage réseau PXE (BIOS & UEFI)
- Installation Linux automatique
- Distribution via NFS
- Menus PXE personnalisés (pxelinux / GRUB)
- Intégration post-installation avec **Rudder**

### Expérience utilisateur

```text
1. Le poste démarre en PXE
2. L’utilisateur choisit un profil
3. L’installation se déroule sans intervention
4. Le poste redémarre et est prêt à l’emploi
```

👉 Aucune action utilisateur requise pendant l’installation.

---

## 🧩 Partie I – Mise en place de l’infrastructure PXE

Cette première partie couvre la **mise en place complète de l’infrastructure PXE**, jusqu’à l’affichage d’un menu permettant de lancer une **installation manuelle de l’OS**.

📘 **Documentation complète de la partie I** : voir [README-full_FR.md](https://github.com/LeMSC/DeployMe/blob/main/README-full_FR.md)

La **Partie II** (à venir) traitera de l’industrialisation complète de l’installation (mode automatique et silencieux).

---



---

## 🇬🇧 Overview (EN)

This project provides an **industrial-grade PXE-based deployment workflow for Linux workstations**.

The goal is not only to install an operating system, but to **deliver a fully operational workstation**, automatically configured and enrolled into a **configuration and compliance management system (Rudder)**.

### Key features

- PXE network boot (BIOS & UEFI)
- Automated Linux installation
- NFS content distribution
- Custom PXE menus (pxelinux / GRUB)
- Post-install configuration management with **Rudder**

### User experience

```text
1. Workstation boots via PXE
2. Installation profile is selected
3. Installation runs unattended
4. System reboots ready for use
```

📘 **Full documentation**: see [README-full_US.md](https://github.com/LeMSC/DeployMe/blob/main/README-full_US.md)

---


## 📂 Repository structure

```text
README.md           # Short version (FR / EN)
README-full.md      # Full technical documentation (FR)
README-full.en.md   # Full technical documentation (EN)
```

---

> *Living easy, living free.*  
> **MSC**

