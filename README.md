# Setup an Ethereum Mainnet Staking System with Prysm/Geth on Ubuntu

These pages contain instructions for setting up an Ethereum mainnet staking system on Ubuntu. These instructions have been tested on Intel NUC systems with 2TB SSD and 32GB RAM, but the instructions should apply equally to most AMD64 architecture systems running Ubuntu.

If you are setting up a new system for staking, I recommend you begin with my [v2 instructions](v2/). If you set up your system on or before July 19, 2022, you may be looking for information about my [v1 instructions](v1/).

## Current Setup and Maintenance Instructions

- [Setup an Ethereum Mainnet Staking System on Ubuntu](v2/setup.md) - v2 of my setup instructions to install and configure an Ethereum mainnet staking system using Prysm and Geth.
- [Prune Geth](prune_geth.md) - How to prune Geth to reduce disk usage.
- [MEV-Boost: Mainnet](v2/mev-boost.md) - How to set up MEV-Boost for mainnet for v2 installations.

## Setup and Maintenance for Prior Versions

### v1 Installations

- [Setup an Ethereum Mainnet Staking System on Ubuntu](v1/setup.md) - v1 of my setup instructions to install and configure an Ethereum mainnet staking system using Prysm and Geth.
- [Merge Updates for v1 Installations](v1/merge_updates.md) - Prepare your v1 Prysm/Geth installation for the merge.
- [Prune Geth](prune_geth.md) - How to prune Geth to reduce disk usage.
- [MEV-Boost: Mainnet](v1/mev-boost.md) - How to set up MEV-Boost for mainnet for v1 installations.

## Which Version Do You Have?

The easiest way to tell which version of my instructions you used is to look at the systemd service files for Prysm.

```console
ls -l /etc/systemd/system/*beacon*.service
```

If this command returns a file named `beacon-chain.service`, then you installed with [v1](v1/) of my instructions. If this command returns a file named `prysm-beacon.service`, then you installed with [v2](v2/) of my instructions.
