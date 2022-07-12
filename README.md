# PalomaPigeonSoftwareSetUp
Paloma Pigeon Software Set Up Guide

# Paloma Testnet-6 Pigeon Relayer Software Set Up Guide

![Logo!](assets/paloma.png)

# Pigeon

> A Golang cross-chain message relayer system for Paloma validators to deliver messages to any blockchain.

For Crosschain software engineers that want simultaneous control of mulitiple smart contracts, on any blockchain, Paloma is decentralized and consensus-driven message delivery, fast state awareness, low cost state computation, and powerful attestation system that enables scaleable, crosschain, smart contract execution with any data source.

## Opening an Alchemy Account
* Yükleme işlemine geçmeden önce [Alchemy](https://alchemy.com/?r=zc3NjI5NzM1NzMxN)'den bir hesap oluşturup ETH Mainnet App oluşturuyoruz. Burada `View Key` bölümünden `https` ile başlayan linkimizi alıyoruz ve kurulum sırasında Alchemy linki geçen yerde kullanmak üzere bir txt dosyasına kaydediyoruz.

## Pigeon Install

```shell
wget -O - https://github.com/palomachain/pigeon/releases/download/v0.2.5-alpha/pigeon_0.2.5-alpha_Linux_x86_64v3.tar.gz | \
tar -C /usr/local/bin -xvzf - pigeon
chmod +x /usr/local/bin/pigeon
mkdir ~/.pigeon
```

## EVM (ETH) Wallet

### Create New Wallet
* Do not forget your password and mnemonics, save them.
```
pigeon evm keys generate-new ~/.pigeon/keys/evm/eth-main
```

### Importing Existing Wallet
* If you have menemonics belonging to your wallet, import your wallet with this code.
* I will import my ETH address that I gave in the Paloma form.

```
pigeon evm keys import ~/.pigeon/keys/evm/eth-main
```
### Loading the Variables
* Friends, we enter the information written below to the relevant places in the code below. After this stage, no changes will be made to any code unless otherwise stated.
	* '$VALIDATOR' your validator name
	* '$WALLET' your paloma wallet name 
	* '$ETH_RPC_URL' your alchemy link, starting with https
	* '$ETH_SIGNING_KEY' When you create a wallet, the code consisting of trailing numbers in the output
	* '$PALOMA_KEYRING_PASS' your paloma wallet pass
	* '$ETH_PASSWORD' ETH wallet pass
```shell
echo 'export VALIDATOR='$VALIDATOR >> $HOME/.bash_profile
echo 'export ETH_RPC_URL='$ETH_RPC_URL >> $HOME/.bash_profile
echo 'export ETH_SIGNING_KEY='$ETH_SIGNING_KEY >> $HOME/.bash_profile
echo 'export PALOMA_KEYRING_PASS='$PALOMA_KEYRING_PASS >> $HOME/.bash_profile
echo 'export ETH_PASSWORD='$ETH_PASSWORD >> $HOME/.bash_profile
echo 'export WALLET='$WALLET >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Config Setup
* Here we import our Paloma wallet.
```shell
palomad keys add "$VALIDATOR" --recover
```

* If your wallet name is different from your validator name, we enter this code instead of the code above.
```shell
palomad keys add "$WALLET" --recover 
```

Yukarıdaki kodu girdiğinizde şifrenizi girdiktensonra şöyle bir çıktı alacaksınız `override the existing name VALIDATOR_ADINIZ [y/N]:` buna yes yani y diyerek devam ediyoruz. Ardından sizden > `Enter your bip39 mnemonic` cüzdanınıza ait menemonicleri isteyecek onları yazıp işleme devam ediyoruz.

### Set The VALIDATOR env Variable

```shell
export VALIDATOR="$(palomad keys list --list-names | head -n1)"
```

### Create Configuration File
We create config.yaml file in the directory specified here `~/.pigeon/config.yaml` .
!! Before doing this, we look at what is written in the `keyring-backend` section in the `~/.paloma/config/client.toml` file.
if it says `keyring-backend = "os"` then we also make the keyring-type part `os` below, like this `keyring-type: os`
if it says `keyring-backend = "test"` then we also `test` the `keyring-type` part below, like this `keyring-type: test`

You can access this file by typing `cat .paloma/config/client.toml` from the terminal and see what is written in the `keyring-backend` section in the output.
We are not making any changes to the code below!

```yaml
cat <<EOT >~/.pigeon/config.yaml

loop-timeout: 5s

paloma:
  chain-id: paloma-testnet-6
  call-timeout: 20s
  keyring-dir: ~/.paloma
  keyring-pass-env-name: PALOMA_KEYRING_PASS
  keyring-type: os
  signing-key: ${VALIDATOR}
  base-rpc-url: http://localhost:26657
  gas-adjustment: 1.5
  gas-prices: 0.001ugrain
  account-prefix: paloma

evm:
  eth-main:
    chain-id: 1
    base-rpc-url: ${ETH_RPC_URL}
    keyring-pass-env-name: ETH_PASSWORD
    signing-key: ${ETH_SIGNING_KEY}
    keyring-dir: ~/.pigeon/keys/evm/eth-main
EOT
```

### Start Pigeon
Our first pigeon needs several keys, for this we create the 'env.sh' file.

```shell
cat <<EOT >~/.pigeon/env.sh
PALOMA_KEYRING_PASS=${PALOMA_KEYRING_PASS}
ETH_RPC_URL=${ETH_RPC_URL}
ETH_PASSWORD=${ETH_PASSWORD}
ETH_SIGNING_KEY=${ETH_SIGNING_KEY}
VALIDATOR=${VALIDATOR}
EOT
```

### Run Pigeon

```shell
source ~/.pigeon/env.sh
pigeon start
```

#### Creating a Service File

```shell
cat <<EOT >/etc/systemd/system/pigeond.service
[Unit]
Description=Pigeon daemon
After=network-online.target
ConditionPathExists=/usr/local/bin/pigeon

[Service]
Type=simple
Restart=always
RestartSec=5
User=${USER}
WorkingDirectory=~
EnvironmentFile=${HOME}/.pigeon/env.sh
ExecStart=/usr/local/bin/pigeon start
ExecReload=

[Install]
WantedBy=multi-user.target
EOT
```

### Restar Pigeon

```shell
systemctl daemon-reload
systemctl enable pigeond
systemctl restart pigeond
service pigeond start
```

#### Check Service Status
```shell
service pigeond status
```

#### Getting ChainInfos Information
`palomavaloper` yazan yere palomavaloper ile başlayan adresimizi yazıyoruz.
```shell
palomad q valset validator-info palomavaloper
```

#### Watch Logs:
```shell
journalctl -u pigeond.service -f -n 100
```

### Definitions and Descriptions of Pigeons Variables

  - for paloma key:
	- keyring-dir
      - right now it's not really super important where this points. The important things for the future is that pigeon needs to send transactions to Paloma using it's validator (operator) key!
	  - it's best to leave it as is
	- keyring-pass-env-name
	  - this one is super important!
	  - it is the name of the ENV variable where password to unlock the keyring is stored!
	  - you are not writing password here!! You are writing the ENV variable's name where the password is stored.
	  - you should obviously use a bit more advanced method than shown here, but here is the example:
	    - if the `keyring-pass-env-name` is set to `MY_SUPER_SECRET_PASS` then you should provide ENV variable `MY_SUPER_SECRET_PASS` and store the password there
	    - e.g. `MY_SUPER_SECRET_PASS=abcd pigeon start`
	- keyring-type
	  - it should be the same as it's defined for paloma's client. Look under the ~/.paloma/config/client.toml
	- signing-key
	  - right now it's again not important which key we are using. It can be any key that has enough balance to submit TXs to Paloma. It's best to use the same key that's set up for the validator.
	- gas-adustment:
	  - gas multiplier. The pigeon will estimate the gas to run a TX and then it will multiply it with gas-adjustment (if it's a positive number)
 - for evm -> eth-main:
	- keyring-pass-env-name: as as above for paloma.
	- signing-key
	  - address of the key from the keyring used to sign and send TXs to EVM network (one that you got when running `pigeon evm keys generate-new` from the install section)
	- keyring-dir:
	  - a directory where keys to communicate with the EVM network is stored


