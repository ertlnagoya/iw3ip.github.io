# ハンズオン

実際に手を動かして IW3IP を理解する章です。
**基本** で全体像を確認し、**機能拡張** で安全な共有や段階アクセスを扱い、**知能統合** で AI によるデータの意味づけと要求の解釈を扱います。

> 先に [事前準備 (ツールのインストール)](../setup/prerequisites.md) を済ませてから、このページに進んでください。[最短起動 (スタックを動かす)](../setup/quickstart.md) は、このページの順番の中で案内します。

この章のページは、コマンドと期待する出力、確認ポイントを順に示す手順書です。各機能の仕組みと設計判断の理由は [設計](../design/index.md) の章に、手順どおりに動かないときの対処は [困ったとき](../operations/index.md) の章にまとめています。

!!! tip "初めての人の進め方"
    初めての人は、次の順に進みます。

    1. [事前準備 (ツールのインストール)](../setup/prerequisites.md)
    2. 1.1 [実機なしで一連を動かす (HA Demo Simulator)](ha-demo-simulator.md)
    3. [最短起動 (スタックを動かす)](../setup/quickstart.md)
    4. 1.2 の 3 つのページから 1 つ
    5. 1.3 の出品と購入
    6. 1.4 (任意)

    例外: 1.2 で [Home Assistant のデータを取り込む (HA SSI Publisher)](ha-ssi-publisher.md) を選ぶ場合は、1.1 のあと最短起動をせずに 1.2 を行います。HA SSI Publisher のページで起動する publisher は、最短起動の IPFS と同じポート 8080 を使うためです。1.2 を終えたら publisher を止めて最短起動を行い、1.3 に進みます。

    1.2 と 2.1 のように 1 つを選ぶグループでは、画面下の「次へ」が、同じグループの別の選択肢など、次に進むページではないページを指すことがあります。進む先は、各ページの「次に進む」に従ってください。

    Part 2 以降は、必要になったら進みます。

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
      <a href="mobile-viewer/">Mobile Viewer (任意)</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Part 2 / 機能拡張 (v2)</div>
    <h3>同意・条件付き共有・段階アクセス</h3>
    <p>VC とウォレットを使って、目的や信頼度に応じた共有を扱います。途中で止めても問題ありません。</p>
    <p class="iw3ip-phase-links">
      <a href="webcam-event-sharing/">2.1 イベント共有</a> /
      <a href="ha-ssi-wallet/">2.2 同意 (Consent VC)</a> /
      <a href="ha-ssi-viewer/">2.3 役割の分離</a> /
      <a href="marketplace-vc-bridge/">2.4 マーケット × ウォレット</a> /
      <a href="data-user-vc-tiered/">2.5 段階アクセスと来歴</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Part 3 / 知能統合</div>
    <h3>AI に要求を解釈させる</h3>
    <p>人の要求を AI が解釈し、データを材料に判断・処理する応用段階です。Part 2 をすべて終えていなくても始められます。</p>
    <p class="iw3ip-phase-links">
      <a href="local-vlm-distribution/">ローカル VLM でカメラデータを意味づけ</a> /
      <a href="privacy-compare/">AI に渡す前にカメラ画像を守る</a> /
      <a href="regional-safety-assistant/">Regional Safety Assistant</a> /
      <a href="llm-planner/">LLM Planner</a>
    </p>
  </div>
</div>

## 進む順 — 目的別の経路

| 目的 | 進む範囲 | 所要時間 |
|---|---|---|
| まず動かしたい | 1.1 [実機なしで一連を動かす (HA Demo Simulator)](ha-demo-simulator.md) のみ | 約 20 分 |
| 基本を一通り | Part 1 の 1.1〜1.3 (1.2 のデバイスは 1 つ選ぶ) | 約 50〜65 分 (1.1 と 1.2 の合計。1.3 の時間は含まない) |
| スマホでも購入する (任意) | Part 1 の 1.4 | 約 20 分 |
| 安全な共有まで | Part 2 の 2.1〜2.3 (2.1 は 1 ページ選ぶ) | 約 3〜3.25 時間 |
| マーケット連携の v2 まで | Part 2 の 2.4 (Stage 5、6、7、A) | 約 3 時間 45 分 |
| 段階アクセス | Part 2 の 2.5: Stage T の 1 ページ目 | 約 60 分 |
| 画像・動画と意味レベルの段階化 | Part 2 の 2.5: Stage T の続きの 2 ページ | 約 2.5〜3 時間 |
| 画像の来歴を証明する | Part 2 の 2.5: Stage P の P0 | 約 30〜40 分 |
| 来歴を publisher とチェーンにつなぐ | Part 2 の 2.5: Stage P の P1・P2 | 約 40〜60 分 |
| VLM の要約から元の画像をたどる | Part 2 の 2.5: Stage P の P3 | 約 20〜30 分 (本物の VLM を試す場合は、さらに 30 分〜1 時間) |
| AI 統合まで | Part 3 | 約 4〜5 時間 |

