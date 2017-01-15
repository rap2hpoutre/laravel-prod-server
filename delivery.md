# Delivery

## Deploy
Every time you want to deploy a new version (update your website), just run:
```bash
sudo -u myapp HOME=/home/myapp deployer deploy /home/myapp/www https://github.com/your/app.git
```
## Rollback
If you want to rollback (previous version)
```bash
sudo -u myapp HOME=/home/myapp deployer rollback
```
## Setup continuous delivery (optional step)
TODO!
