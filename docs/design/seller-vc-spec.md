# SellerVC — マーケット出品ガバナンス VC (Stage 7 設計仕様)

!!! abstract "このドキュメントの位置付け"
    iot-market への Merchandise 登録に身元確認を追加する **Stage 7**
    の設計仕様である。ConsentVC / ViewerVC / ServiceVC /
    PurchaseViewerVC に続く **5 種類目の VC** (Verifiable Credential) を導入し、
    誰がどの dataset を売ってよいかを VC で示す。実装前に書いた草稿である。
    本文中の Stage は Phase 2 のハンズオンの段階番号を指す (一覧は [VC アーキテクチャ全体像](vc-architecture-overview.md) の §5)。
    Stage 7 はハンズオン [marketplace-seller-vc](../hands-on/marketplace-seller-vc.md)、Stage 6 は [marketplace-vc-end-to-end](../hands-on/marketplace-vc-end-to-end.md)、Stage 5 は [marketplace-vc-bridge](../hands-on/marketplace-vc-bridge.md) に対応する。
    Stage 8 以降はまだハンズオンが無く、将来の検討事項を指す。

## 1. 動機

### 現状 (Stage 6 まで)

誰でも `IoTMarket.registerMerchandise(merchandiseAddr)` を呼べる。
Merchandise の owner は constructor 時の `msg.sender` で固定されるが、
その owner が正規の seller かどうかは誰も検証していない。

具体的なリスク:

- 第三者が偽データの Merchandise を IoTMarket に登録できる
- 同じ dataset を複数の seller が同時に出品すると、buyer が正しい提供元を
  判別できない
- audit log には Merchandise.owner (eth address) しか残らない

### Stage 7 でやること

Seller が `registerMerchandise()` を呼ぶ前に SellerVC を保持していることを
publisher が検証し、検証に成功したときだけ marketplace 側で「この Merchandise は
正規 seller 由来」と記録する。

実装では Merchandise.sol を変更せず、off-chain (publisher) で seller の
身元を audit log に記録する。on-chain でのガードは Stage 8 以降で検討する。

## 2. v6 / v7 の差分

ここでは、Stage 6 までの現行の構成を v6、本仕様 (Stage 7) の構成を v7 と呼ぶ。

| 観点 | v6 (現行) | v7 (本仕様) |
| --- | --- | --- |
| Merchandise 登録権限 | 誰でも可 | 誰でも可 (互換)。ただし seller の身元は別途 publisher で検証 |
| Seller 身元 | Merchandise.owner (eth) のみ | + did:jwk + SellerVC claims (`seller_id`, `licensed_datasets`。有効期限は `exp`) |
| 不正出品の検出 | 不可 | publisher の audit `marketplace/seller_registered` 行で追跡可 |
| buyer の参照 | Merchandise.owner | + `/platform/data` レスポンスに `seller_did` 同梱 |

v6 との互換性は完全に保つ。SellerVC を提示しない seller は v6 の経路で
従来通り動作する (publisher 側で seller_did = "unknown" として記録する)。

## 3. SellerVC のスキーマ

