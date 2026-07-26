# Projet : Interconnexion de trois entreprises avec BGP sécurisé par un pare-feu

## Contexte

Trois entreprises souhaitent être interconnectées via Internet afin d'échanger des données
de manière sécurisée. Chaque entreprise possède son propre Système Autonome (AS) et son
propre réseau local (LAN). Toutes les communications entre les entreprises passent par un
pare-feu central qui applique des règles de sécurité.

Ce laboratoire a été réalisé dans **GNS3**, avec des routeurs Cisco IOS.

## Objectifs visés

- [x] Configurer le routage BGP
- [x] Établir des voisinages eBGP
- [x] Annoncer des préfixes réseau
- [x] Vérifier la table BGP
- [x] Configurer un pare-feu
- [x] Autoriser uniquement certains flux
- [x] Bloquer le trafic non autorisé
- [x] Mettre en place le NAT (configuration validée, test externe limité — voir section 7)
- [x] Diagnostiquer des problèmes de routage et de sécurité (démontré à plusieurs reprises, voir section 8)

## 1. Architecture du réseau

Trois entreprises (A, B, C), chacune avec son propre AS et son propre LAN, se connectent à
un routeur central **FW** qui fait à la fois du transit BGP entre les trois AS et du
filtrage/pare-feu. FW est également relié à un nœud représentant l'accès Internet, pour le NAT.

```
                LAN A                                   LAN B
          192.168.10.0/24                          192.168.20.0/24
                 |                                         |
              [R-A]  AS100                            [R-B]  AS200
                 \                                         /
       172.16.1.0/30 \                                    / 172.16.2.0/30
                       \                                  /
                        \                                /
                         [ FW ]  AS65000  (pare-feu + BGP + NAT)
                        /        \
       172.16.3.0/30  /            \  Internet (Cloud/NAT)
                      /              \
                 [R-C]  AS300      [NAT1 / Cloud]
                    |
                 LAN C
           192.168.30.0/24
```

## 2. Plan d'adressage final

| Équipement | Interface | Rôle | Adresse IP |
|---|---|---|---|
| R-A (AS100) | Gi1/0 | LAN Entreprise A | 192.168.10.1/24 |
| R-A | Gi0/0 | Lien vers FW | 172.16.1.1/30 |
| R-A | Loopback0 | Serveur simulé A | 1.1.1.1/32 |
| R-B (AS200) | Gi0/0 | LAN Entreprise B | 192.168.20.1/24 |
| R-B | Gi1/0 | Lien vers FW | 172.16.2.1/30 |
| R-B | Loopback0 | Serveur simulé B | 2.2.2.2/32 |
| R-C (AS300) | Gi0/0 | LAN Entreprise C | 192.168.30.1/24 |
| R-C | Gi1/0 | Lien vers FW | 172.16.3.1/30 |
| R-C | Loopback0 | Serveur simulé C | 3.3.3.3/32 |
| FW (AS65000) | Gi0/0 | Vers R-A | 172.16.1.2/30 |
| FW | Gi1/0 | Vers R-B | 172.16.2.2/30 |
| FW | Gi2/0 | Vers R-C | 172.16.3.2/30 |
| FW | Gi3/0 | Vers Internet (NAT) | 192.168.122.10/24 (imposé par le nœud NAT GNS3) |
| PC-A | - | Poste de test LAN A | 192.168.10.10 (GW .1) |
| PC-B | - | Poste de test LAN B | 192.168.20.10 (GW .1) |
| PC-C | - | Poste de test LAN C | 192.168.30.10 (GW .1) |

> ⚠️ Remarque importante : sur R-A, l'interface LAN (Gi1/0) et l'interface WAN (Gi0/0) sont
> inversées par rapport au plan initial, à cause d'un câblage physique différent dans GNS3
> (voir section 8, incident n°1). C'est un exemple concret de la différence entre "adresse
> configurée" et "port réellement câblé".

## 3. Configuration BGP

Chaque entreprise (AS 100, 200, 300) établit une session **eBGP** avec le routeur central FW
(AS 65000), qui joue le rôle de fournisseur de transit entre les trois AS.

