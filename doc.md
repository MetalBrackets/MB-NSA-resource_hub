# Monter un réseau sécurisé avec OpenBSD et un server sur FreeBSD

---

## Spécifications

Mise en pratique avec la configuration de 4 VMs dans virtualbox

- VM.0 - server Gatteway - OpenBSD -> 1 brigde et 3 réseaus privés
- VM.1 - client Administration -> réseau interne
- VM.2 - server web FreeBSD -> réseau interne
- VM.3 - client Employee -> réseau interne

![Plan d'adressage](/plan-adressage.png "nsa map")

---

---

## LAN 2 -> VM.2 - server Nginx sur FreeBSD

On va déployer une page web et une base de données MySQL sur un web server Nginx.  
Let's go!

### Note à la création de la VM FreeBSD

\- use entire disk  
\- All files in one partition

\- AZERTY ⌨️ keymap -> fr  
`kbdcontrol -l /usr/share/syscons/keymaps/fr.iso.kbd`

Retirer le .iso pour éviter que ça boot sur l'image lors d'un reboot :  
\- Éteindre la vm -> Stockage + choisir le fichier -> click sur l'icon + supp

---

### 0) Se connecter en ssh car c'est un gros fichier à copier

> [ si vous avez un msg d'erreur du type "performing sanity check on sshd configuration no host key files found", la solution est de regénerer une clé ssh - `ssh-keygen -A` `service sshd restart` `service sshd status`]

---

### 1) Mettre à jour FreeBDS

`freebsd-update fetch install`  
`pkg update && pkg upgrade -y`  
`pkg install -y sudo bash curl nano`

`su`

---

### 2) Install Nginx

> Pour info, les documents racine dans Nginx se trouve dans le répertoire /usr/local/www/nginx/

`pkg install -y nginx-devel`  
`pkg install -y nginx`

`nginx -v`

`sudo sysrc nginx_enable=yes`  
`sudo service nginx start`  
`sudo service nginx status`

🗨️ en output on a "nginx is running as pid XXXX"

Aller à l'adresse ip sur un navigateur pour voir la page "Welcome to Ngninx !"  
`ifconfig`

🌐 sources [1](https://www.cyberciti.biz/faq/freebsd-install-nginx-webserver/) et [2](https://cloudo3.com/fr/cloud-compute/installer-nginx-sur-freebsd/2699)

---

### 3) Install Mysql8.0 :

`sudo pkg install -y mysql80-client mysql80-server mysqli`

`mysql --version`

`sudo sysrc mysql_enable=yes`  
`sudo service mysql-server start`  
`sudo service mysql-server status`  
🗨️ en output on a "mysql running as pid XXX

👍 Ajouter une sécure install
`sudo mysql_secure_installation`  
Définir un mot de passe et appuyez sur ENTER pour sélectionner les valeurs par défaut.

---

### 👍 3-bis) Install Mysql8.0 avec Port System

Une demi journée...  
mourir puis revivre.

---

### 4) Installation PHP 7.4

`sudo pkg install -y php74`  
`sudo pkg install php74-mysqli`

`php --version`

`sudo sysrc php_fpm_enable=yes`  
`sudo service php-fpm start`  
`sudo service php-fpm status`  
🗨️ en output on a "php_fpm is running as pid XXXX

---

### 5) Générer un certificat SSL auto-signé pour l'accès en HTTPS

**A) Générer la key et le crt SSL**

`mkdir /etc/ssl/private`

`sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt`

`sudo openssl dhparam -out /usr/local/etc/nginx/dhparam.pem 4096`  
On attend 5 minutes...  
Jolie la neige !

**A) Configuration pointant vers la key et le cert SSL**

`nano /etc/nginx/snippets/self-signed.conf`

```
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

`nano /usr/local/etc/nginx/snippets/ssl-params.conf`

```
ssl_protocols TLSv1.3;
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
```

🌐 [source](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-20-04-1)

---

### 6) Configuration de Nginx

**A) Éditer test.conf**  
`sudo nano /usr/local/etc/nginx/test.conf`

```
server {
        listen 80;
        listen 443 ssl;
        server_name 192.168.42.70;
        root /usr/local/www/nginx-dist;
        index index.php index.html index.htm;

        # https - key + cert SSL
        include snippets/self-signed.conf;
        include snippets/ssl-params.conf;

        location / {
        try_files $uri $uri/ =404;
        }

        # pour utiliser le module PHP
        location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
      }
}
```

**B) Éditer nginx.conf**  
`sudo nano /usr/local/etc/nginx/nginx.conf`

Ajoutez la ligne suivante au http {}bloc :

inclure test.conf;

Start nginx  
`sudo service nginx restart`  
Test de la configuration  
`sudo nginx -t`

🗨️ en output "nginx: the configuration file /usr/local/etc/nginx/nginx.conf syntax is ok
nginx: configuration file /usr/local/etc/nginx/nginx.conf test is successful"

Enfin recharger le service
`sudo service nginx reload`

---

### 7) Insertion du fichier data.php\*

\* Avec FileZilla ou copier/coller avec nano  
 `nano /usr/local/www/nginx-dist/data.php`

```
<?php
$servername = "localhost";
$username = "backend";
$password = "Bit8Q6a6G";
$dbname = "nsa501";

