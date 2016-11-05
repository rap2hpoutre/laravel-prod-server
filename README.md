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

## Recipe

Let's say:

- **raphael** is your name.
- **nantes** is your hostname.
- **myapp** is your app name.

You will have to replace each occurrence of theses values in the code below.

### Initialization

Connect to your server with root user. Then, name your host
```
hostname nantes
```

Install packages (PHP7, Nginx, PostgreSQL, etc.)
```sh
apt update
apt upgrade
apt install php php-mbstring php-xml php-zip git nginx redis-server postgresql fail2ban htop
```

### Composer
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
mv composer.phar /usr/local/bin/composer
```

### PostgreSQL

Create the user (don't forget to use your own password):
```
sudo -u postgres psql -c "CREATE ROLE myapp LOGIN UNENCRYPTED PASSWORD 'xxxxx' SUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION;"
```
Write down the password, you will need it later. Then restart service and create the database:
```
service postgresql restart
sudo -u postgres /usr/bin/createdb --echo --owner=xxx xxx
```
### Firewall
Configure and enable ufw
```
ufw allow 443
ufw allow 80
ufw allow 22
ufw enable
```
### Application user
You need to create an application user. It's not your name. Replace **myapp** everywhere with the name of your application.
```
useradd myapp
mkdir -p /home/myapp
chsh -s /bin/bash myapp
chown -R myapp:myapp /home/myapp
sudo -u myapp ssh-keygen -t rsa -b 4096 -C "myapp"
```
Optional step: Copy your public key, located in `/home/myapp/.ssh/id_rsa.pub`, to your github account or repository (only if the repository is private). It will be required to make continuous delivery work.

### Deployer
Deployer is a small tool for initializing and deploying Laravel applications. Install it.
```
cd /home/myapp
wget https://github.com/yepform/deployer/releases/download/v0.1.0/deployer-x86_64-unknown-linux-gnu
chmod +x deployer-x86_64-unknown-linux-gnu
mv deployer-x86_64-unknown-linux-gnu /usr/local/bin/deployer
```
Init your app with deployer (replace `https://github.com/your/app.git` with the URL to your git repo).
```
sudo -u myapp HOME=/home/myapp deployer init www https://github.com/your/app.git
```

```
### Code
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
