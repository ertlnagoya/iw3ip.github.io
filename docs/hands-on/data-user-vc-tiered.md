# DataUserVC × 段階アクセス（Phase 2 拡張）

このハンズオンは、[スマホSSIウォレットサンプル](ha-ssi-wallet.md) と
[USBウェブカメライベント共有サンプル](webcam-event-sharing.md) を踏まえ、
**データ受信者の信頼属性に応じてカメラ視野を 3 段階に絞る**流れを体験します。

設計の根拠は [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) を
先に読むと理解が早いです。

パイプライン:

`Wallet -> DataUserVC 提示 -> trustScore 算出 -> /marketplace/claim -> PurchaseViewerVC -> /platform/data に image/video が出る/出ない`

## 最短ルート

1. publisher / hardhat / bridge を起動する
2. **DataUserVC** を 3 種類のプロファイルで発行する（gov-full / enterprise-access / low-deny）
3. 各プロファイルで `/marketplace/claim` を投げて `allowed_views` を比較する
4. PurchaseViewerVC を提示して `/platform/data` の出力差を見る

## このページで分かること

- DataUserVC を発行・提示する OID4VCI / OID4VP の流れ
- `entityType / purpose / legalCompliance / dataHandlingPolicy / misuseRecord`
  の組み合わせで `full / access / denied` がどう変わるか
- `/platform/data` の出力に `image_cid` / `video_cid` が現れる / 消える条件

## つまずきやすい点

- `data_user_attrs` を省くと既定で **`event` のみ**になる（image/video は出ない）
- score >= 80 でも `entityType` が `Enterprise` のままだと **`full` には届かない**
- ViewerToken 発行後に `data_user_attrs` を変えても **過去の token には反映されない**

## 前提

- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) を一度動かしている
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) を理解している
- Docker / Docker Compose が使える
- `curl`, `jq` が使える

## 1. サービス起動

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml up -d publisher hardhat bridge mosquitto
```

ヘルスチェック:

```bash
curl -s localhost:8080/healthz | jq .
curl -s localhost:8080/issuer/.well-known/openid-credential-issuer | jq '.credential_configurations_supported | keys'
# -> ["ConsentVC", "DataUserVC", "PurchaseViewerVC", "SellerVC", "ServiceVC", "ViewerVC"]
```

`DataUserVC` がリストに出ていれば OK です。

## 2. 3 種類の DataUserVC オファーを作る

### 2a. Tier 3（full）プロファイル — 政府機関 + 犯罪捜査 + ISO27001

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=GovernmentOrganization&purpose=CrimeSearch&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

### 2b. Tier 2（access）プロファイル — 企業 + 研究 + ISO27001

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

### 2c. Tier 1（denied）プロファイル — 企業 + 研究 + ポリシーなし + 濫用記録あり

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true' | jq .
```

返ってくる `credential_offer_uri` を iPhone のウォレットで読み取り、それぞれ
ウォレット内に DataUserVC として保存します。

## 3. /marketplace/claim を 3 通り投げる

`merchandise_id` は webcam-event-sharing で出品済みのものを再利用します
（手元で 1 件出品しておく）。

### 3a. Tier 3 — 動画まで開く

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{
    "merchandise_id": "M-0001",
    "buyer_did": "did:jwk:GOV_USER_DID",
    "data_user_attrs": {
      "entityType": "GovernmentOrganization",
      "purpose": "CrimeSearch",
      "legalCompliance": true,
      "dataHandlingPolicy": "ISO27001",
      "misuseRecord": false
    }
  }' | jq '.allowed_views, .access_level, .trust_score'
# -> ["event","image","video"]
#    "full"
#    90
```

### 3b. Tier 2 — 画像まで

```bash
# data_user_attrs.entityType: "Enterprise", purpose: "Research"
# allowed_views: ["event", "image"], access_level: "access", score: 60
```

### 3c. Tier 1 — `data_user_attrs` を省く既定値

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{"merchandise_id":"M-0001","buyer_did":"did:jwk:LOW_USER_DID"}' | jq .
# allowed_views: ["event"]
```

## 4. PurchaseViewerVC を発行 → 提示 → /platform/data

`/issuer/offer?vc_kind=PurchaseViewerVC&claim_id=...` で各 buyer 用の
PurchaseViewerVC を発行し、ウォレットで取得 → 提示します。

提示が通った後の ViewerToken で `/platform/data` を叩くと:

| プロファイル | `event` | `image_cid` | `video_cid` |
|---|---|---|---|
| 3a Tier 3 (gov full) | あり | あり | あり |
| 3b Tier 2 (enterprise) | あり | あり | **なし** |
| 3c Tier 1 (default) | あり | **なし** | **なし** |

```bash
curl -s -H "authorization: Bearer $VIEWER_TOKEN" \
     localhost:8080/platform/data?dataset_id=home/event/possible_littering | jq .
```

`image_cid` / `video_cid` フィールドが **消えている**ことを確認します
（`null` ではなくキーごと消える）。

## 5. 監査ログ

```bash
curl -s localhost:8080/audit/logs | jq '.[-5:]'
```

`vc_kind: "DataUserVC"` の verify 行と、続く `claim` / token mint /
`/platform/data` 取得の一連が紐づくことを確認します。

## 6. テストで突き合わせ

```bash
cd ~/program/Blockchain_IoT_Marketplace
uv run pytest tests/test_data_user_vc_tiered.py -v
# 13 passed
```

スマートコントラクト `DataUserVerifier.sol` の重みと、`trust_score.py` の
重みが一致することは、テストの 4 ケース（GovernmentOrganization / Enterprise /
低スコア / score>=80 でも entity が違うと full にならない）で確認します。

## 次に進む

- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) — Phase 2 wallet を立ち上げる
- [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) — 設計根拠
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) — 商品化の前段