// Create connection
$conn = new mysqli($servername, $username, $password, $dbname);
// Check connection
if ($conn->connect_error) {
  die("Connection failed: " . $conn->connect_error);
}

$sql = "SELECT id, username FROM user";
$result = $conn->query($sql);

if ($result->num_rows > 0) {
  // output data of each row
  while($row = $result->fetch_assoc()) {
    echo "id: " . $row["id"]. " - Name: " . $row["username"]. "<br>";
  }
} else {
  echo "0 results";
}
$conn->close();
?>
```

**Aller voir le fichier à l'adresse ip**  
http://votre-ip/data.php  
**Vous avez cette erreur php car la db n'est pas encore installé** :

> Warning: mysqli:: **construct(): The server requested authentication method unknown to the client [caching_sha2_password] in /usr/local/www/nginx-dist/data.php on line 8
> Warning: mysqli:: **construct(): (HY000/2054): The server requested authentication method unknown to the client in /usr/local/www/nginx-dist/data.php on line 8
> Connection failed: The server requested authentication method unknown to the client

---

### 8) Insérer une base de données et créer un user 👤

**A) Création d'un user "backend"**

Se connecter avec
`sudo mysql -u root -p`  
avec le mdp `Bit8Q6a6G`  
(parce que c'est ce que j'ai mis lors de la configuration de mysql_secure_installation)  
Sinon se connecter avec `sudo mysql -u root`

- Dans la db mysql  
  `use mysql`  
  [mysql]- `CREATE USER 'backend'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Bit8Q6a6G';`

- Ensuite  
  Retourner dans mysql, puis,  
  `GRANT ALL PRIVILEGES ON nsa501.\* TO 'backend'@'localhost';`
  `FLUSH PRIVILEGES;`

**B) Création d'une db nsa501**

Se connecter avec backend  
`sudo mysql -u backend -p`  
password: `Bit8Q6a6G`

- Créer une db
  `CREATE DATABASE nsa501;`  
  `SHOW DATABASES;`

> pour info, la localisation des db -> /var/db/mysql/nsa501

**C) Insertion du .sql dans la db**

Avec FileZilla ou nano, placer ou éditer le fichier nsa501.sql dans un dossier /app à la racine.

```
SET FOREIGN_KEY_CHECKS=0;
-- ----------------------------
-- Table structure for `user`
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `email` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `password` varchar(255) COLLATE utf8_unicode_ci NOT NULL,
  `roles` longtext COLLATE utf8_unicode_ci NOT NULL COMMENT '(DC2Type:array)',
  PRIMARY KEY (`id`),
  UNIQUE KEY `UNIQ_8D93D649F85E0677` (`username`),
  UNIQUE KEY `UNIQ_8D93D649E7927C74` (`email`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;

-- ----------------------------
-- Records of user
-- ----------------------------
INSERT INTO `user` VALUES ('1', 'Marvin42', 'marvin@epitech.eu', 'toto4242', 'a:1:{i:0;s:4:\"USER\";}');
```

Insérer le sql dans la table nsa501  
`mysql nsq501 < /app/nsa501.sql`

Se mettre dans la base de données  
`mysql`  
`use nsa501`  
[nsa501]- `SHOW TABLES;`  
[nsa501]- `SELECT \* FROM user`

Marvin42 a bien été inséré.

---

Retourner sur la page web  
http://votre-ip/data.php

et voilà !

---

---

---

## VM.0 Gateway - OpenBSD

Open BSD peut être utilisé pour un accès wifi  
c'est l'un des systèmes les plus sécurisés du monde.

---

### 1) MAJ

Mettre le système à jour + correctifs du système de base  
 `syspatch`  
