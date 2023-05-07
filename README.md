Installation d'une VM Ubuntu Server et configuration de SSH, Docker, Nginx avec WAF, Backup et FileBrowser
Prérequis
Accès à une machine virtuelle (VM) vierge sur laquelle installer Ubuntu Server
Connexion internet
carte réseau local
Accès administrateur sur la machine virtuelle
Connaissance de base en ligne de commande Linux
---
Étapes
- [1. Installation d'Ubuntu Server](#i-Installation d'Ubuntu Server)
- [2. Configuration de SSH](#i-Configuration de SSH)
- [3. Installation fail2ban](#i-Installation fail2ban)
- [4. Installation de Docker](#i-Installation de Docker)
- [5 Installation de Nginx avec WAF](#i-Installation de Docker)
- [6. Configuration de la backup](#i-Installation de Docker)
- [4. Installation de Docker](#i-Installation de Docker)
## 1. Installation d'Ubuntu Server
Télécharger l'image ISO d'Ubuntu Server et créer une nouvelle VM en utilisant cet ISO. Lors de la configuration de la VM, définir la quantité de RAM et d'espace disque appropriée.
## 2. Configuration de SSH
SSH est un protocole sécurisé pour accéder à distance à une machine. Nous allons l'installer et le configurer pour se connecter avec une clé SSH.
### 2.1. Installation de SSH
Ouvrir un terminal et exécuter la commande suivante pour installer SSH :
```
sudo apt update
sudo apt install ssh
```
### 2.2. Création d'une clé SSH
Générer une nouvelle clé SSH en utilisant la commande suivante :
```
ssh-keygen -t rsa -b 4096
```
Suivez les instructions pour créer une clé SSH avec une passphrase sécurisée.

### 2.3. Changement de port SSH
Modifier le port SSH par défaut pour améliorer la sécurité. Éditer le fichier de configuration SSH en utilisant la commande suivante :
```
sudo nano /etc/ssh/sshd_config
```
Trouver la ligne # Port 22 et la modifier en Port 2200 ou tout autre port de votre choix. Enregistrez le fichier et redémarrez le service SSH avec la commande suivante :
```
sudo systemctl restart sshd
```
### 2.4. Copie de la clé publique
Copier la clé publique vers le serveur distant en utilisant la commande suivante :
```
ssh-copy-id -p 2200 user@server_ip
```
Remplacer user par votre nom d'utilisateur et server_ip par l'adresse IP de votre serveur. Vous serez invité à saisir le mot de passe de l'utilisateur distant.

## 3. Installation fail2ban
Ouvrez un terminal sur votre serveur Ubuntu et mettez à jour la liste des packages de votre système en exécutant la commande suivante :
```
sudo apt-get update
```
Installez Fail2Ban en exécutant la commande suivante :
```
sudo apt-get install fail2ban
```
Créez un fichier de filtre personnalisé Fail2Ban pour le port 2200 en exécutant la commande suivante :
```
sudo nano /etc/fail2ban/filter.d/sshd-2200.conf
```
Ajoutez les lignes suivantes au fichier pour définir le filtre :
```
[Definition]
failregex = ^.*Failed .* from <HOST>:.*\s$
ignoreregex =
```
Enregistrez et fermez le fichier en appuyant sur "Ctrl+X", puis "Y", puis "Entrée".

Créez une nouvelle jail Fail2Ban pour le port 2200 en exécutant la commande suivante :
```
sudo nano /etc/fail2ban/jail.d/sshd-2200.conf
```
Ajoutez les lignes suivantes au fichier pour définir la jail :
```
[sshd-2200]
enabled = true
port = 2200
filter = sshd-2200
logpath = /var/log/auth.log
maxretry = 3
```
Enregistrez et fermez le fichier en appuyant sur "Ctrl+X", puis "Y", puis "Entrée".

Redémarrez le service Fail2Ban en exécutant la commande suivante :
```
sudo service fail2ban restart
```
Fail2Ban est maintenant installé et configuré pour surveiller le port 2200 sur votre serveur Ubuntu. Toute tentative de connexion échouée à SSH sur le port 2200 sera enregistrée et après le nombre maximum de tentatives, l'adresse IP en question sera bannie.

## 4. Installation de Docker
Docker permet de déployer des applications dans des conteneurs. Nous allons l'installer pour déployer Nginx avec un WAF.
### 4.1. Installation de Docker
Suivre les instructions d'installation de Docker sur Ubuntu Server en utilisant les commandes suivantes :
```
sudo apt update
sudo apt install docker.io
```
Ajouter votre utilisateur au groupe Docker pour éviter de devoir utiliser sudo à chaque commande Docker :
```
sudo usermod -aG docker $USER
```
## 5 Installation de Nginx avec WAF
Nginx est un serveur web populaire. Nous allons le configurer avec un WAF (Web Application Firewall) pour améliorer la sécurité.
Tout d'abord, créer un fichier de configuration nginx.conf en utilisant la commande suivante :
```
sudo nano nginx.conf
```
Copier le contenu suivant dans le fichier :
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    log_format 
    access_log /var/log/nginx/access.log;
sendfile on;
tcp_nopush on;
tcp_nodelay on;
keepalive_timeout 65;
types_hash_max_size 2048;

# WAF configuration
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;

# HTTPS server
server {
    listen 443 ssl http2;
    server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# HTTP server
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
}
```
Enregistrez le fichier et créez un dossier modsec dans /etc/nginx :
```
sudo mkdir /etc/nginx/modsec
```
Téléchargez les règles du WAF en utilisant la commande suivante :
```
sudo curl -o /etc/nginx/modsec/main.conf https://raw.githubusercontent.com/SpiderLabs/owasp-modsecurity-crs/v3.4.0/crs-setup.conf
```
Créez un dossier pour les fichiers de configuration de Nginx :
```
sudo mkdir /etc/nginx/sites-available
sudo mkdir /etc/nginx/sites-enabled
```
Créez un fichier de configuration pour votre site web en utilisant la commande suivante :
```
sudo nano /etc/nginx/sites-available/example.com
```
Copiez le contenu suivant dans le fichier :
```
server {
    listen 8080;
    server_name localhost;
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
Remplacez example.com par votre nom de domaine réel et modifiez les autres paramètres en fonction de vos besoins.

Activez le site en créant un lien symbolique vers sites-enabled :
```
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```
Vérifiez que la configuration Nginx est valide avec la commande suivante :
```
sudo nginx -t
```
Redémarrez le service Nginx avec la commande suivante :
```
sudo systemctl restart nginx
```
## 6. Configuration de la backup
Il est important de faire des sauvegardes régulières de vos données pour éviter les pertes de données. Nous allons configurer une sauvegarde automatique en utilisant Rsync.

### 6.1. Installation de Rsync
Rsync est un utilitaire de synchronisation de fichiers. Nous allons l'installer en utilisant la commande suivante :
```
sudo apt update
sudo apt install rsync
```
### 6.2. Configuration de la backup
Créez un script de sauvegarde en utilisant la commande suivante :
```
sudo nano /opt/backup.sh
```
Copiez le contenu suivant dans le fichier :
```
#!/bin/bash

# Set backup directory
BACKUP_DIR="/var/backups"

# Set backup filename
BACKUP_FILE="backup_$(date +%Y%m%d_%H%M%S).tar.gz"

# Set source directory
SOURCE_DIR="/var/www/html"

# Set destination directory
DEST_DIR="user@remote-server:/path/to/backups"

# Create backup directory if it does not exist
mkdir -p $BACKUP_DIR

# Create backup archive
tar czf $BACKUP_DIR/$BACKUP_FILE $SOURCE_DIR

# Transfer backup archive to remote server
rsync -avz -e "ssh -p 2222" $BACKUP_DIR/$BACKUP_FILE $DEST_DIR
```
Ce script crée une archive de sauvegarde de votre répertoire web dans /var/backups avec un nom contenant la date et l'heure actuelles. Il transfère ensuite l'archive sur un serveur distant via SSH.

Remplacez user@remote-server:/path/to/backups par les informations de votre serveur de sauvegarde distant.

Enregistrez le fichier et rendez-le exécutable en utilisant la commande suivante :
```
sudo chmod +x /opt/backup.sh
```
Testez le script en l'exécutant manuellement :
```
sudo /opt/backup.sh
```
Vous devriez voir un nouveau fichier de sauvegarde dans /var/backups et il devrait être transféré sur votre serveur de sauvegarde distant.

## 7. Configuration de Filebrowser
Filebrowser est un gestionnaire de fichiers basé sur le web. Il vous permet de gérer facilement vos fichiers à partir de n'importe quel navigateur web. Nous allons le configurer pour notre serveur web.

### 7.1. Installation de Filebrowser
Téléchargez le fichier binaire Filebrowser en utilisant la commande suivante :
```
sudo curl -fsSL https://filebrowser.org/get.sh | bash
```
Déplacez le fichier binaire dans /usr/local/bin :
```
sudo mv filebrowser /usr/local/bin/
```
Créez un dossier pour les fichiers de configuration de Filebrowser :
```
sudo mkdir /etc/filebrowser
```
Créez un utilisateur pour Filebrowser :
```
sudo useradd -r -s /bin/false filebrowser
```
Donnez à l'utilisateur filebrowser l'accès au répertoire /var/www/html :
```
sudo chown -R filebrowser: /var/www/html
```
6.2. Configuration de Filebrowser
Créez un fichier de configuration pour Filebrowser en utilisant la commande suivante :
```
sudo nano /etc/filebrowser/config.json
```
Copiez le contenu suivant dans le fichier :
```
{
    "port": 8080,
    "baseURL": "",
    "address": "127.0.0.1",
    "log": "stdout",
    "database": "/etc/filebrowser/database.db",
    "root": "/var/www/html",
    "auth": {
        "method": "basicauth",
        "basicauth": {
            "users": [
                {
                    "username": "admin",
                    "password": "password"
                }
            ]
        }
    }
}
```
Ce fichier configure Filebrowser