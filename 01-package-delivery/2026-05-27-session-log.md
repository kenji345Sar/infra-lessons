# 2026-05-27 ミニRHUI: 自前 apt repo の構築と配信

**対象ステップ:** [step-07-build-apt-repo.md](step-07-build-apt-repo.md) + [step-08-client-apt-install.md](step-08-client-apt-install.md)

`01-package-delivery` の step-07 + step-08 を完走。`client` から `apt install hello-infra` が通り、自作の `.deb` パッケージを mTLS 越しに配信できる状態を作った。

## 達成したこと

- `mgmt` で `.deb` を作り、GPG 署名つき flat apt repo を生成
- `cds01` の nginx で `https://cds01/repo/` を mTLS 配信
- `client` から apt 経由でインストールと実行に成功
  ```
  $ multipass exec client -- hello-infra
  Hello from the infra-lessons private package!
  ```

## 4 つの登場人物（役割と置いた鍵）

| 役割 | VM | 持つもの |
|---|---|---|
| 管理ノード | `mgmt` | GPG 秘密鍵（パッケージ署名用） |
| 配信ノード | `cds01` | サーバ証明書 `cds01.crt` + 秘密鍵 |
| クライアント | `client` | クライアント証明書 `client01.crt` + 秘密鍵 |
| CA（認証局） | （mgmt 上で自作） | CA 秘密鍵（証明書を発行するハンコ） |

CA の証明書 `infra-lessons-ca.crt` は **両方の VM** に置かれていて、用途が逆向き:
- `cds01`: 「届いた client 証明書がウチの CA で署名されているか」を検証
- `client`: 「届いた cds01 サーバ証明書がウチの CA で署名されているか」を検証

## ステップごとに何が動いていたか

### step-07 リポジトリ構築

- `apt-ftparchive packages .` → `Packages` メタデータ生成
- `gzip -9kc Packages > Packages.gz` → 圧縮版
- `Release` ファイルを自分で組み立て（`SHA256:` の下に各ファイルのハッシュとサイズ）
- `gpg --clearsign` → `InRelease`（署名埋め込み版）
- `gpg -abs` → `Release.gpg`（分離署名版）
- `gpg --export --armor` → `infra-lessons-repo.pub`（クライアント配布用公開鍵）

これらを `tar` で `mgmt → cds01` に転送し、`/var/www/html/repo/` 配下に展開。

### step-08 クライアント側設定

- 公開鍵を curl で取得 → `gpg --dearmor` でバイナリ形式に変換 → `/etc/apt/keyrings/infra-lessons-repo.gpg`
- `/etc/apt/apt.conf.d/99-cds01-mtls.conf` で `Acquire::https::cds01::SslCert/SslKey/CaInfo` を設定（**cds01 宛のときだけ** mTLS が発動する作り）
- `/etc/apt/sources.list.d/infra-lessons.list` に `deb [signed-by=...] https://cds01/repo/ ./`（flat repo を意味する ` ./`）
- `apt update` → `InRelease`/`Packages` 取得 → GPG 検証 → SHA256 照合
- `apt install hello-infra` → `.deb` 取得 → 展開

## つまずきポイントと原因

### 1. ヒアドキュメントが閉じない（`[EOF]` 事故）

- 原因: ターミナルの **bracketed paste mode**。ペースト時に挿入される `\e[200~` / `\e[201~` が、`zsh` 側で解釈されずに `[` `]` として heredoc 本文に混入し、終端 `EOF` が一致しなくなる。
- 対症: `printf '\e[?2004l'` でモードを切る。ただし次のプロンプトで戻ることがある。
- 恒久対症: `multipass transfer` でローカルファイルを VM に置く方式に切り替えた。heredoc を回避すれば再発しない。

### 2. mTLS ログフォーマット `mtls` が未定義

- 原因: step-06 で定義するはずの `/etc/nginx/conf.d/mtls-log.conf` を入れていなかった。`access_log ... mtls;` だけ書くと `unknown log format` で nginx 起動に失敗。
- 対症: `log_format mtls '...';` を `/etc/nginx/conf.d/mtls-log.conf` として追加。