```json
{
  "vct": "https://iw3ip.example/credentials/SellerVC/v1",
  "iss": "did:jwk:<publisher-issuer>",
  "sub": "did:jwk:<seller-holder>",
  "iat": 1735000000,
  "exp": 1766536000,
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

| VC | 発行されるトークンの TTL (有効期間) | 利用回数 | 権限を表す claim |
| --- | --- | --- | --- |
| ConsentVC | 5 min Token | single (単回) | `allowed_purposes` |
| ViewerVC | 60 s Token | multi (TTL 内で多回) | `allowed_actions=[read]` |
| ServiceVC | 1 h Token | multi | `allowed_actions=[write_continuous]` |
| PurchaseViewerVC | 60 s Token | multi | `allowed_actions=[read]` + 購入の文脈 |
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

ServiceToken と同じ使い方 (多回利用で TTL が長い) である。違いは、`licensed_datasets`
を持つことと、register API でしか使えないことである。

### 4.2 新エンドポイント

#### `POST /marketplace/register`

Seller の Merchandise が IoTMarket に登録された後 (Hardhat 上の
`registerMerchandise()` の呼び出し後) に、その Merchandise の seller の身元情報を
publisher に通知する endpoint である。

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

サーバ側の検証:

1. SellerToken が有効
2. Merchandise.getAllAdditionalInfo() で `dataset_id` を読み出し
3. `dataset_id` が SellerToken の `licensed_datasets` に含まれる
4. (任意) Merchandise.getOwner() == `seller_eth_addr` を検証。publisher に `MARKETPLACE_HARDHAT_RPC` を設定したときだけ行う

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

`GET /platform/data?merchandise=<addr>` のレスポンスに、オプションで
`seller_did` を含める。

```json
{
  "dataset_id": "home/env/temperature",
  "count": 5,
  "read_count": 1,
  "seller_did": "did:jwk:...",
  "rows": [...]
}
```

`seller_did` が `"unknown"` の場合、その Merchandise は v7 経路で seller
登録されていない (v6 の経路で出品された、または未登録である)。

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

**MVP** (Minimum Viable Product、ハンズオンで動かす最小限の実装): publisher 自身が SellerVC を発行する (Stage 1〜6 と同じ方針)。
具体的には `/issuer/offer?type=SellerVC&seller_id=...&licensed_datasets=...`
のように、seller_id と licensed_datasets を query で渡す単純な API とする。

教育用ハンズオンとしては、これにより「seller がどの dataset を扱う権限を
持つか」を VC で示せる。

**本番想定 (将来)**: SellerVC は別の "marketplace authority" サービスが
発行し、publisher は検証のみ行う。これは本仕様には含めない。

## 7. iot-market-ui の Seller mode

新ルート `/seller`:

1. SellerVC 提示 deeplink を表示 (まだ持っていなければ /issuer/offer 案内)
2. SellerToken を保持
3. Merchandise deploy 用フォーム (price, dataset_id, fileType, dataSize)
4. deploy + registerMerchandise + /marketplace/register を一連で実行

MVP では `/seller` は簡単なページに留める (Hardhat console を使う
代替手順をハンズオンで案内する)。

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

### 9.1 publisher 単体 (`tests/test_ssi_seller_token.py`, `tests/test_marketplace_register.py`)

- SellerVC 発行 (issuer metadata に出る)
- 提示 → SellerToken
- `/marketplace/register` 200 で seller_did 紐付け
- `licensed_datasets` 不一致 → 403
- 期限切れ → 401
- 不明 token → 401
- `/platform/data` レスポンスに `seller_did` 同梱
- 未登録 Merchandise の `seller_did = "unknown"` 表示

### 9.2 e2e

e2e (end-to-end) テストには、ハンズオンの手順をそのまま用いる (Stage 5/6 と同様)。

## 10. マイルストーン

| ID | 内容 | 期間 |
| --- | --- | --- |
| **C1** | 設計仕様 (本書) | (このページ) |
| **C2** | publisher: SellerVC + SellerToken + state | 3〜4 日 |
| **C3** | publisher: `/marketplace/register` + `seller_did` in `/platform/data` | 2 日 |
| **C4** | tests + e2e validation | 2 日 |
| **C5** | iot-market-ui `/seller` ページ | 2〜3 日 |
| **C6** | ハンズオン `marketplace-seller-vc.md` (JA + EN) | 2〜3 日 |

合計は 2〜3 週間で、Stage 5/6 とほぼ同等である。

## 11. オープンクエスチョン

1. **`licensed_datasets` のワイルドカード**
   - `["*"]` を許可するか? 大手 seller には有用だが運用判断が必要
   - 推奨: MVP では明示リストのみ
2. **Merchandise.getOwner() == seller_eth_addr の検証**
   - Stage 7 で必須にするか / Stage 8 以降に回すか
   - 推奨: 必須 (なりすまし対策として最低限必要)
   - 実装: 任意とした。`MARKETPLACE_HARDHAT_RPC` を設定したときだけ検証し、未設定の場合は検証を省いて監査ログに `owner_verify=skipped` と記録する
3. **`/seller` UI でどこまで自動化**
   - Hardhat への deploy も UI 内で行うか / ハンズオンでは Hardhat console で代替するか
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
- [Seller VC で出品身元を裏付ける (Stage 7)](../hands-on/marketplace-seller-vc.md) (C6 で作成するハンズオン)
- [VC アーキテクチャ全体像](vc-architecture-overview.md) (Stage 番号の一覧は §5)
