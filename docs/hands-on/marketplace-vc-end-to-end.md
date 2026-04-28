# 4 種 VC を 1 動線で繋ぐ end-to-end (v2 / Stage 6)

!!! abstract "Stage 1〜5 の集大成"
    Seller が **ServiceVC** で連続書き込みしたデータを、Buyer が
    **PurchaseViewerVC** で読み出す、というシナリオを 1 セッションで
    通します。バックエンドは Stage 1〜5 で実装済 + Stage 6 case B
    (dataset_id を Merchandise.additionalInfo から動的取得) のみ。

## 目的

- データフローに関わる 4 種類の VC (ConsentVC / ViewerVC /
  **ServiceVC** / **PurchaseViewerVC**) が **1 つの dataset を介して協調**
  することを体験 (5 種目の SellerVC は出品身元のガバナンス層なので
  本ハンズオンの主役ではないが、Stage 7 で並走可能)
- on-chain 支払い (MetaMask) と off-chain 認可 (VC) の **役割分担**を
  完全な動線で理解する
- audit log の **チェーン**を読み解く: ETH 鍵 → did:jwk → ServiceVC holder
  → PurchaseViewerVC holder の関係を追える

## このページで分かること

- ServiceVC で書き込んだデータを **同じ dataset の PurchaseViewerVC で
  正しく読み出せる**こと (Stage 4 prep × Stage 5 の連結)
- Stage 6 case B により Merchandise が **on-chain で dataset_id を保持**し、
  bridge / iot-market-ui がハードコードなしで dataset を解決すること
- 1 dataset を巡る audit log の **多層チェーン**

## 全体像

```
[Seller]                              [Buyer]
   │                                     │
   │ wallet で ServiceVC 提示             │
   │  → ServiceToken (1h)                │
   │                                     │
   │ POST /platform/ingest x N            │
   │  → app.state.ingested 蓄積          │
   │                                     │
   │                                     │ MetaMask で Merchandise.purchase()
   │                                     │  → Purchase event
   │                                     ▼
   │                                  [bridge listener]
   │                                     │ Merchandise.additionalInfo から
   │                                     │ dataset_id を解決
   │                                     │ POST /marketplace/claim
   │                                     ▼
   │                                  [publisher]
   │                                     │ PurchaseViewerVC offer
   │                                     ▼
   │                                  [iw3ip-wallet]
   │                                     │ OID4VCI 受領 → eth↔did 紐付け
   │                                     │ OID4VP 提示 → ViewerToken
   │                                     ▼
   │                                  GET /platform/data?merchandise=<addr>
   │                                     │
   │  ── Seller が書いた N 件が Buyer に届く ──
```

## 前提

- [Stage 1](ha-ssi-wallet.md) / [Stage 3](ha-ssi-viewer.md) /
  [Stage 4 prep](ha-ssi-service.md) / [Stage 5](marketplace-vc-bridge.md)
  を一通り通している
- 既存の publisher + bridge + Hardhat + iot-market-ui を稼働させたまま、
  本ハンズオンを上に積み上げる想定
- LAN IP は `192.168.68.53` で示すので、あなたの環境の IP に読み替え
- このハンズオンでは **同一 iPhone wallet が 1 人で 2 役 (seller + buyer)**
  を演じます (実運用の seller / buyer 分離は将来課題)

---

## Step E0. 前提環境の起動 (Stage 5 と共通)

