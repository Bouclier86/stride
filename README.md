# Stride-testnet. Task #8
Set up a relayer / ibc channel

<p align="center">
  <img height="100" height="auto" src="https://github.com/Bouclier86/stride/blob/8b65b5beeeb08eb19203bffc8ce0dac3b7efb84d/images/stride_logo.png">
</p>

# Set up a relayer / ibc channel with another testnet
In current example we will learn how to set up IBC relayer between two cosmos chains.

## Preparation before you start
Before setting up relayer you need to make sure you already have:
1. Fully synchronized RPC nodes for each Cosmos project you want to connect. In my case it's Gaia and Stride.
2. RPC enpoints should be exposed and available from hermes instance.
3. Indexing is set to `kv` and is enabled on each node — indexer = "kv"
4. For each chain you will need to have wallets that are funded with tokens. This wallets will be used to do all relayer stuff and pay commission.

## Update system
```
sudo apt update && sudo apt upgrade -y
```

## Install dependencies
```
sudo apt install unzip -y
```

## Set up variables
All settings below are just example for IBC Relayer between stride `STRIDE-TESTNET-2` and juno `GAIA` testnets. Please, fill with your own values. You can find your RPC_ADDR from /root/<.NODE_NAME>/config/config.toml and GRPC_ADDR from /root/.<.NODE_NAME>/config/app.toml
```
RELAYER_NAME='Boúcliér86#6449' # Add your Discord username here
```

### Chain S
```
CHAIN_ID_S='STRIDE-TESTNET-2'
RPC_ADDR_S='127.0.0.1:26657'
GRPC_ADDR_S='127.0.0.1:9090'
ACCOUNT_PREFIX_S='stride'
TRUSTING_PERIOD_S='8hours'
DENOM_S='ustrd'
MNEMONIC_S='<YOUR MNEMONIC>'
```

### Chain G
```
CHAIN_ID_G='GAIA'
RPC_ADDR_G='127.0.0.1:23657'
GRPC_ADDR_G='127.0.0.1:23090'
ACCOUNT_PREFIX_G='cosmos'
TRUSTING_PERIOD_G='8hours'
DENOM_G='uatom'
MNEMONIC_G='<YOUR MNEMONIC>'
```

## Make hermes home dir
```
mkdir $HOME/.hermes
```

## Download Hermes
```
wget https://github.com/informalsystems/ibc-rs/releases/download/v1.0.0-rc.1/hermes-v1.0.0-rc.1-x86_64-unknown-linux-gnu.tar.gz
mkdir -p $HOME/.hermes/bin
tar -C $HOME/.hermes/bin/ -vxzf hermes-v1.0.0-rc.1-x86_64-unknown-linux-gnu.tar.gz
echo 'export PATH="$HOME/.hermes/bin:$PATH"' >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## Create hermes config
Generate hermes config file using variables we have defined above
```
sudo tee $HOME/.hermes/config.toml > /dev/null <<EOF
[global]
log_level = 'info'

[mode]

[mode.clients]
enabled = true
refresh = true
misbehaviour = true

[mode.connections]
enabled = false

[mode.channels]
enabled = false

[mode.packets]
enabled = true
clear_interval = 100
clear_on_start = true
tx_confirmation = true

[rest]
enabled = true
host = '127.0.0.1'
port = 3000

[telemetry]
enabled = true
host = '127.0.0.1'
port = 3001

[[chains]]
### CHAIN_S ###
id = '${CHAIN_ID_S}'
rpc_addr = 'http://${RPC_ADDR_S}'
grpc_addr = 'http://${GRPC_ADDR_S}'
websocket_addr = 'ws://${RPC_ADDR_S}/websocket'
rpc_timeout = '10s'
account_prefix = '${ACCOUNT_PREFIX_S}'
key_name = 'wallet'
address_type = { derivation = 'cosmos' }
store_prefix = 'ibc'
default_gas = 100000
max_gas = 2500000
gas_price = { price = 0.0025, denom = '${DENOM_S}' }
gas_multiplier = 1.1
max_msg_num = 30
max_tx_size = 2097152
clock_drift = '5s'
max_block_time = '30s'
trusting_period = '${TRUSTING_PERIOD_S}'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '${RELAYER_NAME}'
[chains.packet_filter]
policy = 'allow'
list = [
  ['ica*', '*'], # allow relaying on all channels whose port starts with ica
  ['transfer', 'channel-0'],
]

