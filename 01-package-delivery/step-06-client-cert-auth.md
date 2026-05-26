# ステップ 6: クライアント証明書認証 (mTLS) の有効化

cds01 の nginx を「**クライアント証明書を提示しない接続は拒否**」する設定に変える。これが RHUI の核 ―「契約者しかパッケージを取りに来られない」を実現する仕組み。

## このステップのゴール

- cds01 の nginx が **mTLS (相互 TLS)** を要求する状態になる
- `client` から **証明書なしで接続** → サーバが拒否
- `client` から **`--cert` / `--key` で証明書を提示** → 200 OK で HTML が返る

これで「鍵を持っている VM だけ繋がる」状態が完成する。

## 所要時間

15 〜 20 分

## 前提

- ステップ5まで完了している（cds01 で HTTPS が動いている）
- `mgmt` の `~/ca/` に `client01.crt` / `client01.key` がある
- 各 VM が Running

`multipass list` で `mgmt`, `cds01`, `client` の 3 台が `Running` になっていることを確認してから進めてください（cds02 は今回未使用）。

---

## 何が変わるのか

### ステップ5まで（普通の HTTPS）

```
client ─── 「あなたが本物のcds01か証明書見せて」 ──→ cds01
client ←── サーバ証明書 (cds01.crt) ────────────── cds01
client ─── 検証OK、通信開始 ──→ cds01
```

サーバ側は「誰がアクセスしてきたか」を **知らない**。

### ステップ6（mTLS）

```
client ─── 「あなたが本物のcds01か証明書見せて」 ──→ cds01
client ←── サーバ証明書 (cds01.crt) ────────────── cds01
client ─── 検証OK ──→ cds01
client ─── 「私はclient01です、見て」+ client01.crt ──→ cds01
                                                      cds01 検証: 「OK、CAで署名されてる」
client ←── 通信開始 ────────────── cds01
```

**双方向で証明書を見せ合う**ので、サーバ側も「誰なのか」を確実に知れる。

---

## 手順

### 1. client01.crt と client01.key を `client` VM に配布

```bash
multipass exec mgmt -- cat /home/ubuntu/ca/client01.crt | multipass exec client -- sudo tee /etc/ssl/client01.crt
multipass exec mgmt -- cat /home/ubuntu/ca/client01.key | multipass exec client -- sudo tee /etc/ssl/client01.key
multipass exec client -- sudo chmod 600 /etc/ssl/client01.key
multipass exec client -- sudo chown root:root /etc/ssl/client01.crt /etc/ssl/client01.key
```

これで `client` VM の `/etc/ssl/` 配下に身分証一式が揃った。

### 2. cds01 の nginx 設定を mTLS 仕様に書き換え

ステップ5で書いた設定に **2行追加** する。

```bash
multipass exec cds01 -- sudo tee /etc/nginx/sites-available/cds01 > /dev/null <<'EOF'
server {
    listen 443 ssl;
    server_name cds01;

    ssl_certificate     /etc/ssl/cds01.crt;
    ssl_certificate_key /etc/ssl/cds01.key;

    ssl_client_certificate /usr/local/share/ca-certificates/infra-lessons-ca.crt;
    ssl_verify_client on;

    root /var/www/html;
    index index.html;
}
EOF
```

追加部分の意味:

| 設定 | 意味 |
|---|---|
| `ssl_client_certificate /usr/local/share/.../ca.crt` | 「**この CA が署名したクライアント証明書を信頼する**」と nginx に教える |
| `ssl_verify_client on` | **クライアント証明書の提示を必須にする**。なければ TLS ハンドシェイク段階で拒否 |

設定を有効化:

```bash
multipass exec cds01 -- sudo nginx -t
multipass exec cds01 -- sudo systemctl reload nginx
```

`nginx -t` で `syntax is ok` / `test is successful` が出ればOK。

### 3-a. 証明書なしで接続 → 拒否されることを確認

```bash
multipass exec client -- curl -v https://cds01/
```

**期待される結果**: HTTP 400 が返り、レスポンス本文に「No required SSL certificate was sent」のような文言が見える。

具体的には以下のような出力:

```
< HTTP/1.1 400 Bad Request
...
<center>400 No required SSL certificate was sent</center>
```

つまり nginx は「**クライアントが証明書出してこないから蹴った**」状態。これが「契約者じゃない人は配信を受けられない」を実現している瞬間。

### 3-b. 証明書を提示して接続 → 成功することを確認

`--cert` と `--key` で証明書を渡す。**`sudo` を付ける** ことに注意（`/etc/ssl/client01.key` は `chmod 600` + `root:root` なので、`multipass exec` のデフォルトユーザ (`ubuntu`) からは読めない。実運用でも `apt` / `dnf` は root として動くので、これが正しい姿）:

