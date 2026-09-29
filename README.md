# Documentation - Mise en place du serveur Starfleet

## Cahier des charges du projet

**Infrastructure**
- 2 VM Debian :
  - **VM Serveur** : Debian sans interface graphique, 2 Go RAM, 2 vCPU, disque 32 Go, 2 cartes réseau (WAN + LAN)
  - **VM Cliente** : Debian avec GUI, 2 Go RAM, 2 vCPU, disque 16 Go, connectée sur le LAN de la VM serveur, avec un navigateur web
- La VM serveur héberge : DHCP, DNS, FTP, Web, base de données SQL et LDAP (sur la carte LAN)

**Contraintes**
- Pas de compte sudo (conforme aux directives Starfleet) → toutes les commandes s'exécutent en root
- Pare-feu en place, n'autorisant que les ports strictement nécessaires
- Serveur Web en Nginx, en HTTPS obligatoire
- Nginx, PHP et MariaDB doivent être en dernière version (pas celle des dépôts Debian)
- Cohabitation de PHP 7.x et PHP 8.x

**Domaine et services**
- Domaine : `starfleet.lan`
- `www8.starfleet.lan` → site web en PHP 8
- `www7.starfleet.lan` → site web en PHP 7
- `php.starfleet.lan` → phpMyAdmin
- `admin.starfleet.lan` → administration de la VM
- Serveur FTP en SSL/TLS, chrooté sur le dossier web, pour déposer les fichiers du serveur Web
- Un même certificat SSL sert au serveur Web et au serveur FTP
- Authentification des utilisateurs du serveur Web via un annuaire **LDAP**

