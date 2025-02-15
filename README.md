# PulseChain Testnet Validator Node Setup Helper Scripts

![pls-testnet-validator-htop](https://user-images.githubusercontent.com/100790377/229965674-75593b5a-3fa6-44fe-8f47-fc25e9d3ce21.png)

**Please read ALL the instructions as they will explain and tell you how to run these scripts and the caveats.**

When you download the script, you may need to `chmod +x pulsechain-validator-setup.sh` to make the script executable and able to run on the system.

# Description

The setup script installs pre-reqs, golang, rust, go-pulse (geth fork) and lighthouse on a fresh, clean Ubuntu OS for getting a PulseChain Testnet (V3) Validator Node setup and running with **Geth (go-pulse)** and **Lighthouse** clients.

There are other helper scripts that do various things, check the notes for each one specifically for more info.

Note: the pulsechain validator setup script currently DOES NOT install monitoring/metrics packages such as Grafana or Prometheous, that may be done in a separate monitoring setup script or guides as referenced [below](https://github.com/rhmaxdotorg/pulsechain-validator#setting-up-monitoring-with-prometheus-and-grafana).

# Usage

```
$ ./pulsechain-validator-setup.sh [0x...YOUR ETHEREUM FEE ADDRESS] [12.89...YOUR SERVER IP ADDRESS]
```

## Command line options

- **ETHEREUM FEE ADDRESS** is the FEE_RECIPIENT value for --suggested-fee-recipient to a wallet address you want to recieve priority fees from users whenever your validator proposes a new block (else it goes to the burn address)

- **SERVER_IP_ADDRESS** to your validator server's IP address

Note: you may get prompted throughout the process to hit [Enter] for OK and continue the process

For example when running Ubuntu on AWS EC2 cloud service, you can expect to hit OK on kernel upgrade notice, [Enter] or "1" to continue Rust install process and so on

# Environment
Tested on **Ubuntu 22.04** (on Amazon AWS EC2 /w M2.2Xlarge VM) running as a non-root user (ubuntu) with sudo privileges

**IMPORTANT things to do AFTER RUNNING THIS SCRIPT to complete the node setup**

1) Generate validator keys with deposit tool, import them into lighthouse and make your 32m tPLS deposit on the launchpad

Note: generate your keys on a different, secure machine (NOT on the validator server) and transfer them over for import

```
$ sudo apt install -y python3-pip
$ git clone https://gitlab.com/pulsechaincom/staking-deposit-cli.git
$ cd staking-deposit-cli && pip3 install -r requirements.txt && sudo python3 setup.py install
$ ./deposit.sh new-mnemonic
```

Then follow the instructions from there, copy them over to the validator and import into lighthouse AS THE NODE USER (not the 'ubuntu' user on ec2)

Something like this should work
```
$ sudo -u node bash
$ lighthouse account validator import --directory ~/validator_keys --network=pulsechain_testnet_v3
```

2) Start the beacon and validator clients

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable lighthouse-beacon lighthouse-validator
$ sudo systemctl start lighthouse-beacon lighthouse-validator
```

If you want to look at lighthouse debug logs (similar to geth)

```
$ journalctl -u lighthouse-beacon.service (with -f to get the latest logs OR without the get the beginning)
$ journalctl -u lighthouse-validator.service
```

Now let's get validating! @rhmaximalist

# Debugging

## Check the Blockchain Sync Progress

### Geth
```
$ curl -s http://localhost:8545 -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":67}' | jq

{
  "jsonrpc": "2.0",
  "id": 67,
  "result": {
  "currentBlock": "0xffe4e3", // THIS IS WHERE YOU ARE
  "highestBlock": "0xffe8fa", // THIS IS WHERE YOU’RE GOING
  [full output was truncated for brevity]
  }
}
```

So you can compare the current with the highest to see how far you are from being fully sync’d. Or is result=false, you are sync'd.

```
$ curl -s http://localhost:8545 -H "Content-Type: application/json" --data "{\"jsonrpc\":\"2.0\",\"method\":\"eth_syncing\",\"params\":[],\"id\":67}" | jq
{
  "jsonrpc": "2.0",
  "id": 67,
  "result": false
}
```
### Lighthouse
```
$ curl -s http://localhost:5052/lighthouse/ui/health | jq
{
  "data": {
	"total_memory": XXXX,
	"free_memory": XXXX,
	"used_memory": XXXX,
	"os_version": "Linux XXXX",
	"host_name": "XXXX",
	"network_name": "XXXX",
	"network_bytes_total_received": XXXX,
	"network_bytes_total_transmit": XXXX,
	"nat_open": true,
	"connected_peers": 0, // PROBLEM
	"sync_state": "Synced"
  [full output was truncated for brevity]
  }
}
```

Use use boot nodes in your Lighthouse beacon node cmdline to help with increasing your peer connections (can take it from 0 to 30 real fast).

```
--boot-nodes enr:-L64QNIt1R1_ou9Aw5ci8gLAsV1TrK2MtWiPNGy21YsTW0HpA86hGowakgk3IVEZNjBOTVdqtXObXyErbEfxEi8Y8Z-CARSHYXR0bmV0c4j__________4RldGgykFuckgYAAAlE__________-CaWSCdjSCaXCEA--2T4lzZWNwMjU2azGhArzEiK-HUz_pnQBn_F8g7sCRKLU4GUocVeq_TX6UlFXIiHN5bmNuZXRzD4N0Y3CCIyiDdWRwgiMo

