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

- [Home Assistant Demo Simulator サンプル](ha-demo-simulator.md): 実機なしで始める最短入口
- [HUSKYLENS2サンプル](huskylens2.md): HUSKYLENS2 と PC を使う Phase 1 device page
- [USBウェブカメラサンプル](webcam.md): USB カメラを使う Phase 1 device page
- [HA x SSI Publisherサンプル](ha-ssi-publisher.md): Home Assistant / MQTT / Consent VC / audit log の基本構成
- [スマホ閲覧アプリ](mobile-viewer.md): 共有結果の閲覧確認

### Phase 1 の成功判定

- データまたはイベントファイルが生成される
- API や画面で共有結果を確認できる
- どこでデータが生成され、どこで共有されるか説明できる

## Phase 2: イベント共有

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

#### 設計上の対称性

```
              書き込み (write)              読み出し (read)
              ───────────────────────────   ───────────────────────────
JSON 登録    │ /consents POST              │ (既存対応なし)
ウォレット   │ ConsentVC → PolicyToken     │ ViewerVC → ViewerToken
              ↓ POST /platform/ingest      ↓ GET /platform/data
```

連続 MQTT を wallet モードで回すには **多回利用可の M2M ServiceVC**
が必要で、これは将来のハンズオンで扱う予定です。

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

- [地域安全アシスタントサンプル（Phase 3）](regional-safety-assistant.md): 要求から `plan` と `execute` に進む基本形を学ぶ
- [LLM Plannerハンズオン](llm-planner.md): rule-based planner を LLM planner に差し替える実践例
- [LLM Planner置き換え仕様](llm-planner-spec.md): より厳密に設計意図を理解するための仕様ページ

### Phase 3 の成功判定

- 自然言語要求から `plan` を生成できる
- 条件成立時に `executed` を確認できる
- `planner_diagnostics` の各項目を説明できる
- frontend demo から結果を確認できる

## Workshop との関係

この章は、[Workshop 概要](../workshop/index.md) で示す全体進行のうち、参加者が実際に作業する部分です。  
講師・TA はまず Workshop 側で全体像と分岐を説明し、その後に参加者をこの章の各ページへ案内する想定です。
