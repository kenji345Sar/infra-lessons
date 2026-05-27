# ステップ 8: client から `apt install` まで通す

step-07 で `cds01` 側に「mTLS 越しでしか取れない apt repo」を公開した。今度は `client` 側に、その repo を **apt の世界に登録** して、最終的に `apt install hello-infra` まで通す。

## このステップのゴール

- `client` の apt が `https://cds01/repo/` を mTLS 認証つきで参照できる
- repo の GPG 公開鍵を `client` の信頼ストアに登録できている
- `apt update` がエラーなく通る
- `apt install hello-infra` が成功し、`hello-infra` コマンドが実行できる

## 所要時間

15 〜 20 分

---

## 前提

- step-07 まで完了している（cds01 の `/var/www/html/repo/` に 7 ファイルが存在し、client から mTLS で取得できる状態）
- `client` の `/etc/hosts` に `cds01` のエントリが入っている

---

## 手順

### 1. GPG 公開鍵を client 側に取得

step-07 でリポジトリに同梱した `infra-lessons-repo.pub` を、client 側に持ってくる。mTLS チャネルは信頼している前提なので、`curl` で直接取得してよい。

```bash
multipass exec client -- sudo curl -o /tmp/infra-lessons-repo.pub \
  --cert /etc/ssl/client01.crt \
  --key /etc/ssl/client01.key \
  https://cds01/repo/infra-lessons-repo.pub
multipass exec client -- ls -l /tmp/infra-lessons-repo.pub
```

`-rw-r--r-- 1 root root 1867 ... /tmp/infra-lessons-repo.pub` のように 1867 バイト前後あれば OK。

### 2. apt の keyring 形式（バイナリ）に変換して配置

apt は ASCII armor 形式 (`.pub`) ではなく、dearmor したバイナリ (`.gpg`) を `/etc/apt/keyrings/` に置いて参照する作法が現代的。

```bash
multipass exec client -- sudo mkdir -p /etc/apt/keyrings
multipass exec client -- sudo bash -c \
  'gpg --dearmor < /tmp/infra-lessons-repo.pub > /etc/apt/keyrings/infra-lessons-repo.gpg'
multipass exec client -- sudo chmod 644 /etc/apt/keyrings/infra-lessons-repo.gpg
multipass exec client -- ls -l /etc/apt/keyrings/infra-lessons-repo.gpg
```

### 3. apt の mTLS 設定を入れる

apt が `https://cds01/...` にアクセスするときだけ、クライアント証明書を提示するように設定する。

Mac 側でファイルを用意:

ファイル: `99-cds01-mtls.conf`

```
Acquire::https::cds01::SslCert "/etc/ssl/client01.crt";
Acquire::https::cds01::SslKey  "/etc/ssl/client01.key";
Acquire::https::cds01::CaInfo  "/usr/local/share/ca-certificates/infra-lessons-ca.crt";
Acquire::https::cds01::Verify-Peer "true";
Acquire::https::cds01::Verify-Host "true";
```

転送と配置:

```bash
multipass transfer /Users/apple/Desktop/Site/infra-lessons/01-package-delivery/99-cds01-mtls.conf client:/tmp/99-cds01-mtls.conf
multipass exec client -- sudo mv /tmp/99-cds01-mtls.conf /etc/apt/apt.conf.d/99-cds01-mtls.conf
multipass exec client -- sudo chown root:root /etc/apt/apt.conf.d/99-cds01-mtls.conf
```

> `Acquire::https::<host>::...` の形にしておくと、**`cds01` 宛のときだけ** クライアント証明書が送られる。普通の Ubuntu の公式リポジトリには漏れない。

### 4. sources.list.d にリポジトリを登録

flat repo を指す行を 1 つ書く。`signed-by` で先ほど配置した鍵に明示的に紐づける。

Mac 側でファイルを用意:

ファイル: `infra-lessons.list`

```
deb [signed-by=/etc/apt/keyrings/infra-lessons-repo.gpg] https://cds01/repo/ ./
```

転送と配置:

```bash
multipass transfer /Users/apple/Desktop/Site/infra-lessons/01-package-delivery/infra-lessons.list client:/tmp/infra-lessons.list
multipass exec client -- sudo mv /tmp/infra-lessons.list /etc/apt/sources.list.d/infra-lessons.list
multipass exec client -- sudo chown root:root /etc/apt/sources.list.d/infra-lessons.list
```

> 末尾の ` ./` は flat repo を意味する書き方。通常の `deb http://archive.ubuntu.com/ubuntu jammy main` の代わりに、ディレクトリパスを直接指している。

### 5. `apt update` を通す

```bash
multipass exec client -- sudo apt-get update
```

期待される出力:

```
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
...
Get:N https://cds01/repo ./ InRelease [865 B]
Get:N https://cds01/repo ./ Packages [501 B]
Reading package lists... Done
```

`https://cds01/repo ./ InRelease` と `Packages` が取れていれば成功。GPG 署名検証も同時に走っているので、ここを通った時点で「**鍵があって**、**証明書があって**、**SHA256 が一致した**」が全部 OK ということ。

### 6. パッケージをインストール

```bash
multipass exec client -- sudo apt-get install -y hello-infra
```

### 7. インストールしたコマンドを実行

```bash
multipass exec client -- hello-infra
```

期待される出力:

```
Hello from the infra-lessons private package!
```

これが出れば、**ミニ RHUI の機能が一通り動いた** ことになる。

---

## ここまでで何ができたか

- 自作 CA 配下のクライアント証明書を持つマシンだけが、自前リポジトリから `.deb` を取得・インストールできる
- 取得した `.deb` は GPG 署名で改ざんが検出される
- 通常の Ubuntu 公式 repo へのアクセスはそのまま（mTLS 設定は `cds01` 宛にのみ適用）

「ライセンス契約者（=鍵を持つマシン）にだけ、改ざんされていないパッケージを配る」という、RHUI が達成していることを、自前で組み立てた最小構成で再現できている。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `Certificate verification failed` | step-04 の自作 CA が `/usr/local/share/ca-certificates/infra-lessons-ca.crt` に置かれ、`update-ca-certificates` 済みか確認 |
| `The following signatures couldn't be verified because the public key is not available` | 手順 2 の dearmor 配置を確認。`/etc/apt/keyrings/infra-lessons-repo.gpg` がバイナリ形式で存在し、`sources.list.d` の `signed-by` がそのパスを指しているか |
| `400 No required SSL certificate was sent` | apt が証明書を提示できていない。手順 3 の `99-cds01-mtls.conf` のパスとファイル存在を確認 |
| `Could not resolve host: cds01` | client の `/etc/hosts` に `cds01` の行が無い。step-03 を参照 |
| `apt-get install` で見つからない | `apt-cache policy hello-infra` で repo に出ているか確認。出ていなければ `apt-get update` が成功しているか戻って確認 |

---

## 次のステップ

ここまでで 1 台の配信ノードからのインストールが通った。次は:

- 配信ノード 2 台目 (`cds02`) に同じリポジトリを同期
- HAProxy 経由で 2 台への負荷分散
- 片方を落としても client から見えるか
- ここまでを Ansible 化
