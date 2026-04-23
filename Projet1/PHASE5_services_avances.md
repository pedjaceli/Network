# PHASE 5 — Services avancés (NTP, Syslog, SNMP, réservations DHCP)

> Objectif : ajouter les services qui font la différence entre un réseau "qui marche" et un réseau "qui se gère vraiment".
> Durée estimée : 2 h.
> Prérequis : Phases 1 à 4 terminées.

---

## 🎯 Ce que tu vas mettre en place

| Élément | Objectif |
|---|---|
| NTP | Synchronisation de l'heure (indispensable pour les logs) |
| Syslog | Envoi centralisé des logs vers un serveur |
| SNMP v2c | Supervision en lecture seule |
| Réservations DHCP | Stabiliser les IP des équipements critiques |
| ACL anti-spoofing | Bloquer les IP sources invalides |
| Sauvegarde automatique | Copie périodique de la config |

---

## 1. NTP — Synchronisation horaire

### 1.1 Pourquoi c'est critique

Sans NTP, chaque équipement a sa propre heure locale → les logs sont **inexploitables** (tu ne peux pas corréler un événement entre R1, SW1 et un serveur).

### 1.2 Architecture

- R1 sert d'horloge maître pour le LAN
- R1 se synchronise lui-même sur un serveur NTP externe (simulé avec le serveur 8.8.8.8 ou via le Cloud)

### 1.3 Configuration sur R1

```
enable
configure terminal

clock timezone CET +1
clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00

ntp server 8.8.8.8
ntp master 3
exit
```

- `clock timezone CET +1` = Europe/Paris (hiver)
- `clock summer-time` = gestion du changement heure d'été
- `ntp master 3` = R1 devient source NTP pour le LAN (stratum 3)

### 1.4 Configurer SW1 pour se synchroniser sur R1

Sur SW1 :
```
configure terminal
ntp server 192.168.99.1
exit
```

### 1.5 Vérifications

```
show ntp status
show clock
```

---

## 2. Syslog — Logs centralisés

### 2.1 Pourquoi c'est important

Toutes les actions (connexion SSH, changement de config, interface qui tombe, paquet refusé par ACL...) deviennent **traçables** et historisées.

### 2.2 Activer le service Syslog sur SRV1

- Sur SRV1 → onglet **Services** → **SYSLOG**
- **Service** : **On**

Les messages arriveront automatiquement quand R1 et SW1 les enverront.

### 2.3 Configuration R1

```
enable
configure terminal

logging 192.168.30.53
logging trap informational
logging source-interface fa0/0.99
service timestamps log datetime msec localtime show-timezone

exit
```

- `logging 192.168.30.53` = destination syslog (SRV1)
- `logging trap informational` = niveau de sévérité (informational = niveau 6)
- `logging source-interface fa0/0.99` = l'IP source des logs (interface MGMT)
- `service timestamps` = date précise dans chaque log

### 2.4 Configuration SW1

Même chose :
```
configure terminal

logging 192.168.30.53
logging trap informational
logging source-interface vlan 99
service timestamps log datetime msec localtime show-timezone

exit
```

### 2.5 Vérification

- Fais une action (ex : `shutdown` sur une interface puis `no shutdown`)
- Sur SRV1 → Services → SYSLOG → tu dois voir des lignes apparaître
- Exemple : `%LINK-5-CHANGED: Interface FastEthernet0/5, changed state to administratively down`

### 2.6 Niveaux Syslog

| Niveau | Nom | Description |
|---|---|---|
| 0 | emergencies | Système inutilisable |
| 1 | alerts | Action immédiate requise |
| 2 | critical | Condition critique |
| 3 | errors | Erreur |
| 4 | warnings | Avertissement |
| 5 | notifications | Événement normal mais notable |
| 6 | informational | Information |
| 7 | debugging | Debug (très verbeux) |

En prod : niveau 6 (informational) est le bon compromis.

---

## 3. SNMP — Supervision

### 3.1 Concept

SNMP permet à un outil de monitoring (PRTG, Zabbix, LibreNMS, Nagios) de **lire** les compteurs d'un équipement : CPU, mémoire, trafic par interface, état des ports, etc.

### 3.2 Activer SNMP v2c en lecture seule sur R1

```
enable
configure terminal

snmp-server community PublicReadOnly RO
snmp-server location Bureau_Yannick_Paris
snmp-server contact admin@pme.local
snmp-server enable traps
snmp-server host 192.168.30.53 version 2c PublicReadOnly

exit
```

- `PublicReadOnly` = "nom de communauté" (comme un mot de passe simple)
- `RO` = read-only
- `snmp-server host` = destination des traps (alertes proactives)

### 3.3 Même chose sur SW1

```
configure terminal

snmp-server community PublicReadOnly RO
snmp-server location Bureau_Yannick_Paris
snmp-server contact admin@pme.local
snmp-server host 192.168.30.53 version 2c PublicReadOnly

exit
```

### 3.4 Vérification côté serveur

Dans PT, il n'y a pas de vrai "serveur SNMP", mais on peut **simuler** :
- SRV1 → onglet **Desktop** → **MIB Browser**
- Address : `192.168.99.1` (R1 via MGMT) ou `192.168.1.1`
- Community : `PublicReadOnly`
- Clique sur **Advanced** puis sélectionne un OID (ex : sysUpTime) → **Go**

Tu vois la valeur remontée par SNMP.

> ⚠️ SNMP v2c envoie la community en clair. En prod moderne : **SNMP v3** (authentification + chiffrement).

---

## 4. Réservations DHCP

### 4.1 Pourquoi

