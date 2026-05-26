# ステップ 5: HTTPS サーバを cds01 に立てて TLS 接続を成立させる

cds01 で nginx を HTTPS で動かし、`client` から `https://cds01/` で実際に接続する。**「鍵マーク」相当が成立する瞬間** を実物で見る回。

## このステップのゴール

- cds01 で nginx が HTTPS (443) で待ち受けている
- client から `curl https://cds01/` が **何も警告なく** 成功する
- `curl -v` の出力で「TLS ハンドシェイク成功」「証明書の検証成功」が見える

このステップで初めて、ステップ4で作った証明書が **実際に使われる**。

## 所要時間

20 〜 30 分

---

## 概略

3 つの配布を先にやってから、nginx を立てる。

| 配布元 | 配布先 | 何を | 何のため |
|---|---|---|---|
| mgmt | cds01 + client | `ca.crt` | 「この CA を信頼する」ためのルート証明書 |
| mgmt | cds01 | `cds01.crt` + `cds01.key` | サーバが自分を名乗るための証明書 |
| - | cds01 | nginx | HTTPS で配信する Web サーバ |

`ca.crt` は **両方** に配るのがポイント:
- cds01 にも入れるのは慣例（後でクライアント証明書認証を追加するとき必要になる）
- client に入れるのが今回の主役。「`ca.crt` を信頼するという宣言」がないと TLS は成立しない

---

## 手順

すべて Mac のターミナルから `multipass exec` で叩く。

### 1. ca.crt を cds01 と client に配布

Ubuntu の「信頼する CA を置く場所」は `/usr/local/share/ca-certificates/`。ここにファイルを置いて `update-ca-certificates` を叩くと、システム全体（curl, wget, apt など）から信頼される。

```bash
multipass exec mgmt -- cat /home/ubuntu/ca/ca.crt | multipass exec cds01 -- sudo tee /usr/local/share/ca-certificates/infra-lessons-ca.crt
multipass exec mgmt -- cat /home/ubuntu/ca/ca.crt | multipass exec client -- sudo tee /usr/local/share/ca-certificates/infra-lessons-ca.crt
multipass exec cds01 -- sudo update-ca-certificates
multipass exec client -- sudo update-ca-certificates
```

期待される出力（末尾）:

```
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
```

`1 added` が出ていれば OK。

### 2. cds01 にサーバ証明書と秘密鍵を配置

```bash
multipass exec mgmt -- cat /home/ubuntu/ca/cds01.crt | multipass exec cds01 -- sudo tee /etc/ssl/cds01.crt
multipass exec mgmt -- cat /home/ubuntu/ca/cds01.key | multipass exec cds01 -- sudo tee /etc/ssl/cds01.key
multipass exec cds01 -- sudo chmod 600 /etc/ssl/cds01.key
multipass exec cds01 -- sudo chown root:root /etc/ssl/cds01.crt /etc/ssl/cds01.key
```

> 秘密鍵 (`cds01.key`) は `chmod 600`（所有者だけ読める）にして他ユーザーから守る。これも PKI の鉄則の一つ。

### 3. nginx をインストール

```bash
multipass exec cds01 -- sudo apt-get update
multipass exec cds01 -- sudo apt-get install -y nginx
```

数十秒で完了。

### 4. nginx を HTTPS 設定にする

サイト設定ファイルを書き込む:

```bash
multipass exec cds01 -- sudo tee /etc/nginx/sites-available/cds01 > /dev/null <<'EOF'
server {
    listen 443 ssl;
    server_name cds01;

    ssl_certificate     /etc/ssl/cds01.crt;
    ssl_certificate_key /etc/ssl/cds01.key;

    root /var/www/html;
    index index.html;
}
EOF
```

設定の有効化（`sites-enabled/` にシンボリックリンクを張る Ubuntu/Debian 流の作法）:

```bash
multipass exec cds01 -- sudo ln -sf /etc/nginx/sites-available/cds01 /etc/nginx/sites-enabled/cds01
multipass exec cds01 -- sudo rm -f /etc/nginx/sites-enabled/default
multipass exec cds01 -- sudo nginx -t
multipass exec cds01 -- sudo systemctl restart nginx
```

