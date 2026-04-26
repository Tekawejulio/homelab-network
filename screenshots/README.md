# 🖧 HomeLab Network — Projet d'Administration Système et Réseaux

> Projet réalisé dans le cadre du bachelier en Informatique (option Systèmes et Réseaux)  
> Simulation d'une infrastructure réseau d'entreprise complète sous VMware Workstation

---

## 📋 Table des matières

- [Objectifs du projet](#objectifs-du-projet)
- [Architecture réseau](#architecture-réseau)
- [Stack technique](#stack-technique)
- [Infrastructure virtuelle](#infrastructure-virtuelle)
- [Phase 1 — Environnement VMware](#phase-1--environnement-vmware)
- [Phase 2 — Firewall pfSense](#phase-2--firewall-pfsense)
- [Phase 3 — Serveur Debian DMZ](#phase-3--serveur-debian-dmz)
- [Phase 4 — Windows Server 2022 / Active Directory](#phase-4--windows-server-2022--active-directory)
- [Phase 5 — Monitoring](#phase-5--monitoring)
- [Compétences démontrées](#compétences-démontrées)
- [Auteur](#auteur)

---

## 🎯 Objectifs du projet

Ce projet simule l'infrastructure réseau d'une **petite entreprise** avec :

- Une segmentation réseau en 3 zones (WAN / LAN / DMZ)
- Un firewall central gérant les flux entre les zones
- Un serveur web sécurisé exposé en DMZ
- Un domaine Active Directory pour la gestion des utilisateurs
- Un système de monitoring de l'infrastructure

---

## 🗺️ Architecture réseau

```
                        INTERNET (WAN)
                              │
                    ┌─────────▼─────────┐
                    │   pfSense 2.7.2   │
                    │  Firewall / NAT   │
                    │  VPN / ACL / IDS  │
                    └──┬────────────┬───┘
                       │            │
          ┌────────────▼──┐    ┌────▼────────────┐
          │  DMZ          │    │  LAN interne     │
          │  10.0.2.0/24  │    │  10.0.1.0/24     │
          │               │    │                  │
          │ ┌───────────┐ │    │ ┌──────────────┐ │
          │ │Debian DMZ │ │    │ │Windows Server│ │
          │ │10.0.2.10  │ │    │ │  10.0.1.10   │ │
          │ │Apache2    │ │    │ │  AD DS / DNS │ │
          │ │SSH :2222  │ │    │ │  DHCP / GPO  │ │
          │ │fail2ban   │ │    │ └──────────────┘ │
          │ └───────────┘ │    └──────────────────┘
          └───────────────┘
```

---

## 🛠️ Stack technique

| Composant | Technologie | Version |
|-----------|-------------|---------|
| Hyperviseur | VMware Workstation | 17.5 |
| Firewall | pfSense CE | 2.7.2 |
| Serveur Linux | Debian | 13 (Trixie) |
| Serveur Windows | Windows Server | 2022 Standard |
| Serveur Web | Apache2 | 2.4.x |
| Protection SSH | fail2ban | 1.1.x |
| Annuaire | Active Directory DS | — |

---

## 💻 Infrastructure virtuelle

| VM | OS | IP | Réseau | RAM | Rôle |
|----|----|----|--------|-----|------|
| pfSense-HomeLab | FreeBSD | 10.0.1.1 / 10.0.2.1 | VMnet1+2+8 | 512 MB | Firewall / Routeur |
| Debian-DMZ | Debian 13 | 10.0.2.10 | VMnet2 | 1 GB | Serveur Web / SSH |
| WinServer-LAN | Windows Server 2022 | 10.0.1.10 | VMnet1 | 4 GB | AD DS / DHCP / DNS |

---

## Phase 1 — Environnement VMware

### Réseaux virtuels configurés

| VMnet | Type | Sous-réseau | Rôle |
|-------|------|-------------|------|
| VMnet1 | Host-only | 10.0.1.0/24 | LAN interne |
| VMnet2 | Host-only | 10.0.2.0/24 | DMZ |
| VMnet8 | NAT | 192.168.217.0/24 | WAN (accès internet) |

---

## Phase 2 — Firewall pfSense

### Interfaces configurées

```
WAN  (em0) → 192.168.217.136/24  [DHCP VMware]
LAN  (em1) → 10.0.1.1/24         [Gateway LAN]
DMZ  (em2) → 10.0.2.1/24         [Gateway DMZ]
```

### Règles firewall DMZ

| Priorité | Action | Source | Destination | Description |
|----------|--------|--------|-------------|-------------|
| 1 | ❌ BLOCK | DMZ net | LAN net | Isolation DMZ → LAN |
| 2 | ✅ PASS | DMZ net | Any | Accès internet autorisé |

> **Principe de sécurité** : La DMZ peut accéder à internet mais jamais au LAN interne. Cela protège le réseau interne en cas de compromission du serveur web.

---

## Phase 3 — Serveur Debian DMZ

### Configuration réseau

```bash
# /etc/network/interfaces
auto ens33
iface ens33 inet static
    address 10.0.2.10
    netmask 255.255.255.0
    gateway 10.0.2.1
    dns-nameservers 8.8.8.8
```

### Services installés

#### Apache2 — Serveur Web
```bash
apt install apache2 -y
systemctl enable apache2
systemctl start apache2
# Accessible sur http://10.0.2.10
```

#### SSH durci
```bash
# /etc/ssh/sshd_config
Port 2222               # Port non standard
PermitRootLogin no      # Root désactivé
MaxAuthTries 3          # Max 3 tentatives
PasswordAuthentication yes
```

#### fail2ban — Protection brute force
```bash
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 3600    # Ban 1 heure
findtime = 600    # Fenêtre de 10 minutes
```

#### Utilisateur non-root
```bash
adduser adminlab
usermod -aG sudo adminlab
# Connexion SSH : ssh -p 2222 adminlab@10.0.2.10
```

### Tests de validation

```bash
# Depuis le PC hôte
ping 10.0.2.10              # ✅ Ping DMZ
curl http://10.0.2.10       # ✅ Page Apache
ssh -p 2222 adminlab@10.0.2.10  # ✅ SSH sécurisé
ping 10.0.1.10              # ❌ Bloqué par pfSense (isolation DMZ→LAN)
```

---

## Phase 4 — Windows Server 2022 / Active Directory

### Configuration réseau

```
IP Statique : 10.0.1.10
Masque      : 255.255.255.0
Gateway     : 10.0.1.1
DNS         : 10.0.1.10 (lui-même)
```

### Installation Active Directory

```powershell
# Installation du rôle AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promotion en contrôleur de domaine
Install-ADDSForest `
    -DomainName "homelab.local" `
    -DomainNetbiosName "HOMELAB" `
    -InstallDns:$true `
    -Force:$true
```

### Structure Active Directory

```
homelab.local
└── OU=Utilisateurs_Homelab
    ├── alice.dupont@homelab.local
    └── bob.martin@homelab.local
```

### Création des utilisateurs

```powershell
# Création de l'OU
New-ADOrganizationalUnit -Name "Utilisateurs_Homelab" -Path "DC=homelab,DC=local"

# Création des utilisateurs
New-ADUser -Name "Alice Dupont" `
    -SamAccountName "alice.dupont" `
    -UserPrincipalName "alice.dupont@homelab.local" `
    -Path "OU=Utilisateurs_Homelab,DC=homelab,DC=local" `
    -AccountPassword (ConvertTo-SecureString "User@homelab2026!" -AsPlainText -Force) `
    -Enabled $true

New-ADUser -Name "Bob Martin" `
    -SamAccountName "bob.martin" `
    -UserPrincipalName "bob.martin@homelab.local" `
    -Path "OU=Utilisateurs_Homelab,DC=homelab,DC=local" `
    -AccountPassword (ConvertTo-SecureString "User@homelab2026!" -AsPlainText -Force) `
    -Enabled $true
```

### Vérification

```powershell
Get-ADDomain
Get-ADUser -Filter * | Select Name
```

---

## Phase 5 — Monitoring

> 🚧 En cours d'implémentation — Zabbix sur Debian

---

## ✅ Compétences démontrées

### Réseaux
- Segmentation réseau (WAN / LAN / DMZ)
- Configuration d'un firewall pfSense
- Règles ACL et NAT
- Plan d'adressage IP

### Administration Linux
- Installation et configuration Debian
- Administration Apache2
- Durcissement SSH
- Protection fail2ban
- Gestion des utilisateurs Linux

### Administration Windows
- Installation Windows Server 2022
- Déploiement Active Directory
- Gestion des utilisateurs et OU
- PowerShell Administration

### Sécurité
- Isolation des zones réseau
- Principe du moindre privilège
- Protection contre les attaques brute force
- Désactivation du compte root SSH

---

## 👤 Auteur

**Bachelier en Informatique — Option Systèmes et Réseaux**  
Projet réalisé avec VMware Workstation 17.5  
Année académique 2025-2026

---

*Ce projet simule une infrastructure réelle d'entreprise dans un environnement virtualisé.*
