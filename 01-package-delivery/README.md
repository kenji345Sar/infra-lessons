# 01-package-delivery

自前のパッケージ配信基盤を最小構成で構築する。RHUI (Red Hat Update Infrastructure) の同等構成を、ライセンス不要のツールで再現する。

## 用語

知らない用語が出てきたら [../GLOSSARY.md](../GLOSSARY.md) を参照。

## 扱うテーマ

- パッケージリポジトリの作成と GPG 署名
- 配信サーバの HTTPS 化
- クライアント証明書による配信先認証（誰に配るかを制御）
- ロードバランサ前段配置による複数配信サーバへの分散
- Ansible Playbook による再現可能化

## 構成図

```
[ クライアント VM ] ── HTTPS + クライアント証明書 ──┐
                                                    │
                                            [ HAProxy VM ]
                                                    │
                                          ┌─────────┴─────────┐
                                          ▼                   ▼
                                  [ 配信ノード #1 ]    [ 配信ノード #2 ]
                                  (yum repo 配信)     (yum repo 配信)
                                          ▲                   ▲
                                          └──── 同期 ─────────┘
                                                    │
                                          [ 管理ノード VM ]
                                          (リポジトリ生成・署名)
```

## RHUI との対応

| 本演習 | RHUI v5 |
|---|---|
| 管理ノード | RHUA (Red Hat Update Appliance) |
| 配信ノード | CDS (Content Delivery Server) |
| HAProxy VM | HAProxy（RHUI v5 同梱） |
| 自作 CA + クライアント証明書 | エンタイトルメント証明書 |

## 管理ノード / 配信ノードという分け方について

「管理ノード」「配信ノード」は **RHEL/RHUI 特有の用語ではなく、役割で分けるアーキテクチャ思想** の話。OS で物理的に分かれているわけでもなく、論理的な役割分担。

### どこにでもある分割

| エコシステム | 管理ノード相当 | 配信ノード相当 |
|---|---|---|
| RHEL (RHUI) | RHUA | CDS |
| Debian/Ubuntu | apt repo 構築サーバ（`aptly` / `reprepro` を動かす） | apt repo mirror（nginx で `/repo/` を公開） |
| 一般 | Pulp / Spacewalk / Satellite / Nexus / Artifactory のビルド側 | 同左の配信側 / CDN / mirror |

本演習でも、`mgmt` が「管理」、`cds01` / `cds02` が「配信」で、やっていることは **Ubuntu 上の nginx + aptly**。RHEL でなくても成立する。

### なぜ分けるのか（OS 的な必然ではない）

- **秘密鍵を外に出したくない**: GPG 秘密鍵・CA 秘密鍵は管理側だけに置く。配信側が侵害されても署名は奪われない
- **配信側をステートレスにして水平スケールしたい**: 配信ノードは「同じファイルを置いた nginx」を何台でも並べられる（HAProxy で振る）
- **ネットワーク境界が違う**: 管理は内部ネットワーク、配信はクライアント向けに公開
- **責務が違う**: 作る人と渡す人を分けると運用が単純になる

### 物理 / OS との関係

- 1 台に同居させても動く（学習や検証ならアリ）
- 本番では VM / コンテナ / 物理を分けるのが普通
- OS を揃える必要はない。`mgmt` が Ubuntu、`cds01` が RHEL でも問題ない（配信するパッケージ形式と、配信サーバの OS は別の話）

つまり「管理 / 配信」は **RHUI の専門用語ではあるが、思想は一般的**。RHUI が特殊なのは、その役割に RHUA・CDS という製品名を付けて Red Hat が売っている、という点だけ。

## なぜこれを学ぶか — 現実のサプライチェーン攻撃との対応

step-04〜07 で扱う「自作 CA + mTLS + GPG 署名」は、**現実のサプライチェーン攻撃を技術で防ぐための最小モデル** になっている。学習用のお遊びではない。

### 例: Trivy 改ざん事件（2026 年 3 月）

