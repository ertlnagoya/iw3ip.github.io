# USBウェブカメライベント共有サンプル（Phase 2）

この Hands-on は、既存の [USBウェブカメラサンプル](webcam.md) を発展させた **Phase 2: Event / Intelligence Sharing** の例です。  
Phase 1 では、USBウェブカメラで `person_detected` や `possible_littering` を検知し、イベントファイルを生成して商品化まで確認しました。  
このページでは、その次の段階として、**検知イベントをデータ共有基盤へ送って、Consent VC に基づいて共有可否を制御する**流れを体験します。

パイプライン:

`USB webcam -> webcam-bridge -> event -> Publisher -> Consent VC check -> Platform API / Audit Log`

## この Hands-on で学べること

- Phase 1 の「検知して可視化する」流れを、Phase 2 の「検知イベントを共有する」流れへ拡張する考え方
- カメラ画像そのものではなく、`person_detected` や `possible_littering` のようなイベントだけを共有する利点
- 同じイベントでも、`purpose` によって `allowed` / `denied` が変わること
- 共有結果と監査結果を、`/platform/ingest` と `/audit/logs` で分けて確認する方法

## 前提

- [USBウェブカメラサンプル](webcam.md) の内容を理解している
- [HA x SSI Publisherサンプル](ha-ssi-publisher.md) を起動できる
- Docker / Docker Compose が使える
- `curl` が使える

対応するサンプルファイル:

- `examples/consent_possible_littering.json`
- `examples/payload_possible_littering.json`

## シナリオ

地域のカメラでポイ捨ての可能性が検知されたとします。  
このとき、次のような考え方を取ります。

- カメラ映像そのものは共有しない
- 代わりに `possible_littering` イベントだけを共有する
- 地域美化や研究目的なら共有してよい
- 広告目的では共有したくない

ここで共有するイベントは、たとえば次のような JSON です。

```json
{
  "event_type": "possible_littering",
  "data": {
    "camera_id": "webcam-401",
    "location": "park-north",
    "object_class": "bottle",
    "confidence": 0.87
  },
  "ts": "2026-03-08T11:00:00Z",
  "source": "edge_inference"
}
```

このイベントは、`homeassistant/event/possible_littering` に送ると、Publisher 側で `dataset_id = home/event/possible_littering` に正規化されます。

同じ内容は、ソースコードリポジトリの `examples/payload_possible_littering.json` にも入っています。

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

この Hands-on では、`home/event/possible_littering` を `community_cleaning` と `research` の目的でのみ許可します。

同じ内容は、ソースコードリポジトリの `examples/consent_possible_littering.json` にも入っています。

```bash
curl -X POST http://localhost:8080/consents \
  -H 'Content-Type: application/json' \
  -d '{
    "vc_id": "consent-littering-001",
    "subject_did": "did:example:park-community",
    "dataset_id": "home/event/possible_littering",
    "allowed_purposes": ["community_cleaning", "research"],
    "retention_days": 14,
    "reshare_allowed": false,
    "valid_from": "2026-02-01T00:00:00Z",
    "valid_to": "2026-05-01T00:00:00Z",
    "signature": "PLACEHOLDER"
  }'
```

確認:

```bash
curl http://localhost:8080/consents
```

## 3. 許可されるケースを再現する

`purpose = community_cleaning` として、ポイ捨てイベントを疑似投入します。

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/possible_littering",
    "payload":{
      "event_type":"possible_littering",
      "data":{
        "camera_id":"webcam-401",
        "location":"park-north",
        "object_class":"bottle",
        "confidence":0.87
      },
      "ts":"2026-03-08T11:00:00Z",
      "source":"edge_inference"
    },
    "purpose":"community_cleaning"
  }'
```

期待結果:

```json
{"status":"allowed","dataset_id":"home/event/possible_littering"}
```

## 4. Platform API に届いた内容を確認する

```bash
curl http://localhost:8080/platform/ingest
```

確認ポイント:

- `dataset_id` が `home/event/possible_littering`
- `purpose` が `community_cleaning`
- `payload.message_type` が `event`
- `payload.payload.event_type` が `possible_littering`

## 5. 拒否されるケースを再現する

同じイベントを `advertising` 目的で送ります。

```bash
curl -X POST http://localhost:8080/simulate/publish \
  -H 'Content-Type: application/json' \
  -d '{
    "topic":"homeassistant/event/possible_littering",
    "payload":{
      "event_type":"possible_littering",
      "data":{
        "camera_id":"webcam-401",
        "location":"park-north",
        "object_class":"bottle",
        "confidence":0.87
      },
      "ts":"2026-03-08T11:01:00Z",
      "source":"edge_inference"
    },
    "purpose":"advertising"
  }'
```

期待結果:

```json
{"status":"denied","dataset_id":"home/event/possible_littering","reason":"no_matching_consent"}
```

## 6. 監査ログを確認する

```bash
curl http://localhost:8080/audit/logs?limit=10
```

確認ポイント:

- `allow` と `deny` の両方が記録される
- `dataset_id` が `home/event/possible_littering`
- `purpose` が `community_cleaning` または `advertising`
- `message_hash` が記録される

## 7. MQTT 経路でも試す

```bash
docker exec -i iw3ip-mosquitto mosquitto_pub \
  -h localhost -p 1883 \
  -t homeassistant/event/possible_littering \
  -m '{"event_type":"possible_littering","data":{"camera_id":"webcam-401","location":"park-north","object_class":"bottle","confidence":0.87},"ts":"2026-03-08T11:02:00Z","source":"edge_inference"}'
```

その後、監査ログを再確認します。

```bash
curl http://localhost:8080/audit/logs?limit=10
```

## Phase 1 から何が発展したのか

Phase 1 の USBウェブカメラサンプルでは、主な関心は次の流れでした。

1. カメラで状況を検知する
2. イベントファイルを作る
3. 商品化の流れへつなぐ

この Phase 2 では、関心が次へ移ります。

1. エッジでイベントを生成する
2. そのイベントを共有してよいか判定する
3. 許可されたものだけを共有する
4. 共有履歴を監査ログに残す

つまり、**「検知」から「条件付き共有」へ主題が移る**のがポイントです。

## 成功判定

- `home/event/possible_littering` がイベント共有として扱われる
- `community_cleaning` では `allowed` になる
- `advertising` では `denied` になる
- 許可されたイベントが `/platform/ingest` に届く
- `/audit/logs` に `allow` / `deny` が残る

## 発展の方向

1. `person_detected` と `possible_littering` を組み合わせて共有条件を変える
2. 地域美化、研究、自治体連携などで共有先を分ける
3. 映像本体は非共有、イベントだけ共有するポリシーを明示する
4. スマホ閲覧アプリと連携して、イベントの一覧表示や通知へつなげる
