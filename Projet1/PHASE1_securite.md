# PHASE 1 — Sécurité de base

> Objectif : passer d'un réseau "ouvert à tous" à un réseau protégé.
> Durée estimée : 1 à 2 h.
> Niveau : essentiel pour une PME réelle.

---

## 🎯 Ce que tu vas mettre en place

| Élément | Objectif |
|---|---|
| Mots de passe console / enable / vty | Empêcher l'accès physique et distant |
| SSH v2 | Remplacer Telnet (Telnet = trafic en clair) |
| Utilisateurs locaux avec rôles | Traçabilité de qui fait quoi |
| Banner MOTD légal | Avertissement d'accès non autorisé |
| Port-security sur le switch | Éviter qu'un intrus branche son PC sur un port |
| Désactivation des ports inutilisés | Réduire la surface d'attaque |

---

## 1. Configuration sur R1

### 1.1 Utilisateurs locaux (avec niveau de privilège)

```
enable
configure terminal

username admin privilege 15 secret Admin2026!
username tech privilege 1 secret Tech2026!
```

- `admin` = administrateur (niveau 15 = tout)
- `tech` = technicien (niveau 1 = consultation seulement)

### 1.2 Mots de passe sur les lignes

```
line console 0
 login local
 exec-timeout 10 0
 logging synchronous
exit

line vty 0 4
 login local
 transport input ssh
 exec-timeout 10 0
exit

line vty 5 15
 login local
 transport input ssh
 exec-timeout 10 0
exit
```

- `login local` = utilise les comptes définis avec `username`
- `transport input ssh` = seul SSH est accepté (Telnet bloqué !)
- `exec-timeout 10 0` = déconnexion auto après 10 min d'inactivité

### 1.3 Mot de passe enable + chiffrement

```
enable secret Enable2026!
service password-encryption
```

### 1.4 Génération de la clé RSA pour SSH

```
ip domain-name pme.local
crypto key generate rsa
```

Quand il demande la taille : tape **1024**.

```
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
```

### 1.5 Banner MOTD légal

```
banner motd #
************************************************
*          ACCES RESERVE AU PERSONNEL           *
*                                               *
*   Tout acces non autorise est interdit et     *
*   sera poursuivi conformement a la loi.       *
*                                               *
*   Admin : Yannick  -  Avril 2026              *
************************************************
#
```

### 1.6 Désactiver les services inutiles

```
no ip http server
no ip http secure-server
no cdp run
```

> NOTE : si tu utilises CDP pour le voice VLAN, garde-le activé. Dans ce cas, on le désactive seulement sur les interfaces externes.

### 1.7 Sauvegarde

```
end
copy running-config startup-config
```

---

## 2. Configuration sur SW1

### 2.1 Utilisateurs + SSH (comme sur R1)

```
enable
configure terminal

hostname SW1
ip domain-name pme.local

username admin privilege 15 secret Admin2026!
username tech privilege 1 secret Tech2026!

enable secret Enable2026!
service password-encryption

crypto key generate rsa
```

(Taille 1024)

```
ip ssh version 2

line console 0
 login local
 exec-timeout 10 0
 logging synchronous
exit

line vty 0 15
 login local
 transport input ssh
 exec-timeout 10 0
exit
```

### 2.2 Interface VLAN de management (temporaire, sera refait en Phase 4)

```
interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.1.1
```

### 2.3 Banner

```
banner motd #
************************************************
*          SW1 - Switch principal PME           *
*   Acces reserve au personnel autorise.        *
************************************************
#
```

### 2.4 Port-security sur les ports utilisateurs

```
interface range fa0/2 - 5
 switchport port-security
 switchport port-security maximum 1
 switchport port-security violation restrict
 switchport port-security mac-address sticky
exit
```

Explications :
- `maximum 1` = 1 seule MAC autorisée par port
- `violation restrict` = bloque le trafic du nouveau MAC et log l'incident (moins agressif que `shutdown`)
- `mac-address sticky` = la MAC est apprise dynamiquement puis mémorisée

### 2.5 Désactiver les ports inutilisés

```
interface range fa0/7 - 9
 shutdown
exit

interface range fa0/12 - 24
 shutdown
exit

interface range gi0/1 - 2
 shutdown
exit
```

### 2.6 Sauvegarde

```
end
copy running-config startup-config
```

---

## 3. Tests de validation

### 3.1 SSH depuis un PC

Sur **PC1** → Desktop → **Command Prompt** :

```
ssh -l admin 192.168.1.1
```

→ Demande le mot de passe `Admin2026!` → accès à R1.

> Si `ssh` n'est pas dans la Command Prompt de PT, essaie l'onglet "Telnet/SSH Client" du Desktop.

### 3.2 Vérifier que Telnet est bloqué

```
telnet 192.168.1.1
```

→ Doit échouer (connection refused).

### 3.3 Tester le banner

Reconnecte-toi en SSH → le banner doit s'afficher avant la demande d'identifiant.

### 3.4 Vérifier port-security

Sur SW1 :
```
show port-security
show port-security interface fa0/2
```

Tu dois voir la MAC apprise.

### 3.5 Vérifier les ports désactivés

```
show ip interface brief
```

Les ports désactivés apparaissent avec le statut `administratively down`.

---

## 4. Commandes utiles à connaître

```
show running-config | section line
show users
show ssh
show crypto key mypubkey rsa
show port-security
show port-security address
```

---

## ⚠️ Pièges courants

| Problème | Solution |
|---|---|
| `crypto key generate rsa` refuse | Vérifie que `ip domain-name` est défini |
| SSH échoue depuis PC | Vérifie `ip ssh version 2` et que `transport input ssh` est sur les vty |
| "Login invalid" permanent | Vérifie que tu utilises `login local` ET que l'utilisateur existe |
| Port-security coupe tous les PC | Remets `maximum 2` ou efface avec `clear port-security sticky interface fa0/X` |

---

## ✅ Checklist phase 1

- [ ] Compte `admin` créé avec privilège 15
- [ ] Enable secret en place
- [ ] SSH v2 activé
- [ ] Telnet refusé
- [ ] Banner MOTD affiché à la connexion
- [ ] Port-security sur les ports PC
- [ ] Ports inutilisés shutdown
- [ ] Connexion SSH testée depuis un PC
- [ ] Configs sauvegardées (R1 et SW1)
