# 事前準備

このページは、初めて環境構築する受講者向けの入口です。  
各ツールは公式サイトからインストールし、インストール後にバージョン確認コマンドを実行してください。

## 必須ソフト

- Docker
  - 公式: <https://www.docker.com/>
  - macOS / Windows では通常 [Docker Desktop](https://www.docker.com/products/docker-desktop/) を使います。
- VSCode + Dev Containers
  - VS Code: <https://code.visualstudio.com/>
  - Dev Containers 拡張: <https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers>
- Node.js / npm
  - 公式: <https://nodejs.org/>
  - `iot-market` と `iot-market-ui` の起動に使います。
- Rust / Cargo
  - 公式: <https://www.rust-lang.org/tools/install>
  - `simple-storage`, `mediator-owner`, `mediator-buyer` のビルドと起動に使います。
- Git
  - 公式: <https://git-scm.com/>
  - リポジトリ取得に使います。

## 推奨

- Docker Desktop（`host.docker.internal` を使うため）
- MetaMask
  - 公式: <https://metamask.io/>
  - ローカルチェーンに接続してフロントエンドから購入操作を試すために使います。

## 推奨事前知識

- HTTP/REST の基本（GET/POST と JSON）
- コマンドラインの基本操作（`cd`, `ls`, `cat`）
- Git の基本（clone, branch）
- データベースの基本（テーブル、ログ）

## 受講者向けチェック

まず、以下のコマンドがすべて実行できることを確認してください。

```bash
docker --version
npm -v
cargo --version
git --version
```

期待結果:

- それぞれバージョン番号が表示される
- `command not found` が出ない

次に、教材リポジトリを取得します。

```bash
git clone --recursive https://github.com/ertlnagoya/Blockchain_IoT_Marketplace.git
cd Blockchain_IoT_Marketplace
```

初回 clone に時間がかかることがあります。これは submodule を含むため正常です。

## OS別の補足

- Windows:
  - WSL2 + Docker Desktop の構成を推奨します。
  - 参考: <https://learn.microsoft.com/windows/wsl/>
- macOS:
  - Apple Silicon 環境では一部コンテナの初回ビルドが遅いことがあります。
- Linux:
  - Docker daemon の権限設定が必要な場合があります。

## この段階で理解しておくとよいこと

- `iot-market` / `iot-market-ui`: ローカルブロックチェーンとUI
- `simple-storage`: データ本体の保存先
- `ipfs`: メタデータ保存の補助
- `mediator-owner` / `mediator-buyer`: データ共有を仲介するプロセス

詳細は [学習基礎 / Hardhat基礎](../foundations/hardhat-basics.md) と [学習基礎 / SSI/DID/VC基礎](../foundations/ssi-did-vc-basics.md) を参照してください。

## 講師向けチェック

1. デモ用アカウント（MetaMask）を事前用意
2. `iot-market` デプロイ済みログを確認
3. `ipfs` / PostgreSQL のテーブル作成完了を確認
4. サンプル入力（HUSKYLENS2またはWebcam）の動作確認
5. 受講者向けレポートテンプレートを配布
