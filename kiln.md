# Setup a Merge Kiln Testnet Validator System on Ubuntu

This document contains instructions for setting up an Ethereum Kiln merge testnet staking system using Prysm and geth.

These instructions have been adapted from the instructions available at https://hackmd.io/dFzKxB3ISWO8juUqPpJFfw for the Kintsugi testnet, and have been updated based on Kiln-specific configurations. I have added enough details so that I can start from a base Ubuntu Server installation and get all the way through setting up monitoring. The monitoring portion is optional, is not fully functional under Kiln and does not presently include details for monitoring geth. These instructions are also not intended for a production system. I have kept the folder structure in line with the instructions in the link above, which may provide an easier upgrade to future merge testnets. In a final production version, the executables and data would not be in my user directories.

These instructions were developed to configure an Ethereum Kiln merge testnet staking system using Ubuntu 20.04 LTS Server on an Intel NUC 10i5FNK with 2TB SSD and 32GB RAM. I have added a few additional packages that I believe should cover Ubuntu Desktop as well.

Setup includes installation and configuration of the following services, including setting up systemd to automatically run services, where applicable:

- Prysm Beacon Chain
- Prysm Validator
- geth 
- Prometheus
- Grafana
- node_exporter
- blackbox_exporter
- json_exporter

Steps to install and configure all software have been copied from or inspired by a number of additional sources, which are cited at the end of this file.

This process assumes starting from first login on a clean Ubuntu 20.04 LTS installation, and were last tested on March 12, 2022.

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

### Install Prerequisites

- net-tools - Used to determine the network device for bandwidth reporting.
- make - Used to build geth
- gcc - Compiling software
- g++ - Compiling software
- curl - Ubuntu Desktop may not include this by default
- git - Ubuntu Desktop may not include this by default
- jq - Used for JSON parsing in deposit script

```console
sudo apt-get install net-tools make gcc g++ curl git jq
```

### Install golang

Install golang v1.17 and create a link to the executable at /usr/bin/go.

```console
cd
wget https://go.dev/dl/go1.17.8.linux-amd64.tar.gz
cd /usr/local/bin
sudo tar xzvf ~/go1.17.8.linux-amd64.tar.gz
echo "export PATH=\$PATH:/usr/local/bin/go" >> ~/.profile
source ~/.profile
sudo ln -s /usr/local/bin/go/bin/go /usr/bin/go
```

### Install bazelisk

```
cd
go install github.com/bazelbuild/bazelisk@latest
echo "export PATH=\$PATH:\$(go env GOPATH)/bin" >> ~/.profile
source ~/.profile
```

### Download Kiln Package

```
cd
git clone https://github.com/eth-clients/merge-testnets.git
```

## geth

A geth full node is required to provide access to deposits made to the deposit contract. It could take some time for geth to sync, so start this process immediately.

### Install geth

```console
cd ~/merge-testnets/kiln/
git clone -b merge-kiln-v2 https://github.com/MariusVanDerWijden/go-ethereum.git
cd go-ethereum 
make geth
```

### Set Up systemd Service File

This sets up geth to automatically run on start.

```console
sudo nano /etc/systemd/system/geth.service
```

Copy and paste the following text into the geth.service file.

**Replace <USERNAME> with your system username/login.**

```
[Unit]
Description=Go Ethereum Client
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=<USERNAME>
WorkingDirectory=/home/<USERNAME>/merge-testnets/kiln/
ExecStart=/home/<USERNAME>/merge-testnets/kiln/go-ethereum/build/bin/geth --datadir /home/<USERNAME>/merge-testnets/kiln/datadir-prysm --networkid=1337802 --http --http.api engine,net,eth --ws --ws.api net,eth,engine --bootnodes enode://c354db99124f0faf677ff0e75c3cbbd568b2febc186af664e0c51ac435609badedc67a18a63adb64dacc1780a28dcefebfc29b83fd1a3f4aa3c0eb161364cf94@164.92.130.5:30303

[Install]
WantedBy=multi-user.target
```

### Initialize Genesis State

```
~/merge-testnets/kiln/go-ethereum/build/bin/geth --datadir ~/merge-testnets/kiln/datadir-prysm init ~/merge-testnets/kiln/genesis.json
```

