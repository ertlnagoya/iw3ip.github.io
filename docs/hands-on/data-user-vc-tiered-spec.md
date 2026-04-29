# DataUserVC × 段階アクセス制御 仕様

データ受信者（buyer/recipient）が保持する **DataUserVC** の信頼属性に応じて、
Publisher が公開するカメラ視野を **イベント／画像／動画** の 3 段階に
動的に絞り込むための設計文書です。

実装手順ではなく、**どこを固定し、どこを差し替えるか** を定義する設計文書です。
ハンズオン手順は [DataUserVC × 段階アクセス](data-user-vc-tiered.md) を参照してください。

## このページで分かること

- 既存 5 種類の VC（ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC）と、
  6 種類目として追加する **DataUserVC** の関係
- Li 研の `DataUserVerifier.sol` に書かれた trustScore 算出ロジックを
  Phase 2 の Publisher にどのように取り込むか
- 「VC 検証 → trustScore → allowed_views」という単方向データフロー
- データ供給側（HA / RaspberryPi / USB Webcam）と Publisher の責務分離

## つまずきやすい点

- 「DataUserVC でカメラ映像が見える／見えない」を **オンチェーン契約**で判定すると
  オフラインで失敗する。今回の実装は **Publisher 側の Python で評価**する
- trustScore は **属性の和**ではなく、`HIGH_TRUST かつ score>=80` のときだけ `full`
  になる、という非線形条件を含む
- ViewerVC / PurchaseViewerVC が持つ `allowed_views` は **Token を発行する瞬間**に
  確定し、以後変えない（事後昇格させない）

## 目的

Phase 2 で実現済みの「VC 提示 → 検証 → token mint → /platform/data でイベント取得」
というフローに対し、**「何を取得できるか」を VC 属性で段階化**します。

- entityType（GovernmentOrganization / Police / Enterprise / ResearchOrganization）
- purpose（CrimeSearch / TrafficManagement / Research）
- legalCompliance（true / false）
- dataHandlingPolicy（ISO27001 / その他）
- misuseRecord（過去の濫用記録）

これらを Li 研スマートコントラクト `DataUserVerifier.sol` と同じ重み付けで
スコア化し、`full` / `access` / `denied` の 3 段に落とします。

## この仕様の前提

差し替え対象は **Publisher 内の VC 検証層 + token 層 + projection 層** のみ。
以下はそのまま再利用します。

- 既存 5 種類の VC（ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC）
- OID4VCI / OID4VP / DCQL の出入口
- bridge ↔ Hardhat の Purchase event 経路
- Webcam / HA / Demo simulator のデバイス側コード（変更なし）

**Li 研の `DataUserVerifier.sol` は Phase 2 の Publisher が信頼する「ロジック源」**
であり、オンチェーン呼び出しはしません（理由は後述）。

## アーキテクチャ

```
[Wallet (DataUserVC 保持)]
        | OID4VP (DCQL: DataUserVC)
        v
[Publisher /verifier/*]
        | post-PEX checks
        v
[trust_score.evaluate(entityType, purpose, legalCompliance,
                      dataHandlingPolicy, misuseRecord)]
        | -> trust_score: int, access_level: "full"/"access"/"denied",
        |    allowed_views: ["event"|"image"|"video", ...]
        v
[/marketplace/claim]  -> MarketplaceClaim.allowed_views
        v
[PurchaseViewerVC issuance]  -> VC.claims.allowed_views
        v
[/verifier/* で PurchaseViewerVC 提示]
        v
[ViewerToken.allowed_views]
        v
[/platform/data] image_cid / video_cid / video_duration_sec を選別投影
```

`DataUserVC` 単体では token を発行しません。**ViewerVC / PurchaseViewerVC を
発行するときの「クレデンシャル素材」**として使います。

## 既存 5 種 VC との関係

| VC | 役割 | Token 発行 | allowed_views を持つ |
|---|---|---|---|
| ConsentVC | データ提供同意 | PolicyToken (5min/single) | × |
| ViewerVC | 限定閲覧者 | ViewerToken (60s/multi) | ✓ |
| ServiceVC | サービス提供者 | ServiceToken (1h/multi) | × |
| PurchaseViewerVC | 購入後の閲覧 | ViewerToken (60s/multi) | ✓ |
| SellerVC | 出品者 | SellerToken (24h/multi) | × |
| **DataUserVC（新）** | **受信者属性表明** | **発行しない（素材のみ）** | **n/a** |

## trust_score 評価ロジック

`publisher/app/ssi/trust_score.py` に純関数として実装します。
`DataUserVerifier.sol` と一致させること（テストで突き合わせ）。

