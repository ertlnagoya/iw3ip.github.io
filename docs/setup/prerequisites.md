# 事前準備 (ツールのインストール)

ハンズオンを始める前に、PC へ必要なソフトを入れて動作確認します。
ここでは **「なぜそのツールがいるのか」** を一行ずつ添えています。難しい言葉は出てきたところで補足します。

> 既にツールが揃っている人 (経験者) は、最後の [動作確認チェックリスト](#動作確認チェックリスト) だけ済ませて [最短起動](quickstart.md) に進んで構いません。

## 全体像 — 何のためにツールを入れるのか

ハンズオンでは PC の中に **「データを安全に共有する練習用の街」** を作ります。
その街は次の建物 (= プロセス) でできています。

| 建物 (プロセス) | 何をする | 必要なツール |
|---|---|---|
| ローカルブロックチェーン | 売買の取引記録を残す | **Node.js** (`npx hardhat node`) |
| マーケットプレイス UI | 出品・購入のブラウザ画面 | **Node.js** + ブラウザ + **MetaMask** |
| データ保管庫 | データ本体を置く場所 | **Rust** (`cargo run`) |
| ファイル分散保管 | データのコピーを分散保存 | **Docker** (IPFS コンテナ) |
| publisher / 仲介サービス | 上のすべてをつなぐ | **Docker** |

つまり、**Docker / Node.js / Rust / Git / ブラウザ + MetaMask** の 5 つが揃えば街を建てられます。

## 必須ソフト

### 1. Docker

街全体をまとめて起動するためのツール。

- **macOS / Windows**: [Docker Desktop](https://www.docker.com/products/docker-desktop/) を入れます。GUI 付きで管理が簡単。
- **Linux**: ディストリ標準の `docker` パッケージか公式 install スクリプト。
- 公式サイト: <https://www.docker.com/>

> **メモリ設定の注意**: Docker Desktop の Settings → Resources → Memory を **6 GB 以上** にしてください。デフォルトの 2 GB では一部コンテナが落ちます。

### 2. Node.js (npm 同梱)

JavaScript のランタイム。マーケットプレイスの UI とブロックチェーンの起動に使います。

- 公式: <https://nodejs.org/>
- バージョンは **LTS (Long Term Support)** を選んでおけば OK
- 複数バージョンを切り替える人は [nvm](https://github.com/nvm-sh/nvm) や [volta](https://volta.sh/) も便利

### 3. Rust (cargo 同梱)

データ保管庫 (`simple-storage`) を動かすための言語。

- 公式 install: <https://www.rust-lang.org/tools/install>
- 1 行コマンドで入ります: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

### 4. Git

教材リポジトリを取得するため。

- 公式: <https://git-scm.com/>
- macOS は Xcode Command Line Tools に同梱、Linux はディストリ標準で大体入っています

### 5. ブラウザ + MetaMask 拡張

マーケットプレイスでデータを購入するときに使います。Part 1 (基本) で使用。

- ブラウザ: Chrome / Edge / Firefox / Brave のどれでも可
- [MetaMask](https://metamask.io/) — 拡張機能をブラウザにインストール
- 「ローカルチェーンに接続するための Ethereum 風ウォレット」と思えば OK

## 任意 (機能拡張パートで使う)

ハンズオン Part 2 (機能拡張) を進める場合に追加で必要です。

### スマートフォン + SSI ウォレットアプリ

VC (デジタル証明書) を受け取る入れ物。

- iPhone / Android のどちらでも可
- 推奨アプリ: **Sphereon Wallet** (iPhone は App Store、Android は Play Store)
- インストール後の画面例:

| クレデンシャル一覧 | 受け取り画面 |
|---|---|
| <img src="../hands-on/images/ha-ssi-wallet/01-credential-list.png" alt="credential list" width="220"> | <img src="../hands-on/images/ha-ssi-wallet/02-credential-offer.png" alt="credential offer" width="220"> |

### VS Code + Dev Containers (推奨)

publisher 内部のコードを読み解きたい人向け。

- VS Code: <https://code.visualstudio.com/>
- 拡張: [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

## 推奨事前知識

無くてもハンズオンは進みますが、あると理解が早くなります:

- **コマンドラインの基本** (`cd`, `ls`, `cat`)
- **HTTP/REST の基本** (GET / POST と JSON)
- **Git の基本** (clone, branch)
- **ブラウザの開発者ツール** (F12 で開ける Network / Console タブ)

## 動作確認チェックリスト

ターミナルを開いて、次のコマンドがすべて **バージョン番号を表示** すれば OK です。

```bash
docker --version
node --version
npm --version
cargo --version
git --version
```

期待する出力例:

```
Docker version 25.0.x, build ...
v20.x.x
10.x.x
cargo 1.7x.x (...)
git version 2.x.x
```

`command not found` が出るツールは、上のセクションに戻って入れ直してください。

### Docker が動くか確認

```bash
docker run --rm hello-world
```

「Hello from Docker!」のメッセージが表示されれば成功です。

### 教材リポジトリを取得

```bash
git clone --recursive https://github.com/ertlnagoya/Blockchain_IoT_Marketplace.git
cd Blockchain_IoT_Marketplace
```

> **`--recursive` を忘れずに**: サブリポジトリ (submodule) を含むため、忘れると一部ファイルが空のディレクトリになります。
> 既に clone 済みで submodule が空なら `git submodule update --init --recursive` で取得できます。

初回 clone は数分かかります (リポジトリサイズが大きいため)。

## ここまでできていれば次に進める

- [ ] 上記 5 コマンドが全部バージョン番号を返す
- [ ] `docker run hello-world` が成功する
- [ ] `Blockchain_IoT_Marketplace` ディレクトリができている
- [ ] (Part 1 を進める人) MetaMask が拡張機能としてブラウザに表示されている
- [ ] (Part 2 を進める人) スマホに Sphereon Wallet が入っている

すべてチェックできたら → [最短起動](quickstart.md) に進んでください。

## OS 別の補足

### Windows

- **WSL2 + Docker Desktop** の組み合わせを推奨します。
- 参考: <https://learn.microsoft.com/windows/wsl/>
- パスの区切りなどでハマったら WSL2 内でハンズオン作業をする方が安全です。

### macOS

- Apple Silicon (M1〜M4) でも基本動作します。一部コンテナの初回ビルドが Intel より遅いことがあります。

### Linux

- Docker daemon にアクセスする権限 (`docker` グループへの追加) が必要な場合があります。
- `sudo usermod -aG docker $USER && newgrp docker`

## うまくいかないとき

- ポートが競合している → [トラブルシュート](../operations/troubleshooting.md) の「ポート」項
- Docker のメモリ不足で落ちる → Docker Desktop のメモリを 6 GB 以上に
- それ以外 → [FAQ](../operations/faq.md) を確認

## このページで覚える用語

| 用語 | 短い意味 |
|---|---|
| **ターミナル** | 文字でコマンドを打つアプリ (macOS: Terminal、Windows: PowerShell / WSL) |
| **submodule** | リポジトリの中に別のリポジトリを埋め込む仕組み |
| **コンテナ** | アプリを「箱詰め」して動かす単位。Docker が動かす |
| **拡張機能** | ブラウザに追加できる小さなアプリ。MetaMask はこれ |

詳しい用語は [環境構築 Overview のミニ辞書](index.md)。
