
# Prysm/Geth Merge Updates for v1 Installations

**Instructions Compatibility: v1** 

The following instructions apply to systems set up under v1 of my installation instructions. v1 instructions were written for Ubuntu 20.04 LTS. Systems installed before July 19, 2022 were likely set up using v1 instructions. Instructions for v2 installations are available [here](../v2/).

------

These instructions are only directly applicable if you followed a pre-July 19, 2022 version of my instructions to set up a Prysm/Geth staking system. If you are configuring a new mainnet, merge-ready staking system from scratch, please use the [current version](../v2/) of my full staking instructions and not these merge update instructions. File locations, configuration locations, and service names may not match between the older versions of my instructions and the newer merge versions.

The following changes are required to prepare Prysm/Geth for the merge:

- Create a Shared Secret File for Prysm and Geth
  - After the merge, Prysm and Geth will use a secret to authenticate each other.
- Update Geth Command Line Arguments
  - Add `authrpc.vhosts` to define the hosts allowed to connect
  - Add `authrpc.jwtsecret` to define the location of the secret
- Update Prysm Configuration
  - Add `suggested-fee-recipient` to Prysm Beacon Chain configuration to provide a fallback address to which fees/tips should be sent.
  - Add `jwtsecret` to Prysm Beacon Chain configuration to define the location of the secret
  - Replace `http-web3provider` with `execution-endpoint` and port 8545 with port 8551 in the Prysm Beacon Chain configuration.
  - Add `suggested-fee-recipient` to Prysm Validator configuration to provide an address to which fees/tips should be sent.
- Update and Restart Services
- Monitor Clients for Additional Updates Prior to the Merge
  - Keep Geth and Prysm updated to the latest official releases

## Create a Shared Secret for Prysm and Geth

After the merge, Consensus Layer and Execution Layer clients will authenticate with one another using a shared secret.

First, we create a new user group called `ethereum`. We are going to add the `geth` and `beacon` user accounts as members of the `ethereum` group. 

```console
sudo groupadd ethereum
sudo usermod -a -G ethereum beacon
sudo usermod -a -G ethereum geth
```

Next, we will create a folder in which we will put the secret.

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

## Update Geth Command Line Arguments

Edit the Geth serviced file and add `--authrpc.jwtsecret=/srv/ethereum/secrets/jwtsecret --authrpc.vhosts="*"` to the command line arguments for Geth.

```console
sudo nano /etc/systemd/system/geth.service
```

Add `--authrpc.jwtsecret=/srv/ethereum/secrets/jwtsecret --authrpc.vhosts="*"` to the end of the `ExecStart` line. Afterward, your `ExecStart` line might look like the following

```console
ExecStart=/usr/bin/geth --http --http.addr 0.0.0.0 --authrpc.jwtsecret=/srv/ethereum/secrets/jwtsecret --authrpc.vhosts="*"
```

- The `authrpc.vhosts` value can be modified to suit your security needs.

## Update Prysm Configuration

Edit the Prysm Beacon configuration file...

```console
sudo nano /home/beacon/prysm-beacon.yaml
```

...and add the following lines...


```
suggested-fee-recipient: "0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
jwt-secret: "/srv/ethereum/secrets/jwtsecret"
execution-endpoint: "http://127.0.0.1:8551/"
```

 - Replace `0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` with an Ethereum address at which you will receive tips/fees.

Remove the following line, because `http-web3provider` has been replaced with `execution-endpoint` inserted above, and port 8545 has been replaced with port 8551.

```
http-web3provider: "http://127.0.0.1:8545/"
```

Edit the Prysm Validator configuration file...

```console
sudo nano /home/validator/prysm-validator.yaml
```

...and add the following line...

```
suggested-fee-recipient: "0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
```

- Replace `0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` with an Ethereum address at which you will receive tips/fees.

## Update and Restart Services

### Update Prysm

Your Prysm installation should update on its own after restarting using the following commands. Review logs for errors.

```console
sudo systemctl daemon-reload
sudo systemctl restart beacon-chain ; sudo journalctl -fu beacon-chain
sudo systemctl restart validator ; sudo journalctl -fu validator
```

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

If you have completed all of these steps, you should be merge-ready!

## Monitor Clients for Additional Updates Prior to the Merge

Please keep up to date on Geth and Prysm releases prior to the merge in case there are any bug releases. Use the following commands to update software as needed.

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

```console
sudo systemctl restart beacon-chain ; sudo journalctl -fu beacon-chain
sudo systemctl restart validator ; sudo journalctl -fu validator
```

Restart the Prysm beacon and monitor logs for any trouble connecting to Geth.