```

```
$ curl -s http://localhost:5052/lighthouse/syncing | jq
{
  "data": "Synced"
}
```
## Look at Client Service Status

```
$ sudo systemctl status geth lighthouse-beacon lighthouse-validator

● geth.service - Geth (Go-Pulse)
     Loaded: loaded (/etc/systemd/system/geth.service; enabled; vendor preset: enabled)
     [some output truncated for brevity]

Apr 00 19:30:20 server geth[126828]: INFO Unindexed transactions blocks=1 txs=56   tail=14,439,524 elapsed=2.966ms
Apr 00 19:30:30 server geth[126828]: INFO Imported new potential chain segment blocks=1 txs=35   mgas=1.577  elapsed=21.435ms     mgasps=73.569  number=16,789,524 hash=xxxxd7..xxx>
Apr 00 19:30:30 server geth[126828]: INFO Chain head was updated                   number=16,789,xxx hash=xxxxd7..cdxxxx root=xxxx9c..03xxxx elapsed=1.345514ms
Apr 00 19:30:30 server geth[126828]: INFO Unindexed transactions blocks=1 txs=96   tail=14,439,xxx elapsed=4.618ms

● lighthouse-beacon.service - Lighthouse Beacon
     Loaded: loaded (/etc/systemd/system/lighthouse-beacon.service; enabled; vendor preset: enabled)
     [some output truncated for brevity]

Apr 00 19:30:05 server lighthouse[126782]: INFO Synced slot: 300xxx, block: 0x8355…xxxx, epoch: 93xx, finalized_epoch: 93xx, finalized_root: 0x667f…707b, exec_>

Apr 00 19:30:10 server lighthouse[126782]: INFO New block received root: 0xxxxxxxxxf5e1364e34de345ab72bf1632e814915eb3fdc888e5b83aaxxxxxxxx, slot: 300061

Apr 00 19:30:15 server lighthouse[126782]: INFO Synced slot: 300xxx, block: 0x681e…xxxx, epoch: 93xx, finalized_epoch: 93xx, finalized_root: 0x667f…707b, exec_>

● lighthouse-validator.service - Lighthouse Validator
     Loaded: loaded (/etc/systemd/system/lighthouse-validator.service; enabled; vendor preset: enabled)
     [some output truncated for brevity]

