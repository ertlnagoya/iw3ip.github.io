# SSI ビューワサンプル（Phase 2 / Stage 3）

!!! note "[SSI Wallet ハンズオン](ha-ssi-wallet.md) の続編です"
    Stage 1 (PolicyToken による書き込み認可) を体験済の前提で進めます。
    本ページでは **読み出し** を VC ゲートする Stage 3 を扱います。

## 目的

ConsentVC が「書き込み (ingest) を許可する VC」だったのに対し、
**ViewerVC** は「読み出し (data 取得) を許可する VC」です。
ウォレットで ViewerVC を提示 → 短命 ViewerToken を受領 →
`/platform/data` API でセンサーデータを取得する流れを体験します。

パイプライン:

`スマホウォレット (ViewerVC保持) -> OID4VP Verifier -> ViewerToken -> /platform/data`

## このページで分かること

- 「書き込み VC」と「読み出し VC」を分ける理由
- ViewerVC を OID4VCI で発行し、OID4VP で提示する
- ViewerToken の TTL 60 秒 / 多回利用の扱い
- `/platform/data` で取得した行が監査ログに残る様子

## つまずきやすい点

- ConsentVC と ViewerVC は **別 VC**。ConsentVC を提示しても `/platform/data` には入れない
- `/platform/data` は ViewerToken のみ受け付ける（PolicyToken は通らない）
- ViewerToken は TTL 60 秒。連続閲覧で期限が切れたら再提示が必要

## 公式リンク

- OpenID for Verifiable Presentations 仕様: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- DCQL: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html#name-digital-credentials-query-l>
- SD-JWT VC: <https://www.ietf.org/archive/id/draft-ietf-oauth-sd-jwt-vc-latest.html>

## 前提

- [SSI Wallet ハンズオン](ha-ssi-wallet.md) の §1〜§8 を一度通している
- publisher が `feat/ssi-viewer-vc` 以降のコードで起動している
- スマホで `iw3ip-wallet` が使える

## 関連リポジトリ

