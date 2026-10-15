# Home Assistant のデータを取り込む (HA SSI Publisher)

Home Assistant の状態を MQTT で受け、Consent VC で共有可否を判定する最小構成を組みます。

> **やること**: MQTT → publisher → Consent VC 判定 → 監査ログ までを一通り動かす
>
> **前提**: [最短起動](../setup/quickstart.md)、できれば [実機なしで一連を動かす (HA Demo Simulator)](ha-demo-simulator.md)
>
> **使うもの**: PC (Docker)
>
> **所要時間**: 約 45 分

## 目的

Home Assistant のデータを MQTT 経由で受け取り、Consent VC（同意VC）で許可判定し、
許可データのみを送信・監査ログ保存する最小構成を扱います。

パイプライン:

`Home Assistant -> MQTT -> (任意 Node-RED) -> Data Publisher -> Platform API`

## 最短ルート

最初は次の 4 手順で十分です。

1. `PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml up --build -d`
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

## よくある問題

- `topic` と `dataset_id` の関係が最初は把握しにくい
- Consent VC を登録していても `purpose` が合わなければ拒否される
- HTTP 疑似投入と MQTT 経路が別の確認ポイントを持つ

## 前提

- Docker / Docker Compose が使える
- `curl` が使える
- [最短起動](../setup/quickstart.md) で起動した IPFS が動いたままの場合は、教材リポジトリで先に `cd ~/program/Blockchain_IoT_Marketplace/ipfs && docker compose down` を実行して止めておく (IPFS と publisher がどちらもポート 8080 を使うため)

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

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase1_ha_ssi_publisher/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase1_ha_ssi_publisher/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/phase1_ha_ssi_publisher/README.md)

この演習では、`/simulate/publish` に送る最小 JSON を自分で組み立てます。  
問題用プログラムでは、`topic`・`payload`・`purpose` をどのようにまとめるかを確認できます。

## Phase 1: 最小構成を確認する

### 1. 起動

`PLATFORM_INGEST_READ_ENABLED=true` は、手順 6 で届いたデータを確かめる `GET /platform/ingest` を有効にする指定です (認証なしで全件を返すデモ用の機能のため、既定では無効です)。

```bash
cd ~/program/Blockchain_IoT_Marketplace
PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml up --build -d
```

確認:

```bash
curl http://localhost:8080/health
```

期待結果:

```json
{"status":"ok","service":"publisher"}
```

### 2. Consent VC を登録

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

### 3. 許可されるケース

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

### 4. 拒否されるケース

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

入力形式が正しくても、`purpose` が一致しなければ拒否されます。同じ判定を MQTT 経路でも確認します。

## Phase 1: MQTT 経路と監査ログを確認する

### 5. MQTT経路を試す

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/person_detected \
  -m '{"event_type":"person_detected","data":{"camera_id":"front_door","confidence":0.93},"ts":"2026-02-28T10:00:20Z","source":"edge_inference"}'
```

### 6. 監査ログ確認

```bash
curl 'http://localhost:8080/audit/logs?limit=10'
```

確認ポイント:

- `allow` / `deny` / `send_error` が記録される
- `dataset_id`, `purpose`, `message_hash` が残る

許可されたデータが送信先 (Platform API) に届いたかも確認します。

```bash
curl http://localhost:8080/platform/ingest
```

確認ポイント:

- 手順 3 で送った `"dataset_id":"home/env/temperature"` のデータが含まれる (拒否された手順 4 の `home/energy/power` は含まれない)
- `{"detail":"platform_ingest_read_disabled"}` が返る場合は、手順 1 で `PLATFORM_INGEST_READ_ENABLED=true` を付け忘れています。付けて起動し直してください ([トラブルシュート](../operations/troubleshooting.md) の項目 9)

### 7. 停止

```bash
docker compose -f infra/docker-compose.yml down
```

## 拡張ヒント

- Phase 2: `homeassistant/event/...` 系を中心にイベント共有へ拡張
- Phase 3: 前段に SSI Gateway（PEP: Policy Enforcement Point）を追加し、VC 提示による制御を実装

Phase 2 をすぐ試したい場合は、次の Hands-on を参照してください。

- [防災イベントを共有する](environment-disaster.md) (Phase 2)
- [検知イベントだけを共有する](webcam-event-sharing.md) (Phase 2)
