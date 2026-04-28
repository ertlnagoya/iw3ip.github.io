# Hands-on（実習手順）

参加者が手を動かしながら IW3IP を理解するための実習集です。  
**Phase 1 → Phase 2 → Phase 3** の順に難度が上がります。

## この章の全体像

IW3IP のハンズオンは、次の 3 段階で発展します。

| Phase | 何を学ぶか | 代表的な確認内容 |
|---|---|---|
| Phase 1: データ交換 | センサやカメラから得たデータを基盤へ渡す | データが生成される、共有される、閲覧できる |
| Phase 2: イベント共有 | 生データではなくイベントや推論結果を共有する | `allowed` / `denied`、Consent、監査ログ |
| Phase 3: 知能統合 | 人間の要求を解釈し、収集・判断・制御まで行う | `plan`、`execute`、`planner_diagnostics`、UI |

## Phase別の見取り図

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Phase 1 / Data Exchange</div>
    <h3>📡 データを集めて渡す</h3>
    <p>まずは、カメラやセンサからデータを取り出し、共有基盤へ渡す基本経路を理解します。</p>
    <p>実機なしで始める場合は <a href="ha-demo-simulator.md">Home Assistant Demo Simulator</a> が最短です。</p>
    <p class="iw3ip-phase-links"><a href="ha-demo-simulator.md">HA Demo Simulator</a> / <a href="huskylens2.md">HUSKYLENS2</a> / <a href="webcam.md">USBウェブカメラ</a> / <a href="ha-ssi-publisher.md">HA x SSI Publisher</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Phase 2 / Event Sharing</div>
    <h3>🧭 イベントとして条件付き共有する</h3>
    <p>生データ全量ではなく、イベントや推論結果を目的・同意・監査付きで共有する考え方を学びます。</p>
    <p class="iw3ip-phase-links"><a href="webcam-event-sharing.md">Webcam Event Sharing</a> / <a href="environment-disaster.md">Environment Disaster</a> / <a href="ha-ssi-wallet.md">HA x SSI Wallet</a> / <a href="ha-ssi-viewer.md">HA x SSI Viewer</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Phase 3 / Intelligence Integration</div>
    <h3>🧠 要求を解釈して判断・制御する</h3>
    <p>人間の要求を AI が解釈し、plan、execute、frontend demo までつなぐ段階です。</p>
    <p class="iw3ip-phase-links"><a href="regional-safety-assistant.md">Regional Safety Assistant</a> / <a href="llm-planner.md">LLM Planner</a></p>
  </div>
</div>

## 各 Phase の位置づけ

| Phase | 主題 |
|---|---|
| Phase 1 | データをどう渡すか |
| Phase 2 | どのイベントを、どの条件で共有するか |
| Phase 3 | 人間の要求を AI がどう解釈し、処理に分解するか |

## 学習順の推奨

### はじめて触る人

1. 実機なしで始めるなら [Home Assistant Demo Simulator サンプル](ha-demo-simulator.md)
2. 実機を使うなら [HUSKYLENS2サンプル](huskylens2.md) または [USBウェブカメラサンプル](webcam.md)
3. [HA x SSI Publisherサンプル](ha-ssi-publisher.md)
4. [USBウェブカメライベント共有サンプル（Phase 2）](webcam-event-sharing.md) または [環境・防災イベント共有サンプル（Phase 2）](environment-disaster.md)
5. [地域安全アシスタントサンプル（Phase 3）](regional-safety-assistant.md)
6. [LLM Plannerハンズオン](llm-planner.md)

### 最短で Phase 3 を見たい人

`assistant-demo` を使うと、Phase 3 の最短デモを 1 コマンドで起動できます。

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

開く URL:

- `http://localhost:4173`

対応ページ:

- [LLM Plannerハンズオン](llm-planner.md)

## Phase 1: データ交換

### この Phase の目的

- カメラやセンサからデータを取り出す
- データ共有基盤へ渡す前の基本的な流れを理解する
- 出力ファイルや画面表示など、目に見える結果を確認する

