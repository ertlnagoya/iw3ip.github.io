# HA x SSI Publisherサンプル（Phase 1）

## 目的

Home Assistant のデータを MQTT 経由で受け取り、Consent VC（同意VC）で許可判定し、
許可データのみを送信・監査ログ保存する最小構成を体験します。

パイプライン:

`Home Assistant -> MQTT -> (任意 Node-RED) -> Data Publisher -> Platform API`

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
