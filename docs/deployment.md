# Deployment

## Backend

### Server Setup

* Ubuntu 22.04.2 LTS (Jammy Jellyfish)
* Python 3.11.3

### Secure server (Optional)

* Make sure server has the latest software

```bash
sudo apt update && sudo apt upgrade -y
```
* `sudo apt update` - update the package list index on the system
* `sudo apt upgrade` - upgrade the installed packages on the system to their latest verions. `-y` flag will automatically answer yes to all questions

* Install `unattended-upgrades` package to automatically install security updates

```bash
sudo apt install unattended-upgrades
```
Once installed, edit the `/etc/apt/apt.conf.d/20auto-upgrades` file and include the following line:

```bash
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

* `APT::Periodic::Update-Package-Lists "1";` - update the package list index on the system (update every day)
* `APT::Periodic::Unattended-Upgrade "1";` - upgrade the installed packages on the system to their latest verions
* `APT::Periodic::AutocleanInterval "7";` - remove old packages from the cache (remove every 7 days)

Edit the `/etc/apt/apt.conf.d/50unattended-upgrades` file and include the following line to auto reboot the kernel updates (if any)

```bash
Unattended-Upgrade::Automatic-Reboot "true";
```

#### Create a Non-Root User

* Create a new user

```bash
sudo adduser <username>
sudo gpasswd -a <username> sudo
su - <username>
```

#### Setup SSH Key

* Generate SSH key on local machine

```bash
ssh-keygen -t ed25519 -C "username@email.com"
```

* Copy the public key to the server

```bash
cat ~/.ssh/id_ed25519.pub
```

* Login to the server and create `.ssh` directory

```bash
mkdir ~/.ssh
chmod 700 ~/.ssh
```

* Once this user is created, we need to disable root login and password authentication

```bash
sudo nano /etc/ssh/sshd_config
```

* Change the following lines

```bash
PermitRootLogin no
PasswordAuthentication no
```

### Install Pre-requisites

* Install python3.11

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.11 python3.11-dev python3.11-venv
```

* Install Supervisor and Nginx

```bash
sudo apt install supervisor nginx
```

* Enable Supervisor

```bash
sudo systemctl enable supervisor
sudo systemctl start supervisor
```

### Install MongoDB

* Install Gnupg and curl

```bash
sudo apt install gnupg curl
```

```bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor
```

* Create the list file `/etc/apt/sources.list.d/mongodb-org-7.0.list` for MongoDB

```bash
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

* Reload local package database

```bash
sudo apt update
```

* Install MongoDB

```bash
sudo apt-get install -y mongodb-org
```

Optional. Although you can specify any available version of MongoDB, apt-get will upgrade the packages when a newer version becomes available. To prevent unintended upgrades, you can pin the package at the currently installed version:

```bash
echo "mongodb-org hold" | sudo dpkg --set-selections
echo "mongodb-org-database hold" | sudo dpkg --set-selections
echo "mongodb-org-server hold" | sudo dpkg --set-selections
echo "mongodb-mongosh hold" | sudo dpkg --set-selections
echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
echo "mongodb-org-tools hold" | sudo dpkg --set-selections
```

* Start MongoDB

```bash
sudo systemctl start mongod
```

* Verify that MongoDB has started successfully

```bash
sudo systemctl status mongod
```

* Enable MongoDB during system startup

```bash
sudo systemctl enable mongod
```

* Restart MongoDB

```bash
sudo systemctl restart mongod
```

* MongoDB Shell
    
```bash
    mongosh
```

Note: Refer [MongoDB](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/) for more details

### Setup App

* Create a directory for the app

```bash
mkdir -p ~/apps/smart-digi-build
```

* Clone the app

```bash
git clone <git-url> ~/apps/smart-digi-build
```

* Filezilla (Optional)

If you are using Filezilla, you can use the below steps to copy the files to the server.

* Open Filezilla and connect to the server
* Navigate to the `~/apps/smart-digi-build` directory
* Copy the files from local machine to the server

* Create a virtual environment

```bash
cd ~/apps/smart-digi-build
python3.11 -m venv venv
```

* Activate the virtual environment

```bash
source venv/bin/activate
```

* Install the requirements

```bash
pip install -r requirements.txt
```

* Verify the installation

```bash
Uvicorn app.main:app --reload
```

### Configure Gunicorn

* Install Gunicorn (make sure virtual environment is activated)

```bash
pip install gunicorn
```

* Create a Gunicorn config file

```bash
sudo nano gunicorn_start
```

* Add the following lines

```bash
#!/bin/bash

