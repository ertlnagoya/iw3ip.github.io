# SSI サービスサンプル（Phase 2 / Stage 4 prep）

!!! note "wallet ハンズオンの M2M 拡張です"
    Stage 1 (ConsentVC + PolicyToken) と Stage 3 (ViewerVC + ViewerToken)
    を体験済の前提で進めます。本ページは「**人ではないサービス**が
    VC を持って継続的に書き込む」M2M ケースを扱います。

!!! tip "dataset の選択"
    例は `home/env/temperature` ですが、Stage 0 と同じ
    `home/event/possible_littering` でも同じ手順で通ります
    ([service-possible-littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/ssi_wallet/service-possible-littering.json)
    が登録済)。

## 目的

ConsentVC の PolicyToken は **5 分・単回利用** で、人間が 1 件ずつ
同意するワークフローに最適化されています。一方で **MQTT publisher の
ように毎秒イベントを ingest する主体**には、毎回提示が要求される
PolicyToken は使い物になりません。

ServiceVC は「**サービスが継続的に書き込んでよい**」ことを表す M2M VC
で、提示すると 1 時間有効・多回利用可能な ServiceToken が払い出されます。
publisher プロセスがこの ServiceToken を保持し、各 MQTT メッセージに
Bearer ヘッダで添えて `/platform/ingest` を叩く想定です。

```
[サービス] -- ServiceVC 提示 (1 回) ----> [Verifier]
                                              ↓
                                         ServiceToken (1h, 多回)
[サービス] <- ServiceToken --------------------┘
[サービス] -- POST /platform/ingest x N -----> [Publisher]
                                                  認可は ServiceToken で
```

## このページで分かること

- 「人の VC (ConsentVC)」と「サービスの VC (ServiceVC)」を分ける理由
- 単回 (write) / 多回 read (read) / 多回 write (continuous) の
  トークンセマンティクスの 3 つの形
- ServiceToken の TTL 1 時間・write_count 増加の振る舞い
- `/platform/ingest` が PolicyToken と ServiceToken の両方を受け付ける
  仕組み (順序: PolicyToken → 不一致なら ServiceToken)

## つまずきやすい点

- ConsentVC / ViewerVC / ServiceVC は **別 VC**。流用できない
- ServiceToken は ViewerToken と TTL も用途も違う。`/platform/ingest`
  だけで使える (read 側 `/platform/data` は ViewerToken のみ)
- ServiceVC は M2M 想定。スマホウォレットで持つことも可能だが、
  本来は publisher に組み込まれた holder が保持する

## 前提

- [SSI Wallet ハンズオン](ha-ssi-wallet.md) (Stage 1) を通している
- publisher が `feat/ssi-service-vc` 以降のコードで起動している
- スマホ wallet で ServiceVC を保持する経路を試す場合、
  iw3ip-wallet が利用可能

## 1. 起動とメタデータ確認

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet up --build -d
PUB=$(docker ps -qf name=publisher)

# ServiceVC が含まれていることを確認 (現在のリリースでは ConsentVC /
# ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC の 5 種が並ぶ)
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer | python3 -m json.tool | grep -A2 ServiceVC
```

期待: `"vct": "https://iw3ip.example/credentials/ServiceVC/v1"` が見える。

## 2. ServiceVC を発行 (人手の代用 = 実機 wallet)

実運用では publisher 内蔵 holder に発行しますが、ハンズオンとしては
スマホ wallet を使うのが簡単です。PC ブラウザで:

```
http://192.168.68.53:8080/issuer/offer?type=ServiceVC&dataset_id=home/env/temperature&purpose=write_continuous
```

→ AirDrop deeplink → wallet で受領。

claim:
- `dataset_id`: `home/env/temperature`
- `allowed_actions`: `["write_continuous"]`

## 3. ServiceVC を提示

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=ServiceVC
```

