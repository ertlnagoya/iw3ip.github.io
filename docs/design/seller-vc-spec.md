# SellerVC — マーケット出品ガバナンス VC (Stage 7 prep / M1 spec)

!!! abstract "このドキュメントの位置付け"
    iot-market への Merchandise 登録に身元確認層を追加する **Stage 7
    (case C)** の設計仕様。ConsentVC / ViewerVC / ServiceVC /
    PurchaseViewerVC に続く **5 つ目の VC kind** を導入し、
    "誰がどの dataset を売ってよいか" を VC で裏付ける。**M1 ドラフト**。

## 1. 動機

### 現状 (Stage 6 まで)
誰でも `IoTMarket.registerMerchandise(merchandiseAddr)` を呼べる。
Merchandise の owner は constructor 時の `msg.sender` で固定されるが、
**その owner が「正規の seller か」は誰も検証していない**。

具体的なリスク:
- 第三者が偽データの Merchandise を IoTMarket に紛れ込ませる
- 同じ dataset を複数の seller が同時出品して、buyer が「正しい source」を
  判別できない
- audit log には Merchandise.owner (eth address) しか残らない

### Stage 7 でやること
**Seller が `registerMerchandise()` を呼ぶ前に SellerVC を保持していること**を
publisher が検証し、検証成功時のみ marketplace 側で「この Merchandise は
正規 seller 由来」と記録する。

実体としては Merchandise.sol を変更せず、**off-chain (publisher) で seller
身元を audit log に記録する**形を取る。on-chain ガードは Stage 8+ 以降。

## 2. v6 / v7 の差分

| 観点 | v6 (現行) | v7 (本仕様) |
| --- | --- | --- |
| Merchandise 登録権限 | 誰でも可 | 誰でも可 (互換) — but seller 身元は別途 publisher で検証 |
| Seller 身元 | Merchandise.owner (eth) のみ | + did:jwk + SellerVC claims (`licensed_datasets`, `valid_to`) |
| 不正出品の検出 | 不可 | publisher の audit `marketplace/seller_registered` 行で追跡可 |
| buyer の参照 | Merchandise.owner | + `/platform/data` レスポンスに `seller_did` 同梱 |

v6 互換性は完全保持。SellerVC を提示しない seller は **v6 lane**として
従来通り動く (publisher 側で seller_did = "unknown" として記録)。

## 3. SellerVC のスキーマ

```json
{
  "vct": "https://iw3ip.example/credentials/SellerVC/v1",
  "iss": "did:jwk:<publisher-issuer>",
  "sub": "did:jwk:<seller-holder>",
  "iat": 1735000000,
  "exp": 1735000000 + 86400 * 365,
  "seller_id": "ertl-nagoya-seller-001",
  "licensed_datasets": [
    "home/env/temperature",
    "home/env/humidity"
  ],
  "iw3ip_issuer": "iw3ip-publisher-issuer",
  "cnf": { "jwk": { ... } }
}
```

ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC との比較:

| VC | TTL | Use | scope claim |
| --- | --- | --- | --- |
| ConsentVC | 5 min Token | single | `allowed_purposes` |
| ViewerVC | 60 s Token | multi | `allowed_actions=[read]` |
| ServiceVC | 1 h Token | multi | `allowed_actions=[write_continuous]` |
| PurchaseViewerVC | 60 s Token | multi | `allowed_actions=[read]` + 購入 context |
| **SellerVC** | **24 h Token** | **multi** | **`licensed_datasets: [...]`** |

## 4. SellerToken と新エンドポイント

### 4.1 SellerToken

```python
@dataclass
class SellerToken:
    jti: str
    token: str
    seller_did: str
    licensed_datasets: list[str]
    issued_at: float
    expires_at: float           # 24h
    register_count: int = 0     # 出品ごとに +1
```

ServiceToken と同パターン (多回利用 + 長 TTL)。違いは `licensed_datasets`
を持つこと、register API でしか使えないこと。

### 4.2 新エンドポイント

#### `POST /marketplace/register`

Seller の Merchandise が IoTMarket に登録された後 (Hardhat 上の
`registerMerchandise()` 呼出後)、その Merchandise の身元情報を
publisher に教える hook。

Request:
```json
{
  "merchandise_address": "0x...",
  "seller_eth_addr": "0x...",
  "tx_hash": "0x..."   // registerMerchandise tx
}
```

Headers:
```
Authorization: Bearer <SellerToken>
```

Server-side checks:
1. SellerToken が有効
2. Merchandise.getAllAdditionalInfo() で `dataset_id` を読み出し
3. `dataset_id` が SellerToken の `licensed_datasets` に含まれる
4. (任意 / Stage 7+) Merchandise.getOwner() == `seller_eth_addr` を検証

成功時:
- `marketplace/seller_registered` audit 行を書く (seller_did, dataset_id, merchandise_address, tx_hash)
- 内部に `merchandise -> seller_did` インデックスを保持
- 200 + `{registered: true, seller_did, dataset_id}`

失敗時 (代表例):
| Reason | HTTP |
| --- | --- |
| `seller_token_unknown` | 401 |
| `seller_token_expired` | 401 |
| `dataset_not_licensed` | 403 |
| `merchandise_lookup_failed` | 502 |

### 4.3 既存エンドポイントへの拡張

`GET /platform/data?merchandise=<addr>` のレスポンスに **オプションで**
`seller_did` を含める:

```json
{
  "dataset_id": "home/env/temperature",
  "count": 5,
  "read_count": 1,
  "seller_did": "did:jwk:..." | "unknown",
  "rows": [...]
}
```

