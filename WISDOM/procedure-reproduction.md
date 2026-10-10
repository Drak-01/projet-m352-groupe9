# Partie 0 — Socle OPNsense, WAN, VPN et intégration

Responsable : MOUKPE Wisdom Essomadaw

## 1. Création de la VM-FW

| Paramètre | Valeur |
|---|---|
| Nom | VM-FW |
| Type | BSD → FreeBSD (64-bit) |
| RAM | 2048 Mo |
| CPU | 2 vCPU |
| Chipset | ICH9 |
| Disque | 16 Go (VDI, dynamique) |
| ISO | OPNsense-26.7-dvd-amd64.iso |

## 2. Configuration réseau VirtualBox

| Adaptateur | Mode | Réseau | Rôle |
|---|---|---|---|
| NIC 1 | NAT | — | em0 — WAN |
| NIC 2 | Réseau interne | intnet-dmz | em1 — DMZ |
| NIC 3 | Réseau interne | intnet-lan | em2 — LAN |
| NIC 4 | Réseau interne | intnet-mgmt | em3 — MGMT |
| NIC 5 | Réseau interne | intnet-sup | em4 — SUP |

La 5ᵉ interface n'est pas accessible depuis l'interface graphique VirtualBox. Elle a été ajoutée via :

```bash
VBoxManage modifyvm "VM-FW" --nic5 intnet --intnet5 intnet-sup
```

### Redirections de ports NAT (adaptateur 1)

```bash
VBoxManage modifyvm "VM-FW" --natpf1 "http,tcp,,80,,80"
VBoxManage modifyvm "VM-FW" --natpf1 "https,tcp,,443,,443"
VBoxManage modifyvm "VM-FW" --natpf1 "wireguard,udp,,51820,,51820"
```

## 3. Installation OPNsense

- Boot sur l'ISO → login `installer` / `opnsense`
- Partitionnement : `Install (UFS)`
- Mot de passe root défini à l'installation
- Retrait de l'ISO du lecteur optique + désactivation du boot sur `Optical` (Paramètres → Système → Carte mère → ordre de démarrage) avant le premier redémarrage sur le disque

## 4. Assignation des interfaces (menu console, option 1)

| Rôle OPNsense | Interface physique | Zone |
|---|---|---|
| WAN | em0 | Internet (NAT) |
| LAN | em1 | DMZ (VLAN 10) |
| OPT1 | em2 | LAN interne (VLAN 20) |
| OPT2 | em3 | Management (VLAN 99) |
| OPT3 | em4 | Supervision (VLAN 30) |

## 5. Adressage IP (menu console, option 2)

| Interface | IP | Masque |
|---|---|---|
| LAN (DMZ) | 10.10.10.1 | /24 |
| OPT1 (LAN) | 10.10.20.1 | /24 |
| OPT2 (MGMT) | 10.10.99.1 | /24 |
| OPT3 (SUP) | 10.10.30.1 | /24 |
| WAN | DHCP (10.0.2.15) | /24 |

Sur le WAN : décocher **Block private networks** et **Block bogon networks** (nécessaire car le NAT VirtualBox utilise une plage privée).

## 6. Accès temporaire à l'interface web (OPT4)

Comme les interfaces du projet sont toutes sur des réseaux internes isolés, un adaptateur **temporaire** a été ajouté pour piloter la configuration depuis l'interface web, avant la création des vraies VMs (VM-ADMIN notamment) :

```bash
VBoxManage hostonlyif create
VBoxManage modifyvm "VM-FW" --nic6 hostonly --hostonlyadapter6 vboxnet0
```

- OPT4 (em5) : `192.168.56.2/24`
- Règle de pare-feu associée : autoriser `192.168.56.0/24` → `This Firewall` port `443` (TCP/UDP)

> **⚠️ À l'attention de tous les membres du groupe :**
> Le `.ova` partagé pendant la phase de construction contient **6 interfaces réseau** au lieu des 5 prévues dans l'architecture. La 6ᵉ (OPT4/em5, host-only `vboxnet0`) est un accès temporaire utilisé par le responsable de la Partie 0 pour administrer OPNsense via l'interface web, tant que VM-ADMIN (Partie 4) n'existe pas encore.
>
> - **Ne pas s'appuyer dessus** dans vos propres configurations ou règles de pare-feu : elle ne fait pas partie de l'architecture finale.
> - Elle reste active jusqu'à l'**intégration finale** (retour de la Partie 0 en fin de projet), car elle sert aussi aux tests globaux.
> - Elle sera **supprimée avant l'export final soumis pour la notation**, avec la règle de pare-feu associée. L'architecture finale aura bien 5 interfaces (em0 à em4) conformes au schéma du projet.
>
> **Tant que VM-ADMIN (Partie 4) n'existe pas encore**, n'importe quel membre qui a besoin d'accéder à l'interface web OPNsense pour sa propre configuration peut utiliser **directement cet accès existant (OPT4 / `https://192.168.56.2`)**, sans créer de nouvelle configuration ni de nouvel adaptateur. Il suffit d'ajouter un adaptateur **Host-only** (`vboxnet0`) sur sa propre machine hôte si ce n'est pas déjà fait, et de se connecter sur `192.168.56.2`.
>
> **Identifiants de connexion** (console VM et interface web) : utilisateur `root`, mot de passe communiqué à part en privé au groupe (jamais dans le dépôt Git, conformément à la règle 8 des consignes communes).

