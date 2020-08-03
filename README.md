# Setup an Eth2 Validator System on Ubuntu
These instructions represent my current process for setting up an Eth2 staking system on Ubuntu 20.04 LTS on an Intel NUC 10i5FNK with 512GB SSD and 16GB RAM. These instructions are primarily for my own purposes, so that I can recreate my environment if I need to. They are not intended to represent best practices and may not be applicable to your hardware, software, or network configuration. There are many other good sources for instructions on setting up these services, and those may be more generally written and applicable.

Setup includes installation and configuration of the following services, including setting up systemd to automatically run services, where applicable:

- Prysm Beacon Chain
- Prysm Validator
- geth 
- Prometheus
- Grafana
- node_exporter
- blackbox_exporter
- eth2stats

Steps to install and configure all software have been copied from or inspired by a number of sources, which are cited at the end of this file. Discord discussions may have provided additional details or ideas. In addition, though I have never been a professional Linux administrator, I have many years experience running Linux servers for a variety of public and private hobby projects, which may have informed some of my decisions, for better or worse.

This process assumes starting from first login on a clean Ubuntu 20.04 LTS installation, and were last tested on August 1, 2020.

## Prerequisities

### Software Update
After an initial install, it is a good idea to update everything to the latest versions.
```console
sudo apt-get update
sudo apt-get upgrade
sudo reboot
```

### Set Time Zone
Run the following command to see the list of time zones, then copy the appropriate time zone to your clipboard.
```console
timedatectl list-timezones
```

Run the following command, replacing `<SELECTED_TIMEZONE>` with the time zone you have copied onto your clipboard.
```console
sudo timedatectl set-timezone <SELECTED_TIMEZONE>
```

### net-tools
Installing net-tools in order to determine network device via ifconfig.
```console
sudo apt-get install net-tools
```

### git
Installing git should not be necessary if you are running on Ubuntu Server. It should already be installed.
```console
sudo apt-get install git
```

### make
```console
sudo apt-get install make
```

## Prysm

### Create User Accounts
```console
sudo adduser --home /home/beacon --disabled-password --gecos 'Ethereum 2 Beacon Chain' beacon
sudo adduser --home /home/validator --disabled-password --gecos 'Ethereum 2 Validator' validator
sudo -u beacon mkdir /home/beacon/bin
sudo -u validator mkdir /home/validator/bin
```

### Install prysm.sh

```console
cd /home/validator/bin
sudo -u validator curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && sudo -u validator chmod +x prysm.sh
cd /home/beacon/bin
sudo -u beacon curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output prysm.sh && sudo -u beacon chmod +x prysm.sh
```

### Set Up systemd Service File
This sets up prysm.sh to automatically run on start. This file is slightly different than the version under the Building Prysm section.

#### Beacon Chain
```console
sudo nano /etc/systemd/system/beacon-chain.service
```

Copy and paste the following text into the beacon-chain.service file.

```
[Unit]
Description=Ethereum 2 Beacon Chain
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=beacon
ExecStart=/home/beacon/bin/prysm.sh beacon-chain --config-file /home/beacon/prysm-beacon.yaml

[Install]
WantedBy=multi-user.target
Alias=beacon
```

#### Validator

```console
sudo nano /etc/systemd/system/validator.service
```

Copy and paste the following text into the validator.service file.

```
[Unit]
Description=Ethereum 2 Validator
Wants=beacon-chain.service
After=beacon-chain.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=validator
ExecStart=/home/validator/bin/prysm.sh validator --config-file /home/validator/prysm-validator.yaml

[Install]
WantedBy=multi-user.target
```

### Create Prysm Configuration Files

#### prysm-beacon.yaml

```console
sudo -u beacon nano /home/beacon/prysm-beacon.yaml
```

Copy and paste the following text into the prysm-beacon.yaml configuration file.


```
datadir: "/home/beacon/prysm"
p2p-host-ip: "XXX.XXX.XXX.XXX"
http-web3provider: "http://YYY.YYY.YYY.YYY:8545"
monitoring-host: "0.0.0.0"
p2p-tcp-port: 13000
p2p-udp-port: 12000
```

