# Home Assistant Demo Simulator サンプル（Phase 1 / Phase 2 / Phase 3）

## 目的

この Hands-on は、**実機センサやカメラがなくても Home Assistant と IW3IP を連携できる最小シミュレーション環境**を体験するためのものです。

Home Assistant の `demo` エンティティを使い、次の流れをローカル PC 上で再現します。

`Home Assistant demo -> MQTT -> (任意 Node-RED) -> Data Publisher -> Platform API / Audit Log`

このページでは、Phase 1 の状態共有、Phase 2 のイベント共有、Phase 3 の assistant 実行までを同じ環境で確認します。

## 最短ルート

最初は次の 5 手順だけで十分です。

1. `docker compose -f infra/docker-compose.yml --profile ha-demo up --build -d`
2. `http://localhost:8123` を開き、`MQTT` integration を追加する
3. Consent VC を登録する
4. `Developer Tools -> Actions` から `script.iw3ip_publish_demo_temperature` または `script.iw3ip_publish_demo_possible_littering` を実行する
5. `curl http://localhost:8080/platform/ingest` で共有結果を見る

分岐:

- Phase 1 だけ確認したい場合: `temperature` と `power` を送って `platform/ingest` を見る
- Phase 2 まで進む場合: `possible_littering` を送って `allow` と `deny` の両方を見る
- Phase 3 まで進む場合: `ha-demo-phase3` を起動して `run_phase3_from_ingest.py --plan-only` を実行する

## Phase 別の目次

<details class="iw3ip-toc-details" open>
  <summary>Phase 1: 状態共有までを確認する</summary>
  <p>この段階では、環境起動、Home Assistant 初期設定、Consent VC 登録、基本的な script 実行、`/platform/ingest` の確認まで進みます。</p>
  <ol>
    <li><a href="#1-起動">起動</a></li>
    <li><a href="#2-home-assistant-の初期設定">Home Assistant の初期設定</a></li>
    <li><a href="#3-consent-vc-を登録">Consent VC を登録</a></li>
    <li><a href="#4-home-assistant-から-demo-データを送る">Home Assistant から demo データを送る</a></li>
    <li><a href="#5-publisher-側の結果を確認">publisher 側の結果を確認</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Phase 2: イベント共有と拒否を確認する</summary>
  <p>この段階では、許可される共有だけでなく、Consent VC に一致しない要求が拒否されることも確認します。</p>
  <ol>
    <li><a href="#6-拒否ケースを確認">拒否ケースを確認</a></li>
    <li><a href="#7-mqtt-経路を直接試す">MQTT 経路を直接試す</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>Phase 3: assistant の plan と execute を確認する</summary>
  <p>最後に、publisher に蓄積されたイベントを assistant 側へ橋渡しし、人間の要求から `plan` と `execute` がどう分かれるかを確認します。</p>
  <ol>
    <li><a href="#8-phase-3-を試す">Phase 3 を試す</a></li>
  </ol>
</details>

## このページで分かること

- 実機がなくても `temperature`、`power`、`person_detected`、`flood_risk_high`、`possible_littering` を再現できること
- Home Assistant から MQTT へ流したデータを、IW3IP publisher がどう受け取り、正規化し、記録するか
- Phase 1 の状態共有と Phase 2 のイベント共有の違い
- Consent VC によって `allowed` / `denied` がどう変わるか
- `/platform/ingest` に蓄積したイベントを Phase 3 の `assistant` へ橋渡しする方法

## つまずきやすい点

- Home Assistant を起動しただけでは MQTT publish は動かず、最初に `MQTT` integration の追加が必要です
- Consent VC を登録していないと、スクリプトを実行しても期待した共有結果になりません
- Home Assistant 側ではイベントを送れていても、`purpose` が合わなければ publisher 側では拒否されます
- 実機がないぶん流れが見えにくいので、`/platform/ingest` と `/audit/logs` を毎回確認するのが重要です
- Phase 3 では publisher と assistant を別サービスとして見る必要があり、`share` と `execute` を混同しやすいです

## 前提

- Docker / Docker Compose が使える
- ブラウザで `http://localhost:8123` を開ける
- `curl` が使える

この Hands-on は、ソースコードリポジトリ `Blockchain_IoT_Marketplace` の `codex/ha-demo-simulator` ブランチを使う想定です。

対応する主なファイル:

- [README（英語）](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/README.md)
- [README（日本語）](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/README_ja.md)
- [examples/ha_demo/README.md](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/README.md)
- [configuration.yaml](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/home-assistant-demo/config/configuration.yaml)
- [scripts.yaml](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/home-assistant-demo/config/scripts.yaml)
- [run_phase3_from_ingest.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/run_phase3_from_ingest.py)

## 読み進め方

このページは長めの手順書です。短時間で確認したい場合は `最短ルート` と `Phase 別の目次` を先に見て、必要な段階だけを開いて進めてください。

一方で、ワークショップや自主学習で全体像まで理解したい場合は、Phase 1 から順に読み進める方が流れをつかみやすくなります。特に、`allowed` と `denied`、さらに `plan` と `execute` の違いは、前の段階を見ておくと理解しやすくなります。

