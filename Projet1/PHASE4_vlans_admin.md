# PHASE 4 — VLAN d'administration + isolation des serveurs

> Objectif : séparer clairement l'administration du réseau des utilisateurs, et isoler les serveurs dans leur propre VLAN. Convention pro incontournable.
> Durée estimée : 1 h 30.
> Prérequis : Phases 1 à 3 terminées.

---

## 🎯 Ce que tu vas mettre en place

| Élément | Objectif |
|---|---|
| VLAN 30 SERVERS | Isoler les serveurs dans leur propre segment |
| VLAN 99 MGMT | Administration des équipements réseau |
| SVI sur SW1 (VLAN 99) | IP d'administration du switch |
| Relocation du serveur vers VLAN 30 | Isolation physique/logique |
| ACL d'accès au MGMT | Seul l'admin peut se connecter aux équipements |

---

## 1. Plan d'adressage final (la version PME complète)

| VLAN | Nom | Sous-réseau | Passerelle | Usage |
|---|---|---|---|---|
| 1 | USERS | 192.168.1.0/24 | 192.168.1.1 | PC filaires, imprimante |
| 10 | VOICE | 192.168.10.0/24 | 192.168.10.1 | Téléphones IP |
| 20 | GUEST | 192.168.20.0/24 | 192.168.20.1 | Wi-Fi invités |
| **30** | **SERVERS** | **192.168.30.0/24** | **192.168.30.1** | **Serveurs (DNS, HTTP, ...)** |
| **99** | **MGMT** | **192.168.99.0/24** | **192.168.99.1** | **Administration réseau** |

---

## 2. Configuration R1

### 2.1 Nouvelles sous-interfaces

```
enable
configure terminal

interface fa0/0.30
 description LAN Serveurs - VLAN 30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
 ip nat inside
exit

interface fa0/0.99
 description Administration - VLAN 99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.0
exit
```

### 2.2 Pool DHCP pour SERVERS (optionnel, les serveurs sont souvent en statique)

On garde les serveurs en statique — pas de pool pour le VLAN 30.

### 2.3 Pool DHCP pour MGMT (optionnel, rarement utilisé)

Idem, l'admin utilise des IP statiques connues.

### 2.4 ACL : protéger le VLAN MGMT

Objectif : seul le poste admin (ex : 192.168.1.10 = PC1) peut accéder au VLAN 99.

```
access-list 199 permit ip host 192.168.1.10 192.168.99.0 0.0.0.255
access-list 199 permit ip 192.168.99.0 0.0.0.255 any
access-list 199 deny ip any 192.168.99.0 0.0.0.255
access-list 199 permit ip any any

interface fa0/0.99
 ip access-group 199 in
exit

interface fa0/0.1
 ip access-group 199 in
exit
```

> ⚠️ Attention : tu as déjà une ACL 110 sur fa0/0.1 (bloquage PC3). Il faut fusionner les deux ou choisir. Pour simplifier, on remplace :

```
interface fa0/0.1
 no ip access-group 110 in
 ip access-group 199 in
exit
```

Et on fusionne les 2 ACL :
```
no access-list 110
no access-list 199

access-list 199 deny ip host 192.168.1.12 any
access-list 199 permit ip host 192.168.1.10 192.168.99.0 0.0.0.255
access-list 199 deny ip any 192.168.99.0 0.0.0.255
access-list 199 permit ip any any
```

Ordre : PC3 bloqué d'abord, puis MGMT réservé à PC1, puis permit par défaut.

### 2.5 Autoriser le VLAN 30 dans le NAT

```
access-list 1 permit 192.168.30.0 0.0.0.255
```

### 2.6 Sauvegarde

```
end
copy running-config startup-config
```

---

## 3. Configuration SW1

### 3.1 Créer les VLAN

```
enable
configure terminal

vlan 30
 name SERVERS
exit

vlan 99
 name MGMT
exit
```

### 3.2 Déplacer le port du serveur vers VLAN 30

```
interface fa0/6
 description SRV1 Serveur DNS HTTP - VLAN 30
 switchport access vlan 30
exit
```

### 3.3 Créer la SVI VLAN 99 pour l'administration du switch

Supprimer l'ancienne SVI sur VLAN 1 :
```
interface vlan 1
 no ip address
 shutdown
exit
no ip default-gateway
```

Créer la nouvelle :
```
interface vlan 99
 description SVI Administration
 ip address 192.168.99.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.99.1
```

### 3.4 Port dédié pour brancher un poste admin dans le VLAN MGMT

Par exemple Fa0/13 :

```
interface fa0/13
 description Poste admin - VLAN 99
 no shutdown
 switchport mode access
 switchport access vlan 99
exit
```

### 3.5 Sauvegarde

```
end
copy running-config startup-config
```

---

## 4. Mettre à jour le serveur SRV1

### 4.1 Nouvelle IP dans le VLAN 30