所要時間は、その行で進むページの冒頭にある所要時間を合計した目安です。前の行までの時間は含みません。1.3 はこのページの節で、所要時間を示していません。2.1 の 2 ページを両方進める場合は、2.1 で約 75 分かかります。[事前準備 (ツールのインストール)](../setup/prerequisites.md) と [最短起動 (スタックを動かす)](../setup/quickstart.md) (約 15〜30 分) の時間は含みません。

各ハンズオンページの先頭には **「やること」「前提」「起動しておくもの」「使うもの」「所要時間」** を載せています。途中で詰まったら [トラブルシュート](../operations/troubleshooting.md) を参照してください。

## Part / Phase / Stage の関係 { #part-phase-stage }

このサイトでは、3 種類の番号を使っています。

- **Part 1〜3**: このサイトでの学習順です。基本、機能拡張、知能統合の順に進みます。
- **Phase 1〜3**: 基盤の発展段階です。Phase 1 はデータ共有、Phase 2 はイベント共有、Phase 3 は AI による判断と制御を指します（[プラットフォーム概要](../platform-overview.md#フェーズ構成) のフェーズ構成）。
- **Stage 0〜7、A、T、P**: Part 2 のうち、VC を使うハンズオンと、その準備 (Stage 0)、画像の来歴の署名 (Stage P) に付けた番号です。A はスマホアプリ (PWA) で購入から閲覧まで、T は Tier (信頼度に応じた段階アクセス)、P は Provenance (画像の来歴) を扱います。番号は進む順の目安です。どのページを先に済ませておくかは、各ページの冒頭にある「前提」の行に従ってください。たとえば、Stage P の最初のページ (P0) は前の Stage と VC を使わず、教材リポジトリと Python だけで単独で動かせます。続きの P1〜P3 のページは publisher とチェーンにつなぎ、Tier ごとの ViewerToken (PurchaseViewerVC で得る) を使います。

Stage 2 はありません。Stage 4 prep は、LLM Planner の実行を VC で認可する段階 (Stage 4 として計画) の準備にあたる ServiceVC のページです。Stage 8 と Stage 9 は、一部のページに今後の検討事項として出てくる番号です。これらのハンズオンはありません。

| Stage | ハンズオン | 扱う VC |
|---|---|---|
| 0 | [検知イベントだけを共有する](webcam-event-sharing.md)、[防災イベントを共有する](environment-disaster.md) | なし (`/consents` に JSON を登録) |
| 1 | [データ提供者の認証 (Consent VC / Stage 1)](ha-ssi-wallet.md) | ConsentVC |
| 3 | [データ閲覧者の認証 (Viewer VC / Stage 3)](ha-ssi-viewer.md) | ViewerVC |
| 4 prep | [機器間での連携認証 (Service VC / Stage 4 prep)](ha-ssi-service.md) | ServiceVC |
| 5 | [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md) | PurchaseViewerVC |
| 6 | [ServiceVC で書いたデータを PurchaseViewerVC で読み出す (Stage 6)](marketplace-vc-end-to-end.md) | ServiceVC + PurchaseViewerVC (Stage 1・3 の ConsentVC・ViewerVC の流れの上に組み立てる) |
| 7 | [出品者の身元を裏付ける (Seller VC / Stage 7)](marketplace-seller-vc.md) | SellerVC |
| A | [スマホアプリで購入から閲覧まで (Stage A)](marketplace-mobile-app.md) | PurchaseViewerVC (スマホだけで操作) |
| T | [信頼度に応じて見せる中身を変える (DataUserVC / Stage T)](data-user-vc-tiered.md) | DataUserVC、PurchaseViewerVC (受け取りと提示は Stage 5 の Step 7〜8 と同じ操作) |
| P | [画像の来歴を証明する (C2PA / Stage P)](c2pa-provenance.md) | なし (C2PA の署名に X.509 証明書を使う。publisher とウォレットは不要) |
| P | [来歴を publisher とチェーンにつなぐ (C2PA / Stage P の P1・P2)](c2pa-provenance-publisher.md) | PurchaseViewerVC (DataUserVC の属性で Tier が決まる。プログラムの中のウォレットが使うため、スマホは不要) |
| P | [VLM の要約から元の画像をたどる (C2PA / Stage P の P3)](c2pa-provenance-vlm.md) | PurchaseViewerVC (DataUserVC の属性で Tier が決まる。プログラムの中のウォレットが使うため、スマホは不要) |

VC とトークンの対応は [VC アーキテクチャ全体像 (Part 2 / Stage 0〜7, T, A, P)](../design/vc-architecture-overview.md) にまとめています。

## 使う技術要素の早見表

左側のナビゲーションのグループごとに、使う技術要素をまとめます。Part 2 には、ブロックチェーンを使うグループと使わないグループがあります。

| グループ | publisher (Docker) | ブロックチェーン (Hardhat) | MetaMask | SSI ウォレット (スマホ) | IPFS・保管庫 (v1) | 画像処理 / LLM |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| **Part 1 / 1.1 HA Demo Simulator** | ✓ | – | – | – | – | – |
| **Part 1 / 1.2 デバイスからデータを取る (どれか 1 つ)** | ✓ <small>(HA 経路)</small> | ✓ <small>(カメラ経路)</small> | – | – | ✓ <small>(カメラ経路)</small> | – |
| **Part 1 / 1.3〜1.4 マーケットプレイスで売買 (1.3 はハンズオンのページ)** | – | ✓ | ✓ | – | ✓ | – |
| **Part 2 / 2.1 イベント共有 (Stage 0、どれか 1 つ)** | ✓ | – | – | ✓ <small>(補論のみ)</small> | – | – |
| **Part 2 / 2.2〜2.3 同意と役割の分離 (Stage 1・3・4 prep)** | ✓ | – | – | ✓ | – | – |
| **Part 2 / 2.4 マーケットプレイス × ウォレット (v2)** | ✓ | ✓ | ✓ | ✓ | – | – |
| **Part 2 / 2.5 段階アクセスと来歴 (Stage T / P)**<sup>※</sup> | ✓ | ✓ <small>(Tier 3 実機時)</small> | ✓ <small>(同)</small> | ✓ | – | ✓ <small>(OpenCV / VLM 任意)</small> |
| **Part 3: 知能統合 — AI に要求を解釈させる** | ✓ | – | – | – | – | ✓ <small>(VLM / LLM)</small> |

※ このグループのうち、[画像の来歴を証明する (C2PA / Stage P)](c2pa-provenance.md) は Python だけで動き、publisher、ブロックチェーン、MetaMask、SSI ウォレットと VC を使いません。続きの [来歴を publisher とチェーンにつなぐ (C2PA / Stage P の P1・P2)](c2pa-provenance-publisher.md) と [VLM の要約から元の画像をたどる (C2PA / Stage P の P3)](c2pa-provenance-vlm.md) は publisher と Hardhat を使い、プログラムの中のウォレットで PurchaseViewerVC を受け取って Tier ごとの ViewerToken を得ます。MetaMask とスマホのウォレットは使いません。

### 主に使う VC (Verifiable Credential) の種類

| グループ | Consent | Viewer | Service | Purchase&shy;Viewer | Seller | DataUser |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| Part 2 / 2.1 イベント共有 (Stage 0、どれか 1 つ) | ✓ <small>(補論のみ)</small> | – | – | – | – | – |
| Part 2 / 2.2〜2.3 同意と役割の分離 (Stage 1・3・4 prep) | ✓ | ✓ | ✓ | – | – | – |
| Part 2 / 2.4 マーケットプレイス × ウォレット (v2) | – | – | ✓ | ✓ | ✓ | – |
| Part 2 / 2.5 段階アクセスと来歴 (Stage T / P)<sup>※</sup> | – | – | – | ✓ | – | ✓ |

### 表の見方

- **publisher**: ハンズオン用のサーバ。`docker compose up` で起動します。Part 1 のマーケットでの売買 (v1) では使いません。v1 のスタックでは IPFS も起動します。IPFS と publisher はどちらもポート 8080 を使うため、同時には起動できません。
- **ブロックチェーン**: Hardhat ローカルチェーン + デプロイ済コントラクト。マーケットでの売買が起点になるグループで必要です。
- **MetaMask**: ブロックチェーンに対する署名・送金に使います。ローカルチェーンに接続して使います。
- **SSI ウォレット**: スマホに入れる iw3ip-wallet (Sphereon Wallet の fork) です。VC の受け取りと提示に使います。
- **IPFS・保管庫 (v1)**: v1 で使います。購入後のデータは保管庫 (simple-storage) に置き、その場所 (URI) を暗号化してチェーンに書き込みます。IPFS と PostgreSQL は、動画のメタデータの登録と検索画面に使います。v2 では VC を使う経路が加わります。
- **カメラ経路 / HA 経路**: Part 1 の HUSKYLENS2 と USB ウェブカメラはイベントファイルを `mediator-owner` に渡すので、[最短起動 (スタックを動かす)](../setup/quickstart.md) のスタック (Hardhat、保管庫、IPFS など) が必要です。HA SSI Publisher は publisher にデータを送ります。
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

1. **出品**: 1.2 のサンプル ([AI カメラで人物を検知する (HUSKYLENS2)](huskylens2.md) または [USB カメラでポイ捨てを検知する (OpenCV)](webcam.md)。mock モードで可) を動かすと、イベントファイルが `mediator-owner/raw_data/output` に出力されます。1.2 で HA SSI Publisher を選んだ場合は、[USB カメラでポイ捨てを検知する (OpenCV)](webcam.md) のサンプルを mock モードで動かして出品します。`mediator-owner` がイベントファイルを検知して商品として登録し、ターミナルに商品のアドレスを表示します。

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

各プロセスの起動手順は [最短起動 (スタックを動かす)](../setup/quickstart.md) にあります。

1.3 のあとに 1.4 (任意) に進む場合は、最短起動のスタックを動かしたままにします。Part 2 に進む前には、最短起動の IPFS を止めます (`ipfs` ディレクトリで `docker compose down`)。Part 2 のページで起動する publisher が、IPFS と同じポート 8080 を使うためです。最短起動のフロントエンド (ターミナル 3、ポート 5173) も Ctrl+C で止めます。2.4 の [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md) の Step 6 で、iot-market-ui を 5173 番で起動し直すためです。

### 1.4 スマホでマーケットを開いて購入する

この節は任意です。スマホに MetaMask モバイルを入れて試します。Part 1 の成功判定には含みません。

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

### 2.5 段階アクセスと来歴 (Stage T / P) { #25-信頼度に応じた段階アクセス-stage-t--p }

利用者の **DataUserVC** に基づき、見せ方を段階的に変えます。最初の 2 ページは、Tier 3 (イベント・画像・動画) / Tier 2 (イベント・画像) / Tier 1 (イベントのみ。判定は `denied`) の 3 段階です。3 ページ目では VLM の拡張を有効にし、Tier 3 / Tier 2 (イベント・ぼかした画像・説明文) / Tier 1 (イベント・要約の文) / Tier 0 (イベントのみ。判定は `denied`) の 4 段階になります。
最初のページでは、PurchaseViewerVC を 2.4 の [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md) の Step 7〜8 と同じ操作で受け取り、提示します。
後半では、画像の内容を構造化したデータ（意味的中間表現）を作り、信頼度に応じて見せ方を変える処理（trust-aware rendering）も扱います。

- [信頼度に応じて見せる中身を変える (DataUserVC / Stage T)](data-user-vc-tiered.md)
- [画像・動画を段階アクセスで配信する (DataUserVC / Stage T)](data-user-vc-tiered-media.md) — 続きの §8〜§11
- [意味レベルで段階化する (VLM と意味的中間表現 / Stage T)](data-user-vc-tiered-semantic.md) — 続きの §12〜§13
- 設計の背景: [段階アクセスの設計仕様](data-user-vc-tiered-spec.md) ([設計](../design/index.md) の章)
- [画像の来歴を証明する (C2PA / Stage P)](c2pa-provenance.md) — P0。段階アクセスで配る画像に、C2PA の来歴を付けて署名する (publisher やウォレットは不要)
- [来歴を publisher とチェーンにつなぐ (C2PA / Stage P の P1・P2)](c2pa-provenance-publisher.md) — publisher で来歴を検証し、Tier 2 の画像に署名し、トークンをチェーンに記録して、Tier ごとの来歴の見え方を確かめる
- [VLM の要約から元の画像をたどる (C2PA / Stage P の P3)](c2pa-provenance-vlm.md) — VLM の要約を描いた要約カードに publisher が署名し、要約の文だけを受け取った人が撮影画像とトークンまでたどる

### Part 2 の成功判定

- Consent / DataUserVC / PurchaseViewerVC のいずれかをスマホ wallet に取り込めた
- `purpose` を変えると、共有 (`/simulate/publish`) や提示 (`/verifier/request`) の結果が `allow` / `deny` に切り替わり、信頼度を変えると `/platform/data` で見える内容が変わる
- audit log に「誰が・いつ・どの purpose で・何を見たか」が残っている

---

## Part 3: 知能統合 — AI に要求を解釈させる

Part 3 は、Part 2 をすべて終えていなくても始められます。3.1 と 3.2 は、Docker、Python、USB ウェブカメラがあれば動きます (3.2 はカメラの代わりに JPEG 画像でも動きます)。最短起動のスタックは使いません。3.3 はサンプルのイベントファイルを使い、2.1 のどちらか 1 ページを前提にしています ([検知イベントだけを共有する](webcam-event-sharing.md) を済ませていると、`possible_littering` の例が分かりやすくなります)。3.4 は 3.3 を前提にしています。

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

[環境構築](../setup/index.md#このハンズオンで使う用語のミニ辞書) のミニ辞書を参照してください。
出てくる用語のほぼ全てを 1 行で説明しています。
