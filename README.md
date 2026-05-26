# infra-lessons

インフラ層の構築・運用を、最小構成のサンプルで 1 つずつ手を動かして学ぶ。

## 位置づけ

- `security-lessons` は **アプリケーション寄り** のセキュリティを扱う。
- `infra-lessons` は **インフラ寄り** の構築・運用を扱う。
- 両者は別プロジェクトとして進め、後で俯瞰してレイア全体を繋げる。

レイアの境界と、同じ「サーバ／クライアント」が別物を指す件は [notes/app-vs-infra.md](./notes/app-vs-infra.md) に整理。

## 進め方

1 題材につき以下のセットで進める。

1. **構成図** — VM / プロセス / 通信経路を図で示す
2. **構築手順** — コマンドと設定ファイルで再現可能にする
3. **動作確認** — 正常動作だけでなく「壊れるとどう失敗するか」まで見る

## 全体像（予定）

| 層 | 扱うテーマ | 主なツール |
|---|---|---|
| パッケージ配信 | リポジトリ運用、署名、配信認証 | createrepo, openssl, HAProxy |
| ロードバランシング | L4/L7 分散、ヘルスチェック、SSL終端 | HAProxy, nginx |
| 構成管理 / IaC | 冪等な構築、Playbook、インベントリ | Ansible |
| 監視・ログ | メトリクス、ログ集約 | （未定） |
| 認証基盤 | PKI、ディレクトリ | openssl, OpenLDAP |

層は固定せず、扱った順に番号を振っていく。

## 環境前提

- macOS (Apple Silicon)
- VM: **Multipass**（コマンド 1 発で Ubuntu VM が立つ）
- ゲスト OS: **Ubuntu 22.04**

RHEL 系 (AlmaLinux など) でも本質は同じだが、Multipass は Ubuntu 専用に近いツールでセットアップが軽い。本プロジェクトでは「リポジトリ・署名・配信・LB・IaC」の概念を学ぶことが目的で、RPM/DEB の差は本質ではないので Ubuntu で進める。

## 用語

知らない用語は [GLOSSARY.md](./GLOSSARY.md) を参照。

## フォルダ構成（予定）

```
infra-lessons/
├── README.md             # このファイル
├── GLOSSARY.md           # 用語集
├── notes/                # 横断メモ（レイア俯瞰、設計判断など）
│   └── app-vs-infra.md
└── 01-package-delivery/  # 自前パッケージ配信（ミニ RHUI）
```

以降の層は、扱い始めたタイミングで追加する。

## 由来

きっかけは RHUI (Red Hat Update Infrastructure) v4→v5 構築案件の検討。実機 RHUI は有償サブスクリプションが必要で個人では構築できないため、**同等構成をライセンス不要のツールで最小規模に再現**することを最初の題材とする。
