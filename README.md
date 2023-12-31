# Red Lab

> Got a raspberry-pi

> Set up ubuntu server 22.04 lts

## Table of Contents

1. [Getting system up-to-date](#getting-system-up-to-date)
2. [Pi-hole with nginx](#pi-hole-with-nginx)
3. [Pi-VPN with wireguard](#pi-vpn-with-wireguard)
4. [Monitoring with Prometheus, Grafana](#monitoring-with-prometheus-grafana)
   - [Prometheus](#prometheus-installation)
   - [Node Exporter](#node-exporter-installation)
   - [Grafana](#grafana-with-nginx-reverse-proxy)
   - [SNMP Exporter](#snmp-exporter)
5. [Memos](#memos)
6. [Self-signed certificate for internal services](#self-signed-certificate-for-internal-services)
7. [ExcaliDraw](#excalidraw)
8. [Backup](#backup)

## Getting system up-to-date.

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

- Add Dhcp address reservation at 192.168.0.200 for raspberrypi
- Install pi-hole

```bash
curl -sSL https://install.pi-hole.net | sudo bash
```

- Changing the lighttpd server port to 3141
  Open the config file

```bash
 sudo vim /etc/lighttpd/lighttpd.conf
```

Change `server.port = 3141`

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

- Add blocklist. Custom blocklist can be found at https://firebog.net/ . After updating run `pihole -g` for the updated adlist to take effect.

## [Pi-VPN](https://docs.pivpn.io/) with wireguard

- Install Pi-vpn

```bash
sudo su -
curl https://raw.githubusercontent.com/pivpn/pivpn/master/auto_install/install.sh | bash
```

- Set vpn to be `openvpn`. (wireguard mac client has issues with connectivity on ventura.)
- Set port to be `51820`
- Set ad blocking with pi-vpn to be on.

- Reboot the system.
- Add port forwarding from router to port 51820. ![image](https://github.com/reduan2660/home-lab/assets/61122163/90ce87fd-e8f2-416e-b8ed-7e7317ed6122)
- Run `pivpn -d` to check everything is configured properly.

Pi-vpn is now configured. Great!.

To add a client `pivpn add`. To view qr codes `pivpn -qr`. Config files should be available at `/home/<user>/configs`.
Wireguard client can be found here https://www.wireguard.com/install/ .

## Monitoring with Prometheus, Grafana

> Some great monitoring resources can be found [here](https://github.com/awesome-foss/awesome-sysadmin#monitoring).

### Prometheus installation

- Installation

```bash
cd ~/Downloads/
wget https://github.com/prometheus/prometheus/releases/download/v2.37.9/prometheus-2.37.9.linux-armv7.tar.gz
tar xfz prometheus-2.37.9.linux-armv7.tar.gz
mv prometheus-2.37.9.linux-armv7/ /prometheus
rm prometheus-2.37.9.linux-armv7.tar.gz

# Setting up the data directory
sudo mkdir /prometheus/data

# Setting up a Service for Prometheus
sudo vi /etc/systemd/system/prometheus.service
# Insert the code mentioned below in the file

# Start the prometheus server
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

- Service routine

```service
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=red
Restart=on-failure

ExecStart=/prometheus/prometheus \
  --config.file=/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus/data

[Install]
WantedBy=multi-user.target
```

Prometheus should be exposed in port `9090`.
To view logs `journalctl -u prometheus`.

### Node Exporter Installation

The documentation is modified for Raspberry PI (with ARMv7 architecture) from [this great documentation](https://ourcodeworld.com/articles/read/1686/how-to-install-prometheus-node-exporter-on-ubuntu-2004).

- Installation

```bash
# Install requirements
sudo apt-get install build-essential
cd ~/apps/

# Download node_exporter, extract and copy to /usr/local/bin
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-armv7.tar.gz
tar xvf node_exporter-1.6.1.linux-armv7.tar.gz
cd node_exporter-1.6.1.linux-armv7
sudo cp node_exporter /usr/local/bin


# Remove the extracted directory
cd ..
rm -rf ./node_exporter-1.3.1.linux-amd64

# Create Node Exporter User & set ownership
sudo useradd --no-create-home --shell /bin/false node_exporter
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

# Create a service. Insert the code mentioned below.
sudo vim /etc/systemd/system/node_exporter.service
```

- Service routine

```service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

- Start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

Metrics should now be available at http://192.168.0.200:9100/metrics .
Also check http://192.168.0.200:9090/targets to have a metrics entry up and running. ![image](https://github.com/reduan2660/home-lab/assets/61122163/efa02588-dec8-4c77-a2eb-ec7f5db02ac3)

### Grafana

Summary of [this official documentation](https://grafana.com/tutorials/install-grafana-on-raspberry-pi/).

- Installation

```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
```

- Start the server

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Grafana should be exposed in port `3000`.
Default credentials: admin : admin

- Create a new data source.
- Select prometheus.
- Give a name and prometheus url (http://192.168.0.200:9090 or http://localhost:9090).

- Import a new dashboard with id `1860`. Point to proper data source.
- System should now be collecting data. ![image](https://github.com/reduan2660/home-lab/assets/61122163/f9d16f77-928c-4718-9dac-46e14f6e1fd1)

### Grafana with nginx reverse proxy

> We'll be setting up grafana behind a domain `grafana.monitoring` for our local network.

- Nginx config. Write the following content at `sudo vim /etc/nginx/sites-available/grafana.monitoring`.

```conf
server {
    listen 80;
    server_name grafana.monitoring;

    location / {
        proxy_pass http://127.0.0.1:3000; # Proxy to port 3000
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- Create symlink `sudo ln -s /etc/nginx/sites-available/grafana.monitoring /etc/nginx/sites-enabled/`.
- Restart nginx `sudo systemctl restart nginx`.
- Add a new record to pi.hole's dns pointing grafana.monitoring to 192.168.0.200.

Grafana should now be available at http://grafana.monitoring. Great!.

### SNMP Exporter

- Install & configure SNMP
   - Install snmp ``` sudo apt-get install snmp snmpd ```
   - Open ``` sudo vim /etc/snmp/snmpd.conf ``` and update the sysLocation, sysContact variables
   - Restart ```sudo systemctl restart snmpd```.
   - Check ```snmpwalk -v 2c -c public localhost```.

- Install snmp exporter
```bash
mkdir ~/apps/snmp/
cd ~/apps/snmp/

wget https://github.com/prometheus/snmp_exporter/releases/download/v0.24.1/snmp_exporter-0.24.1.linux-armv7.tar.gz
tar xzf ./snmp_exporter-0.24.1.linux-armv7.tar.gz

cd snmp_exporter-0.24.1.linux-armv7/
sudo cp ./snmp_exporter /usr/local/bin/snmp_exporter
sudo cp ./snmp.yml /usr/local/bin/snmp.yml

cd /usr/local/bin/
./snmp_exporter -h
```

- Create a new user
```bash
sudo useradd --system prometheus
```

- Create a service routine at ```sudo vim /etc/systemd/system/snmp.service``` with the following content
```service
[Unit]
Description=Prometheus SNMP Exporter Service
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/snmp_exporter --config.file="/usr/local/bin/snmp.yml"

[Install]
WantedBy=multi-user.target
```
- Start the service
```bash
sudo systemctl daemon-reload
sudo systemctl enable snmp
sudo systemctl start snmp
sudo systemctl status snmp
```

SNMP exporter should spit metrics at port ```9116```.

- Add following job to prometheus ```sudo vim /prometheus/prometheus.yml```

```yml
   - job_name: 'snmp'
    static_configs:
      - targets:
        - localhost # SNMP device.
    metrics_path: /snmp
    params:
      auth: [public_v2]
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9116  # The SNMP exporter's real hostname:port.


  - job_name: 'snmp_exporter'
    static_configs:
    - targets: ['localhost:9116']
```

> A new snmp entry is up at http://192.168.0.200:9090/targets. ![image](https://github.com/reduan2660/home-lab/assets/61122163/8675c479-aa24-44ff-88ac-42e26cd3eede)

- Setup grafana dashboard
- Import ```12197``` template dashboard.
- I'll update if better config/dashboard is found.

### NGINX Exporter
- [NGINX-to-Prometheus log file exporter](https://github.com/martin-helmich/prometheus-nginxlog-exporter)
- [Install exporter](https://github.com/martin-helmich/prometheus-nginxlog-exporter#deb-and-rpm-packages)
```bash
cd ~/apps/
wget https://github.com/martin-helmich/prometheus-nginxlog-exporter/releases/download/v1.11.0/prometheus-nginxlog-exporter_1.11.0_linux_arm64.deb
sudo apt install ./prometheus-nginxlog-exporter_1.11.0_linux_arm64.deb
sudo systemctl status prometheus-nginxlog-exporter
```
If any error occurs make sure /var/log/nginx/access.log has the same group the service.

- Add following job to prometheus ```sudo vim /prometheus/prometheus.yml```
```yml

  - job_name: 'nginx_exporter'
    static_configs:
    - targets: ['localhost:4040']
```
- Restart prometheus ```sudo systemctl restart prometheus```
- A new nginx_exporter entry is up at http://192.168.0.200:9090/targets.

- Custom config ```sudo vim /etc/prometheus-nginxlog-exporter.hcl```.
```
format = "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" \"$http_x_forwarded_for\" \"$geoip_country_code\" $upstream_addr $http_response_time_seconds"
```

- Modify access log format with the following code at ```/etc/nginx/nginx.conf```.
```
log_format nginx_exporter '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for" "$geoip_country_code" $http_response_time_seconds';
access_log /var/log/nginx/access.log nginx_exporter;
```
- Install nginx add-ons ```sudo apt-get install nginx-extras```.

  
- Import the ```6482``` dashboard to grafana.
- Dashboard and logging needs to better.
## Memos

> First step towards data-ownership. Memos is a simple sticky notes app.

> More Resources can be found [here](https://github.com/awesome-selfhosted/awesome-selfhosted#note-taking--editors).

We'll be installing memos with docker compose.

- Installing docker. [Official Documentation](https://docs.docker.com/engine/install/ubuntu/)
```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- Installing Memos
```bash
sudo mkdir /memos
cd /memos
sudo vim docker-compose.yml
```
Put the following content in the docker-compose.yml file

```yml
version: "3.0"
services:
  memos:
    image: neosmemo/memos:latest
    container_name: memos
    volumes:
      - ~/.memos/:/var/opt/memos
    ports:
      - 5230:5230
```
- Write the service at ``` sudo vim /etc/systemd/system/memos.service ```
Put the following content in the file

```yml
[Unit]
Description=Docker Compose Service for memos
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/memos
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target

```
- Start the service
```bash
sudo systemctl daemon-reload
sudo systemctl enable memos
sudo systemctl start memos

sudo systemctl status memos
```
Memos should be running at port ```5230```. And data should be stored at ``` ~/.memos/ ```.


We'll be running the memos app at http://mem.os internally. And https://memos.<static-ip/url> externally.
- Write the following nginx config file at ```sudo vim /etc/nginx/sites-available/memos```.
```yml
server {
    listen 80;
    server_name mem.os;

    location / {
        proxy_pass http://127.0.0.1:5230; # Proxy to port 5230
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- Restart nginx ```sudo systemctl restart nginx```.
- To configure https://memos.<static-ip/url> externally, add the following block of code to memos's config file.
```conf
   server_name mem.os memos.<static-ip/url>;
```
- Add ssl with ```sudo certbot --nginx```.

Memos should be running at port http://mem.os internally and http://memos.<static-ip/url> externally. And data should be stored at ``` ~/.memos/ ```.
Great!.


## Self-signed certificate for internal services

> This documentation is a summary of [this great documentation by Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-nginx-in-ubuntu-20-04-1).

- Generate certificate.
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
sudo openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

- Create snippets for nginx.
Write the following content at ```sudo vim /etc/nginx/snippets/self-signed.conf```.
```conf
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

Write the following content at ```sudo vim /etc/nginx/snippets/ssl-params.conf```.
```conf
ssl_protocols TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/nginx/dhparam.pem; 
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
ssl_ecdh_curve secp384r1;
ssl_session_timeout  10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 192.168.0.200 valid=300s;
resolver_timeout 5s;
# Disable strict transport security for now. You can uncomment the following
# line if you understand the implications.
#add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
```

- Configure nginx. A sample nginx config will look like this.
```conf
server {
    server_name mem.os;

    location / {
        proxy_pass http://127.0.0.1:5230; # Proxy to port 5230
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    listen 443 ssl;

    # self-signed certificate
    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;
}
server {
    if ($host = mem.os) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name mem.os;
    return 404; # managed by Certbot
}
```

Self-signed certificate is now configured for internal network. Great!.


## ExcaliDraw

- Excalidraw doesn't have an official arm image yet, but [this image](https://hub.docker.com/r/thisisbenny/excalidraw/tags) from thisisbenny seems to work.
- Create a docker-compose.yaml file at ```vim /excalidraw/docker-compose.yaml``` with the following content.
```yaml
version: "3.8"

services:
  excalidraw:
    image: thisisbenny/excalidraw:latest
    restart: always
    ports:
      - 4200:80
    container_name: excalidraw
```
- Create a service at ``` sudo vim /etc/systemd/system/excalidraw.service ```
```bash
[Unit]
Description=Docker Compose Service for ExcaliDraw
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/excalidraw
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target

```

- Enable and Start the service.
```bash
sudo systemctl daemon-reload
sudo systemctl enable excalidraw
sudo systemctl start excalidraw

sudo systemctl status excalidraw
```

ExcaliDraw should be ready at port 4200. But we'll add an nginx block and DNS record to host in internally at excali.draw. Great!




## Backup

Creating an image of pi's current state. [We'll be following the great article from tom's hardware](https://www.tomshardware.com/how-to/back-up-raspberry-pi-as-disk-image).

0. Plug in a flash drive of the boot memory size.

1. Install ```pi-shrink```
```
cd apps/pishrink
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
sudo chmod +x pishrink.sh
sudo mv pishrink.sh /usr/local/bin
```

2. Check the mount point path of your USB drive by entering ```lsblk```

Output:
```
sda           8:0    1  57.8G  0 disk 
└─sda1        8:1    1  57.8G  0 part 
mmcblk0     179:0    0  29.8G  0 disk 
├─mmcblk0p1 179:1    0   256M  0 part /boot/firmware
└─mmcblk0p2 179:2    0  29.5G  0 part /
```

3. If the drive is not mounted, mount manually ```mount /dev/sda1 /media```.
4. Copy pi into an image: ```dd if=/dev/mmcblk0 of=/media/<image-name>.img bs=1M status=progress```.
5. Shrink the image ```sudo pishrink.sh -z /media/<image-name>.img```

That's it. This image can now be booted from with Raspberry pi imager.
