# Installation
This document provides a guide on how to install Moonraker & Mainsail on a Raspberry
Pi running Raspian/Rasperry Pi OS. Other SBCs and/or linux distributions
may work, however they may need a custom install script.

**Please don't use a octopi image!!!**

## Installing operation system image
It's recommend to use a clean Raspberry Pi OS 32-bit Lite (previosly called Raspbian)
image. You can download it [here](https://downloads.raspberrypi.org/raspios_lite_armhf_latest).

More information about the installtion of Raspberry Pi OS are
[here](https://www.raspberrypi.org/documentation/installation/installing-images/).

**If you have finished the installation and connected via ssh, you can continue.**

## Install Requirements
```
sudo apt update && sudo apt upgrade
sudo apt install git dfu-util
```

## Install Klipper
Moonraker is still in alpha development, and thus some of its dependencies
in Klipper have yet to be merged.  Until this has been done it will be
necessary to add a remote and work off a developmental branch of Klipper
to correctly run Moonraker.

In this step, we install klipper and change to the development branch of Klipper.
```
cd ~
git clone https://github.com/KevinOConnor/klipper
cd ~/klipper
git remote add arksine https://github.com/arksine/klipper
git fetch arksine
git reset --hard arksine/dev-moonraker-testing
./scripts/install-octopi.sh
```

Note that you are now in a detached head state and you cannot pull. Any
time you want to update to the latest version of this branch you have to
`git fetch arksine` and `git reset --hard arksine/dev-moonraker-testing`.

Now you can copy your `printer.cfg` in your home directory and restart klipper.
Please check the `klippy.log`, if Klipper starts correctly and then continue
the guide.

## Install Moonraker (API)
You can now install the Moonraker application:
```
cd ~
git clone https://github.com/Arksine/moonraker.git
```

Finally, run Moonraker's install script:
```
cd ~/moonraker/scripts
./install-moonraker.sh
```

When the script completes it should start both Moonraker and Klipper. In
`klippy.log` you should find the following entry:\
`Moonraker: server connection detected`

### Configure Moonraker
All other configuration is sent to the server via Klippy, thus it is done
in printer.cfg. A basic configuration that authorizes clients on a range
from 192.168.1.1 - 192.168.1.254 is as follows.

Edit your printer.cfg with nano ~/printer.cfg and add the following lines:
```
[moonraker]
trusted_clients:
    192.168.1.0/24
```

We also need a `virtual_sdcard` for store the gcode files:
```
[virtual_sdcard]
path: /home/pi/sdcard
```

Finally, we create the virtual_sdcard directory:
```
mkdir ~/sdcard
```

Restart klipper (sudo service klipper restart`) and check your klippy.log if klipper is starting correct again.

When klipper is running, open the url `http://<printer-ip>:7125/printer/info` in your browser. If you see a content like this

```
{"result": {"hostname": "voron250", "error_detected": false, "version": "v0.8.0-643-g528f9f25", "is_ready": true, "message": "Printer is ready", "cpu": "4 core ARMv7 Processor rev 4 (v7l)"}}
```

your API is working!

For more informations about the Moonraker installation or configuration
look at the Repository from Arksine:
[Moonraker Repository install Documentation](https://github.com/Arksine/moonraker/blob/master/docs/installation.md)

## Install Mainsail (webinterface)

### Install webserver & reverse proxy (nginx)
nginx is important to mount all components on port 80 and host the static files from Mainsail.
To install nginx you only need to execute:

```
sudo apt install nginx
```

now we create the config file with:

```
sudo nano /etc/nginx/sites-available/mainsail
```

and fill it with the following content:
```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream apiserver {
    #edit your api port here
    ip_hash;
    server 127.0.0.1:7125;
}

upstream mjpgstreamer {
    #edit your webcam port here
    ip_hash;
    server 127.0.0.1:8081;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;

    access_log /var/log/nginx/mainsail-access.log;
    error_log /var/log/nginx/mainsail-error.log;

    #web_path from mainsail static files
    root /home/pi/mainsail;

    index index.html;
    server_name _;

    #max upload size for gcodes
    client_max_body_size 200M;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /printer {
        proxy_pass http://apiserver/printer;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
    }

    location /api {
        proxy_pass http://apiserver/api;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
    }

    location /access {
        proxy_pass http://apiserver/access;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
    }

    location /websocket {
        proxy_pass http://apiserver/websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 86400;
    }

    location /machine {
        proxy_pass http://apiserver/machine;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
    }

    location /server {
        proxy_pass http://apiserver/server;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Scheme $scheme;
    }

    location /webcam/ {
        proxy_pass http://mjpgstreamer/;
    }
}
```

Create directory for static files and active nginx config:
```
mkdir ~/mainsail
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/mainsail /etc/nginx/sites-enabled/
sudo service nginx restart
```

Now you can check again the API if it works with the reverse proxy.
Open the url `http://<printer-ip>/printer/info` in your browser. if you see a content like this:
```
{"result": {"hostname": "voron250", "error_detected": false, "version": "v0.8.0-479-gd586fb06", "is_ready": true, "message": "Printer is ready", "cpu": "4 core ARMv7 Processor rev 4 (v7l)"}}
```

Now we can install Mainsail (static httpdocs).

### Install httpdocs
Now you can download the current mainsail static data
```
cd ~/mainsail
wget -q -O mainsail.zip https://github.com/meteyou/mainsail/releases/download/v0.1.2/mainsail-beta-0.1.2.zip && unzip mainsail.zip && rm mainsail.zip
```
Now it should be possible to open the interface: `http://<printer-ip>/`.

### Change the Hostname (optional)
to use the hostname instate of the ip, you can install the avahi-daemon:
```
sudo apt install avahi-daemon 
```

and you can config your hostname:
```
sudo raspi-config
```
in 2 Network Options > N1 Hostname you can edit your hostname of your raspberry pi.
After a reboot you can use `http://<hostname>.local/` to open the webinterface.