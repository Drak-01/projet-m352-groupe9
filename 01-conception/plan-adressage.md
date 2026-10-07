## Plan d'adressage IP

Tous les réseaux internes sont en `10.10.X.0/24`, avec OPNsense en `.1` (passerelle de chaque zone).

| Zone | VLAN | Réseau | Passerelle (OPNsense) | Adresses des VM | DHCP |
|---|---|---|---|---|---|
| WAN | – | 10.0.2.0/24 (NAT VirtualBox) | 10.0.2.2 | OPNsense : 10.0.2.15 | fourni par VirtualBox |
| DMZ | 10 | 10.10.10.0/24 | 10.10.10.1 | VM-DMZ-WEB : .10 | aucun (IP fixes) |
| LAN interne | 20 | 10.10.20.0/24 | 10.10.20.1 | VM-AD-DNS : .10, VM-FILES : .11, VM-DB : .12 | .100 à .200 (postes), servi par VM-AD-DNS |
| Management | 99 | 10.10.99.0/24 | 10.10.99.1 | VM-ADMIN : .10 | aucun (IP fixes) |
| Supervision | 30 | 10.10.30.0/24 | 10.10.30.1 | VM-WAZUH : .10, VM-BACKUP : .11 | aucun (IP fixes) |
| Tunnel VPN WireGuard | – | 10.10.100.0/24 | 10.10.100.1 | clients admin : .2, .3… | statique par client |

### Choix retenus

- **Plan `10.10.x` :** évite le chevauchement avec `10.0.2.0/24`, réseau NAT par défaut de VirtualBox.
- **Plages :** `.1` à `.99` réservé aux équipements à IP fixe ; DHCP limité à `.100`–`.200`.
- **WAN en Bridged :** l'adresse dépend du réseau physique ; le plan NAT est conservé pour la démonstration.
- **Règles de pare-feu :** écrites avec des IP précises (ex. R4 : `10.10.10.10` → `10.10.30.10`, ports 1514/1515).

### Test de cloisonnement (R3)

Depuis VM-DMZ-WEB (`10.10.10.10`), lancer un `ping` ou `nc` vers `10.10.20.10` et `10.10.20.12` : les deux doivent échouer.