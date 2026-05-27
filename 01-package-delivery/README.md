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

## 全体ステップ

1. VM 4 台 を Multipass で用意する
2. 自作 CA を作り、サーバ証明書とクライアント証明書を発行する
3. 管理ノードで RPM を GPG 署名し、`createrepo_c` でリポジトリを生成する
4. 配信ノード 2 台に同期し、HTTPS + クライアント証明書認証で公開する
5. HAProxy で配信ノード 2 台に負荷分散する
6. クライアントから `dnf install` / `dnf update` が通ることを確認する
7. ここまでを Ansible Playbook 化して再現可能にする

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
