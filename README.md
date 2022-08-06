# Stride-testnet. Task #8
Set up a relayer / ibc channel.


<p align="center">
  <img height="100" height="auto" src="https://user-images.githubusercontent.com/50621007/177221972-75fcf1b3-6e95-44dd-b43e-e32377685af8.png">
</p>

# Setup GO Relayer v2 between Stride and GAIA
In current example we will learn how to set up GO Relayer v2 between two cosmos chains

## Preparation before you start
Before setting up relayer you need to make sure you already have:
1. Fully synchronized RPC nodes for each Cosmos project you want to connect
2. RPC enpoints should be exposed and available from relayer instance
#### RPC configuration is located in `config.toml` file
```
# STRIDE
laddr = "tcp://0.0.0.0:16657" in $HOME/.stride/config/config.toml
# GAIA
laddr = "tcp://0.0.0.0:23657" in $HOME/.gaia/config/config.toml  
```

3. Indexing is set to `kv` and is enabled on each node
```
# STRIDE
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.stride/config/config.toml
# GAIA
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.gaia/config/config.toml  
```

4. Dont forget to restart fullnode service after making changes
5. For each chain you will need to have wallets that are funded with tokens. This wallets will be used to do all relayer stuff

## Set up variables
All settings below are just example for IBC Relayer between `STRIDE-TESTNET-2` and `GAIA` testnets. Please fill with your own values.
```
RELAYER_ID='kjnodes#8455' # add your Discord username here
```

### STRIDE
```
STRIDE_RPC_ADDR='<YOUR_STRIDE_RPC_ADDR>:<YOUR_STRIDE_RPC_PORT>' # Example: '127.0.0.1:16657'
STRIDE_MNEMONIC='your mnemonic goes here'
```

### GAIA
```
GAIA_RPC_ADDR='<YOUR_GAIA_RPC_ADDR>:<YOUR_GAIA_RPC_PORT>' # Example: '127.0.0.1:21657'
GAIA_MNEMONIC='your mnemonic goes here'
```

## Update system
```
sudo apt update && sudo apt upgrade -y
```

## Install GO
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.3"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Install and initialize GO Relayer v2
```
git clone https://github.com/cosmos/relayer.git
cd relayer && git checkout v2.0.0-rc4
make install
rly config init
```

## Create relayer configuration files
### 1. Generate CHAIN_A config file using variables we have defined above
```
sudo tee $HOME/.relayer/stride.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "STRIDE-TESTNET-2",
    "rpc-addr": "http://${STRIDE_RPC_ADDR}",
    "account-prefix": "stride",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.001ustrd",
    "debug": true,
    "timeout": "20s",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

### 2. Generate CHAIN_B config file using variables we have defined above
```
sudo tee $HOME/.relayer/gaia.json > /dev/null <<EOF
{
  "type": "cosmos",
  "value": {
    "key": "wallet",
    "chain-id": "GAIA",
    "rpc-addr": "http://${GAIA_RPC_ADDR}",
    "account-prefix": "cosmos",
    "keyring-backend": "test",
    "gas-adjustment": 1.2,
    "gas-prices": "0.001uatom",
    "debug": true,
    "timeout": "20s",
    "output-format": "json",
    "sign-mode": "direct"
  }
}
EOF
```

## Load chain configuration into the relayer
```
rly chains add --file=$HOME/.relayer/stride.json stride
rly chains add --file=$HOME/.relayer/gaia.json gaia
```

Check chains added to relayer
```
rly chains list
```

Successful output:
```
1: GAIA             -> type(cosmos) key(✘) bal(✘) path(✘)
2: STRIDE-TESTNET-2 -> type(cosmos) key(✘) bal(✘) path(✘)

```

## Load wallets into the relayer
```
rly keys restore stride wallet "${STRIDE_MNEMONIC}"
rly keys restore gaia wallet "${GAIA_MNEMONIC}"
```

Check wallet balance
```
rly q balance stride
rly q balance gaia
```

## Add path for STRIDE-TESTNET-2 and GAIA to relayer configuration file
Open `$HOME/.relayer/config/config.yaml` file and replace `paths: {}` with:
```
paths:
    stride-gaia:
        src:
            chain-id: STRIDE-TESTNET-2
            client-id: 07-tendermint-0
            connection-id: connection-0
        dst:
            chain-id: GAIA
            client-id: 07-tendermint-0
            connection-id: connection-0
        src-channel-filter:
            rule: allowlist
            channel-list: [channel-0, channel-1, channel-2, channel-3, channel-4]
```

## Update relayer memo with your discord account
```
sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/download/v4.23.1/yq_linux_amd64 && chmod +x /usr/local/bin/yq
yq -i ".global.memo = \"$RELAYER_ID\"" $HOME/.relayer/config/config.yaml
```

Check path is correct
```
rly paths list
```

Successful output:
```
0: stride-gaia -> chns(✔) clnts(✔) conn(✔) (STRIDE-TESTNET-2<>GAIA)
```
If everything is correct, we can proceed with relayer service creation

## Create service
```
sudo tee /etc/systemd/system/relayerd.service > /dev/null <<EOF
[Unit]
Description=GO Relayer v2 Service
After=network.target
[Service]
Type=simple
User=$USER
ExecStart=$(which rly) start stride-gaia -p events
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

## Start the service
```
sudo systemctl daemon-reload
systemctl enable relayerd
systemctl start relayerd
```

## Check logs
```
journalctl -u relayerd -f -o cat
```

Check your wallet transaction in explorer to find the GO v2 Relayer Update Client(IBC) message:

![image](https://user-images.githubusercontent.com/50621007/183149363-9c5108a5-1e59-400f-ade5-d4aaa12af53e.png)