### Phase 1 のページ構成

<div class="iw3ip-subphase-grid">
  <div class="iw3ip-subphase-card">
    <div class="iw3ip-subphase-kicker">最初に試す入口</div>
    <h4>実機なしで始める</h4>
    <p>まずは配線やカメラ準備なしで全体像を確認したい場合の入口です。Consent、MQTT、監査ログまで含めて順に確認できます。</p>
    <p><a href="ha-demo-simulator.md">Home Assistant Demo Simulator サンプル</a>: 実機なしで Home Assistant demo から Phase 1 / Phase 2 / Phase 3 をつなぐ</p>
    <p><a href="ha-ssi-publisher.md">HA x SSI Publisherサンプル</a>: Home Assistant / MQTT / Consent VC / audit log の基本構成を学ぶ</p>
  </div>
  <div class="iw3ip-subphase-card">
    <div class="iw3ip-subphase-kicker">デバイス入力の基本</div>
    <h4>Phase 1 device pages</h4>
    <p>実機や mock 入力を使って、イベントファイル生成から商品化までの流れを確かめるページ群です。まず `mock`、次に実機、最後にトラブル切り分けという順で読むと迷いにくくなります。</p>
    <p><a href="huskylens2.md">HUSKYLENS2サンプル</a>: HUSKYLENS2 と PC を使ってイベントファイルを生成する</p>
    <p><a href="webcam.md">USBウェブカメラサンプル</a>: USB カメラでイベント候補を扱う基本形を学ぶ</p>
  </div>
  <div class="iw3ip-subphase-card">
    <div class="iw3ip-subphase-kicker">結果の見え方</div>
    <h4>閲覧と確認</h4>
    <p>共有された結果や商品化の見え方を別の端末から確かめる段階です。データが生成された後に、どこで確認するかを整理できます。</p>
    <p><a href="mobile-viewer.md">スマホ閲覧アプリ</a>: 共有された結果をスマホから確認する</p>
  </div>
</div>

### 対応するハンズオン

Phase 1 は **データの発生から共有までの一往復**を体験する Phase です。
ハンズオンは依存順で並べると次のようになります (前段ほど薄く、
後段に向かって構成要素が増えます)。

#### 段階ごとに追加される機能と学べること

##### 段階 1A: 入力ソースを 1 つ動かす (`huskylens2` / `webcam` / `ha-demo-simulator`)

- **追加される機能**
    - 入力デバイス → JSON ペイロードの生成
    - イベントファイル / シミュレータ出力の確認
    - (ha-demo-simulator のみ) Home Assistant コンテナ + Mosquitto が
      実機なしで起動
- **学べること**
    - センサーや画像入力からデータが生まれる地点
    - JSON ペイロードのスキーマ (timestamp, payload, source)
    - 実機を持っていないときの代替入口 (シミュレータ)

##### 段階 1B: パイプラインに繋ぐ (`ha-ssi-publisher`)

- **段階 1A から追加される機能**
    - MQTT subscriber → publisher → `/platform/ingest` の一連のフロー
    - `/consents` JSON 登録による許可データの絞り込み
    - 監査ログ (`/audit/logs`) への記録 (purpose, allow/deny)
    - Consent VC の有効期間 (`valid_from` / `valid_to`) チェック
- **学べること**
    - 「入力 → 共有判定 → 配信 → 監査」の最小パイプライン
    - Consent VC の役割と `purpose`, `dataset_id` の組み合わせ
    - publisher コンテナと外部 API (`PLATFORM_API_URL`) の関係

##### 段階 1C: 結果の閲覧 (`mobile-viewer`)

- **段階 1B から追加される機能**
    - スマホブラウザから `iot-market-ui` 画面を開く確認手順
    - LAN IP / 0.0.0.0 公開設定の整備