## Phase 1: 状態共有を確認する

Phase 1 では、Home Assistant から publisher までの基本経路が正しく動いていることを確かめます。まずは状態データを 1 つ送って、`platform/ingest` に記録されることを見れば十分です。

## 1. 起動

まずはシミュレーション環境を起動します。

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo up --build -d
```

Node-RED も一緒に使う場合:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo --profile nodered up --build -d
```

確認:

```bash
curl http://localhost:8080/health
```

期待結果:

```json
{"status":"ok","service":"publisher"}
```

## 2. Home Assistant の初期設定

ブラウザで次を開きます。

- `http://localhost:8123`

初回はローカルユーザを作成します。ログインできたら、まず Overview が見えることを確認してください。

実画面例:

![Home Assistant overview](../assets/screenshots/ha-dashboard-overview.png)

確認ポイント:

- 左下に `Settings` が見えていること
- 中央に `Overview` が表示されていること
- ログイン後の通常画面まで到達していること

その後、`Settings -> Devices & Services -> Integrations` で `MQTT` integration を追加してください。

設定値:

- Host: `mosquitto`
- Port: `1883`

ここでの目的は、Home Assistant の script から `mqtt.publish` を使える状態にすることです。追加後は `MQTT` の詳細画面で `mosquitto` が見えていれば、この Hands-on の前提は満たせています。

実画面例:

![Home Assistant MQTT integration](../assets/screenshots/ha-mqtt-integration.png)

確認ポイント:

- `MQTT` という integration 名が見えていること
- `mosquitto` サービスが表示されていること
- この画面が見えれば `mqtt.publish` を使う準備は完了していること

## 3. Consent VC を登録

Home Assistant demo で扱うデータセットに対応する Consent VC を登録します。

```bash
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_temperature.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_power.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_person_detected.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_flood_risk_high.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_possible_littering.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/ha_demo/consent_suspicious_activity.json
```

期待結果:

- 各レスポンスに `"status":"stored"` が含まれる

対象ファイル:

- [consent_temperature.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_temperature.json)
- [consent_power.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_power.json)
- [consent_person_detected.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_person_detected.json)
- [consent_flood_risk_high.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_flood_risk_high.json)
- [consent_possible_littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_possible_littering.json)
- [consent_suspicious_activity.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/consent_suspicious_activity.json)

## 4. Home Assistant から demo データを送る

Home Assistant の `Developer Tools -> Actions` から、次の script を実行します。画面上部の action 選択欄で `script.turn_on` を選び、target entity に対象の script を指定すると再現しやすいです。

実画面例:

![Home Assistant Developer Tools Actions](../assets/screenshots/ha-developer-tools-actions.png)

確認ポイント:

- 上部タブで `Actions` が選ばれていること
- action 選択欄で `script.turn_on` を使えること
- 右下の実行ボタンから対象 script を呼び出せること

- `script.iw3ip_publish_demo_temperature`
- `script.iw3ip_publish_demo_power`
- `script.iw3ip_publish_demo_person_detected`
- `script.iw3ip_publish_demo_flood_risk_high`
- `script.iw3ip_publish_demo_possible_littering`
- `script.iw3ip_publish_demo_suspicious_activity`
- `script.iw3ip_publish_demo_phase3_safety_scenario`

これらの script は、`scripts.yaml` に定義された `mqtt.publish` を呼び出します。

Phase の対応:

- Phase 1:
  - `temperature`
  - `power`
  - `person_detected`
- Phase 2:
  - `flood_risk_high`
  - `possible_littering`
  - `suspicious_activity`

## 5. publisher 側の結果を確認

Home Assistant 側で script を実行したら、publisher 側で結果を確認します。

```bash
curl http://localhost:8080/platform/ingest
curl http://localhost:8080/audit/logs?limit=10
```

確認ポイント:

- `platform/ingest` に送信済みデータが増える
- `audit/logs` に `allow` が残る
- `dataset_id`, `purpose`, `message_hash`, `raw_topic` が確認できる

ここで「データを送れたか」だけでなく、「どの条件で送られたか」を監査ログから読むのが重要です。

出力例:

![Publisher outputs](../assets/ha-demo-publisher-output.svg)

特に見る点:

- `/platform/ingest` に `home/event/possible_littering` と `home/event/suspicious_activity` が入っていること
- `/audit/logs` で `action: allow` と `raw_topic` が対応していること

ここまで確認できれば、Home Assistant から publisher までの基本的な共有経路は動いています。次は、同じ経路でも Consent VC によって拒否されるケースを確認します。

## Phase 2: イベント共有と拒否を確認する

## 6. 拒否ケースを確認

Home Assistant script では主に許可されるケースを流すので、拒否ケースは HTTP 疑似投入で確認します。

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/flood_risk_high",
    "payload":{
      "event_type":"flood_risk_high",
      "data":{"zone":"river-west","severity":"high"},
      "ts":"2026-03-10T10:05:00Z",
      "source":"home_assistant_demo"
    },
    "purpose":"advertising"
  }'