```bash
# Hardhat + デプロイ (Merchandise が dataset_id を additionalInfo に持つ最新版)
cd ~/program/Blockchain_IoT_Marketplace/iot-market
git checkout main && git pull --ff-only
npx hardhat node --hostname 0.0.0.0 &   # ターミナル A 推奨
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
# IoTMarket = 0xe7f1725..., Merchandise #0..4 のアドレスをメモ

# PubKey 登録 (buyer = Account #2)
npx hardhat console --network localhost
# > const [_, __, buyer] = await ethers.getSigners();
# > const pk = await ethers.getContractAt("PubKey", "0x5FbDB23...", buyer);
# > await (await pk.registerKey("[handson]")).wait();
# > .exit

# publisher + bridge
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d

# bridge ログ確認
docker logs iw3ip-mv-bridge 2>&1 | tail -5
# → bridge: listening to 5 merchandise(s)

# iot-market-ui (別ターミナル)
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

詰まったら [Stage 5 ハンズオンのトラブルシューティング](marketplace-vc-bridge.md#トラブルシューティング) を参照。

---

## Step E1. Seller フェーズ — ServiceVC で連続書き込み

### 何を確認するか
- 1 つの ServiceVC を提示するだけで **複数件**の ingest が通る
- 書き込まれた dataset (`home/env/temperature`) は後で buyer が読む対象

### 操作

PC ブラウザ (Mac) で:

```
http://192.168.68.53:8080/issuer/offer?type=ServiceVC&dataset_id=home/env/temperature&purpose=write_continuous
```

QR を表示 → iPhone wallet で読み取り → **「IW3IP Service Credential」承認**。

そのまま提示:

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=ServiceVC
```

QR → wallet で **ServiceVC を選択**して提示。

### token 取り出し → 5 件の連続 ingest

```bash
PUB=$(docker ps -qf name=publisher)
SERVICE=$(docker logs $PUB 2>&1 | grep "service_token_issued" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')

for i in 1 2 3 4 5; do
  curl -s -X POST http://192.168.68.53:8080/platform/ingest \
    -H "Authorization: Bearer $SERVICE" \
    -H "Content-Type: application/json" \
    -d "{\"dataset_id\":\"home/env/temperature\",\"value\":$((30 + i))}" \
    | python3 -c "import json,sys;d=json.load(sys.stdin);print('iter',$i,d)"
done
```

### 期待出力
```
iter 1 {'status': 'received', 'count': N}
iter 2 {'status': 'received', 'count': N+1}
... (N+4 まで連番増加)
```

これで `home/env/temperature` に **値 31〜35** の 5 件が seller の "ふり" で蓄積されました。

---

## Step E2. Buyer フェーズ — Merchandise を購入

### 何を確認するか
- Stage 6 case B により、Merchandise の `additionalInfo` に書かれた
  `dataset_id` を bridge が **on-chain から自動取得**する
- 5 つの Merchandise は dataset が異なる (温度 / 湿度 / 洪水) — このハンズオン
  では温度の Merchandise を選ぶ

### Merchandise 一覧の dataset

| Merchandise | アドレス | dataset_id |
|---|---|---|
| #0 | `0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0` | `home/env/temperature` |
| #1 | `0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9` | `home/env/temperature` |
| #2 | `0x0165878A594ca255338adfa4d48449f69242Eb8F` | `home/env/humidity` |
| #3 | `0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6` | `home/env/temperature` |
| #4 | `0x610178dA211FEF7D417bC0e6FeD39F05609AD788` | `home/env/flood_risk_high` |

(値は決定的にデプロイされるので毎回固定)

ServiceVC で書き込んだ温度データを読みたいので、未購入の **温度 Merchandise** を選択 (例: `#3`)。