- **学べること**
    - 共有結果が「どこから見えるのか」というユーザ視点
    - PC とスマホをまたぐネットワーク構成の最低条件
    - (注) この段階は閲覧確認用で、認可はまだ介在しません。
      Phase 2 Stage 3 (ha-ssi-viewer) で読み出しに VC が入ります

### Phase 1 の成功判定

- データまたはイベントファイルが生成される
- API や画面で共有結果を確認できる
- どこでデータが生成され、どこで共有されるか説明できる
- `/consents` 登録の意味と監査ログへの反映を説明できる

## Phase 2: イベント共有

!!! tip "全体像を 1 ページで"
    7 ステージ・5 種 VC・4 種トークン・v1/v2 マーケット・audit log 連鎖の俯瞰は
    [VC アーキテクチャ全体像](../design/vc-architecture-overview.md) を参照。

### この Phase の目的

- 生データ全量共有ではなく、イベント共有へ進む
- Consent VC と purpose による制御を理解する
- `allowed` / `denied` と監査ログの意味を説明できるようにする

### 対応するハンズオン

Phase 2 は 4 つのハンズオンで構成され、**同じ認可ロジック (dataset × purpose)
を 2 つの伝達経路 (JSON 登録 vs ウォレット提示) と 2 つの方向
(書き込み vs 読み出し) で体験する**構造になっています。

| ハンズオン | 認可の保持 | 伝達経路 | ゲート対象 | トークン |
| --- | --- | --- | --- | --- |
| [USBウェブカメライベント共有](webcam-event-sharing.md) | publisher サーバ | `/consents` JSON | 書き込み (`POST /platform/ingest`) | なし (登録ベース) |
| [環境・防災イベント共有](environment-disaster.md) | publisher サーバ | `/consents` JSON | 書き込み | なし |
| [スマホSSIウォレットサンプル (Stage 1)](ha-ssi-wallet.md) | スマホウォレット | OID4VP 提示 | 書き込み | PolicyToken (5 分・単回) |
| [SSI ビューワサンプル (Stage 3)](ha-ssi-viewer.md) | スマホウォレット | OID4VP 提示 | 読み出し (`GET /platform/data`) | ViewerToken (60 秒・多回) |
| [SSI サービスサンプル (Stage 4 prep)](ha-ssi-service.md) | サービス holder | OID4VP 提示 | 書き込み (連続) | ServiceToken (1 時間・多回) |
| [マーケット連携 end-to-end (Stage 6)](marketplace-vc-end-to-end.md) | 両側 (seller / buyer) | OID4VP 提示 + MetaMask | 書き + 読み | ServiceToken & PurchaseViewerToken |
| [マーケット Seller VC (Stage 7)](marketplace-seller-vc.md) | seller (出品身元) | OID4VP 提示 + `/marketplace/register` | 出品ガバナンス | SellerToken (24 時間・多回) |

#### 段階ごとに追加される機能と学べること

以下の順で進めると、**前の段階に少しずつ機能を足していく形**で
イベント共有の認可モデルを段階的に深められます。

##### 段階 0: ベースライン (`webcam-event-sharing` / `environment-disaster`)

- **追加される機能**
    - publisher への `/consents` JSON 登録 API
    - MQTT トピック → `dataset_id` 正規化 → `(dataset_id, purpose)`
      ペアによる `allow` / `deny` 判定
    - `/audit/logs` への判定結果の永続化
- **学べること**
    - 「生データ共有」と「イベント共有」の違い
    - 同意 (consent)、目的 (purpose)、データセット (dataset) という
      3 軸の役割
    - 監査ログ (`raw_topic`, `purpose`, `reason`) の構造と読み方

##### 段階 1: ウォレット書き込み認可 (`ha-ssi-wallet`)

- **段階 0 から追加される機能**
    - OID4VCI による ConsentVC の発行 (`/issuer/offer` 等)
    - OID4VP による ConsentVC の提示 (`/verifier/request` / `/verifier/response`)
    - 短命 PolicyToken (5 分・単回利用) の発行
    - `POST /platform/ingest` の Bearer 認証 (PolicyToken)
    - 監査ログに `holder_did` / `vc_hash` / `presentation_verified` を追加