If you have a dynamic IP addres, remove the `p2p-host-ip` line. Otherwise, update `XXX.XXX.XXX.XXX` to your external IP address.
Update `YYY.YYY.YYY.YYY` to the IP address of your Eth1 node, or remove the `http-web3provider` line entirely to use the default Eth1 node.
The `p2p-tcp-port` and `p2p-udp-port` lines are optional if you use the default values of 13000 and 12000, respectively.

Change permissions of the file.

```console
sudo -u beacon chmod 600 /home/beacon/prysm-beacon.yaml
```

#### prysm-validator.yaml

```console
sudo -u validator nano /home/validator/prysm-validator.yaml
```

Copy and paste the following text into the prysm-beacon.yaml configuration file.

```
monitoring-host: "0.0.0.0"
graffiti: "YOUR_GRAFFITI_HERE"
beacon-rpc-provider: "localhost:4000"
```

`graffiti` can be changed to whatever text you would prefer. To get a POAP badge, follow the instructions at [https://beaconcha.in/poap](https://beaconcha.in/poap) and replace `YOUR_GRAFFITI_HERE` with the value on that site.

Change permissions of the file.

```console
sudo -u validator chmod 600 /home/validator/prysm-validator.yaml
```

### Make Validator Deposits and Install Keys

Follow the latest instructions at [medalla.launchpad.ethereum.org](https://medalla.launchpad.ethereum.org).

Python3 and git should already be installed.

```console
cd
sudo apt-get python3-pip
git clone https://github.com/ethereum/eth2.0-deposit-cli.git
cd eth2.0-deposit-cli
sudo ./deposit.sh install
./deposit.sh --num_validators NUMBER_OF_VALIDATORS --chain medalla
```

Change the `NUMBER_OF_VALIDATORS` to the number of validators you want to create. Follow the prompts and instructions.

**BACKUP YOUR MNEMONIC AND PASSWORD!**

The next step is to upload your deposit data file to the launchpad site. If you are using Ubuntu Server, you can either open up the deposit data file and copy it to a file on your desktop computer with the same name, or you can use scp or an equivalent tool to copy the deposit data to your desktop computer.

Follow the instructions by dragging and dropping the deposit file into the launchpad site. Then continue to follow the instructions until your deposit transaction is successful.

You should be in the `eth2.0-deposit-cli/validator_keys` directory for the next commands.

```console
sudo -u validator /home/validator/bin/prysm.sh validator accounts-v2 import --keys-dir=.
```

Follow the prompts. The default wallet directory should be `/home/validator/.eth2validators/prysm-wallet-v2`, and the default passwords directory should be `/home/validator/.eth2validators/prysm-wallet-v2-passwords`. Use the same password used when you were prompted for a password while running `./deposit.sh --num_validators NUMBER_OF_VALIDATORS --chain medalla`.

### Start Beacon Chain and Validator

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start beacon-chain validator
sudo systemctl enable beacon-chain validator
```

## geth
It is recommended that you run your own geth full node. For testnets, a default node is provided by Prysmatic Labs, but this may not be available for the mainnet launch.

### Install geth

```console
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install ethereum
```

### Create User Account

```console
sudo adduser --home /home/geth --disabled-password --gecos 'Go Ethereum Client' geth
```

### Set Up systemd Service File
This sets up geth to automatically run on start.

```console
sudo nano /etc/systemd/system/geth.service
```

Copy and paste the following text into the geth.service file.

```
[Unit]
Description=Ethereum 1 Go Client
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=geth
WorkingDirectory=/home/geth
ExecStart=/usr/bin/geth --goerli --http --http.addr 0.0.0.0

[Install]
WantedBy=multi-user.target
```

### Start geth

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start geth
sudo systemctl enable geth
```

## Monitoring
The following will set up prometheus for collecting data, grafana for displaying dashboards, node_exporter for providing system data to prometheus, and blackbox_exporter for providing ping data to prometheus.

node_exporter and blackbox_exporter are optional, though some charts on the dashboard provided may need to be removed if those tools are not used. The prometheus configuration file may also need to be updated.

### Prometheus
#### Create User Account
```console
sudo adduser --system prometheus --group --no-create-home
```

#### Install Prometheus

Find the URL to the latest amd64 version of Prometheus at https://prometheus.io/download/. In the commands below, replace any references to the version 2.19.2 to the latest version available.

```console
cd
wget https://github.com/prometheus/prometheus/releases/download/v2.19.2/prometheus-2.19.2.linux-amd64.tar.gz
tar xzvf prometheus-2.19.2.linux-amd64.tar.gz
cd prometheus-2.19.2.linux-amd64
sudo cp promtool /usr/local/bin/
sudo cp prometheus /usr/local/bin/
sudo chown root.root /usr/local/bin/promtool /usr/local/bin/prometheus
sudo chmod 755 /usr/local/bin/promtool /usr/local/bin/prometheus
cd
rm prometheus-2.19.2.linux-amd64.tar.gz
```

#### Configure Prometheus
```console
sudo mkdir -p /etc/prometheus/console_libraries /etc/prometheus/consoles /etc/prometheus/files_sd /etc/prometheus/rules /etc/prometheus/rules.d
```

Copy and paste the following text into the prometheus.yml configuration file:

```console
sudo nano /etc/prometheus/prometheus.yml
```

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'beacon'
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:8080']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:9100']
  - job_name: 'validator'
    scrape_interval: 5s
    static_configs:
    - targets: ['localhost:8081']
  - job_name: 'ping_google'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - 8.8.8.8
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
  - job_name: 'ping_cloudflare'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
        - 1.1.1.1
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
```

Change the ownership of the prometheus directory.

```console
sudo chown -R prometheus.prometheus /etc/prometheus
```

#### Data Directory
```console
sudo mkdir /var/lib/prometheus
sudo chown prometheus.prometheus /var/lib/prometheus
sudo chmod 755 /var/lib/prometheus
```

#### Set Up systemd Service
```console
sudo nano /etc/systemd/system/prometheus.service
```

Copy and paste the following text into the prometheus.service file.
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start prometheus.service
sudo systemctl enable prometheus.service
```

### Grafana
```console
cd
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana-enterprise
```

#### Setup systemd

**Optional:** Edit the `grafana-server.service` file to add "grafana" as an alias to grafana server. I generally forget that the default name for this service is `grafana-server`.

```
sudo nano /lib/systemd/system/grafana-server.service
```

At the end of this file, in the `[Install]` section, add the following line:

```
Alias=grafana.service
```

Start the service.

```console
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Login to grafana at http://XXX.XXX.XXX.XXX:3000/, replacing `XXX.XXX.XXX.XXX` with the IP address of your server. If you do not know the IP address, run `ifconfig`.

Default username `admin`. Default password `admin`. Grafana will ask you to set a new password.

#### Setup Prometheus Data Source
1. On the left-hand menu, hover over the gear menu and click on Data Sources.
2. Then click on the Add Data Source button.
3. Hover over the Prometheus card on screen, then click on the Select button.
4. Enter `http://localhost:9090/` into the URL field, then click Save & Test.

#### Install Grafana Dashboard
1. Hover over the plus symbol icon in the left-hand menu, then click on Import.
2. Copy and paste the dashboard at [https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source.json](https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source.json) into the "Import via panel json" text box on the screen.
3. Then click the Load button.
4. Then click the Import button.

Note: At this point in the process, any widgets showing details from the validator will show "N/A", because the validator still has no keys configured. As soon as keys are configured for the validator, the validator details should begin to show up.

#### Final Grafana Dashboard Configuration
A few of the queries driving the Grafana dashboard may need different settings, depending on your hardware.

##### Network Traffic Configuration
To ensure that network traffic is correctly reflected on your Grafana dashboard, update the network interface in the Network Traffic widget. Run the following command to find your Linux network device.

```console
ifconfig
```

Output of the command should look like the following:
```
eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.10  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::1e69:7aff:fe63:14b0  prefixlen 64  scopeid 0x20<link>
        ether 1c:69:7a:63:14:b0  txqueuelen 1000  (Ethernet)
        RX packets 238936  bytes 78487335 (78.4 MB)
        RX errors 0  dropped 1819  overruns 0  frame 0
        TX packets 257824  bytes 112513038 (112.5 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 16  memory 0x96300000-96320000

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 39805  bytes 29126770 (29.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 39805  bytes 29126770 (29.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Of the two entries shows above, the first lists my IP address on the second line, network interface `eno1`. Find the entry that represents the network connection you want to monitor and copy the device name, which is the part before the colon on the first line of each entry. In my case the value is `eno1`.

1. Go to the Grafana dashboard previously installed
2. Find the Network Traffic widget, and open the drop down that can be found by the Network Traffic title.
3. Click Edit.
4. There will be four references to `eno1` in the queries that appear. Replace all four with the name of the network interface you found in the `ifconfig` command.

### node_exporter
#### Create User Account
```console
sudo adduser --system node_exporter --group --no-create-home
```

#### Install node_exporter
```console
cd
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.0/node_exporter-1.0.0.linux-amd64.tar.gz
tar xzvf node_exporter-1.0.0.linux-amd64.tar.gz
sudo cp node_exporter-1.0.0.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm node_exporter-1.0.0.linux-amd64.tar.gz
```

#### Set Up System Service
```console
sudo nano /etc/systemd/system/node_exporter.service
```

Copy and paste the following text into the node_exporter.service file.

```
[Unit]
Description=Node Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start node_exporter.service
sudo systemctl enable node_exporter.service
```

### blackbox_exporter
#### Create User Account
```console
sudo adduser --system blackbox_exporter --group --no-create-home
```

#### Install blackbox_exporter
```console
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.16.0/blackbox_exporter-0.16.0.linux-amd64.tar.gz
tar xvzf blackbox_exporter-0.16.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.16.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo chown blackbox_exporter.blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chmod 755 /usr/local/bin/blackbox_exporter
```

Allow blackbox_exporter to ping servers.
```console
sudo setcap cap_net_raw+ep /usr/local/bin/blackbox_exporter
```

```console
rm blackbox_exporter-0.16.0.linux-amd64.tar.gz
```

#### Configure blackbox_exporter

```console
sudo mkdir /etc/blackbox_exporter
sudo chown blackbox_exporter.blackbox_exporter /etc/blackbox_exporter
```

```console
sudo nano /etc/blackbox_exporter/blackbox.yml
```

Copy and paste the following text into the blackbox.yml file. 

```
modules:
        icmp:
                prober: icmp
                timeout: 10s
                icmp:
                        preferred_ip_protocol: ipv4
```

`sudo chown blackbox_exporter.blackbox_exporter /etc/blackbox_exporter/blackbox.yml`


#### Set Up System Service
`sudo nano /etc/systemd/system/blackbox_exporter.service`

Copy and paste the following text into the blackbox_exporter.service file.

```
[Unit]
Description=Blackbox Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=blackbox_exporter
ExecStart=/usr/local/bin/blackbox_exporter --config.file /etc/blackbox_exporter/blackbox.yml

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start blackbox_exporter.service
sudo systemctl enable blackbox_exporter.service
```

## Optional

### eth2stats
eth2stats reports some basic beacon chain statistics to eth2stats.io. This service may not be supported in the long term.

#### Create User Account
```console
sudo adduser --system eth2stats --group --no-create-home
```

#### Install go
```console
sudo apt-get install golang-1.14-go

# Create a symlink from /usr/bin/go to the new go installation
sudo ln -s /usr/lib/go-1.14/bin/go /usr/bin/go
```

#### Install eth2stats
```console
cd
git clone https://github.com/alethio/eth2stats-client
cd ~/eth2stats-client
make build
sudo cp eth2stats-client /usr/local/bin
sudo chown root.root /usr/local/bin/eth2stats-client
sudo chmod 755 /usr/local/bin/eth2stats-client
```

#### Create Data Directory
```console
sudo mkdir /var/lib/eth2stats
sudo chown eth2stats.eth2stats /var/lib/eth2stats
sudo chmod 755 /var/lib/eth2stats
```

#### Set Up System Service

```console
sudo nano /etc/systemd/system/eth2stats.service
```

Copy and paste the following text into the validator.service file.

```
[Unit]
Description=eth2stats
After=beacon-chain.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
WorkingDirectory=/var/lib/eth2stats/
User=eth2stats
ExecStart=/usr/local/bin/eth2stats-client run --v --eth2stats.node-name="NODE_NAME" --eth2stats.addr="grpc.medalla.eth2stats.io:443" --beacon.metrics-addr="http://localhost:8080/metrics" --eth2stats.tls=true --beacon.type="prysm" --beacon.addr="localhost:4000"

[Install]
WantedBy=multi-user.target
```

Replace `NODE_NAME` with the name you would like to appear on eth2stats.io.

These instructions were written during the Medalla testnet. The command-line flag `--eth2stats.addr` may need to be updated to a new address for later testnets or the mainnet.

```console
sudo systemctl daemon-reload
sudo systemctl enable eth2stats.service
sudo systemctl start eth2stats.service
```

## Router Configuration
You may need to configure your router to forward the following ports to your staking system. See your router documentation for details.

Prysm Beacon Chain: 12000/udp
Prysm Beacon Chain: 13000/tcp
geth: 30303/udp
geth: 30303/tcp


## Security
### SSH

The following changes can be made to increase the security of SSH, but are not required.

```console
sudo nano /etc/ssh/sshd_config
```

Add the following lines, but replacing <LOGIN> with your login. You are not logging in to ssh with root, right? If you are, you probably don't want to add the `AllowUsers` and `PermitRootLogin` lines below.

```
AllowUsers <LOGIN>
PermitEmptyPasswords no
PermitRootLogin no
Protocol 2
```

**Optional:** I prefer to change the default SSH port to a non-standard port. Do not forget what you change this to. Find the following line, uncomment it line by removing the "#", and replace "22" with your preferred port.

```
#Port 22
```

```console
sudo reboot
```

### Firewall
If your staking system is behind a router with a firewall, you may not want to add another level of firewall to your network security. This section may be skipped.

The following commands set up the minimal firewall rules necessary to run the Prysm beacon-chain and geth

```console
# beacon chain
sudo ufw allow 12000/udp
sudo ufw allow 13000/tcp

# geth
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp

# grafana
sudo ufw allow 3000/tcp
```

Run the following command to set up firewalls rules for SSH. If you changed your default SSH port above, change the `22` in this command to the port you are using.

```console
# ssh
sudo ufw allow 22/tcp
```

Set up default firewall rules and enable the firewall.

```console
# Defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

The following commands open up the remaining ports that are used by the software in this set of instructions. These ports are typically used only by other software internal to the staking system, and do not need to be opened on the firewall unless you would like direct access to some of the administrative/metrics pages, or if systems external to your staking system will be services on your staking system.

```console
# beacon chain
#   - This only needs to be enabled if external validators will be accessing this beacon chain.
sudo ufw allow 4000/tcp

# node_exporter
#   - This only needs to be enabled if you want to access node_exporter stats directly.
sudo ufw allow 9100/tcp

#geth
#   - This only needs to be enabled if external beacon chains will be accessing this geth full node.
sudo ufw allow 8545/tcp

# beacon-chain metrics
#   - This only needs to be enabled if you want to access beacon-chain stats directly.
sudo ufw allow 8080/tcp

# blackbox_exporter
#   - This only needs to be enabled if you want to access blackbox_exporter stats directly.
sudo ufw allow 9115/tcp

# prometheus
#   - This only needs to be enabled if you want to access prometheus directly.
sudo ufw allow 9090/tcp
```


## Future Updates

There are at least one area where I may expand on my system configuration or instructions, but I have not pursued it yet.

 - SSH Key-Based Login
   - This seems to be a good security move, but it also seems to be the perfect way to get me locked out of my own system. I have never set this up before, but may look into it.


## Sources/Inspiration
Prysm: [https://docs.prylabs.network/docs/getting-started/](https://docs.prylabs.network/docs/getting-started/)

Go: [https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html](https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html)

Timezone: [https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/](https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/)

Account creation and systemd setup: [https://github.com/attestantio/ubuntu-server](https://github.com/attestantio/ubuntu-server)

eth2stats: [https://eth2stats.io/](https://eth2stats.io/)

blackbox_exporter: [https://github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter)

nod_exporter: [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)

Prometheus: [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)

Grafana: [https://grafana.com/docs/grafana/latest/installation/debian/](https://grafana.com/docs/grafana/latest/installation/debian/)

Dashboard: [https://github.com/metanull-operator/eth2-grafana](https://github.com/metanull-operator/eth2-grafana)

systemd: [https://www.freedesktop.org/software/systemd/man/systemd.unit.html](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)

geth: [https://geth.ethereum.org/docs/install-and-build/installing-geth](https://geth.ethereum.org/docs/install-and-build/installing-geth)

sshd: [https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh](https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh)

ufw: [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04)

ufw: [https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)