- [Blockchain_IoT_Marketplace](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace) — publisher (OID4VCI/OID4VP)
- [iw3ip-wallet](https://github.com/ertlnagoya/iw3ip-wallet) — モバイル SSI ウォレット

## 最短ルート

1. publisher を起動
2. ConsentVC でデータを書き込む（Stage 1 のおさらい）
3. ViewerVC を発行 → ウォレットに保存
4. ViewerVC を提示 → ViewerToken 取得
5. `/platform/data` で取得 → 監査ログ確認

## 1. 起動

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet up --build -d
curl http://192.168.68.53:8080/health
```

`/.well-known/openid-credential-issuer` に **`ConsentVC` と `ViewerVC` の両方** が登録されていることを確認します。

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer | python3 -m json.tool | grep -A2 ViewerVC
```

## 2. ConsentVC でデータを書き込む（おさらい）

[SSI Wallet ハンズオン §5〜§8](ha-ssi-wallet.md#5-verifier-qrで提示) と同じ手順で
ConsentVC を提示 → PolicyToken 取得 → `/platform/ingest` でデータを書き込みます。
書き込んだデータが §5 で読み出しの対象になります。

## 3. ViewerVC を発行

PC ブラウザで:

```txt
http://192.168.68.53:8080/issuer/offer?type=ViewerVC&dataset_id=home/env/temperature
```

ConsentVC のときと同じフローですが、**`type=ViewerVC`** がポイントです。
QR / AirDrop deeplink でスマホウォレットに送信し、ViewerVC を保存します。

ウォレット内で別カードとして表示されることを確認してください。
Claim:

- `dataset_id`: `home/env/temperature`
- `allowed_actions`: `["read"]`

## 4. ViewerVC を提示

PC ブラウザで:

```txt
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=ViewerVC
```

`vc_kind=ViewerVC` の query が ConsentVC 用 PD ではなく **viewer-temperature.json** を選ばせます。

QR / deeplink でウォレットへ → ViewerVC を選んで提示。

期待結果:

```json
{
  "status": "allowed",
  "dataset_id": "home/env/temperature",
  "vc_kind": "ViewerVC",
  "viewer_token": "Xa9Hk...",
  "viewer_token_jti": "1f2c...",
  "expires_in": 60
}
```

## 5. /platform/data でデータを取得

```bash
TOKEN=<viewer_token>

curl -s -H "Authorization: Bearer $TOKEN" \
  'http://192.168.68.53:8080/platform/data?dataset_id=home/env/temperature' | python3 -m json.tool
```

期待結果:

```json
{
  "dataset_id": "home/env/temperature",
  "count": 3,
  "read_count": 1,
  "rows": [
    {"dataset_id": "home/env/temperature", "value": 21.4, "purpose": "research"},
    {"dataset_id": "home/env/temperature", "value": 22.0, "purpose": "research"}
  ]
}
```

`read_count` は同じトークンで何回 read したかを示します。
TTL (60 秒) 内であれば連続呼び出し可能で、各回 `read_count` がインクリメントされます。

## 6. エラーケース

| 状況 | HTTP | `detail` |
| --- | --- | --- |
| ヘッダ無し | 401 | `missing_authorization_header` |
| 未知のトークン | 401 | `viewer_token_unknown` |
| 期限切れ (60 秒経過) | 401 | `viewer_token_expired` |
| query の `dataset_id` がトークンと不一致 | 403 | `viewer_token_dataset_mismatch` |
| ConsentVC の PolicyToken を使った | 401 | `viewer_token_unknown` (別空間の token なので) |

期限切れの確認:

```bash
sleep 65
curl -i -H "Authorization: Bearer $TOKEN" \
  'http://192.168.68.53:8080/platform/data?dataset_id=home/env/temperature'
# → 401 viewer_token_expired
```

## 7. 監査ログ

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool
```

ViewerVC 経由の read は次のように記録されます:

```json
{
  "action": "allow",
  "raw_topic": "platform/data",
  "reason": "viewer_token_used:1f2c...:1",
  "dataset_id": "home/env/temperature",
  "purpose": "read",
  "holder_did": "did:jwk:...",
  "presentation_verified": "allow"
}
```

`reason` 末尾の数字は `read_count`。同じ token で 3 回 read すれば 3 つ並びます。

## ConsentVC との対称性

| 項目 | ConsentVC (Stage 1) | ViewerVC (Stage 3) |
| --- | --- | --- |
| 用途 | 書き込み認可 | 読み出し認可 |
| 提示先 | `/verifier/request?vc_kind=ConsentVC` | `/verifier/request?vc_kind=ViewerVC` |
| 取得トークン | PolicyToken | ViewerToken |
| TTL | 5 分 | 60 秒 |
| 利用回数 | 1 回 (single-use) | TTL 内なら N 回 |
| 対象 API | `POST /platform/ingest` | `GET /platform/data` |
| audit `raw_topic` | `platform/ingest` | `platform/data` |
| VC claim | `allowed_purposes` | `allowed_actions` |

## 拡張ヒント

- **ViewerVC の有効期限**: VC 自体に短い `exp` を入れて期限管理 (revoke 機能と組み合わせる)
- **dataset 横断 ViewerVC**: `allowed_actions: ["read"]` に加えて `dataset_ids: [...]` を持つ「複数 dataset 対応 VC」を発行する
- **Phase 3 連携**: [LLM Planner](llm-planner.md) が plan 実行時に必要な dataset の ViewerToken を自動取得するフローへ拡張

## トラブル時

- 症状: `no_presentation_definition_for_dataset` が返る
  - 確認: `examples/ssi_wallet/viewer-<dataset>.json` が存在するか
  - 確認: `iw3ip_vc_kind: "ViewerVC"` が PD に書かれているか
- 症状: ウォレットが「該当 VC なし」と表示
  - 確認: ConsentVC ではなく ViewerVC を発行・保存しているか
  - 確認: `vc_kind=ViewerVC` を `/verifier/request` に渡しているか
- 症状: `/platform/data` が 401 viewer_token_unknown
  - 確認: PolicyToken (write 用) を間違えて使っていないか
  - 確認: TTL (60 秒) が切れていないか
