
# Setup an Ethereum Mainnet Staking System on Ubuntu

This document contains instructions for configuring an Ethereum mainnet staking system on Ubuntu Linux 22.04 LTS. These instructions have been tested on an Intel NUC 11i5 with 2TB SSD and 32GB RAM, but the instructions for consensus and execution clients should apply equally to most AMD64 architecture systems. The optional dashboard presented in the monitoring section may require customization to query your specific network/hardware devices.

The following instructions were written to document my configuration for my own purposes. They are not intended to represent best practices and may not be applicable to your hardware, software, or network configuration. Please review these instructions carefully, compare them with instructions from other guides, and run on a testnet before risking your own funds.

If you installed Prysm/Geth using an earlier version of my staking instructions and you are ready to update your system for the merge, you can begin with my [Prysm/Geth Merge Updates for Older Installations](https://github.com/metanull-operator/eth2-ubuntu/blob/master/merge_updates.md) document.

Setup includes installation and configuration of the following services:

##### Consensus Layer

- Prysm Beacon Chain - Consensus layer client
- Prysm Validator - Validator client

##### Execution Layer

- Go Ethereum (Geth) - Execution layer client

##### Monitoring (Optional)

- Prometheus - Collects metrics
- Grafana - Visualizes metrics
- node_exporter - Provides metrics on the server
- blackbox_exporter - Provides metrics on ping times
- json_exporter - Provides metrics on ETH price

**IMPORTANT:** The configuration presented below will allow you to sync to the mainnet network and begin validating immediately in the pre-merge environment. This configuration is not 100% merge-ready. Execution and consensus clients must be updated to their final merge-ready versions prior to the merge. In addition, the `http-web3provider` configuration value must be updated to port 8551 after that port is opened by Geth without an overridden Terminal Total Difficulty (TTD).

## Prerequisities

### BIOS Update
If you have not updated the BIOS on your system, find and follow the manufacturer instructions for updating the BIOS. An updated BIOS may improve system performance or repair issues with your system. Instructions will vary dependent on the hardware you are using, but the following links should direct Intel NUC users to appropriate instructions.

- [2018 and earlier NUC BIOS Update Instructions](https://www.intel.com/content/www/us/en/support/articles/000005636/intel-nuc.html)
- [2019 and later NUC BIOS Update Instructions](https://www.intel.com/content/www/us/en/support/articles/000033291/intel-nuc.html)

### Configure Behavior After Power Failure
After a power failure, you may want your staking system to automatically restart and resume staking. Unfortunately, this is not the default behavior of many systems. Please check your system documentation to determine how to change this behavior in the system BIOS. For an Intel NUC, please check the following instructions.

- [Can Intel NUC Mini PCs turn on automatically as soon as a power source is connected?](https://www.intel.com/content/www/us/en/support/articles/000054773/intel-nuc.html)

### Software Update
After an initial install, it is a good idea to update everything to the latest versions.
```console
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
sudo apt-get autoremove
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

Check that your date/time are correct by running the date command.

```console
date
```

### Install Tools

Install a some commonly used tools needed to build, configure or run software.

- net-tools - Used to determine the network device for bandwidth reporting
- make - Used to build Geth
- curl - Downloading files
- git - Downloading software source code

```console
sudo apt-get install net-tools make curl git
```

## Shared Secret

After the merge, Consensus Layer and Execution Layer clients will authenticate with one another using a shared secret. We will create the secret in a protected directory accessible by a new `ethereum` group, to which both `geth` and `prysm-beacon` belong. This will allow those accounts to access the secret without allowing other system accounts access to it.

First we create the user accounts for Geth and the Prysm Beacon Chain.

```
sudo adduser --home /home/prysm-beacon --disabled-password --gecos 'Prysm Beacon Chain' prysm-beacon
sudo adduser --home /home/geth --disabled-password --gecos 'Go Ethereum Client' geth
```

Then we create a new user group called `ethereum`. We are going to add the `geth` and `prysm-beacon` user accounts as members of the `ethereum` group. 

```console
sudo groupadd ethereum
sudo usermod -a -G ethereum prysm-beacon
sudo usermod -a -G ethereum geth
```

Next we will create a folder in which we will put the secret.

```console
sudo mkdir -p /srv/ethereum/secrets
```

Then we make the the ethereum and secrets directories owned by the ethereum group with permissions to only be read by root or members of the ethereum group.

```console
sudo chgrp -R ethereum /srv/ethereum/ /srv/ethereum/secrets
sudo chmod 750 /srv/ethereum /srv/ethereum/secrets
```

Now create a secret.

```console
sudo openssl rand -hex -out /srv/ethereum/secrets/jwtsecret 32
```

Change the `jwtsecret` file so that it owned by the ethereum group with permissions to only be read by root or members of the ethereum group.

```console
sudo chown root:ethereum /srv/ethereum/secrets/jwtsecret
sudo chmod 640 /srv/ethereum/secrets/jwtsecret
```

## Geth (Go Ethereum)

An execution layer client will be required to stake Ethereum after [the merge](https://ethereum.org/en/upgrades/merge/). The following are instructions to install the Geth execution layer client.

### Install Geth

```console
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install ethereum
```

### Set Up systemd Service File
This sets up Geth to automatically run on start.

```console
sudo nano /etc/systemd/system/geth.service
```

Copy and paste the following text into the geth.service file.

```console
[Unit]
Description=Go Ethereum Client
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=geth
WorkingDirectory=/home/geth
ExecStart=/usr/bin/geth --http --http.addr 0.0.0.0 --authrpc.jwtsecret=/srv/ethereum/secrets/jwtsecret --authrpc.vhosts="*"

[Install]
WantedBy=multi-user.target
```

### Start Geth

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start geth
sudo systemctl enable geth
```

Check the logs to ensure that Geth is running properly.

```console
sudo journalctl -fu geth
```

## Prysm

### Create Validator Account

The Beacon Chain account was created when we created the secret. Now create the validator account.

```console
sudo adduser --home /home/prysm-validator --disabled-password --gecos 'Prysm Validator' prysm-validator
```

Create directories for binary files.

```console
sudo -u prysm-beacon mkdir /home/prysm-beacon/bin
sudo -u prysm-validator mkdir /home/prysm-validator/bin
```

### Install prysm.sh

prysm.sh is a shell script provided by Prysmatic Labs to automatically download the latest version of Prysm each time it starts. Using Prysm.sh makes upgrading as easy as restarting the beacon chain and validator clients, and reviewing the logs.

```console
sudo -u prysm-validator curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output /home/prysm-validator/bin/prysm.sh
sudo -u prysm-validator chmod +x /home/prysm-validator/bin/prysm.sh
sudo -u prysm-beacon curl https://raw.githubusercontent.com/prysmaticlabs/prysm/master/prysm.sh --output /home/prysm-beacon/bin/prysm.sh
sudo -u prysm-beacon chmod +x /home/prysm-beacon/bin/prysm.sh
```

### Set Up systemd Service File
This sets up prysm.sh to automatically run on start. This file is slightly different than the version under the Building Prysm section.

#### Beacon Chain
```console
sudo nano /etc/systemd/system/prysm-beacon.service
```

Copy and paste the following text into the beacon-chain.service file.

```
[Unit]
Description=Prysm Beacon Chain
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=prysm-beacon
ExecStart=/home/prysm-beacon/bin/prysm.sh beacon-chain --config-file /home/prysm-beacon/prysm-beacon.yaml

[Install]
WantedBy=multi-user.target
```

#### Validator

```console
sudo nano /etc/systemd/system/prysm-validator.service
```

Copy and paste the following text into the validator.service file.

```
[Unit]
Description=Prysm Validator
Wants=beacon-chain.service
After=beacon-chain.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=prysm-validator
ExecStart=/home/prysm-validator/bin/prysm.sh validator --config-file /home/prysm-validator/prysm-validator.yaml

[Install]
WantedBy=multi-user.target
```

### Create Prysm Configuration Files

#### prysm-beacon.yaml

```console
sudo -u prysm-beacon nano /home/prysm-beacon/prysm-beacon.yaml
```

Copy and paste the following text into the prysm-beacon.yaml configuration file.


```
datadir: "/home/prysm-beacon/prysm"
p2p-host-ip: "XXX.XXX.XXX.XXX"
http-web3provider: "http://YYY.YYY.YYY.YYY:8551"
monitoring-host: "0.0.0.0"
p2p-tcp-port: 13000
p2p-udp-port: 12000
accept-terms-of-use: true
suggested-fee-recipient: "0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
jwt-secret: "/srv/ethereum/secrets/jwtsecret"
```

 - If you have a dynamic IP address, remove the `p2p-host-ip` line.
   Otherwise, update `XXX.XXX.XXX.XXX` to your external IP address.
 - Update `YYY.YYY.YYY.YYY` to the IP address of your Eth1 node.
 - The `p2p-tcp-port` and `p2p-udp-port` lines are optional if you use the
   default values of 13000 and 12000, respectively.
 - Replace the `0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` value of the `suggested-fee-recipient` line with an Ethereum address at which you would like to receive tips/fees.

Change permissions of the file.

```console
sudo -u prysm-beacon chmod 640 /home/prysm-beacon/prysm-beacon.yaml
```

#### prysm-validator.yaml

```console
sudo -u prysm-validator nano /home/prysm-validator/prysm-validator.yaml
```

Copy and paste the following text into the prysm-validator.yaml configuration file.

```
monitoring-host: "0.0.0.0"
graffiti: "YOUR_GRAFFITI_HERE"
beacon-rpc-provider: "127.0.0.1:4000"
wallet-password-file: "/home/prysm-validator/.eth2validators/wallet-password.txt"
accept-terms-of-use: true
suggested-fee-recipient: "0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
```

- `graffiti` can be changed to whatever text you would prefer.
- Replace the `0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` value of the `suggested-fee-recipient` line with an Ethereum address at which you would like to receive tips/fees.

Change permissions of the file.

```console
sudo -u prysm-validator chmod 640 /home/prysm-validator/prysm-validator.yaml
```

### Make Validator Deposits and Install Keys

Follow the latest instructions at [launchpad.ethereum.org](https://launchpad.ethereum.org) or the correct launch pad for the network to which you will be connecting.

Look for the latest eth2.0-deposit-cli [here](https://github.com/ethereum/eth2.0-deposit-cli/releases/).

```console
cd
wget https://github.com/ethereum/staking-deposit-cli/releases/download/v2.2.0/staking_deposit-cli-9ab0b05-linux-amd64.tar.gz
tar xzvf staking_deposit-cli-9ab0b05-linux-amd64.tar.gz
mv staking_deposit-cli-9ab0b05-linux-amd64 staking_deposit-cli
cd staking_deposit-cli
./deposit new-mnemonic
```

**BACKUP YOUR MNEMONIC AND PASSWORD!**

The next step is to upload your deposit data file to the launchpad site. If you are using Ubuntu Server, you can either open up the deposit data file and copy it to a file on your desktop computer with the same name, or you can use scp or an equivalent tool to copy the deposit data to your desktop computer.

Follow the instructions by dragging and dropping the deposit file into the launchpad site. Then continue to follow the instructions until your deposit transaction is successful.

```console
sudo cp -pR $HOME/staking_deposit-cli/validator_keys /home/prysm-validator
sudo chown -R prysm-validator:prysm-validator /home/prysm-validator/validator_keys
sudo chmod 750 /home/prysm-validator/validator_keys
sudo -u prysm-validator /home/prysm-validator/bin/prysm.sh validator accounts import --keys-dir=/home/prysm-validator/validator_keys
```

Follow the prompts. The default wallet directory should be `/home/prysm-validator/.eth2validators/prysm-wallet-v2`. Use the same password used when you were prompted for a password while running `./deposit new-mnemonic`.

Create a password file and make it readbable only to the validator account.

```console
sudo -u prysm-validator touch /home/prysm-validator/.eth2validators/wallet-password.txt
sudo chmod 600 /home/prysm-validator/.eth2validators/wallet-password.txt
```

Edit the file and put the password you entered into the `deposit` tool into the `wallet-password.txt` file.

```console
sudo nano /home/prysm-validator/.eth2validators/wallet-password.txt
```

Enter the password into the first line and save the file.


### Start Beacon Chain and Validator

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start prysm-beacon prysm-validator
sudo systemctl enable prysm-beacon prysm-validator
```

Check the logs to ensure that both beacon chain and validator are running properly.

```console
sudo journalctl -fu prysm-beacon
sudo journalctl -fu prysm-validator
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

Find the URL to the latest amd64 version of Prometheus at https://prometheus.io/download/. In the commands below, replace any references to the version 2.23.0 to the latest version available.

```console
cd
wget https://github.com/prometheus/prometheus/releases/download/v2.37.0/prometheus-2.37.0.linux-amd64.tar.gz
tar xzvf prometheus-2.37.0.linux-amd64.tar.gz
cd prometheus-2.37.0.linux-amd64
sudo cp promtool /usr/local/bin/
sudo cp prometheus /usr/local/bin/
sudo chown root:root /usr/local/bin/promtool /usr/local/bin/prometheus
sudo chmod 755 /usr/local/bin/promtool /usr/local/bin/prometheus
cd
rm prometheus-2.37.0.linux-amd64.tar.gz
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
    - targets: ['127.0.0.1:9090']
  - job_name: 'beacon node'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:8080']
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:9100']
  - job_name: 'validator'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:8081']
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
  - job_name: json_exporter
    static_configs:
    - targets:
      - 127.0.0.1:7979
  - job_name: json
    metrics_path: /probe
    static_configs:
    - targets:
      - https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=usd
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: 127.0.0.1:7979
```

Change the ownership of the prometheus directory.

```console
sudo chown -R prometheus:prometheus /etc/prometheus
```

#### Data Directory
```console
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
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
    --storage.tsdb.retention.time=31d \
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
wget -qO- https://packages.grafana.com/gpg.key | sudo tee /etc/apt/trusted.gpg.d/grafana.asc
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana
```

#### Setup systemd

Start the service.

```console
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Login to grafana at http://XXX.XXX.XXX.XXX:3000/, replacing `XXX.XXX.XXX.XXX` with the IP address of your server. If you do not know the IP address, run `ifconfig`.

Default username `admin`. Default password `admin`. Grafana will ask you to set a new password.

##### **Optional Service Alias** 

Optionally edit the grafana-server.service file to add "grafana" as an alias to "grafana-server". I generally forget that the default name for this service is grafana-server.

```
sudo nano /lib/systemd/system/grafana-server.service
```

At the end of this file, in the `[Install]` section, add the following line:

```
Alias=grafana.service
```

#### Setup Prometheus Data Source
1. On the left-hand menu, hover over the gear menu and click on Data Sources.
2. Then click on the Add Data Source button.
3. Hover over the Prometheus card on screen, then click on the Select button.
4. Enter `http://127.0.0.1:9090/` into the URL field, then click Save & Test.

#### Install Grafana Dashboard
1. Hover over the plus symbol icon in the left-hand menu, then click on Import.
2. Copy and paste the dashboard at [https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source-beacon_node.json](https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source-beacon_node.json) into the "Import via panel json" text box on the screen.
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

node_exporter provides hardware and OS metrics to prometheus. Prometheus then provides those metrics to Grafana for visualization.

#### Create User Account
```console
sudo adduser --system node_exporter --group --no-create-home
```

#### Install node_exporter
```console
cd
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
tar xzvf node_exporter-1.3.1.linux-amd64.tar.gz
sudo cp node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm node_exporter-1.3.1.linux-amd64.tar.gz
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

### json_exporter

json_exporter provides the Ethereum price to prometheus. Prometheus then provides that information to Grafana for visualization.

#### Install go
Install go, if you haven't already.

```console
cd
wget https://go.dev/dl/go1.18.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.18.4.linux-amd64.tar.gz
sudo ln -s /usr/local/go/bin/go /usr/bin/go
rm go1.18.4.linux-amd64.tar.gz
```

#### Create User Account
```console
sudo adduser --system json_exporter --group --no-create-home
```

#### Install json_exporter
```console
cd
git clone https://github.com/prometheus-community/json_exporter.git
cd json_exporter
make build
sudo cp json_exporter /usr/local/bin/
sudo chown json_exporter:json_exporter /usr/local/bin/json_exporter
```

#### Configure json_exporter

```console
sudo mkdir /etc/json_exporter
sudo chown json_exporter:json_exporter /etc/json_exporter
```

```console
sudo nano /etc/json_exporter/json_exporter.yml
```

Copy and paste the following text into the json_exporter.yml file. 

```
metrics:
- name: ethusd
  path: "{.ethereum.usd}"
  help: Ethereum (ETH) price in USD
```

Change ownership of the configuration file to the json_exporter account.

```console
sudo chown json_exporter:json_exporter /etc/json_exporter/json_exporter.yml
```

#### Set Up System Service
```console
sudo nano /etc/systemd/system/json_exporter.service
```

Copy and paste the following text into the node_exporter.service file.

```
[Unit]
Description=JSON Exporter

[Service]
Type=simple
Restart=always
RestartSec=5
User=json_exporter
ExecStart=/usr/local/bin/json_exporter --config.file /etc/json_exporter/json_exporter.yml

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start json_exporter.service
sudo systemctl enable json_exporter.service
```


## Optional

### Install ntpd
For now, I prefer to use ntpd over the default systemd-timesyncd for syncing my system clock to an official time source.

From [this](https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04) tutorial on setting up time syncing on Ubuntu.

> Though timesyncd is fine for most purposes, some applications that
> are very sensitive to even the slightest perturbations in time may be
> better served by ntpd, as it uses more sophisticated techniques to
> constantly and gradually keep the system time on track.

```console
sudo apt-get install ntp
```
Update the NTP pool time server configuration to those that are geographically close to you. See [http://support.ntp.org/bin/view/Servers/NTPPoolServers](http://support.ntp.org/bin/view/Servers/NTPPoolServers) to find servers near you.

```console
sudo nano /etc/ntp.conf
```
Look for lines that begin with `server` and replace the current values with the values you identified from ntp.org.

Restart ntp. This will automatically shut down systemd-timesyncd, the default Ubuntu time syncing solution.

```console
sudo systemctl restart ntp
```

### blackbox_exporter
I have used blackbox_exporter to provide [ping](https://en.wikipedia.org/wiki/Ping_(networking_utility)) time data between my staking system and two DNS providers. Data is sent to Prometheus and on to Grafana. I have not found a practical use for this yet, though I have seen some interesting short-term shifts in ping times to Google. Therefore, blackbox_exporter is optional. 

The Grafana dashboard in these instructions includes a panel with a ping time graph. If you choose not to install blackbox_exporter, simply remove that panel from your Grafana dashboard. It will not show data.

#### Create User Account
```console
sudo adduser --system blackbox_exporter --group --no-create-home
```

#### Install blackbox_exporter
```console
cd
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.21.1/blackbox_exporter-0.21.1.linux-amd64.tar.gz
tar xvzf blackbox_exporter-0.21.1.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.21.1.linux-amd64/blackbox_exporter /usr/local/bin/
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chmod 755 /usr/local/bin/blackbox_exporter
```

Allow blackbox_exporter to ping servers.
```console
sudo setcap cap_net_raw+ep /usr/local/bin/blackbox_exporter
```

Remove the blackbox_exporter distribution file.

```console
rm blackbox_exporter-0.21.1.linux-amd64.tar.gz
```

#### Configure blackbox_exporter

```console
sudo mkdir /etc/blackbox_exporter
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
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

Change ownership of the configuration file to the blackbox_exporter account.

```console
sudo chown blackbox_exporter:blackbox_exporter /etc/blackbox_exporter/blackbox.yml
```

#### Set Up System Service
```
sudo nano /etc/systemd/system/blackbox_exporter.service
```

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

Restart Prometheus.

```
sudo systemctl restart prometheus
```

## Router Configuration

You may need to configure your router to forward the following ports to your staking system. See your router documentation for details.

Prysm Beacon Chain: 12000/udp
Prysm Beacon Chain: 13000/tcp
Geth: 30303/udp
Geth: 30303/tcp


## Security
### Firewall

The following commands set up the minimal firewall rules necessary to run both Prysm and Geth.

```console
# beacon chain
sudo ufw allow 12000/udp
sudo ufw allow 13000/tcp

# Geth
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp

# grafana
sudo ufw allow 3000/tcp
```

Run the following command to set up firewalls rules for SSH.

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

#Geth
#   - This only needs to be enabled if the consensus layer client (Prysm) is running on an different system.
sudo ufw allow 8545/tcp
sudo ufw allow 8551/tcp

# beacon-chain metrics
#   - This only needs to be enabled if you want to access beacon-chain stats directly.
sudo ufw allow 8080/tcp

# blackbox_exporter
#   - This only needs to be enabled if you want to access blackbox_exporter stats directly.
sudo ufw allow 9115/tcp

# prometheus
#   - This only needs to be enabled if you want to access prometheus directly.
sudo ufw allow 9090/tcp

# json_exporter
#   - This only needs to be enabled if you want to access blackbox_exporter stats directly.
sudo ufw allow 7979/tcp
```

### SSH (Optional)

Optionally you may update your SSH configuration to increase the security of your system.

```console
sudo nano /etc/ssh/sshd_config
```

Add the following lines, but replace <LOGIN> with your login. This will restrict SSH logins to your account. You are not logging in to ssh with root, right? If you are, you probably don't want to add the `AllowUsers` and `PermitRootLogin` lines below.

```console
AllowUsers <LOGIN>
PermitEmptyPasswords no
PermitRootLogin no
Protocol 2
```

#### Change SSH Port (Even More Optional)

I prefer to change the default SSH port to a non-standard port. Do not forget what you change this to. Find the following line, uncomment it line by removing the "#", and replace "22" with your preferred port.

```console
#Port 22
```

It would look something like the following, but replace <PORT> with your new port number.

```console
Port <PORT>
```

If you have the ufw firewall running, allow TCP to this new port. Replace <PORT> with your new port number.

```console
sudo ufw allow <PORT>/tcp
```

Restart your sshd service.

```console
sudo systemctl restart sshd
```

If you had a prior firewall rule for SSH port 22, list the ufw rules...

```console
sudo ufw status numbered
```

The results may look like the following.

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 13000/tcp                   ALLOW IN    Anywhere
[ 2] 12000/udp                   ALLOW IN    Anywhere
[ 3] 22/tcp                      ALLOW IN    Anywhere
[ 4] 30303/tcp                   ALLOW IN    Anywhere
[ 5] 30303/udp                   ALLOW IN    Anywhere
[ 6] 3000/tcp                    ALLOW IN    Anywhere
[ 7] 9090/tcp                    ALLOW IN    Anywhere
[ 8] 13000/tcp (v6)              ALLOW IN    Anywhere (v6)
[ 9] 12000/udp (v6)              ALLOW IN    Anywhere (v6)
[10] 22/tcp (v6)                 ALLOW IN    Anywhere (v6)
[11] 30303/tcp (v6)              ALLOW IN    Anywhere (v6)
[12] 30303/udp (v6)              ALLOW IN    Anywhere (v6)
[13] 3000/tcp (v6)               ALLOW IN    Anywhere (v6)
[14] 9090/tcp (v6)               ALLOW IN    Anywhere (v6)
```

Find the numbers of the two rules for the old SSH port 22. The rule numbers are shown in square brackets. In the results above, the port 22 rules are rule #3 and rule #10. **Important:** Delete the higher numbered rule first, otherwise that rule's number will change when you delete the lower number.

```console
sudo ufw delete 10
sudo ufw delete 3
```

## Important! Before the Merge

Please keep up to date on Geth and Prysm releases prior to the merge.

**Note:** The following process will take your Geth node offline for as long as it take the upgrade to complete. This may affect the performance of your validator for that period of time. You can switch to a backup provider while Geth is down, but those instructions are not covered here.

### Update Geth

To update Geth: 

```console
sudo apt-get update
sudo systemctl stop geth
sudo apt-get upgrade ethereum
sudo systemctl start geth
```

Monitor Geth logs for errors and warnings:

```console
sudo journalctl -fu geth
```

### Update Prysm

To update Prysm and view the logs:

```
sudo systemctl restart prysm-beacon ; sudo journalctl -fu prysm-beacon
sudo systemctl restart prysm-validator ; sudo journalctl -fu prysm-validator
```

### Reconfigure http-web3provider Port

**Note:** This section only applies if you used these instructions to set up a new system between July 19, 2022 and August 18, 2022. After August 18, 2022, this section is accounted for in the base instructions above.

Previously, Geth only exposed port 8551 if a Terminal Total Difficulty is configured for the network (mainnet). The Geth team subsequently removed the TTD constraint on that RPC service, and now the beacon chain can connect to Geth port 8551. Edit the Prysm Beacon configuration file to point `http-web3provider` at the new port 8551.

```console
sudo nano /home/prysm-beacon/prysm-beacon.yaml
```

Change port `8545` to port `8551` in the `http-web3provider` line.

```
http-web3provider: "http://127.0.0.1:8551/"
```

Restart the Prysm beacon.

```console
sudo systemctl restart prysm-beacon ; sudo journalctl -fu prysm-beacon
```

If you have completed all of these steps, you should be merge-ready!

## Common Commands

The following are some common commands you may want to use while running this setup.

### Service Statuses
To see the status of system services:

```console
sudo systemctl status prysm-beacon
sudo systemctl status prysm-validator
sudo systemctl status geth
sudo systemctl status prometheus
sudo systemctl status grafana-server
sudo systemctl status node_exporter
sudo systemctl status blackbox_exporter
sudo systemctl status json_exporter
```

Or, to see the status of all at once:
```console
sudo systemctl status prysm-beacon prysm-validator geth prometheus grafana-server node_exporter blackbox_exporter json_exporter
```
### Service Logs
To watch the logs in real time:

```console
sudo journalctl -fu prysm-beacon
sudo journalctl -fu prysm-validator
sudo journalctl -fu geth
sudo journalctl -fu prometheus
sudo journalctl -fu grafana-server
sudo journalctl -fu node_exporter
sudo journalctl -fu blackbox_exporter
sudo journalctl -fu json_exporter
```
### Restarting Services
To restart a service:

```console
sudo systemctl restart prysm-beacon
sudo systemctl restart prysm-validator
sudo systemctl restart geth
sudo systemctl restart prometheus
sudo systemctl restart grafana-server
sudo systemctl restart node_exporter
sudo systemctl restart blackbox_exporter
sudo systemctl restart json_exporter
```

### Stopping Services
Stopping a service is separate from disabling a service. Stopping a service stops the current execution of the server, but does not prohibit the service from starting again after a system reboot. If you intend for the service to stop running and to not restart after a reboot, you will want to stop and disable a service.

To stop a service:

```console
sudo systemctl stop prysm-beacon
sudo systemctl stop prysm-validator
sudo systemctl stop geth
sudo systemctl stop prometheus
sudo systemctl stop grafana-server
sudo systemctl stop node_exporter
sudo systemctl stop blackbox_exporter
sudo systemctl stop json_exporter
```

**Important:** If you intend to stop the beacon chain and validator in order to run these services on a different system, stop the services using the instructions in this section, and disable these services following the instructions in the next section. You will be at risk of losing funds through slashing if you accidentally validate the same keys on two different systems, and failing to disable the services may result in your beacon chain and validator running again after a system reboot.

### Disabling Services
To disable a service so that it no longer starts automatically after a reboot:

```console
sudo systemctl disable prysm-beacon
sudo systemctl disable prysm-validator
sudo systemctl disable geth
sudo systemctl disable prometheus
sudo systemctl disable grafana-server
sudo systemctl disable node_exporter
sudo systemctl disable blackbox_exporter
sudo systemctl disable json_exporter
```

### Enabling Services
To enable a service to automatically run when the server boots:

```console
sudo systemctl enable prysm-beacon
sudo systemctl enable prysm-validator
sudo systemctl enable geth
sudo systemctl enable prometheus
sudo systemctl enable grafana-server
sudo systemctl enable node_exporter
sudo systemctl enable blackbox_exporter
sudo systemctl enable json_exporter
```
### Starting Services
Re-enabling a service will not necessarily start the service as well. To start a service that is stopped:

```console
sudo systemctl start prysm-beacon
sudo systemctl start prysm-validator
sudo systemctl start geth
sudo systemctl start prometheus
sudo systemctl start grafana-server
sudo systemctl start node_exporter
sudo systemctl start blackbox_exporter
sudo systemctl start json_exporter
```

### Upgrading Prysm
Upgrading the Prysm beacon chain and validator clients is as easy as restarting the service when running the prysm.sh script as we are in these instructions. To upgrade to the latest release, simple restart the services and check the log files for any problems. If any important command line flags have changed, a notice will appear in the logs. Even better, read the release notes in advance of an upgrade.

The following commands will restart one client and display the logs.

```console
sudo systemctl restart prysm-beacon ; sudo journalctl -fu prysm-beacon
sudo systemctl restart prysm-validator ; sudo journalctl -fu prysm-validator
```

### Changing systemd Service Files
If you edit any of the systemd service files in `/etc/systemd/system` or another location, run the following command prior to restarting the affected service:

```console
sudo systemctl daemon-reload
```
Then restart the affected service:
```console
sudo systemctl restart SERVICE_NAME
```

- Replace SERVICE_NAME with the name of the service for which the service file was updated. For example, `sudo systemctl restart prysm-beacon`.

### Updating Prysm Options
To update the configuration options of the beacon chain or validator, edit the Prysm configuration file located in the home directories for the services.

```console
sudo nano /home/prysm-validator/prysm-validator.yaml
sudo nano /home/prysm-beacon/prysm-beacon.yaml
```

Then restart the services and check the logs:

```console
sudo systemctl restart prysm-validator ; sudo journalctl -fu prysm-validator
sudo systemctl restart prysm-beacon ; sudo journalctl -fu prysm-beacon
```


## Sources/Inspiration
Prysm: [https://docs.prylabs.network/docs/getting-started/](https://docs.prylabs.network/docs/getting-started/)

Go: [https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html](https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html)

Timezone: [https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/](https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/)

Account creation and systemd setup: [https://github.com/attestantio/ubuntu-server](https://github.com/attestantio/ubuntu-server)

blackbox_exporter: [https://github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter)

Geth: [https://github.com/ethereum/go-ethereum](https://github.com/ethereum/go-ethereum)

node_exporter: [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)

ntpd: [https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04)

Prometheus: [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)

Grafana: [https://grafana.com/docs/grafana/latest/installation/debian/](https://grafana.com/docs/grafana/latest/installation/debian/)

Dashboard: [https://github.com/metanull-operator/eth2-grafana](https://github.com/metanull-operator/eth2-grafana)

systemd: [https://www.freedesktop.org/software/systemd/man/systemd.unit.html](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)

Geth: [https://geth.ethereum.org/docs/install-and-build/installing-geth](https://geth.ethereum.org/docs/install-and-build/installing-geth)

sshd: [https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh](https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh)

ufw: [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04)

ufw: [https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)