`vc_kind=ServiceVC` が必須 (これが無いと ConsentVC 用 PD が選ばれる)。

提示直後、publisher ログに:

```
service_token_issued jti=... token=... dataset=home/env/temperature ttl=3600s
```

`token=` の値が ServiceToken。

## 4. ServiceToken で連続 ingest

```bash
SERVICE=<service_token>

for i in 1 2 3 4 5; do
  curl -s -X POST http://192.168.68.53:8080/platform/ingest \
    -H "Authorization: Bearer $SERVICE" \
    -H "Content-Type: application/json" \
    -d "{\"dataset_id\":\"home/env/temperature\",\"value\":$((20 + i))}" \
    | python3 -c "import json,sys; print(json.load(sys.stdin))"
done
```

期待: 5 件すべて `{"status":"received","count":N}` で 200。

PolicyToken のように 2 回目で `403 already_consumed` にはなりません。

## 5. write_count を audit log で確認

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool | head -60
```

各 ingest が次の形で記録されます:

```json
{
  "action": "allow",
  "raw_topic": "platform/ingest",
  "reason": "service_token_used:<jti>:1",
  "dataset_id": "home/env/temperature",
  "purpose": "write_continuous",
  "holder_did": "did:jwk:...",
  "presentation_verified": "allow"
}
```

`reason` 末尾の数字は `write_count`。5 回 ingest すれば 1〜5 まで並びます。

## 6. エラーケース

| 状況 | HTTP | `detail` |
| --- | --- | --- |
| 期限切れ (1 時間経過) | 401 | `service_token_expired` |
| dataset 不一致 | 403 | `service_token_dataset_mismatch` |
| 未知のトークン | 401 | `policy_token_unknown` (Stage 1 互換のため) |

未知トークンが `service_*` ではなく `policy_*` を返すのは意図的で、
Stage 1 ハンズオン (PolicyToken のみ知る参加者) のエラーメッセージ
互換を保っています。

## 三つの VC とトークンの対比

| | ConsentVC (Stage 1) | ViewerVC (Stage 3) | ServiceVC (Stage 4 prep) |
| --- | --- | --- | --- |
| 用途 | 書き込み認可 | 読み出し認可 | M2M 連続書き込み |
| 想定保持者 | 人 (人間ユーザ) | 人 (閲覧者) | サービス (publisher 等) |
| トークン | PolicyToken | ViewerToken | ServiceToken |
| TTL | 5 分 | 60 秒 | 1 時間 |
| 利用回数 | 単回 | TTL 内多回 | TTL 内多回 |
| 対象 API | `POST /platform/ingest` | `GET /platform/data` | `POST /platform/ingest` |
| VC claim | `allowed_purposes` | `allowed_actions=[read]` | `allowed_actions=[write_continuous]` |

## このハンズオンの限界 (将来課題)

- **publisher 内蔵 holder は未実装**: 本来は publisher が起動時に
  自動的に ServiceVC を取得・提示し、ServiceToken を内部保持して
  各 ingest に添える流れが理想です。現状は人手 (実機 wallet + curl) で
  代用しています
- **ServiceToken の自動更新ループ**: TTL 切れの 5 分前に再提示する
  バックグラウンド処理は未実装
- **ServiceVC の発行ガバナンス**: 誰が ServiceVC を発行するかのポリシー
  (今は publisher 自身が単独で発行) は将来検討

これらが実装されれば Stage 4 LLM Planner の前段として、
plan 各ステップの認可を VC で証明する道が開けます。

## 関連

- [SSIウォレットサンプル (Stage 1)](ha-ssi-wallet.md): 人間の単回書き込み認可
- [SSIビューワサンプル (Stage 3)](ha-ssi-viewer.md): 人間の多回読み出し認可
- [既存 Phase 2 ハンズオンの補論](webcam-event-sharing.md#補論-ウォレットモードによる認可):
  ServiceVC 未実装時の 1 件単発 wallet モード
