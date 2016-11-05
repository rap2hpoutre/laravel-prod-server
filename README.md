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

### Table of contents
## Table of Contents

- [Initialization](#initialization)
- [Composer](#composer)

### Initialization

Connect to your server with root user. Then, name your host
```bash
hostname nantes
```

Install packages (PHP7, Nginx, PostgreSQL, etc.)
```bash
apt update
apt upgrade
apt install php php-mbstring php-xml php-zip git nginx redis-server postgresql fail2ban htop
```
You can go to your server IP with your browser, you should see a welcome message from Nginx

### Composer
```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
mv composer.phar /usr/local/bin/composer
```

### PostgreSQL

Run `psql`:
```bash
sudo -u postgres psql
```
Create the user (don't forget to use your own password):
```sql
CREATE ROLE myapp LOGIN UNENCRYPTED PASSWORD 'xxxxx' SUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION;
```
Write down the password, you will need it later. Disconnect from `psql` with `\q`. Then restart service and create the database:
```bash
service postgresql restart
sudo -u postgres /usr/bin/createdb --echo --owner=xxx xxx
```
### Firewall
Configure and enable ufw
```bash
ufw allow 443
ufw allow 80
ufw allow 22
ufw enable
```
### Application user
You need to create an application user. It's not your name. Replace **myapp** everywhere with the name of your application.
```bash
useradd myapp
mkdir -p /home/myapp
chsh -s /bin/bash myapp
chown -R myapp:myapp /home/myapp
sudo -u myapp ssh-keygen -t rsa -b 4096 -C "myapp"
```
Optional step: Copy your public key, located in `/home/myapp/.ssh/id_rsa.pub`, to your github account or repository (only if the repository is private). It will be required to make continuous delivery work without connecting to your server.

### Deployer
Deployer is a small tool for initializing and deploying Laravel applications. Install it.
```bash
cd /home/myapp
wget https://github.com/yepform/deployer/releases/download/v0.1.0/deployer-x86_64-unknown-linux-gnu
chmod +x deployer-x86_64-unknown-linux-gnu
mv deployer-x86_64-unknown-linux-gnu /usr/local/bin/deployer
```
Init your app with deployer (replace `https://github.com/your/app.git` with the URL to your git repo).
```bash
sudo -u myapp HOME=/home/myapp deployer init www https://github.com/your/app.git
```
### Laravel
Copy `.env.example` to `.env`
```bash
sudo -u yepform cp /home/myapp/www/current/.env.example  /home/myapp/www/current/.env
```
Edit `/home/myapp/www/current/.env`
```bash
vi /home/myapp/www/current/.env
```
Change the values of `DB_*` with your credentials.
```
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=myapp
DB_USERNAME=myapp
DB_PASSWORD=xxxxx
```
Change `APP_ENV=local` to `APP_ENV=production` (you could disable debug too).

Use redis for everything if you want (but don't forget to include `predis/predis` to your laravel project in that case)
```
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_DRIVER=redis
```
Generate the laravel key:
```bash
cd /home/myapp/www/current
sudo -u myapp php artisan key:generate
```
Run your first migration
```bash
sudo -u myapp php artisan migrate --force
```
Authorize `www-data` to write in storage:
```bash
chown -R www-data:www-data /home/myapp/www/shared/storage
```

### Nginx
Edit `/etc/nginx/fastcgi.conf` and replace `SCRIPT_FILENAME` and `DOCUMENT_ROOT` with:
```nginx
fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
fastcgi_param DOCUMENT_ROOT $realpath_root;
```
Then create and edit `/etc/nginx/sites-available/myapp`
```nginx
server {
    listen 80 default_server;
    server_name _;
    root /home/myapp/www/current/public;
    server_tokens off;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    index index.html index.htm index.php;
    charset utf-8;
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
    error_log  /var/log/nginx/hello-error.log error;
    error_page 404 /index.php;
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
        fastcgi_index index.php;
        include fastcgi.conf;
    }
    location ~ /\.ht {
        deny all;
    }
}
```
Disable default Nginx site and enable your myapp site:
```bash
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp
```
Reload nginx
```bash
service nginx reload
```

You should be able to connect to your website with the IP adress.

### Your user

Create and configure your user (type a password when prompted)
```bash
useradd raphael
adduser raphael sudo
chsh -s /bin/bash raphael
mkdir /home/raphael
chown -R raphael:raphael /home/raphael
cp /root/.profile /home/raphael/.profile
cp /root/.bashrc /home/raphael/.bashrc
passwd raphael
```
Go to your computer (not your server) to authorize your SSH key (replace `ADDRESS` with your address and `raphael` with your name)
```bash
ssh-copy-id raphael@ADDRESS
```
Test if connection works (you should not be prompted for password)
```bash
ssh raphael@ADDRESS
```
Now you should be connected with your account. Good bye `root`, we will not use it anymore.

### Disable password auth
Edit `/etc/ssh/sshd_config` and add this
```
PasswordAuthentication no
sudo service ssh restart
```

### Https
Everything is https now, here is how to have one for your domain (replace `your.domain.com`)
```
sudo apt install letsencrypt
sudo service nginx stop
sudo letsencrypt certonly --standalone -d your.domain.com
```

Edit `/etc/nginx/sites-available/myapp` and change the first lines
```nginx
server {
    listen 443 ssl;;
    ssl_certificate /etc/letsencrypt/live/cw.yepform.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cw.yepform.com/privkey.pem;
    server_name _;
    // Keep everything after ...
```

Enable automatic renew of the certificate

```bash
sudo letsencrypt renew --dry-run --agree-tos
sudo service nginx start
```
Edit `/home/raphael/renew.sh`
```
#!/bin/bash
service nginx stop
letsencrypt renew
service nginx start
```

Edit `/etc/crontab` and add this
```
0 4 * * 0,3 root /home/raphael/renew.sh
```

Restart nginx
```
sudo service nginx start
```

** Your website is ready. Go to https://your.domain.com and it works. **

## Delivery

### Deploy
Every time you want to deploy a new version (update your website), just run:
```bash
sudo -u myapp HOME=/home/myapp deployer deploy /home/myapp/www https://github.com/your/app.git
```
### Rollback
If you want to rollback (previous version)
```bash
sudo -u myapp HOME=/home/myapp deployer rollback
```
### Setup continuous delivery (optional step)
TODO!