- **学べること**
    - 同意主体の身元が VC で裏付けられる意味 (誰が同意したかの証跡)
    - SD-JWT VC、`did:jwk`、DCQL の概要
    - 「単回消費トークン」が書き込み認可に向く理由
    - JSON 登録方式とウォレット方式の **等価性と差異** (両方の対比表)

##### 段階 2: 既存ハンズオンへのウォレット連携 (補論セクション)

- **段階 1 から追加される機能** (ドキュメントのみ、バックエンド変更なし)
    - `webcam-event-sharing.md` / `environment-disaster.md` 末尾に
      「補論: ウォレットモードによる認可」を追加
    - 1 件のイベントを curl で wallet 経由 ingest する手順
- **学べること**
    - 既存サンプルが wallet 化したときに何がそのままで何が変わるか
    - 「連続 MQTT を wallet で回す」には別仕組み (M2M ServiceVC) が
      要る理由 (PolicyToken の単回消費仕様の限界)

##### 段階 3: ウォレット読み出し認可 (`ha-ssi-viewer`)

- **段階 1 から追加される機能**
    - ViewerVC (read 用 VC、claim: `dataset_id`, `allowed_actions=["read"]`)
    - `vc_kind=ViewerVC` でディスパッチする `/verifier/request`
    - ViewerToken (60 秒・TTL 内多回利用)
    - `GET /platform/data?dataset_id=...` の Bearer 認証 (ViewerToken)
    - 監査ログ `raw_topic=platform/data` の read イベント記録
    - issuer metadata に ConsentVC + ViewerVC の両方を露出
- **学べること**
    - 「書き込み認可」と「読み出し認可」を VC で対称に扱う設計
    - 単回消費 (write) vs TTL 内多回利用 (read) のセマンティクスの差
    - VC の役割分離 (ConsentVC vs ViewerVC) と PolicyToken / ViewerToken
      の token 空間の独立性 (流用拒否)

##### 段階 7: 出品身元の SellerVC (`marketplace-seller-vc`)

詳細は [SellerVC 設計仕様](../design/seller-vc-spec.md) と
[ハンズオン](marketplace-seller-vc.md)。

- **段階 6 から追加される機能**
    - **SellerVC** (5 種目): claim `seller_id`, `licensed_datasets`, `subject_id`
    - **SellerToken** (24 時間・多回利用、`/marketplace/register` 専用)
    - **`POST /marketplace/register`**: Bearer SellerToken で
      Merchandise ↔ seller_did binding。`licensed_datasets` で dataset gate。
      (任意) on-chain `Merchandise.getOwner()` 検証
    - 監査ログに **`raw_topic=marketplace/seller_registered`** + `owner_verify=verified|skipped|rpc_failed`
    - `/platform/data?merchandise=<addr>` レスポンスに **`seller_did`** が同梱
    - iot-market-ui に `/seller` ページ (フォーム 3 ステップ)
- **学べること**
    - 「出品する権利」を VC で表現する方法 (`licensed_datasets`)
    - publisher が **off-chain** で seller 身元を裏付け、buyer 側に
      seller_did を可視化する仕組み
    - `MARKETPLACE_HARDHAT_RPC` 設定時の **on-chain `getOwner()` 検証** の
      役割 (なりすまし防止の最低線)
    - 5 つの VC kind (Consent / Viewer / Service / PurchaseViewer / Seller) が
      役割分担で完成する全体像

##### 段階 6: 4 種 VC を 1 動線で繋ぐ end-to-end (`marketplace-vc-end-to-end`)

Stage 1〜5 の集大成。詳細は [ハンズオン](marketplace-vc-end-to-end.md)。

