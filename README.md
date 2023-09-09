# Red Lab

> Got a raspberry-pi
> Set up ubuntu server 22.04 lts

> Getting system up-to-date.
```bash
sudo apt update
sudo apt upgrade
sudo apt install wget apt-transport-https gnupg2 software-properties-common
sudo reboot now
```

> Everything will stay inside ~/apps directory.
```bash
mkdir apps
```

## [Pi-hole](https://github.com/pi-hole/pi-hole/#one-step-automated-install) with nginx

- Addi Dhcp address reservation at 192.168.0.200 for raspberrypi
- Install pi-hole
```bash
curl -sSL https://install.pi-hole.net | sudo bash
```

- Changing the lighthttp server port to 3141
Open the config file
```bash
 sudo vim /etc/lighttpd/lighttpd.conf
```
Channge `server.port = 3141`{:.yaml}

Restart the server
```bash
sudo systemctl restart lighttpd
```

Pi-hole admin is now at http://192.168.0.200:3141/admin/ .

- Add a new domain pi.hole to Raspberry pi's local dns. ![image](https://github.com/reduan2660/home-lab/assets/61122163/61eba62e-3348-4fff-a4b4-9147a3ce3fda)

- Change router's dhcp dns server to 192.168.0.200. Reboot router
- Nginx reverse proxy to server proxy pass pi.hole to port 3141

Installing nginx
```bash
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
sudo ufw allow 'Nginx Full'
sudo systemctl status nginx
```

http://192.168.0.200/ should now serve a default nginx page.

Configuring reverse proxy
```bash
sudo vim /etc/nginx/sites-available/pi.hole
```

Put the following config 
```conf
server {
    listen 80;
    server_name pi.hole;

    location / {
        proxy_pass http://127.0.0.1:3141; # Proxy to port 3141
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Create symlink
```bash
sudo ln -s /etc/nginx/sites-available/pi.hole /etc/nginx/sites-enabled/
```

Check syntax
```bash
sudo nginx -t
```

Restart nginx server
```bash
sudo systemctl restart nginx
sudo systemctl status nginx
```

Pi-hole should now be available at http://pi.hole and served with nginx. Great!.

- Add blocklist. Custom blocklist can be found at https://firebog.net/ .