OSS の脆弱性スキャナ Trivy の `@v0.34.2` タグが force push で悪意あるコードに差し替えられ、Trivy を使う CI/CD パイプラインから AWS / GCP のクラウドキー、SSH キー、Kubernetes トークンなどが盗まれた事件（Software Design 2026 年 6 月号「押し上げろ！サイバーセキュリティ！」第 2 回より）。

この事件で起きたことを、本演習の各 step に対応させる:

| 守るべきもの | Trivy 事件で起きたこと | 本演習で対応する仕組み |
|---|---|---|
| 配布物の改ざん検出 | `@v0.34.2` タグが force push で悪意あるコードに差し替えられた | **step-07 の GPG 署名**（`Release.gpg`）。タグではなく署名で検証する |
| 配布元の身元保証 | 攻撃者が改ざんした trivy-action を「正規」と誤認した | **step-04〜05 の自作 CA + サーバ証明書** |
| 配布先の限定 | （事件外）契約者以外への流出抑止 | **step-06 の mTLS（クライアント証明書認証）** |

各 step で「なぜこの設定をするのか」を腑に落とすには、**防ぎたい攻撃と紐付けて考える** と効きやすい。

### 4 切りの C（設定の意味）でも繋がる

記事より:

> `@v0.35.0` を利用している方は安全でした。このような違いは、**設定上の堅牢さの違い** などから生まれています。

`uses: aquasecurity/trivy-action@v0.34.2` のタグ指定の仕方（タグ vs SHA ピン留め vs 署名検証）の差が、攻撃の成否を分けた。step-06 の `ssl_verify_client on;` が ON か OFF かで配信成立が変わるのと、**構造は同じ**（[../notes/learning-by-slicing.md](../notes/learning-by-slicing.md) の C）。

## 全体ステップ

1. VM 4 台 を Multipass で用意する
2. 自作 CA を作り、サーバ証明書とクライアント証明書を発行する
3. 管理ノードで RPM を GPG 署名し、`createrepo_c` でリポジトリを生成する
4. 配信ノード 2 台に同期し、HTTPS + クライアント証明書認証で公開する
5. HAProxy で配信ノード 2 台に負荷分散する
6. クライアントから `dnf install` / `dnf update` が通ることを確認する
7. ここまでを Ansible Playbook 化して再現可能にする

## 各ステップの内容

step-01 〜 step-08 の内訳。大きく **基盤層 (01–03) → 暗号層 (04–06) → 配信層 (07–08)** の 3 段で積み上げる。

| Step | 全体ステップ | やっていること | 目的 |
|---|---|---|---|
| [step-01](step-01-vm-setup.md) | 1 の準備 | Multipass を入れて Ubuntu VM を 1 台起動・shell・stop | VM の基本操作を体に入れる |
| [step-02](step-02-launch-four-vms.md) | 1 | 役割名つき VM を 4 台起動（mgmt / cds01 / cds02 / client）、ping で疎通 | サーバ間通信の土台 |
| [step-03](step-03-hostname-resolution.md) | 1 の補助 | 各 VM の `/etc/hosts` を整え、ホスト名で呼べるようにする | IP 変化に耐える参照（本番 RHUI も DNS 名前提） |
| [step-04](step-04-self-signed-ca.md) | 2 | `mgmt` に自作 CA を立て、サーバ証明書/クライアント証明書を発行 | 後段の HTTPS / mTLS の素材を作る |
| [step-05](step-05-https-server.md) | 4 の前半（HTTPS 化） | `ca.crt` を配布し cds01 に nginx で HTTPS を立て、client から `curl https://cds01/` 成功 | TLS ハンドシェイクが実際に成立する瞬間を確認 |
| [step-06](step-06-client-cert-auth.md) | 4 の後半（クライアント証明書認証） | cds01 の nginx を mTLS 必須に変更（証明書なしは拒否） | 「契約者しか取りに来られない」配信の核 |
| [step-07](step-07-build-apt-repo.md) | 3 ＋ 4 の公開部分 | `mgmt` で `.deb` を作り、GPG 署名つき flat apt repo を作って cds01 で配信、`curl --cert` で取得可 | リポジトリの実体（モノ）を mTLS 越しに置く |
| [step-08](step-08-client-apt-install.md) | 6 | client 側に GPG 公開鍵と `sources.list` を登録し、`apt update` → `apt install hello-infra` が通る | apt の世界からエンドツーエンドで利用できる状態 |