`seller_did` が `"unknown"` の場合、その Merchandise は v7 経路で seller
登録されていないことを意味する (v6 レガシー or 未登録)。

## 5. データフロー (v7 シーケンス)

```
[Seller]
  │  /issuer/offer?type=SellerVC&seller_id=...
  │  → wallet で受領
  │
  │  /verifier/request?vc_kind=SellerVC
  │  → wallet 提示
  │  → SellerToken (24h)
  │
  │  Hardhat console / iot-market-ui (seller mode):
  │  Merchandise を deploy
  │  IoTMarket.registerMerchandise(addr)  ← on-chain
  │
  │  POST /marketplace/register
  │  Authorization: Bearer <SellerToken>
  │  → audit: marketplace/seller_registered
  │  → publisher が merchandise -> seller_did を記録
  │
[Buyer (Stage 5/6 と同じ)]
  │  Merchandise.purchase()
  │  → bridge → /marketplace/claim
  │  → PurchaseViewerVC 受領 → 提示 → ViewerToken
  │
  │  GET /platform/data?merchandise=<addr>
  │  → 既存の dataset 解決ロジック + 新規: seller_did も同梱
```

## 6. Issuance ガバナンス

**MVP**: publisher 自身が SellerVC を発行 (Stage 1〜6 と同じ方針)。
具体的には `/issuer/offer?type=SellerVC&seller_id=...&licensed_datasets=...`
で seller_id と licensed_datasets を query で渡す素朴な API。

教育用ハンズオンとしては、これで「seller がどの dataset を扱う権限を
持つか」が VC で示せる体験ができる。

**本番想定 (将来)**: SellerVC は別の "marketplace authority" サービスが
発行し、publisher は検証のみ行う。本仕様には含めない (注のみ)。

## 7. iot-market-ui の Seller mode

新ルート `/seller`:
1. SellerVC 提示 deeplink を表示 (まだ持っていなければ /issuer/offer 案内)
2. SellerToken を保持
3. Merchandise deploy 用フォーム (price, dataset_id, fileType, dataSize)
4. deploy + registerMerchandise + /marketplace/register を一連で実行

MVP では `/seller` は **simple page** に留める (Hardhat console での
代替動線を hands-on で案内)。

## 8. audit log の追加形

新規 `raw_topic`:
- `marketplace/seller_registered` — `register_count:<jti>:N` (1 seller が
  N 個目の Merchandise を登録)

例:
```json
{
  "raw_topic": "marketplace/seller_registered",
  "subject_did": "did:jwk:...",
  "purpose": "register",
  "reason": "seller_register:claim=...:eth=0x...:tx=0x...:dataset=home/env/temperature",
  "holder_did": "did:jwk:..."
}
```

## 9. テスト戦略

### 9.1 publisher 単体 (`tests/test_marketplace_seller_vc.py`, 8〜10 件)
- SellerVC 発行 (issuer metadata に出る)
- 提示 → SellerToken
- `/marketplace/register` 200 で seller_did 紐付け
- `licensed_datasets` 不一致 → 403
- 期限切れ → 401
- 不明 token → 401
- `/platform/data` レスポンスに `seller_did` 同梱
- 未登録 Merchandise の `seller_did = "unknown"` 表示

### 9.2 e2e
ハンズオン手順がそのまま e2e テスト (Stage 5/6 同様)。

## 10. マイルストーン

| ID | 内容 | 期間 |
| --- | --- | --- |
| **C1** | 設計仕様 (本書) | (このページ) |
| **C2** | publisher: SellerVC + SellerToken + state | 3〜4 日 |
| **C3** | publisher: `/marketplace/register` + `seller_did` in `/platform/data` | 2 日 |
| **C4** | tests + e2e validation | 2 日 |
| **C5** | iot-market-ui `/seller` ページ | 2〜3 日 |
| **C6** | hands-on `marketplace-seller-vc.md` (JA + EN) | 2〜3 日 |

合計: 2〜3 週間。Stage 5/6 とほぼ同等。

## 11. オープンクエスチョン

1. **`licensed_datasets` のワイルドカード**
   - `["*"]` を許可するか? 大手 seller のため有用だが運用判断が必要
   - 推奨: MVP では明示リストのみ
2. **Merchandise.getOwner() == seller_eth_addr の検証**
   - Stage 7 で必須にするか / Stage 8 送りか
   - 推奨: 必須 (なりすまし対策の最低線)
3. **`/seller` UI でどこまで自動化**
   - Hardhat への deploy も UI 内で行う vs ハンズオンは Hardhat console fallback で良しとする
   - 推奨: 後者 (UI 工数を抑える)
4. **既存 Merchandise (Stage 6 で deploy 済) への対応**
   - 後付けで `/marketplace/register` を呼べるようにするか / 新規 deploy 必須か
   - 推奨: 後付け可能 (互換性のため)

## 12. 関連

- [Marketplace VC Bridge 設計仕様 (v1/v2)](marketplace-vc-bridge-spec.md)
- [SSI Wallet (Stage 1)](../hands-on/ha-ssi-wallet.md)
- [SSI Viewer (Stage 3)](../hands-on/ha-ssi-viewer.md)
- [SSI Service (Stage 4 prep)](../hands-on/ha-ssi-service.md)
- [Marketplace × Wallet bridge (Stage 5)](../hands-on/marketplace-vc-bridge.md)
- [Marketplace VC end-to-end (Stage 6)](../hands-on/marketplace-vc-end-to-end.md)
- 将来: `hands-on/marketplace-seller-vc.md` (C6)
