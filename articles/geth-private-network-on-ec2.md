---
title: "geth で EC2 にプライベートネットワークを構築"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["geth", "ethereum", "blockchain"]
published: true
---

# 概要

EC2 Linux インスタンスを利用して geth プライベートネットワークを構築する手順です。

昨年開発して現在も稼働しているサービスの構築手順なので、情報が古い可能性がある旨、ご了承ください。

なお、AWS 側の設定は割愛します。

# 環境

- geth 1.10.18 (当時)
- EC2 / Amazon Linux / c3.xlarge

EC2 インスタンスは多少のスペックがないと厳しいです。

# 手順

## sudoer としてユーザを追加する

```sh
% ssh <EC2 インスタンス作成時のユーザー>
```

```sh
$ sudo useradd -d "/home/<username>" -s "/bin/bash" <username>
$ sudo passwd -d <username>
$ sudo vi /etc/sudoers.d/<username>
```

```sh
<username> ALL=(ALL) NOPASSWD:ALL
```

SSH 鍵は任意のものを用いてください。

```sh
$ sudo cp -r /home/ec2-user/.ssh /home/<username>/
$ sudo chown -R <username>:<username> /home/<username>/.ssh
$ rm /home/ec2-user/.ssh/authorized_keys
$ exit
```

## プライベートネットワーク用ディレクトリを作成し、 geth をインストールする

```sh
% ssh <先ほど追加したユーザ>
```

```sh
$ cd /usr/local/src/
$ sudo curl -O https://gethstore.blob.core.windows.net/builds/geth-linux-amd64-1.10.18-de23cf91.tar.gz
$ sudo tar xzf geth-linux-amd64-1.10.18-de23cf91.tar.gz
$ sudo mkdir /<projectname>
$ sudo chown -R <username>:<username> /<projectname>/
$ sudo mv geth-linux-amd64-1.10.18-de23cf91/geth /<projectname>/
$ sudo chown <username>:<username> /<projectname>/geth 
```

* ここでインストールしてる geth は構築当時の古いバージョンです、本稿執筆時点の最新バージョンは 1.10.26 です。


## genesis.json を作成する

```sh
$ vi /<projectname>/genesis.json
```

`config.chainId` は 任意の値を設定できますが、 Metamask では同じ chainId のネットワークは複数登録できないので適宜変更してください。

```json
{
  "config": {
    "chainId": 10,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0
  },
  "alloc": {},
  "coinbase": "0x0000000000000000000000000000000000000001",
  "difficulty": "0x4000",
  "extraData": "",
  "gasLimit": "0x2fefd8",
  "nonce": "0x0000000000000042",
  "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp": "0x00"
}
```

## genesis.json でプライベートネットワークを初期化する

```sh
$ cd /<projectname>
$ ./geth --datadir ./<networkname> init ./genesis.json
```

## プライベートネットワークを起動する

`--http.addr` は、プライベートチェーンを利用するサービスが外部から参照するエンドポイントです。
限られた環境からのみ参照可能にしてください。

HTTP 経由で geth の API を利用しない場合、セキュリティ観点から不必要に `--http.api` オプションを付与する必要はありません。

```sh
$ ./geth --networkid 10 --datadir ./<networkname> --nodiscover --allow-insecure-unlock --http --http.addr <public domain or public ip> --http.corsdomain "*"  --http.vhosts "*" --http.api "eth,net,web3,personal,web3" console 2>> node.log
```

大きい処理を実験的に DApps 側に行わせる場合、 --targetgaslimit でブロックごとのガスリミットを上げてください。

参考設定値

- eth mainnet: 8000000
- eth testnet ropsten: 4712388


## プライベートネットワークでの初期設定

先ほど `console` オプションを付けて起動した geth の CLI にて操作します。

`console` を付けずに起動した geth にて CLI 操作をする場合には `attach` コマンドを利用します。

```sh
$ ./geth --datadir ./<networkname> attach ipc:./<networkname>/geth.ipc
```

### ノード情報の取得

```
> admin.nodeInfo.enode
"enode://xxxx@127.0.0.1:30303?discport=0"
```

ノード情報は控えておきます。

### マイニング用アカウントの作成

```
> personal.newAccount("<任意のパスワード>")
"0x......"
> eth.accounts[0] // 作成したアカウントの確認
"0x......"
> miner.setEtherbase(eth.accounts[0]) // 作成したアカウントをマイニング報酬を受け取るアカウントとして設定
> eth.coinbase // 設定を確認
> exit
```

ログ出力をしている node.log に、作成したアカウントの秘密鍵情報が下記のように記載されています。
ウォレットにインポートする際などに利用してください。
インポートの際は、 `personal.newAccount()` の引数で利用したパスワードが求められます。

```
WARN [06-07|04:20:24.251] Please backup your key file!             path=/geth/geth_private_net/keystore/UTC--2022-06-07T04-19-34.626120400Z--xxxxxxxxxxx
```

### (optional) p2p ノードの追加

同じ genesis.json ファイルを用いて作成したノードが他にある場合、 geth のコンソールモードの admin.nodeInfo.enode で得られた情報を用いて p2p ノードを固定します。

```sh
$ vi ./<networkname>/geth/static-nodes.json
```

```json
[
  "enode:xxxx@xxx.xxx.xxx.xxx:30303"
]
```

static-nodes.json 作成後、geth を再起動すると自動的に記載された p2p ノードが追加されます。

geth に console モードで入り、ピア情報を確認することで疎通確認ができます。

```
> admin.peers
...
```

### プライベートネットワークをバックグラウンドプロセスとして稼働

```sh
$ nohup ./geth --networkid 10 --datadir ./<networkname> --nodiscover --allow-insecure-unlock --http --http.addr <public domain or public ip> --http.corsdomain "*"  --http.vhosts "*" --http.api "eth,net,web3,personal,web3"  --mine 2>> node.log &
```

上記では `--http.api` オプションに personal が含まれていますが、不要であれば削除してください。
また `--mine` オプションを渡しているにも関わらずマイニングが開始されていない場合があります。
`eth.mining` が `true` を返しているにも関わらずログ上でマイニングが進んでいない場合、 console で入り、 `miner.stop()` -> `miner.start()` と実行してください。

## 動作確認

### マイニング

geth に console モードで入り、 blockNumber が時間とともに更新されていることを確認してください。

```
> eth.blockNumber
```

プライベートネットワーク用のノードを複数用意している場合、下記の操作で同じ結果が得られることを複数のノードで確認します。

```
> eth.getBlock(任意のノードが採掘したブロック番号).miner
> eth.getBlock(異なる任意のノードが採掘したブロック番号).miner
```

### Metamask

マイニング用に作成したアカウントの秘密鍵を Metamask にインポートします。

秘密鍵の場所は、「マイニング用アカウントの作成」の項目での作業時にログ出力されています。

ETH 残高が増えていれば成功です。

## プライベートネットワークのデータの破棄

geth で構築したプライベートネットのノードデータを破棄したい場合
下記のファイルやフォルダを削除してください。

```sh
$ rm -r /home/<username>/.ethash
$ rm -r /home/<username>/<networkname>
$ rm -r /home/<username>/node.log
```

# 参考記事

もみじに熊 / ethereumで複数ノードを接続する　part1
https://maple-bear.hatenablog.com/entry/2019/11/02/190611

NTTデータ先端技術株式会社 / ブロックチェーンEthereum入門 2
https://www.intellilink.co.jp/column/fintech/2016/060600.aspx

OpenGroove / MacにEthereumクライアント入れてひとりブロックチェーンしてみた
https://open-groove.net/other-tools/ethereum-geth-in-mac/