**R-A**
```
router bgp 100
 bgp log-neighbor-changes
 neighbor 172.16.1.2 remote-as 65000
 network 192.168.10.0 mask 255.255.255.0
 network 1.1.1.1 mask 255.255.255.255
```

**R-B**
```
router bgp 200
 bgp log-neighbor-changes
 neighbor 172.16.2.2 remote-as 65000
 network 192.168.20.0 mask 255.255.255.0
 network 2.2.2.2 mask 255.255.255.255
```

**R-C**
```
router bgp 300
 bgp log-neighbor-changes
 neighbor 172.16.3.2 remote-as 65000
 network 192.168.30.0 mask 255.255.255.0
 network 3.3.3.3 mask 255.255.255.255
```

**FW**
```
router bgp 65000
 bgp log-neighbor-changes
 neighbor 172.16.1.1 remote-as 100
 neighbor 172.16.2.1 remote-as 200
 neighbor 172.16.3.1 remote-as 300
```

### Vérification (extrait réel)

```
FW#show ip bgp summary
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.16.1.1      4          100       6      10       19    0    0 00:01:41        2
172.16.2.1      4          200       6       8       19    0    0 00:01:42        2
172.16.3.1      4          300       6       8       19    0    0 00:01:42        2
```

Les trois voisinages sont `Established`, avec 2 préfixes reçus de chaque entreprise. La table
BGP de R-A confirme le mécanisme de transit : FW relaie automatiquement à R-A les préfixes
appris des AS 200 et 300 (AS-PATH `65000 200` et `65000 300`).

## 4. Configuration du pare-feu (ACL étendues nommées)

Politique de sécurité appliquée :
- **A ↔ B** : autorisé en ICMP et HTTP/HTTPS (80/443), dans les deux sens
- **A → C** : autorisé en ICMP uniquement, dans ce sens seulement
- **C → A** initié par C : bloqué (seule la réponse au ping de A est permise)
- **B ↔ C** : bloqué (non prévu par la politique)
- Trafic BGP (TCP port 179) entre chaque routeur et FW : explicitement autorisé (voir incident n°2)
- Trafic vers Internet (`192.168.122.0/24`) : explicitement autorisé pour permettre le NAT

```
ip access-list extended ACL-FROM-A
 1  permit tcp host 172.16.1.1 host 172.16.1.2 eq bgp
 2  permit tcp host 172.16.1.1 eq bgp host 172.16.1.2
 10 permit icmp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 20 permit tcp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 eq www
 30 permit tcp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 eq 443
 40 permit icmp 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255 echo
 41 permit icmp 192.168.10.0 0.0.0.255 192.168.122.0 0.0.0.255
 42 permit tcp 192.168.10.0 0.0.0.255 192.168.122.0 0.0.0.255 eq 80
 43 permit tcp 192.168.10.0 0.0.0.255 192.168.122.0 0.0.0.255 eq 443
 50 deny   ip any any log

ip access-list extended ACL-FROM-B
 1  permit tcp host 172.16.2.1 host 172.16.2.2 eq bgp
 2  permit tcp host 172.16.2.1 eq bgp host 172.16.2.2
 10 permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 20 permit tcp 192.168.20.0 0.0.0.255 eq www 192.168.10.0 0.0.0.255
 30 permit tcp 192.168.20.0 0.0.0.255 eq 443 192.168.10.0 0.0.0.255
 31 permit icmp 192.168.20.0 0.0.0.255 192.168.122.0 0.0.0.255
 32 permit tcp 192.168.20.0 0.0.0.255 192.168.122.0 0.0.0.255 eq 80
 33 permit tcp 192.168.20.0 0.0.0.255 192.168.122.0 0.0.0.255 eq 443
 40 deny   ip any any log

ip access-list extended ACL-FROM-C
 1  permit tcp host 172.16.3.1 host 172.16.3.2 eq bgp
 2  permit tcp host 172.16.3.1 eq bgp host 172.16.3.2
 10 permit icmp 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 11 permit icmp 192.168.30.0 0.0.0.255 192.168.122.0 0.0.0.255
 12 permit tcp 192.168.30.0 0.0.0.255 192.168.122.0 0.0.0.255 eq 80
 13 permit tcp 192.168.30.0 0.0.0.255 192.168.122.0 0.0.0.255 eq 443
 20 deny   ip any any log

interface GigabitEthernet0/0
 ip access-group ACL-FROM-A in
interface GigabitEthernet1/0
 ip access-group ACL-FROM-B in
interface GigabitEthernet2/0
 ip access-group ACL-FROM-C in
```

