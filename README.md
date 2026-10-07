# Réseau d'une PME sur 2 sites (GNS3)

Projet personnel de conception et de mise en place d'une infrastructure réseau d'entreprise, entièrement virtualisée avec GNS3 : un siège et une agence reliés par un lien WAN, avec segmentation en VLAN et routage dynamique entre les sites.

> **État d'avancement** : les deux sites sont opérationnels (VLAN, passerelles, routage inter-VLAN), reliés par OSPF, et sortent sur Internet par un NAT au siège. Les configurations survivent à un redémarrage des routeurs. DHCP, DNS et ACL sont à venir (voir la section « Suite du projet »).

---

## 1. Objectifs

- Concevoir une architecture réseau simple et cohérente pour une petite entreprise sur deux sites
- Segmenter le réseau du siège par service (Direction, Compta/RH, Serveurs) grâce aux VLAN
- Faire communiquer les VLAN entre eux via un routeur
- Relier le siège et l'agence par un lien WAN et mettre en place le routage dynamique (OSPF)
- Donner un accès à Internet aux deux sites par un seul point de sortie (NAT au siège)
- Rendre les configurations persistantes après un redémarrage
- Documenter chaque étape, avec les tests et les problèmes rencontrés

## 2. Environnement

| Élément | Détail |
|---|---|
| Poste hôte | Windows 11, 16 Go de RAM, processeur à 12 threads |
| Simulateur | GNS3 2.2.61 |
| Hyperviseur | VirtualBox 7.2.20 |
| Serveur GNS3 | GNS3 VM (4 vCPU, 4 Go de RAM), accessible en `192.168.56.101` |
| Routeurs | FRRouting 8.2.2 (images QEMU, Alpine Linux 3.16) |
| Commutateurs | Ethernet switch intégré à GNS3 (VLAN 802.1Q) |
| Postes et serveur | VPCS (PC virtuels légers) |

## 3. Topologie

Équipements :

- `Siege-R1` : routeur du siège
- `Agence-R2` : routeur de l'agence
- `Switch1` (siège) et `Switch2` (agence)
- `NAT1` : nœud NAT de GNS3, qui simule l'accès à Internet, relié à Siege-R1
- Siège : `PC-Direction`, `PC-Compta`, `SRV-1`
- Agence : `PC-Agence1`, `PC-Invite`

![Topologie du projet](captures/11-topologie-cablee.png)

Câblage :

| De | Interface | Vers | Interface |
|---|---|---|---|
| Siege-R1 | eth0 | Agence-R2 | eth0 (lien WAN) |
| Siege-R1 | eth1 | Switch1 | Ethernet0 (trunk) |
| Siege-R1 | eth2 | NAT1 | nat0 (sortie Internet) |
| Switch1 | Ethernet1 | PC-Direction | Ethernet0 |
| Switch1 | Ethernet2 | PC-Compta | Ethernet0 |
| Switch1 | Ethernet3 | SRV-1 | Ethernet0 |
| Agence-R2 | eth1 | Switch2 | Ethernet0 (trunk) |
| Switch2 | Ethernet1 | PC-Agence1 | Ethernet0 |
| Switch2 | Ethernet2 | PC-Invite | Ethernet0 |

## 4. Plan d'adressage

**Lien WAN** : `10.0.0.0/30` (Siege-R1 `10.0.0.1`, Agence-R2 `10.0.0.2`)

**Sortie Internet** : `eth2` de Siege-R1, adresse obtenue en DHCP auprès du nœud NAT de GNS3 (réseau `192.168.42.0/24`, passerelle `192.168.42.1`)

**Siège**

| VLAN | Usage | Réseau | Passerelle | Poste |
|---|---|---|---|---|
| 10 | Direction | 192.168.10.0/24 | 192.168.10.1 | PC-Direction : 192.168.10.10 |
| 20 | Compta / RH | 192.168.20.0/24 | 192.168.20.1 | PC-Compta : 192.168.20.10 |
| 30 | Serveurs | 192.168.30.0/24 | 192.168.30.1 | SRV-1 : 192.168.30.10 |
| 99 | Management | 192.168.99.0/24 | 192.168.99.1 | (prévu, non encore utilisé) |

