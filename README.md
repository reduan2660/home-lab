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
5. [Memos](#memos)

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

- Set vpn to be `wireguard`.
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
tar xvf tar xvf node_exporter-1.6.1.linux-armv7.tar.gz
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
        proxy_pass http://127.0.0.1:3000; # Proxy to port 3141
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
