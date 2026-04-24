# 環境・防災イベント共有サンプル（Phase 2）

この Hands-on は、**Phase 2: Event / Intelligence Sharing** を最小構成で体験するためのものです。  
カメラ画像そのものではなく、エッジ側で生成された防災イベントを共有対象にします。

このページでは、既存の `HA x SSI Publisher` サンプルを土台にして、疑似データで次の流れを再現します。

`疑似センサ / エッジ推論 -> 防災イベント -> Consent VC 判定 -> 共有 / 拒否 -> 監査ログ`

## 最短ルート

最初は次の 4 手順で十分です。

1. publisher を起動する
2. `consent_flood_risk_high.json` を登録する
3. `purpose = disaster_response` で `flood_risk_high` を送る
4. `/platform/ingest` と `/audit/logs` を確認する

その後の分岐:

- 許可だけ確認したい場合: `allowed` のケースまでで止めてよいです
- Consent の意味まで確認したい場合: `advertising` を送って `denied` も見るべきです
- MQTT 経路まで見たい場合: 最後に `mosquitto_pub` で同じイベントを送ります

## このページで分かること

- Phase 2 で「生データ共有」から「イベント共有」へ移る意味
- `flood_risk_high` を `allowed` / `denied` に分ける条件
- `/platform/ingest` と `/audit/logs` の見方

## つまずきやすい点

- `dataset_id` と `event_type` の対応が頭の中で混ざりやすい
- Consent VC の `allowed_purposes` と送信時の `purpose` が一致しないと拒否される
- `allowed` でも `/platform/ingest`、`denied` でも `/audit/logs` を別々に確認する必要がある

## 前提

- [HA x SSI Publisherサンプル](ha-ssi-publisher.md) を起動できる
- Docker / Docker Compose が使える
- `curl` が使える

この Hands-on は、ソースコードリポジトリ `Blockchain_IoT_Marketplace` の `codex/ha-ssi-publisher-sample` ブランチを使う想定です。

対応するサンプルファイル:

- `examples/consent_flood_risk_high.json`
- `examples/payload_flood_risk_high.json`

演習用プログラム:

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_environment_disaster/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_environment_disaster/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_environment_disaster/README.md)

この演習では、`flood_risk_high` イベントを `/simulate/publish` に送るリクエストを完成させます。  
問題用プログラムでは、同じイベントでも `purpose` によって `allowed` / `denied` が変わることをコードレベルで確認できます。

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>準備: サービス起動と Consent VC 登録</summary>
  <p>最初に publisher を起動し、このイベント共有で使う Consent VC を登録します。ここまでは以後の許可・拒否判定の前提作業です。</p>
  <ol>
    <li><a href="#1-サービス起動">サービス起動</a></li>
    <li><a href="#2-この-hands-on-用の-consent-vc-を登録">この Hands-on 用の Consent VC を登録</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 1: allowed と denied を比較する</summary>
  <p>次に、同じ `flood_risk_high` イベントを使って、共有される場合と拒否される場合を比べます。ここで `purpose` が判定結果を変えることを確認します。</p>
  <ol>
    <li><a href="#3-許可されるケースを再現する">許可されるケースを再現する</a></li>
    <li><a href="#4-platform-api-に届いた内容を確認する">Platform API に届いた内容を確認する</a></li>
    <li><a href="#5-拒否されるケースを再現する">拒否されるケースを再現する</a></li>
    <li><a href="#6-監査ログを確認する">監査ログを確認する</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: MQTT 経路で同じイベント共有を試す</summary>
  <p>最後に HTTP 疑似投入ではなく MQTT からも同じイベント共有を再現し、経路が変わっても判定の考え方は同じであることを確認します。</p>
  <ol>
    <li><a href="#7-mqtt-経路でも試す">MQTT 経路でも試す</a></li>
  </ol>
</details>

## 読み進め方

Phase 2 の代表例として、イベント共有の考え方を確認するページです。短時間なら `allowed` だけでも構いませんが、主題は「同じイベントでも目的で共有可否が変わる」ことなので、`denied` まで見てから進むことを勧めます。

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

## Phase 2: イベント共有の前提を整える

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

`flood_risk_high` の共有判定に必要な準備が整いました。同じイベントを 2 つの目的で送り、結果を比較します。

## Phase 2: allowed と denied を比較する

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

Phase 2 で重要なのは「イベントを作ること」ではなく「どの条件で共有されるか説明できること」です。最後に MQTT 経路でも確認します。

## Phase 2: MQTT 経路でも同じ判定を確認する

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

重点は次のように変わっています。

- Phase 1: `state` や生データ中心
- Phase 2: `event` や推論結果中心

## さらに発展させるなら

1. `high_co2` や `abnormal_vibration` も同じ流れで追加する
2. 水位そのものは非共有とし、イベントだけ共有するポリシーを作る
3. 地図表示や通知アプリと連携する
4. 複数の共有先ごとに Consent VC を分ける