## 7. Aliases créés (Firewall → Aliases)

| Nom | Type | Contenu | Description |
|---|---|---|---|
| WAZUH | Host(s) | 10.10.30.10 | Serveur Wazuh |
| PBS | Host(s) | 10.10.30.11 | Proxmox Backup Server |
| SERVEURS_LAN | Host(s) | 10.10.20.10, .11, .12 | AD-DNS, Files, DB |
| DEPOTS_MAJ | URL Table | dépôts Ubuntu | Mises à jour système |

## 8. Règles de pare-feu créées

| Règle | Interface | Détail |
|---|---|---|
| R1 | WAN (NAT → Destination NAT) | TCP 443 → 10.10.10.10:443 |
| R2 | WAN (Rules) | UDP 51820 → This Firewall (WireGuard) |
| R10 (DMZ) | LAN/em1 | Bloquer tout, en dernier (les règles d'autorisation R3/R4/R6 seront ajoutées par le responsable Partie 1) |

Les règles par défaut **"Default allow LAN to any rule"** (IPv4/IPv6), créées automatiquement par OPNsense, ont été désactivées car elles court-circuitaient R10.

## 9. WireGuard

- Activé dans **VPN → WireGuard**
- Instance `wg0` : port `51820`, tunnel `10.10.100.1/24`
- Peer modèle `client-admin-template` généré via **Peer generator** (clés générées automatiquement, jamais stockées dans le dépôt)
- Fichier modèle sans secret : `02-reseau-vpn/client-template.conf.example`

## 10. Export de la configuration

- **System → Configuration → Backups** → Download configuration (case "Do not backup RRD data" cochée)
- Export nettoyé des secrets (mot de passe root, clé privée WireGuard serveur, clé publique/PSK du client) → `02-reseau-vpn/opnsense-config-export.xml`
- Le fichier original (non nettoyé) reste local, exclu du dépôt via `.gitignore`

## 11. Tests

### Test nmap depuis l'extérieur (simulation via redirections NAT VirtualBox)

```bash
nmap -Pn -sT -p 80,443,51820 127.0.0.1
```

Résultat au stade actuel :

| Port | État | Explication |
|---|---|---|
| 80/tcp | closed | Aucune règle NAT dédiée au port 80 (redirection prévue vers Nginx, pas encore déployé) |
| 443/tcp | closed | R1 redirige vers `10.10.10.10:443`, mais VM-DMZ-WEB n'existe pas encore |
| 51820/udp | open\|filtered | `sudo nmap -Pn -sU -p 51820 127.0.0.1` — résultat attendu pour WireGuard (pas de réponse à un paquet non sollicité). Confirme que R2 et le service WireGuard fonctionnent. |

**Conclusion :** test **partiellement validé**. R2/WireGuard est confirmé fonctionnel. Le comportement sur 80/443 est cohérent avec l'état actuel (R1 bien configurée), mais la validation complète (*« seuls 80 et 443 visibles »*) nécessite que VM-DMZ-WEB (Partie 1) soit déployée et Nginx actif. **Ce test sur 80/443 est à relancer après l'intégration de la Partie 1.**

### Test d'accès à l'interface d'administration

- Accès `https://192.168.56.2` (interface temporaire OPT4) : **autorisé**, conforme (c'est l'accès admin temporaire utilisé pour la configuration)
- Accès depuis WAN (hors VPN) : à tester une fois le tunnel WireGuard validé avec un vrai client — **à faire en Partie 4** avec VM-ADMIN et R7/R8/R9.

### Test de restauration d'une VM

Non réalisé à ce stade — nécessite une sauvegarde Proxmox Backup Server (VM-BACKUP, Partie 5), pas encore déployée. **Dépendance croisée, à exécuter après l'intégration finale.**

## Écarts par rapport au plan initial

- Ajout temporaire d'un 6ᵉ adaptateur (OPT4, host-only) non prévu dans l'architecture, uniquement pour piloter la configuration avant la création de VM-ADMIN. Sera retiré avant l'export final.
- R7/R8/R9 (accès MGMT à l'interface web) ne sont pas créées dans cette partie : elles relèvent de la Partie 4 (bastion administrateur), conformément à la règle « chaque membre écrit les règles de son interface ».