```bash
multipass exec client -- sudo curl -v --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/
```

**期待される結果**:

```
< HTTP/1.1 200 OK
< Server: nginx/...
...
<h1>Hello from cds01 via HTTPS</h1>
```

curl の冗長出力 (`-v`) には以下のような行が見えるはず:

```
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
```

これが **クライアントが自分の証明書をサーバに送っている瞬間**。普通の HTTPS 接続では出てこない、mTLS 特有のステップ。

### 4. (任意) サーバ側でクライアントを識別できることを確認

nginx は接続してきたクライアント証明書の情報を変数で持っている。アクセスログに含めれば、「誰がアクセスしてきたか」を全部記録できる。

`/etc/nginx/nginx.conf` に専用のログフォーマットを追加:

```bash
multipass exec cds01 -- sudo tee /etc/nginx/conf.d/mtls-log.conf > /dev/null <<'EOF'
log_format mtls '$remote_addr - "$ssl_client_s_dn" [$time_local] "$request" $status';
EOF
```

サイト設定でこのフォーマットを使う:

```bash
multipass exec cds01 -- sudo tee /etc/nginx/sites-available/cds01 > /dev/null <<'EOF'
server {
    listen 443 ssl;
    server_name cds01;

    ssl_certificate     /etc/ssl/cds01.crt;
    ssl_certificate_key /etc/ssl/cds01.key;

    ssl_client_certificate /usr/local/share/ca-certificates/infra-lessons-ca.crt;
    ssl_verify_client on;

    access_log /var/log/nginx/access.log mtls;

    root /var/www/html;
    index index.html;
}
EOF

multipass exec cds01 -- sudo nginx -t
multipass exec cds01 -- sudo systemctl reload nginx
```

そして再度アクセス:

```bash
multipass exec client -- sudo curl --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/
```

ログを見る:

```bash
multipass exec cds01 -- sudo tail -5 /var/log/nginx/access.log
```

こんな行が見えるはず:

```
192.168.252.6 - "CN=client01,O=infra-lessons,C=JP" [...] "GET / HTTP/1.1" 200
```

`CN=client01` がポイント。**サーバ側に「client01 さんが来た」と確実に記録される**。これが「誰に何を配ったか」のトレーサビリティの基盤。

---

## RHUI との対応

| 本演習 | RHUI v5 |
|---|---|
| `client01.crt` を `client` だけが持つ | **エンタイトルメント証明書**を契約者だけが持つ |
| `ssl_verify_client on` で nginx が要求 | CDS が同じ要求をする |
| ログに `CN=client01` が残る | 「どの契約 ID が、どのリポジトリを引いたか」が記録される |

これが「**契約者しかパッケージを取りに来られない**」の中身。エンタープライズ Linux 配信、IoT デバイス管理、Kubernetes 内部通信など、同じ構造が無数の場所で使われている。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `nginx -t` で `cannot load certificate` | パスが間違っている。`/usr/local/share/ca-certificates/infra-lessons-ca.crt` が存在することを `multipass exec cds01 -- ls -l ...` で確認 |
| 証明書ありでも 400 が返る | `client01.crt` が CA で署名されているか確認: `multipass exec mgmt -- openssl verify -CAfile ~/ca/ca.crt ~/ca/client01.crt` で `OK` が出るか |
| `curl: (58) unable to load client cert` | `/etc/ssl/client01.key` が `chmod 600` で root 所有か確認。所有者がずれていると root として動く `multipass exec` でも読めない場合がある |
| `curl: (58) unable to set private key file: ... type PEM` | `multipass exec client -- curl ...` だと `ubuntu` ユーザとして実行され、root 所有の鍵を読めない。**`multipass exec client -- sudo curl ...`** と sudo を付ける |

---

## 達成したこと

ここまでで:

1. 自前 CA を立てた
2. サーバ証明書で HTTPS を成立させた
3. **クライアント証明書を要求するように nginx を切り替えた**
4. 持っている人だけが繋がる、ログに「誰が来たか」が残る、を確認した

これは **RHUI、社内認証付き配信、Kubernetes mTLS、AWS IoT のクライアント証明書認証** すべての中身。

## 次のステップ

「鍵を持っている人だけ HTTPS で繋がる」までできた。次は:

- **実際のパッケージリポジトリ** を載せる（`.deb` ファイルを `aptly` でまとめて、署名付きで配信）
- `client` から `apt update` & `apt install` できる状態にする

ここまで来れば「ミニ RHUI」が機能として一通り動く。
