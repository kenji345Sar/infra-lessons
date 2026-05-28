# プロセス × プロセス間通信 という共通フレーム

「HTTP」「TLS」「ACP」「LSP」「MCP」── 別物に見えるが、**全部「プロセスが別のプロセスと会話する」話** という同じフレームに入る。この切り口に気づくと「何をしているのか」が見えやすくなる。

## 共通の構造

すべてのソフト（エディタ、エージェント、nginx、curl、データベースクライアント …）は OS 上の **プロセス**。何かの言語で書かれたコードを OS が動かしているという点では全部同じ。

プロセスは単独では完結しない。多くの場合、別のプロセス（同じマシン内 or 別マシン）と会話する必要がある。その会話の取り決めが **プロトコル**。

```
[ プロセス A ] ←─ プロトコル ─→ [ プロセス B ]
   (何かの言語で                    (何かの言語で
    書かれたコード)                   書かれたコード)
       │                              │
       └──────── OS が動かしている ─────┘
```

## 違って見えるのは「運び方」と「中身の形」が違うから

プロセス間通信は、**運び方 (transport)** と **中身の形 (wire format)** の組み合わせで決まる。プロトコルごとに違うのはここだけ。

| プロトコル | 運び方 (transport) | 中身 (wire format) | 会話相手 |
|---|---|---|---|
| HTTP | TCP socket | HTTP テキスト | 別マシン or 同マシンの nginx |
| HTTPS | TCP + TLS | HTTP テキスト（TLS で包む） | 同上（暗号化される） |
| **ACP** | **stdio パイプ（親子プロセス）** | **JSON-RPC 2.0** | **同マシンの Agent プロセス** |
| LSP | stdio パイプ | JSON-RPC 2.0 | 同マシンの言語サーバー |
| MCP (local) | stdio パイプ | JSON-RPC 2.0 | 同マシンの MCP サーバー |
| MCP (remote) | HTTP + SSE / WebSocket | JSON-RPC 2.0 | 別マシンの MCP サーバー |
| PostgreSQL wire | TCP socket | PostgreSQL 独自バイナリ | 同 or 別マシンの postgres プロセス |

「全部違って見える」のは、運び方と中身の形が違うから。**構造は全部同じ**。

## HTTPS と ACP を並べて見る

infra-lessons step-05〜06 でやった HTTPS と、Software Design 2026/6 月号の ACP は、同じフレームの別バリエーション:

| | HTTPS（step-05〜06） | ACP（Software Design 2026/6） |
|---|---|---|
| 関わるプロセス | curl と nginx | Zed エディタと Agent |
| 物理的にどこ | 別マシン（VM） | 同じマシン（親子プロセス） |
| 運び方 | TCP の 443 ポート | stdio パイプ |
| 中身 | TLS で包んだ HTTP テキスト | JSON-RPC メッセージ |

構造は完全に同じ ── **2 つのプロセスが約束ごと（プロトコル）に従って会話している**。違うのは「運び方」と「中身の形」だけ。

## 4 切りもそのまま当たる

[learning-by-slicing.md](learning-by-slicing.md) の 4 切り（A プロトコル / B パケット / C 設定の意味 / D 実装）は、HTTP 以外のプロトコルにもそのまま使える:

|  | ACP | HTTPS |
|---|---|---|
| A. プロトコル | ACP 仕様（Zed 公開） | RFC 8446 + 9110 |
| B. パケット | JSON-RPC メッセージ列 | TLS records + HTTP テキスト |
| C. 設定の意味 | エディタ側で agent コマンドを登録 | nginx の `ssl_*` |
| D. 実装 | Zed (Rust) + ACP SDK + Deep Agents (TS) | OpenSSL + nginx + curl |

新しいプロトコルに出会ったときは、まず A〜D の 4 つの欄を埋めにいくと、その正体に近づける。

## このフレームに気づくと何が変わるか

- 「エディタ」「エージェント」「サーバ」「クライアント」も、特別な存在ではなく **すべてプロセス** として等価に扱える
- 新しいプロトコル（MCP, ACP, A2A, …）が出てきても「**運び方は何? 中身は何? 会話相手はどこのプロセス?**」の 3 問で位置付けられる
- 「OS 上で動く」「言語で動く」という前提は全プロセス共通なので、そこは差別化要因にならない
- 違いは結局「**プロセス間通信の手段の違い**」に帰着する

## 関連

- アプリ層とインフラ層の対比は [app-vs-infra.md](app-vs-infra.md)
- 「設定 / コマンド / プロトコル / 実装」の 4 切りは [learning-by-slicing.md](learning-by-slicing.md)
- HTTPS（TLS + HTTP）の具体は [../01-package-delivery/](../01-package-delivery/) の step-04〜06
