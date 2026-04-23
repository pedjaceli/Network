# PHASE 3 — Wi-Fi (avec VLAN invités)

> Objectif : ajouter un réseau Wi-Fi pour les invités et les appareils mobiles, isolé du réseau filaire principal.
> Durée estimée : 1 h.
> Prérequis : Phases 1 et 2 terminées.

---

## 🎯 Ce que tu vas mettre en place

| Élément | Objectif |
|---|---|
| VLAN 20 GUEST | Réseau séparé pour invités et Wi-Fi |
| Point d'accès Wi-Fi | Connectivité sans fil |
| Sous-interface fa0/0.20 sur R1 | Passerelle du VLAN 20 |
| Pool DHCP GUEST | Distribuer des IP aux appareils Wi-Fi |
| ACL d'isolement | Empêcher les invités d'accéder au LAN interne |

---

## 1. Plan d'adressage actualisé

| VLAN | Nom | Sous-réseau | Passerelle | Usage |
|---|---|---|---|---|
| 1 | USERS | 192.168.1.0/24 | 192.168.1.1 | PC filaires, imprimante, serveur |
| 10 | VOICE | 192.168.10.0/24 | 192.168.10.1 | Téléphones IP |
| **20** | **GUEST** | **192.168.20.0/24** | **192.168.20.1** | **Wi-Fi (invités + mobiles)** |

---

## 2. Configuration R1

### 2.1 Nouvelle sous-interface pour le VLAN 20

```
enable
configure terminal

interface fa0/0.20
 description LAN Invites Wifi - VLAN 20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip nat inside
exit
```

### 2.2 Pool DHCP GUEST

```
ip dhcp excluded-address 192.168.20.1 192.168.20.9

ip dhcp pool GUEST
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
exit
```

Note : les invités utilisent un DNS externe (8.8.8.8) plutôt que le DNS interne — ils n'ont pas à connaître les noms internes.

### 2.3 Autoriser le VLAN 20 dans le NAT

```
access-list 1 permit 192.168.20.0 0.0.0.255
```

(Si l'ACL 1 existe déjà, cette ligne s'ajoute aux existantes.)

### 2.4 ACL d'isolement GUEST ↔ LAN interne

Objectif : les invités peuvent sortir sur Internet mais pas toucher au VLAN 1 ni VLAN 10.

```
access-list 120 deny ip 192.168.20.0 0.0.0.255 192.168.1.0 0.0.0.255
access-list 120 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 120 permit ip any any

interface fa0/0.20
 ip access-group 120 in
exit
```

### 2.5 Sauvegarde

```
end
copy running-config startup-config
```

---

## 3. Configuration SW1

### 3.1 Créer le VLAN 20

```
enable
configure terminal

vlan 20
 name GUEST
exit
```

### 3.2 Attribuer un port au VLAN 20 pour l'AP

On utilise le port Fa0/12 pour le point d'accès.

```
interface fa0/12
 description Point d acces Wi-Fi (VLAN 20)
 no shutdown
 switchport mode access
 switchport access vlan 20
exit
```

### 3.3 Sauvegarde

```
end
copy running-config startup-config
```

---

## 4. Ajouter le point d'accès dans Packet Tracer

### 4.1 Choix de l'équipement

- Panneau d'équipements → **Wireless Devices** → **WRT300N** (routeur Wi-Fi grand public) OU **AccessPoint-PT** (AP pro simple)

Pour simplifier, utilise **AccessPoint-PT**.

### 4.2 Ajouter et câbler

- Glisser l'AP sur le plan de travail
- Renommer "AP-WIFI"
- Câbler : **AP Port 0** → **SW1 Fa0/12** (câble cuivre droit)

### 4.3 Configurer le SSID et la sécurité

- Clique sur AP → onglet **Config** → **Port 1 (Wi-Fi)**
- **SSID** : `MaisonWifi`
- **Authentication** : **WPA2-PSK**
- **Encryption type** : **AES**
- **PSK Pass Phrase** : `Wifi2026Secure!`

---

## 5. Ajouter des appareils Wi-Fi

### 5.1 Laptop

