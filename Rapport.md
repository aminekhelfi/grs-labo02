# Gestion des réseaux et sécurité opérationnelle (GRS)

## Laboratoire #2 - SNMP

**Étudiant :** Amine Khelfi, Ledda Anthony
**Date :** 18.10.2026

| Machine | Rôle | Adresse IP |
|---|---|---|
| Windows 10 (`win10-GRS-A`) | Console SNMP (snmpB), Wireshark, agent SNMP | `192.168.201.100` |
| Ubuntu (`grs-srv`) | Agent SNMP (snmpd) | `192.168.201.101` |
| Cisco CSR 1000V | Agent SNMP | `192.168.201.102` |

---

# 2.2 Configuration des agents en SNMP v2c

## 2.2.1 Poste de travail Windows

### Question 1

Configuration du service SNMP Windows (onglet *Sécurité*) :

- ajout de la communauté `grs-read-only` avec le droit **LECTURE SEULE** ;
- paquets SNMP acceptés uniquement depuis `localhost`, puisque snmpB tourne sur la même machine.

Le service SNMP est ensuite redémarré pour appliquer la configuration.

![Q1 - Configuration du service SNMP Windows](Q1.png)

### Question 2

Interrogation de l'agent SNMP local (profil `localhost`) avec snmpB :

| Objet | OID | Valeur retournée |
|---|---|---|
| sysDescr | 1.3.6.1.2.1.1.1 | Hardware: Intel64 Family 6 Model 154 Stepping 4 AT/AT COMPA... |
| sysName | 1.3.6.1.2.1.1.5 | win10-GRS-A |
| sysUpTime | 1.3.6.1.2.1.1.3 | 0:07:55.94 |
| ifNumber | 1.3.6.1.2.1.2.1 | 28 |

![Q2 - sysDescr](Q2-1.png)

![Q2 - sysName](Q2-2.png)

![Q2 - sysUpTime](Q2-3.png)

![Q2 - ifNumber](Q2-4.png)

Adresse IP de la machine Windows : `192.168.201.100`

![Q2 - ipconfig](Q2-5.png)

## 2.2.2 Routeur Cisco

### Question 3

```bash
enable
configure terminal

! --- SNMP v2c ---
snmp-server community ciscoRO RO
snmp-server community ciscoRW RW
snmp-server location HEIG-VD Labo GRS
snmp-server contact amine.khelfi@heig-vd.ch

! --- Traps vers snmpB (Windows 192.168.201.100) ---
snmp-server host 192.168.201.100 version 2c ciscoRO
snmp-server enable traps snmp authentication linkdown linkup coldstart warmstart
snmp-server enable traps config

! --- Horodatage à la milliseconde, en heure locale ---
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec localtime show-timezone

! --- Synchronisation du temps (NTP METAS) ---
clock timezone CET 1 0
clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
ntp server 195.176.26.205

end
write memory
```

### Question 4

Interrogation du routeur Cisco (profil `Cisco`, SNMPv2c) avec snmpB :

