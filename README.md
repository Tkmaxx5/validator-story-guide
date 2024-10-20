# Installation
## Install dependencies
```
sudo apt update
sudo apt-get update
sudo apt install curl git make jq build-essential gcc unzip wget lz4 aria2 pv -y
```
## Install Go/bin
```
cd $HOME && \
ver="1.22.0" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile && \
go version
```
## Download Story-Geth binary v0.9.3
```
wget https://story-geth-binaries.s3.us-west-1.amazonaws.com/geth-public/geth-linux-amd64-0.9.3-b224fdf.tar.gz
tar -xzvf geth-linux-amd64-0.9.3-b224fdf.tar.gz
[ ! -d "$HOME/go/bin" ] && mkdir -p $HOME/go/bin
if ! grep -q "$HOME/go/bin" $HOME/.bash_profile; then
  echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
fi
sudo cp geth-linux-amd64-0.9.3-b224fdf/geth $HOME/go/bin/story-geth
source $HOME/.bash_profile
story-geth version
```
## Download Story binary v0.10.1
```
wget https://story-geth-binaries.s3.us-west-1.amazonaws.com/story-public/story-linux-amd64-0.10.1-57567e5.tar.gz
tar -xzvf story-linux-amd64-0.10.1-57567e5.tar.gz
[ ! -d "$HOME/go/bin" ] && mkdir -p $HOME/go/bin
if ! grep -q "$HOME/go/bin" $HOME/.bash_profile; then
  echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
fi
cp $HOME/story-linux-amd64-0.10.1-57567e5/story $HOME/go/bin
source $HOME/.bash_profile
story version
```
# Initiate Iliad node
```
story init --network iliad --moniker "Your_moniker_name"
Replace  "Your_moniker_name" 
```
## Create story-geth service file
```
sudo tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth Client
After=network.target
[Service]
User=root
ExecStart=/root/go/bin/story-geth --iliad --syncmode full
Restart=on-failure
RestartSec=3
LimitNOFILE=4096
[Install]
WantedBy=multi-user.target
EOF
```
## Create story service file
```
sudo tee /etc/systemd/system/story.service > /dev/null <<EOF
[Unit]
Description=Story Consensus Client
After=network.target
[Service]
User=root
ExecStart=/root/go/bin/story run
Restart=on-failure
RestartSec=3
LimitNOFILE=4096
[Install]
WantedBy=multi-user.target
EOF
```
# Sync Using Snapshot File
## Install tool
```
sudo apt-get install wget lz4 aria2 pv -y
```
## Stop node
```
sudo systemctl stop story
sudo systemctl stop story-geth
```
## Download Snapshort Story-data
###99Gb
```
cd $HOME
rm -f Story_snapshot.lz4
aria2c -x 16 -s 16 -k 1M https://story.josephtran.co/Story_snapshot.lz4
Waiting for Story data download successful
```
## Download Snapshort Geth-data
###43Gb
```
cd $HOME
rm -f Geth_snapshot.lz4
aria2c -x 16 -s 16 -k 1M https://story.josephtran.co/Geth_snapshot.lz4
Waiting for Geth data download successful
```
## Backup priv_validator_state.json:
```
cp ~/.story/story/data/priv_validator_state.json ~/.story/priv_validator_state.json.backup
```
## Remove old data
```
rm -rf ~/.story/story/data
rm -rf ~/.story/geth/iliad/geth/chaindata
```
## Extract Story-data
```
sudo mkdir -p /root/.story/story/data
lz4 -d -c Story_snapshot.lz4 | pv | sudo tar xv -C ~/.story/story/ > /dev/null
Waiting for successfully extraction.
```
## Extract Geth-data
```
sudo mkdir -p /root/.story/geth/iliad/geth/chaindata
lz4 -d -c Geth_snapshot.lz4 | pv | sudo tar xv -C ~/.story/geth/iliad/geth/ > /dev/null
```
Waiting for successful extraction
## Move priv_validator_state.json back
```
cp ~/.story/priv_validator_state.json.backup ~/.story/story/data/priv_validator_state.json
```
## Restart node
```
sudo systemctl start story
sudo systemctl start story-geth
```
## Reload and start story-geth & Story
```
sudo systemctl daemon-reload && \
sudo systemctl enable story-geth && \
sudo systemctl enable story && \
sudo systemctl start story-geth && \
sudo systemctl start story && \
sudo systemctl status story-geth
```
# Upgrade Node v0.11.0
## Stop node
```
sudo systemctl stop story
```
## Download new binary v0.11.0
```
cd $HOME
wget https://story-geth-binaries.s3.us-west-1.amazonaws.com/story-public/story-linux-amd64-0.11.0-aac4bfe.tar.gz
tar -xzvf story-linux-amd64-0.11.0-aac4bfe.tar.gz
```
## Replace new binary version
```
cp $HOME/story-linux-amd64-0.11.0-aac4bfe/story $(which story)
source $HOME/.bash_profile
story version
```
# UPDATE Story-Geth v0.9.4
## Download Story-Geth v0.9.4
```
wget https://github.com/piplabs/story-geth/releases/download/v0.9.4/geth-linux-amd64
chmod +x geth-linux-amd64
```
## Stop the service
```
sudo systemctl stop story
sudo systemctl stop story-geth
```
## Copy the new binary
```
sudo cp $HOME/geth-linux-amd64 $HOME/go/bin/story-geth
```
## Start the services
```
sudo systemctl start story
sudo systemctl start story-geth
```
# Check logs
```
sudo journalctl -u story -f -o cat
sudo journalctl -u story-geth -f -o cat
```
# Check sync
```
curl -s localhost:26657/status | jq
```
## Check block sync left
```
while true; do
    local_height=$(curl -s localhost:26657/status | jq -r '.result.sync_info.latest_block_height');
    network_height=$(curl -s https://rpc-story.josephtran.xyz/status | jq -r '.result.sync_info.latest_block_height');
    blocks_left=$((network_height - local_height));
    echo -e "\033[1;38mYour node height:\033[0m \033[1;34m$local_height\033[0m | \033[1;35mNetwork height:\033[0m \033[1;36m$network_height\033[0m | \033[1;29mBlocks left:\033[0m \033[1;31m$blocks_left\033[0m";
    sleep 5;
done
```
# Register your Validator
## Export wallet
```
story validator export --export-evm-key
```
## Private key preview
```
cat /root/.story/story/config/private_key.txt
```
## Import your Priavte Key to Metamask
Faucet : https://faucet.story.foundation/
# Validator registering
```
story validator create --stake 10000000000000000000 --private-key "your_private_key"
```
Check the sync the "catching_up" must be 'false' before
## Check your validator info:
```
curl -s localhost:26657/status | jq -r '.result.validator_info' 
```
# Validator Staking
```
story validator stake \
   --validator-pubkey "VALIDATOR_PUB_KEY_IN_BASE64" \
   --stake 1024000000000000000000 \
   --private-key xxxxxxx
```
# Validator Unstakeing
```
story validator unstake \
   --validator-pubkey "VALIDATOR_PUB_KEY_IN_BASE64" \
   --unstake 1024000000000000000000 \
   --private-key xxxxxxxxxx
```
# Delete node (Backup your data, private key, validator key before remove node)
```
sudo systemctl stop story-geth
sudo systemctl stop story
sudo systemctl disable story-geth
sudo systemctl disable story
sudo rm /etc/systemd/system/story-geth.service
sudo rm /etc/systemd/system/story.service
sudo systemctl daemon-reload
sudo rm -rf $HOME/.story
sudo rm $HOME/go/bin/story-geth
sudo rm $HOME/go/bin/story
