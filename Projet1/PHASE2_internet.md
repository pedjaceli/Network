# PHASE 2 — Accès Internet + NAT

> Objectif : permettre à tous les PC et téléphones de sortir sur Internet via R1.
> Durée estimée : 1 h.
> Prérequis : Phase 1 terminée.

---

## 🎯 Ce que tu vas mettre en place

| Élément | Objectif |
|---|---|
| Objet "Cloud" dans PT | Simule le FAI (fournisseur d'accès Internet) |
| Interface WAN sur R1 | Lien vers le "monde extérieur" |
| Route par défaut | Vers l'Internet |
| NAT PAT (overload) | Tous les PC sortent via une seule IP publique |
| Serveur Internet fictif | Pour tester la sortie |

---

## 1. Topologie cible après Phase 2

```
       [ Internet simule ]
       (Server public 8.8.8.8)
                |
            [ Cloud ]      <- FAI
                |
          (203.0.113.1/30)
                |  WAN
                |
          Fa0/1 |
         [  R1  ]
                | Fa0/0 (trunk)
          [ SW1 ]
                |
           PCs + Phones
```

---

## 2. Ajouter les équipements dans Packet Tracer

### 2.1 Ajouter le Cloud (FAI)

- Panneau d'équipements → **Network Devices** → **WAN Emulation** → glisser un **Cloud-PT**
- Le renommer "ISP" (double-clic sur le nom)

### 2.2 Ajouter un serveur Internet fictif

- Glisser un **Server-PT** et le relier au Cloud par un câble cuivre droit
- Renommer "SRV-INTERNET"
- Configurer IP statique :
  - IP : `8.8.8.8`
  - Masque : `255.255.255.0`
  - Passerelle : `8.8.8.1`

### 2.3 Câblage R1 ↔ Cloud

- Câble cuivre droit :
  - **R1 Fa0/1** → **Cloud Fa0** (ou Eth0 selon ton cloud)

### 2.4 Configurer le Cloud

- Clique sur Cloud → onglet **Config**
- Tu n'as quasi rien à faire dans PT Cloud, mais vérifie que les ports sont connectés (voyant vert).

---

## 3. Configuration R1 — interface WAN

```
enable
configure terminal

interface fa0/1
 description Lien vers FAI (Internet)
 ip address 203.0.113.2 255.255.255.252
 no shutdown
exit
```

Note : le sous-réseau /30 ne contient que 2 IP utilisables (.1 pour le FAI, .2 pour nous).

---

## 4. Route par défaut

```
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

Traduction : "tout ce que je ne connais pas, je l'envoie à 203.0.113.1 (le FAI)".

---

## 5. NAT PAT (overload)

### 5.1 Définir les interfaces inside / outside

```
interface fa0/0.1
 ip nat inside
exit

interface fa0/0.10
 ip nat inside
exit

interface fa0/1
 ip nat outside
exit
```

### 5.2 Définir le trafic à translater (ACL 1)

```
access-list 1 permit 192.168.1.0 0.0.0.255
access-list 1 permit 192.168.10.0 0.0.0.255
```

Note : `0.0.0.255` est un **wildcard mask** (l'inverse d'un masque réseau).
- `255.255.255.0` → `0.0.0.255`
- `255.255.255.128` → `0.0.0.127`

### 5.3 Activer le NAT overload

```
ip nat inside source list 1 interface fa0/1 overload
```

Traduction :
- `list 1` = ACL qui définit qui a le droit d'être translaté
- `interface fa0/1` = l'IP publique utilisée
- `overload` = PAT (plusieurs PC partagent la même IP publique via différents ports)

### 5.4 Sauvegarde

```
end
copy running-config startup-config
```

---

## 6. Tests de validation

### 6.1 Ping vers Internet depuis R1

```
ping 8.8.8.8
```

→ Doit répondre (R1 a une route par défaut).

### 6.2 Ping vers Internet depuis PC1

PC1 → Desktop → Command Prompt :

```
ping 8.8.8.8
```

→ Doit répondre si le NAT fonctionne.

### 6.3 Visualiser la table NAT

Sur R1 :
```
show ip nat translations
```

Tu dois voir une ligne du type :
```
Pro Inside global      Inside local       Outside local      Outside global
icmp 203.0.113.2:1     192.168.1.10:1     8.8.8.8:1          8.8.8.8:1
```

→ PC1 (192.168.1.10) sort via 203.0.113.2 vers 8.8.8.8.

### 6.4 Tester le HTTP externe (si tu as activé HTTP sur SRV-INTERNET)

Depuis PC1 → Web Browser :
```
http://8.8.8.8
```

---

## 7. Bonus : résolution DNS externe

Pour que les PC puissent aussi résoudre des noms externes (ex : www.google.com) :

### 7.1 Modifier le pool DHCP DATA sur R1

```
configure terminal
ip dhcp pool DATA
 dns-server 192.168.1.53 8.8.8.8
exit
```

→ Les PC auront 2 DNS : le local en priorité, puis 8.8.8.8 pour l'extérieur.

### 7.2 Sur le serveur DNS local (SRV1)

Pour déléguer les résolutions externes, tu peux configurer un **forwarder** (pas trivial dans PT, mais le concept existe).

En pratique dans PT, on se contente de la config DHCP avec 2 DNS.

---

## ⚠️ Pièges courants

| Problème | Solution |
|---|---|
| Ping 8.8.8.8 échoue depuis R1 | Vérifie la route par défaut `show ip route` |
| Ping 8.8.8.8 échoue depuis PC mais OK depuis R1 | Problème NAT : vérifie inside/outside et ACL 1 |
| `show ip nat translations` est vide | Personne n'a essayé de sortir OU l'ACL ne matche pas |
| Cloud déconnecté | Vérifie que les 2 extrémités du câble clignotent vert |

---

## ✅ Checklist phase 2

- [ ] Cloud ISP ajouté et câblé à R1 Fa0/1
- [ ] Serveur 8.8.8.8 derrière le Cloud
- [ ] Interface WAN R1 configurée (203.0.113.2/30)
- [ ] Route par défaut pointant vers 203.0.113.1
- [ ] NAT overload actif (ACL 1 + ip nat inside source)
- [ ] `ping 8.8.8.8` fonctionne depuis PC1
- [ ] `show ip nat translations` affiche des entrées
- [ ] Configs sauvegardées
