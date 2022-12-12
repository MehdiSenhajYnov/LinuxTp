# SPITOFY

**Configuration de 3 Vm rocky différentes**
                
                    -spitofy.tp5.linux (1)
                    -spitofytwo.tp5.linux (2)
                    -proxy.tp5.linux (3)


## Dans nos VM serveur (1 & 2)

Nous y installerons docker a l'aide de la documentation officielle que vous pouvez trouver avec le lien suivant: https://docs.docker.com/engine/install/centos/.

**Ouverture du port 80 du Firewall**

```bash
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload
```

Nous allons créer un dossier et y mettre le fichier docker-compse.yml a l'intérieur. 

Créer les dossiers suivants:

```bash
mkdir -p jellyfin/cache
mkdir -p jellyfin/config
mkdir -p jellyfin/media
mkdir -p jellyfin/media2
```



Ensuite naviguer dans le dossier et y faire la commande suivante:

```bash
docker compose up -d
```


## Dans la VM proxy (3)

**Installation de Nginx**

```bash
sudo dnf install nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

Copier le fichier proxy.conf dans ``/etc/nginx/conf.d/``, puis remplacer les ip des lignes 2 & 3 du fichier par les ip des machines (1 & 2).

**Génération d'une clé et d'un certificat SSL**

```bash
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout spitofy.tp5.linux.key -out spitofy.tp5.linux.crt

Common Name (eg, your name or your server's hostname) []: spitofy.tp5.linux

mv spitofy.tp5.linux.crt /etc/pki/tls/certs/;
mv spitofy.tp5.linux.key /etc/pki/tls/private/; 
```

**Ouverture du Firewall**

```bash
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload
```

## MONITORING

**Dans toutes les machines**

**Installer et démarrer netdata**

```bash
dnf install epel-release -y
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh && sh /tmp/netdata-kickstart.sh
systemctl start netdata
systemctl enable netdata
systemctl status netdata
```

**Ouverture firewall**

```bash
firewall-cmd --permanent --add-port=7030/tcp
firewall-cmd --reload
```

Nous allons pouvoir surveiller l'etat actuel de la machine en allant dans l'url suivant : http://"IpDeLaMachine":7030/.



