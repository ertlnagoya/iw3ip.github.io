# USBウェブカメライベント共有サンプル（Phase 2）

この Hands-on は、既存の [USBウェブカメラサンプル](webcam.md) を発展させた **Phase 2: Event / Intelligence Sharing** の例です。  
Phase 1 では、USBウェブカメラで `person_detected` や `possible_littering` を検知し、イベントファイルを生成して商品化まで確認しました。  
このページでは、その次の段階として、**検知イベントをデータ共有基盤へ送って、Consent VC に基づいて共有可否を制御する**流れを体験します。

パイプライン:

`USB webcam -> webcam-bridge -> event -> Publisher -> Consent VC check -> Platform API / Audit Log`

## 最短ルート

最初は次の 4 手順で十分です。

1. publisher を起動する
2. `consent_possible_littering.json` を登録する
3. `purpose = community_cleaning` で `possible_littering` を送る
4. `/platform/ingest` と `/audit/logs` を確認する

その後の分岐:

- 許可されるケースだけ確認したい場合: `allowed` のケースまでで止めてよいです
- Consent VC による制御まで理解したい場合: `advertising` を送って `denied` も確認します
- MQTT 経路まで見たい場合: 最後に `mosquitto_pub` を使います

## このページで分かること

- Phase 1 の検知イベントを Phase 2 の共有イベントとして扱う方法
- `possible_littering` を共有するときに何を残し、何を送らないか
- `allowed` / `denied` と監査ログの関係

## つまずきやすい点

- Phase 1 のイベント生成と、Phase 2 の共有判定を同じ処理だと思いやすい
- `purpose` を変えるだけで結果が変わる理由が見えにくい
- `/platform/ingest` だけ見ていると `denied` ケースを見落とす

## 前提

- [USBウェブカメラサンプル](webcam.md) の内容を理解している
- [HA x SSI Publisherサンプル](ha-ssi-publisher.md) を起動できる
- Docker / Docker Compose が使える
- `curl` が使える

対応するサンプルファイル:

- `examples/consent_possible_littering.json`
- `examples/payload_possible_littering.json`

演習用プログラム:

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_webcam_event_sharing/problem_program.py)
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_webcam_event_sharing/answer_program.py)
- [演習説明](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase2-hands-on-sample-programs/examples/hands_on/phase2_webcam_event_sharing/README.md)

この演習では、`possible_littering` イベントを `/simulate/publish` に送るコードを書きます。  
問題用プログラムでは、Phase 1 の検知イベントが Phase 2 では「条件付き共有イベント」へ変わることを確認できます。

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>準備: publisher と Consent VC を用意する</summary>
  <p>最初に publisher を起動し、このイベント共有で必要な Consent VC を登録します。ここまでは共有可否を判定するための準備段階です。</p>
  <ol>
    <li><a href="#1-サービス起動">サービス起動</a></li>
    <li><a href="#2-この-hands-on-用の-consent-vc-を登録">この Hands-on 用の Consent VC を登録</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 1: allowed と denied を比較する</summary>
  <p>次に、同じ `possible_littering` イベントを 2 つの目的で送り、許可される場合と拒否される場合を比較します。</p>
  <ol>
    <li><a href="#3-許可されるケースを再現する">許可されるケースを再現する</a></li>
    <li><a href="#4-platform-api-に届いた内容を確認する">Platform API に届いた内容を確認する</a></li>
    <li><a href="#5-拒否されるケースを再現する">拒否されるケースを再現する</a></li>
    <li><a href="#6-監査ログを確認する">監査ログを確認する</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: MQTT 経路でも同じイベント共有を試す</summary>
  <p>最後に MQTT 経路でも同じイベント共有を再現し、判定の考え方が変わらないことを確認します。</p>
  <ol>
    <li><a href="#7-mqtt-経路でも試す">MQTT 経路でも試す</a></li>
  </ol>
</details>

## 読み進め方

Phase 1 のカメラ検知から Phase 2 のイベント共有へ進むときの違いを確認するページです。短時間なら `allowed` だけでも構いませんが、この Hands-on の主題は「同じイベントでも目的次第で共有結果が変わる」ことなので、`denied` と監査ログまで見ることを勧めます。

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

`possible_littering` の共有判定に必要な準備が整いました。同じイベントを 2 つの目的で送り、`allowed` / `denied` を比較します。

## Phase 2: allowed と denied を比較する

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

Phase 2 の主題が「検知」から「条件付き共有」に移っていることが見えてきます。最後に MQTT 経路でも同じ共有を試します。

## Phase 2: MQTT 経路でも同じ判定を確認する

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

**「検知」から「条件付き共有」へ主題が移る** のがポイントです。

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

## 補論: ウォレットモードによる認可

本編では `consent_possible_littering.json` を `/consents` に POST して
ポリシーを登録しました。これは「同意の記録を平文 JSON でサーバに置く」
方式です。**同じ認可を、参加者がスマホで保持する VC を提示する形**
で扱う発展が、[スマホSSIウォレットサンプル](ha-ssi-wallet.md) です。

### 何が等価で何が違うのか

| 観点 | 本編 (`/consents` JSON 登録) | Wallet モード (Stage 1: ConsentVC) |
| --- | --- | --- |
| 認可の保持者 | publisher サーバ | スマホウォレット |
| 認可の伝達 | JSON POST | OID4VP 提示 (QR/Deeplink) |
| 期間 | `valid_to` の TTL | 提示ごとに短命 PolicyToken (5 分・単回) |
| 撤回 | `/consents` から削除 | VC を revoke / wallet で破棄 |
| 監査 | `purpose` / `dataset_id` を記録 | `holder_did` / `vc_hash` も記録 |

**判定ロジックそのものは共通** で、`dataset_id` と `purpose` の
組み合わせで `allow` / `deny` が決まります。差は「同意者の身元が VC で
裏付けられるか」と「同意の伝達経路」です。

### 試す手順 (概要)

連続する MQTT イベントすべてを wallet 経由でゲートするのは
PolicyToken の単回消費仕様と相性が悪いため、**1 件のイベントだけ
curl で送る**体験を推奨します。

1. [スマホSSIウォレットサンプル §1〜§5](ha-ssi-wallet.md#1-起動) で
   ConsentVC を発行・提示し PolicyToken を取得
2. 本編で MQTT を回している場合は一旦止める
3. PolicyToken を Bearer に付けて 1 件 ingest:
   ```bash
   curl -X POST http://localhost:8080/platform/ingest \
     -H "Authorization: Bearer $POLICY" \
     -H "Content-Type: application/json" \
     -d '{"dataset_id":"home/env/temperature","purpose":"research","value":21.4}'
   ```
4. `/audit/logs` で `reason=policy_token_consumed:<jti>` を確認

### この補論の限界

- 現状 publisher に登録された Presentation Definition は
  `home/env/temperature` 等で、本編の `home/event/possible_littering`
  とは名前空間が異なります。可能性検証として
  `home/env/temperature` 系で進めるのが手早いです。
- MQTT 連続フローを wallet で回すには **多回利用可の M2M トークン
  (ServiceVC)** が必要で、これは将来のハンズオンで扱います。

### 関連

- 読み出し側を VC でゲートする対称的なフロー: [SSI ビューワサンプル](ha-ssi-viewer.md)
