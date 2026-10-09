# Répartition des tâches par VLAN

Groupe 9 · ENSIASD · M352 · Sujet 1 : Architecture virtualisée sécurisée pour une PME

Chaque membre prend en charge un VLAN de bout en bout : VMs, services exposés, règles de pare-feu de son interface, tests. Tout le monde utilise OPNsense en respectant le schéma d'architecture, le plan d'adressage et la matrice de flux (règles R1 à R10).

## Vue d'ensemble

| Partie | Périmètre | Interface OPNsense | Réseau | Responsable |
|---|---|---|---|---|
| 0 | Socle OPNsense, WAN, VPN, intégration, tests globaux | em0 (WAN) | 10.0.2.0/24 (NAT) | MOUKPE Wisdom Essomadaw |
| 1 | DMZ : Nginx | em1 | 10.10.10.0/24 (VLAN 10) | |
| 2 | LAN : réseau, règles, PostgreSQL, serveur de fichiers | em2 | 10.10.20.0/24 (VLAN 20) | |
| 3 | LAN : AD, DNS, DHCP | em2 | 10.10.20.0/24 (VLAN 20) | |
| 4 | Management : bastion administrateur | em3 | 10.10.99.0/24 (VLAN 99) | |
| 5 | Supervision : Wazuh et sauvegarde | em4 | 10.10.30.0/24 (VLAN 30) | |

6 membres pour 6 parties. Le VLAN 20 est découpé en deux parties (2 et 3) car il regroupe le plus de services.

## Règles communes

1. Respecter le plan d'adressage : passerelle en `.1` sur chaque zone, VM en IP fixe.
2. Respecter la convention de nommage des VMs : `VM-DMZ-WEB`, `VM-AD-DNS`, `VM-FILES`, `VM-DB`, `VM-ADMIN`, `VM-WAZUH`, `VM-BACKUP`.
3. Chacun crée ses VMs de son côté, sans `.ova`.
4. Sous VirtualBox, un réseau interne par zone (`intnet-dmz`, `intnet-lan`, `intnet-mgmt`, `intnet-sup`). Les numéros de VLAN servent de repère logique, car VirtualBox ne gère pas le tagging 802.1Q.
5. Chaque membre écrit uniquement les règles de pare-feu de son interface (règles évaluées de haut en bas, première correspondance appliquée), avec la règle de blocage R10 en dernier.
6. Chaque partie livre : liste des services et ports exposés (chaque port justifié), tableau des règles de son interface, configuration des services, procédure de reproduction, résultats de tests.
7. Chaque livrable est testé par un autre membre avant d'être accepté.
8. Les secrets (mots de passe, clés VPN, clés de sauvegarde) restent hors du dépôt Git.
9. Pour tester seul, il est possible de démarrer sur un réseau à plat, puis de basculer sur les zones avec la configuration OPNsense de la partie 0.

---

## Partie 0 : Socle OPNsense, WAN, VPN et intégration

| Élément | Détail |
|---|---|
| VM | VM-FW, OPNsense, 2 vCPU, 2 Go RAM, 5 interfaces (chipset ICH9, `VBoxManage modifyvm --nic5`) |
| Interfaces | em0 WAN, em1 DMZ, em2 LAN, em3 MGMT, em4 SUP |
| Règles à créer | R1 (DNAT 443 vers 10.10.10.10), R2 (UDP 51820 WireGuard) |
| Services exposés | WAN : 443/tcp, 80/tcp (redirection), 51820/udp. Interface web OPNsense : 443/tcp, depuis MGMT et VPN uniquement |
| Redirections VirtualBox | hôte 443, 80, 51820/udp vers l'IP WAN d'OPNsense. Décocher « Block private networks » sur le WAN |

Livrables :
- Configuration OPNsense de base (interfaces assignées, alias, ordre des règles), exportée sans mots de passe ni clés.
- Configuration du serveur WireGuard (réseau tunnel 10.10.100.0/24) et modèle de client sans clé.
- Alias : `WAZUH = 10.10.30.10`, `PBS = 10.10.30.11`, `SERVEURS_LAN = 10.10.20.10-12`, `DEPOTS_MAJ`.
- Procédure de reproduction, plan de tests globaux, rapport d'écarts, assemblage du rapport final.

Tests : scan `nmap` depuis l'extérieur (seuls 80 et 443 visibles), accès à l'interface d'administration refusé hors VPN, restauration d'une VM.

---

## Partie 1 : DMZ (VLAN 10, em1)

| Élément | Détail |
|---|---|
| Réseau | 10.10.10.0/24, passerelle 10.10.10.1 |
| VM | VM-DMZ-WEB, 10.10.10.10, Ubuntu Server, Nginx, agent Wazuh, 1 Go RAM |
| DHCP | Aucun, IP fixe |

Services et ports exposés :

