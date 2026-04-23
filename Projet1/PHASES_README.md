# Plan d'évolution : du projet pédagogique à la maquette PME réaliste

> Ce dossier contient 5 phases d'amélioration progressives pour transformer ton projet actuel en une maquette défendable en entretien technique.

## 📋 Vue d'ensemble

| Phase | Fichier | Focus | Durée | Priorité |
|---|---|---|---|---|
| 1 | `PHASE1_securite.md` | Sécurité de base (SSH, passwords, port-security) | 1-2 h | 🔴 Critique |
| 2 | `PHASE2_internet.md` | Accès Internet + NAT | 1 h | 🔴 Critique |
| 3 | `PHASE3_wifi.md` | Wi-Fi + VLAN invités | 1 h | 🟡 Important |
| 4 | `PHASE4_vlans_admin.md` | VLAN MGMT + VLAN Servers | 1 h 30 | 🟡 Important |
| 5 | `PHASE5_services_avances.md` | NTP, Syslog, SNMP, sauvegardes | 2 h | 🟢 Pro |

**Total estimé : ~7 h** pour enrichir la maquette PME avec les fonctions d'un vrai réseau d'entreprise.

---

## 🎯 Ordre recommandé

Respecte cet ordre — chaque phase s'appuie sur la précédente.

1. **Phase 1 d'abord** : ne jamais avancer sans sécurité de base
2. **Phase 2 ensuite** : Internet ouvre la voie aux autres services
3. **Phase 3** : Wi-Fi réutilise le NAT de la Phase 2
4. **Phase 4** : refonte VLAN, modifie aussi l'ACL de la Phase 1
5. **Phase 5** : services qui supposent que tout le reste tourne

---

## 📐 Architecture finale (après les 5 phases)

```
                    [Internet / FAI simule]
                              |
                     [Cloud ISP]
                              |
                     (203.0.113.1/30)
                              |
                          WAN Fa0/1
                     +-----------------+
                     |       R1        |
                     | NAT + DHCP + CME|
                     | ACL + NTP + SNMP|
                     +--------+--------+
                              | trunk 802.1Q
                              |
                     +--------+--------+
                     |       SW1       |  (VLAN 99 MGMT)
                     | Port-security   |
                     | SSH + SNMP      |
                     +--------+--------+
                              |
     +------+------+------+------+------+------+
     |      |      |      |      |      |      |
   V1 PC  V10   V20   V30   V99   Trunk
          Voice Guest Servers MGMT  (inter-VLAN)
     |      |      |      |      |
   4 PCs  2 Tel  AP WiFi SRV1  Poste admin
          1001/  + LAP/ DNS +
          1002  Smart HTTP
                phone  NTP
                      SNMP
                      Syslog
                      TFTP
```

---

## 🎓 Compétences acquises par phase

### Phase 1 — Sécurité
- SSH, RSA keys, usernames avec privilèges
- Port-security, MAC sticky
- Banner MOTD légal

### Phase 2 — Internet + NAT
- Routes statiques par défaut
- NAT PAT (overload)
- ACL pour filtrage NAT
- Wildcard masks

### Phase 3 — Wi-Fi
- WPA2-PSK, SSID
- Isolation VLAN invités
- DHCP multi-pool

### Phase 4 — VLAN Admin & Servers
- SVI (Switch Virtual Interface)
- Segmentation par rôle
- ACL d'administration
- Default gateway sur switch

### Phase 5 — Services pro
- NTP (master/client)
- Syslog centralisé (niveaux de sévérité)
- SNMP v2c (OID, MIB, traps)
- Réservations DHCP (client-identifier)
- Anti-spoofing
- Sauvegarde TFTP

---

## 🛠️ Convention de nommage recommandée (PME réaliste)

Applique progressivement ces noms quand tu avances dans les phases.

| Type | Ancien | Nouveau |
|---|---|---|
| Routeur | Router0 | `RTR-CORE-01` |
| Switch | Switch0 | `SW-ACCESS-01` |
| Serveur | Server0 | `SRV-LAN-01` |
| AP | Access Point0 | `AP-WIFI-01` |
| Imprimante | Printer0 | `PRN-BUR-01` |
| PC | PC1 | `WS-SALON-01` |
| Laptop | Laptop0 | `WS-LAPTOP-01` |
| Téléphone | IP Phone0 | `PHONE-SALON-01` |

Format : `TYPE-LOCALISATION-NUMERO`

---

## 📁 Organisation suggérée du dossier projet

```
D:\Yannick\Network\Projet1\
├── Projet1_Reseau_Domestique_Telephonie_IP.docx
├── Reseau_Domestique_Telephonie_IP.pkt
├── topologie.pdf
├── topologie.png
├── PHASES_README.md                 <- ce fichier
├── PHASE1_securite.md
├── PHASE2_internet.md
├── PHASE3_wifi.md
├── PHASE4_vlans_admin.md
├── PHASE5_services_avances.md
├── ROUTER1.txt
├── SWITCH1.txt
├── SERVER.txt
└── AutresConfig.txt
```

---

## 🧭 Conseils pratiques

1. **Sauvegarde ton .pkt avant chaque phase** (ex : `Projet1_avant_phase2.pkt`). Si ça casse, tu peux revenir en arrière.

2. **Garde tes configs .txt à jour** : à la fin de chaque phase, remets à jour ROUTER1.txt / SWITCH1.txt / SERVER.txt avec les nouvelles commandes, pour avoir une antisèche finale complète.

3. **Teste après chaque phase** avant de passer à la suivante.

4. **Documente tes mots de passe** dans un fichier séparé (et sécurisé !), surtout quand tu empiles les phases.

5. **N'hésite pas à adapter** : certaines IP, noms ou choix peuvent être modifiés selon tes préférences — l'important c'est de comprendre le principe.

---

## 🚀 Pour aller encore plus loin (après les 5 phases)

Si tu veux vraiment pousser jusqu'au niveau "jeune ingénieur" :

- **HSRP** (redondance de passerelle avec 2 routeurs)
- **OSPF** sur plusieurs sites
- **VPN IPsec site-à-site**
- **RADIUS** pour authentification 802.1X
- **QoS** avancée avec policy-maps pour priorité voix
- **IPv6** en dual-stack

Tous ces sujets existent dans Packet Tracer et sont accessibles après les 5 phases.

Bon courage !
