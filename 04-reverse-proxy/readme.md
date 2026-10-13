# Règles fire-wall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

Autoriser HTTPS vers DMZ
Vérification : Enregistrez et cliquez sur le bouton rouge Apply changes en haut de la page. La règle doit apparaître active avec une icône verte dans le tableau des règles WAN.



