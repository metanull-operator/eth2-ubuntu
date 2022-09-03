# Prune Geth

To prune the Geth database, use the following instructions.

**Note:** Geth will be down while pruning. Prior to the merge, your validators will not be able to propose blocks while pruning. After the merge, your validator may be down entirely.

```console
sudo systemctl stop geth
sudo -u geth /usr/bin/geth --datadir /home/geth/.ethereum snapshot prune-state
```

This will take a long time. After it is complete, restart Geth's normal operation.

```console
sudo systemctl start geth
```

Review the logs looking for any problems at start-up. Geth will need to sync back yup to the head of the chain, which will also take some time.

```console
sudo journalctl -fu geth
```