Apr 00 19:30:05 server lighthouse[126779]: Apr 06 19:30:05.000 INFO Connected to beacon node(s)             synced: X, available: X, total: X, service: notifier
Apr 00 19:30:05 server lighthouse[126779]: INFO All validators active slot: 300xxx, epoch: 93xx, total_validators: X, active_validators: X, current_epoch_proposers: 0, servic>
```

# Reset Validator Script
This helper script deletes all your validator data so you can try the setup again if you want a fresh install or feel like you made an error.

Be careful! It deletes and resets things, so read the code and make sure you understand what it does before using it.

# AWS EC2 Helper Script
Just some nice-to-haves if you're using the AWS Cloud for your validator server.

# AWS Cloud Setup
* [How to run a cloud server on AWS](https://docs.google.com/document/d/1eW0SDT8IvZrla7gywK32Rl3QaQtVoiOu5OaVhUKIDg8/edit)

# Staking Deposit Client Walkthrough

* [Validator Key Generation and Management](https://docs.google.com/document/d/1tl_nql6-Bqyo5yqFDJ2aqjAQoBAK0FtcCYSKpGXg0hw/edit)

# Details for all PulseChain clients (/w Ethereum Testnet notes)
* [Geth, Erigon, Prysm and Lighthouse](https://docs.google.com/document/d/1RkAWt0Q_DmYpnykHFM4Qf5ItDLPLi-kaj1PDG74Mftg/edit)

# Setting up monitoring with Prometheus and Grafana
See the guides below for help getting them setup
* https://www.coincashew.com/coins/overview-eth/guide-or-how-to-setup-a-validator-on-eth2-mainnet/part-i-installation/monitoring-your-validator-with-grafana-and-prometheus
* https://schh.medium.com/port-forwarding-via-ssh-ba8df700f34d

You can setup grafana for secure access externally as opposed to the less secure way of forwarding port 3000 on the firewall and open it up to the world, which could put your server at risk next time Grafana has a security bug that anyone interested enough can exploit.

```
ssh -i key.pem -N ubuntu@validator-server-IP -L 8080:localhost:3000
```

Then open a browser window on your computer and login to grafana yourself without exposing it externally to the world. Magic, huh!

# Community Guides and Scripts
* https://gitlab.com/davidfeder/validatorscript/-/blob/5fa11c7f81d8292779774b8dff9144ec3e44d26a/PulseChain_V3_Script.txt
* https://www.hexpulse.info/docs/node-setup.html
* https://togosh.medium.com/pulsechain-validator-setup-guide-70edae00b344
* https://github.com/tdslaine/install_pulse_node

# FAQ

* What server specs do you need to be a validator?

Specs and preferences vary between who you talk to, but at least 32gb ram and a beefy i7 or server-based processor and 2TB SSD hard disk. In the cloud, this roughly translates into a M2.2XLarge EC2 instance + 2TB disk.

* How long does it take to sync the blockchain clients?

It depends on your bandwidth, server specs and the state of the network, but you should expect anywhere from 24 - 96hrs for a validator node to sync.

* Where can I find additional help on PulseChain dev stuff and being a validator?

https://t.me/PulseDev

# Additional Resources and References
- https://gitlab.com/pulsechaincom
- https://gammadevops.substack.com/p/part-1-introduction-to-validator
- https://gitlab.com/davidfeder/validatorscript/-/blob/5fa11c7f81d8292779774b8dff9144ec3e44d26a/PulseChain_V3_Script.txt
- https://www.hexpulse.info/docs/node-setup.html
- https://togosh.medium.com/pulsechain-validator-setup-guide-70edae00b344
- https://github.com/tdslaine/install_pulse_node
- https://gitlab.com/Gamesys10/pulsechain-node-guide
- https://lighthouse-book.sigmaprime.io/api-lighthouse.html
- https://lighthouse-book.sigmaprime.io/key-management.html
- https://docs.gnosischain.com/node/guide/validator/run/lighthouse
- https://ethereum.stackexchange.com/questions/394/how-can-i-find-out-what-the-highest-block-is
- https://www.coincashew.com/coins/overview-eth/guide-or-how-to-setup-a-validator-on-eth2-mainnet/part-i-installation/monitoring-your-validator-with-grafana-and-prometheus
- https://schh.medium.com/port-forwarding-via-ssh-ba8df700f34d
- https://www.youtube.com/watch?v=lbUnlIL_yLs&ab_channel=Oooly
- https://www.reddit.com/r/ethstaker/comments/txj5vh/technical_overview_of_validator_need_some_help/
- https://docs.prylabs.network/docs/troubleshooting/issues-errors
- https://pawelurbanek.com/ethereum-node-aws
- https://chasewright.com/getting-started-with-turbo-geth-on-ubuntu/
- https://docs.prylabs.network/docs/prysm-usage/p2p-host-ip
- https://www.blocknative.com/blog/an-ethereum-stakers-guide-to-slashing-other-penalties
- https://goerli.launchpad.ethstaker.cc/en/faq
- https://www.coincashew.com/coins/overview-eth/guide-or-how-to-setup-a-validator-on-eth2-mainnet/part-i-installation/guide-or-security-best-practices-for-a-eth2-validator-beaconchain-node
- https://ethereum.stackexchange.com/questions/3887/how-to-reduce-the-chances-of-your-ethereum-wallet-getting-hacked
- https://docs.prylabs.network/docs/install/install-with-script
- https://7goldfish.com/Eth_Staking_Testnet_on_AWS.html
- https://mirror.xyz/steinkirch.eth/F5PI4eqShKTGlx0GzL0Lq0-vHQ6b14OoV4ylE2FMsAc
- https://consensys.net/blog/developers/my-journey-to-being-a-validator-on-ethereum-2-0-part-5/
- https://www.monkeyvault.net/secure-aws-infrastructure-with-vpc-a-terraform-guide/ (VPCs guide too)
- https://hackmd.io/@prysmaticlabs/HkSSMpDtt
- https://medium.com/@mshmulevich/running-ethereum-nodes-in-high-availability-cluster-on-aws-aefd08d4d81
- https://chasewright.com/getting-started-with-turbo-geth-on-ubuntu/
- https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-prysm-581fb1969460
- https://www.blocknative.com/blog/ethereum-validator-lighthouse-geth
