#  ICQ between GAIA and Stride

# Download and compile to binary file
      cd $HOME
      wget https://github.com/Stride-Labs/interchain-queries/archive/refs/tags/v0.0.4.zip
      unzip v0.0.4.zip
      cd interchain-queries-0.0.4/
      go build
      chmod +x interchain-queries
      mv interchain-queries /usr/local/bin/`

# Make configuration file for ICQ

        cd $HOME && mkdir .icq
  
        sudo tee $HOME/.icq/config.yaml > /dev/null <<EOF
        default_chain: STRIDE-TESTNET-2
     chains:
       GAIA:
         key: <your-key-name-Gaia>
         chain-id: GAIA
         rpc-addr: http://154.12.229.177:26957
         grpc-addr: http://154.12.229.177:9490
         account-prefix: cosmos
         keyring-backend: test
         gas-adjustment: 1.2
         gas-prices: 1uatom
         key-directory: /root/.icq/keys
         debug: false
         timeout: 20s
         block-timeout: ""
         output-format: json
         sign-mode: direct
       STRIDE-TESTNET-2:
         key: <your-key-name-Stride>
         chain-id: STRIDE-TESTNET-2
         rpc-addr: http://localhost:26657
         grpc-addr: http://localhost:9090
         account-prefix: stride
         keyring-backend: test
         gas-adjustment: 1.2
         gas-prices: 1ustrd
         key-directory: /root/.icq/keys
         debug: false
         timeout: 20s
         block-timeout: ""
         output-format: json
         sign-mode: direct
     cl: {}
     EOF

***Edit RPC,GPRC and Port if you using your own GAIA/STRIDE nodes***

# Create v2 GO relayer as below link
  . V2 GO relayer https://github.com/thonguyen14/Stride/blob/ICQ/v2%20GO%20Relayer
# Import these wallets of STRIDE and GAIA chains which are being used for V2 GO Relayer

     interchain-queries keys restore --chain GAIA <your-key-name-Gaia>
     interchain-queries keys restore --chain STRIDE-TESTNET-2 <your-key-name-Stride>

# Create systemd service for ICQ

     sudo tee /etc/systemd/system/icqd.service > /dev/null <<EOF
     [Unit]
     Description=ICQ 
     After=network-online.target

     [Service]
     User=$USER
     ExecStart=$(which interchain-queries) run
     Restart=on-failure
     RestartSec=3
     LimitNOFILE=65535

     [Install]
     WantedBy=multi-user.target
     EOF

     sudo systemctl daemon-reload
     sudo systemctl enable icqd
     sudo systemctl restart icqd

# Monitor journal log and wait a moment

     journalctl -u icqd -f -o cat

- Output will be as below

     Started V2 Go relayer.
     store/bank/key
     height parsed from GetHeightFromMetadata= 0
     store/bank/key
     height parsed from GetHeightFromMetadata= 0
     Fetching client update for height height 164764
     Fetching client update for height height 164764
     Requerying lightblock
     Requerying lightblock
     ICQ RELAYER | query.Height= 0
     ICQ RELAYER | res.Height= 164763
     ICQ RELAYER | query.Height= 0
     ICQ RELAYER | res.Height= 164763
     Send batch of 4 messages
     1 ClientUpdate message
     1 SubmitResponse message
     1 ClientUpdate message
     1 SubmitResponse message
     Sent batch of 2 (deduplicated) messages
# To explorer check txhash 

*****Thanks to VienNguyen#4040 for the support for this guide****


