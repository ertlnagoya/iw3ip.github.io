# Hands-on（実習手順）

この章は、参加者が実際に手を動かしながら IW3IP を理解するための実習集です。  
最初に **Phase 1 -> Phase 2 -> Phase 3** という全体像を理解し、その後に各ページへ進む構成にします。

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
    <p class="iw3ip-phase-links"><a href="webcam-event-sharing.md">Webcam Event Sharing</a> / <a href="environment-disaster.md">Environment Disaster</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Phase 3 / Intelligence Integration</div>
    <h3>🧠 要求を解釈して判断・制御する</h3>
    <p>人間の要求を AI が解釈し、plan、execute、frontend demo までつなぐ段階です。</p>
    <p class="iw3ip-phase-links"><a href="regional-safety-assistant.md">Regional Safety Assistant</a> / <a href="llm-planner.md">LLM Planner</a></p>
  </div>
</div>

## まず理解してほしいこと

- **Phase 1** は「データをどう渡すか」を学ぶ段階です。
- **Phase 2** は「どのイベントを、どの条件で共有するか」を学ぶ段階です。
- **Phase 3** は「人間の要求を AI がどう解釈し、複数の処理へ分解するか」を学ぶ段階です。

つまり、

1. データを扱えるようにする
2. イベントとして条件付き共有できるようにする
3. 要求理解と制御まで含める

という順に難しくなります。

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

### 対応するハンズオン

- [HUSKYLENS2サンプル](huskylens2.md): HUSKYLENS2 と PC を使ってイベントファイルを生成する
- [USBウェブカメラサンプル](webcam.md): Web カメラでイベント候補を扱う基本形を学ぶ
- [HA x SSI Publisherサンプル](ha-ssi-publisher.md): Home Assistant / MQTT / Consent VC / audit log の基本構成を学ぶ
- [Home Assistant Demo Simulator サンプル](ha-demo-simulator.md): 実機なしで Home Assistant demo から Phase 1 / Phase 2 / Phase 3 をつなぐ
- [スマホ閲覧アプリ](mobile-viewer.md): 共有された結果をスマホから確認する

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

- [USBウェブカメライベント共有サンプル（Phase 2）](webcam-event-sharing.md): `possible_littering` をイベントとして共有する
- [環境・防災イベント共有サンプル（Phase 2）](environment-disaster.md): `flood_risk_high` をイベント共有として扱う

### Phase 2 の成功判定

- `allowed` / `denied` の両方を再現できる
- `purpose` と `dataset_id` の違いを説明できる
- `/audit/logs` や `/platform/ingest` の役割を説明できる

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