### Résultats de tests (compteurs `show access-lists`, extrait réel)

| Règle | Matches observés | Interprétation |
|---|---|---|
| `ACL-FROM-A` ligne 10 (ICMP A→B) | 10 | Ping PC-A → PC-B autorisé ✅ |
| `ACL-FROM-B` ligne 10 (ICMP B→A) | 10 | Ping PC-B → PC-A autorisé ✅ |
| `ACL-FROM-A` ligne 40 (ICMP echo A→C) | 5 | Ping PC-A → PC-C autorisé ✅ |
| `ACL-FROM-C` ligne 10 (echo-reply C→A) | 5 | Réponse de C au ping de A autorisée ✅ |
| `ACL-FROM-B` deny final | +5 | Tentative PC-B → PC-C bloquée ✅ |
| `ACL-FROM-C` deny final | +10 | Tentatives PC-C → PC-A et PC-C → PC-B bloquées ✅ |

Chaque flux se comporte exactement comme prévu par la politique de sécurité.

## 5. Routes par défaut

Pour permettre au trafic non reconnu par BGP (comme l'accès Internet) de sortir vers FW,
une route par défaut a été ajoutée sur chaque routeur d'entreprise :

```
ip route 0.0.0.0 0.0.0.0 172.16.1.2   (sur R-A)
ip route 0.0.0.0 0.0.0.0 172.16.2.2   (sur R-B)
ip route 0.0.0.0 0.0.0.0 172.16.3.2   (sur R-C)
```

## 6. Configuration NAT

```
ip access-list standard NAT-INSIDE
 permit 192.168.10.0 0.0.0.255
 permit 192.168.20.0 0.0.0.255
 permit 192.168.30.0 0.0.0.255

ip nat inside source list NAT-INSIDE interface GigabitEthernet3/0 overload

interface GigabitEthernet0/0
 ip nat inside
interface GigabitEthernet1/0
 ip nat inside
interface GigabitEthernet2/0
 ip nat inside
interface GigabitEthernet3/0
 ip nat outside
```

Vérifié fonctionnel via `show ip nat statistics` : les 3 interfaces LAN sont bien déclarées
`inside`, Gi3/0 est `outside`, et le mapping dynamique par PAT (overload) est enregistré.

## 7. Limite rencontrée : test externe du NAT

Le nœud GNS3 utilisé pour simuler l'accès Internet (**NAT1**, nœud spécial "NAT" de GNS3,
hébergé sur un serveur distant) n'a pas pu être joint, malgré une configuration IP cohérente
des deux côtés (`192.168.122.0/24`).

Le diagnostic mené (voir section 8, incident n°4) a permis d'isoler la cause : l'absence de
réponse ARP de NAT1 sur son propre lien, alors que FW répond correctement à toutes ses autres
interfaces. Cela pointe vers un problème de communication entre le serveur GNS3 exécutant FW
et le serveur distant exécutant NAT1 — une limite d'infrastructure indépendante de la
configuration réseau réalisée, plutôt qu'une erreur de conception.

**Conclusion sur ce point** : la configuration NAT est validée sur la base des commandes de
vérification internes (`show ip nat statistics`), mais le test de bout en bout vers un point
de sortie Internet réel n'a pas pu être finalisé dans cet environnement.

## 8. Journal de diagnostic (incidents rencontrés et résolus)

Cette section illustre la démarche de diagnostic suivie tout au long du projet — l'un des
objectifs explicites de l'énoncé.

### Incident 1 — PC-A ne pouvait pas joindre sa passerelle R-A

- **Symptôme** : `ping 192.168.10.1` échouait depuis PC-A, alors que l'adressage IP des deux
  côtés était correct.
- **Méthode de diagnostic** : test inverse depuis le routeur (`ping 192.168.10.10` depuis R-A),
  puis vérification de la table ARP (`show arp`) — aucune entrée pour PC-A, confirmant
  l'absence de communication de niveau 2.
