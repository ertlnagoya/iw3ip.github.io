# Workshop（進行設計）

この章は、講師・TA が **ワークショップ全体をどう説明し、どの順番で参加者を各ハンズオンへ案内するか** を整理するための章です。  
最初に全体像を共有し、その後に参加者を Phase 1 / 2 / 3 のどこへ導くかを判断します。

## Workshop と Hands-on の関係

- **Workshop**: 全体説明、学習順、時間配分、分岐、まとめを扱う
- **Hands-on**: 参加者が実際にコマンドを打ち、結果を確認する

基本の流れは次のとおりです。

1. Workshop で全体像を説明する
2. 参加者を適切な Hands-on ページへ案内する
3. 実行結果を共有する
4. Workshop に戻って意味づけと発展課題を整理する

## まず説明すべき全体像

IW3IP の実習は、次の 3 段階で説明すると混乱しにくくなります。

| Phase | 講師が説明すべき中心 | 参加者が体験すること |
|---|---|---|
| Phase 1: データ交換 | データがどこで作られ、どこへ渡るか | カメラやセンサからデータを生成し、結果を確認する |
| Phase 2: イベント共有 | 生データではなくイベント共有へ進む理由 | Consent VC、purpose、監査ログを確認する |
| Phase 3: 知能統合 | 人間の要求を plan と execution に分ける考え方 | request -> plan -> execute -> UI を確認する |

## Phase別の進行カード

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Phase 1 / Data Exchange</div>
    <h3>📡 導入に向いた基本実習</h3>
    <p>データがどこで作られ、どこへ渡るかを、参加者が目で見て理解できる段階です。</p>
    <p class="iw3ip-phase-links"><a href="../hands-on/huskylens2.md">HUSKYLENS2</a> / <a href="../hands-on/webcam.md">USBウェブカメラ</a> / <a href="../hands-on/ha-ssi-publisher.md">HA x SSI Publisher</a> / <a href="../hands-on/ha-demo-simulator.md">HA Demo Simulator</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Phase 2 / Event Sharing</div>
    <h3>🧭 条件付き共有を理解させる段階</h3>
    <p>Consent VC、purpose、監査ログを使って、なぜ共有が許可・拒否されるかを説明します。</p>
    <p class="iw3ip-phase-links"><a href="../hands-on/webcam-event-sharing.md">Webcam Event Sharing</a> / <a href="../hands-on/environment-disaster.md">Environment Disaster</a></p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Phase 3 / Intelligence Integration</div>
    <h3>🧠 全体像を見せる応用段階</h3>
    <p>request、plan、execute、frontend demo を通して、知能統合の全体像を見せる段階です。</p>
    <p class="iw3ip-phase-links"><a href="../hands-on/regional-safety-assistant.md">Regional Safety Assistant</a> / <a href="../hands-on/llm-planner.md">LLM Planner</a></p>
  </div>
</div>

## 推奨する説明順

### 導入

最初に次の 3 点を説明します。

- このワークショップは Phase 1 -> Phase 2 -> Phase 3 で発展すること
- 各 Phase で学ぶ対象が異なること
- 参加者は機材や時間に応じて分岐すること

### 実習前に明確にすること

- 参加者が使える機材は何か
- どこまで体験するか
  - Phase 1 まで
  - Phase 2 まで
  - Phase 3 まで
- frontend demo を見せるか

## 推奨タイムライン（120分）

1. 0-15分: 全体像説明
   - Phase 1 / 2 / 3 の違い
   - 今日どこまで進むか
2. 15-30分: 環境確認
3. 30-55分: Phase 1
4. 55-85分: Phase 2
5. 85-110分: Phase 3 または frontend demo
6. 110-120分: まとめ

## Phaseごとの案内先

### Phase 1: データ交換

参加者に最初に触らせる候補です。

- HUSKYLENS2 がある場合: [HUSKYLENS2サンプル](../hands-on/huskylens2.md)
- USB カメラのみの場合: [USBウェブカメラサンプル](../hands-on/webcam.md)
- Home Assistant と SSI の流れを見せたい場合: [HA x SSI Publisherサンプル](../hands-on/ha-ssi-publisher.md)
- 実機なしで Home Assistant と MQTT の流れを見せたい場合: [Home Assistant Demo Simulator サンプル](../hands-on/ha-demo-simulator.md)
- 閲覧側も見せたい場合: [スマホ閲覧アプリ](../hands-on/mobile-viewer.md)

特に、参加者の PC 構成をそろえやすく、機材依存を避けたい授業では、`Home Assistant Demo Simulator` を Phase 1 / Phase 2 の共通導線として使うと進行が安定します。

### Phase 2: イベント共有

Phase 1 を終えた後に、共有条件と監査を説明する段階です。

- カメラ系イベント共有: [USBウェブカメライベント共有サンプル（Phase 2）](../hands-on/webcam-event-sharing.md)
- 非カメラ系イベント共有: [環境・防災イベント共有サンプル（Phase 2）](../hands-on/environment-disaster.md)

### Phase 3: 知能統合

自然言語要求、plan、execute、frontend demo を扱う段階です。

- 基本構成: [地域安全アシスタントサンプル（Phase 3）](../hands-on/regional-safety-assistant.md)
- 実践: [LLM Plannerハンズオン](../hands-on/llm-planner.md)
- 設計理解: [LLM Planner置き換え仕様](../hands-on/llm-planner-spec.md)

## 最短デモ導線

時間が少ない場合や、最初に全体像を見せたい場合は `assistant-demo` が最も分かりやすい導線です。

```bash
docker compose -f infra/docker-compose.yml --profile assistant-demo up --build -d
```

これで起動されるもの:

- `assistant-demo`
- `llm-mock`
- `assistant-ui`

対応ページ:

- [LLM Plannerハンズオン](../hands-on/llm-planner.md)

## 関連ページ

- [Workshop / 事前準備](prerequisites.md)
- [Workshop / 最短起動](quickstart.md)
- [Workshop / 講師進行ガイド](facilitator-guide.md)
- [Hands-on / 概要](../hands-on/index.md)
- [授業ガイド](../foundations/course-guide.md)
- [学習ロードマップ](../foundations/roadmap.md)