- Panneau d'équipements → **End Devices** → **Laptop-PT**
- Glisser sur le plan de travail, renommer "LAPTOP1"

### 5.2 Remplacer la carte filaire par une carte Wi-Fi

- Clique sur le laptop → onglet **Physical**
- **Eteins-le** (bouton on/off à droite)
- **Retire** le module Ethernet (glisse-le vers la liste de gauche)
- **Installe** le module `WPC300N` (carte Wi-Fi) dans l'emplacement libre
- **Rallume** le laptop

### 5.3 Se connecter au Wi-Fi

- Onglet **Desktop** → **PC Wireless**
- Onglet **Connect** → sélectionne le SSID `MaisonWifi` → **Connect**
- Tape la passphrase `Wifi2026Secure!`

Le laptop doit obtenir une IP en 192.168.20.x (via DHCP).

### 5.4 Ajouter un smartphone

- **End Devices** → **Smart Device** (ou **Tablet-PT**)
- Glisse sur le plan, renomme "SMARTPHONE1"
- Onglet **Config** → **Wireless0** → entre le SSID et la passphrase
- Desktop → IP Configuration → DHCP

---

## 6. Tests de validation

### 6.1 Le laptop reçoit une IP Wi-Fi

Laptop → Desktop → IP Configuration → tu dois voir :
- IP : `192.168.20.10` (ou autre)
- Passerelle : `192.168.20.1`
- DNS : `8.8.8.8`

### 6.2 Le laptop peut sortir sur Internet

```
ping 8.8.8.8
```

→ Doit répondre (grâce à la passerelle + NAT).

### 6.3 Le laptop NE DOIT PAS accéder au LAN interne

```
ping 192.168.1.10     (vers PC1)
ping 192.168.1.53     (vers serveur)
```

→ Doivent **échouer** (bloqués par l'ACL 120).

### 6.4 Un PC filaire peut toujours tout faire

PC1 :
```
ping 8.8.8.8          (OK)
ping 192.168.1.50     (OK)
ping 192.168.10.1     (OK)
```

---

## 7. Bonus : portail captif (concept, pas faisable en PT de base)

Dans un vrai déploiement PME, on ajouterait :
- Un **portail captif** (page de connexion avec mot de passe) — nécessite un équipement Mikrotik, FortiGate, pfSense, ou un serveur RADIUS.
- Une **limitation de débit** pour les invités (QoS).
- Un **log des connexions** (légal en France : conservation 1 an).

Packet Tracer n'émule pas ces éléments, mais l'architecture VLAN + ACL que tu viens de faire est **la fondation** sur laquelle on bâtit ça en vrai.

---

## ⚠️ Pièges courants

| Problème | Solution |
|---|---|
| Laptop ne voit pas le SSID | Vérifie que la carte Wi-Fi est installée (onglet Physical) |
| Laptop voit le SSID mais ne se connecte pas | Vérifie WPA2-PSK et la passphrase (exacte) |
| Laptop connecté mais pas d'IP | Vérifie le pool DHCP GUEST et la sous-interface fa0/0.20 |
| Laptop peut ping PC1 (pas normal) | Vérifie que l'ACL 120 est bien appliquée `in` sur fa0/0.20 |
| Pas de sortie Internet depuis le Wi-Fi | Vérifie que VLAN 20 est dans l'ACL 1 du NAT |

---

## ✅ Checklist phase 3

- [ ] VLAN 20 créé sur SW1
- [ ] Sous-interface fa0/0.20 sur R1 avec IP 192.168.20.1/24
- [ ] Pool DHCP GUEST fonctionnel
- [ ] ACL 120 isole GUEST du LAN interne
- [ ] NAT autorisé pour VLAN 20
- [ ] Port SW1 Fa0/12 configuré en access VLAN 20
- [ ] AP-WIFI ajouté et configuré (SSID + WPA2-PSK)
- [ ] LAPTOP1 associé au Wi-Fi et IP 192.168.20.x reçue
- [ ] Laptop peut ping 8.8.8.8
- [ ] Laptop NE peut PAS ping 192.168.1.x
- [ ] Configs sauvegardées
