## MEV-Boost: Mainnet

**Instructions Compatibility: v2** 

The following instructions apply to systems set up under v2 of my installation instructions. v2 instructions were written for Ubuntu 22.04 LTS. Systems installed on or after July 19, 2022 were likely set up using v2 instructions. Instructions for v1 installations are available [here](../v1/mev-boost.md).

------

These instructions will help you install and configure [MEV-Boost](https://github.com/flashbots/mev-boost) for a mainnet, merge-ready [Prysm](https://github.com/prysmaticlabs/prysm/) installation.

## Install Go

Compiling MEV-Boost requires Go version 1.18 or higher.

Run the following to check your current version.

```console
go version
```

If Go is not at least version 1.18 or if Go is not installed...

```console
cd
wget https://go.dev/dl/go1.19.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.19.linux-amd64.tar.gz
sudo ln -s /usr/local/go/bin/go /usr/bin/go
rm go1.19.linux-amd64.tar.gz
```

Verify that you are now running Go 1.18+ by running the following command.

```console
go version
```

## MEV-Boost Installation

### MEV-Boost User Account

Create a user account in which to run MEV-Boost.

```console
sudo adduser --home /home/mev-boost --disabled-password --gecos 'MEV-Boost Relay' mev-boost
```

Create a directory to store the `mev-boost` binary.

- `-u mev-boost` runs the command as the mev-boost user so we don't have to change the directory permissions to the `mev-boost` user later on.

```console
sudo -u mev-boost mkdir /home/mev-boost/bin
```

### Install MEV-Boost

Install the latest release of MEV-Boost using `go install`. This will install into your user directories.

```console
cd
go install github.com/flashbots/mev-boost@latest
```

### Install MEV-Boost

Copy the `mev-boost` binary to the `bin` directory of the `mev-boost` account, and change the ownership of the binary to the `mev-boost` account.

```console
sudo cp ~/go/bin/mev-boost /home/mev-boost/bin
sudo chown mev-boost:mev-boost /home/mev-boost/bin/mev-boost
```

### Configure MEV-Boost to Run as System Service

Create a systemd service file to start the MEV-Boost service.

```console
sudo nano /etc/systemd/system/mev-boost.service
```

Add the following lines to the mev-boost service file. The `ExecStart` line includes two relays, Flashbots and bloXroute. You can remove one or the other to suit your ethical inclinations.

```
[Unit]
Description=MEV-Boost Relay
StartLimitIntervalSec=0
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
Restart=always
RestartSec=5
User=mev-boost
WorkingDirectory=/home/mev-boost
ExecStart=/home/mev-boost/bin/mev-boost \
		-mainnet \
		-relay-check \
		-relays https://0xac6e77dfe25ecd6110b8e780608cce0dab71fdd5ebea22a16c0205200f2f8e2e3ad3b71d3499c54ad14d6c21b41a37ae@boost-relay.flashbots.net,https://0xad0a8bb54565c2211cee576363f3a347089d2f07cf72679d16911d740262694cadb62d7fd7483f27afd714ca0f1b9118@bloxroute.ethical.blxrbdn.com

[Install]
WantedBy=multi-user.target
```

Enable MEV-Boost as a system service so that it automatically starts when the system boots, and start it now.

```
sudo systemctl enable --now mev-boost
```

Check the logs for success.

```console
sudo journalctl -fu mev-boost.service
```

If successful, the logs should look something like the following.

```
Sep 05 23:38:56 nuc systemd[1]: Started MEV-Boost Relay.
Sep 05 23:38:56 nuc mev-boost[59746]: time="2022-09-05T23:38:56-06:00" level=info msg="mev-boost v0.8.2-5-ge9b82f5" module=cli
Sep 05 23:38:56 nuc mev-boost[59746]: time="2022-09-05T23:38:56-06:00" level=info msg="Using genesis fork version: 0x00000000" module=cli
Sep 05 23:38:56 nuc mev-boost[59746]: time="2022-09-05T23:38:56-06:00" level=info msg="using 2 relays" module=cli relays="[{0xac6e77dfe25ecd6110b8e780608cce0dab71fdd5ebea22a16c0205200f2f8e2e3ad3b71d3499c54ad14d6c21b41a37ae https://0xac6e77dfe25ecd6110b8e780608cce0dab71fdd5ebea22a16c0205200f2f8e2e3ad3b71d3499c54ad14d6c21b41a37ae@boost-relay.flashbots.net} {0xad0a8bb54565c2211cee576363f3a347089d2f07cf72679d16911d740262694cadb62d7fd7483f27afd714ca0f1b9118 https://0xad0a8bb54565c2211cee576363f3a347089d2f07cf72679d16911d740262694cadb62d7fd7483f27afd714ca0f1b9118@bloxroute.ethical.blxrbdn.com}]"
Sep 05 23:38:56 nuc mev-boost[59746]: time="2022-09-05T23:38:56-06:00" level=info msg="Checking relay" module=service relay="https://0xac6e77dfe25ecd6110b8e780608cce0dab71fdd5ebea22a16c0205200f2f8e2e3ad3b71d3499c54ad14d6c21b41a37ae@boost-relay.flashbots.net"
Sep 05 23:38:56 nuc mev-boost[59746]: time="2022-09-05T23:38:56-06:00" level=info msg="Checking relay" module=service relay="https://0xad0a8bb54565c2211cee576363f3a347089d2f07cf72679d16911d740262694cadb62d7fd7483f27afd714ca0f1b9118@bloxroute.ethical.blxrbdn.com"
Sep 05 23:38:56 nuc mev-boost[59746]: time="2022-09-05T23:38:56-06:00" level=info msg="listening on localhost:18550" module=cli
```

You can check the status of the MEV-Boost service with the `systemctl status` command.

```console
sudo systemctl status mev-boost
```

The output will look similar to the following.

```
● mev-boost.service - MEV-Boost Relay
     Loaded: loaded (/etc/systemd/system/mev-boost.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-09-05 23:57:06 MDT; 20s ago
   Main PID: 59985 (mev-boost)
      Tasks: 10 (limit: 38085)
     Memory: 7.4M
     CGroup: /system.slice/mev-boost.service
             └─59985 /home/mev-boost/bin/mev-boost -mainnet -relay-check -relays https://0xac6e77dfe25ecd6110b8e780608cce0dab71fdd5ebea22a16c0205200f2f8e2e3ad3b71d3499c54ad14d6c21b41a37ae@boost-relay.flashbots.net,https://0xad0a8bb54565c2211cee576363f3>
```

This will be followed by the last few log lines.

Be sure to look for the following:

- The `Loaded` line says `enabled` instead of `disabled`. This means that the MEV-Boost service will start on system boot
- The `Active` line says `active (running)` which means the MEV-Boost service appears to be running currently

## Update Prysm Configuration

### Beacon Chain

Edit the Prysm Beacon Chain configuration file to configure communication with MEV-Boost.

```console
sudo nano /home/prysm-beacon/prysm-beacon.yaml
```

Add the following line to that file.

```console
http-mev-relay: "http://127.0.0.1:18550"
```

Restart the Prysm beacon chain client and review the logs.

```console
sudo systemctl restart prysm-beacon; sudo journalctl -fu prysm-beacon
```

If successful, the logs should include lines similar to the following.

```
Sep 05 23:40:46 nuc beacon-chain-v3.0.0[59788]: time="2022-09-05 23:40:46" level=info msg="Builder has been configured" endpoint="http://127.0.0.1:18550"
Sep 05 23:40:46 nuc beacon-chain-v3.0.0[59788]: time="2022-09-05 23:40:46" level=warning msg="Outsourcing block construction to external builders adds non-trivial delay to block propagation time.  Builder-constructed blocks or fallback blocks may get orphaned. Use at your own risk!"
```

### Validator

Edit the Prysm Validator configuration file to configure registration of validators.

```console
sudo nano /home/prysm-validator/prysm-validator.yaml
```

Add the following line to that file.

```console
enable-builder: true
```

Restart the Prysm validator client and review the logs.

```console
sudo systemctl restart prysm-validator; sudo journalctl -fu prysm-validator
```

Look for any errors/warnings that appear.

## Updating MEV-Boost

Monitor MEV-Boost releases [here](https://github.com/flashbots/mev-boost/releases). When a new release is available. Use the following instructions to update.

Download latest release.

```console
cd
go install github.com/flashbots/mev-boost@latest
```

Stop `mev-boost` service.

```console
sudo systemctl stop mev-boost
```

Copy the `mev-boost` binary to the `bin` directory of the `mev-boost` account, and change the ownership of the binary to the `mev-boost` account. Changing ownership should not be necessary if you followed the original installation instructions.

```console
sudo cp ~/go/bin/mev-boost /home/mev-boost/bin
sudo chown mev-boost:mev-boost /home/mev-boost/bin/mev-boost
```

Start MEV-Boost and monitor logs.

```console
sudo systemctl start mev-boost; sudo journalctl -fu mev-boost
```

See example of logs from the installation section above.
