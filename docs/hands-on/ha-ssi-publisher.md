# HA x SSI Publisherサンプル（Phase 1）

## 目的

Home Assistant のデータを MQTT 経由で受け取り、Consent VC（同意VC）で許可判定し、
許可データのみを送信・監査ログ保存する最小構成を体験します。

パイプライン:

`Home Assistant -> MQTT -> (任意 Node-RED) -> Data Publisher -> Platform API`

## 最短ルート

最初は次の 4 手順で十分です。

1. `docker compose -f infra/docker-compose.yml up --build -d`
2. `consent_temperature.json` を登録する
3. `/simulate/publish` で `temperature` を送る
4. `/audit/logs` を見て `allow` を確認する

その後の分岐:

- HTTP 疑似投入だけ確認したい場合: `allowed` と `denied` を 1 回ずつ見る
- MQTT 経路まで見たい場合: `mosquitto_pub` で `person_detected` を送る
- Phase 2 へ進みたい場合: `flood_risk_high` や `possible_littering` の Hands-on へ移る

## このページで分かること

- Phase 1 の最小構成として、どこでデータを受け取り、どこで判定し、どこに記録するか
- `topic`、`payload`、`purpose` を 1 つの要求として組み立てる方法
- `allowed` / `denied` / 監査ログの基本的な読み方

## つまずきやすい点

- `topic` と `dataset_id` の関係が最初は見えにくい
- Consent VC を登録していても `purpose` が合わなければ拒否される
- HTTP 疑似投入と MQTT 経路が別の確認ポイントを持つ

## 前提

- Docker / Docker Compose が使える
- `curl` が使える

対応するサンプルファイル:

- `examples/consent_temperature.json`
- `examples/consent_power.json`
- `examples/consent_person_detected.json`
- `examples/consent_flood_risk_high.json`
- `examples/consent_possible_littering.json`
- `examples/payload_temperature.json`
- `examples/payload_power.json`
- `examples/payload_person_detected.json`
- `examples/payload_flood_risk_high.json`
- `examples/payload_possible_littering.json`

演習用プログラム:

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase1_ha_ssi_publisher/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase1_ha_ssi_publisher/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase1_ha_ssi_publisher/README.md)

この演習では、`/simulate/publish` に送る最小 JSON を自分で組み立てます。  
問題用プログラムでは、`topic`・`payload`・`purpose` をどのようにまとめるかを確認できます。

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>準備: publisher を起動して Consent VC を登録する</summary>
  <p>まずは publisher 自体が起動していることを確認し、最低限の Consent VC を登録します。ここまでは、後続の許可・拒否判定の前提作業です。</p>
  <ol>
    <li><a href="#1-起動">起動</a></li>
    <li><a href="#2-consent-vc-を登録">Consent VC を登録</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 1: HTTP 疑似投入で allowed と denied を比較する</summary>
  <p>次に、同じ API 入口を使って、許可される場合と拒否される場合の違いを比較します。ここで `purpose` と `dataset_id` の関係を押さえるのが重要です。</p>
  <ol>
    <li><a href="#3-許可されるケース">許可されるケース</a></li>
    <li><a href="#4-拒否されるケース">拒否されるケース</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: MQTT 経路と監査ログを確認する</summary>
  <p>最後に、HTTP 疑似投入ではなく MQTT から publisher へ入る経路を試し、監査ログに何が残るかを確認します。</p>
  <ol>
    <li><a href="#5-mqtt経路を試す">MQTT経路を試す</a></li>
    <li><a href="#6-監査ログ確認">監査ログ確認</a></li>
    <li><a href="#7-停止">停止</a></li>
  </ol>
</details>

## 読み進め方

Phase 1 の基本構成を確認するページです。短時間なら `最短ルート` だけで十分。共有制御の考え方まで理解したい場合は `allowed` と `denied` を両方見てから MQTT 経路へ進んでください。

## Phase 1: 最小構成を確認する

## 1. 起動

```bash
docker compose -f infra/docker-compose.yml up --build -d
```

確認:

```bash
curl http://localhost:8080/health
```

期待結果:

```json
{"status":"ok","service":"publisher"}
```

## 2. Consent VC を登録

```bash
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_temperature.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_power.json
curl -X POST http://localhost:8080/consents -H 'Content-Type: application/json' -d @examples/consent_person_detected.json
```

Phase 2 の Hands-on と対応する追加ファイル:

- `examples/consent_flood_risk_high.json`
- `examples/consent_possible_littering.json`

判定に必要な準備は完了です。次に、許可される要求と拒否される要求を比較します。

## Phase 1: allowed と denied を比較する

## 3. 許可されるケース

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/state/sensor/temperature",
    "payload":{
      "entity_id":"sensor.living_room_temperature",
      "state":"24.1",
      "attributes":{"unit_of_measurement":"C"},
      "ts":"2026-02-28T10:00:00Z",
      "source":"home_assistant"
    },
    "purpose":"research"
  }'
```

期待結果:

```json
{"status":"allowed","dataset_id":"home/env/temperature"}
```

## 4. 拒否されるケース

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/state/sensor/power",
    "payload":{
      "entity_id":"sensor.main_power",
      "state":"520",
      "attributes":{"unit_of_measurement":"W"},
      "ts":"2026-02-28T10:00:10Z",
      "source":"home_assistant"
    },
    "purpose":"marketing"
  }'
```

期待結果:

```json
{"status":"denied","dataset_id":"home/energy/power","reason":"no_matching_consent"}
```

入力形式が正しくても `purpose` が一致しなければ拒否される、というのがポイントです。同じ判定を MQTT 経路でも確認します。

## Phase 1: MQTT 経路と監査ログを確認する

## 5. MQTT経路を試す

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/person_detected \
  -m '{"event_type":"person_detected","data":{"camera_id":"front_door","confidence":0.93},"ts":"2026-02-28T10:00:20Z","source":"edge_inference"}'
```

## 6. 監査ログ確認

```bash
curl http://localhost:8080/audit/logs?limit=10
```

確認ポイント:

- `allow` / `deny` / `send_error` が記録される
- `dataset_id`, `purpose`, `message_hash` が残る

## 7. 停止

```bash
docker compose -f infra/docker-compose.yml down
```

## 拡張ヒント

- Phase 2: `homeassistant/event/...` 系を中心にイベント共有へ拡張
- Phase 3: 前段にSSI Gateway（PEP）を追加し、VC提示制御を実装

Phase 2 をすぐ試したい場合は、次の Hands-on を参照してください。

- [環境・防災イベント共有サンプル（Phase 2）](environment-disaster.md)
- [USBウェブカメライベント共有サンプル（Phase 2）](webcam-event-sharing.md)
