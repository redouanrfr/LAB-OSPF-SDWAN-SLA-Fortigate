# 🌐 LAB OSPF SD-WAN - FortiGate

> Architecture réseau multi-sites avec basculement automatique OSPF et SD-WAN pour haute disponibilité

[![FortiOS](https://img.shields.io/badge/FortiOS-6.0.3-red.svg)](https://www.fortinet.com/)
[![Cisco IOS](https://img.shields.io/badge/Cisco%20IOS-15.2-blue.svg)](https://www.cisco.com/)
[![OSPF](https://img.shields.io/badge/Protocol-OSPFv2-green.svg)](https://datatracker.ietf.org/doc/html/rfc2328)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen.svg)]()

## 📑 Table des matières

- [À propos](#-à-propos)
- [Architecture](#-architecture)
- [Fonctionnalités](#-fonctionnalités)
- [Prérequis](#-prérequis)
- [Tests validés](#-tests-validés)
- [Documentation](#-documentation)
- [Auteur](#-auteur)

## 🎯 À propos

Ce projet implémente une architecture réseau redondante Full-OSPF avec mécanismes SD-WAN sur FortiGate, permettant un basculement automatique et zero-touch lors de pannes de liens WAN ou de pare-feu. La solution assure une haute disponibilité et une continuité de service pour l'utilisateur final via :

Convergence OSPF dynamique : Interconnexion automatique entre Area 0 et Area 1, garantissant la propagation des routes sans configuration manuelle (Zero Touch).

Réactivité SD-WAN : Monitoring par Health-checks avec basculement du flux local en sub-seconde.

Redondance complète : Architecture multi-chemins (R1, R2, R3, R4) avec pivot central sur un ABR (R5).

Monitoring SLA en temps réel : Sélection intelligente du meilleur chemin basée sur la performance réelle des liens (latence, jitter, perte).

Note sur la convergence : Le moteur SD-WAN assure un basculement quasi instantané du trafic sortant local (<1s). En parallèle, OSPF reconverge automatiquement pour 
l'ensemble de la topologie réseau sous ~40 secondes (ip ospf dead-interval), garantissant la cohérence complète des tables de routage sans intervention humaine.


- **Convergence OSPF automatique** entre Area 0 et Area 1
- **Health-checks SD-WAN** avec basculement sub-seconde
- **Redondance complète** des chemins de routage
- **Monitoring SLA** en temps réel

### Version

- **Version** : 1.0
- **Date** : Novembre 2025
- **Plateforme** : FortiGate 6.0.3 + Cisco IOS 15.2

## 🏗️ Architecture

### Topologie globale

```
![](https://raw.githubusercontent.com/redouanrfr/LAB-OSPF-SDWAN-SLA-Fortigate/main/Docs/Schema.png)

```

### Composants

#### 🏢 SITE 1 (Area 0)
- **FortiGate-site1-222** : Firewall/SD-WAN Controller
- **R1, R2** : Routeurs WAN redondants (Cisco)
- **PC3** : Client test (40.40.1.1/24)
- **Metasploitable-1** : Serveur LAN

#### 🌍 BACKBONE OSPF
- **R5** : ABR (Area Border Router) - interconnexion Area 0 ↔ Area 1
- **Liaisons /30** : Interconnexions point-à-point haute performance

#### 🏢 SITE 2 (Area 1)
- **FortiGate-site2-223** : Firewall/SD-WAN Controller
- **R3, R4** : Routeurs WAN redondants (Cisco)
- **PC1, PC4** : Clients utilisateurs (50.50.x.x/24)
- **Kali-1** : Plateforme de test sécurité

## ✨ Fonctionnalités

### OSPF Multi-Area
- ✅ Configuration OSPF hierarchisée (Area 0 + Area 1)
- ✅ R5 configuré comme ABR (Area Border Router)
- ✅ Propagation automatique des routes inter-area via LSA Type 3
- ✅ Router-ID uniques par équipement
- ✅ 0 route statique (full dynamic routing)

### SD-WAN
- ✅ Health-checks ICMP bidirectionnels
- ✅ Basculement automatique en cas de panne
- ✅ Monitoring SLA (latence, jitter, packet loss)
- ✅ Service de basculement configuré sur chaque site
- ✅ Support des membres Virtual-WAN-Link

### Redondance
- ✅ Double lien WAN par site
- ✅ Basculement sub-seconde (SD-WAN)
- ✅ Convergence OSPF < 45 secondes (dead-interval peut être réduit)
- ✅ Perte de paquets minimale (3-5 pings)

## 📋 Prérequis

### Matériel / Logiciels
- **GNS3** (émulation réseau) ou matériel physique
- **FortiGate VM** version 6.0.3-build0200 minimum
- **Cisco IOS** version 15.2 ou supérieure
- **Wireshark** (pour analyse de trafic)

### Connaissances requises
- Configuration OSPF (areas, LSA, ABR)
- Administration FortiGate (CLI & GUI)
- Routage Cisco IOS
- Protocoles de basculement (health-checks, SLA)

> 📖 **Documentation ** disponible dans [`Docs/lab_sdwan_ospf_doc.md`](Docs/lab_sdwan_ospf_doc.md)

## ✅ Tests validés

### Test 1 : Connectivité end-to-end OSPF
- **Objectif** : Vérifier la connectivité Site 1 ↔ Site 2
- **Statut** : ✅ **VALIDÉ**
- **Résultat** : Ping 40.40.1.1 → 50.50.1.1 via R1/R4 (chemins primaires)

### Test 2 : Basculement SD-WAN Site 1 (Panne R1)
- **Objectif** : Tester le basculement automatique lors de la panne du lien principal
- **Statut** : ✅ **VALIDÉ**
- **Métriques** :
  - Détection panne : ~5 secondes
  - Basculement SD-WAN : <1 seconde
  - Convergence OSPF : ~40 secondes
  - Perte de paquets : 3-5 pings

### Test 3 : Basculement SD-WAN Site 2 (Panne R4)
- **Objectif** : Validation du basculement bidirectionnel
- **Statut** : ✅ **VALIDÉ**
- **Métriques** : Similaires au Test 2

### Test 4 : Convergence OSPF multi-area
- **Objectif** : Propagation des routes inter-area via R5 (ABR)
- **Statut** : ✅ **VALIDÉ**
- **Vérification** : LSA Type 3 présents dans LSDB

> 📊 **Détails des tests** : Consultez le section tests validés dans [`Docs/lab_sdwan_ospf_doc.md`](Docs/lab_sdwan_ospf_doc.md)

## 📚 Documentation

### Fichiers principaux
- **[Documentation complète](Docs/lab_sdwan_ospf_doc.md)** : Guide détaillé avec architecture, plan d'adressage, configurations
- **[Schéma réseau](Docs/fortigate-ospf-sdwan.xml)** : Diagramme Draw.io (éditable)
- **[Schéma réseau](Docs/Schema.png)** : Diagramme Png (ScreenShot)


## 🎥 Demos & Preuves de Concept (PoC)

Cette section regroupe les captures vidéo des tests de validation de l'architecture SD-WAN et du routage OSPF.
---
### 🌐 SD-WAN & OSPF avec Fortigate
Cette vidéo démontre l'établissement des tunnels SD-WAN, la convergence du protocole OSPF entre les sites, et la répartition de charge (Load Balancing) selon les SLA définis.
* **Fichier :** `../Demos/fortigate-sd-wan-ospf.mp4`
* **Points clés démontrés :**  Voir "Tests validés"
 
### Références externes
- [FortiOS 6.0.3 Administration Guide](https://docs.fortinet.com/)
- [RFC 2328 - OSPF Version 2](https://datatracker.ietf.org/doc/html/rfc2328)
- [SD-WAN Best Practices - Fortinet](https://docs.fortinet.com/document/fortigate/6.0.0/handbook/39701/sd-wan)

## 📝 Licence
CC BY-NC-ND 4.0
⚠️ Avertissement de sécurité (Cyber-Resilience) :
Ce contenu (configurations OSPF, scripts d'analyse, topologies GNS3) est fourni à des fins éducatives et de recherche uniquement en LAB.
## 🙏 Remerciements
Je tiens à exprimer ma gratitude aux contributeurs et aux communautés qui rendent la recherche en cybersécurité et en réseau accessible.