**Agence**

| VLAN | Usage | Réseau | Passerelle | Poste |
|---|---|---|---|---|
| 10 | Utilisateurs | 192.168.110.0/24 | 192.168.110.1 | PC-Agence1 : 192.168.110.10 |
| 20 | Invités | 192.168.120.0/24 | 192.168.120.1 | PC-Invite : 192.168.120.10 |

Identifiants de routeur OSPF : `1.1.1.1` pour Siege-R1, `2.2.2.2` pour Agence-R2.

## 5. Mise en œuvre

### 5.1 Préparation de l'environnement

1. Installation de GNS3 et de VirtualBox, import de la GNS3 VM, installation de l'appareil FRR 8.2.2 depuis le catalogue GNS3.
2. Désactivation de l'accélération KVM, indisponible sur ce poste (voir la section « Problèmes rencontrés »).

![Interface GNS3 avec la VM connectée](captures/05-gns3-interface.png)

### 5.2 Lien WAN entre les deux routeurs

Configuration de `eth0` sur chaque routeur (`10.0.0.1/30` et `10.0.0.2/30`), puis test de connectivité.

```
configure terminal
interface eth0
ip address 10.0.0.1/30
no shutdown
exit
exit
write memory
```

![Configuration de Siege-R1](captures/09-config-frr1.png)
![Configuration d'Agence-R2](captures/09-config-frr2.png)
![Ping entre les deux routeurs](captures/10-ping-r1-r2.png)

### 5.3 VLAN sur le commutateur du siège

Sur `Switch1` : le port 0 est en mode trunk (`dot1q`) vers le routeur, les ports 1, 2 et 3 sont en mode accès dans les VLAN 10, 20 et 30.

| Port | VLAN | Type | Connecté à |
|---|---|---|---|
| 0 | 1 | dot1q | Siege-R1 |
| 1 | 10 | access | PC-Direction |
| 2 | 20 | access | PC-Compta |
| 3 | 30 | access | SRV-1 |

![VLAN du Switch1](captures/12-vlan-switch1.png)

### 5.4 Routage inter-VLAN du siège (« router on a stick »)

Un seul câble relie le routeur au commutateur. Il est découpé en sous-interfaces, une par VLAN, qui servent de passerelles. Les sous-interfaces VLAN sont créées au niveau Linux du routeur, puis adressées dans FRR.

```
# Dans le shell Linux du routeur
ip link add link eth1 name eth1.10 type vlan id 10
ip link add link eth1 name eth1.20 type vlan id 20
ip link add link eth1 name eth1.30 type vlan id 30
ip link set eth1 up
ip link set eth1.10 up
ip link set eth1.20 up
ip link set eth1.30 up
```

```
# Dans vtysh (configuration FRR)
configure terminal
interface eth1.10
ip address 192.168.10.1/24
exit
interface eth1.20
ip address 192.168.20.1/24
exit
interface eth1.30
ip address 192.168.30.1/24
exit
exit
write memory
```

![Sous-interfaces VLAN sur Siege-R1](captures/13b-vlan-siege-r1.png)
![Configuration de Siege-R1](captures/14-config-siege-r1.png)

### 5.5 Configuration des postes du siège (VPCS)

```
PC-Direction> ip 192.168.10.10 255.255.255.0 192.168.10.1
PC-Compta>    ip 192.168.20.10 255.255.255.0 192.168.20.1
SRV-1>        ip 192.168.30.10 255.255.255.0 192.168.30.1
```

![Configuration de PC-Direction](captures/15a-config-pc-direction.png)

Sur un VPCS, la commande `save` est indispensable après chaque configuration : sans elle, l'adresse est perdue à l'arrêt du poste.

### 5.6 Site de l'agence

L'agence reprend le même principe que le siège, avec deux VLAN.

**Switch2** : le port 0 est en trunk (`dot1q`) vers Agence-R2, le port 1 en accès VLAN 10 (PC-Agence1) et le port 2 en accès VLAN 20 (PC-Invite).

| Port | VLAN | Type | Connecté à |
|---|---|---|---|
| 0 | 1 | dot1q | Agence-R2 |
| 1 | 10 | access | PC-Agence1 |
| 2 | 20 | access | PC-Invite |

![VLAN du Switch2](captures/12-vlan-switch2.png)

**Agence-R2** : sous-interfaces VLAN sur `eth1` (shell Linux), puis adresses et lien WAN dans FRR.

```
# Dans le shell Linux du routeur
ip link add link eth1 name eth1.10 type vlan id 10
ip link add link eth1 name eth1.20 type vlan id 20
ip link set eth1 up
ip link set eth1.10 up
ip link set eth1.20 up
```

```
# Dans vtysh (configuration FRR)
configure terminal
hostname Agence-R2
interface eth0
ip address 10.0.0.2/30
no shutdown
exit
interface eth1.10
ip address 192.168.110.1/24
exit
interface eth1.20
ip address 192.168.120.1/24
exit
end
write memory
```

**Postes de l'agence**

```
PC-Agence1> ip 192.168.110.10 255.255.255.0 192.168.110.1
PC-Invite>  ip 192.168.120.10 255.255.255.0 192.168.120.1
```

![Configuration d'Agence-R2](captures/19-config-agence-r2.png)
![Configuration des postes de l'agence et ping vers leurs passerelles](captures/20-ping-agence.png)

### 5.7 Routage dynamique OSPF

Les deux routeurs annoncent leurs réseaux dans l'aire 0 (backbone). Chaque routeur apprend ainsi les réseaux de l'autre site sans route statique.

```
# Siege-R1
configure terminal
router ospf
 ospf router-id 1.1.1.1
 network 10.0.0.0/30 area 0
 network 192.168.10.0/24 area 0
 network 192.168.20.0/24 area 0
 network 192.168.30.0/24 area 0
end
write memory
```

```
# Agence-R2
configure terminal
router ospf
 ospf router-id 2.2.2.2
 network 10.0.0.0/30 area 0
 network 192.168.110.0/24 area 0
 network 192.168.120.0/24 area 0
end
write memory
```

Vérification sur Siege-R1 : `show ip ospf neighbor` montre le voisin `2.2.2.2` à l'état `Full`, et `show ip route ospf` contient deux routes apprises via `10.0.0.2` (réseaux `192.168.110.0/24` et `192.168.120.0/24`).

![Voisin OSPF et routes apprises sur Siege-R1](captures/21-ospf.png)

### 5.8 Persistance des configurations

**Le problème.** Après une fermeture de GNS3, les routeurs sont revenus à zéro : plus de sous-interfaces VLAN, plus d'adresses, plus d'OSPF. Il y avait deux causes :

- Les **sous-interfaces VLAN** créées avec `ip link` sont gérées par Linux, qui ne les conserve pas après un redémarrage.
- La **configuration FRR** (adresses, OSPF) n'est conservée que si elle a été enregistrée avec `write memory`. Elle est alors écrite dans `/etc/frr/frr.conf`.

**La solution.** Alpine Linux exécute au démarrage les scripts présents dans `/etc/local.d/`. Sur chaque routeur, un script `vlan.start` recrée les sous-interfaces. Exemple pour Siege-R1 (Agence-R2 n'a que les VLAN 10 et 20) :

```
mkdir -p /etc/local.d
cat > /etc/local.d/vlan.start << 'EOF'
#!/bin/sh
ip link set eth1 up
ip link add link eth1 name eth1.10 type vlan id 10
ip link add link eth1 name eth1.20 type vlan id 20
ip link add link eth1 name eth1.30 type vlan id 30
ip link set eth1.10 up
ip link set eth1.20 up
ip link set eth1.30 up
EOF
chmod +x /etc/local.d/vlan.start
rc-update add local default
```

La configuration FRR est ensuite enregistrée avec `write memory` dans vtysh, puis `sync` dans le shell Linux pour forcer l'écriture sur le disque.

![Script de démarrage vlan.start](captures/23-persistance-script-vlan.png)

**Vérification.** Chaque routeur a été arrêté puis relancé : les sous-interfaces reviennent toutes seules, la configuration FRR est rechargée et OSPF repasse à l'état `Full`.

![Interfaces et voisin OSPF après redémarrage](captures/24-apres-redemarrage.png)

**Sauvegarde dans le dépôt.** Les configurations sont copiées dans le dossier `configs/` :

| Fichier | Contenu |
|---|---|
| `siege-r1-frr.conf` | Configuration FRR de Siege-R1 |
| `siege-r1-vlan.start` | Script de démarrage des sous-interfaces VLAN de Siege-R1 |
| `siege-r1-nat.start` | Script de démarrage du NAT de Siege-R1 (adresse DHCP et règle `iptables`) |
| `agence-r2-frr.conf` | Configuration FRR d'Agence-R2 |
| `agence-r2-vlan.start` | Script de démarrage d'Agence-R2 |

### 5.9 NAT et accès à Internet

Un seul point de sortie vers Internet, au siège, est utilisé par les deux sites, comme dans une vraie PME.

**1. Nœud NAT dans GNS3.** Le nœud `NAT1` est relié à `eth2` de Siege-R1. Il joue le rôle de box Internet et distribue une adresse en DHCP.

**2. Adresse et NAT sur Siege-R1.** Sous Linux, `udhcpc` récupère l'adresse et la route par défaut, et `iptables` masque les adresses privées derrière celle de `eth2` :

```
ip link set eth2 up
udhcpc -i eth2
iptables -t nat -A POSTROUTING -o eth2 -j MASQUERADE
```

**3. Persistance.** Comme pour les VLAN, ces commandes sont mises dans un script de démarrage `/etc/local.d/nat.start`, que le service `local` exécute à chaque boot. Le test `iptables -C` évite d'ajouter la règle en double :

```
#!/bin/sh
ip link set eth2 up
udhcpc -i eth2 -b
iptables -t nat -C POSTROUTING -o eth2 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -o eth2 -j MASQUERADE
```

![Adresse, route par défaut et règle NAT sur Siege-R1](captures/25-nat-siege-r1.png)

**4. Route par défaut vers l'agence.** Siege-R1 annonce sa route par défaut en OSPF, et Agence-R2 l'apprend automatiquement :

```
router ospf
 default-information originate
```

Sur Agence-R2, `show ip route` contient alors `O>* 0.0.0.0/0 via 10.0.0.1`.

![Route par défaut apprise en OSPF sur Agence-R2](captures/26-route-defaut-agence.png)

**5. Tests.** Un poste du siège (SRV-1) et un poste de l'agence (PC-Agence1) joignent `8.8.8.8`. Sur Siege-R1, les compteurs de la règle `MASQUERADE` augmentent à chaque ping, ce qui prouve que le NAT traite bien les paquets des réseaux privés. Le NAT a aussi été vérifié après un redémarrage de Siege-R1.

![Ping vers Internet depuis SRV-1](captures/27-ping-internet-srv1.png)
![Ping vers Internet depuis PC-Agence1 et compteurs du NAT](captures/28-ping-internet-agence.png)

## 6. Tests et validation

| Test | Résultat |
|---|---|
| Siege-R1 vers Agence-R2 (`10.0.0.2`) | OK, environ 2 ms |
| PC-Direction vers sa passerelle (`192.168.10.1`) | OK, TTL 64 |
| PC-Direction vers PC-Compta (`192.168.20.10`) | OK, TTL 63 |
| PC-Direction vers SRV-1 (`192.168.30.10`) | OK, TTL 63 |
| SRV-1 vers sa passerelle (`192.168.30.1`) | OK, TTL 64 |
| SRV-1 vers PC-Agence1 (`192.168.110.10`) | OK, TTL 62 |
| SRV-1 vers PC-Invite (`192.168.120.10`) | OK, TTL 62 |
| Voisin OSPF `2.2.2.2` depuis Siege-R1 | État `Full` |
| Siege-R1 vers `8.8.8.8` et `google.com` | OK (la résolution de nom fonctionne) |
| SRV-1 vers `8.8.8.8` | OK, TTL 60 |
| PC-Agence1 vers `8.8.8.8` | OK, TTL 58 |
| Route par défaut `0.0.0.0/0` sur Agence-R2 | Apprise en OSPF via `10.0.0.1` |
| Compteurs de la règle `MASQUERADE` | Augmentent à chaque ping des postes (siège et agence) |
| Tous les tests après redémarrage des routeurs | OK |

Le TTL de 64 vers la passerelle puis de 63 vers un autre VLAN montre que les paquets traversent un routeur : le routage entre VLAN fonctionne. Le TTL de 62 vers l'agence montre qu'ils en traversent deux (Siege-R1 puis Agence-R2) : le routage entre les deux sites fonctionne.

![Pings de PC-Direction vers sa passerelle et les autres VLAN](captures/16-routage-inter-vlan.png)
![Ping du siège vers l'agence](captures/22-ping-siege-agence.png)

## 7. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Les routeurs refusaient de démarrer (« KVM acceleration cannot be used ») | Pas d'accélération matérielle KVM dans la GNS3 VM | Ajout de `enable_kvm = False` dans la section `[Qemu]` de `/opt/gns3/server/gns3_server.conf`, puis redémarrage de la VM |
| Console de routeur vide ou sans réponse | Le client console par défaut ne s'ouvrait pas correctement | Activation du client Telnet de Windows et connexion avec `telnet 192.168.56.101 <port>` |
| GNS3 gelé, VM plantée (`soft lockup`) | Émulation sans KVM trop gourmande, avec plusieurs équipements démarrés en même temps | Passage de la VM à 4 vCPU, démarrage des routeurs un par un, fermeture des applications gourmandes sous Windows |
| Commande `start-shell` inconnue dans FRR | Cette version de l'appareil ne la fournit pas | Accès au shell Linux avec `exit` depuis vtysh, retour avec la commande `vtysh` |
| Configuration des routeurs perdue après la fermeture de GNS3 | Sous-interfaces VLAN non persistantes et configuration FRR non enregistrée | Scripts de démarrage `/etc/local.d/vlan.start` et `write memory` (voir 5.8) |
| `frr.conf` vide après un arrêt brutal de la GNS3 VM | Le fichier n'avait pas été écrit sur le disque avant le plantage | Restauration depuis la copie de secours `frr.conf.sav` créée par FRR, puis `write memory` et `sync` |
| « Address already in use » sur le port de console d'un routeur | Processus bloqué dans la GNS3 VM figée, qui gardait le port occupé | Arrêt propre de la GNS3 VM puis redémarrage, et démarrage des routeurs un par un |
| Console d'un routeur qui se ferme en boucle, port 5000 injoignable en `telnet` | Le serveur GNS3 ne relançait plus correctement la console du nœud | Fermeture puis relance de GNS3, qui a rétabli la console. Un `ping` de la GNS3 VM permet de vérifier qu'elle répond |
| Voisin OSPF absent sur Agence-R2, route par défaut non reçue | Siege-R1 n'était pas démarré, il n'y avait donc personne avec qui échanger | Démarrage de Siege-R1 : voisin `Full` et route `0.0.0.0/0` reçue au bout de quelques minutes |

## 8. Limites connues

- L'absence de KVM ralentit fortement le démarrage des routeurs (plusieurs minutes chacun).
- Le nom d'hôte FRR repasse à `frr` après un redémarrage. C'est cosmétique et sans effet sur le réseau.
- Les scripts de démarrage doivent être créés sur chaque nouveau routeur.
- Les arrêts brutaux de la GNS3 VM peuvent corrompre un fichier : il faut arrêter les nœuds avec **Stop**, enregistrer le projet (Ctrl+S), puis arrêter la VM proprement.

## 9. Suite du projet

- [x] Agence : sous-interfaces VLAN sur Agence-R2, configuration de Switch2 et des postes
- [x] Routage dynamique OSPF entre les deux sites
- [x] Rendre les VLAN persistants au démarrage
- [x] NAT vers un accès « Internet » simulé, avec route par défaut diffusée en OSPF
- [ ] DHCP et DNS
- [ ] Filtrage entre VLAN avec des ACL
- [ ] VPN site à site
- [ ] Supervision et sauvegarde automatisée des configurations (sauvegarde manuelle déjà faite dans `configs/`)
