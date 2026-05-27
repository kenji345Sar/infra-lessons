# ステップ 7: 自前 apt リポジトリの作成と配信

実物の `.deb` パッケージを作って、署名付きの apt リポジトリとして cds01 から配信する。これで「ミニ RHUI」の中身が **モノを配る** ところまで揃う。

## このステップのゴール

- `mgmt` 上でカスタム `.deb` パッケージ (`hello-infra`) を作る
- それを **GPG 署名つき** の flat apt リポジトリ形式にまとめる
- cds01 の nginx でその repo を `https://cds01/repo/` に公開する
- `client` から `curl --cert ...` で `Packages.gz` などのファイルが取れることを確認

実際に `apt install` できるようにする設定は次のステップ。**この step は「リポジトリが存在し、mTLS 越しにファイルが取れる」までを作る**。

## 所要時間

30 〜 40 分

---

## リポジトリ構造の概要 (flat repo)

flat repo は最小構成の apt リポジトリ形式。1 つのディレクトリに以下を並べる:

```
repo/
├── hello-infra_1.0.0_all.deb   ← パッケージ本体
├── Packages                    ← 含まれるパッケージのメタデータ
├── Packages.gz                 ← 上を gzip 圧縮したもの
├── Release                     ← リポジトリ全体の情報 + 各ファイルのSHA256
├── Release.gpg                 ← Release の GPG 署名（分離型）
├── InRelease                   ← Release + 署名の埋め込み版
└── infra-lessons-repo.pub      ← クライアントが信頼する GPG 公開鍵
```

`client` 側の apt は次の順で動く:

1. `https://cds01/repo/InRelease` を取得 → GPG 検証
2. `Release` の中の `Packages.gz` の SHA256 を見る
3. `Packages.gz` を取得 → SHA256 を計算して照合
4. `Packages.gz` の中から欲しいパッケージを探す
5. 該当 `.deb` を取得してインストール

**改ざんが効かない** のはこの「Release → Packages → .deb の SHA チェーン」のおかげ。出発点の Release が GPG 署名されているので、攻撃者が中身を差し替えてもクライアントが拒否する。

---

## 手順

すべて Mac のターミナルから `multipass exec` で叩く。

### 1. 必要なツールを mgmt に導入

`apt-utils`（`apt-ftparchive` を含む）と `gnupg` を入れる。

```bash
multipass exec mgmt -- sudo apt-get update
multipass exec mgmt -- sudo apt-get install -y apt-utils gnupg
```

### 2. リポジトリ署名用の GPG 鍵を生成（無対話）

通常 `gpg --gen-key` は対話式だが、バッチファイルを渡すと自動生成できる。

```bash
multipass exec mgmt -- bash -c 'cat > /tmp/gpg-batch' <<'EOF'
%no-protection
Key-Type: RSA
Key-Length: 2048
Name-Real: infra-lessons repo signing
Name-Email: noreply@example.com
Expire-Date: 0
%commit
EOF

multipass exec mgmt -- gpg --batch --gen-key /tmp/gpg-batch
multipass exec mgmt -- gpg --list-keys
```

`pub` の行に `infra-lessons repo signing <noreply@example.com>` が見えれば OK。`%no-protection` を入れているのでパスフレーズなし（学習用。実運用ではちゃんと付ける）。

### 3. 最小の `.deb` パッケージを作る

「`hello-infra` を実行すると挨拶する」だけの極小パッケージを作る。

ディレクトリ準備:

```bash
multipass exec mgmt -- mkdir -p /home/ubuntu/repo-build/hello-infra/DEBIAN
multipass exec mgmt -- mkdir -p /home/ubuntu/repo-build/hello-infra/usr/local/bin
```

パッケージのメタデータ (control ファイル):

```bash
multipass exec mgmt -- bash -c 'cat > /home/ubuntu/repo-build/hello-infra/DEBIAN/control' <<'EOF'
Package: hello-infra
Version: 1.0.0
Architecture: all
Maintainer: infra-lessons <noreply@example.com>
Description: Minimal demo package for infra-lessons
EOF
```

実行されるスクリプト:

```bash
multipass exec mgmt -- bash -c 'cat > /home/ubuntu/repo-build/hello-infra/usr/local/bin/hello-infra' <<'EOF'
#!/bin/sh
echo "Hello from the infra-lessons private package!"
EOF

multipass exec mgmt -- chmod +x /home/ubuntu/repo-build/hello-infra/usr/local/bin/hello-infra
```

`.deb` をビルド:

```bash
multipass exec mgmt -- bash -c 'cd ~/repo-build && dpkg-deb --build hello-infra hello-infra_1.0.0_all.deb'
multipass exec mgmt -- ls -l /home/ubuntu/repo-build/
```

`hello-infra_1.0.0_all.deb` が出来ていればOK。

### 4. flat apt repo を組み立てるスクリプトを mgmt に配置

リポジトリのメタデータ生成と署名を一気にやるスクリプト:

```bash
multipass exec mgmt -- bash -c 'cat > /home/ubuntu/build-repo.sh' <<'EOF'
#!/bin/bash
set -euo pipefail

REPO_DIR=/home/ubuntu/repo
SRC_DEB=/home/ubuntu/repo-build/hello-infra_1.0.0_all.deb
SIGNER_EMAIL=noreply@example.com

mkdir -p "$REPO_DIR"
cp "$SRC_DEB" "$REPO_DIR/"
cd "$REPO_DIR"

# Packages metadata
apt-ftparchive packages . > Packages
gzip -9kc Packages > Packages.gz

# Release ファイル (SHA256 行は手作りで生成)
{
  echo "Origin: infra-lessons"
  echo "Label: infra-lessons"
  echo "Suite: stable"
  echo "Codename: stable"
  echo "Date: $(date -Ru)"
  echo "Architectures: all"
  echo "SHA256:"
  for f in Packages Packages.gz; do
    h=$(sha256sum "$f" | cut -d' ' -f1)
    s=$(stat -c%s "$f")
    echo " $h $s $f"
  done
} > Release

# 署名 (分離型 + 埋め込み型の両方)
rm -f Release.gpg InRelease
gpg --default-key "$SIGNER_EMAIL" --batch --yes -abs -o Release.gpg Release
gpg --default-key "$SIGNER_EMAIL" --batch --yes --clearsign -o InRelease Release

# クライアント配布用の公開鍵をエクスポート
gpg --export --armor "$SIGNER_EMAIL" > infra-lessons-repo.pub

echo "=== Built repo ==="
ls -l
EOF

multipass exec mgmt -- chmod +x /home/ubuntu/build-repo.sh
```

実行:

```bash
multipass exec mgmt -- /home/ubuntu/build-repo.sh
```

最後の `ls -l` 出力に以下が並んでいれば成功:

- `hello-infra_1.0.0_all.deb`
- `Packages`, `Packages.gz`
- `Release`, `Release.gpg`, `InRelease`
- `infra-lessons-repo.pub`

### 5. リポジトリ一式を cds01 に配布

`cds01` の `/var/www/html/repo/` 配下に置く。`tar` でまとめて転送する。

```bash
multipass exec cds01 -- sudo mkdir -p /var/www/html/repo
multipass exec mgmt -- tar -C /home/ubuntu -cf - repo | multipass exec cds01 -- sudo tar -C /var/www/html -xf -
multipass exec cds01 -- sudo chown -R www-data:www-data /var/www/html/repo
multipass exec cds01 -- ls -l /var/www/html/repo/
```

`mgmt:~/repo/` の中身がそのまま `cds01:/var/www/html/repo/` に展開される。

### 6. nginx のサイト設定に autoindex を追加（任意・デバッグ用）

`/repo/` を **ブラウザや curl でファイル一覧が見える** ようにしておくと、後でデバッグが楽。

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

    location /repo/ {
        autoindex on;
    }
}
EOF

multipass exec cds01 -- sudo nginx -t
multipass exec cds01 -- sudo systemctl reload nginx
```

> `access_log` 行は前ステップで mtls フォーマットを設定済みの場合のみ。設定していなければその1行を削除。

### 7. client から mTLS 越しにリポジトリを取得できるか確認

まず **トップディレクトリ**:

```bash
multipass exec client -- sudo curl --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/repo/
```

autoindex で生成された HTML（ファイル一覧）が返ってくるはず。`hello-infra_1.0.0_all.deb`、`Release`、`Packages.gz` などが見えればOK。

次に **個別のファイル**:

```bash
multipass exec client -- sudo curl -o /dev/null -w "HTTP %{http_code}\n" --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/repo/Release
multipass exec client -- sudo curl -o /dev/null -w "HTTP %{http_code}\n" --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/repo/Packages.gz
multipass exec client -- sudo curl -o /dev/null -w "HTTP %{http_code}\n" --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/repo/hello-infra_1.0.0_all.deb
```

3 つすべて `HTTP 200` であれば、**リポジトリのファイルが mTLS 越しに取得できる状態** が完成。

### 8. 中身が正しいかも軽く確認

`Release` の中身が GPG 署名されていて、SHA256 が並んでいるか:

```bash
multipass exec client -- sudo curl -s --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/repo/Release
```

`Origin: infra-lessons` から始まり、`SHA256:` の下に Packages / Packages.gz のハッシュが並んでいる。

`InRelease` は同じ内容に GPG 署名が埋め込まれた版:

```bash
multipass exec client -- sudo curl -s --cert /etc/ssl/client01.crt --key /etc/ssl/client01.key https://cds01/repo/InRelease | head -30
```

`-----BEGIN PGP SIGNED MESSAGE-----` で始まっていれば、署名が埋め込まれた形式になっている。

---

## ここまでで何ができたか

- 自前で **本物の `.deb`** を作った
- それを **GPG 署名つきの apt リポジトリ** にまとめた
- そのリポジトリを **mTLS で守られた HTTPS 経由でしか取れない状態** で公開した

「鍵を持っている契約者だけが、改ざんされていない署名付きパッケージを取得できる」 ─ RHUI の本質的な機能が、自分で組み立てたインフラの上で動いている。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `gpg: command not found` | `multipass exec mgmt -- sudo apt-get install -y gnupg` |
| `apt-ftparchive: command not found` | `multipass exec mgmt -- sudo apt-get install -y apt-utils` |
| `gpg: secret key not found` | 鍵生成が失敗している。`multipass exec mgmt -- gpg --list-secret-keys` で確認。なければ手順2を再実行 |
| `curl: (60) SSL certificate problem` | `update-ca-certificates` が済んでいるか確認（step-05 参照） |
| `curl: (35) error:1408F119` 等 | mTLS 設定が壊れている可能性。step-06 の動作確認を先にやる |
| 404 で `Release` が取れない | `tar` 転送が失敗していた可能性。手順5を再実行して `ls -l /var/www/html/repo/` で中身を確認 |

---

## 次のステップ

リポジトリの **配信側** は揃った。次は client 側で:

- `/etc/apt/sources.list.d/` にリポジトリを登録
- mTLS 用の cert/key を apt に教える設定
- GPG 公開鍵を信頼ストアに追加
- `apt update` & `apt install hello-infra` まで通す

これで「ミニ RHUI が機能として一通り動く」状態が完成する。