Mettre à jour les paquets  
 `pkg_add -Uu`  
Install nano  
 `pkg_add nano`  
 `reboot`  
Voir son ip  
 `ifconfig`  
Se connecter en SSH  
 `ssh user@votre-ip`

---

### 2) Configuration du server DHCP

🌐 sources [1](https://www.binorassocies.com/fr/blogs/2016/01/openbsd-gateway.html) et [2](https://docstore.mik.ua/manuals/openbsd/faq/fr/faq6.html)

- 3 subnets
- 1 router, une range et 1 broadcast par subnet
- 1 IP fixe pour le server
- Le masque de sous réseaux `.192` a été calculé avec https://www.subnet-calculator.com/

**A) DHCPD, la conf**  
Copier le fichier d'exemple afin de ne pas partir de zéro, puis éditer dhcpd.conf  
`cp /etc/examples/dhcpd.conf /etc/`  
`nano /etc/dhcpd.conf`

```
# Administration configuration
subnet 192.168.42.0 netmask 255.255.255.192 {

        option broadcast-address 192.168.42.63;
        option routers 192.168.42.1;
        range 192.168.42.40 192.168.42.60;
}

# Server configuration
subnet 192.168.42.64 netmask 255.255.255.192 {
        option broadcast-address 192.168.42.127;
        option routers 192.168.42.65;
        range 192.168.42.70 192.168.42.110;
        host serverfreebsd {
                hardware ethernet 08:00:27:c9:52:55;
                fixed-address 192.168.42.70;
        }
}

# Employee configuration
subnet 192.168.42.128 netmask 255.255.255.192 {
        option broadcast-address 192.168.42.191;
        option routers 192.168.42.129;
        range 192.168.42.140 192.168.42.180;

}
```

**B) Configuration des interfaces du réseau**  
Spécifier le mot "NONE" permet de configurer l'adresse de brodcast en fonction du masque de réseau. L' option netmask doit être présente pour pouvoir utiliser cette option.

`nano /etc/hostname.em1`

```
inet 192.168.42.1 255.255.255.192 NONE
```

`nano /etc/hostname.em2`

```
inet 192.168.42.65 255.255.255.192 NONE
```

`nano /etc/hostname.em3`

```
inet 192.168.42.129 255.255.255.192 NONE
```

> Pour infos :  
> dans hosts /etc/hosts il y a le localhost.  
> dans /var/db/dhcpd.leases il y a les contrats.

**C) SYSCTL**  
Configuration pour pouvoir transférer des paquets IP entre les interfaces.  
Éditer dans sysctl.conf avec cette commande rendra le changement permanent.  
`echo "net.inet.ip.forwarding=1" >> /etc/sysctl.conf`

**D) Activer et démarrer DHCPD**  
La dernière étape, activer le démarrage automatique du service **dhcpd** qui mettra à jour la passerelle ([)et le fichier /etc/resolv.conf).  
`rcctl enable dhcpd`  
`rcctl start dhcpd`

Voir les IPs écoutées  
`dhcpd em1 em2 em3`

**Voir les informations DNS**  
`cat /etc/resolv.conf`

```
nameserver 163.5.42.30 # resolvd: em0
nameserver 163.5.42.31 # resolvd: em0
search adresse-du-FAI
lookup file bind
```

---

### 3) - En option - Configuration du service DNS Unbound

Une solution de cache DNS livrée par défaut avec OpenBSD.  
Configuration dans /var/unbound/etc/unbound.conf

🌐 sources [1](https://doc.huc.fr.eu.org/fr/trad/unixsheikh.com/guide-du-routeur-openbsd/)

Copiez le fichier de configuration existant d’unbound  
`mv /var/unbound/etc/unbound.conf /var/unbound/etc/unbound.conf.backup`  
`nano /var/unbound/etc/unbound.conf`

```
server:
    # Définir les interfaces sur lesquelles le démon Unbound acceptera la requête
    interface: 127.0.0.1
    interface: ::1
    # Désactiver IPv6
    do-ip6: no
    # Activer la journalisation des requêtes DNS
    verbosity: 3
    log-queries: yes

    # Les paramètres de contrôle d'accès définissent d'où les requêtes DNS peuvent être acceptées
    access-control: 0.0.0.0/0 refuse
    access-control: 127.0.0.0/8 allow
    access-control: ::0/0 refuse
    access-control: ::1 allow
    access-control: 192.168.42.0/24 allow
    access-control: 192.168.42.64/24 allow
    access-control: 192.168.42.128/24 allow

    # "id.server" and "hostname.bind" queries are refused.
    hide-identity: yes

    # "version.server" and "version.bind" queries are refused.
    hide-version: yes

    # We want DNSSEC validation.
    auto-trust-anchor-file: "/var/unbound/db/root.key"
    val-log-level: 2

    # Enable the usage of the unbound-control command.
    remote-control:
       control-enable: yes
       control-interface: /var/run/unbound.sock

    # Use TCP for "forward-zone" requests. Useful if you are making
    # DNS requests over an SSH port forwarding.
    #
    # tcp-upstream: yes

    # Définir où les requêtes sont transmises
    # ici, par les serveurs DNS publics de Google
    forward-zone:
    name: "."
    forward-addr: 8.8.8.8
    forward-addr: 8.8.4.4
    forward-first: yes

```

**D) Activer et demarrer le service démon Unbound**  
`rcctl enable unbound`  
`rcctl start unbound`