### Start geth

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start geth
sudo systemctl enable geth
```

You can check the geth logs with the following command.

```console
sudo journalctl -u geth -f
```

## Prysm

### Build Prysm

```console
cd ~/merge-testnets/kiln
git clone -b kiln https://github.com/prysmaticlabs/prysm.git
cd prysm
bazelisk build //beacon-chain:beacon-chain
bazelisk build //validator:validator
```

### Set Up systemd Service File

This sets up prysm.sh to automatically run on start. This file is slightly different than the version under the Building Prysm section.

#### Beacon Chain

```console
sudo nano /etc/systemd/system/beacon-chain.service
```

Copy and paste the following text into the beacon-chain.service file. 

**Replace <USERNAME> with your system username/login.**

```
[Unit]
Description=Prysm Ethereum Beacon Chain
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=<USERNAME>
WorkingDirectory=/home/<USERNAME>/merge-testnets/kiln/prysm
ExecStart=/home/<USERNAME>/go/bin/bazelisk run //beacon-chain -- --genesis-state /home/<USERNAME>/merge-testnets/kiln/genesis.ssz --datadir /home/<USERNAME>/merge-testnets/kiln/datadir-prysm --http-web3provider=http://127.0.0.1:8545 --execution-provider=http://127.0.0.1:8545 --chain-config-file=/home/<USERNAME>/merge-testnets/kiln/config.yaml --accept-terms-of-use

[Install]
WantedBy=multi-user.target
Alias=beacon
```

#### Validator

```console
sudo nano /etc/systemd/system/validator.service
```

Copy and paste the following text into the validator.service file.

**Replace <USERNAME> with your system username/login.**

```
[Unit]
Description=Prysm Ethereum Validator
Wants=beacon-chain.service
After=beacon-chain.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=5
User=<USERNAME>
WorkingDirectory=/home/<USERNAME>/merge-testnets/kiln/prysm
ExecStart=/home/<USERNAME>/go/bin/bazelisk run validator -- --accept-terms-of-use --wallet-password-file /home/<USERNAME>/merge-testnets/kiln/datadir-prysm/password.txt --wallet-dir /home/<USERNAME>/merge-testnets/kiln/datadir-prysm/prysm-wallet-v2

[Install]
WantedBy=multi-user.target

```

### Setup MetaMask

Connect MetaMask to the Kiln network by going to https://kiln.themerge.dev/ and clicking on the "Add to MetaMask" button. If necessary, create a new account in MetaMask into which the deposit funds will be stored. Request Kiln ETH from the [Kiln faucet](https://faucet.kiln.themerge.dev/).

### Make Validator Deposits and Install Keys

#### Install eth2-val-tools

```
cd
git clone https://github.com/protolambda/eth2-val-tools
cd eth2-val-tools
go install .
```

#### Install ethereal

```console
cd
go install github.com/wealdtech/ethereal/v2@latest
```

#### Generate Mnemonics

Create the secrets.env file.

```
touch ~/merge-testnets/kiln/secrets.env
chmod 600 ~/merge-testnets/kiln/secrets.env
nano ~/merge-testnets/kiln/secrets.env
```

Copy and paste the following text into the secrets.env file.

```
# sets the deposit amount to use
DEPOSIT_AMOUNT=32000000000
# sets the genesis fork version of the testnet
FORK_VERSION="0x60000069"
# sets the mnemonic to derive the keys from
VALIDATORS_MNEMONIC=""
# sets the mnemonic for withdrawal credentials
WITHDRAWALS_MNEMONIC=""
# temporary location to store the deposit data
DEPOSIT_DATAS_FILE_LOCATION="/tmp/deposit_data.txt"
# sets the deposit contract address
DEPOSIT_CONTRACT_ADDRESS="0x4242424242424242424242424242424242424242"
# sets the eth1 address from which the transaction will be made
ETH1_FROM_ADDR=""
# sets the eth1 private key used to sign the transaction
ETH1_FROM_PRIV=""
# forces the deposit since the deposit contract will not be recognized by the tool
FORCE_DEPOSIT=true
# sets an RPC endpoint to submit the transaction to
ETH1_RPC=https://rpc.kiln.themerge.dev
```

Generate passphrases to secure your validator keys and your withdrawal keys. The command to generate a pass phrase will be run once for the validator keys and once for the withdrawal keys. Save both sets of keys in the secrets.env file, and save copies in a secure offline location as well.

```console
eth2-val-tools mnemonic
```

Insert the generated validator pass phrase in between the quotes on the VALIDATORS_MNEMONIC line. Insert the generated withdrawal pass phrase in between the quotes on the WITHDRAWALS_MNEMONIC line. Save and close the file.

#### Create Prysm Wallet Password

Create a unique password and store it in the password.txt file. Have this password available when generating the Prysm keys.

```console
touch ~/merge-testnets/kiln/datadir-prysm/password.txt
chmod 600 ~/merge-testnets/kiln/datadir-prysm/password.txt
nano ~/merge-testnets/kiln/datadir-prysm/password.txt
```

#### Add MetaMask Account Details

Add MetaMask account details to the secrets.env file so that the deposit script can transfer funds.

In MetaMask, go to the account with the Kiln ETH and copy the account address into the ETH1_FROM_ADDR line in the secrets.env file. The address must begin with "0x".

Copy the private key for this account into the ETH1_FROM_PRIV line in the secrets.env file. The private key can be found in MetaMask by going to the account, clicking on the three dots button, clicking on Account Details, and then clicking on Export Private Key. The private key must begin with "0x". If the private key does not begin with "0x", simply add "0x" to the beginning of the private key.

#### Deposit Kiln ETH

```
cd ~/merge-testnets/kiln
nano devnet_deposits.sh
```

Insert the following text into the devnet_deposits.sh file. Then save and close the file.

```bash
#!/bin/bash

