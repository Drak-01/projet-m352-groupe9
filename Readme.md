# Cadrage du projet : Architecture virtualisée sécurisée pour une PME

Groupe 9 · ENSIASD · M352

Chaque membre reproduit l'infrastructure de son côté. Chaque étape produit des livrables partagés dans le dépôt, qui servent d'entrée à l'étape suivante.

---

## Étape 1 : Conception de l'architecture

| Livrable | Format | Contenu |
|---|---|---|
| Schéma d'architecture physique / couches | PNG + source (draw.io) | Hôte, VirtualBox, VMs, couches (matériel, hyperviseur, réseau, services) |
| Schéma des flux | PNG + source (draw.io) | Flèches autorisées / interdites entre zones |
| Matrice des flux | Tableau (md / xlsx) | Source, destination, protocole, port, autorisé / refusé, justification |
| Plan d'adressage IP | Tableau | VLAN, sous-réseau, passerelle, IP de chaque VM, plage DHCP |
| Convention de nommage | Document | Noms des VMs, interfaces, réseaux VirtualBox |
| Spécifications des VMs | Tableau | OS, version, CPU, RAM, disque, rôle, interfaces réseau |
| Inventaire services / ports | Tableau | Reprise de la section 6 du canevas, sert de référence d'audit |
| Comptes et rôles (convention) | Tableau | Noms des comptes admin, groupes, politique de mots de passe (sans les secrets) |

---

## Étape 2 : Réalisation de l'environnement

**Entrée :** plan d'adressage, spécifications des VMs, matrice des flux.

- Création des VMs
- Mise en place du réseau avec segmentation (VLAN)
- Pare-feu OPNsense

| Livrable | Format | Contenu |
|---|---|---|
| Règles de pare-feu | `config.xml` (sans mots de passe ni clés) + tableau lisible par interface | Interfaces, VLAN, DHCP, règles, NAT, alias |
| Procédure de création des VMs | Document | Pas à pas reproductible |
| Configuration réseau segmenté | Document | Réseaux internes VirtualBox, tags VLAN |

---

## Étape 3 : Déploiement du reverse proxy Nginx

**Entrée :** règles de pare-feu de la DMZ, IP de la VM.

| Livrable | Format | Contenu |
|---|---|---|
| Fichiers de configuration Nginx | `nginx.conf`, `sites-available/*.conf` | Configuration du proxy |
| Configuration TLS | Fichiers `.conf` + document | Protocoles, suites de chiffrement, HSTS, redirection 80 vers 443 |
| Procédure de certificat | Document | Let's Encrypt, ou certificat auto-signé en lab |

---

## Étape 4 : Monitoring, sauvegardes et supervision

**Entrée :** IP des VMs, flux syslog autorisés.

### Wazuh

| Livrable | Format | Contenu |
|---|---|---|
| `ossec.conf` | XML | Configuration du manager et des agents |
| Règles et décodeurs personnalisés | `local_rules.xml`, `local_decoder.xml` | Détections spécifiques au projet |
| Configuration du Syslog | Document / extrait de config | Envoi des logs d'OPNsense vers Wazuh |
| Procédure de déploiement des agents | Document | Commande, clé, groupes |
| Dashboards et alertes | Export | Seuils d'alerte retenus |

---

## Règles communes

- Versions figées pour tous (ISO, OPNsense, Nginx, Wazuh, VirtualBox).
- Ne jamais commiter de secrets (mots de passe, clés privées, clés de chiffrement).
- Chaque livrable suit la structure : objectif, prérequis, procédure, vérification.