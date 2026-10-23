# ハンズオン

実際に手を動かして IW3IP を理解する章です。
**基本** で全体像を確認し、**機能拡張** で安全な共有や段階アクセスを扱い、**知能統合** で AI によるデータの意味づけと要求の解釈を扱います。

> 先に [環境構築](../setup/index.md) を済ませてからこのページに進んでください。

この章のページは、コマンドと期待する出力、確認ポイントを順に示す手順書です。各機能の仕組みと設計判断の理由は [設計](../design/index.md) の章に、手順どおりに動かないときの対処は [困ったとき](../operations/index.md) の章にまとめています。

## 基本 と 機能拡張 の関係

このハンズオンは、マーケットプレイス周辺の機能を **基本 (v1)** と **機能拡張 (v2)** の 2 段階で説明します。
v2 は v1 を置き換えるものではなく、v1 に追加する経路です。基本だけでもデータの売買は完結します。

### 基本 (v1) — 最小構成

- データを **マーケットプレイスに出品** し、**MetaMask で購入** して受け取る。出品者側はデータを保管庫 (simple-storage) に置き、その場所 (URI) を購入者の公開鍵で暗号化してチェーンに書き込む。購入者側はそれを復号してデータを取りに行く
- 必要なのは PC と MetaMask のみ
- ハンズオン Part 1 ではこの v1 経路を扱います

### 機能拡張 (v2) — 安全・条件付き・段階アクセス

- 購入したあと、**スマホ SSI ウォレット** が **VC** を受け取る
- 受信側は VC を提示してデータを取得する。何のために使うか (purpose) や信頼度に応じて、見せる中身を段階的に変える
- ハンズオン Part 2 ではこの v2 経路と、それを前提とする同意・段階アクセスを扱います

```
購入 (v1 と v2 で共通)
    ↓
    ├── v1 経路: 暗号化した URI → 復号 → 保管庫からデータ ← Part 1 で扱う
    └── v2 経路: bridge → PurchaseViewerVC → wallet     ← Part 2 で扱う
                  → ViewerToken → /platform/data
```

詳しい設計は [Marketplace VC Bridge — v1 / v2 設計仕様](../design/marketplace-vc-bridge-spec.md) を参照してください。

## Part / Phase / Stage の関係 { #part-phase-stage }

このサイトでは、3 種類の番号を使っています。

