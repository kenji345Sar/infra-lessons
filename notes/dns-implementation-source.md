# DNS の実装はどこにあるか

「ドメイン名から IP を引く」── 実際に名前解決を行っているコードがどこにあるか、という視点。[crypto-implementation-source.md](crypto-implementation-source.md) の OpenSSL と同じ構造で、**サーバ・クライアント・CLI ・運営層** に分かれている。

## 役割ごとの実装地図

DNS の実装は次の 4 種類に分かれる。

### 1. DNS サーバ（権威サーバ / Authoritative）

ドメインの **正解** を持っている側。「`example.io` の IP は `192.0.2.10` です」と答える。

| 実装 | 特徴 |
|---|---|
| **BIND** | 最古参のリファレンス実装。今も標準的 |
| **NSD** | 権威専用、高速、高負荷向け（ルート / TLD サーバで使われる） |
| **Knot DNS** | CZ.NIC 開発、権威専用 |
| **PowerDNS Authoritative** | DB バックエンド対応 |
| **CoreDNS** | Kubernetes 標準。Go 製、プラグインで拡張 |

### 2. DNS サーバ（リゾルバ / キャッシュ）

クライアントの問い合わせを受けて、ルート → TLD → 権威サーバ と辿って答えを返す側。

| 実装 | 特徴 |
|---|---|
| **Unbound** | 軽量・高速、リゾルバ専用 |
| **dnsmasq** | 小規模 LAN・家庭用ルータ・コンテナで多用 |
| **BIND**（リゾルバモード）| サーバとリゾルバ両対応 |
| **PowerDNS Recursor** | 大規模 ISP 向け |
| **systemd-resolved** | systemd 系 Linux のデフォルト |

### 3. DNS クライアント側のリゾルバ（スタブ）

アプリが「名前を引いて」と頼む先。**OS や言語ライブラリの中** にある。

| 環境 | 実装 |
|---|---|
| Linux (glibc) | `getaddrinfo()` / `gethostbyname()` |
| Alpine (musl) | musl の resolver |
| macOS | `mDNSResponder` / `dns-sd` |
| Windows | DNS Client service (Dnscache) |
| Python | `socket.getaddrinfo()` → OS の resolver を呼ぶ |
| Go | 標準ライブラリ内蔵（OS の resolver を使わない選択肢あり） |

OpenSSL のときと同じく、**OS のリゾルバを呼ぶか / 自前で実装するか** が言語・ランタイムによって違う。

### 4. CLI ツール（自分で叩いて確認する）

| ツール | 由来 |
|---|---|
| `dig` | BIND 同梱、最もよく使われる |
| `nslookup` | BIND / Windows 標準 |
| `host` | BIND 同梱、簡易 |
| `drill` | LDNS 同梱、`dig` の代替 |
| `kdig` | Knot 同梱 |
| `resolvectl` | systemd-resolved |

`dig example.io A +short` でその場で名前解決を試せる。

## プロトコル拡張系（暗号化 DNS / 完全性検証）

| プロトコル | 用途 | 代表実装・提供者 |
|---|---|---|
| **DNS over TLS (DoT)** RFC 7858 | 問い合わせを暗号化 | Cloudflare 1.1.1.1, Quad9, Unbound, dnsmasq |
| **DNS over HTTPS (DoH)** RFC 8484 | 問い合わせを HTTPS 上で運ぶ | Cloudflare, Google 8.8.8.8, Firefox 内蔵 |
| **DNSSEC** | 応答に署名して改ざん検出 | BIND / Unbound / Knot |

DoT / DoH は内部で OpenSSL 等の TLS ライブラリを呼ぶので、[crypto-implementation-source.md](crypto-implementation-source.md) の世界と直接繋がる。

## 4 切りに当てる

| 切り方 | DNS での実体 |
|---|---|
| **A. プロトコル** | RFC 1034 / 1035（基本）、RFC 7858（DoT）、RFC 8484（DoH）など |
| **B. パケット** | UDP 53 / TCP 53 上の DNS メッセージ。`tcpdump -i any port 53` や Wireshark で観察可 |
| **C. 設定** | `/etc/resolv.conf`（クライアント）、`named.conf`（BIND）、zone file（権威ゾーン）|
| **D. 実装** | BIND / Unbound / dnsmasq / CoreDNS など |

## 運営層（実装ではないが、知っておくと判断に効く）

| 層 | 何 | 例 |
|---|---|---|
| **IANA / ICANN** | ルート DNS と TLD 全体の管理 | root server / TLD 配分 |
| **TLD レジストリ** | 個別 TLD の運営 | Verisign（.com）、JPRS（.jp）、Identity Digital（.io 等）|
| **レジストラ** | ユーザーへのドメイン販売 | GoDaddy、Cloudflare Registrar、さくらインターネット |
| **DNS ホスティング** | 権威サーバの運営代行 | Route 53（AWS）、Cloud DNS（GCP）、Cloudflare DNS |

技術より制度の話だが、`.io` 廃止リスクのような議論はこの層で起きる。

## infra-lessons との繋がり

- **step-03** の `/etc/hosts` = DNS の **ローカル置き換え版**（DNS を引かずにファイルで解決）
- 「本物の DNS をミニ構成で作る」拡張なら、`mgmt` VM に **dnsmasq か BIND を立てて、cds01 / cds02 / client のゾーン** を持たせる方向が自然
- 現状の演習には入っていないが、step-03 の延長線上に置ける

## なぜこの視点が重要か

- **名前解決の失敗デバッグ**: アプリがエラーを出したとき、どの層（クライアントのスタブリゾルバ / OS のキャッシュ / フォワーダ / 権威サーバ）で詰まっているかを切り分けられる
- **DNS の設定変更**: `/etc/resolv.conf` を変える、systemd-resolved の挙動を変える、`dnsmasq` を入れるなど、**どこを触ると何が変わるか** が分かる
- **暗号化 DNS の選択**: DoT / DoH を有効にすると、リクエストが OS の resolver ではなく **アプリ自身（ブラウザ等）の内蔵 resolver** から出ることがある
- **ドメイン運用の判断**: ccTLD（`.io` 等）と gTLD（`.com` 等）のリスク差を理解する土台

## まとめ

- DNS の実装は **権威サーバ / リゾルバ / スタブ / CLI / 拡張プロトコル** に分かれる
- OpenSSL と同じく、**OS にインストールされた実装をアプリが呼ぶ** 構造（特に Linux + glibc）
- 一番触る機会が多いのは **`dig` / `resolvectl`**、自分でサーバを立てるなら **dnsmasq / Unbound / BIND**
- さらに上に **TLD レジストリ → ICANN** という制度層がある

## 関連

- 暗号実装の出所（同じ構造の視点）→ [crypto-implementation-source.md](crypto-implementation-source.md)
- プロトコルを「プロセス × プロセス間通信」で捉える共通フレーム → [process-and-protocol.md](process-and-protocol.md)
- 4 切り（A / B / C / D） → [learning-by-slicing.md](learning-by-slicing.md)
- /etc/hosts による名前解決 → [../01-package-delivery/step-03-hostname-resolution.md](../01-package-delivery/step-03-hostname-resolution.md)
