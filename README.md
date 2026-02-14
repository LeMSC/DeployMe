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

📘 **Documentation complète** : voir [README-full_FR.md](https://help.semmle.com/codeql/codeql-for-vscode.html)

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

📘 **Full documentation**: see [README-full_US.md](https://help.semmle.com/codeql/codeql-for-vscode.html)

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

