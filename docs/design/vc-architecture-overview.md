# VC アーキテクチャ全体像 (Phase 2 / Stage 1〜7)

!!! abstract "このドキュメントの位置付け"
    Phase 2 で構築した **5 種類の VC × 4 種類のトークン × 7 ステージ
    のハンズオン**を 1 枚にまとめた俯瞰図。各仕様の詳細は個別ページを
    参照。e2e 動作は publisher 75 tests + 7 stage 全て iPhone 実機検証済。

## 1. 全体像 1 行

データの **書き込み / 読み出し / 出品身元** を、人または機械が VC を提示することで認可する。VC ごとに「誰に何を許すか」が claims に焼かれ、提示で短命トークンが得られ、API はトークンで gate される。

## 2. 5 種類の VC

| Kind | 役割 | 主要 claim | 想定 holder |
| --- | --- | --- | --- |
| **ConsentVC** | データ書き込みの**単回**同意 | `dataset_id`, `allowed_purposes[]` | 人間 (データ提供者) |
| **ViewerVC** | データ読み出し (任意 dataset) | `dataset_id`, `allowed_actions=["read"]` | 人間 (閲覧者) |
| **ServiceVC** | M2M 連続書き込み | `dataset_id`, `allowed_actions=["write_continuous"]` | サービス (publisher 等) |
| **PurchaseViewerVC** | 購入連動の読み出し | `dataset_id`, `merchandise_address`, `tx_hash`, `buyer_eth_addr` | 人間 (買い手) |
| **SellerVC** | 出品身元 + dataset ライセンス | `seller_id`, `licensed_datasets[]` | 人間 / サービス (出品者) |

VC は SD-JWT VC (`vct: https://iw3ip.example/credentials/<Kind>/v1`) で発行され、wallet が保持。提示は OID4VP (DCQL) で行う。

## 3. 4 種類のトークン

| Token | TTL | 利用回数 | ゲート対象 API | 由来 VC |
| --- | --- | --- | --- | --- |
| **PolicyToken** | 5 分 | 単回 | `POST /platform/ingest` | ConsentVC |
| **ViewerToken** | 60 秒 | TTL 内多回 | `GET /platform/data` | ViewerVC / **PurchaseViewerVC** |
| **ServiceToken** | 1 時間 | TTL 内多回 | `POST /platform/ingest` | ServiceVC |
| **SellerToken** | 24 時間 | TTL 内多回 | `POST /marketplace/register` | SellerVC |

ViewerToken は ViewerVC と PurchaseViewerVC の両方から発行されるが、**namespace は同じ** (issue 後は同等に扱う)。それ以外は token 種ごとに独立 namespace で、流用できない (例: SellerToken を ingest に使うと 401)。

## 4. 認可マトリクス (どの VC でどの API がゲートされるか)

```
                書き込み (write)             読み出し (read)
                ───────────────────────      ───────────────────────
                単回         連続              短期         購入連動
─────────────────────────────────────────────────────────────────
JSON 登録      │ /consents POST  │ —              │ —          │ —
ウォレット (人) │ ConsentVC       │ —              │ ViewerVC   │ PurchaseViewerVC
ウォレット (M2M)│ —               │ ServiceVC      │ —          │ —
─────────────────────────────────────────────────────────────────
出品ガバナンス  │ —               │ —              │ —          │ —
 (横断)        │   ↓ SellerVC が "/marketplace/register" を gate
```

**SellerVC は data API を直接 gate しない**。代わりに出品時の身元証跡を audit log に残し、buyer の `/platform/data?merchandise=` レスポンスに `seller_did` として surface される。

## 5. 7 段階のハンズオン

```
Phase 2 hands-on の依存グラフ

  Stage 0 (baseline: /consents JSON 登録)
   ├── webcam-event-sharing
   └── environment-disaster
        ↓
  Stage 1 ── ha-ssi-wallet         ConsentVC + PolicyToken (write 単回)
        ↓
  Stage 3 ── ha-ssi-viewer         ViewerVC + ViewerToken (read 多回)
        ↓
  Stage 4 prep ── ha-ssi-service   ServiceVC + ServiceToken (M2M 連続)
        ↓
  Stage 5 ── marketplace-vc-bridge PurchaseViewerVC (購入連動 read)
        ↓
  Stage 6 ── marketplace-vc-end-to-end
                                   Service (write) × PurchaseViewer (read) 連結
        ↓
  Stage 7 ── marketplace-seller-vc SellerVC (出品身元)
```

各ステージの詳細:

| Stage | ハンズオン | 主な追加 |
| --- | --- | --- |
| 0 | [webcam-event-sharing](../hands-on/webcam-event-sharing.md), [environment-disaster](../hands-on/environment-disaster.md) | `/consents` JSON 登録、MQTT → ingest |
| 1 | [ha-ssi-wallet](../hands-on/ha-ssi-wallet.md) | OID4VCI/OID4VP + ConsentVC + PolicyToken |
| 3 | [ha-ssi-viewer](../hands-on/ha-ssi-viewer.md) | ViewerVC + ViewerToken + `GET /platform/data` |
| 4 prep | [ha-ssi-service](../hands-on/ha-ssi-service.md) | ServiceVC + ServiceToken (M2M) |
| 5 | [marketplace-vc-bridge](../hands-on/marketplace-vc-bridge.md) | bridge service + PurchaseViewerVC + `merchandise=<addr>` 逆引き |
| 6 | [marketplace-vc-end-to-end](../hands-on/marketplace-vc-end-to-end.md) | 4 種 VC が同一 dataset で協調 + dataset_id を `additionalInfo` から動的取得 |
| 7 | [marketplace-seller-vc](../hands-on/marketplace-seller-vc.md) | SellerVC + `/marketplace/register` + on-chain `getOwner()` 検証 + `seller_did` 同梱 |

