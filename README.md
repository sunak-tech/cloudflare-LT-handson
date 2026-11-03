# Cloudflare Tunnel を使った安全なコンテナ公開ハンズオン

## 📌 概要

このリポジトリは、**ルーターのポート開放ゼロ・グローバルIPアドレス秘匿**のまま、手元のPCで動くDockerコンテナを **Cloudflare Tunnel** 経由で安全にインターネットへ公開するハンズオン用リポジトリです。

### 🛡️ セキュリティ設計（多層防御のポイント）

本ハンズオンの [`docker-compose.yml`](docker-compose.yml) では、コンテナを安全に公開するため以下の3つの設計を採用しています。

1. **非rootコンテナの採用（UID 101）**
   - Webサーバーには公式イメージ `nginxinc/nginx-unprivileged:alpine` を使用しています。コンテナ内部で最初から一般ユーザー（UID 101）としてプロセスが動作するため、万が一Webサーバーに脆弱性があってもroot権限を奪われるリスクを軽減します。
2. **ブリッジネットワーク隔離（抽象ソケット保護）**
   - `network_mode: host` を使わず、Dockerのデフォルトブリッジネットワークでコンテナを隔離しています。ホストOSのネットワーク名前空間や抽象UNIXドメインソケットへ直接アクセスされることを防ぎます。
3. **ホスト側ポート非公開（`ports:` 設定なし）**
   - ホストPC側へのポートマッピング（`ports: - "8080:8080"` など）を一切記述していません。同じWi-Fi内の他のPCからでも直接ポートスキャンやアクセスはできず、**Cloudflare Tunnel 経由の通信のみ**を受け付けます。

---

## 💻 事前準備（OS別注意点）

### Windows の方
- **Docker Desktop** を起動し、画面左下のステータスバーが **緑色（Engine running）** になっていることを確認してください。
- GitHubからZIPファイルでダウンロードした場合は、**必ず「すべて展開（解凍）」してから**作業してください（ZIPの中身を直接開いた状態ではDockerが動きません）。
- **WSL2** をお使いの方は、ファイル監視（リアルタイム反映）やI/O性能のトラブルを防ぐため、`/mnt/c/` 配下ではなく **Linuxホームディレクトリ（`~/`）配下**にリポジトリを配置してください。

### Mac (Intel / Apple Silicon) の方
- **Docker Desktop**（または **OrbStack**）を起動しておいてください。

### Linux の方
- ターミナルから `docker` および `docker compose` コマンドが使えることを確認してください。
- 実行時にソケットの権限エラー（`permission denied while trying to connect to the Docker daemon socket`）が出る場合は、コマンドの先頭に `sudo` を付けて実行してください。

### 📥 【重要】事前 Pull のお願い
当日の会場Wi-Fiの混雑を防ぐため、**自宅などで事前に以下のコマンドを実行してDockerイメージをダウンロード**しておいてください。

```bash
docker pull nginxinc/nginx-unprivileged:alpine
docker pull cloudflare/cloudflared:latest
```

---

## 🚀 当日のハンズオン手順

### Step 1: ファイルの取得

ターミナル（Windowsの方は PowerShell または WSL2）を開き、以下のいずれかの方法でファイルを取得してディレクトリに移動します。

#### 方法A: `git clone` を使う場合
```bash
git clone https://github.com/sunak-tech/cloudflare-LT-handson.git
cd cloudflare-LT-handson
```

#### 方法B: ZIPファイルでダウンロードする場合
1. GitHubページの右上にある緑色の **「Code」** ボタンをクリックし、**「Download ZIP」** を選択します。
2. ダウンロードしたZIPファイルを**展開（解凍）**します。
3. ターミナルを開き、展開したフォルダへ移動します。
   ```bash
   cd 展開したフォルダのパス
   ```

---

### Step 2: `.env` ファイルの作成とトークン設定

テンプレートファイル（`.env.example`）をコピーして、環境変数ファイル `.env` を作成します。

- **Mac / Linux / WSL2 の場合:**
  ```bash
  cp .env.example .env
  ```

- **Windows (PowerShell) の場合:**
  ```bash
  Copy-Item .env.example .env
  ```

作成した `.env` ファイルをテキストエディタ（VS Code やメモ帳など）で開き、発表者から配布された `TUNNEL_TOKEN` を `=` の後ろに貼り付けて保存します。

```env
# 発表者から受け取ったトークンを = の後ろに貼り付けてください（スペースや引用符 "" は不要）
TUNNEL_TOKEN=eyJhIjoi...（配布されたトークン）
```

---

### Step 3: コンテナの起動

以下のコマンドを実行して、Webサーバー（`my-web`）とトンネル（`tunnel`）の2つのコンテナをバックグラウンドで起動します。

```bash
docker compose up -d
```

起動したら、以下のコマンドで状態を確認します。

```bash
docker compose ps
```

`my-web` と `tunnel` の2つのコンテナの `STATUS` が **`Up`** になっていれば成功です！

---

### Step 4: スマホからアクセス＆リアルタイム書き換え体験

1. 発表者がCloudflare側のルーティング（転送先: `http://my-web:8080`）を設定します。
2. **スマートフォンのWi-Fiをオフにして（4G / 5G モバイル回線で）**、案内された自分のサブドメインURLにアクセスしてみましょう！手元のPCで動いているページが表示されます。
3. PCのエディタで `html/index.html` を開き、以下のコメントがある箇所を探します。
   ```html
   <!-- ★ここを自分の名前や好きな文字に書き換えて保存してみよう！ -->
   ```
4. 自分の名前や好きなメッセージに書き換えてファイルを保存し、スマホのブラウザをリロードしてみてください。**コンテナを再起動することなく、即座に変更が反映されます！**

---

### Step 5: 片付け

ハンズオン終了後は、以下のコマンドを実行してコンテナとトンネルを安全に停止・削除します。

```bash
docker compose down
```

---

## 🛠️ トラブルシューティング（よくある質問）

### Q1. ブラウザで `502 Bad Gateway` が出る場合
- **コンテナが起動しているか確認:**
  `docker compose ps` を実行し、`my-web` と `tunnel` の両方が `Up` になっているか確認してください。
- **Cloudflare側の転送先設定を確認:**
  Cloudflare Zero Trust ダッシュボード側の Public Hostname 設定にて、Service の転送先が **`http://my-web:8080`**（ポート80ではなく **8080**）になっているか発表者と一緒に確認してください。

### Q2. `tunnel` コンテナがすぐ落ちる・再起動（Restarting）を繰り返す場合
- **`.env` ファイルの名前と中身を確認:**
  - ファイル名が `.env.txt` や `.env.example` のままになっていないか確認してください。
  - `TUNNEL_TOKEN=` の前後に**余計なスペース**や**ダブルクォーテーション（`""`）**が入っていないか、トークンが途中で切れていないか確認してください。
- **ログの確認方法:**
  以下のコマンドで `tunnel` コンテナのエラーログを確認できます。
  ```bash
  docker compose logs tunnel
  ```
  修正した後は `docker compose up -d` を再実行してください。

### Q3. `html/index.html` を書き換えたのに画面に反映されない場合
- ファイルが**上書き保存（Ctrl+S / Cmd+S）**されているか確認してください。
- ブラウザのキャッシュが残っている可能性があるため、**スーパーリロード（PCなら `Ctrl + F5` / `Cmd + Shift + R`）** や、スマホブラウザのキャッシュ削除・シークレットタブでのアクセスを試してください。
