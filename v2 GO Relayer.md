# Setup v2 GO Relayer between Stride and GAIA
# install GO
+sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu -y
+ver="1.18.3"
+cd $HOME
+wget "[https://golang.org/dl/go$ver.linux-amd64.tar.gz](https://golang.org/dl/go$ver.linux-amd64.tar.gz)"
+sudo rm -rf /usr/local/go
+sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
+rm "go$ver.linux-amd64.tar.gz"
+echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
+source ~/.bash_profile
+go version

# install RUST

curl [https://sh.rustup.rs/](https://sh.rustup.rs/) -sSf | sh
source $HOME/.cargo/env
rustc --version

# Set indexer to kv on each chain

sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.stride/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"kv\"/" $HOME/.gaia/config/config.toml

# Expose your RPC endpoint to public, then Hermes can be reached (If Hermes and your fullnodes are on same vps, no need to do it)

sed -i.bak -e 's|^laddr = \"tcp:\/\/.*:\([0-9].*\)57\"|laddr = \"tcp:\/\/0\.0\.0\.0:\157\"|' $HOME/.stride/config/config.toml
sed -i.bak -e 's|^laddr = \"tcp:\/\/.*:\([0-9].*\)57\"|laddr = \"tcp:\/\/0\.0\.0\.0:\157\"|' $HOME/.gaia/config/config.toml

Restart nodes after changing in step 3 & 4

# Install v2 GO Relayer

git clone [https://github.com/cosmos/relayer.git](https://github.com/cosmos/relayer.git)
cd relayer/
git checkout v2.0.0-rc4
make install
which rly
chmod +x $(which rly)

# Setup v2 GO Relayer between Stride and GAIA

cd ~ && rly config init --memo "DISDCORD-ID"
rly config show

# Set variable
SRC_CHAIN="STRIDE-TESTNET-2"
SRC_KEY="stride-rly"
SRC_MNEMONIC_PHRASE="24 PHRASE"

DST_CHAIN="GAIA"
DST_KEY="gaia-rly"
DST_MNEMONIC_PHRASE="24 PHRASE"

# Make config data

cd $HOME/.relayer/config
cp config.yaml config.yaml-bak

sudo tee $HOME/.relayer/config/config.yaml > /dev/null <<EOF
global:
api-listen-addr: :5183
timeout: 10s
memo: $DISDCORD-ID
light-cache-size: 20
chains:
GAIA:
type: cosmos
value:
key: $DST_KEY
chain-id: $DST_CHAIN
rpc-addr: http://154.12.229.177:26957
grpc-addr: http://154.12.229.177:9490
account-prefix: cosmos
keyring-backend: test
gas-adjustment: 1.2
gas-prices: 0.0025uatom
debug: true
timeout: 300s
output-format: json
sign-mode: direct
memo-prefix: $DISDCORD-ID
STRIDE-TESTNET-2:
type: cosmos
value:
key: $SRC_KEY
chain-id: $SRC_CHAIN
rpc-addr: http://207.180.226.203:26957
grpc-addr: http://207.180.226.203:9490
account-prefix: stride
keyring-backend: test
gas-adjustment: 1.2
gas-prices: 0.0025ustrd
debug: true
timeout: 300s
output-format: json
sign-mode: direct
memo-prefix: $DISCORD-ID
paths:
STRIDE-GAIA:
src:
chain-id: GAIA
client-id: 07-tendermint-0
connection-id: connection-0
dst:
chain-id: STRIDE-TESTNET-2
client-id: 07-tendermint-0
connection-id: connection-0
src-channel-filter:
rule: "allowlist"
channel-list: [channel-0, channel-1, channel-2, channel-3, channel-4]
EOF

# IMPORT WALLET

### Import keys into GO Relayer

rly keys restore $SRC_CHAIN $SRC_KEY "$SRC_MNEMONIC_PHRASE"
rly keys restore $DST_CHAIN $DST_KEY "$DST_MNEMONIC_PHRASE"

rly q balance $SRC_CHAIN
rly q balance $DST_CHAIN

# Make systemd service and run

sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF
[Unit]
Description=V2 Go relayer
After=network-online.target

[Service]
User=$USER
ExecStart=$(which rly) start STRIDE-GAIA --memo "DISDCORD-ID"
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl restart rlyd

#check log :
sudo journalctl -fu rlyd -o cat

# Try to send raw data between 2 relayers

### STRIDE to GAIA

rly transact transfer src_chain_name dst_chain_name amount dst_addr src_channel_id [flags]

rly transact transfer STRIDE-TESTNET-2 GAIA 1000ustrd $(rly chains address GAIA) channel-0 --path STRIDE-GAIA

2022-08-07T02:59:31.061938Z info Successful transaction {"provider_type": "cosmos", "chain_id": "STRIDE-TESTNET-2", "packet_src_channel": "channel-0", "packet_dst_channel": "channel-0", "gas_used": 88277, "fees": "234ustrd", "fee_payer": "stride1fhqd9s7usrk79gp5skuj0j6hlwck7muns7smg0", "height": 117723, "msg_types": ["/ibc.applications.transfer.v1.MsgTransfer"], "tx_hash": "692EFF298E5D30AD8CF0525396DA34375613F62958F9456261D85FD793AD9664"}

### GAIA to STRIDE

rly transact transfer GAIA STRIDE-TESTNET-2 100uatom $(rly chains address STRIDE-TESTNET-2) channel-0 --path STRIDE-GAIA

2022-08-07T03:00:30.663366Z info Successful transaction {"provider_type": "cosmos", "chain_id": "GAIA", "packet_src_channel": "channel-0", "packet_dst_channel": "channel-0", "gas_used": 88769, "fees": "234uatom", "fee_payer": "cosmos1sn93drjt2sms348pxpm2zyd2pkkcxnhp7fplk9", "height": 190333, "msg_types": ["/ibc.applications.transfer.v1.MsgTransfer"], "tx_hash": "50D12FA71BFDFC58D11C3716A8DB8D0986D65E80CA9CA24C57469081EDDD3D90"}

# To explorer check Update Client for GO-Relayer --> ok

****Thanks to VienNguyen#4040 for the support for this guide.****