- **Part 1〜3**: このサイトでの学習順です。基本、機能拡張、知能統合の順に進みます。
- **Phase 1〜3**: 基盤の発展段階です。Phase 1 はデータ共有、Phase 2 はイベント共有、Phase 3 は AI による判断と制御を指します（[プラットフォーム概要のフェーズ構成](../platform-overview.md#フェーズ構成)）。
- **Stage 0〜7、A、T、P**: Part 2 のうち、VC を使うハンズオンと、その準備 (Stage 0)、画像の来歴の署名 (Stage P。VC は使わない) に付けた通し番号です。各ページは、前の Stage を済ませていることを前提にしています。ただし、Stage P は前の Stage を使わず、教材リポジトリと Python だけで単独で動かせます。Stage 2 は欠番です。

| Stage | ハンズオン | 扱う VC |
|---|---|---|
| 0 | [検知イベントだけを共有する](webcam-event-sharing.md)、[防災イベントを共有する](environment-disaster.md) | なし (`/consents` に JSON を登録) |
| 1 | [データ提供者の認証](ha-ssi-wallet.md) | ConsentVC |
| 3 | [データ閲覧者の認証](ha-ssi-viewer.md) | ViewerVC |
| 4 prep | [機器間での連携認証](ha-ssi-service.md) | ServiceVC |
| 5 | [購入後に証明書をウォレットへ渡す](marketplace-vc-bridge.md) | PurchaseViewerVC |
| 6 | [ServiceVC で書いたデータを PurchaseViewerVC で読み出す](marketplace-vc-end-to-end.md) | ServiceVC + PurchaseViewerVC (Stage 1・3 の ConsentVC・ViewerVC の流れの上に組み立てる) |
| 7 | [出品者の身元を裏付ける](marketplace-seller-vc.md) | SellerVC |
| A | [スマホアプリで購入から閲覧まで](marketplace-mobile-app.md) | PurchaseViewerVC (スマホだけで操作) |
| T | [信頼度に応じて見せる中身を変える](data-user-vc-tiered.md) | DataUserVC |
| P | [画像の来歴を証明する](c2pa-provenance.md) | なし (C2PA の署名に X.509 証明書を使う。publisher とウォレットは不要) |

VC とトークンの対応は [VC アーキテクチャ全体像](../design/vc-architecture-overview.md) にまとめています。

## コマンド中の `$HOST_IP` について { #host-ip }

スマホから PC に接続するハンズオンでは、PC の LAN 内の IP アドレスを使います。各ページのコマンドは `$HOST_IP`、ブラウザやスマホに入力する URL は `<HOST_IP>` と書いています。

まず PC の IP アドレスを調べます。

| OS | コマンド |
|---|---|
| macOS | `ipconfig getifaddr en0` (有線 LAN の場合は `en1` などに変える) |
| Linux | `hostname -I` (最初に表示されるアドレス) |
| Windows | PowerShell で `ipconfig` を実行し、「IPv4 アドレス」を見る |

コマンドを実行するターミナルごとに、最初に次を実行します (`192.168.1.20` は自分の IP に置き換えます)。

```bash
export HOST_IP=192.168.1.20
```

以後、`$HOST_IP` を含むコマンドはそのまま貼り付けて実行できます。URL の `<HOST_IP>` は、同じ IP アドレスに置き換えて入力します。Windows で WSL2 を使う場合も、`HOST_IP` には Windows 側の `ipconfig` で表示される IP アドレスを設定します。

## 全体構成

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Part 1 / 基本 (v1)</div>
    <h3>IoT データを取って共有する</h3>
    <p>カメラやセンサからデータを取り、見て、マーケットプレイスで売買する基本の経路を扱います。</p>
    <p class="iw3ip-phase-links">
      <a href="ha-demo-simulator/">HA Demo Simulator</a> /
      <a href="huskylens2/">HUSKYLENS2</a> /
      <a href="webcam/">USB Webcam</a> /
      <a href="ha-ssi-publisher/">HA Publisher</a> /
      <a href="mobile-viewer/">Mobile Viewer</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Part 2 / 機能拡張 (v2)</div>
    <h3>同意・条件付き共有・段階アクセス</h3>
    <p>VC とウォレットを使って、目的や信頼度に応じた共有を扱います。途中で止めても問題ありません。</p>
    <p class="iw3ip-phase-links">
      <a href="webcam-event-sharing/">Webcam Event Sharing</a> /
      <a href="environment-disaster/">Environment Disaster</a> /
      <a href="ha-ssi-wallet/">HA SSI Wallet</a> /
      <a href="marketplace-vc-bridge/">VC Bridge</a> /
      <a href="data-user-vc-tiered/">DataUserVC Tiered</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Part 3 / 知能統合</div>
    <h3>AI に要求を解釈させる</h3>
    <p>人の要求を AI が解釈し、Part 1/2 で集めたデータを材料に判断・処理する応用段階です。</p>
    <p class="iw3ip-phase-links">
      <a href="local-vlm-distribution/">ローカル VLM でカメラデータを意味づけ</a> /
      <a href="privacy-compare/">AI に渡す前にカメラ画像を守る</a> /
      <a href="regional-safety-assistant/">Regional Safety Assistant</a> /
      <a href="llm-planner/">LLM Planner</a>
    </p>
  </div>
</div>

## 進む順 — 目的別の経路

| 目的 | 進む順 |
|---|---|
| まず動かしたい | [HA Demo Simulator](ha-demo-simulator.md) のみ (約 20 分) |
| 基本を一通り | Part 1 全部 (約 1.5 時間。1.2 のデバイスは 1 つ選ぶ) |
| 安全な共有まで | Part 1 → Part 2 の 2.1〜2.3 (+ 約 4 時間) |
| マーケット連携の v2 まで | Part 1 → Part 2 の 2.4 まで (+ 約 4 時間) |
| 段階アクセスまで | Part 2 の 2.5 まで (+ 約 1 時間。画像・動画の配信と意味レベルの段階化まで進む場合は、さらに約 3 時間。C2PA で画像の来歴を証明する (Stage P) 場合は、さらに約 30〜40 分) |
| AI 統合まで | Part 3 まで (+ 約 5 時間) |

時間は、各ページの冒頭にある所要時間を合計した目安です。「+」は、1 つ上の行までにかかる時間に加えて必要な時間です。

各ハンズオンページの先頭には **「このハンズオンで分かること」「前提」「使うもの」「所要時間」** を載せています。途中で詰まったら [トラブルシュート](../operations/troubleshooting.md) を参照してください。

## 使う技術要素の早見表

サブグループごとに、使う技術要素をまとめます。Part 2 には、ブロックチェーンを使うサブグループと使わないサブグループがあります。

| サブグループ | publisher (Docker) | ブロックチェーン (Hardhat) | MetaMask | SSI ウォレット (スマホ) | IPFS・保管庫 (v1) | 画像処理 / LLM |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| **Part 1 / データを取り込む** | ✓ <small>(HA 経路)</small> | ✓ <small>(カメラ経路)</small> | – | – | ✓ <small>(カメラ経路)</small> | – |
| **Part 1 / マーケットで売買 (v1)** | – | ✓ | ✓ | – | ✓ | – |
| **Part 2 / イベントを共有してみよう (Stage 0)** | ✓ | – | – | ✓ <small>(補論のみ)</small> | – | – |
| **Part 2 / 認証してみよう (SSI)** | ✓ | – | – | ✓ | – | – |
| **Part 2 / マーケットと連携 (v2)** | ✓ | ✓ | ✓ | ✓ | – | – |
| **Part 2 / データ処理を入れる**<sup>※</sup> | ✓ | ✓ <small>(Tier 3 実機時)</small> | ✓ <small>(同)</small> | ✓ | – | ✓ <small>(OpenCV / VLM 任意)</small> |
| **Part 3 / 知能統合** | ✓ | – | – | – | – | ✓ <small>(VLM / LLM)</small> |

※ このサブグループのうち、[画像の来歴を証明する (C2PA / Stage P)](c2pa-provenance.md) は Python だけで動き、publisher、ブロックチェーン、MetaMask、SSI ウォレットと VC を使いません。

### 主に使う VC (Verifiable Credential) の種類

| サブグループ | Consent | Viewer | Service | Purchase&shy;Viewer | Seller | DataUser |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Part 2 / イベントを共有してみよう | ✓ <small>(補論のみ)</small> | – | – | – | – | – |
| Part 2 / 認証してみよう | ✓ | ✓ | ✓ | – | – | – |
| Part 2 / マーケットと連携 (v2) | – | – | ✓ | ✓ | ✓ | – |
| Part 2 / データ処理を入れる<sup>※</sup> | – | – | – | ✓ | – | ✓ |

### 表の見方

- **publisher**: ハンズオン用のサーバ。`docker compose up` で起動します。Part 1 のマーケットでの売買 (v1) では使いません。v1 のスタックでは IPFS も起動します。IPFS と publisher はどちらもポート 8080 を使うため、同時には起動できません。
- **ブロックチェーン**: Hardhat ローカルチェーン + デプロイ済コントラクト。マーケットでの売買が起点になるサブグループで必要です。
- **MetaMask**: ブロックチェーンに対する署名・送金に使います。ローカルチェーンに接続して使います。
- **SSI ウォレット**: スマホに入れる iw3ip-wallet (Sphereon Wallet の fork) です。VC の受け取りと提示に使います。
- **IPFS・保管庫 (v1)**: v1 で使います。購入後のデータは保管庫 (simple-storage) に置き、その場所 (URI) を暗号化してチェーンに書き込みます。IPFS と PostgreSQL は、動画のメタデータの登録と検索画面に使います。v2 では VC を使う経路が加わります。
- **カメラ経路 / HA 経路**: Part 1 の HUSKYLENS2 と USB ウェブカメラはイベントファイルを `mediator-owner` に渡すので、[最短起動](../setup/quickstart.md) のスタック (Hardhat、保管庫、IPFS など) が必要です。HA SSI Publisher は publisher にデータを送ります。
- **画像処理 / LLM**: OpenCV (Tier 2 のぼかし処理)、Ollama で動かす VLM（視覚言語モデル。LLaVA / moondream）、Phase 3 の LLM Planner を指します。

---

## Part 1: 基本 — IoT データの基本経路

### この Part のゴール

- **データがどこで発生し、どこへ届くか** を追跡できるようになる
- **マーケットプレイス v1 経路** で、出品 → 購入 → 受け取りを一通り実行できる

### 1.1 まず動かす — 実機なしで全体像を確認する

実機がなくても、Home Assistant のシミュレータでハンズオン全体を確認できます。
最初に試すのに適しています。

- [実機なしで一連を動かす (HA Demo Simulator)](ha-demo-simulator.md)

### 1.2 デバイスからデータを取る

カメラやセンサからデータを取り出す経路を作ります。HUSKYLENS2 と USB ウェブカメラはイベントファイルを `mediator-owner` に渡し、HA SSI Publisher は publisher にデータを送ります。
手元の機材に合わせてどれか 1 つ選んでください。

- [AI カメラで人物を検知する (HUSKYLENS2)](huskylens2.md) — 専用 AI カメラを使う
- [USB カメラでポイ捨てを検知する (OpenCV)](webcam.md) — USB カメラ + OpenCV
- [Home Assistant のデータを取り込む (HA SSI Publisher)](ha-ssi-publisher.md) — Home Assistant 連携

### 1.3 マーケットプレイスで売買 (v1 経路)

出品 → 購入 → 受け取り、までを一通り行います。受け取りでは、保管庫 (simple-storage) に置いたデータの場所 (URI) を、暗号化してチェーン経由で渡します。
ここまでが **基本 (v1)** の完了点です。

1. **出品**: 1.2 のサンプル ([HUSKYLENS2](huskylens2.md) または [USB ウェブカメラ](webcam.md)。mock モードで可) を動かすと、イベントファイルが `mediator-owner/raw_data/output` に出力されます。1.2 で HA SSI Publisher を選んだ場合は、[USB ウェブカメラサンプル](webcam.md) を mock モードで動かして出品します。`mediator-owner` がイベントファイルを検知して商品として登録し、ターミナルに商品のアドレスを表示します。

    ```
    Product deployed: 0x948b3c65b89df0b4894abe91e6d02fe579834f8f for file 301_huskylens_1791551565.txt
    ```

2. **購入**: 商品ページ (`http://localhost:5173/merchandise/<表示されたアドレス>`) を開き、右上の「Connect your wallet!」で MetaMask (Account #2) を接続してから「Purchase!」を押し、MetaMask で承認します。承認後に移るページには「エラー」と表示されますが、v1 では publisher が動いていないためで、購入は済んでいます。購入できたかは、次の `mediator-buyer` のログで確かめます。

3. **受け取り**: 出品者側の `mediator-owner` が購入を検知し、データの置き場所を購入者の公開鍵で暗号化してチェーンに書き込みます。購入者側の `mediator-buyer` がそれを復号してデータを取得し、内容を検証します。

    `mediator-owner` のターミナル:

    ```
    Your Product is bought by 0x0000…93bc
    File uploaded successfully: cf701007aad74177b6ecaa8e4a63c931
    Upload event sent successfully
    Verification is successful
    Withdraw successful
    ```

    `mediator-buyer` のターミナル:

    ```
    You bought a product of 0x0000…79c8
    Downloading file...
    File downloaded successfully
    Verification result: true
    ```

    購入したイベントファイルは `mediator-buyer/downloads` に保存されます。`Verification result: true` は、届いたデータのハッシュが商品に登録された値と一致したことを示し、これを受けて出品者に代金が支払われます (`Withdraw successful`)。

最短起動の手順 2 (デプロイ) で登録される 5 件は、`mediator-owner` が出品したものではありません。購入はできますが、データは届きません。

各プロセスの起動手順は [最短起動](../setup/quickstart.md) にあります。

### 1.4 スマホでマーケットを開いて購入する

1.3 のマーケットプレイス画面をスマホのブラウザで開き、購入操作まで確かめます。

- [スマホでマーケットを開いて購入する (v1)](mobile-viewer.md)

### Part 1 の成功判定

- 1.2 のデータが届いたことを確認できた。HA 経路では `curl http://localhost:8080/platform/ingest` (HA SSI Publisher の手順 6) に、カメラ経路では `mediator-owner` の `Product deployed` にデータが現れる
- マーケットで出品 → 購入 → 受け取りが一通りでき、`mediator-buyer/downloads` にデータが保存された

---

## Part 2: 機能拡張 — 同意・条件付き共有・段階アクセス

### この Part のゴール

- 条件付きで見せることがなぜ必要かを理解する
- **VC とウォレット** でその条件付き共有を実装する
- **段階アクセス (Tier 3 / 2 / 1)** で、信頼度に応じて見せる中身を変える

### 2.1 イベント共有 — 生データではなく結果だけ (Stage 0)

カメラの生映像の代わりに、「人物検出: あり」「ゴミ捨て: あり」のような **イベント** だけを送る方式を試します。

- [検知イベントだけを共有する](webcam-event-sharing.md)
- [防災イベントを共有する](environment-disaster.md)

### 2.2 同意 (Consent VC) を入れる (Stage 1)

共有の前に「この目的なら共有してよい」という同意 VC を発行・提示する流れを学びます。

- [データ提供者の認証 (Consent VC / Stage 1)](ha-ssi-wallet.md)

### 2.3 役割の分離 — 閲覧と書き込みを分ける

データを「見るだけ」のロールと「書き込みだけ」のロールを VC で分けます。

- [データ閲覧者の認証 (Viewer VC / Stage 3)](ha-ssi-viewer.md)
- [機器間での連携認証 (Service VC / Stage 4 prep)](ha-ssi-service.md)

### 2.4 マーケットプレイス × ウォレット (v2 経路)

購入後の受け渡しを、暗号化した URI (v1) ではなく **PurchaseViewerVC + ViewerToken** で行う v2 経路を扱います。

- [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md) — bridge を起動し、購入から VC 発行までをつなぐ
- [ServiceVC で書いたデータを PurchaseViewerVC で読み出す (Stage 6)](marketplace-vc-end-to-end.md) — 書き込みから購入、閲覧までを通す
- [出品者の身元を裏付ける (Seller VC / Stage 7)](marketplace-seller-vc.md) — 出品者側の VC
- [スマホアプリで購入から閲覧まで (Stage A)](marketplace-mobile-app.md) — 専用モバイルアプリ

### 2.5 信頼度に応じた段階アクセス (Stage T / P)

利用者の **DataUserVC** に基づき、Tier 3 (動画+全部) / Tier 2 (画像+派生) / Tier 1 (要約のみ) の 3 段階で見せ方を変えます。
後半では、画像の内容を構造化したデータ（意味的中間表現）を作り、信頼度に応じて見せ方を変える処理（trust-aware rendering）も扱います。

- [信頼度に応じて見せる中身を変える (DataUserVC / Stage T)](data-user-vc-tiered.md)
- [画像・動画を段階アクセスで配信する (DataUserVC / Stage T)](data-user-vc-tiered-media.md) — 続きの §8〜§11
- [意味レベルで段階化する (VLM と意味的中間表現 / Stage T)](data-user-vc-tiered-semantic.md) — 続きの §12〜§13
- 設計の背景: [段階アクセスの設計仕様](data-user-vc-tiered-spec.md) ([設計](../design/index.md) の章)
- [画像の来歴を証明する (C2PA / Stage P)](c2pa-provenance.md) — 段階アクセスで配る画像に、C2PA の来歴を付けて署名する (publisher やウォレットは不要)

### Part 2 の成功判定

- Consent / DataUserVC / PurchaseViewerVC のいずれかをスマホ wallet に取り込めた
- `purpose` を変えると、共有 (`/simulate/publish`) や提示 (`/verifier/request`) の結果が `allow` / `deny` に切り替わり、信頼度を変えると `/platform/data` で見える内容が変わる
- audit log に「誰が・いつ・どの purpose で・何を見たか」が残っている

---

## Part 3: 知能統合 — AI に要求を解釈させる

### この Part のゴール

- ローカルの AI モデルでカメラ映像を意味づけし、その派生データを基盤で流通させる
- 人の自由文の要求 (例:「最近の異常を教えて」) を AI が **plan** に分解する
- AI に渡す前に、何が PC の外に出るかを構成ごとに比べて説明できる
- plan を **execute** で実行し、サンプルのイベントデータをもとに応答を返す
- frontend demo まで一連の流れを通して確認する

### 3.1 ローカル VLM でカメラデータを意味づけ

ラップトップ + USB カメラの映像をローカルモデルで解析し、意味データを基盤で流通させます。構造の理解 → 雛形課題 → ゼロからの機能追加、と段階的に進められる演習です。

- [ローカル VLM でカメラデータを意味づけ](local-vlm-distribution.md)

### 3.2 AI に渡す前にカメラ画像を守る

同じカメラ画像を、生のままクラウドへ、PC 内だけで、顔をぼかしてからクラウドへ、文字にしてからクラウドへ、の 4 通りで AI に渡し、何が PC の外に出たかと説明の内容を比べます。クラウドは模擬サーバで代用します。

- [AI に渡す前にカメラ画像を守る (ローカルとクラウドの比較)](privacy-compare.md)

### 3.3 地域安全アシスタント

「いつ・どこで・何が起きた」を記録したサンプルのイベントファイル (`examples/phase3_events_park_safety.json`) を読み込み、自由文の要求に応答するデモです。

- [地域安全アシスタントを動かす](regional-safety-assistant.md)

### 3.4 LLM Planner

要求を **plan → execute → UI** に分解する設計を確認します。

- [AI に手順を考えさせる (LLM Planner)](llm-planner.md)
- 設計の背景: [LLM Planner 置き換え仕様](llm-planner-spec.md) ([設計](../design/index.md) の章)

### Part 3 の成功判定

- 4 通りの渡し方で、何が PC の外に出たかを比べて説明できる
- 自由文の要求が plan ステップに分解されて実行される
- 実行結果が frontend demo に表示される
- planner_diagnostics でどう判断したかを追える

---

## 用語が分からないとき

[環境構築のミニ辞書](../setup/index.md#このハンズオンで使う用語のミニ辞書) を参照してください。
出てくる用語のほぼ全てを 1 行で説明しています。