echo "USE AT YOUR OWN RISK"
read -p "Are you sure you've doubl e checked the values and want to make this deposit? " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]
then
    [[ "$0" = "$BASH_SOURCE" ]] && exit 1 || return 1
fi

source secrets.env

if [[ -z "${ETH1_FROM_ADDR}" ]]; then
  echo "need ETH1_FROM_ADDR environment var"
  exit 1 || return 1
fi
if [[ -z "${ETH1_FROM_PRIV}" ]]; then
  echo "need ETH1_FROM_PRIV environment var"
  exit 1 || return 1
fi


eth2-val-tools deposit-data \
  --source-min=0 \
  --source-max=1 \
  --amount=$DEPOSIT_AMOUNT \
  --fork-version=$FORK_VERSION \
  --withdrawals-mnemonic="$WITHDRAWALS_MNEMONIC" \
  --validators-mnemonic="$VALIDATORS_MNEMONIC" > $DEPOSIT_DATAS_FILE_LOCATION


# Iterate through lines, each is a json of the deposit data and some metadata
while read x; do
   account_name="$(echo "$x" | jq '.account')"
   pubkey="$(echo "$x" | jq '.pubkey')"
   echo "Sending deposit for validator $account_name $pubkey"
   ethereal beacon deposit \
      --allow-unknown-contract=$FORCE_DEPOSIT \
      --address="$DEPOSIT_CONTRACT_ADDRESS" \
      --connection=$ETH1_RPC \
      --data="$x" \
      --value="$DEPOSIT_ACTUAL_VALUE" \
      --from="$ETH1_FROM_ADDR" \
      --privatekey="$ETH1_FROM_PRIV"
   echo "Sent deposit for validator $account_name $pubkey"
   sleep 3
done < "$DEPOSIT_DATAS_FILE_LOCATION"
```

Reset permissions on the script so that it is executable.

```console
chmod +x devnet_deposits.sh
```

Assuming you have sufficient Kiln ETH in your MetaMask account, run the deposit script.

```console
./devnet_deposits.sh
```

### Generate Prysm Keys

Use the validator mnemonic/pass phrase to generate Prysm keys.

- When asked for your pass phrase, enter the validator pass phrase generated earlier and entered into the secrets.env file.
- When asked for the director into which the keys should be stored, enter the following `~/merge-testnets/kiln/datadir-prysm/prysm-wallet-v2`.
- When asked for the wallet password, enter the same password you added to the file at `~/merge-testnets/kiln/datadir-prysm/password.txt`.

```console
cd ~/merge-testnets/kiln/prysm/
bazelisk run validator wallet recover
```

### Start Beacon Chain and Validator

Start and enable the validator service.

```console
sudo systemctl daemon-reload
sudo systemctl start beacon-chain validator
sudo systemctl enable beacon-chain validator
```

You can check the geth logs with the following command.

```console
sudo journalctl -u beacon-chain -f
sudo journalctl -u validator -f
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
wget https://github.com/prometheus/prometheus/releases/download/v2.32.1/prometheus-2.32.1.linux-amd64.tar.gz
tar xzvf prometheus-2.32.1.linux-amd64.tar.gz
cd prometheus-2.32.1.linux-amd64
sudo cp promtool /usr/local/bin/
sudo cp prometheus /usr/local/bin/
sudo chown root:root /usr/local/bin/promtool /usr/local/bin/prometheus
sudo chmod 755 /usr/local/bin/promtool /usr/local/bin/prometheus
cd
rm prometheus-2.32.1.linux-amd64.tar.gz
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
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana
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
4. Enter `http://127.0.0.1:9090/` into the URL field, then click Save & Test.

#### Install Grafana Dashboard

