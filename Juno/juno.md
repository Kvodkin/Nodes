## JUNO.
Packages
```bash
sudo apt update && sudo apt upgrade -y
```
## Dependencies.
```bash
sudo apt install curl build-essential git wget jq make gcc tmux chrony -y
```
## Go.

```bash
ver="1.19.1" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```
## Moniker, whrite something cool.
```bash
NODENAME=Do_not_copypaste
```
## Save and import variables.

```bash
JUNO_PORT=34
echo "export NODENAME=$NODENAME" >> $HOME/.bash_profile
if [ ! $WALLET ]; then
	echo "export WALLET=wallet" >> $HOME/.bash_profile
fi
echo "export JUNO_CHAIN_ID=juno-1" >> $HOME/.bash_profile
echo "export JUNO_PORT=${JUNO_PORT}" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
## Binaries
```bash
cd $HOME
git clone https://github.com/CosmosContracts/juno juno
cd juno
git checkout v15.0.0
make install
```

## Config app

```bash
junod config chain-id $JUNO_CHAIN_ID
junod config node tcp://localhost:${JUNO_PORT}657
```
Init app

```bash
junod init $NODENAME --chain-id $JUNO_CHAIN_ID
```

## Addrbook

```
wget -O addrbook.json https://snapshots.polkachu.com/addrbook/juno/addrbook.json --inet4-only
mv addrbook.json ~/.juno/config
```
## Genesis

```bash
wget -O genesis.json https://snapshots.polkachu.com/genesis/juno/genesis.json --inet4-only
mv genesis.json ~/.juno/config
```
## Custom ports 

```bash
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${JUNO_PORT}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${JUNO_PORT}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${JUNO_PORT}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${JUNO_PORT}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${JUNO_PORT}660\"%" $HOME/.juno/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${JUNO_PORT}317\"%; s%^address = \":8080\"%address = \":${JUNO_PORT}080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${JUNO_PORT}090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${JUNO_PORT}091\"%" $HOME/.juno/config/app.toml
```
## Pruning

```bash
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.juno/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.juno/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.juno/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.juno/config/app.toml
```
## Indexing 
```bash
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.juno/config/config.toml
```

## Minimum gas price

```bash
sed -E -i 's/minimum-gas-prices = \".*\"/minimum-gas-prices = \"0.0025ujunox\"/' $HOME/.juno/config/app.toml
```
Reset chain data

```bash
junod tendermint unsafe-reset-all --home $HOME/.juno --keep-addr-book
```
## State-sync.
```
SNAP_RPC="https://juno-rpc.polkachu.com:443"
```
```
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 500)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
```
```
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.juno/config/config.toml
```

## Service.

```bash
sudo tee /etc/systemd/system/junod.service > /dev/null <<EOF
[Unit]
Description=Stake to Kolot
After=network-online.target

[Service]
User=$USER
ExecStart=$(which junod) start --home $HOME/.juno
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
## Register and start service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable junod
sudo systemctl restart junod && sudo journalctl -u junod -f -o cat
```