- **段階 4 prep + 段階 5 から追加される機能** (Stage 6 case B)
    - Merchandise の `additionalInfo` に `dataset_id` を含めて on-chain 公開
    - bridge listener が `getAllAdditionalInfo()` で dataset_id を解決
      (per-merchandise キャッシュ、fallback あり)
    - iot-market-ui が `additionalInfo` から dataset を抽出して redirect
    - 5 つの Merchandise を 3 dataset に分散して deploy
- **学べること**
    - **Seller の ServiceVC 書き込み → Buyer の PurchaseViewerVC 読み出し**が
      同一 dataset を介して連鎖する全体像
    - dataset_id がハードコードでなく **on-chain 由来**であることの利点
    - 1 dataset を巡る audit ログの **多層チェーン** (write x N → claim →
      issued → oid4vp → data) の読み解き

##### 段階 5: マーケット連携によるデータ受け渡し v2 (`marketplace-vc-bridge`)

詳細は [Marketplace VC Bridge 設計仕様](../design/marketplace-vc-bridge-spec.md)
と [ハンズオン](marketplace-vc-bridge.md) を参照。

- **段階 1/3 から追加される機能**
    - **bridge service**: Hardhat 上の `Merchandise.Purchase` event を購読
    - **PurchaseViewerVC**: 購入連動の閲覧用 VC (`merchandise_address`,
      `tx_hash`, `buyer_eth_addr` を claim に持つ、4 つ目の VC kind)
    - `POST /marketplace/claim`: bridge → publisher 連携 endpoint
      (`tx_hash` で冪等)
    - `GET /platform/data?merchandise=<addr>`: 購入連動データ取得 API
      (merchandise から dataset_id を逆引き)
    - iot-market-ui の `/purchased/[txHash]` 画面で OID4VCI deeplink + QR
    - audit log に `eth_addr ↔ did:jwk` の紐付け
      (`raw_topic=marketplace/issued`, `eth_did_bound`)
- **学べること**
    - **v1 (現行マーケット + MetaMask + 暗号化 IPFS)** と **v2 (VC 連携)** の対比
    - 同一人物が ETH 鍵と did:jwk 鍵の **2 つの身元**を持つことの意味
    - ETH 支払いと VC 認可の **役割分離** (支払いと閲覧権限が別レイヤー)
    - bridge service と iot-market-ui が同じ `/marketplace/claim` を叩いても
      冪等性で整合する設計
    - 1 購入で **4 行**の audit log が連鎖する全体像

##### 段階 4 prep: M2M 連続書き込み認可 (`ha-ssi-service`)

- **段階 1/3 から追加される機能**
    - ServiceVC (M2M 用 VC、claim: `dataset_id`, `allowed_actions=["write_continuous"]`)
    - `vc_kind=ServiceVC` でディスパッチする `/verifier/request`
    - ServiceToken (1 時間・TTL 内多回利用)
    - `/platform/ingest` が PolicyToken と ServiceToken の両方を受理
      (PolicyToken → 不一致なら ServiceToken の順)
    - 監査ログ `reason=service_token_used:<jti>:<write_count>`
    - issuer metadata がこの段階で ConsentVC / ViewerVC / ServiceVC の 3 種を露出 (Stage 5/7 でさらに 2 種追加され最終的に 5 種)
- **学べること**
    - 「人の VC」と「サービスの VC」を分ける動機
    - 単回 / 多回 read / 多回 write の **3 つのトークン形態**
    - PolicyToken と ServiceToken の独立空間で同じ ingest API を共有する仕組み
    - publisher 内蔵 holder の必要性 (現状未実装、将来課題)

#### 設計上の対称性

```
              書き込み (write, 単回)        書き込み (write, 連続)        読み出し (read)
              ───────────────────────────   ───────────────────────────   ───────────────────────────
JSON 登録    │ /consents POST              │ (既存対応なし)              │ (既存対応なし)
ウォレット   │ ConsentVC → PolicyToken     │ ServiceVC → ServiceToken    │ ViewerVC → ViewerToken
              ↓ POST /platform/ingest      ↓ POST /platform/ingest       ↓ GET /platform/data
```

