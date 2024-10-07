Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.20.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**download binaries**
```
cd $HOME
rm -rf bin
mkdir bin
cd bin
wget https://story-geth-binaries.s3.us-west-1.amazonaws.com/geth-public/geth-linux-amd64-0.9.2-ea9f0d2.tar.gz
tar -xvf geth-linux-amd64-0.9.2-ea9f0d2.tar.gz
mv ~/bin/geth-linux-amd64-0.9.2-ea9f0d2/geth ~/go/bin/
mkdir -p ~/.story/story
mkdir -p ~/.story/geth
```

**Install Story**
```
cd $HOME
rm -rf story
git clone https://github.com/piplabs/story
cd story
git checkout v0.10.0
go build -o story ./client
sudo mv ~/story/story ~/go/bin/
```

**initialize the story client**
```
story init --moniker test --network iliad
```

**create geth servie file**
```
sudo tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$(which geth) --iliad --syncmode full --http --http.api eth,net,web3,engine --http.vhosts '*' --http.addr 0.0.0.0 --http.port 8545 --ws --ws.api eth,web3,net,txpool --ws.addr 0.0.0.0 --ws.port 8546
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

**create story service file**
sudo tee /etc/systemd/system/story.service > /dev/null <<EOF
[Unit]
Description=Story Service
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/.story/story
ExecStart=$(which story) run

Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

**enable and start geth**
```
sudo systemctl daemon-reload
sudo systemctl enable story-geth
sudo systemctl restart story-geth && sudo journalctl -u story-geth -f
```

**enable and start story**
```
sudo systemctl daemon-reload
sudo systemctl enable story
sudo systemctl restart story && sudo journalctl -u story -f
```

**Node Sync Status Checker**
```
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.story/story/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://story-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mYour Node Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
```

**Create validator**
```
cat $HOME/.story/story/config/private_key.txt
```
Use this private key to import your account into a wallet, e.g. Metamask or Phantom. Add the Iliad testnet to your wallet via faucet. Then, copy your 'EVM address' from the wallet and request $IP tokens. Now you can see the balance and make transactions in the wallet app.

**Before creating a validator, wait for your node to get fully synced. Once "catching_up" is "false", move on to the next step**
```
curl localhost:$(sed -n '/\[rpc\]/,/laddr/ { /laddr/ {s/.*://; s/".*//; p} }' $HOME/.story/story/config/config.toml)/status | jq
```

**Create validator**
```
story validator create --stake 1000000000000000000 --private-key $(cat $HOME/.story/story/config/private_key.txt | grep "PRIVATE_KEY" | awk -F'=' '{print $2}')
```

**Remember to backup your validator priv_key from here**
```
cat $HOME/.story/story/config/priv_validator_key.json
Firewall rules
Configure firewall rules:
```

sudo ufw allow 30303/tcp comment geth_p2p_port
sudo ufw allow 26656/tcp comment story_p2p_port
Delete node
sudo systemctl stop story story-geth
sudo systemctl disable story story-geth
rm -rf $HOME/.story
sudo rm /etc/systemd/system/story.service /etc/systemd/system/story-geth.service
sudo systemctl daemon-reload
