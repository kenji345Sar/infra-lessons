# ステップ 3: ホスト名で VM を呼び出せるようにする

毎回 IP を調べて打つのは面倒だし、Multipass を再起動すると IP が変わることがある。各 VM の `/etc/hosts` に名前と IP を書いておけば、**`ping cds01` のようにホスト名で呼べる** ようになる。

## このステップのゴール

- 各 VM から、他の 3 台を **ホスト名で** ping できる
  - `ping mgmt` / `ping cds01` / `ping cds02` / `ping client` が通る

## 所要時間

10 〜 15 分

## なぜやるのか

- IP は環境ごとに変わる（あなたの環境では `192.168.252.x`、別の Mac だと `192.168.64.x` になる）
- VM 再起動で IP が変わると、設定ファイルを全部書き直すことになる
- 名前で書いておけば、IP が変わっても **`/etc/hosts` の 1 ファイルだけ直せばよい**

これは本番の RHUI でも同じ思想で、ノードは DNS 名で参照する。

---

## 前提知識: `/etc/hosts` とは

Linux で「**この名前は、この IP**」というローカルな対応表を書くファイル。DNS の前段にあって、ここに書いてあれば DNS を引かずに即解決する。

書式は単純:

```
192.168.252.3   mgmt
192.168.252.4   cds01
```

`IPアドレス<空白>名前` を 1 行に書くだけ。

---

## 手順

### 1. 現在の IP を確認

Mac のターミナルで:

```bash
multipass list
```

4 台の IP をメモする。以下、メモした値を **`<mgmtのIP>`** のように表記する。

### 2. `mgmt` に入って `/etc/hosts` を編集

```bash
multipass shell mgmt
```

VM 内で、`/etc/hosts` の末尾に 4 行を追加する:

```bash
sudo tee -a /etc/hosts <<EOF
<mgmtのIP>    mgmt
<cds01のIP>   cds01
<cds02のIP>   cds02
<clientのIP>  client
EOF
```

> `tee -a` は「ファイル末尾に追記」。`sudo` は `/etc/hosts` が root 所有のため必要。`<<EOF ... EOF` で複数行を一気に流し込んでいる。

`<mgmtのIP>` などはあなたの環境の実際の IP に置き換えてから実行する。例:

```bash
sudo tee -a /etc/hosts <<EOF
192.168.252.3   mgmt
192.168.252.4   cds01
192.168.252.5   cds02
192.168.252.6   client
EOF
```

### 3. `mgmt` から ping をホスト名で試す

```bash
ping -c 3 cds01
ping -c 3 cds02
ping -c 3 client
```

3 回ずつ返ってくれば成功。失敗するなら `cat /etc/hosts` で末尾の追記が反映されているか確認する。

抜ける:

```bash
exit
```

### 4. 同じことを残り 3 台にもやる

`cds01`, `cds02`, `client` でも同じ内容を `/etc/hosts` に追記する。**4 台すべてに同じ 4 行を入れる**。

```bash
multipass shell cds01
sudo tee -a /etc/hosts <<EOF
<mgmtのIP>    mgmt
<cds01のIP>   cds01
<cds02のIP>   cds02
<clientのIP>  client
EOF
exit
```

`cds02` と `client` についても同様に実施。

### 5. 各 VM からホスト名で ping して確認

例えば `client` から:

```bash
multipass shell client
ping -c 1 mgmt
ping -c 1 cds01
ping -c 1 cds02
exit
```

すべて返ってくれば、4 台が **互いをホスト名で認識できる状態** になった。

---

## 楽な方法（Mac から一括設定）

各 VM に手で入って `tee` するのは面倒。Mac 側から `multipass exec` を使うと、VM の中に入らずに一括設定できる。

**何度実行しても安全**（既存エントリを消してから書き直すので、`stop`/`start` で IP が変わっても再実行で追従できる）。

### 1. 4 VM がすべて起動していることを確認

```bash
multipass start mgmt cds01 cds02 client
multipass list
```

`State` がすべて `Running` で、`IPv4` 列が埋まっていること。

> Multipass は `stop` → `start` で IP が変わることがある。下記スクリプトは「**今の IP** を取得して反映する」ので、IP が前回と違っていても問題ない。

### 2. 一括設定スクリプトを実行

Mac のターミナル（Macのプロンプト、VMの中ではない）で、以下を**まるごとコピペ**して実行する:

