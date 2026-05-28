# 暗号実装はどこから来るか

「TLS で守られている」「JWT で署名している」── 実際の **暗号処理は誰がどこで実装したコード** が動いているのか、という視点。インフラ・セキュリティを判断するときの土台になる。

## 構造: OpenSSL は道具箱、TLS / 認証・認可はその利用者

OpenSSL は暗号技術を **全部入り** で提供するライブラリ。暗号化・ハッシュ・署名・HMAC・鍵交換・乱数生成がまとめて入っている。

```
[ OpenSSL（道具箱・ライブラリ） ]
       │ 提供する
       ├── 暗号化 (AES, ChaCha20, RSA …)
       ├── ハッシュ (SHA-256, SHA-3 …)
       ├── 署名 (RSA-PSS, ECDSA, EdDSA …)
       ├── HMAC (HMAC-SHA256 …)
       ├── 鍵交換 (ECDH, X25519 …)
       └── CSPRNG
              │ 必要な道具を呼び出す
              ▼
  [ TLS / 認証・認可 / パスワード保存 / JWT 等 ]
```

認証・認可は OpenSSL とは **対立関係ではなく包含関係**。内部で OpenSSL（または同等のライブラリ）を呼んでいる。

## Linux + 動的リンク = OS の OpenSSL を共有

Linux 系では多くのアプリが OS の OpenSSL を **動的リンク** で共有する:

```
[ OS (Ubuntu 等) ]
   └── /usr/lib/x86_64-linux-gnu/libssl.so.3       ← OS の共有ライブラリ
        ▲
        │ 動的リンクで呼び出す
        │
   ┌────┼─────────────────────────────────────┐
   │    │                                      │
[ nginx ]  [ curl ]  [ Python ssl ]  [ apt の HTTPS ]  ...
```

このため、OpenSSL に脆弱性が出ると **OS のパッケージ更新だけで全アプリに対策が行き渡る**。

## 言語ごとの暗号実装の出所

「OS の OpenSSL を呼ぶ」のが普通とは限らない。言語・ランタイムによって実装の出所が違う。

| 言語 / ランタイム | 暗号の実装元 | OpenSSL の事故に影響を受けるか |
|---|---|---|
| C / C++（直接） | OS の OpenSSL（動的リンク） | ★ 受ける |
| Python 標準 `ssl` | OS の OpenSSL | ★ 受ける |
| Ruby（標準）| OS の OpenSSL | ★ 受ける |
| PHP（標準）| OS の OpenSSL | ★ 受ける |
| nginx | OS の OpenSSL | ★ 受ける |
| **Go** | **標準ライブラリ内蔵 (`crypto/tls`)** | 受けない（自前実装） |
| **Java** | **JCE / SunJSSE 内蔵** | 受けない（自前実装） |
| **.NET (Windows)** | **SChannel（Windows 標準）** | 受けない（別実装） |
| **Node.js** | **OpenSSL を自前バンドル** | 受ける（ただし Node 本体の更新で対応） |
| Rust (`rustls`) | 自前実装 | 受けない |
| Rust (`openssl` crate) | OS の OpenSSL | ★ 受ける |

## Heartbleed の例で「OS 更新が効く / 効かない」

Heartbleed（CVE-2014-0160）は OpenSSL の重大な脆弱性。出たとき何が起きたか:

- **Linux + nginx / Apache / curl**: `apt upgrade libssl*` で対策完了
- **Go で書かれたサーバ**: そもそも影響なし（OpenSSL を使っていない）
- **Java の HTTPS サーバ**: 影響なし（JSSE を使っている）
- **Node.js**: Node.js 自体のバージョンを上げる必要があった
- **静的リンクされた C アプリ**: アプリの再ビルド・再配布が必要

「OpenSSL の脆弱性が出た」ニュースに対して、自分のシステムが **影響を受けるかどうかを判断する** には、上の表のような「**暗号実装がどこから来ているか**」を知っている必要がある。

## OS / 環境ごとの標準も違う

| OS / 環境 | 標準の暗号ライブラリ |
|---|---|
| Linux (Ubuntu / RHEL 系) | OpenSSL |
| macOS | LibreSSL（プリインストール）+ Security framework |
| Windows | SChannel（Windows 標準） |
| Docker（通常イメージ）| ベース OS の OpenSSL |
| Docker（`distroless` / `scratch`）| バイナリにバンドルされたもの |
| ブラウザ | 各ブラウザベンダーの実装（BoringSSL 等） |

クロスプラットフォームで動くアプリは、暗号実装の出所が **環境ごとに違う** ことがある。

## なぜこの視点が重要か

実用上、以下のような判断で効いてくる:

1. **脆弱性対応**: OS 更新で対策が当たる範囲を判断できる
2. **言語選定**: Go / Java は OpenSSL 事故に強い、Python / nginx は OS と運命を共にする
3. **コンテナ運用**: `distroless` や `scratch` イメージで何が変わるか分かる
4. **サプライチェーン**: 自分のシステムの暗号実装がどこから来ているかを答えられないと、攻撃に備えられない（[Trivy 事件](../01-package-delivery/README.md#なぜこれを学ぶか--現実のサプライチェーン攻撃との対応) と同系統）
5. **クロスプラットフォーム**: macOS / Windows / Linux で同じコードを動かしても、暗号の実体が違うことがある

## infra-lessons / 他レッスンとの関係

- **infra-lessons step-04**: `openssl` コマンド = OS の OpenSSL CLI
- **infra-lessons step-05〜06**: nginx が OS の OpenSSL を動的リンクで呼び出して TLS を張っている
- **http-from-scratch step1 の TLS 版（未着手）**: Python の `ssl` モジュール経由で OS の OpenSSL を呼ぶことになる
- **http-from-scratch auth 系**: JWT / bcrypt / セッション ID 生成も内部で OpenSSL（または同等ライブラリ）を呼ぶ
- **security-lessons**: AV / EDR が監視するのは「**プロセスが OpenSSL をどう呼んでいるか**」を含むファイルシステム・ネットワーク操作

「**暗号は自分で書かず、ライブラリに任せる**」が業界の鉄則だが、**そのライブラリが誰のもので、どこにあるか** を知らないと、いざというときに守れない。

## 関連

- 4 切り（A / B / C / D）の D（実装）を深掘りする方向 → [learning-by-slicing.md](learning-by-slicing.md)
- プロセス × プロセス間通信のフレーム → [process-and-protocol.md](process-and-protocol.md)
- TLS の具体（OpenSSL が動いている現場） → [../01-package-delivery/](../01-package-delivery/) の step-04〜06
- サプライチェーン攻撃の話 → [../01-package-delivery/README.md](../01-package-delivery/README.md) の「なぜこれを学ぶか」節
