# Setup an Eth2 Validator System on Ubuntu
These instructions represent my current process for setting up an Eth2 staking system on Ubuntu 20.04 LTS on an Intel NUC 10i5FNK with 512GB SSD and 16GB RAM. These instructions are primarily for my own purposes, so that I can recreate my environment if I need to. They are not intended to represent best practices and may not be applicable to your hardware, software, or network configuration. There are many other good sources for instructions on setting up these services, and those may be more generally written and applicable.

Setup includes installation and configuration of the following services, including setting up systemd to automatically run services, where applicable:

- Prysm Beacon Chain
- Prysm Validator
- eth2stats
- Prometheus
- Grafana
- node_exporter
- blackbox_exporter
- ethereal (optional)
- ethdo (optional)

ethdo and ethereal are optional, depending on your preference for key management. Prysm will be updating their key management tool set soon, and the updated tools may be preferred. Instructions on creating and managing keys are not included here.

Steps to install and configure all software have been copied from or inspired by a number of sources, which are cited where available. Discord discussions may have provided additional details or ideas. In addition, though I have never been a professional Linux administrator, I have 25 years experience running Linux servers for a variety of public and private hobby projects, which may have informed some of my decisions, for better or worse.

This process assumes starting from first login on a clean Ubuntu 20.04 LTS installation, and were last tested on July 4, 2020.

Though this is mostly documented for my own purposes, I am open to suggestions for improvement.

## Initial System Setup
I prefer to change the default SSH port to a non-standard port. This step is optional. Do not forget what you have changed this to.
```console
sudo nano /etc/ssh/sshd_config
```

Find the following line, uncomment it line by removing the "#", and replace "22" with your preferred port.

```
#Port 22
```

```console
sudo reboot
```

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

### make
```console
sudo apt-get install make
```

### go
```console
sudo apt-get install golang-1.14-go

# Create a symlink from /usr/bin/go to the new go installation
sudo ln -s /usr/lib/go-1.14/bin/go /usr/bin/go
```

### bazelisk
```console
cd
go get github.com/bazelbuild/bazelisk

echo "export PATH=\$PATH:\$(go env GOPATH)/bin" >> ~/.profile
source ~/.profile
```


## Prysm
These instructions build Prysm from source using bazelisk. Prysmatic Labs recommends using their prysm.sh script. I have not included instructions for that here.

### Install Prysm
#### Create User Accounts
```console
sudo adduser --home /home/beacon --disabled-password --gecos 'Ethereum 2 Beacon Chain' beacon
sudo adduser --home /home/validator --disabled-password --gecos 'Ethereum 2 Validator' validator
sudo mkdir -p /home/beacon/bin
sudo mkdir -p /home/validator/bin
sudo chown -R beacon:beacon /home/beacon/bin
sudo chown -R validator:validator /home/validator/bin
```

#### Install Additional Libraries
```console
sudo apt-get install libssl-dev
sudo apt-get install libgmp-dev
sudo apt-get install libtinfo5
```

#### Build Prysm
```console
cd
git clone https://github.com/prysmaticlabs/prysm
cd prysm
bazelisk build //beacon-chain:beacon-chain --config=release
bazelisk build //validator:validator --config=release
bazelisk build //slasher:slasher --config=release
```

Note: These instructions do not include instructions on configuring or running a slasher, but I included the command to build the slasher anyhow.

#### Copy Binaries to User Accounts
```console
sudo cp bazel-bin/beacon-chain/linux_amd64_stripped/beacon-chain /home/beacon/bin/
sudo chown -R beacon.beacon /home/beacon/bin
sudo cp bazel-bin/validator/linux_amd64_stripped/validator /home/validator/bin
sudo chown -R validator:validator /home/validator/bin
```

#### Validator Key Directory
For my current iteration of this setup, I am using ethdo for key management and ethereal for deposits, but I do not go into detail about setting those up in these instructions. Below are steps I take to create a directory for my keys in the validator account.

This part of my set up may change when Prysmatic Labs introduces new key management functionality in Prysm.

```console
sudo mkdir -p /home/validator/keys/wallets
sudo chown validator.validator /home/validator/keys
sudo chmod 700 /home/validator/keys
```

