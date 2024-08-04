# Symphony


```
sudo apt update && sudo apt upgrade -y
sudo apt install make curl git wget htop tmux build-essential jq lz4 gcc unzip -y
```

#### Install GO
```
ver="1.22.2"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

#### Download and Build Binaries
```
cd $HOME
rm -rf symphony
git clone https://github.com/Orchestra-Labs/symphony
cd symphony
git checkout v0.2.1
make install
```

##### Initialize the Node
```
cd $HOME
MONIKER=gantimonikermu
echo "export MONIKER=$MONIKER" >> $HOME/.bash_profile
echo "export CHAIN_ID=symphony-testnet-2" >> $HOME/.bash_profile
echo "export SYMPHONY_PORT=15" >> $HOME/.bash_profile
source $HOME/.bash_profile

symphonyd init $MONIKER --chain-id $CHAIN_ID

wget -O $HOME/.symphonyd/config/genesis.json https://files.nodesync.top/Symphony/symphony-genesis.json
wget -O $HOME/.symphonyd/config/addrbook.json https://files.nodesync.top/Symphony/symphony-addrbook.json
```

#### Configure Node
```
SEEDS=""
PEERS="688b148e0a99b45c6b6ca6fbeae42f7a86c8ad4b@65.21.202.124:24856,..."

sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.symphonyd/config/config.toml
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.025note\"/" $HOME/.symphonyd/config/app.toml
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.symphonyd/config/config.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.symphonyd/config/config.toml
```

```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"

sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.symphonyd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.symphonyd/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.symphonyd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.symphonyd/config/app.toml
```

#### Set Custom Port
```
sed -i.bak -e "s%:1317%:${SYMPHONY_PORT}317%g;
s%:8080%:${SYMPHONY_PORT}080%g;
s%:9090%:${SYMPHONY_PORT}090%g;
s%:9091%:${SYMPHONY_PORT}091%g;
s%:8545%:${SYMPHONY_PORT}545%g;
s%:8546%:${SYMPHONY_PORT}546%g;
s%:6065%:${SYMPHONY_PORT}065%g" $HOME/.symphonyd/config/app.toml
sed -i.bak -e "s%:26658%:${SYMPHONY_PORT}658%g;
s%:26657%:${SYMPHONY_PORT}657%g;
s%:6060%:${SYMPHONY_PORT}060%g;
s%:26656%:${SYMPHONY_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${SYMPHONY_PORT}656\"%;
s%:26660%:${SYMPHONY_PORT}660%g" $HOME/.symphonyd/config/config.toml
sed -i \
  -e 's|^chain-id *=.*|chain-id = "symphony-testnet-2"|' \
  -e 's|^keyring-backend *=.*|keyring-backend = "test"|' \
  -e 's|^node *=.*|node = "tcp://localhost:15657"|' \
  $HOME/.symphonyd/config/client.toml
```

#### Download Latest Chain Snapshot
```
wget -O symphony_243703.tar.lz4 https://snapshots.polkachu.com/testnet-snapshots/symphony/symphony_243703.tar.lz4 --inet4-only

sudo systemctl stop symphonyd

cp ~/.symphonyd/data/priv_validator_state.json  ~/.symphonyd/priv_validator_state.json

symphonyd tendermint unsafe-reset-all --home $HOME/.symphonyd --keep-addr-book

lz4 -c -d symphony_243703.tar.lz4  | tar -x -C $HOME/.symphonyd

cp ~/.symphonyd/priv_validator_state.json  ~/.symphonyd/data/priv_validator_state.json

sudo systemctl start symphonyd

rm -v symphony_243703.tar.lz4

sudo journalctl -u symphonyd -f -o cat
```

#### Create Service and Start Node
```
sudo tee /etc/systemd/system/symphonyd.service > /dev/null <<EOF
[Unit]
Description=symphony-testnet
After=network-online.target

[Service]
User=$USER
ExecStart=$(which symphonyd) start --home $HOME/.symphonyd
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

#### Start Service
```
sudo systemctl daemon-reload
sudo systemctl enable symphonyd
sudo systemctl restart symphonyd
sudo journalctl -u symphonyd -f --no-hostname -o cat
```

#### Add New Wallet Key
```
symphonyd keys add wallet

symphonyd keys add wallet --recover
```

#### List All Keys
```
symphonyd keys list
```

##### Query Wallet Balance
```
symphonyd q bank balances $(symphonyd keys show wallet -a)
```
#### Check Sync Status
```
symphonyd status 2>&1 | jq .SyncInfo.catching_up
```
##### Create Validator
```
symphonyd tx staking create-validator \
--amount 1000000note \
--pubkey $(symphonyd tendermint show-validator) \
--moniker "your-moniker-name" \
--identity "your-keybase-id" \
--details "your-details" \
--website "your-website" \
--security-contact "your-email" \
--chain-id symphony-testnet-2 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--fees 800note \
-y
```

#### Services Management

```
# Reload Service Configuration
sudo systemctl daemon-reload

# Enable Service
sudo systemctl enable symphonyd

# Restart Service
sudo systemctl restart symphonyd

# Check Service Status
sudo systemctl status symphonyd

# Check Service Logs
sudo journalctl -u symphonyd -f --no-hostname -o cat
```
