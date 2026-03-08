# 環境・防災イベント共有サンプル（Phase 2）

この Hands-on は、**Phase 2: Event / Intelligence Sharing** を最小構成で体験するためのものです。  
カメラ画像そのものではなく、エッジ側で生成された防災イベントを共有対象にします。

このページでは、既存の `HA x SSI Publisher` サンプルを土台にして、疑似データで次の流れを再現します。

`疑似センサ / エッジ推論 -> 防災イベント -> Consent VC 判定 -> 共有 / 拒否 -> 監査ログ`

## この Hands-on で学べること

- Phase 1 の「データ共有」を、Phase 2 の「イベント共有」へ拡張する考え方
- 生データではなく、イベントだけを共有する利点
- `purpose` によって `allowed` / `denied` が変わること
- `/platform/ingest` と `/audit/logs` を見て、共有結果と監査結果を分けて確認する方法

## 前提

- [HA x SSI Publisherサンプル](ha-ssi-publisher.md) を起動できる
- Docker / Docker Compose が使える
- `curl` が使える

この Hands-on は、ソースコードリポジトリ `Blockchain_IoT_Marketplace` の `codex/ha-ssi-publisher-sample` ブランチを使う想定です。

対応するサンプルファイル:

- `examples/consent_flood_risk_high.json`
- `examples/payload_flood_risk_high.json`

## シナリオ

河川近くのエッジセンサが、水位上昇や周辺情報から `flood_risk_high` というイベントを生成したとします。  
このとき、自治体の防災対応や研究目的には共有してよいが、広告目的には共有したくないとします。

ここでは、生データそのものではなく、次のようなイベントだけを共有します。

```json
{
  "event_type": "flood_risk_high",
  "data": {
    "sensor_id": "river-west-01",
    "location": "west_river_area",
    "water_level_m": 1.82,
    "severity": "high"
  },
  "ts": "2026-03-08T10:15:05Z",
  "source": "edge_inference"
}
```

このイベントは、`homeassistant/event/flood_risk_high` に送ると、Publisher 側で `dataset_id = home/event/flood_risk_high` に正規化されます。

同じ内容は、ソースコードリポジトリの `examples/payload_flood_risk_high.json` にも入っています。

## 1. サービス起動

ソースコードリポジトリで次を実行します。

```bash
docker compose -f infra/docker-compose.yml up --build -d
```

ヘルスチェック:

```bash
curl http://localhost:8080/health
```

期待結果:

```json
{"status":"ok","service":"publisher"}
```

## 2. この Hands-on 用の Consent VC を登録

この Hands-on では、`home/event/flood_risk_high` を `disaster_response` と `research` の目的でのみ許可します。

同じ内容は、ソースコードリポジトリの `examples/consent_flood_risk_high.json` にも入っています。

```bash
curl -X POST http://localhost:8080/consents \
  -H 'Content-Type: application/json' \
  -d '{
    "vc_id": "consent-flood-risk-001",
    "subject_did": "did:example:river-community",
    "dataset_id": "home/event/flood_risk_high",
    "allowed_purposes": ["disaster_response", "research"],
    "retention_days": 30,
    "reshare_allowed": false,
    "valid_from": "2026-02-01T00:00:00Z",
    "valid_to": "2026-05-01T00:00:00Z",
    "signature": "PLACEHOLDER"
  }'
```

期待結果:

```json
{"status":"stored", ...}
```

確認:

```bash
curl http://localhost:8080/consents
```

## 3. 許可されるケースを再現する

`purpose = disaster_response` として、防災イベントを疑似投入します。

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/flood_risk_high",
    "payload":{
      "event_type":"flood_risk_high",
      "data":{
        "sensor_id":"river-west-01",
        "location":"west_river_area",
        "water_level_m":1.82,
        "severity":"high"
      },
      "ts":"2026-03-08T10:15:05Z",
      "source":"edge_inference"
    },
    "purpose":"disaster_response"
  }'
```

期待結果:

```json
{"status":"allowed","dataset_id":"home/event/flood_risk_high"}
```

## 4. Platform API に届いた内容を確認する

許可された場合、Publisher 内のダミー Platform API に転送されます。

```bash
curl http://localhost:8080/platform/ingest
```

確認ポイント:

- `dataset_id` が `home/event/flood_risk_high`
- `purpose` が `disaster_response`
- `payload.message_type` が `event`
- `payload.payload.event_type` が `flood_risk_high`

## 5. 拒否されるケースを再現する

今度は同じイベントを `advertising` 目的で送ります。

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/flood_risk_high",
    "payload":{
      "event_type":"flood_risk_high",
      "data":{
        "sensor_id":"river-west-01",
        "location":"west_river_area",
        "water_level_m":1.82,
        "severity":"high"
      },
      "ts":"2026-03-08T10:16:05Z",
      "source":"edge_inference"
    },
    "purpose":"advertising"
  }'
```

期待結果:

```json
{"status":"denied","dataset_id":"home/event/flood_risk_high","reason":"no_matching_consent"}
```

## 6. 監査ログを確認する

```bash
curl http://localhost:8080/audit/logs?limit=10
```

確認ポイント:

- `allow` と `deny` の両方が記録される
- `dataset_id` が `home/event/flood_risk_high`
- `purpose` が `disaster_response` または `advertising`
- `message_hash` が記録される

## 7. MQTT 経路でも試す

HTTP 疑似投入ではなく、MQTT 経由でも同じイベント共有を再現できます。

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/flood_risk_high \
  -m '{"event_type":"flood_risk_high","data":{"sensor_id":"river-west-01","location":"west_river_area","water_level_m":1.82,"severity":"high"},"ts":"2026-03-08T10:17:05Z","source":"edge_inference"}'
```

その後、もう一度監査ログを確認します。

```bash
curl http://localhost:8080/audit/logs?limit=10
```

## 成功判定

この Hands-on では、次を確認できれば成功です。

- `home/event/flood_risk_high` がイベント共有として扱われる
- `disaster_response` では `allowed` になる
- `advertising` では `denied` になる
- 許可されたイベントは `/platform/ingest` に届く
- 許可・拒否の両方が `/audit/logs` に残る

## 何が Phase 2 なのか

Phase 1 では、温度や電力のようなデータ共有を中心に見ました。  
この Hands-on では、その次の段階として、**生データそのものではなく、エッジで解釈されたイベントを共有する**ことを体験します。

つまり、重点は次のように変わっています。

- Phase 1: `state` や生データ中心
- Phase 2: `event` や推論結果中心

## さらに発展させるなら

1. `high_co2` や `abnormal_vibration` も同じ流れで追加する
2. 水位そのものは非共有とし、イベントだけ共有するポリシーを作る
3. 地図表示や通知アプリと連携する
4. 複数の共有先ごとに Consent VC を分ける
