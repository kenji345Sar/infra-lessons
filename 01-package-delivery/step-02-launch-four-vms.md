# ステップ 2: 用途別 VM 4 台の起動と通信確認

役割ごとに名前を付けた VM を 4 台立てて、VM どうしで通信できることを確認する。

## このステップのゴール

- 4 台の VM が「Running」状態で見える
- 各 VM の IP アドレスがわかる
- VM どうしで `ping` が通る
- 各 VM に `multipass shell` で入れる

ここまでくれば、以降のステップでサーバ間通信を扱う土台ができたことになる。

## 所要時間

15 〜 20 分

## 4 台の役割

| VM 名 | 役割 | RHUI v5 で対応するもの |
|---|---|---|
| `mgmt` | 管理ノード。リポジトリ生成・パッケージ署名 | RHUA |
| `cds01` | 配信ノード #1。クライアントに配信 | CDS |
| `cds02` | 配信ノード #2。冗長化用 | CDS |
| `client` | パッケージを受け取る側 | RHEL クライアント |

> HAProxy（ロードバランサ）用の VM は、HAProxy を扱うステップで追加する。今は不要。

## リソースの目安

Multipass のデフォルト設定は VM 1 台あたり CPU 1 / RAM 1GB / Disk 5GB。4 台で **RAM 約 4GB を消費** する。手元の Mac の空きメモリに余裕があるか確認しておく。

---

## 手順

### 1. 4 台を順に起動

```bash
multipass launch --name mgmt 22.04
multipass launch --name cds01 22.04
multipass launch --name cds02 22.04
multipass launch --name client 22.04
```

1 台あたり 30 秒〜 1 分ほど（イメージはキャッシュ済みなので前回より速い）。

> 4 行コピペでも問題ないが、後から繰り返したくなるので、シェルスクリプト化はステップ後半で扱う。

### 2. 4 台すべて Running を確認

```bash
multipass list
```

こうなっていれば OK:

```
Name     State     IPv4             Image
mgmt     Running   192.168.64.x     Ubuntu 22.04 LTS
cds01    Running   192.168.64.y     Ubuntu 22.04 LTS
cds02    Running   192.168.64.z     Ubuntu 22.04 LTS
client   Running   192.168.64.w     Ubuntu 22.04 LTS
```

IP アドレスは Multipass が自動で割り当てる（`192.168.64.x` 系が典型）。

### 3. 各 VM の詳細を確認

```bash
multipass info mgmt cds01 cds02 client
```

各 VM の CPU・メモリ・ディスク・IP がまとめて出る。**この IP アドレスは次の手順で使うのでメモしておく**。

### 4. VM 間で ping が通ることを確認

> ⚠️ **IP は自分の環境のものを使う**。Multipass が割り当てるサブネットは環境によって違う（`192.168.64.x` のことも `192.168.252.x` のこともある）。下記コマンドの `<cds01のIP>` などは、手順 3 の `multipass info` の出力に置き換えること。

`mgmt` に入って、他 3 台に ping を打つ。

```bash
multipass shell mgmt
```

VM に入ったら（プロンプトが `ubuntu@mgmt:~$` になる）:

```bash
ping -c 3 <cds01のIP>     # cds01 へ
ping -c 3 <cds02のIP>     # cds02 へ
ping -c 3 <clientのIP>    # client へ
```

`64 bytes from ...` が 3 回返って `3 packets transmitted, 3 received` であれば成功。

抜ける:

```bash
exit
```

### 5. 別の VM からも試す

念のため `client` から他 3 台にも ping してみる:

```bash
multipass shell client
ping -c 3 <mgmtのIP>
ping -c 3 <cds01のIP>
ping -c 3 <cds02のIP>
exit
```

これで「**どの VM からどの VM にも通信できる**」状態が確認できた。

### 6. (任意) 全 VM を停止

作業を中断するなら停止しておく:

```bash
multipass stop mgmt cds01 cds02 client
```

`multipass list` で全部 `Stopped` になれば OK。

再開は:

```bash
multipass start mgmt cds01 cds02 client
```

> 注: 再起動すると IP アドレスが変わることがある。次のステップでホスト名解決の設定を入れて IP 直書きから卒業する。

---

## なぜ ping で確認するのか

ping は ICMP という最も基本的なプロトコルで通信できるかを試すコマンド。「**そもそも届くのか**」を切り分けるための最低限の確認。

ここで通らなければ、後続の HTTP / TLS の話は全部成立しないので、土台を先に固める。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| 2 台目以降の `launch` が「Insufficient memory」で失敗 | Mac の空きメモリ不足。他アプリを閉じるか、`--memory 512M` で VM のメモリを減らす |
| `multipass info` で IP が `--` のまま | VM の起動直後でネットワーク初期化中。1 分待って再実行 |
| `ping` が通らない | (1) IP を見間違えていないか確認、(2) 両 VM ともに `Running` か確認、(3) Mac のファイアウォール設定で Multipass がブロックされていないか確認 |

---

## 次のステップ

VM 間で IP 直書きの ping は通った。次は:

- 各 VM に **わかりやすい名前で接続できるよう** にする（`mgmt` という名前で cds01 から呼べるようにする）
- 自作 CA を立てて、サーバ証明書・クライアント証明書を発行する準備に入る