- **Cause réelle** : le câble physique dans GNS3 reliait PC-A au port `GigabitEthernet1/0` de
  R-A, alors que l'adresse `192.168.10.1` était configurée sur `GigabitEthernet0/0`.
- **Correction** : réattribution des adresses IP aux interfaces réellement câblées
  (Gi0/0 = WAN vers FW, Gi1/0 = LAN), plutôt que de recâbler physiquement.
- **Leçon** : toujours vérifier le port réellement utilisé par un câble (clic sur le lien dans
  GNS3) avant de supposer qu'une configuration logique correspond au câblage physique.

### Incident 2 — Les sessions BGP se coupaient après l'activation du pare-feu

- **Symptôme** : logs `%SEC-6-IPACCESSLOGP` indiquant que le trafic TCP port 179 (BGP) était
  bloqué par les ACL, juste après leur application aux interfaces de FW.
- **Cause réelle** : les ACL, pensées uniquement pour filtrer le trafic entre LAN, filtraient
  aussi le trafic de contrôle BGP entre chaque routeur et FW, car elles étaient appliquées en
  entrée sur les mêmes interfaces WAN utilisées par BGP.
- **Correction** : ajout de règles `permit tcp ... eq bgp` explicites, avec un numéro de
  séquence bas pour être évaluées avant le `deny` final.
- **Leçon** : un pare-feu placé entre deux routeurs BGP doit explicitement autoriser le trafic
  BGP entre eux, sous peine de couper le routage en même temps que le trafic filtré.

### Incident 3 — Le sous-réseau prévu pour le NAT n'avait pas assez d'adresses

- **Symptôme** : la pool NAT prévue (`/30`) ne laissait aucune adresse libre pour la
  translation, l'interface WAN occupant déjà l'une des deux adresses utilisables.
- **Correction** : élargissement du masque de l'interface concernée à un `/28`, puis, après
  connexion au nœud NAT spécial de GNS3, adoption de son adressage imposé (`192.168.122.0/24`)
  avec `overload` sur l'interface plutôt qu'une pool dédiée.

### Incident 4 — Le NAT ne se déclenchait jamais malgré une configuration correcte

- **Symptôme** : `show ip nat translations` restait vide malgré des tests répétés.
- **Méthode de diagnostic, par élimination successive** :
  1. Vérification que les routeurs d'entreprise avaient une route par défaut vers FW
     (sans quoi le trafic vers Internet était abandonné avant même d'atteindre FW).
  2. Vérification que le pare-feu autorisait bien le trafic vers le réseau Internet simulé
     (ajout de règles ICMP/HTTP/HTTPS vers `192.168.122.0/24`).
  3. Test direct depuis FW lui-même vers le nœud Internet, pour éliminer pare-feu et NAT du
     diagnostic.
  4. Vérification de la table ARP de FW : absence de toute entrée pour le nœud Internet,
     malgré une interface `up/up` des deux côtés.
- **Conclusion** : la cause se situe au niveau de l'infrastructure GNS3 (communication entre
  serveurs), voir section 7.

## 9. Améliorations possibles

- Authentification BGP par MD5 entre chaque routeur et FW.
- Filtrage des préfixes BGP avec `prefix-list` et `route-map` pour n'accepter que les
  préfixes attendus de chaque entreprise.
- Redondance du pare-feu (HSRP/VRRP) pour éviter un point unique de défaillance.
- Remplacement des ACL classiques par un Zone-Based Policy Firewall (stateful) pour éviter
  d'avoir à ouvrir manuellement le retour de chaque flux.
- VPN IPsec site-à-site entre les trois entreprises pour chiffrer le trafic transitant par FW.

## 10. Conclusion

Les objectifs de routage BGP et de sécurité par pare-feu sont entièrement atteints et validés
par des tests reproductibles (sessions BGP établies, trafic filtré conformément à la
politique définie). La configuration NAT est correcte et vérifiée en interne ; seul le test
de connectivité vers un point de sortie Internet externe n'a pas pu être finalisé, pour une
raison d'infrastructure du serveur GNS3 distant plutôt qu'une erreur de conception. Le
parcours de diagnostic suivi tout au long du projet (câblage, filtrage du trafic BGP, NAT)
répond directement au dernier objectif de l'énoncé.
