# HOW TO BECOME REPUBLIC VALIDATOR #

***Install dependencies Required***
```
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

***Install Go***
- If you already have GO installed you can skip this
```
ver="1.22.5"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
source ~/.bash_profile
go version
```

***Install binary***
```
wget https://github.com/RepublicAI/networks/raw/refs/heads/main/testnet/releases/v0.1.0/republicd-linux-amd64 -O republicd
chmod +x republicd
mv republicd $HOME/go/bin/
```
- Verify that you’ve installed successfully
```
republicd version  --long | grep -e version -e commit
```

***Initialize Node***
- Please change MONIKER To your own Moniker.
```
republic init MONIKER --chain-id raitestnet_77701-1
```

***Download Genesis file and addrbook***
- Genesis
```
curl -L https://snapshot-t.vinjan-inc.com/republic/genesis.json > $HOME/.republic/config/genesis.json
```
- Addrbook
```
curl -L https://snapshot-t.vinjan-inc.com/republic/addrbook.json > $HOME/.republic/config/addrbook.jso
```

***Custom Port***
```
PORT=133
sed -i -e "s%:26657%:${PORT}57%" $HOME/.republic/config/client.toml
sed -i -e "s%:26658%:${PORT}58%; s%:26657%:${PORT}57%; s%:6060%:${PORT}60%; s%:26656%:${PORT}56%; s%:26660%:${PORT}60%" $HOME/.republic/config/config.toml
sed -i -e "s%:1317%:${PORT}17%; s%:9090%:${PORT}90%; s%:8545%:${PORT}45%; s%:8546%:${PORT}46%; s%:6065%:${PORT}65%" $HOME/.republic/config/app.toml
```

***Configure Gas***
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"2500000000arai\"/" $HOME/.republic/config/app.toml
```

***Prunning***
```
sed -i \
-e 's|^pruning *=.*|pruning = "custom"|' \
-e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
-e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
-e 's|^pruning-interval *=.*|pruning-interval = "20"|' \
$HOME/.republic/config/app.toml
```

***Indexer Off***
```
sed -i 's|^indexer *=.*|indexer = "null"|' $HOME/.republic/config/config.toml
```

***Create service file and start node***
```
sudo tee /etc/systemd/system/republicd.service > /dev/null <<EOF
[Unit]
Description=republic
After=network-online.target

[Service]
User=$USER
ExecStart=$(which republicd) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

***Enable Service and Start Node***
```
sudo systemctl daemon-reload
sudo systemctl enable republicd
sudo systemctl restart republicd
sudo journalctl -u republicd -f -o cat
```

***Use Statesync***
```
sudo systemctl stop republicd
cp $HOME/.republic/data/priv_validator_state.json $HOME/.republic/priv_validator_state.json.backup
republicd comet unsafe-reset-all --home $HOME/.republic --keep-addr-book
```

```
SNAP_RPC="https://rpc-t.republic.vinjan-inc.com:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" "$HOME/.republicd/config/config.toml"
PEERS="a5d2fe7d932c3b6f7c9633164f102315d1f575c6@195.201.160.23:13356"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" "$HOME/.republic/config/config.toml"
```

```
sudo systemctl restart republicd
sudo journalctl -u republicd -f -o cat
```

***WAIT FOR SYNC***
```
republicd status 2>&1 | jq .sync_info
```
- Wait until "catching_up": false then u can continue to next step


***Create Wallet***
```
republicd keys add wallet
```
- Create Password
- Dont forget to BACKUP your Adress, Pubkey, Phrase

***Check Wallet Balance***
```
republicd q bank balances $(republicd keys show wallet -a)
```

***Validator Management***
- Check your Validator Pubkey
```
republicd comet show-validator
```

- Make File validator.json
```
nano $HOME/.republic/validator.json
```

```
{
  "pubkey":  ,
  "amount": "1000000000000000000arai",
  "moniker": "",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.05",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.05",
  "min-self-delegation": "1"
}
```
- Pubkey fill with your Validator Pubkey
- Identity u can use from keybase.io
- The moniker that was created earlier
- U can empty web, security, details
- Save

***Create Validator***
```
republicd tx staking create-validator $HOME/.republic/validator.json \
--from wallet \
--chain-id raitestnet_77701-1 \
--gas-prices=2500000000arai \
--gas-adjustment=1.5 \
--gas=auto
```

***Delete Node***

WARNING! Use this command wisely Backup your key first it will remove Defund from your system
```
sudo systemctl stop republicd
sudo systemctl disable republicd
sudo rm /etc/systemd/system/republicd.service
sudo systemctl daemon-reload
rm -f $(which republicd)
rm -rf .republic
```

***Special thanks to Vinjan Inc. for permission to share this tutorial***

Source : https://vinjan-inc.com/