#### Installing Validator Keys
This step should come later in the process, after keys are generated, which is not covered in these instructions. However, when you are ready to install keys, first add a keymanager.json file.

```console
sudo nano /home/validator/keys/keymanager.json
```

The following is the skeleton of a keymanager.json file, which includes the specification of the wallet directory. Copy this into keymanager.json. Note that this is not sufficient for the validator to run. You will have to complete the details for at least a single account for the validator correctly start.

```
{
        "location": "/home/validator/keys/wallets/",
        "accounts": [
        ],
        "passphrases": [
        ]
}
```

Make keymanager.json read-only to other groups/users.

```console
sudo chmod 600 /home/validator/keys/keymanager.json
```

Then copy all keys files to the wallets directory (command not shown), change the ownership of all key/wallet files to the validator account, and change permissions so that these files are only readable to the validator account.

```console
sudo chown -R validator.validator /home/validator/keys
sudo chmod 600 /home/validator/keys/wallets/<WALLET_FILENAME>
```

After adding keys, restart the validator:

```console
sudo systemctl restart validator
```

#### Set Up systemd Service
##### Beacon Chain
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
ExecStart=/home/beacon/bin/beacon-chain --datadir=/home/beacon/prysm --p2p-host-ip=XXX.XXX.XXX.XXX --http-web3provider http://YYY.YYY.YYY.YYY:8545 --monitoring-host 0.0.0.0

[Install]
WantedBy=multi-user.target
```
Update `XXX.XXX.XXX.XXX` to your IP address.
Update `YYY.YYY.YYY.YYY` to the IP address of your Eth1 node, or remove the `--http-web3provider` flag entirely to use the default Eth1 node.

Enable and start the beacon chain service.
```console
sudo systemctl daemon-reload
sudo systemctl enable beacon-chain.service
sudo systemctl start beacon-chain.service
```

##### Validator

```console
sudo nano /etc/systemd/system/validator.service
```

Copy and paste the following text into the validator.service file.

```
[Unit]
Description=Ethereum 2 Validator
After=beacon-chain.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=validator
ExecStart=/home/validator/bin/validator --keymanager=wallet --keymanageropts="/home/validator/keys/keymanager.json" --monitoring-host "0.0.0.0"

[Install]
WantedBy=multi-user.target
```

Start and enable the validator service. Note that the validator will fail until a valid keymanager.json file is in place.

```console
sudo systemctl daemon-reload
sudo systemctl enable validator.service
sudo systemctl start validator.service
```

## eth2stats

### Create User Account
```console
sudo adduser --system eth2stats --group --no-create-home
```

### Install eth2stats
```console
cd
git clone https://github.com/alethio/eth2stats-client
cd ~/eth2stats-client
make build
sudo cp eth2stats-client /usr/local/bin
sudo chown root.root /usr/local/bin/eth2stats-client
sudo chmod 755 /usr/local/bin/eth2stats-client
```

### Create Data Directory
```console
sudo mkdir /var/lib/eth2stats
sudo chown eth2stats.eth2stats /var/lib/eth2stats
sudo chmod 755 /var/lib/eth2stats
```

### Set Up System Service

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
ExecStart=/usr/local/bin/eth2stats-client run --v --eth2stats.node-name="NODE_NAME" --eth2stats.addr="grpc.onyx.eth2stats.io:443" --beacon.metrics-addr="http://localhost:8080/metrics" --eth2stats.tls=true --beacon.type="prysm" --beacon.addr="localhost:4000"

[Install]
WantedBy=multi-user.target
```

Replace `NODE_NAME` with the name you would like to appear on eth2stats.io.

These instructions were written during the Prysm Onyx testnet. The command-line flag `--eth2stats.addr` may need to be updated to a new address for Altona or later testnets.