### 操作 (Hardhat console fallback で確実)

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
const [_, __, buyer] = await ethers.getSigners();
const merch = await ethers.getContractAt(
  "Merchandise",
  "0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6",   // Merchandise #3
  buyer
);
const tx = await merch.purchase({ value: await merch.getPrice() });
const r = await tx.wait();
console.log("tx hash:", r.hash, "status:", r.status);
```

→ `tx hash: 0x...`, `status: 1`

### 期待: bridge が dataset_id を on-chain から解決

```bash
docker logs iw3ip-mv-bridge 2>&1 | grep "Purchase event" | tail -1
```

→ ログに **`dataset=home/env/temperature`** が含まれる:

```
bridge: Purchase event from 0x2279B7A0... buyer=0x3C44... dataset=home/env/temperature tx=0x...
bridge: claim ok jti=...
```

dataset がハードコードでなく on-chain から来ていることを確認。

```bash
docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok" | tail -1
# → jti と deeplink を取り出す
```

---

## Step E3. Buyer フェーズ — PurchaseViewerVC 受領

### 何を確認するか
- Stage 5 と同じ受領手順だが、対象 dataset が **seller の書き込み先と一致**している

```bash
DEEPLINK=$(docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok" | tail -1 | sed -E 's/.*deeplink=//')
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$DEEPLINK")
open "https://api.qrserver.com/v1/create-qr-code/?size=400x400&data=$ENCODED"
```

iPhone wallet で QR を読み → 「IW3IP Purchase Viewer Credential」承認。

VC claims に `dataset_id: home/env/temperature` が入っていることを wallet で確認。

audit:
```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=3' | python3 -m json.tool | grep -A1 marketplace/issued
```
→ `eth_did_bound:claim=...:eth=0x3C44...:tx=0x...`

---

## Step E4. Buyer フェーズ — ViewerToken 取得 + データ取得

### 何を確認するか
- PurchaseViewerVC を提示すると ViewerToken が出る (Stage 5)
- `merchandise=<addr>` で取得すると、bridge が解決した dataset_id 経由で
  **Step E1 で seller が書いた値 31〜35 が読める**
- これが Stage 6 のクライマックス

### 操作

PC ブラウザで提示要求:

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=PurchaseViewerVC
```

QR → wallet で **PurchaseViewerVC** (NOT ServiceVC, NOT ViewerVC) を選んで提示。

```bash
TOKEN=$(docker logs $PUB 2>&1 | grep "viewer_token_issued vc_kind=PurchaseViewerVC" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')
echo "TOKEN=$TOKEN"

MERCHANDISE=0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCHANDISE" | python3 -m json.tool
```

### 期待出力

```json
{
  "dataset_id": "home/env/temperature",
  "count": 5,
  "read_count": 1,
  "rows": [
    {"dataset_id": "home/env/temperature", "value": 31},
    {"dataset_id": "home/env/temperature", "value": 32},
    {"dataset_id": "home/env/temperature", "value": 33},
    {"dataset_id": "home/env/temperature", "value": 34},
    {"dataset_id": "home/env/temperature", "value": 35}
  ]
}
```

**Step E1 で Seller が書いた 5 件が、Buyer の手に渡った** — Stage 6 完成。

---

## Step E5. audit log の連鎖

### 何を確認するか
- 1 dataset を巡って **複数の主体・操作の鎖**が記録されている

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=20' | python3 -m json.tool \
  | grep -E "raw_topic|reason|holder_did|subject_did" | head -40
```

期待される連鎖 (新しい順、抜粋):

| 段階 | raw_topic | reason | 主体 |
| --- | --- | --- | --- |
| 1. Buyer が読む | `platform/data` | `viewer_token_used:<jti>:1` | did:jwk:... (buyer) |
| 2. Buyer 提示 | `oid4vp/response` | `ok` | did:jwk:... (buyer) |
| 3. Buyer 受領 | `marketplace/issued` | `eth_did_bound:claim=...:eth=0x3C44...:tx=...` | did:jwk:... (buyer) |
| 4. bridge 連携 | `marketplace/claim` | `claim_received:<id>:tx=...` | eth:0x3C44... |
| 5. Seller 書き込み x5 | `platform/ingest` | `service_token_used:<jti>:1〜5` | did:jwk:... (seller) |
| 6. Seller 提示 | `oid4vp/response` | `ok` | did:jwk:... (seller) |

**注**: ハンズオン上は同じ wallet が seller / buyer を兼ねるので、
holder_did は同一になります。実運用では別端末・別 did:jwk になります。

### 観察ポイント

- (4) と (5) を結ぶのは **dataset_id**: 両方 `home/env/temperature`
- (4) が ETH 主体 → (3) で did:jwk と紐付き → (1) で did:jwk のまま読み出し、
  という **3 形態の身元**が連鎖
- ServiceVC の `service_token_used` が 5 連続 → **`viewer_token_used:1` で 1 回読まれて 5 件が一気に届く**

---

## 完了判定マトリクス

| 工程 | 確認項目 | 出力例 |
| --- | --- | --- |
| E0 | 環境起動 | `bridge: listening to 5 merchandise(s)` |
| E1 | ServiceVC 受領 | wallet に「IW3IP サービスクレデンシャル」 |
| E1 | ServiceToken 発行 | publisher ログ `service_token_issued ttl=3600s` |
| E1 | 5 件の連続 ingest | iter 1〜5 すべて `status: received` |
| E2 | bridge が dataset_id を on-chain 解決 | `dataset=home/env/temperature` |
| E2 | bridge claim 発行 | `claim ok jti=...` |
| E3 | PurchaseViewerVC 受領 | wallet に「IW3IP 購入閲覧クレデンシャル」 |
| E3 | eth_did_bound | audit `marketplace/issued` |
| E4 | merchandise から逆引きで 5 件取得 | `count: 5, rows: [..., {value: 31}, ..., {value: 35}]` |
| E5 | audit 連鎖が読める | `service_token_used` と `viewer_token_used` が同 dataset で連鎖 |

---

## v1 lane との対比 (再掲)

| 観点 | v1 (`emitUpload`) | v2 / Stage 6 |
| --- | --- | --- |
| seller の身元 | 暗黙 (Merchandise の owner) | ServiceVC の holder_did |
| seller の書き込み | (off-line で IPFS に upload) | publisher への ingest (audit 残る) |
| buyer の身元 | MetaMask の eth address のみ | eth + did:jwk (eth_did_bound で紐付け) |
| データ配信 | encryptURI 復号 | publisher API (Bearer ViewerToken) |
| 監査の網羅性 | on-chain Upload event のみ | audit 6 行 (write x5 + read x1 + bridge 等) |

---

## トラブルシューティング

### A. Step E2 で `dataset=home/env/temperature` ではなく fallback の値が出る

**原因**: Merchandise が古いデプロイで `dataset_id` を持っていない (Stage 6 case B 以前のデプロイ)。

**対処**: Hardhat ノードを再起動 (`Ctrl+C` → `npx hardhat node`) してから
最新の `deployMerchandiseWithIoTMarket.ts` で再デプロイ。MetaMask は
chainId キャッシュリセット必要 (Step 5 トラブルシューティング A)。

### A2. Step E1 開始時に `SERVICE` が空になる

```
SERVICE=
```

**原因**: Step 0-D で publisher を再起動した直後で、過去の
`service_token_issued` ログが消えており、まだ E1-A (発行) と
E1-B (提示) を完了していない。

**対処**: 「ServiceVC を **発行** → wallet で受領 → ServiceVC を
**提示**」の 2 ステップを実行してから token を grep する。発行と
提示は別 URL (`/issuer/offer` と `/verifier/request`) なので両方
踏むこと。

### B. Step E4 で `count: 0`

**原因**: Step E1 の ServiceToken ingest が、提示と別 publisher セッションだった
(再起動した) 等で `app.state.ingested` がクリアされている。

**対処**: Step E1 と Step E4 の間で publisher を再起動しないこと。
再起動した場合は Step E1 から仕切り直し。

### C. その他

[Stage 5 のトラブルシューティング](marketplace-vc-bridge.md#トラブルシューティング)
が引き続き有効。

---

## 限界 (Stage 7+ で扱う候補)

- **seller / buyer が同一 wallet**: 真の seller-buyer 分離には iPhone を 2 台
  使うか、wallet 内のアカウント切替に対応する UI が必要
- **SellerVC**: seller が `Merchandise` を登録するときに身元 VC を要求する
  ガバナンス層は未実装 (案 C)
- **連続性の観測**: ServiceVC の TTL 1 時間内に複数の buyer が読み出す
  ケース (multi-tenant read) のストレステストは別途
- **EIP-712 署名**: eth_addr ↔ did:jwk のなりすまし対策はまだ MVP 状態

## 関連

- [Marketplace VC Bridge 設計仕様](../design/marketplace-vc-bridge-spec.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
- [SSI Viewer (Stage 3)](ha-ssi-viewer.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
- [Marketplace × Wallet bridge (Stage 5)](marketplace-vc-bridge.md)