```

期待結果:

```json
{"status":"denied","dataset_id":"home/event/flood_risk_high","reason":"no_matching_consent"}
```

この確認により、「イベントを作れたこと」と「共有が許可されること」は別であると分かります。

流れの見方:

![Denied flow](../assets/ha-demo-denied-flow.svg)

ここで見る点:

- 入力 topic は正しくても、`purpose` が Consent VC と一致しないと拒否されること
- `simulate/publish` のレスポンスと `audit/logs` の両方で拒否を確認すること

## 7. MQTT 経路を直接試す

Home Assistant を使わず、同じ topic に直接 publish することもできます。

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/possible_littering \
  -m @examples/ha_demo/payload_possible_littering.json
```

関連ファイル:

- [payload_temperature.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_temperature.json)
- [payload_power.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_power.json)
- [payload_person_detected.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_person_detected.json)
- [payload_flood_risk_high.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_flood_risk_high.json)
- [payload_possible_littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/payload_possible_littering.json)

Phase 2 まで見ると、同じ MQTT 経路を通っても、常に共有されるわけではないことが分かります。次の Phase 3 では、共有されたイベントをさらに assistant 側へ渡し、要求理解と実行を確認します。

## Phase 3: assistant の plan と execute を確認する

## 8. Phase 3 を試す

Phase 3 では、publisher に入ったイベントを `assistant` の `observed_events` へ変換して、`plan -> execute` を確認します。

起動:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo-phase3 up --build -d
```

その後、Home Assistant で次の script を実行します。

- `script.iw3ip_publish_demo_phase3_safety_scenario`

この script は、`park-north` に対して次のイベントを連続投入します。

- `possible_littering` を 3 回
- `suspicious_activity` を 1 回

次に、まず `plan` を確認します。

```bash
python3 examples/ha_demo/run_phase3_from_ingest.py \
  --plan-only \
  --request-file examples/ha_demo/phase3_request_park_safety.json
```

この段階で確認する項目:

- `status` が `planned`
- `plan.watch_events` に `possible_littering` と `suspicious_activity` が入る
- `plan.actions` に `light_on` と `send_notification` が入る

出力例:

```json
{
  "status": "planned",
  "plan": {
    "target_area": "park-north",
    "watch_events": ["possible_littering", "suspicious_activity"],
    "actions": [
      {"action_type": "light_on", "target": "park-north-light-1"},
      {"action_type": "send_notification", "target": "park-north-manager"}
    ]
  }
}
```

その後、publisher に蓄積されたイベントを assistant へ橋渡しして `execute` を確認します。

```bash
python3 examples/ha_demo/run_phase3_from_ingest.py \
  --request-file examples/ha_demo/phase3_request_park_safety.json
```

期待結果:

- `execution.evaluation.triggered` が `true`
- `actions_executed` に `light_on` と `send_notification` が含まれる

この段階では、「要求から plan を作る段階」と、「観測済みイベントを使って execute する段階」が分かれていることを確認できます。

出力例:

![Plan and execute outputs](../assets/ha-demo-plan-execute.svg)

`execute` で特に見る点:

- `evaluation.triggered` が `true`
- `matched_counts.possible_littering = 3`
- `matched_counts.suspicious_activity = 1`
- `actions_executed` に 2 件の action が入る

## 9. Node-RED を使う場合

Node-RED を使うと、時刻条件や手動ボタンで疑似イベントを流しやすくなります。

インポート用 flow:

- [nodered_flows.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/ha-demo-simulator/examples/ha_demo/nodered_flows.json)

この構成では、Node-RED は必須ではありません。  
まずは Home Assistant demo だけで全体の流れを確認し、その後にイベント注入の見通しを良くしたい場合に追加するのが適切です。

## 10. 何が Phase 1 で、何が Phase 2 / 3 なのか

このサンプルは、1 つの環境で Phase 1 と Phase 2 の両方を確認できる点が重要です。

- Phase 1:
  - `temperature` や `power` のような状態共有
  - 基本的な受信、正規化、送信、記録
- Phase 2:
  - `flood_risk_high` や `possible_littering` のようなイベント共有
  - `purpose` と Consent VC に基づく条件付き共有
- Phase 3:
  - publisher にたまったイベントを assistant に渡して `execute` する
  - `triggered` と `actions_executed` を確認する

そのため、このページは次の 2 つの Hands-on をつなぐ橋渡しにもなります。

- [HA x SSI Publisherサンプル（Phase 1）](ha-ssi-publisher.md)
- [環境・防災イベント共有サンプル（Phase 2）](environment-disaster.md)
- [地域安全アシスタントサンプル（Phase 3）](regional-safety-assistant.md)

## 11. 停止

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo down
```

Node-RED も含めて止める場合:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo --profile nodered down
```

Phase 3 を含めて止める場合:

```bash
docker compose -f infra/docker-compose.yml --profile ha-demo-phase3 down
```
