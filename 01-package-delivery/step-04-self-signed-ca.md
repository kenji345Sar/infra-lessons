# ステップ 4: 自作 CA と証明書の発行

`mgmt` 上に自分専用の認証局（CA）を立てて、サーバ証明書とクライアント証明書を発行する。後のステップで使う HTTPS と「クライアント証明書認証」の土台。

## このステップのゴール

`mgmt` の `~/ca/` 配下に以下のファイルが揃う:

| ファイル | 意味 |
|---|---|
| `ca.key` | 自作 CA の **秘密鍵**。CA の身元の元。漏れたら全部終わり |
| `ca.crt` | 自作 CA の **公開証明書**。「この CA を信頼する」と各サーバ・クライアントに配布する |
| `cds01.key` / `cds01.crt` | cds01 用の **サーバ証明書**一式 |
| `client01.key` / `client01.crt` | **クライアント証明書**一式 |

このステップでは「作るだけ」。配布や HTTPS 接続は次のステップ。

## 所要時間

15 〜 20 分

## 用語

`CA` / `クライアント証明書認証` / `TLS` は [../GLOSSARY.md](../GLOSSARY.md) 参照。

---

## なぜ 3 種類の鍵・証明書を作るのか

| 種類 | 役割 | 信頼の仕組み |
|---|---|---|
| **CA 証明書** (`ca.crt`) | 「これを信頼する」とみんなが事前に知っておく根本 | 各サーバ・クライアントに `ca.crt` を配布 |
| **サーバ証明書** (`cds01.crt`) | サーバが自分を名乗る身分証 | CA で署名 → `ca.crt` を信頼する人なら検証可能 |
| **クライアント証明書** (`client01.crt`) | クライアントが自分を名乗る身分証 | サーバ側で CA 署名済みかを検証 |

公的 CA（Let's Encrypt など）の仕組みと同じ。今回は学習用に自前で立てる。

---

## 手順

すべて mgmt 上で実行するが、Mac から `multipass exec` で叩いて完結させる。

### 1. 作業ディレクトリの準備

```bash
multipass exec mgmt -- mkdir -p /home/ubuntu/ca
```

mgmt の `~/ca/` が以降の作業領域。

### 2. CA の作成

Mac のターミナルから、まるごと貼り付け:

```bash
multipass exec mgmt -- bash -c '
cd ~/ca
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/C=JP/O=infra-lessons/CN=infra-lessons CA" \
  -out ca.crt
ls -l ca.key ca.crt
'
```

中身:

| 行 | 何をしているか |
|---|---|
| `openssl genrsa -out ca.key 4096` | 4096 bit RSA の秘密鍵を作る（CA は長めの鍵にするのが慣例） |
| `openssl req -x509 ...` | その鍵で **自己署名証明書** を作る（自分で自分にハンコを押す = 自分が CA だと宣言する） |
| `-subj "/C=JP/O=...."` | 証明書に書く名前。`CN` (Common Name) が「誰の証明書か」 |
| `-days 3650` | 10 年有効 |

### 3. サーバ証明書（cds01 用）の発行

```bash
multipass exec mgmt -- bash -c '
cd ~/ca
openssl genrsa -out cds01.key 2048
openssl req -new -key cds01.key \
  -subj "/C=JP/O=infra-lessons/CN=cds01" \
  -out cds01.csr
cat > cds01.ext <<EOF
subjectAltName = DNS:cds01
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF
openssl x509 -req -in cds01.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out cds01.crt -days 365 -sha256 \
  -extfile cds01.ext
ls -l cds01.*
'
```

順序:

1. **`cds01.key`** （サーバの秘密鍵）を生成
2. **`cds01.csr`** （Certificate Signing Request = 証明書発行依頼書）を作る
3. **拡張情報** を `cds01.ext` に書く（SAN: DNS 名 / 用途: サーバ用）
4. **CSR を CA で署名** して `cds01.crt` を発行