### 3. `multipass restart client` が SSH タイムアウト

- 原因: `multipass list` は `Running` を返すが、VM 内部の SSH エージェントが応答しなくなっていた。`restart` は SSH 経由で操作するので失敗する。
- 対症: `multipass stop --force client` → `multipass start client`。

### 4. `/etc/hosts` から `cds01` のエントリが消えた

- 原因: cloud-init の `manage_etc_hosts` が VM 起動時に `/etc/hosts` を書き戻す可能性がある。step-03 でこれを無効化する設定があるが、`client` に効いていなかった。
- 対症: `cds01 → 192.168.252.4` を `/etc/hosts` に再追記。`manage_etc_hosts: false` を入れる恒久対策は要確認。

### 5. apt が client 証明書を読めない（`Error while reading file`）

- 原因: apt の HTTP 取得モジュールは、デフォルトで `APT::Sandbox::User`（通常 `_apt`）にユーザを切り替えて走る。root しか読めない `/etc/ssl/client01.key`（`600 root:root`）を、`_apt` ユーザは読めない。
- 観察: `curl` を `sudo` で叩いた時は root なので成功するが、apt は内部で権限を落とすため失敗する。
- 対症: `/etc/apt/apt.conf.d/99-no-sandbox.conf` に `APT::Sandbox::User "root";` を入れて sandbox を無効化（学習環境のため許容）。
- 本番ならどうするか: クライアント鍵を `_apt` グループに読ませる（`chgrp _apt`, `chmod 640`）か、ACL で限定的に読ませる。

## 用語の混乱しがちな点

- **CDS** (Content Delivery Server, RHUI 用語) と **CDN** (Content Delivery Network, 一般用語) は別物。`cds01` の名前は前者から来ている。
- **CA** はクライアント環境ではなく独立した「証明書を発行する役割」。本演習では mgmt 上で自作している。Let's Encrypt 等に相当。
- **配信ノード** という用語は Web 一般ではなく RHUI 特有。「作る側 (mgmt)」と「配る側 (cds)」を分けるのが RHUI の構成。

## 学習方針メモ

このlesson以降の進め方として、自分の中で意識しておきたい点:

- 手順を通すだけで終わらせず、**裏で何が起きているか**（開かれるファイル・起動するプロセス・流れるバイト列・権限境界）を理解する。
- アプリ層のプログラミングは経験があるが、低レイヤのソースを追ってこなかったので、コマンドや conf が「いきなり出てくる」感覚がある。観察手段（`strace` / `tcpdump` / `apt -o Debug::...` / `journalctl` / `ps`）をセットで覚える。
- 仕掛けの解説は手順本編 (`step-XX.md`) と独立させた `appendix-*.md` に書く。読み返す時に役割を分離する。

## 深堀りしたい仕掛け（後日 appendix で）

| トピック | 解説する仕掛け |
|---|---|
| A. mTLS の握手の中身 | クライアント証明書がいつ・どのバイトで送られるか、tcpdump で見て nginx 側のログと突き合わせる |
| B. apt がパッケージを取りに行く順序 | InRelease → Release → Packages.gz → .deb の取得とハッシュ照合を `apt -o Debug::Acquire::https=true update` で観察 |
| C. nginx がリクエストを処理する流れ | `strace` で nginx ワーカーがどのファイルを開くか、`access_log` の各フィールドが何由来か |
| D. systemd の reload で起きていること | `nginx -s reload` と `systemctl reload nginx` の違い、master/worker プロセスの入れ替わり |
| E. apt のサンドボックス機構 | `_apt` ユーザ、権限ドロップの実装、`/usr/lib/apt/methods/` の各メソッドプロセス |

## 次のステップ

- 配信ノード 2 台目 (`cds02`) を立てて同期
- HAProxy 経由で `cds01` / `cds02` に負荷分散
- 片方を落としても client から見えるか
- ここまでを Ansible で再現可能化