> 全体ステップ 5（HAProxy 負荷分散）と 7（Ansible 化）、および 4 の cds02 への同期は未着手。

## http-from-scratch との関係

`curl -v https://cds01/` を打ったとき、curl は次の順番で動く。**両プロジェクトはこのスタックの隣り合った層を扱っている**。

```
1. DNS / hosts で cds01 を解決               ← infra-lessons step-03
2. TCP で 443 に接続                          ← http-from-scratch step1 (の TLS 版)
3. TLS ハンドシェイク (ca.crt で検証)         ← ★ infra-lessons step-05
4. クライアント証明書を提示                   ← ★ infra-lessons step-06
5. TLS の中で HTTP リクエストを送信           ← http-from-scratch step1〜6 の中身
6. HTTP レスポンスを受け取る                  ← 同上
```

curl の `-v` 出力で `*` 始まりの行（TLS）と `>` `<` 始まりの行（HTTP）の両方が見えるのは、curl がこの両方を扱っているから。

### 何が違うか: 「自分で書く層」が違う

|  | http-from-scratch | infra-lessons step-04〜06 |
|---|---|---|
| **HTTP** | 自分で実装する (step1〜6) | nginx 任せ |
| **TLS** | 扱わない | nginx の設定として触る（自分で実装はしない） |
| **証明書 (ca.crt 等)** | 出てこない | 自分で発行・配布する (step-04) |

- http-from-scratch: **HTTP を中から見る**（socket からテキストを読む）
- infra-lessons step-04〜06: **TLS を外から組み立てる**（nginx を設定し、証明書を配る）

### 対応関係の整理

しいて対応を取るなら、http-from-scratch **step1**（socket で TCP を直接読む）の TLS 版が、infra-lessons **step-05**（TCP の上に TLS を載せて nginx で待ち受ける）。ただし「自分で書く」レベルが違うので **等価ではない**。

つまり両者は **同じスタックの隣の層**であり、「相当する」ではない。

- 同じスタック → curl で両方観察できる
- 違う層 → 一方の知識でもう一方を代替できるわけではない

## 動作確認の観点

各ステップ後に、正常系だけでなく「壊れるとどう失敗するか」を見る。

- クライアント証明書を提示せずにアクセスしたら、サーバはどう拒否するか
- GPG 署名が一致しないパッケージを置いたら、`dnf` はどう拒否するか
- 配信ノードの片方を落としたら、HAProxy はどう振り分けるか
- 管理ノードと配信ノードの同期が止まったら、クライアントには何が見えるか

## 環境

- VM: Multipass（コマンド 1 発で Ubuntu VM を立てられる）
- ゲスト OS: Ubuntu 22.04
- ツール: `aptly`（リポジトリ生成）, `gnupg`（パッケージ署名）, `openssl`（証明書発行）, `haproxy`（負荷分散）, `ansible`（構成管理）

> パッケージ系は本来 RHEL 系の `createrepo_c` を使うが、Ubuntu では `aptly` で同等のことができる。学ぶ概念（リポジトリ・署名・配信・認証）は同じ。

## 進捗

- step-01 〜 step-08 完了。`client` から `apt install hello-infra` が成功し、mTLS 越しに自前 apt repo を配信できる状態。
- 作業ログ: [2026-05-27-session-log.md](2026-05-27-session-log.md)
- 残: cds02 の同期、HAProxy での負荷分散、Ansible 化。