> **SAN (Subject Alternative Name)** は重要。TLS 接続時にクライアントは「接続先の名前と、証明書の SAN/CN が一致するか」を見る。ここで `DNS:cds01` を入れたから、ホスト名 `cds01` で接続したときに有効になる。

### 4. クライアント証明書（client01 用）の発行

```bash
multipass exec mgmt -- bash -c '
cd ~/ca
openssl genrsa -out client01.key 2048
openssl req -new -key client01.key \
  -subj "/C=JP/O=infra-lessons/CN=client01" \
  -out client01.csr
cat > client01.ext <<EOF
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
EOF
openssl x509 -req -in client01.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client01.crt -days 365 -sha256 \
  -extfile client01.ext
ls -l client01.*
'
```

サーバ証明書とほぼ同じ。違いは:

- `extendedKeyUsage = clientAuth` …「クライアント用途」と明示
- SAN は不要（クライアントは自分のホスト名を名乗らないため）

### 5. 発行した証明書の中身を確認

```bash
multipass exec mgmt -- bash -c '
cd ~/ca
echo "=== CA 証明書 ==="
openssl x509 -in ca.crt -noout -subject -issuer -dates
echo
echo "=== サーバ証明書 (cds01) ==="
openssl x509 -in cds01.crt -noout -subject -issuer -dates
openssl x509 -in cds01.crt -noout -ext subjectAltName -ext extendedKeyUsage
echo
echo "=== クライアント証明書 (client01) ==="
openssl x509 -in client01.crt -noout -subject -issuer -dates
openssl x509 -in client01.crt -noout -ext extendedKeyUsage
echo
echo "=== CA で検証 ==="
openssl verify -CAfile ca.crt cds01.crt
openssl verify -CAfile ca.crt client01.crt
'
```

最後に **`cds01.crt: OK`** と **`client01.crt: OK`** が出れば、CA が正しく両方を署名しているということ。これがこのステップの最終確認。

### 6. ファイル一覧の確認

```bash
multipass exec mgmt -- ls -l /home/ubuntu/ca/
```

最低限以下が見えていれば OK:

- `ca.crt`, `ca.key`
- `cds01.crt`, `cds01.csr`, `cds01.ext`, `cds01.key`
- `client01.crt`, `client01.csr`, `client01.ext`, `client01.key`
- `ca.srl`（シリアル管理ファイル。`-CAcreateserial` で自動生成）

---

## 各ファイルの取り扱い (重要)

| ファイル | 漏らしていいか | 後でどこに配るか |
|---|---|---|
| `ca.key` | **絶対 NG**。漏れたら CA の意味なし | mgmt に置きっぱなし |
| `ca.crt` | OK（公開情報） | cds01, cds02, client 全部に配る |
| `cds01.key` | NG | cds01 だけに渡す |
| `cds01.crt` | OK | cds01 に渡す |
| `client01.key` | NG | client だけに渡す |
| `client01.crt` | OK | client に渡す |

「秘密鍵 (`.key`) は外に出さない」が PKI の鉄則。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `openssl: command not found` | `multipass exec mgmt -- sudo apt-get install -y openssl`（Ubuntu には標準で入っているはずだが） |
| `subjectAltName = DNS:cds01` がなくて TLS 接続が失敗 | 次のステップで顕在化する。今は気にしなくていい |
| `Cannot read ca.key` 系のエラー | パスがずれている。`cd ~/ca` から始まっているか確認 |

---

## 次のステップ

PKI 材料が揃った。次は:

1. `ca.crt` を `cds01` と `client` に配る（信頼ルートの配布）
2. `cds01.key` / `cds01.crt` を `cds01` にコピーして nginx で HTTPS サーバを立ち上げ
3. `client` から `https://cds01` で接続して、TLS が成立することを確認

クライアント証明書認証は **その次** のステップで足す（まずは普通の HTTPS まで）。