## 6. v1 / v2 マーケットプレイスの並走

Stage 5 以降では、既存マーケット (v1) と新方式 (v2) が **同じ Purchase イベントから両 lane が走る**ように設計されている。

| 観点 | v1 (現行) | v2 (Stage 5+) |
| --- | --- | --- |
| 識別子 | Ethereum address | + did:jwk |
| 認可 | `confirmedBuyers` mapping | + PurchaseViewerVC + ViewerToken |
| 配信 | `emitUpload(encryptURI)` 復号 | + `/platform/data?merchandise=<addr>` |
| 監査 | on-chain Upload event | + publisher audit log (eth_did_bound, seller_register, viewer_token_used 等) |
| 出品身元 | Merchandise.owner (eth) | + SellerVC の seller_did |

詳細: [Marketplace VC Bridge 設計仕様](marketplace-vc-bridge-spec.md), [SellerVC 設計仕様](seller-vc-spec.md)

## 7. 監査ログのチェーン

1 つの購入から最大 6 行の audit log が連鎖する (Stage 4-prep+5+7 すべて経由した場合):

| # | raw_topic | reason | 主体 (subject_did) |
| --- | --- | --- | --- |
| 1 | `oid4vp/response` | `ok` (SellerVC 提示) | seller did:jwk |
| 2 | `marketplace/seller_registered` | `seller_register:...:owner_verify=verified` | seller did:jwk |
| 3 | `oid4vp/response` | `ok` (ServiceVC 提示) | service did:jwk |
| 4 | `platform/ingest` × N | `service_token_used:<jti>:<n>` | service did:jwk |
| 5 | `marketplace/claim` | `claim_received:<id>:tx=...` | `eth:<buyer_addr>` |
| 6 | `marketplace/issued` | `eth_did_bound:claim=...:eth=...:tx=...` | buyer did:jwk |
| 7 | `oid4vp/response` | `ok` (PurchaseViewerVC 提示) | buyer did:jwk |
| 8 | `platform/data` | `viewer_token_used:<jti>:<n>` | buyer did:jwk |

注目すべき点:
- `marketplace/claim` (eth) と `marketplace/issued` (did:jwk) が **`claim_id` でリンク**して同一人物 ↔ 異なる鍵を結ぶ
- `marketplace/seller_registered` の `seller_did` が `/platform/data` レスポンスの `seller_did` と一致 (出品から閲覧までの追跡可能性)

## 8. Token namespace の独立性

各 token は **専用の API でしか有効でない**:

| Token | 有効 API | 流用すると |
| --- | --- | --- |
| PolicyToken | `POST /platform/ingest` | `/platform/data`: 401 / `/marketplace/register`: 401 |
| ViewerToken | `GET /platform/data` | `/platform/ingest`: 401 / `/marketplace/register`: 401 |
| ServiceToken | `POST /platform/ingest` | `/platform/data`: 401 / `/marketplace/register`: 401 |
| SellerToken | `POST /marketplace/register` | `/platform/ingest`: 401 / `/platform/data`: 401 |

`/platform/ingest` は PolicyToken と ServiceToken の **両方を受理**するが、内部では PolicyToken を先に試し、失敗時 (`unknown`) のみ ServiceToken にフォールバック (互換性とエラーメッセージ統一のため)。

## 9. 鍵と身元の関係

ハンズオン参加者は次の **3 種類の鍵**に同時に触れる:

| 鍵 | 場所 | 用途 |
| --- | --- | --- |
| **Ethereum 鍵** (Account #N) | MetaMask / Hardhat console | Merchandise 購入の支払い、コントラクト呼出 |
| **wallet の did:jwk** | iw3ip-wallet | OID4VCI/OID4VP の subject、VC の `cnf` |
| **publisher issuer 鍵** (did:jwk) | publisher コンテナ | VC 発行の署名 |

Stage 5/7 の `eth_did_bound` audit row が、**Ethereum 鍵 ↔ did:jwk** のリンクを記録する。これは off-chain (publisher の audit log) で **「ETH 支払い者」と「VC 提示者」が同一人物**を主張する仕組み (現状は bridge を信用するだけの MVP; 将来は EIP-712 署名で堅化予定 = Stage 8)。

## 10. 既知の限界 (Stage 8+ の検討事項)

以下は Phase 2 の Stage 1〜7 では **意図的にスコープ外**:

- **EIP-712 署名による身元バインド** (eth ↔ did:jwk のなりすまし対策)
- **on-chain ガード** (Merchandise 登録時に SellerToken proof を Solidity 側で要求)
- **失効 (revocation)** Status List 2021 などによる VC 失効リスト
- **Trust framework**: SellerVC を発行できる "marketplace authority" の身元検証
- **複数 publisher 間でのトークン共有** (現状 in-memory なので分散運用不可)
- **Phase 3 LLM Planner との接続** (plan 各ステップの認可を VC で証明)

## 11. 関連ページ

- 設計仕様
    - [Marketplace VC Bridge (v1/v2 spec)](marketplace-vc-bridge-spec.md) — Stage 5/6 の構造
    - [SellerVC spec (Stage 7 / case C M1)](seller-vc-spec.md) — Stage 7 の設計判断
- ハンズオン
    - [Phase 2 overview](../hands-on/index.md#phase-2-イベント共有)
    - 各 Stage は §5 の表を参照