NAME=<app-name>
DIR=/apps/smart-digi-build/
USER=<user-name>
GROUP=<user-group>
WORKERS=3
WORKER_CLASS=uvicorn.workers.UvicornWorker
VENV=$DIR/.venv/bin/activate
BIND=unix:$DIR/run/gunicorn.sock
LOG_LEVEL=error

cd $DIR
source $VENV

exec gunicorn app.main:app \
  --name $NAME \
  --workers $WORKERS \
  --worker-class $WORKER_CLASS \
  --user=$USER \
  --group=$GROUP \
  --bind=$BIND \
  --log-level=$LOG_LEVEL \
  --log-file=-

```

* Make the file executable

```bash
chmod u+x gunicorn_start
```

* Create run folder in the project directory
    
```bash
mkdir run
```

### Configure Supervisor

* Create directory called logs in the project directory

```bash
mkdir logs
```

* Create a Supervisor config file

```bash
sudo nano /etc/supervisor/conf.d/<app-name>.conf
```

* Add the following lines

```bash
[program:<app-name>]
command=/apps/smart-digi-build/gunicorn_start
user=<user-name>
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/apps/smart-digi-build/logs/gunicorn_supervisor.log
```

* Update Supervisor

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

* Start the app

```bash
sudo supervisorctl start <app-name>
```

* Check the status

```bash
sudo supervisorctl status <app-name>
```

* Restart the app

```bash
sudo supervisorctl restart <app-name>
```

### Configure Nginx

* Create a Nginx config file

```bash
sudo nano /etc/nginx/sites-available/<app-name>
```

* Add the following lines

```
upstream app_server {
    server unix:/apps/smart-digi-build/run/gunicorn.sock fail_timeout=0;
}

server {
    listen 80;

    # add here the ip address of your server
    # or a domain pointing to that ip (like example.com or www.example.com)
    server_name XXXX;

    keepalive_timeout 5;
    client_max_body_size 4G;

    access_log /apps/smart-digi-build/logs/nginx-access.log;
    error_log /apps/smart-digi-build/logs/nginx-error.log;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;
                        
        if (!-f $request_filename) {
            proxy_pass http://app_server;
            break;
        }
	}
}
```

Depend on the port you are using, you can change the `listen` port in the above config file.

* Create a symbolic link

```bash
sudo ln -s /etc/nginx/sites-available/<app-name> /etc/nginx/sites-enabled/<app-name>
```

* Test the Nginx config

```bash
sudo nginx -t
```

* Restart Nginx

```bash
sudo systemctl restart nginx
```

### SSL Certificate (Lets Encrypt)

* Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx
```

* Obtain a certificate

```bash
sudo certbot --nginx -d <domain-name>
```

* Test automatic renewal

```bash
sudo certbot renew --dry-run
```


## Frontend

### Setup App

* Create a directory for the app

```bash
mkdir -p ~/apps/smart-digi-build-web
```

* Clone the app

```bash
git clone <git-url> ~/apps/smart-digi-build-web
```

* Filezilla (Optional)

If you are using Filezilla, you can use the below steps to copy the files to the server.

* Open Filezilla and connect to the server
* Navigate to the `~/apps/smart-digi-build-web` directory
* Copy the files from local machine to the server

### Nginx Config

* Create a Nginx config file

```bash
sudo nano /etc/nginx/sites-available/<app-name>
```

* Add the following lines

```
server {
    listen 80;
    server_name XXXX;

    location / {
        root /apps/smart-digi-build-web/build;
        index index.html index.htm;
        try_files $uri /index.html;
    }
}
```

* Create a symbolic link

```bash
sudo ln -s /etc/nginx/sites-available/<app-name> /etc/nginx/sites-enabled/<app-name>
```

* Test the Nginx config

```bash
sudo nginx -t
```

* Restart Nginx

```bash
sudo systemctl restart nginx
```

Depend on the port you are using, you can change the `listen` port in the above config file. 

Note: When building front end application make sure to point the backend
url to the server url. For example, if the backend is running on port 8000 and the server url is `http://localhost:8000`, then the backend url should be `http://<server-ip>:8000`