### 重み

```
entityType:
  GovernmentOrganization | Police       -> 35
  Enterprise                            -> 20
  ResearchOrganization                  -> 15
  その他                                -> 0

purpose:
  CrimeSearch                           -> 25
  TrafficManagement                     -> 20
  Research                              -> 15
  その他                                -> 0

legalCompliance == true                 -> +15
dataHandlingPolicy == "ISO27001"        -> +15
misuseRecord == false                   -> +10
misuseRecord == true                    -> -10
```

### しきい値 → アクセスレベル

```
score >= 80 AND entityType in {GovernmentOrganization, Police}
                                          -> "full"
score >= 60                               -> "access"
それ以外                                  -> "denied"
```

### アクセスレベル → allowed_views

```
"full"   -> ["event", "image", "video"]
"access" -> ["event", "image"]
"denied" -> []     # 閲覧不可（claim 自体を拒否）
```

## API 仕様（差分のみ）

### `POST /issuer/offer`（拡張）

```
?vc_kind=DataUserVC
&entity_type=GovernmentOrganization
&purpose=CrimeSearch
&legal_compliance=true
&data_handling_policy=ISO27001
&misuse_record=false
```

`DataUserVC` のとき、5 属性すべてが必須。欠落 → 400。

### `POST /verifier/request_object`

`vc_kind=DataUserVC` を受け付ける。DCQL の `claims` に上記 5 つを要求。

### `POST /verifier/*`（提示エンドポイント）

`DataUserVC` 提示時：
- post-PEX で 5 属性が claim に存在するか検証
- **token を発行しない**
- レスポンス: `{trust_score, access_level, allowed_views, holder_did}`

### `POST /marketplace/claim`（拡張）

```json
{
  "merchandise_id": "...",
  "buyer_did": "...",
  "data_user_attrs": {
    "entityType": "GovernmentOrganization",
    "purpose": "CrimeSearch",
    "legalCompliance": true,
    "dataHandlingPolicy": "ISO27001",
    "misuseRecord": false
  }
}
```

`data_user_attrs` 省略時は `allowed_views=["event"]` の最小権限で claim。

### `GET /platform/data`

`ViewerToken.allowed_views` を見て:

| allowed_views に含まれる | 出力フィールド |
|---|---|
| `event` | 既存 event JSON 全体 |
| `image` | `image_cid` 追加 |
| `video` | `video_cid`, `video_duration_sec` 追加 |

含まれないキーは **欠落させる**（null ではなく省く）。

## なぜオンチェーン呼び出しをしないか

`DataUserVerifier.sol` は研究プロトタイプで、以下の事情があります。

- VC フォーマットが W3C VC v1 JSON-LD（Phase 2 は SD-JWT VC）
- DID 方式が `did:key`（Phase 2 は `did:jwk`）
- gas / 確認待ちが OID4VP のフローに合わない
- ハンズオン環境（Hardhat ローカル）では永続性が壊れやすい

そこで **ロジック（重みとしきい値）だけを Python に再実装**し、
スマートコントラクト側は将来の本番遷移ポイントとして残します。

`trust_score.py` は **`DataUserVerifier.sol` のミラー**として保守し、
変更時はテストで両者が一致することを確認します。

## ssi-ui の扱い

`ssi-ui/` 配下の Next.js 試験 UI は **deprecated** とし、
今後の機能追加は Phase 2 wallet（`iw3ip-wallet`）+ Publisher 側に集約します。

## テスト方針

`tests/test_data_user_vc_tiered.py` に 13 ケース：

1. `evaluate()` の純関数テスト（4）
   - GovernmentOrganization × Research × ISO27001 → full
   - Enterprise × Research × ISO27001 → access
   - 低スコア → denied
   - score>=80 でも entity が gov/police でなければ full にならない
2. `evaluate_from_claims()` のラッパテスト（1）
3. issuer metadata に `DataUserVC` が含まれる（1）
4. `/issuer/offer?vc_kind=DataUserVC` が 5 属性を必須化（1）
5. `/marketplace/claim` の trust 昇格 / 既定 event のみ（2）
6. `PurchaseViewerVC` が claim の `allowed_views` を継承（1）
7. `/platform/data` の Tier 1/2/3 投影（3）

合計 88 = 既存 75 + 新規 13。退行なし。

## 将来拡張

- DataUserVC を **Verifiable Presentation** にして複数 VC を束ねる
- `DataUserVerifier.sol` を Polygon zkEVM などで再運用し、
  `trust_score.py` をフォールバックに変更
- `allowed_views` を時間帯ベースに拡張（夜間のみ video など）