`nginx -t` で `syntax is ok` と `test is successful` が出れば設定 OK。

### 5. テスト用 HTML を置く

```bash
multipass exec cds01 -- sudo bash -c 'echo "<h1>Hello from cds01 via HTTPS</h1>" > /var/www/html/index.html'
```

### 6. client から HTTPS で接続

ここが本番:

```bash
multipass exec client -- curl -v https://cds01/
```

成功すると、出力の中に以下のような行が見える:

```
* Server certificate:
*  subject: C=JP; O=infra-lessons; CN=cds01
*  issuer: C=JP; O=infra-lessons; CN=infra-lessons CA
*  SSL certificate verify ok.
...
* TLSv1.3 (IN), TLS handshake, Server hello
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
...
<h1>Hello from cds01 via HTTPS</h1>
```

**ポイント**:

- `subject: ... CN=cds01` … サーバ証明書が「自分は cds01 だ」と名乗っている
- `issuer: ... CN=infra-lessons CA` … 名乗りを保証している CA がさっき作った自前 CA
- `SSL certificate verify ok.` … client が **「あなたの CA は信頼する」** と認めた（ステップ1で `update-ca-certificates` したから）
- 最後に HTML が返ってきて、これは **暗号化通信** で運ばれている

これが TLS の成立。Web ブラウザのアドレスバーに「鍵マーク」が付くのと同じことが、ここで起きた。

---

## 確認: 何が「成立した」のか

| 要素 | 今回の値 |
|---|---|
| サーバ証明書 (cds01.crt) | mgmt の自前 CA が署名 |
| クライアントが信頼する CA | `/usr/local/share/ca-certificates/infra-lessons-ca.crt` |
| 接続先名 (`https://cds01/`) | cds01.crt の SAN (`DNS:cds01`) と一致 |
| 検証結果 | OK |

これらが全部揃って **初めて** TLS が成立する。1 個でも欠ければ失敗する。

---

## (任意) 壊れるとどう失敗するか

### CA を信頼しなかったら？

`--cacert` で「この CA だけを信頼する」と限定しつつ、別の存在しない CA を渡すと:

```bash
multipass exec client -- bash -c 'curl https://cds01/ --cacert /etc/hostname'
```

`curl: (77) error setting certificate file` のようなエラー。要するに **「あなたは私の知らない CA で署名されている」** と弾かれる。

### CN/SAN が一致しなかったら？

cds02 の IP を `cds01` の名前で叩いてみる（実機で試したい場合）:

`client` の `/etc/hosts` で `cds01` 行の IP を **cds02 の IP** に書き換えると、cds01.crt の SAN が `DNS:cds01` なのに、接続先の中身は cds02。これでは TLS は **「名前が違うサーバに接続しようとしている可能性」** を検知する。実運用ではこれが MITM 検知の根拠。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `curl: (60) SSL certificate problem: unable to get local issuer certificate` | `ca.crt` を client に置いた後 `update-ca-certificates` を実行したか確認 |
| `curl: (7) Failed to connect to cds01 port 443: Connection refused` | nginx が起動していない。`multipass exec cds01 -- sudo systemctl status nginx` で確認 |
| `nginx: configuration file /etc/nginx/nginx.conf test failed` | サイト設定の文法ミス。`nginx -t` の出力を読む |
| `Permission denied` で `/etc/ssl/cds01.key` を nginx が読めない | `chmod 600` のままで OK（nginx は root として起動するので読める）。それでもダメなら chown が違う |

---

## ここまでで何ができたか

- 自分が立てた CA が発行したサーバ証明書で、TLS が成立した
- **Let's Encrypt や ACM が裏で同じことをやっている** 仕組みを、自分で全部組み立てた
- 配信先 (client) は CA を 1 個信頼するだけで、その CA が署名する全サーバを信頼できる構造

## 次のステップ

普通の HTTPS は成立。次は:

- **クライアント証明書認証 (mTLS) を有効にして、「client01.crt を持っている VM だけ」アクセスできる状態を作る**
- これが RHUI の「契約者しかパッケージを取りに来られない」を担っている仕組み