Sur SRV1 → Desktop → IP Configuration → Static :
- IP : `192.168.30.53`
- Masque : `255.255.255.0`
- Passerelle : `192.168.30.1`
- DNS : `127.0.0.1`

### 4.2 Mettre à jour le DNS (nouvelles IP)

Dans Services → DNS :
- Modifie `serveur.pme.local` → `192.168.30.53`
- Modifie `www.pme.local` → `192.168.30.53`

### 4.3 Mettre à jour le pool DHCP DATA sur R1

```
configure terminal
ip dhcp pool DATA
 dns-server 192.168.30.53
exit
```

Sur chaque PC : Desktop → IP Configuration → Static puis DHCP (pour renouveler).

### 4.4 Exclure la nouvelle IP du DHCP

```
ip dhcp excluded-address 192.168.30.1 192.168.30.9
ip dhcp excluded-address 192.168.30.53
```

### 4.5 Mettre à jour aussi les enregistrements DNS des autres équipements

Si tu veux garder les noms pme.local cohérents :

| Nom | Ancienne IP | Nouvelle IP |
|---|---|---|
| routeur.pme.local | 192.168.1.1 | 192.168.1.1 (inchangé) |
| serveur.pme.local | 192.168.1.53 | **192.168.30.53** |
| imprimante.pme.local | 192.168.1.50 | 192.168.1.50 (inchangé) |

---

## 5. Tests de validation

### 5.1 Depuis PC1 — accès au serveur dans le nouveau VLAN

```
ping 192.168.30.53
ping serveur.pme.local
```

→ Doit fonctionner (inter-VLAN via R1).

### 5.2 Depuis PC1 — accès au MGMT (autorisé)

```
ping 192.168.99.2     (SVI du switch)
ssh -l admin 192.168.99.2
```

→ Doit fonctionner (ACL 199 autorise PC1 vers MGMT).

### 5.3 Depuis PC2 — accès au MGMT (refusé)

```
ping 192.168.99.2
```

→ Doit échouer (ACL 199 n'autorise que PC1).

### 5.4 Depuis le Wi-Fi — accès au MGMT (refusé)

Laptop :
```
ping 192.168.99.2
```

→ Doit échouer (l'ACL 120 sur VLAN 20 bloque aussi).

### 5.5 Le site HTTP est toujours accessible

PC1 → Web Browser :
```
http://www.pme.local
```

→ Doit afficher la page (DNS redirige vers 192.168.30.53).

---

## 6. Vérifications CLI

Sur SW1 :
```
show vlan brief
show interfaces vlan 99
show ip route
```

Sur R1 :
```
show ip interface brief
show access-lists
show ip nat translations
```

---

## 7. Vue d'ensemble après la phase 4

```
+---------------------+--------------------+---------------------+
| VLAN 1 (USERS)      | VLAN 10 (VOICE)    | VLAN 20 (GUEST)     |
| PC1-PC4, imprimante | TEL-1001, 1002     | LAPTOP, smartphone  |
| 192.168.1.0/24      | 192.168.10.0/24    | 192.168.20.0/24     |
+---------------------+--------------------+---------------------+

+---------------------+--------------------+---------------------+
| VLAN 30 (SERVERS)   | VLAN 99 (MGMT)     |                     |
| SRV1 (DNS+HTTP)     | Acces admin SW/R1  |                     |
| 192.168.30.0/24     | 192.168.99.0/24    |                     |
+---------------------+--------------------+---------------------+
```

C'est la structure **standard PME** qu'on retrouve dans 80% des entreprises.

---

## ⚠️ Pièges courants

| Problème | Solution |
|---|---|
| PC1 n'arrive plus à pinger 192.168.30.53 | Vérifier la sous-interface fa0/0.30 et le tag VLAN 30 sur le trunk |
| SSH au switch échoue | Vérifier la SVI VLAN 99 + IP admin + ACL 199 |
| Perte du site HTTP | Vérifier que le DNS a été mis à jour avec la nouvelle IP |
| ACL 199 bloque tout | Ordre des règles : mettre `permit ip any any` à la fin |
| PC3 peut à nouveau tout pinger | Oubli de la ligne `deny ip host 192.168.1.12 any` dans l'ACL fusionnée |

---

## ✅ Checklist phase 4

- [ ] VLAN 30 (SERVERS) et VLAN 99 (MGMT) créés sur SW1
- [ ] Sous-interfaces fa0/0.30 et fa0/0.99 sur R1
- [ ] SRV1 déplacé en VLAN 30 avec IP 192.168.30.53
- [ ] SVI VLAN 99 sur SW1 avec IP 192.168.99.2
- [ ] DNS mis à jour avec la nouvelle IP du serveur
- [ ] Pool DHCP DATA pointe vers le nouveau DNS
- [ ] ACL 199 protège le VLAN MGMT
- [ ] PC1 accède au MGMT, PC2 ne peut pas
- [ ] Site HTTP toujours accessible via www.pme.local
- [ ] Configs sauvegardées