| Service | Port | Justification |
|---|---|---|
| Nginx (HTTPS) | 443/tcp | Seul service publié sur Internet |
| Nginx (redirection) | 80/tcp | Redirection vers 443 uniquement, aucun contenu en clair |
| SSH | 22/tcp | Depuis VM-ADMIN (10.10.99.10) uniquement, jamais depuis Internet |
| Agent Wazuh | sortant 1514/1515 | Envoi des journaux vers 10.10.30.10 |

Règles sur l'interface em1 (dans cet ordre) :

| Ordre | Règle | Source | Destination | Port | Action |
|---|---|---|---|---|---|
| 1 | R3 | 10.10.10.0/24 | 10.10.20.0/24 et 10.10.99.0/24 | Tous | Bloquer + log |
| 2 | R4 | 10.10.10.10 | 10.10.30.10 | TCP 1514, 1515 | Autoriser |
| 3 | R6 | 10.10.10.10 | 10.10.10.1 | TCP/UDP 53 | Autoriser |
| 4 | R6 | 10.10.10.10 | `DEPOTS_MAJ` | TCP 80, 443 | Autoriser |
| 5 | R10 | Toute source | Toute destination | Tous | Bloquer + log |

Livrables :
- `nginx.conf` et `sites-available/*.conf`, configuration TLS (protocoles récents, HSTS, redirection 80 vers 443).
- Procédure de certificat (Let's Encrypt si un nom de domaine public est disponible, sinon certificat auto-signé ou autorité interne).
- Checklist de durcissement (pare-feu local limité à 443, 80, 22 depuis le MGMT, mises à jour, SSH par clé, désactivation de ce qui est inutile).
- Page de démonstration.

Tests : depuis 10.10.10.10, `nc -zv 10.10.20.10 445` doit échouer (R3) ; `nmap` depuis l'extérieur ne doit montrer que 80 et 443 (R1).

---

## Partie 2 : LAN, réseau, règles, PostgreSQL et serveur de fichiers (VLAN 20, em2)

| Élément | Détail |
|---|---|
| Réseau | 10.10.20.0/24, passerelle 10.10.20.1 |
| VM | VM-FILES 10.10.20.11 (1-2 Go RAM), VM-DB 10.10.20.12 (2 Go RAM) |
| DHCP | Désactivé sur OPNsense pour ce réseau, le DHCP est servi par VM-AD-DNS (partie 3) |

Services et ports exposés :

| VM | Service | Port | Justification |
|---|---|---|---|
| VM-FILES | SMB / NFS | 445/tcp, 2049/tcp | Postes internes authentifiés (AD) uniquement |
| VM-DB | PostgreSQL / MySQL | 5432/tcp (ou 3306/tcp) | Accès restreint à la VM applicative, jamais aux utilisateurs ni à Internet |
| Les deux | Agent Wazuh | sortant 1514/1515 | Envoi des journaux |
| Les deux | SSH / RDP | 22/tcp, 3389/tcp | Depuis VM-ADMIN uniquement |

Règles sur l'interface em2 (propriétaire de la configuration de cette interface) :

| Ordre | Règle | Source | Destination | Port | Action |
|---|---|---|---|---|---|
| 1 | R5 | 10.10.20.0/24 | 10.10.30.10 | TCP 1514, 1515 | Autoriser |
| 2 | R5 | 10.10.20.0/24 | 10.10.30.11 | TCP 8007 | Autoriser |
| 3 | R6 | 10.10.20.0/24 | Internet | TCP 80, 443 | Autoriser |
| 4 | R10 | Toute source | Toute destination | Tous | Bloquer + log |

Le DNS interne et le DHCP restent dans le LAN et ne traversent pas le pare-feu.

Livrables :
- Configuration de l'interface em2 et du tableau de règles.
- Configuration PostgreSQL (script SQL de création, `pg_hba.conf` limité à l'IP de la VM applicative, ou `my.cnf` pour MySQL).
- Configuration du partage de fichiers (droits par groupe AD) et pare-feu local des deux VMs.

Tests : depuis la DMZ, aucune connexion vers 10.10.20.10 à .12 ; depuis un poste du LAN, accès au partage OK ; accès à la base refusé depuis tout hôte non autorisé.

---

## Partie 3 : LAN, AD, DNS et DHCP (VLAN 20, em2)

| Élément | Détail |
|---|---|
| VM | VM-AD-DNS, 10.10.20.10, Samba AD ou Windows Server, 2-4 Go RAM |
| Dépendance | Les groupes AD servent à la partie 2 (droits du partage de fichiers) |

Services et ports exposés :

| Service | Port | Justification |
|---|---|---|
| DNS | 53/tcp+udp | Résolution des postes internes |
| Kerberos | 88/tcp+udp | Authentification |
| LDAP | 389/tcp | Annuaire |
| DHCP | 67/68/udp | Adressage des postes (plage .100 à .200) |
| Ports AD complémentaires | à inventorier (par exemple 445, 464, 636, 3268) | Selon la solution retenue, chaque port justifié |
| Agent Wazuh | sortant 1514/1515 | Envoi des journaux |

Livrables :
- Domaine, unités d'organisation, groupes, comptes de test (fichier `.csv`, sans mots de passe réels).
- Configuration DNS (redirecteur vers 10.10.20.1) et DHCP (plage, options passerelle et DNS).
- Pare-feu local de la VM et inventaire des ports ouverts, vérifié par `nmap`.

Tests : un poste du LAN obtient son IP par DHCP, s'authentifie sur l'AD et résout les noms internes ; aucun port non listé n'est ouvert.

---

## Partie 4 : Management, bastion administrateur (VLAN 99, em3)

| Élément | Détail |
|---|---|
| Réseau | 10.10.99.0/24, passerelle 10.10.99.1, tunnel VPN 10.10.100.0/24 |
| VM | VM-ADMIN, 10.10.99.10, poste d'administration, 2 Go RAM |

Services et ports exposés :

| Service | Port | Justification |
|---|---|---|
| SSH | 22/tcp | Entrée depuis le tunnel VPN (10.10.100.0/24) uniquement |
| Navigateur vers OPNsense | sortant 443 vers 10.10.99.1 | Administration du pare-feu |
| Clients d'administration | sortant 22, 3389, 445, 443, 8007 | Vers les zones, selon R8 et R9 |

Règles sur l'interface em3 (et sur l'interface WireGuard) :

| Règle | Source | Destination | Port | Action |
|---|---|---|---|---|
| R7 | 10.10.99.0/24 et 10.10.100.0/24 | 10.10.99.1 | TCP 443 | Autoriser (anti-lockout) |
| R8 | 10.10.99.10 | 10.10.10.10 | TCP 22 | Autoriser |
| R8 | 10.10.99.10 | 10.10.20.10 à .12 | TCP 22, 3389, 445 | Autoriser |
| R9 | 10.10.99.10 | 10.10.30.10 | TCP 443 | Autoriser |
| R9 | 10.10.99.10 | 10.10.30.11 | TCP 8007 | Autoriser |
| R10 | Toute source | Toute destination | Tous | Bloquer + log |

Livrables :
- Configuration du bastion (outils, SSH par clé, durcissement, pare-feu local).
- Tableau des règles de em3 et du tunnel, procédure de connexion VPN puis rebond.
- Politique de comptes et de mots de passe d'administration (sans secrets dans le dépôt).

Tests : accès à `https://10.10.99.1` refusé depuis un réseau non VPN ; accès autorisé via le VPN ; aucune autre zone ne peut initier de connexion vers le MGMT.

---

## Partie 5 : Supervision, Wazuh et sauvegarde (VLAN 30, em4)

| Élément | Détail |
|---|---|
| Réseau | 10.10.30.0/24, passerelle 10.10.30.1 |
| VM | VM-WAZUH 10.10.30.10 (6-8 Go RAM), VM-BACKUP 10.10.30.11 (2 Go RAM) |

Services et ports exposés :

| VM | Service | Port | Justification |
|---|---|---|---|
| VM-WAZUH | Réception des agents | 1514/tcp, 1515/tcp | Depuis la DMZ (R4) et le LAN (R5), IP sources limitées |
| VM-WAZUH | Réception syslog | 514/udp | Depuis 10.10.30.1 (OPNsense) uniquement |
| VM-WAZUH | Dashboard | 443/tcp | Depuis VM-ADMIN uniquement (R9) |
| VM-BACKUP | Proxmox Backup Server | 8007/tcp | Clients (R5) et interface web depuis VM-ADMIN (R9) |

Règles sur l'interface em4 : R10 uniquement (blocage et journalisation). Pour l'installation initiale (accès Internet), ouvrir une règle temporaire puis la supprimer une fois les paquets installés.

Livrables :
- `ossec.conf` du manager et des agents, règles et décodeurs personnalisés (alerte sur toute tentative DMZ vers LAN).
- Collecte des journaux OPNsense en syslog vers Wazuh, déploiement des agents sur les autres VMs.
- Dashboards et alertes exportés.
- Configuration PBS (datastore, chiffrement côté client), planification et procédure de restauration testée.

Tests : alerte déclenchée lors d'un test R3 ; restauration d'une VM à partir d'une sauvegarde, avec redémarrage correct et données attendues.

---

## Ordre de dépendance

```
Partie 0 (socle OPNsense, interfaces, alias)
  -> Parties 1, 2, 3, 4, 5 (en parallèle, chacune sur son interface)
       -> Partie 0 (intégration, tests globaux, rapport final)
```

Les parties 2 et 3 partagent le VLAN 20 : la partie 3 fournit les groupes AD, la partie 2 s'en sert pour les droits du partage de fichiers.