| Objet | OID | Valeur retournée |
|---|---|---|
| sysDescr | 1.3.6.1.2.1.1.1 | Cisco IOS Software [Amsterdam], Virtual XE Software (X8... |
| sysName | 1.3.6.1.2.1.1.5 | GRS_rtr |
| sysUpTime | 1.3.6.1.2.1.1.3 | 7:00:43.72 |
| sysObjectID | 1.3.6.1.2.1.1.2 | enterprises.9.1.1537 |

![Q4 - sysDescr](Q4-1.png)

![Q4 - sysName](Q4-2.png)

![Q4 - sysUpTime](Q4-3.png)

![Q4 - sysObjectID](Q4-4.png)

Adresse IP du routeur (`show ip interface brief`) : `192.168.201.102` sur GigabitEthernet1.

![Q4 - show ip interface brief](Q4-5.png)

### Question 5

`sysObjectID` est l'identifiant du type d'équipement, attribué par le constructeur. Il indique à la console SNMP **quel modèle de matériel ou de logiciel** elle gère (« what kind of box »). Cette valeur est un OID de la branche privée `enterprises` (`1.3.6.1.4.1`). Dans cette branche, chaque constructeur reçoit de l'IANA un numéro unique, sous lequel il organise ses propres identifiants.

La valeur retournée, `enterprises.9.1.1537`, correspond à l'OID complet `1.3.6.1.4.1.9.1.1537` :

| Partie | Signification |
|---|---|
| `1.3.6.1.4.1` | iso.org.dod.internet.private.enterprises |
| `9` | Numéro d'entreprise IANA attribué à **Cisco Systems** |
| `1` | Branche `ciscoProducts` (identifiants des modèles Cisco) |
| `1537` | Identifiant du modèle précis de l'équipement |

snmpB ne sait traduire que le début de l'OID (`enterprises`), car il ne connaît que les MIBs standards. Pour interpréter la valeur entièrement, il manque la **MIB privée de Cisco** qui définit cette branche : `CISCO-PRODUCTS-MIB`, avec sa dépendance `CISCO-SMI`. Une fois ces MIBs chargées dans snmpB, `1.3.6.1.4.1.9.1.1537` s'affiche sous la forme `ciscoCSR1000v`, c'est-à-dire le routeur virtuel Cisco CSR 1000V. Cela concorde avec le sysDescr (« Virtual XE Software »).

### Question 6

Requête GET de `sysName` envoyée au routeur avec snmpB. L'échange se compose de deux trames SNMP (trames 9 et 10) :

![Q6 - Capture Wireshark de l'échange GET](Q6.png)

**Trame 9 : get-request (Windows → Cisco)**

![Q6 - get-request](Q6-1.png)

- **Transport :** UDP, de `192.168.201.100:53635` vers `192.168.201.102:161`. Le port 161 est celui de l'agent SNMP.
- **version :** `v2c (1)`.
- **community :** `ciscoRO`, transmise **en clair**. On la lit directement dans la vue hexadécimale.
- **PDU :** `get-request (0)`.
- **request-id :** `6558`. Cet identifiant permet d'associer la réponse à la requête.
- **error-status / error-index :** `0`, toujours nuls dans une requête.
- **variable-bindings :** un seul objet, `1.3.6.1.2.1.1.5.0` (sysName.0, le `.0` désignant l'instance d'un objet scalaire), avec la valeur `Null`. La console demande la valeur et n'en fournit donc aucune.

**Trame 10 : get-response (Cisco → Windows)**

![Q6 - get-response](Q6-2.png)

- **Transport :** UDP, de `192.168.201.102:161` vers `192.168.201.100:53635`. La réponse revient sur le port source de la requête.
- **community :** `ciscoRO`, recopiée par l'agent.
- **PDU :** `get-response (2)`, avec le même **request-id** `6558` que la requête.
- **error-status :** `noError (0)`, la requête a abouti.
- **variable-bindings :** `1.3.6.1.2.1.1.5.0` = `"GRS_rtr"` (OctetString), soit le nom actuel du routeur.
- Temps de réponse : environ **2 ms**.

Les trames ARP qui précèdent servent à résoudre l'adresse MAC de la passerelle. Elles ne font pas partie de l'échange SNMP.

### Question 7

Modification de `sysName` avec snmpB (opération *Set*). La requête utilise la communauté en écriture `ciscoRW` :

![Q7 - Capture Wireshark de l'échange SET](Q7.png)

**Trame 5 : set-request (Windows → Cisco)**

![Q7 - set-request](Q7-1.png)

- **Transport :** UDP, de `192.168.201.100:53635` vers `192.168.201.102:161`.
- **version :** `v2c (1)`.
- **community :** `ciscoRW`, la communauté **en écriture**, elle aussi **en clair**. Toute personne qui capture cette trame peut ensuite modifier la configuration du routeur.
- **PDU :** `set-request (3)`, avec le **request-id** `6559`.
- **variable-bindings :** `1.3.6.1.2.1.1.5.0` = `"routeur-amine"` (OctetString). Contrairement au GET, la requête contient la **nouvelle valeur** à écrire.

**Trame 6 : get-response (Cisco → Windows)**

![Q7 - get-response](Q7-2.png)

- **PDU :** `get-response (2)`. SNMPv2 utilise le même type de PDU pour répondre à un GET et à un SET.
- **request-id :** `6559`, identique à celui de la requête.
- **error-status :** `noError (0)`, la modification est acceptée.
- **variable-bindings :** `sysName.0` = `"routeur-amine"`. L'agent renvoie la valeur écrite pour confirmer l'opération.
- Temps de réponse : environ **0,6 s**, nettement plus long que pour le GET, puisque l'agent modifie la configuration.

> **Remarque :** par erreur, le routeur a été nommé `routeur-amine` au lieu de `router-amine` comme demandé dans l'énoncé. Cela ne change rien à l'échange observé : seule la valeur de la chaîne transmise diffère.

**Trame 7 : snmpV2-trap (Cisco → Windows, port 162)**

![Q7 - trap de changement de configuration](Q7-3.png)

Cette modification de la configuration déclenche immédiatement une trap `ciscoConfigManEvent` (`1.3.6.1.4.1.9.9.43.2.0.1`, activée par `snmp-server enable traps config`). Le routeur informe ainsi la console qu'un changement de configuration a eu lieu.

### Question 8

Pour générer des traps, une interface Loopback est créée puis désactivée et réactivée :

```bash
configure terminal
interface Loopback1
 ip address 1.1.1.1 255.255.255.255
 shutdown
 no shutdown
end
```

Ces commandes génèrent plusieurs traps, reçues par snmpB (onglet *Traps*) :

| No | Notification | Événement déclencheur |
|---|---|---|
| 0001 | `enterprises.9.9.43...` (ciscoConfigManEvent) | Entrée en mode configuration (`configure terminal`) |
| 0002 | `linkDown` | `shutdown` sur Loopback1 |
| 0003 | `enterprises.9.9.43...` (ciscoConfigManEvent) | Sortie du mode configuration (`end`) |
| 0004 | `linkUp` | `no shutdown` sur Loopback1 |

Toutes les traps sont en SNMPv2c et proviennent de l'agent `192.168.201.102`.

![Q8 - Traps reçues par snmpB](Q8.png)

### Question 9

Les quatre traps apparaissent dans Wireshark (filtre `snmp`), toutes envoyées de `192.168.201.102` vers `192.168.201.100` (port UDP **162**, celui du gestionnaire de traps) :

![Q9 - Traps dans Wireshark](Q9.png)

Détail de la trame 287 (trap `linkUp`) :

| Variable-binding | Valeur | Signification |
|---|---|---|
| `1.3.6.1.2.1.1.3.0` (sysUpTime.0) | 2777521 | Instant de l'événement, en centièmes de seconde depuis le démarrage de l'agent |
| `1.3.6.1.6.3.1.1.4.1.0` (snmpTrapOID.0) | `1.3.6.1.6.3.1.1.5.4` | Type de trap : **linkUp** |
| `1.3.6.1.2.1.2.2.1.1.6` (ifIndex.6) | 6 | Index de l'interface concernée |
| `1.3.6.1.2.1.2.2.1.2.6` (ifDescr.6) | `"Loopback1"` | Nom de l'interface |
| `1.3.6.1.2.1.2.2.1.3.6` (ifType.6) | 24 | `softwareLoopback` |
| `1.3.6.1.4.1.9.2.2.1.1.20.6` (locIfReason) | `"up"` | Raison du changement d'état (MIB Cisco) |

Une trap SNMPv2 n'est **pas acquittée** : aucune réponse ne suit ces trames. Pour une notification confirmée, il faudrait utiliser une *inform*. La communauté `ciscoRO` apparaît également en clair.

### Question 10

Une liste d'accès standard n'autorise que la machine Windows. Elle est ensuite associée aux deux communautés :

```bash
configure terminal
access-list 10 permit host 192.168.201.100
access-list 10 deny any log
snmp-server community ciscoRO RO 10
snmp-server community ciscoRW RW 10
end
write memory
```

- `access-list 10 permit host 192.168.201.100` autorise uniquement la console snmpB.
- `access-list 10 deny any log` refuse toute autre source et journalise les tentatives refusées. Un `deny any` implicite existe déjà, mais le rendre explicite permet d'ajouter `log`.
- Le numéro d'ACL ajouté à la fin des commandes `snmp-server community` applique ce filtrage aux requêtes SNMP utilisant ces communautés.

Vérification : `show access-lists 10` affiche le nombre de correspondances pour chaque règle.

## 2.2.3 Poste de travail Linux

### Question 11

Après l'installation de `snmpd` (`sudo apt update && sudo apt install snmpd`), la configuration se trouve dans le fichier `/etc/snmp/snmpd.conf` :

```bash
sudo nano /etc/snmp/snmpd.conf
```

Lignes ajoutées ou modifiées :

```bash
# Informations système
sysLocation    HEIG-VD Labo GRS
sysContact     amine.khelfi@heig-vd.ch

# Écouter sur toutes les interfaces (au lieu de 127.0.0.1 seulement)
#agentaddress  127.0.0.1,[::1]
agentaddress  udp:161,udp6:[::1]:161

# Vue donnant accès à tout l'arbre (la vue par défaut "systemonly" est limitée)
view   all    included   .1

# Communauté v2c en lecture seule, limitée à la VM Windows
rocommunity  grs-read-only  192.168.201.100   -V all
```

Redémarrage du service, activation au démarrage et vérification :

```bash
sudo systemctl restart snmpd
sudo systemctl enable snmpd
sudo systemctl status snmpd
```

### Question 12

Requête `sysDescr` envoyée au poste Linux depuis snmpB (profil `Ubuntu`, SNMPv2c, communauté `grs-read-only`) :

```
1: sysDescr.0 Linux grs-srv 5.15.0-194-generic #204-Ubuntu SMP ...
```

L'agent retourne le nom du système (`Linux`), le nom d'hôte (`grs-srv`) et la version du noyau Ubuntu.

![Q12 - sysDescr du poste Linux](Q12.png)

---

# 2.3 MIBs privées

### Question 13

MIBs ajoutées dans `C:\Program Files (x86)\SnmpB\mibs` puis chargées dans snmpB (onglet *Modules*) :

MIBs chargées :

- `CISCO-FLASH-MIB` : la MIB principale, qui décrit la mémoire flash ;
- ses dépendances : `CISCO-SMI`, `ENTITY-MIB`, `INET-ADDRESS-MIB`, `HCNUM-TC`, `CISCO-QOS-PIB-MIB` ;
- `CISCO-PRODUCTS-MIB`, pour traduire le `sysObjectID` (Question 5).

![Q13 - MIBs chargées dans snmpB](Q13.png)

**Démarche :** chez Cisco, les objets de la mémoire flash sont définis dans `CISCO-FLASH-MIB`. Après avoir chargé ce module dans snmpB, le champ *Requires* de l'onglet *Modules* affiche en rouge les modules qui manquent. Je les ai ajoutés un par un, jusqu'à ce qu'il n'en manque plus aucun. Les MIBs standards (`SNMPv2-SMI`, `SNMPv2-TC`, etc.) étaient déjà fournies avec snmpB. Toutes les autres viennent du dépôt officiel de Cisco : <https://github.com/cisco/cisco-mibs/tree/main>.

### Question 14

Les noms des fichiers se trouvent dans la colonne `ciscoFlashFileName` de la table `ciscoFlashFileTable`. On y accède par un *Walk* sur le chemin suivant :

`private → enterprises → cisco → ciscoMgmt → ciscoFlashMIB → ciscoFlashMIBObjects → ciscoFlashDevice → ciscoFlashPartitions → ciscoFlashFiles → ciscoFlashFileTable → ciscoFlashFileEntry → ciscoFlashFileName`

Ce chemin correspond à l'OID `1.3.6.1.4.1.9.9.10.1.1.4.2.1.1.5`.

Les 10 premiers fichiers de la mémoire flash sont :

```
-----SNMP query started-----
1: ciscoFlashFileName.1.1.2 /bootflash/
2: ciscoFlashFileName.1.1.11 /bootflash/lost+found
3: ciscoFlashFileName.1.1.12 /bootflash/csr1000v-mono-universalk9.17.03.02.SPA.pkg
4: ciscoFlashFileName.1.1.13 /bootflash/csr1000v-rpboot.17.03.02.SPA.pkg
5: ciscoFlashFileName.1.1.14 /bootflash/packages.conf
6: ciscoFlashFileName.1.1.15 /bootflash/mode_event_log
7: ciscoFlashFileName.1.1.16 /bootflash/ios_core.p7b
8: ciscoFlashFileName.1.1.17 /bootflash/trustidrootx3_ca.ca
9: ciscoFlashFileName.1.1.18 /bootflash/throughput_monitor_params
10: ciscoFlashFileName.1.1.19 /bootflash/cvac.log
```

L'index `x.y.z` se lit ainsi : numéro du périphérique flash, numéro de la partition, puis numéro du fichier.

![Q14 - Fichiers de la mémoire flash](Q14.png)

---

# 2.4 Configuration d'un agent en SNMP v3

### Question 15

L'utilisateur est créé avec la directive `createUser`. Elle se place dans le fichier persistant de snmpd, `/var/lib/snmp/snmpd.conf`, que snmpd lit au démarrage. Au premier démarrage, snmpd remplace cette ligne par les clés localisées (ligne `usmUser`), si bien que les mots de passe ne restent pas stockés en clair.

```bash
# 1. Arrêter snmpd (sinon il réécrit le fichier persistant à l'arrêt)
sudo systemctl stop snmpd

# 2. Créer l'utilisateur dans le fichier persistant
sudo nano /var/lib/snmp/snmpd.conf
```

Ligne ajoutée (authentification HMAC-SHA, chiffrement AES) :

```
createUser grs-ro-user SHA secret-password AES secret-key
```

Dans `/etc/snmp/snmpd.conf`, l'utilisateur reçoit un accès en lecture seule, avec le niveau de sécurité **authPriv** obligatoire :

```bash
sudo nano /etc/snmp/snmpd.conf
```

```
rouser grs-ro-user authpriv
```

Redémarrage et vérifications :

```bash
# 3. Redémarrer snmpd
sudo systemctl start snmpd
sudo systemctl status snmpd

# 4. Vérifier que createUser a été converti en clés dérivées (usmUser)
sudo grep -E "usmUser|createUser" /var/lib/snmp/snmpd.conf

# 5. Vérifier la ligne rouser
sudo grep rouser /etc/snmp/snmpd.conf

# 6. Tester localement en SNMPv3 authPriv
snmpget -v3 -l authPriv -u grs-ro-user -a SHA -A secret-password -x AES -X secret-key localhost 1.3.6.1.2.1.1.5.0
```

### Question 16

En SNMP v1/v2c, l'accès est donné par les directives de communauté (`rocommunity`, `rwcommunity`, `rocommunity6`, `com2sec`). Pour n'autoriser que SNMP v3, il suffit de toutes les désactiver. Seuls les utilisateurs USM définis par `rouser` / `rwuser` restent alors autorisés.

Recherche des communautés actives :

```bash
sudo grep -n -E "^\s*(rocommunity|rocommunity6|rwcommunity|com2sec)" /etc/snmp/snmpd.conf
```

Chaque ligne trouvée est commentée dans `/etc/snmp/snmpd.conf` :

```bash
sudo nano /etc/snmp/snmpd.conf
```

```bash
#rocommunity  grs-read-only  192.168.201.100  -V all
#rocommunity  public  default    -V systemonly
#rocommunity6 public  default    -V systemonly
```

La ligne de l'utilisateur SNMPv3 est conservée :

```
rouser grs-ro-user authpriv
```

Redémarrage et vérification :

```bash
sudo systemctl restart snmpd

# Une requête v2c ne reçoit plus de réponse (timeout)
snmpget -v2c -c grs-read-only localhost 1.3.6.1.2.1.1.5.0

# Une requête v3 authPriv fonctionne toujours
snmpget -v3 -l authPriv -u grs-ro-user -a SHA -A secret-password -x AES -X secret-key localhost 1.3.6.1.2.1.1.5.0
```

Dans snmpB, le profil USM `grs-ro-user` est créé (SHA / `secret-password`, AES128 / `secret-key`). Le profil `Ubuntu` passe ensuite en SNMPv3, avec ce *Security Name* et le niveau `authPriv`.

### Question 17

Requête `sysName` envoyée au poste Linux en SNMPv3 (authPriv) depuis snmpB :

![Q17 - Capture Wireshark SNMPv3](Q17.png)

**Trame 2 : requête (Windows → Linux)**

![Q17 - Requête SNMPv3](Q17-1.png)

**Trame 3 : réponse (Linux → Windows)**

![Q17 - Réponse SNMPv3](Q17-2.png)

**Changements importants par rapport à SNMPv2c, du point de vue de la sécurité :**

- **Plus de communauté en clair.** Le champ `community` est remplacé par un utilisateur USM (`msgUserName: grs-ro-user`). Il n'y a plus de secret lisible dans la capture.
- **PDU chiffrée (AES).** Le contenu du message est remplacé par un `encryptedPDU` illisible, et Wireshark affiche `privKey Unknown`. On ne voit plus ni l'opération, ni l'OID, ni la valeur retournée.
- **Authentification et intégrité (HMAC-SHA).** Les `msgFlags` indiquent `Authenticated` et `Encrypted`, ce qui correspond au niveau authPriv. Le champ `msgAuthenticationParameters` contient le code HMAC : un message modifié ou forgé est rejeté.
- **Anti-rejeu.** Les champs `EngineBoots` et `EngineTime` permettent de refuser les messages trop anciens.

Restent visibles : les adresses IP, les ports, le nom d'utilisateur et l'EngineID.