1. Hover over the plus symbol icon in the left-hand menu, then click on Import.
2. Copy and paste the dashboard at [https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source-beacon_node.json](https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source-beacon_node.json) into the "Import via panel json" text box on the screen. If you used an older version of these instructions, where the Prometheus configuration file uses the beacon node job name of "beacon" instead of "beacon node", please (use this dashboard)[https://raw.githubusercontent.com/metanull-operator/eth2-grafana/master/eth2-grafana-dashboard-single-source.json] instead for backwards compatibility.
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
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.19.0/blackbox_exporter-0.19.0.linux-amd64.tar.gz
tar xvzf blackbox_exporter-0.19.0.linux-amd64.tar.gz
sudo cp blackbox_exporter-0.19.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo chown blackbox_exporter:blackbox_exporter /usr/local/bin/blackbox_exporter
sudo chmod 755 /usr/local/bin/blackbox_exporter
```

Allow blackbox_exporter to ping servers.

```console
sudo setcap cap_net_raw+ep /usr/local/bin/blackbox_exporter
```

```console
rm blackbox_exporter-0.19.0.linux-amd64.tar.gz
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

# json_exporter
#   - This only needs to be enabled if you want to access blackbox_exporter stats directly.
sudo ufw allow 7979/tcp
```

## Common Commands

The following are some common commands you may want to use while running this setup.

### Service Statuses

To see the status of system services:

```console
sudo systemctl status beacon-chain
sudo systemctl status validator
sudo systemctl status geth
sudo systemctl status prometheus
sudo systemctl status grafana-server
sudo systemctl status node_exporter
sudo systemctl status blackbox_exporter
sudo systemctl status json_exporter
```

Or, to see the status of all at once:

```console
sudo systemctl status beacon-chain validator geth prometheus grafana-server node_exporter blackbox_exporter json_exporter
```

### Service Logs

To watch the logs in real time:

```console
sudo journalctl -u beacon-chain -f
sudo journalctl -u validator -f
sudo journalctl -u geth -f
sudo journalctl -u prometheus -f
sudo journalctl -u grafana-server -f
sudo journalctl -u node_exporter -f
sudo journalctl -u blackbox_exporter -f
sudo journalctl -u json_exporter -f
```

### Restarting Services

To restart a service:

```console
sudo systemctl restart beacon-chain
sudo systemctl restart validator
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
sudo systemctl stop beacon-chain
sudo systemctl stop validator
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
sudo systemctl disable beacon-chain
sudo systemctl disable validator
sudo systemctl disable geth
sudo systemctl disable prometheus
sudo systemctl disable grafana-server
sudo systemctl disable node_exporter
sudo systemctl disable blackbox_exporter
sudo systemctl disable json_exporter
```

### Enabling Services

To re-enable a service that has been disabled:

```console
sudo systemctl enable beacon-chain
sudo systemctl enable validator
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
sudo systemctl start beacon-chain
sudo systemctl start validator
sudo systemctl start geth
sudo systemctl start prometheus
sudo systemctl start grafana-server
sudo systemctl start node_exporter
sudo systemctl start blackbox_exporter
sudo systemctl start json_exporter
```

### Upgrading Prysm

Upgrading the Prysm beacon chain and validator clients is as easy as restarting the service when running the prysm.sh script as we are in these instructions. To upgrade to the latest release, simple restart the services. Use the commands above to check the log files of both the beacon chain and validator. If any important command line flags have changed, a notice should appear in the logs. Even better, read the release notes in advance of an upgrade.

```console
sudo systemctl restart beacon-chain
sudo systemctl restart validator
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

- Replace SERVICE_NAME with the name of the service for which the service file was updated. For example, `sudo systemctl restart beacon-chain`.

### Updating Prysm Options

To update the configuration options of the beacon chain or validator, edit the Prysm configuration file located in the home directories for the services.

```console
sudo nano /home/validator/prysm-validator.yaml
sudo nano /home/beacon/prysm-beacon.yaml
```

Then restart the services:

```console
sudo systemctl restart validator
sudo systemctl restart beacon-chain
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

blackbox_exporter: [https://github.com/prometheus/blackbox_exporter](https://github.com/prometheus/blackbox_exporter)

node_exporter: [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)

Prometheus: [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)

Grafana: [https://grafana.com/docs/grafana/latest/installation/debian/](https://grafana.com/docs/grafana/latest/installation/debian/)

Dashboard: [https://github.com/metanull-operator/eth2-grafana](https://github.com/metanull-operator/eth2-grafana)

systemd: [https://www.freedesktop.org/software/systemd/man/systemd.unit.html](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)

geth: [https://geth.ethereum.org/docs/install-and-build/installing-geth](https://geth.ethereum.org/docs/install-and-build/installing-geth)

sshd: [https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh](https://blog.devolutions.net/2017/04/10-steps-to-secure-open-ssh)

ufw: [https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04)

ufw: [https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands](https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands)

ntpd: [https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-time-synchronization-on-ubuntu-18-04)

Connecting to the Kintsugi Testnet: https://hackmd.io/dFzKxB3ISWO8juUqPpJFfw

eth2-val-tools: https://github.com/protolambda/eth2-val-tools

ethereal: https://github.com/wealdtech/ethereal/

Prysm Setup Instructions for Kiln: https://hackmd.io/@prysmaticlabs/B1Q2SluWq