[[chains]]
### CHAIN_G ###
id = '${CHAIN_ID_G}'
rpc_addr = 'http://${RPC_ADDR_G}'
grpc_addr = 'http://${GRPC_ADDR_G}'
websocket_addr = 'ws://${RPC_ADDR_G}/websocket'
rpc_timeout = '10s'
account_prefix = '${ACCOUNT_PREFIX_G}'
key_name = 'wallet'
address_type = { derivation = 'cosmos' }
store_prefix = 'ibc'
default_gas = 100000
max_gas = 2500000
gas_price = { price = 0.0025, denom = '${DENOM_G}' }
gas_multiplier = 1.1
max_msg_num = 30
max_tx_size = 2097152
clock_drift = '5s'
max_block_time = '30s'
trusting_period = '${TRUSTING_PERIOD_G}'
trust_threshold = { numerator = '1', denominator = '3' }
memo_prefix = '${RELAYER_NAME}'
[chains.packet_filter]
policy = 'allow'
list = [
  ['ica*', '*'], # allow relaying on all channels whose port starts with ica
  ['transfer', 'channel-0'],
]
EOF
```

## Verify hermes configuration is correct
Before proceeding further please check if your configuration is correct
```
hermes health-check
```

Healthy output should look like:
```
![image](https://github.com/Bouclier86/stride/blob/029720679a759f4bf81e5db8fe488a5cdbddc680/images/health-check.png)
```

## Recover wallets using mnemonic files
Before you proceed with this step, please make sure you have created and funded with tokens seperate wallets on each chain
```
sudo tee $HOME/.hermes/${CHAIN_ID_S}.mnemonic > /dev/null <<EOF
${MNEMONIC_S}
EOF
sudo tee $HOME/.hermes/${CHAIN_ID_G}.mnemonic > /dev/null <<EOF
${MNEMONIC_G}
EOF
hermes keys add --chain ${CHAIN_ID_S} --mnemonic-file $HOME/.hermes/${CHAIN_ID_S}.mnemonic
hermes keys add --chain ${CHAIN_ID_G} --mnemonic-file $HOME/.hermes/${CHAIN_ID_G}.mnemonic
```

Successful output should look like:
```
2022-07-21T19:54:13.778550Z  INFO ThreadId(01) using default configuration from '/root/.hermes/config.toml'
Success: Restored key 'wallet' (<YOUR_STRIDE_ADDRESS>) on chain STRIDE-TESTNET-2
2022-07-21T19:54:14.956171Z  INFO ThreadId(01) using default configuration from '/root/.hermes/config.toml'
Success: Restored key 'wallet' (<YOUR_COSMOS_ADDRESS>) on chain GAIA
```

## Create hermes service daemon
```
sudo tee /etc/systemd/system/hermesd.service > /dev/null <<EOF
[Unit]
Description=hermes
After=network-online.target

[Service]
User=$USER
ExecStart=$(which hermes) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable hermesd
sudo systemctl restart hermesd
```

## Check hermes logs
```
journalctl -u hermesd -f -o cat
```

## Successfull logs should look like this:
```
2022-07-27T22:43:08.019651Z  INFO ThreadId(01) Scanned chains:
2022-07-27T22:43:08.019675Z  INFO ThreadId(01) # Chain: STRIDE-TESTNET-2
  - Client: 07-tendermint-0
    * Connection: connection-0
      | State: OPEN
      | Counterparty state: OPEN
      + Channel: channel-0
        | Port: transfer
        | State: OPEN
        | Counterparty: channel-0
      + Channel: channel-1
        | Port: icacontroller-GAIA.DELEGATION
        | State: OPEN
        | Counterparty: channel-4
      + Channel: channel-2
        | Port: icacontroller-GAIA.FEE
        | State: OPEN
        | Counterparty: channel-1
      + Channel: channel-3
        | Port: icacontroller-GAIA.WITHDRAWAL
        | State: OPEN
        | Counterparty: channel-2
      + Channel: channel-4
        | Port: icacontroller-GAIA.REDEMPTION
        | State: OPEN
        | Counterparty: channel-3
# Chain: GAIA
  - Client: 07-tendermint-0
    * Connection: connection-0
      | State: OPEN
      | Counterparty state: OPEN
      + Channel: channel-0
        | Port: transfer
        | State: OPEN
        | Counterparty: channel-0
      + Channel: channel-1
        | Port: icahost
        | State: OPEN
        | Counterparty: channel-2
      + Channel: channel-2
        | Port: icahost
        | State: OPEN
        | Counterparty: channel-3
      + Channel: channel-3
        | Port: icahost
        | State: OPEN
        | Counterparty: channel-4
      + Channel: channel-4
        | Port: icahost
        | State: OPEN
        | Counterparty: channel-1
2022-07-27T22:43:08.020020Z  INFO ThreadId(01) connection is Open, state on destination chain is Open chain=STRIDE-TESTNET-2 connection=connection-0 counterparty_chain=GAIA
2022-07-27T22:43:08.020035Z  INFO ThreadId(01) connection is already open, not spawning Connection worker chain=STRIDE-TESTNET-2 connection=connection-0
2022-07-27T22:43:08.020045Z  INFO ThreadId(01) no connection workers were spawn chain=STRIDE-TESTNET-2 connection=connection-0
2022-07-27T22:43:08.020052Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=STRIDE-TESTNET-2 counterparty_chain=GAIA channel=channel-0
2022-07-27T22:43:08.024013Z  INFO ThreadId(01) spawned client worker: client::GAIA->STRIDE-TESTNET-2:07-tendermint-0
2022-07-27T22:43:08.028421Z  INFO ThreadId(01) done spawning channel workers chain=STRIDE-TESTNET-2 channel=channel-0
2022-07-27T22:43:08.028473Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=STRIDE-TESTNET-2 counterparty_chain=GAIA channel=channel-1
2022-07-27T22:43:08.031579Z  INFO ThreadId(01) done spawning channel workers chain=STRIDE-TESTNET-2 channel=channel-1
2022-07-27T22:43:08.031606Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=STRIDE-TESTNET-2 counterparty_chain=GAIA channel=channel-2
2022-07-27T22:43:08.034669Z  INFO ThreadId(01) done spawning channel workers chain=STRIDE-TESTNET-2 channel=channel-2
2022-07-27T22:43:08.034698Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=STRIDE-TESTNET-2 counterparty_chain=GAIA channel=channel-3
2022-07-27T22:43:08.037346Z  INFO ThreadId(01) done spawning channel workers chain=STRIDE-TESTNET-2 channel=channel-3
2022-07-27T22:43:08.037363Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=STRIDE-TESTNET-2 counterparty_chain=GAIA channel=channel-4
2022-07-27T22:43:08.041229Z  INFO ThreadId(01) done spawning channel workers chain=STRIDE-TESTNET-2 channel=channel-4
2022-07-27T22:43:08.041412Z  INFO ThreadId(01) spawning Wallet worker: wallet::STRIDE-TESTNET-2
2022-07-27T22:43:08.041445Z  INFO ThreadId(01) connection is Open, state on destination chain is Open chain=GAIA connection=connection-0 counterparty_chain=STRIDE-TESTNET-2
2022-07-27T22:43:08.041453Z  INFO ThreadId(01) connection is already open, not spawning Connection worker chain=GAIA connection=connection-0
2022-07-27T22:43:08.041462Z  INFO ThreadId(01) no connection workers were spawn chain=GAIA connection=connection-0
2022-07-27T22:43:08.041470Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=GAIA counterparty_chain=STRIDE-TESTNET-2 channel=channel-0
2022-07-27T22:43:08.048399Z  INFO ThreadId(01) spawned client worker: client::STRIDE-TESTNET-2->GAIA:07-tendermint-0
2022-07-27T22:43:08.053737Z  INFO ThreadId(01) done spawning channel workers chain=GAIA channel=channel-0
2022-07-27T22:43:08.053770Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=GAIA counterparty_chain=STRIDE-TESTNET-2 channel=channel-1
2022-07-27T22:43:08.057395Z  INFO ThreadId(01) done spawning channel workers chain=GAIA channel=channel-1
2022-07-27T22:43:08.057441Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=GAIA counterparty_chain=STRIDE-TESTNET-2 channel=channel-2
2022-07-27T22:43:08.061244Z  INFO ThreadId(01) done spawning channel workers chain=GAIA channel=channel-2
2022-07-27T22:43:08.061296Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=GAIA counterparty_chain=STRIDE-TESTNET-2 channel=channel-3
2022-07-27T22:43:08.064713Z  INFO ThreadId(01) done spawning channel workers chain=GAIA channel=channel-3
2022-07-27T22:43:08.064742Z  INFO ThreadId(01) channel is OPEN, state on destination chain is OPEN chain=GAIA counterparty_chain=STRIDE-TESTNET-2 channel=channel-4
2022-07-27T22:43:08.071960Z  INFO ThreadId(01) done spawning channel workers chain=GAIA channel=channel-4
2022-07-27T22:43:08.072413Z  INFO ThreadId(01) spawning Wallet worker: wallet::GAIA
2022-07-27T22:43:08.079299Z  INFO ThreadId(01) Hermes has started
2022-07-27T22:43:43.479282Z ERROR ThreadId(56) [STRIDE-TESTNET-2] error during batch processing: supervisor was not able to spawn chain runtime: missing chain config for 'uni-3' in configuration file
2022-07-27T22:44:11.376229Z  INFO ThreadId(442) packet_cmd{src_chain=STRIDE-TESTNET-2 src_port=icacontroller-GAIA.DELEGATION src_channel=channel-1 dst_chain=GAIA}: pulled packet data for 0 events; events_total=1 events_left=0
2022-07-27T22:44:12.592627Z  INFO ThreadId(442) packet_cmd{src_chain=STRIDE-TESTNET-2 src_port=icacontroller-GAIA.DELEGATION src_channel=channel-1 dst_chain=GAIA}:relay{odata=d9ab28da ->Destination @2-11522; len=1}: assembled batch of 2 message(s)
2022-07-27T22:44:12.762428Z  INFO ThreadId(442) packet_cmd{src_chain=STRIDE-TESTNET-2 src_port=icacontroller-GAIA.DELEGATION src_channel=channel-1 dst_chain=GAIA}:relay{odata=d9ab28da ->Destination @2-11522; len=1}: [Async~>GAIA] response(s): 1; Ok:36394BDD136B8E2400D0DFE00735C7D688069E467F281EFE76D3943871FD8D04
2022-07-27T22:44:12.762463Z  INFO ThreadId(442) packet_cmd{src_chain=STRIDE-TESTNET-2 src_port=icacontroller-GAIA.DELEGATION src_channel=channel-1 dst_chain=GAIA}:relay{odata=d9ab28da ->Destination @2-11522; len=1}: success
2022-07-27T22:44:20.873424Z  INFO ThreadId(453) packet_cmd{src_chain=GAIA src_port=icahost src_channel=channel-4 dst_chain=STRIDE-TESTNET-2}: pulled packet data for 0 events; events_total=1 events_left=0
2022-07-27T22:44:20.895913Z  INFO ThreadId(453) packet_cmd{src_chain=GAIA src_port=icahost src_channel=channel-4 dst_chain=STRIDE-TESTNET-2}:relay{odata=d169839a ->Destination @0-16135; len=1}: assembled batch of 2 message(s)
2022-07-27T22:44:20.902820Z  INFO ThreadId(453) packet_cmd{src_chain=GAIA src_port=icahost src_channel=channel-4 dst_chain=STRIDE-TESTNET-2}:relay{odata=d169839a ->Destination @0-16135; len=1}: [Async~>STRIDE-TESTNET-2] response(s): 1; Ok:150EEF4C0EB3EF413115C192C7A5575190AA1E0B8EFC8A52389E556C16A71C57
2022-07-27T22:44:20.902852Z  INFO ThreadId(453) packet_cmd{src_chain=GAIA src_port=icahost src_channel=channel-4 dst_chain=STRIDE-TESTNET-2}:relay{odata=d169839a ->Destination @0-16135; len=1}: success
```

You also should see `Update Client (Ibc)` transactions appearing in the explorer https://stride.explorers.guru/account/<YOUR_STRIDE_ADDRESS>

![image](https://github.com/Bouclier86/stride/blob/a14e586027a521b70d018d19c2c676ea4423a88f/images/irc-1.png)
![image](https://github.com/Bouclier86/stride/blob/a14e586027a521b70d018d19c2c676ea4423a88f/images/irc-2.png)
