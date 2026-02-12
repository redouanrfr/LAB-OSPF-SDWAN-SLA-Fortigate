# 📋 PROJET LAB OSPF SD-WAN/SLA - Fortigate
## Architecture OSPF Multi-Sites avec Basculement Automatique

**Version:** 1.0  
**Date:** Novembre 2025  
**Plateforme:** FortiGate 6.0.3 + Cisco IOS 15.2

---

## 📑 TABLE DES MATIÈRES

1. [Architecture Globale](#-architecture-globale)
2. [Plan d'Adressage](#-plan-dadressage-complet)
3. [Mécanismes de Fonctionnement](#-mécanismes-de-fonctionnement)
4. [Scénarios de Test Validés](#-scénarios-de-test-validés)
5. [Performances et Métriques](#-performances-et-métriques)
6. [Commandes de Diagnostic](#-commandes-de-diagnostic)

---

## 🏗️ ARCHITECTURE GLOBALE

### Vue d'ensemble

Mise en œuvre d'une architecture réseau full-OSPF avec redondance SD-WAN, assurant la haute disponibilité via un basculement automatique simultané du forwarding du trafic et de la topologie de routage. Solution autonome de reconvergence durant les pannes.

### Composants principaux

#### SITE 1 (Area 0)
- **FortiGate-site1-222** : Firewall/SD-WAN Controller
- **R1, R2** : Routeurs WAN (liens redondants)
- **PC3** : Poste client test (40.40.1.1/24)
- **Metasploitable-1** : Serveur LAN

#### BACKBONE OSPF
- **R5** : ABR (Area Border Router) interconnectant Area 0 et Area 1
- **Liaison /30** : Interconnexions point-à-point

#### SITE 2 (Area 1)
- **FortiGate-site2-223** : Firewall/SD-WAN Controller
- **R3, R4** : Routeurs WAN (liens redondants)
- **PC1** : Client utilisateur (50.50.1.1/24)
- **PC4** : Serveur applicatif (50.50.2.1/24)
- **Kali-1** : Plateforme de test sécurité

### 🏗️ Topologie de redondance

```

                    ┌─────────────────────────┐
                    │    BACKBONE OSPF        │
                    │         R5-ABR          │
                    │   (5.5.5.5)             │
                    └─────────────────────────┘
                      ║               ║
        ┌─────────────╬───────┐   ┌──╬──────────────┐
        ║             ║       ║   ║  ║              ║
    ┌───▼───┐     ┌───▼───┐   │   │ ┌▼──────┐   ┌───▼───┐
    │  R1   │     │  R2   │   │   │ │ R3    │   │  R4   │
    │1.1.1.1│     │2.2.2.2│   │   │ │3.3.3.3│   │4.4.4.4│
    └───┬───┘     └───┬───┘   │   │ └┬──────┘   └───┬───┘
        ║             ║       ║   ║  ║              ║
        ╚═════════════╬═══════╝   ╚══╬══════════════╝
                  ┌───▼────┐      ┌──▼─────┐
                  │   FG1  │      │  FG2   │
                  │ Site1  │      │ Site2  │
                  └────┬───┘      └───┬────┘
                       ║              ║
                   ┌───▼───┐      ┌───▼───┐
                   │  PC3  │      │PC1/PC4│
                   └───────┘      └───────┘

Legend: ═══ Primary Link  ─── Backup Link
```

### 🛣️ Flux de données

**Mode normal (tous liens UP) :**
- Site 1 → R1 (prioritaire) → R5 → R4 (prioritaire) → Site 2
- Retour : Site 2 → R4 → R5 → R1 → Site 1

**Mode dégradé (panne R1) :**
- Site 1 → R2 (backup activé) → R5 → R4 → Site 2

---

## 🌐 PLAN D'ADRESSAGE COMPLET

### 🌿 1 - Area 0

| Équipement        | Interface | Adresse IP/Masque | Gateway       | Rôle/Description   |
|-------------------|-----------|-------------------|---------------|--------------------|
| **FortiGate-222** | port1     | 192.168.2.222/24  | -             | Management/Admin   |
|                   | port2     | 10.10.1.1/24      | 10.10.1.254   | WAN-1 Primary (R1) |
|                   | port3     | 10.10.2.1/24      | 10.10.2.254   | WAN-2 Backup (R2)  |
|                   | port4     | 40.40.1.254/24    | -             | LAN Utilisateurs   |
|                   | port5     | 40.40.2.254/24    | -             | LAN Serveurs       |
| **R1**            | Gi1/0     | 10.10.1.254/24    | -             | Interface vers FG  |
|                   | Gi2/0     | 30.30.1.1/30      | -             | Backbone vers R5   |
| **R2**            | Gi1/0     | 10.10.2.254/24    | -             | Interface vers FG  |
|                   | Gi2/0     | 30.30.1.5/30      | -             | Backbone vers R5   |
| **PC3**           | e0        | 40.40.1.1/24      | 40.40.1.254   | Poste client test  |
|**Metasploitable** | e0        | 40.40.1.254/24    | -             | Serveur LAN        |

### ⚙️BACKBONE OSPF (Interconnexions /30)

| Segment        | Interface | Équipement   | Adresse IP    | Area OSPF |
|----------------|-----------|--------------|---------------|-----------|
| **Link R1-R5** | Gi2/0     | R1           | 30.30.1.1/30  | 0.0.0.0   |
|                | Gi1/0     | R5           30.30.1.2/30    | 0.0.0.0   |
| **Link R2-R5** | Gi2/0     | R2           | 30.30.1.5/30  | 0.0.0.0   |
|                | Gi2/0     | R5           | 30.30.1.6/30  | 0.0.0.0   |
| **Link R5-R3** | Gi3/0     | R5           | 30.30.1.10/30 | 0.0.0.1   |
|                | Gi2/0     | R3           | 30.30.1.9/30  | 0.0.0.1   |
| **Link R5-R4** | Gi4/0     | R5           | 30.30.1.14/30 | 0.0.0.1   |
|                | Gi2/0     | R4           | 30.30.1.13/30 | 0.0.0.1   |

### 🌿 SITE 2 - Area 1

| Équipement    | Interface | Adresse IP/Masque | Gateway     | Rôle/Description     |
|---------------|-----------|-------------------|-------------|----------------------|
|FortiGate-223**| port1     | 192.168.2.223/24  | -           | Management/Admin     |
|               | port2     | 20.20.1.1/24      | 20.20.1.254 | WAN-1 Backup (R3)    |
|               | port3     | 20.20.2.1/24      | 20.20.2.254 | WAN-2 Primary (R4)   |
|               | port4     | 50.50.1.254/24    | -           | LAN Utilisateurs     |
|               | port5     | 50.50.2.254/24    | -           | LAN Serveurs         |
| **R3**        | Gi1/0     | 20.20.1.254/24    | -           | Interface vers FG    |
|               | Gi2/0     | 30.30.1.9/30      | -           | Backbone vers R5     |
| **R4**        | Gi1/0     | 20.20.2.254/24    | -           | Interface vers FG    |
|               | Gi2/0     | 30.30.1.13/30     | -           | Backbone vers R5     |
|               | Gi3/0     | 60.60.1.254/24    | -           | Segment Kali         |
| **PC1**       | e0        | 50.50.1.1/24      | 50.50.1.254 | Poste utilisateur    |
| **PC4**       | e0        | 50.50.2.1/24      | 50.50.2.254 | Serveur applicatif   |
| **Kali-1**    | e0        | 60.60.1.1/24      | 60.60.1.254 | Pentesting platform  |

### Router-ID et Areas OSPF

| Équipement    | Router-ID   | Area(s) OSPF            | Type            |
|---------------|-------------|-------------------------|-----------------|
| R1            | 1.1.1.1     | 0.0.0.0                 | Internal Router |
| R2            | 2.2.2.2     | 0.0.0.0                 | Internal Router |
| R3            | 3.3.3.3     | 0.0.0.1                 | Internal Router |
| R4            | 4.4.4.4     | 0.0.0.1                 | Internal Router |
| **R5**        | **5.5.5.5** | **0.0.0.0 + 0.0.0.1**   | **ABR**         |
| FG-Site1      | 10.10.10.10 | 0.0.0.0                 | Internal Router |
| FG-Site2      | 20.20.20.20 | 0.0.0.1                 | Internal Router |

### Résumé des plages IP

| Plage réseau          | Utilisation               | Site      |
|-----------------------|---------------------------|-----------|
| 10.10.1.0/24          | WAN Site1-R1              | Site 1    |
| 10.10.2.0/24          | WAN Site1-R2              | Site 1    |
| 20.20.1.0/24          | WAN Site2-R3              | Site 2    |
| 20.20.2.0/24          | WAN Site2-R4              | Site 2    |
| 30.30.1.0/30 à .12/30 | Interconnexions Backbone  | OSPF Core |
| 40.40.1.0/24          | LAN Utilisateurs Site1    | Site 1    |
| 40.40.2.0/24          | LAN Serveurs Site1        | Site 1    |
| 50.50.1.0/24          | LAN Utilisateurs Site2    | Site 2    |
| 50.50.2.0/24          | LAN Serveurs Site2        | Site 2    |
| 60.60.1.0/24          | Segment Kali              | Site 2    |
| 192.168.2.0/24        | Management                | Global    |

---

## 🔄 MÉCANISMES DE FONCTIONNEMENT

### 1. Configuration OSPF

#### Structure des Areas
- **Area 0 (Backbone)** : Site 1 + Liens R1/R2 vers R5
- **Area 1 (Stub Area)** : Site 2 + Liens R3/R4 vers R5
- **R5 (ABR)** : Routeur frontière assurant la communication inter-area

#### Réseaux annoncés - FortiGate Site 1

```bash
config router ospf
    set router-id 10.10.10.10
    config area
        edit 0.0.0.0 # Area 0
        next
    end
   ...
```

#### Réseaux annoncés - FortiGate Site 2

```bash
config router ospf
    set router-id 20.20.20.20
    config area
        edit 0.0.0.1  # Area 1
        next
    end
 ...
```

#### Routes statiques (redistribution)

0 route static sur tout le lab.

### 2. Configuration SD-WAN

#### Membres Virtual-WAN-Link

**FortiGate Site 1 :**
```bash
config system virtual-wan-link
    set status enable
    config members
        edit 1
            set interface "port2"         # WAN1
            set gateway 10.10.1.254       # R1
        next
        edit 2
            set interface "port3"         # WAN2
            set gateway 10.10.2.254       # R2
        next
    end
end
```

**FortiGate Site 2 :**
```bash
config system virtual-wan-link
    set status enable
    config members
        edit 1
            set interface "port2"         # WAN1
            set gateway 20.20.1.254       # R3
        next
        edit 2
            set interface "port3"         # WAN2
            set gateway 20.20.2.254       # R4
        next
    end
end
```

### 3. Health-Check SLA (Fortigate)

#### Configuration Site 1
```bash

    config health-check
        edit "sla_wan"
            set server "50.50.1.1"
            set members 1 2
            config sla
                edit 1
                    set latency-threshold 50
                    set jitter-threshold 25
                    set packetloss-threshold 5
                next
            end
        next
    end
 ```

#### Configuration Site 2
```bash
    config health-check
        edit "sla_wan"
            set server "40.40.1.1"
            set members 1 2
            config sla
                edit 1
                    set latency-threshold 50
                    set jitter-threshold 25
                    set packetloss-threshold 5
                next
            end
        next
    end
```

### 4. Règles de Service SD-WAN

#### Site 1 - Priorité sur R1
```bash
config system virtual-wan-link
    config service
        edit 1
            set name "bascule"
            set mode priority
            set dst "all"
            set src "all"
            set health-check "sla_wan"
            set priority-members 1 2    # R1 prioritaire, R2 backup
        next
    end
end
```

#### Site 2 - Priorité sur R4
```bash
config system virtual-wan-link
    config service
        edit 1
            set name "bascule"
            set mode priority
            set dst "all"
            set src "all"
            set health-check "sla_wan"
            set priority-members 2 1    # R4 prioritaire, R3 backup
        next
    end
end
```

### 5. 🛟 Processus de basculement automatique

#### Séquence de détection de panne

```
1. Health-check ICMP échoue (3-5 probes)
      ↓
2. Member SD-WAN marqué DOWN
      ↓
3. OSPF Adjacency TIMEOUT (Dead Interval)
      ↓
4. Retrait des routes OSPF du membre défaillant
      ↓
5. Activation membre backup SD-WAN
      ↓
6. Réinjection des routes via membre backup
      ↓
7. Convergence OSPF (SPF calculation)
      ↓
8. Trafic bascule automatiquement
```

#### Timers OSPF critiques
- **Hello Interval** : 10 secondes
- **Dead Interval** : 40 secondes (4 × Hello, Réglable)
- **SPF Calculation** : Instantané après changement LSA

#### Timeline typique de basculement

| Événement                   | Temps cumulé | Description             |
|-----------------------------|--------------|-------------------------|
| Panne physique du lien      | T0           | Coupure R1 par exemple  |
| Health-check détecte panne  | T0 + 3-5s    | 3 ICMP probes ratés     |
| SD-WAN marque membre DOWN   | T0 + 5s      | Member 1 inactive       |
| OSPF Adjacency DOWN         | T0 + 40s     | Dead interval expiré    |
| Routes retirées de la table | T0 + 41s     | Flush OSPF routes       |
| Activation membre backup    | T0 + 41s     | Member 2 devient actif  |
| Convergence OSPF            | T0 + 45-50s  | Nouveau calcul SPF      |
| **Trafic rétabli**          | **T0 + 50s** | **Basculement complet** |

**Note :** Le SD-WAN peut basculer en <5s, mais attend la convergence OSPF pour stabilité.

### 6. 🧱 Politique de Firewall (Exemple)

Minimaliste, car le but est de tester la haute disponibilité.

#### Site 1 → Site 2
```bash
config firewall policy
    edit 1
        set name "lan-to-wan"
        set srcintf "port4"                # LAN Site1
        set dstintf "virtual-wan-link"     # SD-WAN
        set srcaddr "network-lan"          # 40.40.1.0/24
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "PING"
    next
end
```

---

## 🧪 SCÉNARIOS DE TEST VALIDÉS

### Test 1 : 🔍 Vérification initiale de la topologie

#### Procédure
```bash
# Sur R5 (ABR)
R5# show ip ospf neighbor
R5# show cdp neighbor

# Sur R1
R1# show cdp neighbor
R1# show ip ospf neighbor

# Sur FortiGate Site1
FG1# get router info ospf neighbor
FG1# get router info ospf database brief

# Sur FortiGate Site2
FG2# get router info ospf neighbor
FG2# get router info ospf database brief
```

#### Résultats attendus
- **R5** : 4 voisins OSPF (R1, R2, R3, R4) + 2 FortiGates = 6 total
- **R1** : 2 voisins (R5 + FG-Site1) mais 1 seul CDP (R5 car FG n'est pas de la famille Cisco 🦹)
- **FG-Site1** : 2 voisins OSPF (1.1.1.1 et 2.2.2.2)
- **FG-Site2** : 2 voisins OSPF (3.3.3.3 et 4.4.4.4)
- **LSDB** : Contient les LSA Type 1, 2, 3 pour toutes les routes

✅ **Statut** : **VALIDÉ** - Tous les voisinages établis correctement

---

### Test 2 : 🛟 Basculement SD-WAN Site 1 (Panne R1)

#### Conditions initiales
- **Source** : PC3 (40.40.1.1)
- **Destination** : PC1 (50.50.1.1)
- **Lien actif** : R1 (port2 - Member 1 prioritaire)

#### Procédure
1. **Lancer ping continu** : `PC3# ping 50.50.1.1 -t`
2. **Capturer trafic** : Wireshark sur interface R1-Gi1/0
3. **Vérifier config SD-WAN** : 
   ```bash
   FG1# diagnose sys virtual-wan-link health-check
   ```
4. **Simuler panne** : Shutdown R1 (timestamp T0 = 7:46)
5. **Observer basculement** : Monitoring GUI FortiGate
6. **Vérifier nouveau chemin** : Wireshark sur R2-Gi1/0
7. **Restaurer R1** : No shutdown après 2 minutes
8. **Confirmer retour** : Wireshark + GUI Monitoring

#### Résultats observés

| Phase              | Timestamp   | Observation              | Outil               |
|--------------------|-------------|--------------------------|---------------------|
| Trafic normal      | 7:46:00     | Paquets ICMP via R1      | Wireshark R1        |
| **Panne R1**       | **7:46:15** | **Routeur R1 DOWN**      | **Lab**             |
| Health-check fail  | 7:46:20     | 5 probes ICMP échoués    | FG1 Logs            |
| SD-WAN bascule     | 7:46:20     | Member 2 (R2) actif      | FG1 GUI             |
| OSPF convergence   | 7:46:55     | Routes via R2 installées | FG1 routing table   |
| **Trafic rétabli** | **8:22:00** | **Paquets ICMP via R2**  | **Wireshark R2**    |
| Coupure visible    | 8:22:15     | Link R1 marqué DOWN      | FG1 Monitoring      |
| Restauration R1    | 8:24:00     | R1 remis en service      | Lab                 |
| Retour à normal    | 8:24:45     | Trafic rebascule sur R1  | Wireshark R1        |

#### Temps de basculement
- **Détection panne** : ~5 secondes
- **Basculement SD-WAN** : < 1 seconde
- **Convergence OSPF totale** : ~40 secondes
- **Perte de paquets** : 3-5 pings (interruption totale : ~5 secondes grâce au SD-WAN)

✅ **Statut** : **VALIDÉ** - Basculement automatique fonctionnel

---

### Test 3 : 🛟 Basculement SD-WAN Site 2 (Panne R4)

#### Conditions initiales
- **Source** : PC1 (50.50.1.1)
- **Destination** : PC3 (40.40.1.1)
- **Lien actif** : R4 (port3 - Member 2 prioritaire)
- **Lien backup** : R3 (port2 - Member 1)

#### Procédure
1. **Vérifier config SD-WAN Site2** :
   ```bash
   FG2# diagnose sys virtual-wan-link health-check
   FG2# diagnose sys virtual-wan-link service
   ```
2. **Lancer ping** : `PC1# ping 40.40.1.1 -t`
3. **Capturer R3** : Wireshark sur R3-Gi1/0 (aucun paquet attendu)
4. **Capturer R4** : Wireshark sur R4-Gi1/0 (trafic actif)
5. **Simuler panne** : Shutdown R4 (timestamp T0 = 16:26)
6. **Observer basculement** : GUI Monitoring FG-Site2
7. **Vérifier R3** : Capture Wireshark (trafic doit apparaître)
8. **Restaurer R4** : No shutdown après 2 minutes
9. **Confirmer retour** : Trafic doit revenir sur R4

#### 🎯 Résultats observés

| Phase              | Timestamp    | Observation               | Outil             |
|--------------------|--------------|---------------------------|-------------------|
| Trafic normal      | 16:26:00     | Paquets ICMP via R4       | Wireshark R4      |
| Vérif R3 (avant)   | 16:26:05     | Aucun paquet ICMP         | Wireshark R3      |
| **Panne R4**       | **16:26:20** | **Routeur R4 DOWN**       | **Lab**           |
| Health-check fail  | 16:26:25     | Member 2 détecté DOWN     | FG2 Logs          |
| **Basculement**    | **16:56:30** | **Member 1 (R3) activé**  | **FG2 GUI**       |
| Trafic via R3      | 16:56:35     | Paquets ICMP via R3       | Wireshark R3      |
| R4 sans trafic     | 16:56:40     | Aucun paquet ICMP         | Wireshark R4      |
| Link DOWN visible  | 16:57:00     | Interface R4 rouge        | FG2 Monitoring    |
| Restauration R4    | 16:58:00     | R4 remis en ligne         | Lab               |
| OSPF Adjacency UP  | 16:58:45     | Voisinage rétabli         | FG2 OSPF          |
| Retour prioritaire | 16:59:00     | Trafic rebascule R4       | Wireshark R4      |
| Confirmation       | 16:59:15     | Link R4 vert              | FG2 Monitoring    |

#### Temps de basculement
- **Détection** : ~5 secondes
- **Basculement** : Immédiat (<1s après détection)
- **Convergence** : ~40-50 secondes
- **Perte de connectivité** : ~5 secondes

✅ **Statut** : **VALIDÉ** - Basculement bidirectionnel fonctionnel

---

### Test 4 : 🧩 Convergence OSPF multi-area

#### Objectif
Vérifier la propagation des routes entre Area 0 et Area 1 via l'ABR R5.

#### Procédure
```bash
# Sur FG-Site1 (Area 0)
FG1# get router info routing-table all

# Sur FG-Site2 (Area 1)
FG2# get router info routing-table all

# Sur R5 (ABR)
R5# show ip ospf database
R5# show ip route ospf

# Vérifier LSA Type 3 (Summary LSA)
R5# show ip ospf database summary
```

#### 📉 Résultats attendus
- FG-Site1 reçoit les routes 50.50.x.0/24 en tant que routes inter-area (LSA Type 3)
- FG-Site2 reçoit les routes 40.40.x.0/24 en tant que routes inter-area
- R5 maintient des routes pour toutes les subnets des deux sites
- LSA Type 3 présents dans la LSDB pour les routes inter-area

✅ **Statut** : **VALIDÉ** - Propagation inter-area fonctionnelle

---

## 🔧 COMMANDES DE DIAGNOSTIC

### Vérification OSPF

#### Routeurs Cisco (R1-R5)
```bash
# Voisinage OSPF
show ip ospf neighbor

# Table de routage OSPF
show ip route ospf

# Détail des interfaces OSPF
show ip ospf interface brief

# Base de données OSPF
show ip ospf database
show ip ospf database router
show ip ospf database summary

# Statistiques OSPF
show ip ospf statistics

# Debug OSPF (à utiliser avec précaution)
debug ip ospf adj
debug ip ospf events
```

#### FortiGate (FG1 et FG2)
```bash
# Voisinage OSPF
get router info ospf neighbor

# Table de routage OSPF
get router info routing-table ospf
get router info routing-table all

# Base de données OSPF
get router info ospf database brief
get router info ospf database router
get router info ospf database summary

# Détail des interfaces OSPF
get router info ospf interface

# Status OSPF général
get router info ospf status
```

### Vérification SD-WAN

#### FortiGate - Health Check
```bash
# Status des health-checks
diagnose sys virtual-wan-link health-check

# Détail d'un health-check spécifique
diagnose sys virtual-wan-link health-check sla_wan

# Historique des probes
diagnose sys virtual-wan-link log
```

#### FortiGate - Membres et Services
```bash
# Status des membres SD-WAN
diagnose sys virtual-wan-link member

# Services SD-WAN configurés
diagnose sys virtual-wan-link service

# Détail du service "bascule"
diagnose sys virtual-wan-link service 1

# Statistiques de performance
diagnose sys virtual-wan-link stats
```

## 📚 RÉFÉRENCES

### Documentation FortiGate
- FortiOS 6.0.3 Administration Guide
- SD-WAN / Application Control Guide
- OSPF Implementation Guide

### RFC pertinentes
- **RFC 2328** : OSPF Version 2

### Outils utilisés
- **GNS3** : Émulation réseau
- **Wireshark** : Analyse de paquets
- **FortiGate VM** : Version 6.0.3-build0200
- **Cisco IOS** : Version 15.2

---

🏁