`/platform/ingest` は PolicyToken と ServiceToken の両方を受け付け、
PolicyToken を試した後に ServiceToken にフォールバックする仕様です。

### Phase 2 の成功判定

- `allowed` / `denied` の両方を再現できる
- `purpose` と `dataset_id` の違いを説明できる
- `/audit/logs` や `/platform/ingest` の役割を説明できる
- ウォレット経由 (Stage 1/3) と JSON 登録経由の **等価性と差異** を説明できる

## Phase 3: 知能統合

### この Phase の目的

- 人間の自然言語要求を plan に変換する
- イベント収集、判定、制御を 1 つの流れとして理解する
- LLM を planner に差し替える設計意図を理解する

### 対応するハンズオン

Phase 3 は **要求 → 計画 → 実行**の 1 ループを段階的に組み立てる
Phase です。最初に rule-based の最小形を動かし、次に LLM へ差し替え、
最後に仕様面を読みます。

#### 段階ごとに追加される機能と学べること

##### 段階 3A: 要求 → plan → execute の最小ループ (`regional-safety-assistant`)

- **追加される機能**
    - `assistant` サービス (`/plan`, `/execute` API)
    - 自然言語要求を **rule-based で plan に変換**するパス
    - Phase 2 で蓄積されたイベント (`/audit/logs` / `/platform/ingest`)
      を planner が参照
    - frontend demo (`assistant` web UI) で plan 結果を可視化
- **学べること**
    - 「要求 → plan → execute → 検証」のループ構造
    - assistant が Phase 2 のイベントをどう材料にするか (依存方向)
    - rule-based planner の表現力の限界 (条件分岐や長い文脈で破綻)

##### 段階 3B: planner を LLM に差し替える (`llm-planner`)

- **段階 3A から追加される機能**
    - rule-based planner と同じ I/O を持つ LLM planner 実装
    - LLM が呼べるツール (Phase 2 の `/audit/logs` 検索など) の宣言
    - `planner_diagnostics` (LLM の判断トレース) の出力
    - 自然言語のゆらぎや複合条件への対応
- **学べること**
    - 「同じ I/O 契約で planner を入れ替える」設計の利点
    - LLM ツール呼び出し (function calling) と plan の対応関係
    - `planner_diagnostics` を読んで LLM の判断過程を追う方法
    - rule-based / LLM の **適材適所** (どちらを選ぶ判断軸)

##### 段階 3C: 仕様レベルの理解 (`llm-planner-spec`)

- **段階 3B から追加される機能** (ドキュメントのみ)
    - LLM planner の入出力契約、エラーケース、評価指標の文書化
- **学べること**
    - 自分で planner を再実装するときの仕様面の指針
    - rule-based / LLM 以外の方式 (ハイブリッド、検証付き LLM など)
      へ拡張するための地ならし

##### (将来) 段階 3D: VC を絡めた認可付き plan 実行 (Stage 4 / 計画中)

Phase 2 の Stage 1/3 で導入した PolicyToken / ViewerToken を、
Phase 3 の plan 実行ステップに紐付ける拡張を予定しています。
plan の各ツール呼び出しが要求する dataset スコープを VC で証明
させ、planner が認可済みの操作だけを実行できるようにする狙いです。
詳細は将来のハンズオンで扱います。

### Phase 3 の成功判定

- 自然言語要求から `plan` を生成できる
- 条件成立時に `executed` を確認できる
- `planner_diagnostics` の各項目を説明できる
- frontend demo から結果を確認できる
- rule-based / LLM の差を I/O 契約と適材適所の観点で説明できる

## Workshop との関係

この章は、[Workshop 概要](../workshop/index.md) で示す全体進行のうち、参加者が実際に作業する部分です。  
講師・TA はまず Workshop 側で全体像と分岐を説明し、その後に参加者をこの章の各ページへ案内する想定です。