```bash
for vm in mgmt cds01 cds02 client; do
  multipass exec $vm -- sudo bash -c "echo 'manage_etc_hosts: false' > /etc/cloud/cloud.cfg.d/99-disable-manage-etc-hosts.cfg"
done

MGMT_IP=$(multipass info mgmt | awk '/^IPv4:/ {print $2}')
CDS01_IP=$(multipass info cds01 | awk '/^IPv4:/ {print $2}')
CDS02_IP=$(multipass info cds02 | awk '/^IPv4:/ {print $2}')
CLIENT_IP=$(multipass info client | awk '/^IPv4:/ {print $2}')
echo "mgmt=$MGMT_IP cds01=$CDS01_IP cds02=$CDS02_IP client=$CLIENT_IP"
HOSTS_BLOCK="# infra-lessons-hosts-begin
$MGMT_IP   mgmt
$CDS01_IP   cds01
$CDS02_IP   cds02
$CLIENT_IP   client
# infra-lessons-hosts-end"
for vm in mgmt cds01 cds02 client; do
  echo "=== $vm ==="
  multipass exec $vm -- sudo sed -i '/# infra-lessons-hosts-begin/,/# infra-lessons-hosts-end/d' /etc/hosts
  echo "$HOSTS_BLOCK" | multipass exec $vm -- sudo tee -a /etc/hosts
done
echo "完了"
```

> **2 つ目以降のループより前に cloud-init を切る** のがポイント。Multipass の Ubuntu イメージは `manage_etc_hosts: true` がデフォルトで、**VM を再起動すると /etc/hosts が cloud-init のテンプレートから再生成されて、自分で追記したブロックが消える**。最初のループで `manage_etc_hosts: false` のドロップイン設定を置いておくと、以降の再起動でも /etc/hosts が保持される。

> zsh はスクリプト中の `#` で始まる単独行を「コメント」ではなく「コマンド」と解釈するので、貼り付け用スクリプトには行頭コメントを置かない。各行の意味は下の表で説明する。

スクリプトの中身:

| 行 | 何をしているか |
|---|---|
| `multipass info VM \| awk ...` | 各 VM の `multipass info` 出力から `IPv4:` の行だけ抜き、`awk` で 2 列目（IP）を取り出す |
| `HOSTS_BLOCK=...` | `/etc/hosts` に追記する 6 行（マーカー2 + 本体 4）を文字列として組み立てる |
| `for vm in ...; do` | 4 VM 分繰り返す |
| `sed -i '/begin/,/end/d'` | マーカー間の行を削除（再実行で重複しないように） |
| `tee -a /etc/hosts` | 末尾にブロックを追記 |
| `multipass exec VM -- コマンド` | VM の中に入らず、Mac からそのコマンドを VM 上で実行 |

### 3. 結果を確認

```bash
for vm in mgmt cds01 cds02 client; do
  echo "=== $vm /etc/hosts ==="
  multipass exec $vm -- cat /etc/hosts
done
```

4 VM 全てで、末尾に `# infra-lessons-hosts-begin` 〜 `# infra-lessons-hosts-end` のブロックが見えれば成功。

### 4. ホスト名で ping 疎通確認

Mac から `multipass exec` でそのまま叩ける:

```bash
multipass exec mgmt -- ping -c 1 cds01
multipass exec mgmt -- ping -c 1 cds02
multipass exec mgmt -- ping -c 1 client
multipass exec client -- ping -c 1 mgmt
multipass exec client -- ping -c 1 cds01
```

すべて `1 received` が返れば、4 VM 間でホスト名による疎通ができている状態。

> `multipass exec VM名 -- コマンド` は「VM に入らずに、その VM 上でコマンドを 1 つ実行する」機能。後で Ansible に置き換えるときの前段階として覚えておくとよい（Ansible はこの仕組みを SSH 経由で大規模にやっている）。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `ping cds01` が `Name or service not known` | `/etc/hosts` に正しく追記できているか `cat /etc/hosts` で確認。IP が間違っている可能性も |
| 同じ名前が複数行あって挙動が変 | `tee -a` を 2 回実行してしまった可能性。`sudo vi /etc/hosts` で重複行を削除 |
| VM を再起動したら ping が通らなくなった | (1) IP が変わった可能性: `multipass list` で確認、(2) **cloud-init の `manage_etc_hosts: true`** で /etc/hosts が再生成された可能性。上の楽な方法のスクリプトを再実行（idempotent） |
| 楽な方法のスクリプト中で「`info failed: ssh connection failed`」が出る | Multipass の VM が一時的にハングしている。該当 VM を `multipass stop --force NAME && multipass start NAME` で復旧してから再実行 |

---

## 次のステップ

VM 間で名前による通信ができるようになった。次は **自作 CA を作って、サーバ証明書とクライアント証明書を発行する** ステップに入る。HTTPS と「誰がアクセスしてきたかを証明書で識別する」仕組みの土台。
