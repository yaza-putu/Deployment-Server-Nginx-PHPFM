# Server Deployment

This documents provide overview on how to initialize a deployment setup on a linux server so that it will fit with current CI / CD process. The server that we will use is Linux Ubuntu (19.10). Please check the compatibility of these commands for deploying to another linux distro / version.

## Reference

[https://www.digitalocean.com/community/tutorials/](https://www.digitalocean.com/community/tutorials/)

[how-to-install-and-configure-laravel-with-lemp-on-ubuntu-18-04](how-to-install-and-configure-laravel-with-lemp-on-ubuntu-18-04)

[https://laravel.com/docs/5.8/deployment#nginx](https://laravel.com/docs/5.8/deployment#nginx)

[https://gitlab.com/help/ci/examples/laravel_with_gitlab_and_envoy/index.md](https://gitlab.com/help/ci/examples/laravel_with_gitlab_and_envoy/index.md)

[https://medium.com/@davidhsianturi/implementasi-gitlab-ci-cd-untuk-test-dan-deploy-aplikasi-laravel-9c5dab7ce138](https://medium.com/@davidhsianturi/implementasi-gitlab-ci-cd-untuk-test-dan-deploy-aplikasi-laravel-9c5dab7ce138)

## Prepare Deployment User

Login as root, and run the command below to add deployment user, and assign permission to it. Do not forget to note the password that you have inputted.

```bash
sudo adduser maproject
sudo usermod -aG sudo maproject
```

Create www folder (if there was not), releases folder, then give access the user to the www folder.

```bash
mkdir /var/www
mkdir /var/www/releases
sudo apt install acl
sudo setfacl -R -m u:maproject:rwx /var/www
```

## Install Nginx and PHP

Open firewall for SSH.

```bash
ufw allow OpenSSH
ufw enable
```

Install nginx and add it to the firewall.

```bash
su - maproject

sudo apt update
sudo apt install nginx

sudo ufw allow 'Nginx HTTP'
sudo ufw status
```

Install PHP.

```bash
sudo add-apt-repository universe
sudo apt install php-fpm php-mysql
```

Add nginx site configuration for our site.
NOTE: Check the existence of the PHP sock file version first inside the `/var/run/php/` folder.

```bash
sudo nano /etc/nginx/sites-available/apps.maproject.net
```

It will show a text editor. Modify the contents to be like this. Put the website address e.g apps.maproject.net into *<web_address>* and server's ip address to the block *<the_ip_address_of_the_server>*

```nginx
server {
        listen 80;
        root /var/www/current/public;
        index index.php index.html index.htm index.nginx-debian.html;
        server_name <web_address> <the_ip_address_of_the_server>;

        charset utf-8;

        location / {
        try_files $uri $uri/ /index.php$is_args$args;
        }

        error_page 404 /index.php;

        location ~ \.php$ {
                fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
                include fastcgi_params;

        }

        location = /favicon.ico { access_log off; log_not_found off; }
        location = /robots.txt  { access_log off; log_not_found off; }

        location ~ /\.(?!well-known).* {
                deny all;
        }
}
```

Activate our site configuration.

```bash
sudo ln -s /etc/nginx/sites-available/apps.maproject.net /etc/nginx/sites-enabled/

sudo unlink /etc/nginx/sites-enabled/default
```

Test the configuration by running this command. If it fails then double check the configuration that we have created previously.

```bash
sudo nginx -t
```

If it succeed then we need to reload nginx to take effect of the settings.

```bash
sudo systemctl reload nginx
```

## Install COMPOSER

Install PHP tools required by the composer.

```bash
sudo apt install curl php-cli php-mbstring git unzip

cd ~
```

Download the composer installer.

```bash
curl -sS https://getcomposer.org/installer -o composer-setup.php
```

Verify the installer by copying the hash key that we can get through accessing <https://composer.github.io/pubkeys.html>. The hash is placed under *Installer Signature (SHA-384)*. Copy the hash key, put into the command below and run.

```bash
HASH=<the copied hash key from the website>
```

Run this command to verify the installation. It must return something like "Installer verified".

```bash
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```

If it is verified, then we can start the installation.

```bash
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

## Install LARAVEL

Install PHP tools required by Laravel

```bash
sudo apt update
sudo apt install php-mbstring php-xml php-bcmath
```

Install the dependencies required by the lib <https://packagist.org/packages/phpoffice/phpspreadsheet> in the project

```bash
sudo apt install php-gd php-zip
```

## CI / CD Preparation

Make sure that you are in deployment (maproject) account that we have created previously, so that the path to the .ssh folder will be `/home/maproject/.ssh/`. Switch user if it does not.

```bash
su maproject
```

Create a new ssh key that will be used for the deployment. Just press enter (empty) to leave as default, if it asks for location to save the key, also press enter (empty passphrase) if it asks to enter passphrase.

```bash
ssh-keygen -t rsa -b 4096 -C "maproject@maproject.net"
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

Run this command below to display the content of the private key.

```bash
cat ~/.ssh/id_rsa
```

Copy the output (beware of the line endings. It is best to copy first to notepad to check the content validity)
They can be added per project by navigating to the project's *Settings > CI/CD > Variables*.

To the field KEY, add the name *SSH_PRIVATE_KEY* (for QA) or *SSH_PRIVATE_KEY_PROD* (for PROD), and to the VALUE field, paste the private key you have copied earlier.
This variable has been used in the .gitlab-ci.yml (please check the deployment job), to easily connect to our remote server as the deployer user without entering its password.

Next, run this command to display the public key. Also beware of the line endings.

```bash
cat ~/.ssh/id_rsa.pub

OR

vim ~/.ssh/id_rsa.pub
```

We also need to add this public key to Project > Settings > Repository as a Deploy Key, which gives us the ability to access our repository from the server through SSH protocol.

Next, check if the ssh is working. Answer yes if it asks `Are you sure you want to continue connecting (yes/no)?`. We need to do this to add GitLab.com to the known hosts, otherwise it will block the ssh connection.

```bash
cd ~
git clone git@gitlab.com:maproject/commission-app.git
```

Next, Copy `commision-app/storage` folder into `/var/www/storage`. You can use SCP to copy that.
Then, make sure that maproject user has full access to that storage/ folder

```bash
cd /var/www
mkdir storage
chmod 777 storage
sudo setfacl -R -m u:maproject:rwx storage
sudo chown -R www-data:www-data storage
cd ~/commision-app
cp -R storage/. /var/www/storage/
```

Next, prepare an .env file in `/var/www` folder. We can use the contents of .env.example.

Note that you will also need to generate a new deployment key 'php artisan key:generate' and then put it inside that .env file.

```bash
cd /var/www
nano .env
```

Last, in Gitlab Projects, go to Settings > CI / CD > Variables. Add two variables required for the script for QA and Prod Deployment. Replace ip_address_qa and ip_address_prod with the correct ip from each of the servers.

- Key : SERVER_SETUP_QA, Value : maproject@ip_address_qa
- Key : SERVER_SETUP_PROD, Value : maproject@ip_address_prod

### Create or Update Docker Image for Build If It Is Required

This is the step to create (if we have not) or update the docker image used by the gitlab CI / CD.

Navigate to the project folder and then run this command

```bash
docker login registry.gitlab.com
docker build -t registry.gitlab.com/maproject/commission-app .
docker push registry.gitlab.com/maproject/commission-app
```

## CRON JOB

We need to put a new cron schedule which will execute the Laravel's schedule job.

Run a crontab editor.

```bash
su maproject
crontab -e
```

Put this line into the crontab content.

```bash
# Run laravel job
* * * * * cd /var/www/current && php artisan schedule:run >> /dev/null 2>&1
```

## HTTPS

Reference : <https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx>

We need to run these commands to enable SSL (HTTPS). This will use certificate provided by Let's Encrypt. Please be noted that we need to have the public domain name (DNS) available to our site first.

Run these commands to install required tools. Note that there might be a `404  Not Found [IP: 91.189.95.83 80]` error but we can still continue.

```bash
sudo apt-get update
sudo apt-get install software-properties-common
sudo add-apt-repository universe
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot python-certbot-nginx
```

Next, run these commands below to install the SSL certificate. During the process, it will ask you to choose which domain that needs to be SSL and whether it needs to enable auto redirect from HTTP to HTTPS. Choose what it is best.

```bash
sudo certbot --nginx
```

Run this command to test the certificate auto-renew facility.

```bash
sudo certbot renew --dry-run
```

Last, modify site name in the .env file to put `https` scheme and add firewall rules for https.

```bash
sudo ufw allow '443/tcp'
sudo ufw status
```

## Deploy Database Schema Changes

Currently the CI / CD script does not support auto deployment of database schema changes (migration) into server. So what we need to do is to remote (ssh) into that server, navigate into `/var/www/current` folder and then run the migration command `php artisan migrate` or `php artisan db:seed` manually.

This is the safest approach to avoid destruction to the existing data which will cause unexpected error, because you have a total control on what needs to be migrated.

## SMTP and other ports

Sometimes you will need to open some ports for some services, for example SMTP. Since we have enabled the firewall, we need to specifically instruct ufw to open a specific port. Firstly you can check list of the ports that have been opened by running command `sudo ufw status`, and then run `sudo ufw allow THE_PORT` for example `sudo ufw allow 234` to open port 234. If we forget to do this, we will have our service behaving unexpectedly.

## Auto versioning by semantic-release

This project utilizes [semantic-release](https://www.npmjs.com/package/semantic-release) npm package to automatic versioning based on [semantic versioning](https://semver.org/). This process installed inside the Gitlab CI / CD process and requires some keys to be added into the Settings > CI / CD > Variables.

- Generate a [personal access token](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html) and then add it as ACCESS_TOKEN variable. You need to add access permission read & write repo and api.

Reference: [https://medium.com/faun/automate-your-releases-versioning-and-release-notes-with-semantic-release-d5575b73d986](https://medium.com/faun/automate-your-releases-versioning-and-release-notes-with-semantic-release-d5575b73d986)