## Sommaire
1. [DHCP (isc-dhcp-server)](#1-dhcp-isc-dhcp-server)
2. [DNS (Bind9)](#2-dns-bind9)
3. [Serveur Web - Nginx, PHP, MariaDB](#3-serveur-web---nginx-php-mariadb)
4. [FTP, SSL/TLS et Cockpit](#4-ftp-ssltls-et-cockpit)
5. [Pare-feu (UFW)](#5-pare-feu-ufw)
6. [LDAP](#6-ldap)

---

## 1. DHCP (isc-dhcp-server)

### 1.1 Installation

Installation du paquet serveur DHCP de l'ISC :

```bash
apt update
apt install -y isc-dhcp-server
```

### 1.2 Configuration de l'interface d'écoute

Le service a besoin de savoir sur quelle interface réseau distribuer les baux. Cela se configure dans `/etc/default/isc-dhcp-server` (renseigner l'interface reliée au LAN, ex. `eth1` ou `ens19`) :

```bash
nano /etc/default/isc-dhcp-server
```

### 1.3 Configuration du fichier `dhcpd.conf`

```bash
nano /etc/dhcp/dhcpd.conf
```

Paramètres généraux (domaine, DNS, durée des baux) :

```conf
option domain-name "starfleet.lan";
option domain-name-servers 192.168.42.10;

default-lease-time 32400;
max-lease-time 604800;
log-facility local7;
authoritative;

ddns-update-style none;
```

> `authoritative;` indique que ce serveur DHCP fait référence sur le réseau : il peut répondre en priorité et corriger les clients qui auraient un bail invalide d'un autre serveur.

Déclaration du sous-réseau et de la plage d'adresses distribuées :

```conf
subnet 192.168.42.0 netmask 255.255.255.0 {
  range 192.168.42.100 192.168.42.200;
  option broadcast-address 192.168.42.255;
  option routers 192.168.42.10;
  option domain-name "starfleet.lan";
  option domain-name-servers 192.168.42.10;
}
```

- Plage distribuée : `192.168.42.100` → `192.168.42.200`
- Passerelle (`routers`) : `192.168.42.10` (le serveur Debian, qui fait aussi routeur/DNS)
- Serveur DNS distribué aux clients : `192.168.42.10` (Bind9 installé plus haut)

### 1.4 Activation et démarrage du service

```bash
systemctl enable --now isc-dhcp-server
```

### 1.5 Vérification

Statut du service :

```bash
systemctl status isc-dhcp-server
```

Résultat attendu : service `active (running)`.

Logs du service :

```bash
journalctl -u isc-dhcp-server -n 50
```

Consultation des baux distribués aux clients :

```bash
cat /var/lib/dhcp/dhcpd.leases
```

---

## 2. DNS (Bind9)

### 2.1 Installation

On commence par mettre à jour le cache de paquets puis installer le paquet `bind9` ainsi qu'un autre paquet contenant des outils DNS supplémentaires :

```bash
apt-get update
apt-get install bind9 dnsutils
```

### 2.2 Organisation des fichiers de configuration

- Les fichiers **`db.<nom>`** correspondent aux fichiers de zones intégrés par défaut dans Bind. On peut s'en inspirer comme modèle pour créer ses propres fichiers de zones.
- Le fichier **`named.conf`** est le fichier de configuration principal de Bind9. Il contient des directives `include` pour charger 3 autres fichiers :
  - **`named.conf.options`** : options de configuration de Bind
  - **`named.conf.local`** : déclaration des zones
  - **`named.conf.default-zones`** : définition des zones incluses par défaut avec Bind

### 2.3 Sauvegarde des fichiers de base

Par sécurité, on copie les fichiers de base (un snapshot de la VM a également été fait) :

```bash
cd /etc/bind
cp named.conf.options named.conf.options.bkp
cp named.conf.local named.conf.local.bkp
```

### 2.4 Configuration de `named.conf.options`

```bash
nano /etc/bind/named.conf.options
```

```conf
// Redirecteurs DNS (résolveurs externes)
forwarders {
    1.1.1.1;
    9.9.9.9;
};

// Mode récursif, pour résoudre les noms externes
recursion yes;

// Active la validation DNSSEC (vérifier l'authenticité des réponses DNS signées)
dnssec-validation auto;

// Écouter sur toutes les interfaces réseau en IPv4 et IPv6
listen-on { any; };
listen-on-v6 { any; };
```

### 2.5 Sécurisation par ACL

Pour indiquer que seules les machines du LAN peuvent contacter ce serveur DNS, on définit une ACL. Elle doit être déclarée **avant** le bloc `options` :

```conf
// Autoriser uniquement certains réseaux à solliciter ce DNS
acl "lan" {
    192.168.42.0/24;
    localhost;
    localnets;
};
```

Puis, dans le bloc `options`, à la suite des directives `listen-on` mais avant la fermeture du bloc :

```conf
// Autoriser les requêtes pour les hôtes de l'ACL "lan"
allow-query { lan; };
```

On vérifie la syntaxe :

```bash
named-checkconf
```

S'il y a des erreurs de syntaxe, les numéros de lignes concernées sont retournés dans la console. Sinon, c'est bon.

### 2.6 Création de la nouvelle zone DNS

On édite le fichier de déclaration des zones :

```bash
nano /etc/bind/named.conf.local
```

```conf
zone "starfleet.lan" {
    type master;
    file "/etc/bind/db.starfleet.lan";
    allow-update { none; };
};
```

Cela crée la zone DNS `starfleet.lan`, dont le fichier de zone sera `/etc/bind/db.starfleet.lan`. L'instruction `allow-update { none; };` refuse les mises à jour des enregistrements DNS par un tiers non autorisé.

### 2.7 Configuration du fichier de zone

Debian fournit des modèles (templates) directement dans `/etc/bind/`. On copie le modèle par défaut `db.local` vers le nouveau fichier de zone avant de l'éditer :

```bash
cp /etc/bind/db.local /etc/bind/db.starfleet.lan
```

Le fichier contient l'en-tête standard avec le **SOA** (Start of Authority), l'enregistrement **NS** et un enregistrement **A** de base. Il faut remplacer `localhost.` par le nom de domaine et ajouter les adresses IP.

On teste ensuite la syntaxe avec `named-checkzone` :

```bash
named-checkzone starfleet.lan /etc/bind/db.starfleet.lan
# zone starfleet.lan/IN: loaded serial 2
# OK
```

**Rappel infra :**
- IP LAN du serveur Debian : `192.168.42.10/24`
- Nom de zone DNS : `starfleet.lan`
- Le serveur est joignable via : `serveur-debian.starfleet.lan`

#### Explications des champs de zone

| Champ | Description |
|---|---|
| `$TTL 604800` | Durée de vie des infos en cache chez les autres serveurs DNS (défaut : 24h / 86400s) |
| `@` | Désigne la racine de la zone (ex : `starfleet.lan`) |
| `SOA` | Start Of Authority : paramètres principaux de la zone, serveur DNS primaire et contact technique (le `@` de l'email est remplacé par un `.`) |
| `Serial` | Numéro de série de la zone, à incrémenter à chaque modification |
| `Refresh (604800)` | Délai de rafraîchissement pour la synchro entre serveurs DNS |
| `Retry (86400)` | Délai avant nouvelle tentative de synchro si le refresh a échoué |
| `Expire (2419200)` | Au bout de ce délai (28 jours par défaut), un serveur secondaire arrête de répondre pour cette zone si la synchro échoue toujours |
| `Negative Cache TTL (86400)` | Durée de conservation en cache d'une réponse `NXDOMAIN` |

### 2.8 Démarrage de Bind9

```bash
systemctl start bind9
systemctl enable named.service
systemctl status bind9
```

### 2.9 Tester la résolution de noms

Sur le serveur DNS lui-même, on modifie `resolv.conf` pour qu'il interroge son propre résolveur local :

```bash
nano /etc/resolv.conf
```

```conf
search debian-serveur.local
domain debian-serveur.local
nameserver 127.0.0.1
```

> **Pourquoi `127.0.0.1` et pas `192.168.42.10` ?**
> Ce fichier est sur le serveur qui fait tourner Bind. Bind écoute sur toutes les interfaces (localhost inclus), donc interroger `127.0.0.1` revient à s'interroger soi-même directement, sans passer par la carte réseau.

Tests de résolution avec `nslookup` :

```bash
nslookup serveur-debian.debian-serveur.local
nslookup dns.debian-serveur.local
```

---

## 3. Serveur Web - Nginx, PHP, MariaDB

### 3.1 Installation de Nginx (dépôt officiel)

Import de la clé de signature officielle et des prérequis :

```bash
apt update
apt install -y curl gnupg2 ca-certificates lsb-release debian-archive-keyring
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor | tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

En root, la commande de récupération de clé change légèrement :

```bash
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
```

Vérification que la clé a bien été téléchargée :

```bash
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

Dépôt apt pour les paquets **mainline** :

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/mainline/debian $(lsb_release -cs) nginx" > /etc/apt/sources.list.d/nginx.list
```

Dépôt apt pour les paquets **stables** (alternative) :

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
https://nginx.org/packages/debian $(lsb_release -cs) nginx" \
    | tee /etc/apt/sources.list.d/nginx.list
```

Configuration de la priorité des paquets nginx.org :

```bash
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" > /etc/apt/preferences.d/99nginx
```

Vérification :

```bash
cat /etc/apt/preferences.d/99nginx
```

Installation :

```bash
apt update
apt install -y nginx
```

#### Dépannage clé GPG / dépôt

Causes fréquentes de problème :
- **Clé GPG invalide ou vide** : URL tapée en double (`nginx.org/keys/nginx.org/keys/`), téléchargeant une page 404 au lieu de la vraie clé.
- **Trousseau corrompu** : `gpg --dearmor` a converti la page d'erreur en fichier `.gpg` inutilisable.
- **Blocage APT (`NO_PUBKEY ...`)** : le trousseau étant vide/illisible, APT refuse de faire confiance au dépôt.

Procédure de correction :

```bash
# 1. Télécharger la clé officielle propre
curl -fsSL https://nginx.org/keys/nginx_signing.key -o /tmp/nginx_signing.key

# 2. Convertir la clé au format binaire APT
gpg --dearmor --yes -o /usr/share/keyrings/nginx-archive-keyring.gpg /tmp/nginx_signing.key

# 3. Accorder les droits de lecture à APT
chmod 644 /usr/share/keyrings/nginx-archive-keyring.gpg

# 4. Déclarer proprement le dépôt Nginx mainline
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/mainline/debian bookworm nginx" > /etc/apt/sources.list.d/nginx.list

# 5. Mettre à jour et installer
apt update
apt install -y nginx
```

### 3.2 Installation de PHP (dépôt Sury)

Sur Debian 12, avant d'installer PHP 7.4 et 8.x, il faut ajouter le dépôt PHP Sury :

```bash
curl -fsSL https://packages.sury.org/php/apt.gpg | gpg --dearmor -o /etc/apt/trusted.gpg.d/php.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list
apt update
```

Installation de PHP 7.4, PHP 8.2 et phpMyAdmin :

```bash
apt install -y php7.4-fpm php8.2-fpm phpmyadmin
```

Redémarrage de Nginx puis vérification du statut des deux versions de PHP :

```bash
systemctl restart nginx
systemctl status php7.4-fpm php8.2-fpm --no-pager
```

### 3.3 Installation de MariaDB

Téléchargement et exécution du script officiel de configuration du dépôt :

```bash
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | bash
```

Mise à jour de l'index des paquets et installation :

```bash
apt update
apt install -y mariadb-server
```

Activation et vérification du statut :

```bash
systemctl enable --now mariadb
systemctl status mariadb
```

Création d'un utilisateur administrateur :

```sql
CREATE USER 'admin'@'%' IDENTIFIED BY 'MonMotDePasseSecurise123!';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

---

## 4. FTP, SSL/TLS et Cockpit

### 4.1 Génération du certificat SSL auto-signé

Certificat pour le domaine `*.starfleet.lan` :

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/starfleet.key \
-out /etc/ssl/certs/starfleet.crt \
-subj "/C=FR/ST=PACA/L=Miramas/O=Starfleet/OU=IT/CN=*.starfleet.lan"
```

Vérification :

```bash
ls -l /etc/ssl/private/starfleet.key
ls -l /etc/ssl/certs/starfleet.crt
```

### 4.2 Serveur FTP (VSFTPD) chrooté avec SSL/TLS

Installation :

```bash
apt install -y vsftpd
```

Configuration complète :

```bash
cat << 'EOF' > /etc/vsftpd.conf
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES

# Restriction Chroot (isolation dans le dossier web)
chroot_local_user=YES
allow_writeable_chroot=YES

# Configuration SSL / TLS (FTPS)
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1_2=YES
rsa_cert_file=/etc/ssl/certs/starfleet.crt
rsa_private_key_file=/etc/ssl/private/starfleet.key

# Mode passif
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=50000
EOF
```

Vérification :

```bash
grep ssl_enable /etc/vsftpd.conf
# doit retourner : ssl_enable=YES
```

### 4.3 Création de l'utilisateur FTP

Création de l'utilisateur `webftp`, dossier racine `/var/www`, et permissions :

```bash
useradd -d /var/www -s /bin/bash webftp
passwd webftp
chown -R webftp:www-data /var/www
chmod -R 775 /var/www
```

Vérification :

```bash
grep webftp /etc/passwd
# la ligne doit se terminer par /var/www:/bin/bash
```

### 4.4 Activer et redémarrer VSFTPD

```bash
systemctl restart vsftpd && systemctl enable vsftpd
systemctl status vsftpd
```

En cas d'échec du service, on peut relancer manuellement pour voir l'erreur :

```bash
vsftpd /etc/vsftpd.conf
```

Si besoin, réinjecter la configuration (variante avec restrictions SSL supplémentaires) :

```bash
cat << 'EOF' > /etc/vsftpd.conf
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
allow_writeable_chroot=YES
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
rsa_cert_file=/etc/ssl/certs/starfleet.crt
rsa_private_key_file=/etc/ssl/private/starfleet.key
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=50000
EOF
```

### 4.5 Configuration des vhosts Nginx (HTTP + HTTPS)

**`www8.starfleet.lan` (PHP 8.2) :**

```bash
cat << 'EOF' > /etc/nginx/conf.d/www8.starfleet.lan.conf
server {
    listen 80;
    listen 443 ssl;
    server_name www8.starfleet.lan;
    root /var/www/www8;
    index index.php index.html;

    ssl_certificate /etc/ssl/certs/starfleet.crt;
    ssl_certificate_key /etc/ssl/private/starfleet.key;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF
```

**`www7.starfleet.lan` (PHP 7.4) :**

```bash
cat << 'EOF' > /etc/nginx/conf.d/www7.starfleet.lan.conf
server {
    listen 80;
    listen 443 ssl;
    server_name www7.starfleet.lan;
    root /var/www/www7;
    index index.php index.html;

    ssl_certificate /etc/ssl/certs/starfleet.crt;
    ssl_certificate_key /etc/ssl/private/starfleet.key;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF
```

**`php.starfleet.lan` (phpMyAdmin) :**

```bash
cat << 'EOF' > /etc/nginx/conf.d/php.starfleet.lan.conf
server {
    listen 80;
    listen 443 ssl;
    server_name php.starfleet.lan;
    root /usr/share/phpmyadmin;
    index index.php index.html;

    ssl_certificate /etc/ssl/certs/starfleet.crt;
    ssl_certificate_key /etc/ssl/private/starfleet.key;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF
```

Test de la syntaxe et rechargement :

```bash
nginx -t
systemctl reload nginx
```

Vérification que les fichiers du site existent et appartiennent au bon utilisateur :

```bash
ls -l /var/www/www8/index.php /var/www/www7/index.php
# doivent appartenir à webftp:www-data
```

### 4.6 Résolution côté client

Sur le poste client, ajout des entrées dans le fichier `hosts` :

```bash
echo "192.168.42.10 www8.starfleet.lan www7.starfleet.lan php.starfleet.lan" >> /etc/hosts
```

Vérification :

```bash
tail -n 1 /etc/hosts
```

Test de connectivité :

```bash
ping -c 2 www8.starfleet.lan
```

### 4.7 Ajout du sous-domaine `php.starfleet.lan` au DNS

Retour sur le serveur, édition du fichier de zone :

```bash
nano /etc/bind/db.starfleet.lan
```

Incrémenter le **Serial** et ajouter la ligne :

```conf
php     IN      A       192.168.42.10
```

Vérification et rechargement :

```bash
named-checkzone starfleet.lan /etc/bind/db.starfleet.lan
systemctl reload bind9
```

### 4.8 Cockpit (administration web)

Installation et activation :

```bash
apt update && apt install -y cockpit
systemctl enable --now cockpit.socket
```

**Autoriser Nginx dans la configuration de Cockpit**

Par sécurité, Cockpit bloque l'accès via reverse proxy s'il n'est pas explicitement configuré. On édite `/etc/cockpit/cockpit.conf` :

```bash
nano /etc/cockpit/cockpit.conf
```

```conf
[WebService]
Origins = https://admin.starfleet.lan wss://admin.starfleet.lan
ProtocolHeader = X-Forwarded-Proto
```

**VirtualHost Nginx pour `admin.starfleet.lan`**

On identifie d'abord les certificats déjà utilisés dans les autres configs Nginx :

```bash
grep -rn "ssl_certificate" /etc/nginx/
```

On réutilise la même paire de certificats (`/etc/ssl/certs/starfleet.crt` / `/etc/ssl/private/starfleet.key`) :

```bash
nano /etc/nginx/conf.d/admin.starfleet.lan.conf
```

```conf
server {
    listen 443 ssl;
    server_name admin.starfleet.lan;

    ssl_certificate /etc/ssl/certs/starfleet.crt;
    ssl_certificate_key /etc/ssl/private/starfleet.key;

    location / {
        proxy_pass https://127.0.0.1:9090;
        proxy_ssl_verify off;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Support WebSockets (requis pour le terminal Cockpit)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Test de la syntaxe et rechargement :

```bash
nginx -t && systemctl reload nginx
```

---

## 5. Pare-feu (UFW)

Installation du paquet :

```bash
apt update && apt install -y ufw
```

Politique de sécurité par défaut : on bloque tout ce qui entre, on autorise tout ce qui sort.

```bash
ufw default deny incoming
ufw default allow outgoing
```

On autorise uniquement les ports nécessaires au fonctionnement des services mis en place :

```bash
# Administration à distance
ufw allow 22/tcp comment 'SSH'
```

```bash
# Web (Nginx - www7, www8, Cockpit proxy)
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'
```

```bash
# Résolution DNS (BIND9)
ufw allow 53/udp comment 'DNS UDP'
ufw allow 53/tcp comment 'DNS TCP'
```

> **À compléter** : il manque encore les règles pour le DHCP (port 67/udp), le FTP/FTPS (port 21/tcp + la plage passive 40000-50000/tcp définie dans `vsftpd.conf`) et le LDAP (389/tcp, ou 636/tcp si LDAPS). Active `ufw enable` une fois toutes les règles nécessaires en place, pour ne pas te couper l'accès SSH ou aux services déjà configurés.

---

## 6. LDAP

*Section à compléter — envoie-moi ta doc Notion sur la mise en place de l'annuaire LDAP et l'intégration de l'authentification côté Nginx/PHP dès que tu l'as, je l'intégrerai ici dans le même format.*