🗨️ en output : unbound(ok)

Checker la config avec  
`unbound-checkconf /etc/unbound/unbound.conf`  
Vérifier les routes  
`netstat -rn`

---

### 4) Configuration du Packet Filter

![Packet Filter](/packet-filter.png "packet filter")

L'accès à internet pour les des LAN 1, 2 et 3 passe obligatoirement par la gateway OpenBSD.  
Pour celà, configurer le filtre de paquets pour activer NAT.  
Bloquer également par défaut toutes les connexions entrantes sur l'interface externe.  
Couper l'accès en SSH.  
Seul Administration peut atteindre le réseau de serveurs LAN 2 en SSH.  
Employee ne peut accéder au réseau serveurs LAN 2 que sur les protocoles http et https.  
LAN 1, 2 et 3 peuvent s'envoyer des pings entre sous-réseaux.

Configurer pf.conf
`nano /etc/pf.conf`

```
#Packet Filter
set skip on lo0
set block-policy drop

block

# em0
pass out on em0 from em0:network to any keep state

# em1
pass in on em1 from em1:network to any keep state
pass out on em0 from em1:network to any nat-to (em0) keep state
pass out on {em2, em3} from em1:network to any keep state

# em2
pass in on em2 from em2:network to any keep state
pass out on em0 from em2:network to any nat-to (em0) keep state

# em3
pass in on em3 from em3:network to any keep state
pass out on em0 from em3:network to any nat-to (em0) keep state
pass out on em2 proto {tcp, udp} from em3:network to port {80, 443} keep state

# ping
pass out on em3 proto icmp from any to any keep state
pass out on em2 proto icmp from any to any keep state
pass out on em1 proto icmp from any to any keep state

# ssh
block in proto tcp from any to port ssh
pass in proto tcp from em1:network to em2:network port ssh
```

Activer la configuration pf  
`pfctl -f /etc/pf.conf`

En output de cette commande, s'il y a une erreur dans la syntax, un message vous donne la ligne problématique

> `pfctl -s info` : statistiques globales  
> `pfctl -s rules` : règles de filtrage chargées en mémoire  
> `pfctl -s state` : l'état des connexions ouvertes  
> `pfctl -ss | wc -l` : connaître le nombre d'entrées dans la table d'états, en cours

---

---

## LAN 1 - VM.1 Administration + LAN 3 - VM.3 Employee

2 VMs avec interface graphique Gnome

- Verifier que vous avez les bonnes ip attribuées
- Verifier que toutes les interfaces peuvent se pinger entre elles
- Verifier que vous avez accès internet et à la page data.php
- Verifier les connections ssh

**Debuger un problème de connexion**

1. Lancer sur le server openBSD la commande  
   `tcpdump -i em1`

2. Sur la em1 (administration) par ex,  
   faire un `ping 8.8.8.8` et `ping google.com`

🗨️ Une erreur en output sur openBSD ressemble à ça :

> tcpdump: listening on em1, link-type EN10MB  
> 14:43:51.990770 192.168.42.40 > dns.google: icmp: echo request (DF)  
> 14:43:52.001102 dns.google > 192.168.42.40: icmp: echo reply  
> 14:43:52.988064 192.168.42.40 > dns.google: icmp: echo request (DF)  
> 14:43:52.999507 dns.google > 192.168.42.40: icmp: echo reply  
> From ip icmp_seq=1 Destination Host Unreachable

`ip link show dev eth1`

Lacher l'ip asoocié à l'interface  
`sudo dhclient -r`  
Recréer l'ip  
`sudo dhclient`