Actuellement, tes PC reçoivent une IP au hasard dans le pool. Pour le DNS interne, tu veux une **IP stable** → réservation basée sur la MAC.

### 4.2 Trouver les MAC des PC

Sur chaque PC : Desktop → Command Prompt :
```
ipconfig /all
```

Note la MAC (ex : `00D0.D3E4.1234`).

### 4.3 Créer des pools individuels sur R1

Pour chaque PC, un pool dédié :

```
configure terminal

ip dhcp pool PC1-Salon
 host 192.168.1.10 255.255.255.0
 client-identifier 0100.D0D3.E412.34
 default-router 192.168.1.1
 dns-server 192.168.30.53
exit

ip dhcp pool PC2-Bureau
 host 192.168.1.11 255.255.255.0
 client-identifier 0100.D0D3.AB56.78
 default-router 192.168.1.1
 dns-server 192.168.30.53
exit
```

> **Note importante sur le client-identifier** :
> Le format est `01` + **MAC sans tirets ni deux-points**, avec des points tous les 4 caractères.
> Ex : MAC `00:D0:D3:E4:12:34` → `client-identifier 0100D0.D3E4.1234`
> Petite variante selon IOS — si ça ne marche pas, tente `hardware-address 00D0.D3E4.1234`.

### 4.4 Test

Sur le PC concerné : IP Configuration → Static → DHCP.
L'IP doit être **toujours la même** (celle réservée).

---

## 5. ACL anti-spoofing

### 5.1 Principe

Bloquer les paquets qui prétendent venir d'une IP impossible (ex : un paquet arrivant sur la sous-interface fa0/0.1 avec une IP source qui n'est pas dans 192.168.1.0/24).

### 5.2 ACL 150 — Anti-spoofing sur VLAN 1

```
configure terminal

access-list 150 permit ip 192.168.1.0 0.0.0.255 any
access-list 150 deny ip any any log

interface fa0/0.1
 ip access-group 150 in
exit
```

> ⚠️ Attention : tu as déjà l'ACL 199 en `in` sur fa0/0.1 (Phase 4). Tu ne peux en mettre **qu'une seule** dans chaque sens. Intègre les règles anti-spoof **dans l'ACL 199**.

Nouvelle version de l'ACL 199 (fusionnée) :
```
no access-list 199

access-list 199 deny ip host 192.168.1.12 any
access-list 199 deny ip any 192.168.99.0 0.0.0.255
access-list 199 permit ip host 192.168.1.10 any
access-list 199 permit ip 192.168.1.0 0.0.0.255 192.168.30.0 0.0.0.255
access-list 199 permit ip 192.168.1.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 199 permit ip 192.168.1.0 0.0.0.255 any
access-list 199 deny ip any any log
```

---

## 6. Sauvegarde automatique de la config

### 6.1 Principe

Pour éviter de perdre la config en cas de panne, on pousse régulièrement le `running-config` vers un serveur TFTP.

### 6.2 Activer TFTP sur SRV1

- SRV1 → **Services** → **TFTP** → **On**

### 6.3 Config R1 — sauvegarde à la main

```
enable
copy running-config tftp://192.168.30.53/R1-config-20260422.cfg
```

### 6.4 Config R1 — automatisation via archive (optionnel, support PT limité)

```
configure terminal

archive
 path tftp://192.168.30.53/R1-backup
 write-memory
 time-period 1440
exit
```

- `time-period 1440` = toutes les 24 h (en minutes)
- `write-memory` = sauvegarde aussi quand on fait `write`

---

## 7. Vérifications globales

### 7.1 Sur R1

```
show ntp status
show logging
show snmp
show ip dhcp pool
show access-lists
```

### 7.2 Sur SRV1 (Services)

- **Syslog** : messages qui arrivent
- **TFTP** : fichiers de config stockés

---

## ⚠️ Pièges courants

| Problème | Solution |
|---|---|
| NTP jamais synchronisé | Vérifie que R1 atteint 8.8.8.8 et que SW1 atteint 192.168.99.1 |
| Pas de logs sur SRV1 | Vérifie `logging source-interface` et qu'il peut joindre le serveur |
| MIB Browser ne répond pas | Vérifie la community et que la config SNMP est active |
| Réservation DHCP inactive | Vérifie le format exact du client-identifier |
| Sauvegarde TFTP échoue | Vérifie que TFTP est `On` et que le chemin est accessible |

---

## ✅ Checklist phase 5

- [ ] NTP configuré sur R1 (master) et SW1 (client)
- [ ] Fuseau horaire et heure d'été définis
- [ ] Syslog activé sur SRV1 et destinataire de R1 + SW1
- [ ] SNMP v2c RO configuré sur R1 et SW1
- [ ] Test SNMP via MIB Browser fonctionne
- [ ] Réservations DHCP créées pour les PC principaux
- [ ] Anti-spoofing intégré dans l'ACL 199
- [ ] TFTP sur SRV1 actif et sauvegarde de R1 testée
- [ ] Configs sauvegardées (`copy running-config startup-config`)

---

## 🏆 Félicitations

Si tu as complété les 5 phases, tu as maintenant une maquette PME **très réaliste**, couvrant :

- Infrastructure : VLAN multi-catégories, trunk, inter-VLAN routing
- Services : DHCP, DNS, HTTP, VoIP, NTP, Syslog, SNMP
- Sécurité : SSH, port-security, ACL anti-spoofing, isolation MGMT
- Exploitation : monitoring SNMP, logs centralisés, sauvegardes

👉 C'est le niveau attendu d'un **technicien réseau junior / stagiaire bien formé** en entreprise.
