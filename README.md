# Install and run a production server for a Laravel application

You already own:
- A domain name
- A Github account
- A fresh Ubuntu 16.04 Server with root access

You want to deploy an application for:
- Laravel with a PostgreSQL database
- Running on PHP7 and served by Nginx
- Optionally, Redis for Queue, Session and Cache

If so, let’s go, this is for you.

```
### Code

apt update
apt upgrade
apt install php php-mbstring php-xml php-zip git nginx redis-server postgresql fail2ban htop
hostname yourhostname
# COMPOSER
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
mv composer.phar /usr/local/bin/composer
# POSTGRES
sudo -u postgres psql -c "CREATE ROLE xxxxx LOGIN UNENCRYPTED PASSWORD 'xxxxx' SUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION;"
# BIEN NOTER LE MOT DE PASSE
service postgresql restart
sudo -u postgres /usr/bin/createdb --echo --owner=xxx xxx
# FIREWALL
ufw allow 22
ufw allow 80
ufw allow 443
ufw enable
# APP USER
useradd xxx
mkdir -p /home/xxx
chsh -s /bin/bash xxx
chown -R xxx:xxx /home/xxx
sudo -u xxx ssh-keygen -t rsa -b 4096 -C "xxx"
# side: add key to github account
# Deployer
cd /home/xxx
wget https://github.com/yepform/deployer/releases/download/v0.1.0/deployer-x86_64-unknown-linux-gnu
chmod +x deployer-x86_64-unknown-linux-gnu
mv deployer-x86_64-unknown-linux-gnu /usr/local/bin/deployer
sudo -u xxx HOME=/home/xxx deployer init www https://github.com/your/app.git
# Laravel
sudo -u yepform cp www/current/.env.example  www/current/.env
vi www/current/.env
# changer les valeurs pour production et postgresql et redis
cd www/current
sudo -u yepform php artisan key:generate
sudo -u yepform php artisan migrate --force
chown -R www-data:www-data home/yepform/www/shared/storage
# Nginx
# Same step as the other article
# mais juste avant : Symlink !!
vi /etc/nginx/fastcgi.conf
fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
fastcgi_param DOCUMENT_ROOT $realpath_root;


# Your User
useradd raphael
adduser raphael sudo
chsh -s /bin/bash raphael
mkdir /home/raphael
chown -R raphael:raphael /home/raphael
cp /root/.profile /home/raphael/.profile
cp /root/.bashrc /home/raphael/.bashrc
passwd raphael
# ON THE HOST
ssh-copy-id raphael@ADDRESS
ssh raphael@ADDRESS # pour essayer
# Maintenant tout le reste sera fait avec le USER
# Désactivation ssh
sudo vi /etc/ssh/sshd_config
PasswordAuthentication no
sudo service ssh restart
# let's encrypt
sudo apt install letsencrypt
sudo service nginx stop
sudo letsencrypt certonly --standalone -d app.yepform.com
# editer le fichier nginx et ajouter les clés et le 443
/etc/letsencrypt/live/app.yepform.com/fullchain.pem
listen 443 ssl;
ssl_certificate /etc/letsencrypt/live/cw.yepform.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/cw.yepform.com/privkey.pem;
sudo letsencrypt renew --dry-run --agree-tos
sudo service nginx start
# renew
0 4 * * 0,3 root /home/raphael/renew.sh
vi /home/raphael/renew.sh
---
#!/bin/bash
service nginx stop
letsencrypt renew
service nginx start
---


sudo service nginx start
#next steps
sudo -u yepform HOME=/home/yepform deployer deploy /home/yepform/www https://github.com/yepform/app.git # deployer le programme

```