```console
sudo systemctl daemon-reload
sudo systemctl enable eth2stats.service
sudo systemctl start eth2stats.service
```

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
sudo chown root.root /usr/local/bin/promtool
sudo chmod 755 /usr/local/bin/promtool
sudo cp prometheus /usr/local/bin/
sudo chown root.root /usr/local/bin/prometheus
sudo chmod 755 /usr/local/bin/prometheus
cd
rm prometheus-2.19.2.linux-amd64.tar.gz
```

### Configure Prometheus
```console
sudo mkdir -p /etc/prometheus/console_libraries
sudo mkdir -p /etc/prometheus/consoles
sudo mkdir -p /etc/prometheus/files_sd
sudo mkdir -p /etc/prometheus/rules
sudo mkdir -p /etc/prometheus/rules.d
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

## node_exporter
### Create User Account
`sudo adduser --system node_exporter --group --no-create-home`

### Install node_exporter
```console
cd
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.0/node_exporter-1.0.0.linux-amd64.tar.gz
tar xzvf node_exporter-1.0.0.linux-amd64.tar.gz
sudo cp node_exporter-1.0.0.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm node_exporter-1.0.0.linux-amd64.tar.gz
```

### Data Directory
```console
sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo chown -R node_exporter.node_exporter /var/lib/node_exporter
```

### Set Up System Service
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
ExecStart=/usr/local/bin/node_exporter --collector.textfile.directory /var/lib/node_exporter/textfile_collector

[Install]
WantedBy=multi-user.target
```

```console
sudo systemctl daemon-reload
sudo systemctl start node_exporter.service
sudo systemctl enable node_exporter.service
```

## blackbox_exporter
### Create User Account
```console
sudo adduser --system blackbox_exporter --group --no-create-home
```

### Install blackbox_exporter
```console
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.16.0/blackbox_exporter-0.16.0.linux-amd64.tar.gz
tar xvzf blackbox_exporter-0.16.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.16.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo chown blackbox_exporter.blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chmod 755 /usr/local/bin/blackbox_exporter
```

Allow blackbox_exporter to ping servers.
```console
setcap cap_net_raw+ep /usr/local/bin/blackbox_exporter
```

```console
rm blackbox_exporter-0.16.0.linux-amd64.tar.gz
```

### Configure blackbox_exporter

```console
sudo mkdir -p /etc/blackbox_exporter
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


### Set Up System Service
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

## Future Updates

There are a few areas where I may expand on my system configuration or instructions, but I have not pursued these yet.

 - Firewall
   - My router has a firewall already, and my system is not running any services that don't come out-of-the-box with Ubuntu, and that are not already listed here.
 - SSH Key-Based Login
   - This seems to be a good security move, but it also seems to be the perfect way to get me locked out of my own system. I have never set this up before, but may look into it.
 - Instructions for Key Management
   - My goal with these instructions is to make my system setup something I can easily duplicate. Key creation is something I do not expect to duplicate often, and it was never my intention to create comprehensive instructions on running a validator. In addition, newer key management tools from Prysmatic Labs are on the way. 

## Sources/Inspiration
Prysm: [https://docs.prylabs.network/docs/getting-started/](https://docs.prylabs.network/docs/getting-started/)

Bazelisk: [https://github.com/bazelbuild/bazelisk](https://github.com/bazelbuild/bazelisk)

Go: [https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html](https://ubuntu.pkgs.org/20.04/ubuntu-main-arm64/golang-1.14-go_1.14.2-1ubuntu1_arm64.deb.html)

Timezone: [https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/](https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-20-04/)

Account creation and systemd setup: [https://github.com/attestantio/ubuntu-server](https://github.com/attestantio/ubuntu-server)

eth2stats: [https://eth2stats.io/](https://eth2stats.io/)

blackbox_exporter: [https://github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter)

nod_exporter: [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)

ethdo, though instructions are not presented here: [https://github.com/wealdtech/ethdo](https://github.com/wealdtech/ethdo)

ethereal, though instructions are not presented here: [https://github.com/wealdtech/ethereal](https://github.com/wealdtech/ethereal)

Prometheus: [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)

Grafana: [https://grafana.com/docs/grafana/latest/installation/debian/](https://grafana.com/docs/grafana/latest/installation/debian/)

Dashboard: [https://github.com/metanull-operator/eth2-grafana](https://github.com/metanull-operator/eth2-grafana)
