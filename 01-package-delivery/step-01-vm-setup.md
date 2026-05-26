# ステップ 1: Multipass セットアップと VM 1 台起動

Mac に Multipass を入れて、Ubuntu の VM を 1 台立てて、入って出る、停止する、までを確認する。

## このステップのゴール

- `multipass list` で VM が 1 台「Running」状態で見える
- `multipass shell` で VM に入って Ubuntu のプロンプトが出る
- `multipass stop` で停止できる

VM をどう扱うかの基本動作を体に入れるのが目的。**配信基盤の構築は次のステップから**。

## 所要時間

10 〜 15 分（初回は Ubuntu イメージのダウンロードに数分かかる）

## 前提

- macOS (Apple Silicon)
- Homebrew がインストール済み（`brew --version` で確認）

---

## 手順

### 1. Multipass をインストール

```bash
brew install --cask multipass
```

インストール後、確認:

```bash
multipass version
```

バージョン番号が表示されれば OK。

> 注: Mac のシステム拡張の許可ダイアログが出る場合がある。出たら「許可」する。

### 2. Ubuntu VM を 1 台立てる

```bash
multipass launch --name test01 22.04
```

意味:

| 部分 | 意味 |
|---|---|
| `launch` | VM を起動するコマンド |
| `--name test01` | VM の名前を `test01` にする |
| `22.04` | Ubuntu 22.04 LTS を使う |

初回はイメージダウンロード (~600MB) があるので数分待つ。プロンプトに戻ってきたら起動完了。

### 3. VM 一覧で状態を確認

```bash
multipass list
```

こういう表示が出る:

```
Name      State     IPv4             Image
test01    Running   192.168.64.x     Ubuntu 22.04 LTS
```

`State` が `Running` であれば成功。

### 4. VM に入る (ログイン)

```bash
multipass shell test01
```

プロンプトが `ubuntu@test01:~$` に変わる。**ここから先は VM の中**。

中で軽く確認:

```bash
whoami                 # → ubuntu
hostname               # → test01
cat /etc/os-release    # → Ubuntu 22.04 と表示される
ip addr show eth0      # → VM の IP アドレスが見える
```

### 5. VM から抜ける

```bash
exit
```

Mac のプロンプトに戻る。

### 6. VM を停止

立てっぱなしだとリソースを食うので、使い終わったら止める。

```bash
multipass stop test01
```

`multipass list` で `Stopped` 表示になれば OK。再開は:

```bash
multipass start test01
```

### 7. (任意) VM の削除

このステップで作った `test01` は動作確認用なので、次に進む前に削除してよい。

```bash
multipass delete test01
multipass purge
```

`delete` は「ゴミ箱に入れる」、`purge` は「ゴミ箱を空にする」イメージ。両方やって完全に消える。

---

## つまずきポイント

| 症状 | 対処 |
|---|---|
| `brew install --cask multipass` でエラー | Homebrew のバージョンが古い可能性。`brew update` してから再試行 |
| `multipass launch` が長時間止まる | 初回はイメージダウンロード待ち。数分は様子見 |
| `Failed to obtain exit status code from process` | VM 起動失敗。`multipass delete test01&& multipass purge` でリセットして再試行 |
| VM 一覧で `Unknown` のまま | Multipass デーモンの起動待ち。1 分ほど待って `multipass list` を再実行 |

---

## ここで覚えるコマンド

| コマンド | 意味 |
|---|---|
| `multipass launch --name NAME 22.04` | VM 起動 |
| `multipass list` | VM 一覧 |
| `multipass shell NAME` | VM にログイン |
| `multipass stop NAME` | VM 停止 |
| `multipass start NAME` | VM 再開 |
| `multipass delete NAME && multipass purge` | VM 完全削除 |

次のステップ以降で繰り返し使う。

---

## 次のステップ

次は「管理ノード」「配信ノード × 2」「クライアント」の合計 4 台を、用途別の名前で一気に立てる。VM 間で通信できることまで確認する。
