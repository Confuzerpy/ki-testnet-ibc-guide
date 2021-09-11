##  _Kichain ibc relayer guide_
- Special thanks to [Northa](https://github.com/Northa) and [goooodnes](https://github.com/goooodnes) from [Let's encrypt community](https://t.me/kichain_ru)

<details>
  <summary>Requirements:</summary>
  
4cpu, 4gb ram, ssd > 80Gb

ubuntu 20.04 lts

[go version go1.16.5](https://dl.google.com/go/go1.16.5.linux-amd64.tar.gz)

[relayer v0.9.3](https://github.com/cosmos/relayer)

[kichain-t-4 node](https://github.com/KiFoundation/ki-networks/tree/v0.1/Testnet/kichain-t-4)

[testnet-croeseid-4 node](https://crypto.org/docs/getting-started/croeseid-testnet.html#step-1-get-the-crypto-org-chain-testnet-binary)
  
</details>

In this guide, we will examine how to setup ibc relayer between [kichain-t-4](https://github.com/KiFoundation/ki-testnet-challenge) and [testnet-croeseid-4](https://crypto.org/docs/getting-started/croeseid-testnet.html#step-1-get-the-crypto-org-chain-testnet-binary)


#### 1. Install ibc relayer from the [official repo](https://github.com/cosmos/relayer)
Check the relayer version. In my case it was 0.9.3 version

```sh 
rly version
version: v0.9.3
commit: 4b81fa59055e3e94520bdfae1debe2fe0b747dc1
cosmos-sdk: v0.42.4
go: go1.15.11 linux/amd64
```

#### 2. Intialize relayer
```
rly config init
```
#### 3. Once initialized lets configure it:
```
cd && mkdir -p ./relayer/kichain && cd ./relayer/kichain
```
Create config for the kichain-t-4 network
```
tee ki_config.json > /dev/null <<EOF
{
"chain-id": "kichain-t-4",
  "rpc-addr": "http://127.0.0.1:26657",
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
EOF
```
Create config for the croeseid testnet network
- _note in my case i configured my croeseid testnet to operate with port 26552. Your port might be different!_
```
tee cro_config.json > /dev/null <<EOF
{
  "chain-id": "testnet-croeseid-4",
  "rpc-addr": "http://127.0.0.1:26652",
  "account-prefix": "tcro",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025basetcro",
  "trusting-period": "48h"
}
EOF
```

#### 4. Next step we should add our configs to the relayer

```rly chains add -f ki_config.json```

```rly chains add -f cro_config.```
#### 5. Generate/import wallets.
- _Notice in my case im using key name ki_test and cro_test.
You can specify watever you want name._

###### Generating new keys
```rly keys add kichain-t-4 ki_test```

```rly keys add testnet-croeseid-4 --coin-type 1 cro_test```
	
#### WARNING! 
- We using ```--coin-type 1``` when generating croeseid wallet
  because croesseid using DIFFERENT derivation path!

###### Import existing keys
```rly keys restore kichain-t-4 ki_test "YOUR MNEMONIC"```

```rly keys restore "testnet-croeseid-4" cro_test --coin-type 1 "YOUR MNEMONIC"```



#### 6. Checking keys

```sh
rly keys list kichain-t-4

key(0): ki_test -> tki1__YOUR_WALLET
```
```sh
rly keys list testnet-croeseid-4

key(0): cro_test -> tcro__YOUR_WALLET
```
#### 7. Add the newly created keys to the config of the relayer:

```rly chains edit kichain-t-4 key ki_test```

```rly chains edit testnet-croeseid-4 key cro_test```

#### 8. Changing timeout in the relayer config

```nano ~/.relayer/config/config.yaml```

Find the line ```timeout: 10s``` and replace to ```timeout: 10m```

#### 9. Make sure you have enough funds in you wallets.
- If you dont have funds. Request some via faucet.

[KI foundation](https://discord.gg/DSSUC7Tt) under testnet-challenge thread

[Croeseid faucet](https://crypto.org/faucet)

#### 10. Check the wallet balance

```sh
user@15102:~# rly q balance kichain-t-4
100000000utki
```
```sh
user@15102:~# rly q balance testnet-croeseid-4
100000000basetcro
```

#### 11. Initialize the clients:
```rly light init kichain-t-4 -f```

```rly light init testnet-croeseid-4 -f```

#### 12. Check that everything is OK at this point

```
rly chains list -d
 0: kichain-t-4          -> key(✔) bal(✔) light(✔) path(✔)
 1: testnet-croeseid-4   -> key(✔) bal(✔) light(✔) path(✔)
```

#### 13. Create a path between the two networks:
- ###### rly paths generate [src-chain-id] [dst-chain-id] [name] [flags]

```rly paths generate kichain-t-4 testnet-croeseid-4 transfer --port=transfer```

If you have some issues try [Troubleshooting guide](#help)


#### 14. Create channel
Note this command might take some time. If you have some errors try to repeat several times.

```rly tx link transfer```

If operation completes successfull the output of the last line should be like:

```★ Channel created: [kichain-t-4]chan{channel-45}port{transfer} -> [testnet-croeseid-4]chan{channel-18}port{transfer}```

#### 14. Checking our channel 

```
rly paths list -d
0: transfer          -> chns(✔) clnts(✔) conn(✔) chan(✔) (kichain-t-4:transfer<>testnet-croeseid-4:transfer)
```

#### 15. Start relayer as a service
```
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF [Unit] Description=relayer client After=network-online.target, kichaind.service [Service] User=$USER ExecStart=$(which rly) start transfer Restart=always RestartSec=3 LimitNOFILE=65535 [Install] WantedBy=multi-user.target EOF
```
Start rlyd daemon
```
sudo systemctl daemon-reload
sudo systemctl enable rlyd
sudo systemctl start rlyd
```

Logs can be checked: ```journalctl -f -u rlyd ```

#### 16. Make some transactions

From croeseid to kichain

```rly tx transfer testnet-croeseid-4 kichain-t-4 1000000basetcro tki1__YOUR_WALLET --path transfer```

The output should be:
###### ✔ [testnet-croeseid-4]@{289697} - msg(0:transfer) hash(1C1....your_hash....07)


From kichain to croeseid

```rly tx transfer kichain-t-4 testnet-croeseid-4 666000utki tcro__YOUR_WALLET --path transfer```

The output should be:
###### ✔ [kichain-t-4]@{221552} - msg(0:transfer) hash(E1....your_hash....6)

Example of txs from kichain to croeseid:

[B5120764919AC7C5F7B4FC03DE0F82C4A33D647402D60A5BD672B092A9110572](https://ki.thecodes.dev/tx/B5120764919AC7C5F7B4FC03DE0F82C4A33D647402D60A5BD672B092A9110572)
[E388D4F4D47707D24947B9275178A850A7CFF068FF05D95205626E84A2418CBF](https://ki.thecodes.dev/tx/E388D4F4D47707D24947B9275178A850A7CFF068FF05D95205626E84A2418CBF)
[CC909D79A86119D689C894AC41DEA28E2FFEB2AE2C6FF7D32F81FA7262C96889](https://ki.thecodes.dev/tx/CC909D79A86119D689C894AC41DEA28E2FFEB2AE2C6FF7D32F81FA7262C96889)

Example of txs from croeseid to kichain:

[280F26EB11925961944ABCDEC14DEBFA87CD7502B915E67AF7B94C1CC496E4F2](https://ki.thecodes.dev/tx/280F26EB11925961944ABCDEC14DEBFA87CD7502B915E67AF7B94C1CC496E4F2)
[280F26EB11925961944ABCDEC14DEBFA87CD7502B915E67AF7B94C1CC496E4F2](https://ki.thecodes.dev/tx/280F26EB11925961944ABCDEC14DEBFA87CD7502B915E67AF7B94C1CC496E4F2)
[707286841BA7330A4AF0EDC550C8F225268DD6EB91E393F12CCDD85C6607A34E](https://ki.thecodes.dev/tx/707286841BA7330A4AF0EDC550C8F225268DD6EB91E393F12CCDD85C6607A34E)

## Help
<details>
  <summary>Troubleshooting guide</summary>
<details>
<summary>Error: no concrete type registered for type URL </summary>
in this case we have to create paths manually. Open relayer config.yaml

```nano ~/.relayer/config/config.yaml```

navigate to paths line. delete "paths{}" and paste the folowing code instead

``` 
paths:
  transfer:
    src:
      chain-id: kichain-t-4
      port-id: transfer
      order: UNORDERED
      version: ics20-1
    dst:
      chain-id: testnet-croeseid-4
      port-id: transfer
      order: UNORDERED
      version: ics20-1
    strategy:
      type: naive
```

</details>
<details>
<summary>Error: failed to get trusted header</summary>
Try to update client manually
	
```rly tx update-clients transfer```
	
If this doesn't help you have to re-generate a new path
</details>
  
<details>
<summary>Error: more than one transaction returned with query</summary>
Check if you have unrelayed packets
	
```rly q unrelayed transfer```
	
If you have some try
	
```rly tx rly transfer```

If that doesn't work, then create a new path
</details>
<details>
<summary>Error: no transactions returned with query</summary>
Check if you have unrelayed packets
	
```rly q unrelayed transfer```
	
If you have some try
	
```rly tx rly transfer```

If that doesn't work, then create a new path

</details>

</details>



## Links:
[KI foundation discord](https://discord.gg/DSSUC7Tt)

[Kichain testnet explorer 1](https://kichain-t-3.blockchain.ki/)

[Kichain testnet explorer 2](https://ki.thecodes.dev/)

[Croeseid testnet explorer](https://crypto.org/explorer/croeseid4/)
