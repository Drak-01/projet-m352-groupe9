# Répartition des lots

Groupe 9 · ENSIASD · M352 · Sujet 1 : Architecture virtualisée sécurisée pour une PME

6 lots pour 6 membres. Chacun choisit un lot et inscrit son nom dans la colonne « Responsable ».
Chacun crée ses VMs de son côté, sans `.ova`.

| Lot | Brique | Livrables à fournir | Débloque | Responsable |
|---|---|---|---|---|
| 1 | Conception de l'architecture | Schéma d'architecture (physique/couches), schéma des flux, matrice des flux, plan d'adressage IP, convention de nommage, specs des VMs (OS, version, CPU, RAM, disque), inventaire services/ports, comptes et rôles, tableau des versions et hash des ISO, dépôt Git + README | Tous les lots | |
| 2 | Réseau VirtualBox et VPN d'administration | Paramètres VirtualBox (type de carte, réseaux internes nommés, promiscuité, tags VLAN), configuration du VPN d'administration (serveur, réseau tunnel, paramètres cryptographiques, modèle client sans clés) | Lots 3, 4, 5, 6 | |
| 3 | Pare-feu OPNsense | `config.xml` nettoyé (sans mots de passe ni clés), tableau des règles par interface, config VLAN/DHCP/NAT, alias, procédure d'import | Lots 4, 5, 6 | |
| 4 | Reverse proxy Nginx (DMZ) | `nginx.conf`, `sites-available/*.conf`, configuration TLS (protocoles, HSTS, redirection 80 vers 443), procédure de certificat, checklist de durcissement, page de démonstration | Lot 6 | |
| 5 | Services internes (LAN) | Config AD/DNS/DHCP (domaine, OU, comptes de test en `.csv`), partage de fichiers, base de données (script SQL, `pg_hba.conf`/`my.cnf`), pare-feu local des VMs | Lot 6 | |
| 6 | Supervision, sauvegardes et tests | `ossec.conf` (manager et agents), règles et décodeurs personnalisés, syslog OPNsense vers Wazuh, déploiement des agents, dashboards et alertes exportés, config PBS et procédure de restauration, plan de tests, scripts, résultats, rapport d'écarts | Rapport final | |

## Ordre de dépendance

```
Lot 1 (conception, versions, plan d'adressage)
  -> Lot 2 (réseau VirtualBox + VPN)
       -> Lot 3 (pare-feu)
            -> Lot 4 (Nginx)
            -> Lot 5 (services internes)
            -> Lot 6 (Wazuh, PBS, tests)
```

## Règles communes

1. Chacun respecte les specs, le plan d'adressage, la convention de nommage et les paramètres réseau du lot 2.
2. Chaque livrable est déposé dans le dossier de son lot, avec le modèle commun (objectif, prérequis, procédure, vérification).
3. Chaque livrable est testé par au moins un autre membre avant d'être accepté.
4. Les écarts sont signalés par une issue Git adressée au responsable du lot.
5. Les secrets (mots de passe, clés VPN, clé PBS) restent hors du dépôt.
6. Les lots 4, 5 et 6 peuvent commencer sur un réseau à plat, puis basculer sur les VLAN à la réception du `config.xml` du lot 3.