# 出品者の身元を裏付ける (Seller VC / Stage 7)

出品者が「誰として」その dataset を売るのかを VC で裏付ける層を加えます。5 種類目の VC である SellerVC を扱います。

> **やること**: SellerVC を発行・提示して出品身元を裏付ける
>
> **前提**: [ServiceVC で書いたデータを PurchaseViewerVC で読み出す (Stage 6)](marketplace-vc-end-to-end.md)
>
> **起動しておくもの**: Stage 6 の Hardhat ノード (Step E0 でデプロイしたチェーン)、publisher、bridge。Step S6 では iot-market-ui も使います
>
> **使うもの**: PC + MetaMask + スマホ (iw3ip-wallet)
>
> **所要時間**: 約 45 分

!!! note "このページの位置付け"
    - Part 2 の 2.4 の 3 ページ目です ([ハンズオン](index.md#24-マーケットプレイス--ウォレット-v2-経路))。
    - 前のページ: [ServiceVC で書いたデータを PurchaseViewerVC で読み出す (Stage 6)](marketplace-vc-end-to-end.md)
    - 次のページ: [スマホアプリで購入から閲覧まで (Stage A)](marketplace-mobile-app.md)

!!! abstract "5 種目の VC: SellerVC"
    マーケット出品時に **「誰が」この dataset を売るのか**を VC で
    裏付ける層を追加します。Stage 1〜6 と同じく、提示 → トークン
    取得 → API 呼び出しの順に進めます。設計の詳細は [SellerVC — マーケット出品ガバナンス VC (Stage 7 設計仕様)](../design/seller-vc-spec.md) を参照してください。

!!! tip "licensed_datasets の選択"
    例は `home/env/temperature,home/env/humidity` ですが、Stage 0 と
    一貫性を持たせるなら `home/event/possible_littering,home/event/flood_risk_high`
    を加えても構いません。SellerVC の `licensed_datasets` は配列なので、両方を載せられます。
    ただし、Step S3〜S6 で使う `home/env/temperature` は必ず含めます (含めないと S4 が 403 `seller_token_dataset_not_licensed` になります)。

## 目的

- ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC に続く **5 つ目の VC**
  (SellerVC) を扱う
- Seller が `IoTMarket.registerMerchandise()` で出品した直後、
  publisher に対して **`/marketplace/register`** を呼び、
  「この Merchandise は私が売っている」を VC で証明する
- Buyer が `/platform/data?merchandise=<addr>` を読むときに
  **`seller_did`** が返ってくる流れを確認する

## このページで分かること

- SellerVC の `licensed_datasets` で「この seller はどの dataset を売って良いか」
  を VC に記載する仕組み
- SellerToken (24 時間、何度でも使える) で `/marketplace/register` を呼ぶこと (SellerToken は、[画像・動画を段階アクセスで配信する (DataUserVC / Stage T)](data-user-vc-tiered-media.md) §11 の `/provider/publish` にも使える)
- (任意) 環境変数 `MARKETPLACE_HARDHAT_RPC` 設定時、publisher が
  on-chain で `Merchandise.getOwner() == seller_eth_addr` を検証すること
- 監査ログに **`marketplace/seller_registered`** 行が追加されること
- buyer 側のレスポンスに **`seller_did`** が含まれ、誰が売ったかを確認できること

## 全体像

```
[Seller wallet]
   │ 1. /issuer/offer?type=SellerVC → 受領
   │ 2. /verifier/request?vc_kind=SellerVC → 提示
   │ 3. SellerToken (24h 複数回可)
   │
[Seller (Hardhat console)]
   │ 4. Merchandise を deploy + IoTMarket.registerMerchandise()
   │
[Seller (curl or /seller UI)]
   │ 5. POST /marketplace/register
   │    Authorization: Bearer SellerToken
   │    body: merchandise_address + seller_eth_addr + tx_hash + dataset_id
   │
[publisher]
   │ - SellerToken 検証 (licensed_datasets に dataset_id が含まれる)
   │ - (任意) Merchandise.getOwner() を on-chain 検証
   │ - audit: raw_topic=marketplace/seller_registered
   │ - publisher 内部に merchandise → seller_did binding
   │
[Buyer (Stage 5/6 の購入動線)]
   │ → GET /platform/data?merchandise=<addr>
   │   レスポンスに seller_did が同梱される
```

## 前提

- Stage 5 / 6 のハンズオンを通せている
- deploy script が、`dataset_id` を `additionalInfo` に持つ Merchandise を作る版 (Stage 6 以降) である
- publisher が Stage 7 に対応したコード (`main` の最新) で起動している
- iw3ip-wallet (iPhone) が動く
- コマンド中の `$HOST_IP` と URL 中の `<HOST_IP>` は、PC の LAN IP を表します。使うターミナルごとに、最初に `export HOST_IP='<PC の LAN IP>'` を実行してください (IP の調べ方は [ハンズオン](index.md#host-ip) を参照)。ブラウザやスマホに入力する URL の `<HOST_IP>` は、同じ IP に置き換えます
- 教材リポジトリは `~/program/Blockchain_IoT_Marketplace`、ウォレットは `~/program/iw3ip-wallet` に clone した前提で、Mac と iPhone を使う手順を示します

---

## Step S0. 環境を最新コードに揃える

最新のコードを取得します。

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only
```

on-chain owner verify を有効にする場合だけ、次を実行します (任意)。

```bash
echo "MARKETPLACE_HARDHAT_RPC=http://host.docker.internal:8545" >> infra/.env
```

git pull の後は、`--build` で publisher を作り直します。古いままだと owner verify が skipped のままになります。

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d
sleep 5
```

5 種目の VC (SellerVC) が出ているかを確かめます。

```bash
curl -s http://$HOST_IP:8080/.well-known/openid-credential-issuer \
  | python3 -m json.tool | grep -A1 SellerVC
```

期待する出力 (抜粋):

```text
            "vct": "https://iw3ip.example/credentials/SellerVC/v1",
```

`grep -A1` は、`SellerVC` を含む行とその次の行をすべて表示するので、実際にはこれより多くの行が出ます。その中にこの `vct` の行があれば、SellerVC が発行できる状態です。

S0 で publisher が作り直されると (新しいイメージになったときや、`infra/.env` を変えたとき)、Stage 6 までに publisher のメモリにあった購入の申告と取り込んだ行は消えます。そのため、Step S5 の `count` が 0 になることがあります。Step S5 で確かめる `seller_did` には影響しません。

---

## Step S1. SellerVC を発行

### 何を確認するか
- `seller_id` と `licensed_datasets` (dataset_id のホワイトリスト) を query で渡し、
  wallet が SellerVC を受領する
- `licensed_datasets` に書いた dataset 以外は後で reject される

### 操作

PC ブラウザで:

```
http://<HOST_IP>:8080/issuer/offer?type=SellerVC&seller_id=ertl-seller-001&licensed_datasets=home/env/temperature,home/env/humidity
```

QR を iPhone wallet で読み取り → **「IW3IP セラークレデンシャル」** 承認 → 受領。

### 期待結果
- wallet の VC 一覧に「IW3IP セラークレデンシャル」が追加
- claim:
    - `seller_id`: `ertl-seller-001`
    - `licensed_datasets`: `["home/env/temperature", "home/env/humidity"]`
    - `subject_id`: `did:jwk:...`

---

## Step S2. SellerVC を提示して SellerToken を取得

### 何を確認するか
- 提示で **SellerToken** が払い出される (PolicyToken / ViewerToken / ServiceToken
  とは別のトークン)
- TTL は **24 時間 (86400 秒)**
- レスポンスに `licensed_datasets` がそのまま入る

### 操作

PC ブラウザで:

```
http://<HOST_IP>:8080/verifier/request?vc_kind=SellerVC
```

`dataset_id` パラメータは不要です (SellerVC は特定の dataset に依存しません)。
QR → wallet で **SellerVC** を選択して提示。

### 期待出力

```bash
PUB=$(docker ps -qf name=publisher)
docker logs $PUB 2>&1 | grep "seller_token_issued" | tail -1
```

→ (publisher のログは 1 行が JSON です。その `"message"` に次の内容が入ります)
```
seller_token_issued jti=... token=... seller_id=ertl-seller-001 licensed=['home/env/temperature', 'home/env/humidity'] ttl=86400s
```

token を変数に入れます。

```bash
SELLER=$(docker logs $PUB 2>&1 | grep "seller_token_issued" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')
echo "SELLER=$SELLER"
```

---

## Step S3. Merchandise を deploy + IoTMarket に登録

### 何を確認するか
- Hardhat console で Merchandise を deploy し、`IoTMarket.registerMerchandise()` を呼ぶ
- deploy 時に `additionalInfo` に `dataset_id=home/env/temperature` を入れる
  (Stage 6 と同じ形式)

### 操作

ターミナル B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
// seller = Account #1 (iotOwner) を使う
const [_, iotOwner] = await ethers.getSigners();
console.log("seller:", await iotOwner.getAddress());

// PubKey 登録 (PubKey 初回利用時のみ)
const pubKey = await ethers.getContractAt("PubKey", "0x5FbDB2315678afecb367f032d93F642f64180aa3", iotOwner);
try { await (await pubKey.registerKey("[seller-key]")).wait(); } catch (e) { /* 登録済なら無視 */ }

// Merchandise を deploy (dataset_id を additionalInfo に含める)
const dataHash = "0x" + "5".repeat(64);
const merch = await (await ethers.getContractFactory("Merchandise", iotOwner)).deploy(
  ethers.parseEther("0.02"),
  dataHash,
  pubKey,
  [],   // deniedBuyers
  ["fileType", "dataSize", "dataset_id"],
  ["json", "1MB", "home/env/temperature"]
);
const deployReceipt = await merch.deploymentTransaction().wait();
const merchandiseAddress = await merch.getAddress();
console.log("merchandise:", merchandiseAddress);
console.log("deploy tx:", deployReceipt.hash);

// IoTMarket に登録
const market = await ethers.getContractAt("IoTMarket", "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512", iotOwner);
const regTx = await market.registerMerchandise(merch);
const regReceipt = await regTx.wait();
console.log("register tx:", regReceipt.hash);
.exit   // コンソールを抜けてシェルに戻る
```

### メモ

console に表示された `seller`、`merchandise`、`deploy tx`、`register tx` の 4 つをメモします。

---

## Step S4. publisher に `/marketplace/register` で seller binding

### 何を確認するか
- Bearer SellerToken で `/marketplace/register` が成功する
- `licensed_datasets` に含まれる dataset_id だけ通る
- (任意) MARKETPLACE_HARDHAT_RPC 有効時、Merchandise.getOwner() が検証される
- audit log に `marketplace/seller_registered` 行が出る

### 操作

Step S2 で `SELLER` を設定したターミナル (リポジトリのトップ) で実行します。最初の 3 行の `<…>` は、山かっこを含めて全体を Step S3 でメモした値に置き換えてから実行してください。置き換え忘れても、引用符で囲んでいるのでエラーにはならず、その文字列がそのまま送られます。`MARKETPLACE_HARDHAT_RPC` が未設定なら、でたらめな値のまま登録されてしまいます。設定済みの場合、`MERCH` を置き換え忘れると on-chain の照会に失敗して HTTP 502 `chain_rpc_failed`、`SELLER_ETH` を置き換え忘れると on-chain の owner と一致せず HTTP 403 `owner_mismatch` で失敗します。`TX_HASH` は照合されないので、置き換え忘れてもそのまま登録されます。

```bash
cd ~/program/Blockchain_IoT_Marketplace
SELLER_ETH='<Step S3 の seller address>'
MERCH='<Step S3 の merchandise address>'
TX_HASH='<Step S3 の register tx hash>'
curl -s -X POST http://$HOST_IP:8080/marketplace/register \
  -H "Authorization: Bearer $SELLER" \
  -H "Content-Type: application/json" \
  -d "{
    \"merchandise_address\": \"$MERCH\",
    \"seller_eth_addr\": \"$SELLER_ETH\",
    \"tx_hash\": \"$TX_HASH\",
    \"dataset_id\": \"home/env/temperature\"
  }" | python3 -m json.tool
```

### 期待出力

```json
{
    "registered": true,
    "merchandise_address": "0x...",
    "seller_did": "did:jwk:...",
    "dataset_id": "home/env/temperature",
    "tx_hash": "0x...",
    "register_count": 1,
    "owner_verify": "skipped"
}
```

`owner_verify` の値:

- `skipped`: `MARKETPLACE_HARDHAT_RPC` 未設定 (テスト/開発用)
- `verified`: on-chain で getOwner() が seller_eth_addr と一致

RPC に到達できない場合は、レスポンスに `owner_verify` は含まれず、HTTP 502 (`chain_rpc_failed`) で失敗します。on-chain の owner が `seller_eth_addr` と一致しない場合は 403 (`owner_mismatch`) です。

### audit log

```bash
curl -s "http://$HOST_IP:8080/audit/logs?limit=3" | python3 -m json.tool
```

結果は新しい順のリストです。先頭の行が `marketplace/seller_registered` になります (残りの行は `...` で省略しています):

```json
[
    {
        "id": ...,
        "ts": "...",
        "action": "allow",
        "subject_did": "did:jwk:...",
        "dataset_id": "home/env/temperature",
        "purpose": "register",
        "reason": "seller_register:jti=...:merchandise=0x...:eth=0x...:tx=0x...:owner_verify=skipped",
        "message_hash": "",
        "raw_topic": "marketplace/seller_registered",
        "holder_did": "did:jwk:...",
        "vc_hash": null,
        "presentation_verified": "allow"
    },
    ...
]
```

---

## Step S5. Buyer から見える `seller_did`

### 何を確認するか
- Buyer が PurchaseViewerVC 経由で `/platform/data?merchandise=<addr>` を呼び出すと、
  レスポンスに **`seller_did`** が含まれる

### 操作

bridge は、起動時に IoTMarket に登録されていた Merchandise だけを監視します。Step S3 で追加した Merchandise を購入する前に、bridge を再作成して認識させます。

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml --profile mv-bridge up -d --force-recreate bridge
```

その後、[購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md) の手順で MetaMask 購入 → wallet 受領
→ ViewerToken 取得 → 取得の順に進めます。ViewerToken の有効期限は 60 秒なので、提示したらすぐに次のコマンドを実行し、`viewer_token_expired` が出たら提示からやり直してください。`<buyer の ViewerToken>` は、山かっこを含めて全体を [Stage 5 の Step 8](marketplace-vc-bridge.md#step-8-viewertoken-を取り出す) で取り出したトークンに置き換えてから実行します。置き換え忘れると、その文字列がそのままトークンとして送られ、401 (`viewer_token_unknown`) で失敗します。

次のコマンドは、Step S4 で設定した `$MERCH` を使います。Step S4 と同じターミナルで実行してください。最初の行で `MERCH=0x...` と表示されれば大丈夫です。`MERCH=` のあとが空なら別のターミナルなので、先に `MERCH='<Step S3 の merchandise address>'` の `<…>` を Step S3 でメモした値に置き換えて実行します。

```bash
echo "MERCH=$MERCH"
TOKEN='<buyer の ViewerToken>'
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://$HOST_IP:8080/platform/data?merchandise=$MERCH" | python3 -m json.tool
```

### 期待出力

```json
{
    "dataset_id": "home/env/temperature",
    "count": N,
    "read_count": 1,
    "seller_did": "did:jwk:...(seller の DID)",
    "allowed_views": [
        "event"
    ],
    "rows": [...]
}
```

`seller_did` は Step S4 で binding した値です。Stage 5/6 だけ通した
Merchandise (`/marketplace/register` を呼んでいない) は `"unknown"` が返ります。

---

## Step S6. iot-market-ui の `/seller` ページから登録

この節は任意です。Step S1〜S4 の発行、提示、登録を、Web UI から行う別の方法です。Stage 7 の完了条件 (`seller_did`) は、Step S5 までで確かめられます。

### 何を確認するか
- Web UI からも `/marketplace/register` を呼び出せること
- フォームの 3 ステップ構造 (発行 → 提示 → 登録) が seller 視点で完結すること

### 操作

iot-market-ui を起動済の状態で:

```
http://<HOST_IP>:5173/seller
```

UI の手順:

1. **Step 1**: `seller_id` と `licensed_datasets` を入れて
   「/issuer/offer を開く」 → wallet で受領
2. **Step 2**: 「/verifier/request を開く」 → wallet で SellerVC を提示 →
   publisher ログから `SellerToken` をコピー
3. **Step 3**: 上で取得した token + Step S3 の merchandise/eth/tx を貼り、
   `dataset_id` を入力して「Register」

成功時、画面下に Step S4 と同じ JSON レスポンスが表示されます。

---

## 完了判定マトリクス

| 工程 | 確認項目 | 出力例 (実機検証で得た値) |
| --- | --- | --- |
| S0 | 5 種類目の SellerVC が metadata に出る | `"vct": "https://iw3ip.example/credentials/SellerVC/v1"` |
| S1 | wallet に SellerVC | claim に `seller_id: "ertl-seller-001"`、`licensed_datasets: ["home/env/temperature", "home/env/humidity"]` |
| S2 | SellerToken 発行 | `seller_token_issued jti=... seller_id=ertl-seller-001 licensed=['home/env/temperature', 'home/env/humidity'] ttl=86400s` |
| S3 | Merchandise + IoTMarket 登録 | deploy tx + register tx の 2 件 |
| S4 | `/marketplace/register` 成功 | `{ "registered": true, "seller_did": "did:jwk:...", "register_count": <累積回数>, "owner_verify": "verified" / "skipped" }` |
| S4 | audit log | `raw_topic=marketplace/seller_registered`, `reason=seller_register:jti=...:merchandise=0x...:eth=0x...:tx=0x...:owner_verify=...` |
| S5 | buyer side seller_did 表示 | `{ "dataset_id": "home/env/temperature", "seller_did": "did:jwk:...", ... }` |
| S6 | UI 経由でも登録可 | `/seller` フォーム送信後にレスポンス JSON が画面下に表示 |

**Stage 7 の完了条件**: `/platform/data?merchandise=<addr>` のレスポンス
JSON に **`seller_did` フィールドが did:jwk: で出る**こと。これにより、どの
seller がそのデータを売ったかを VC で裏付けた記録を確認できます。

---

## トラブルシューティング

### A. `seller_token_dataset_not_licensed` で 403

**原因**: SellerVC の `licensed_datasets` に含まれない dataset を `/marketplace/register`
で渡した。

**対処**: Step S1 で `licensed_datasets` の中身を増やしておく、または対象 dataset の
SellerVC を別途発行する。

### B. `MARKETPLACE_HARDHAT_RPC` 有効化時に `chain_rpc_failed` (502)

**原因**: publisher コンテナから host.docker.internal:8545 に到達できない。

**対処**: Hardhat ノードを `npx hardhat node --hostname 0.0.0.0` で起動しているか、
`infra/.env` に `MARKETPLACE_HARDHAT_RPC=http://host.docker.internal:8545` があるかを確認し、
`cd ~/program/Blockchain_IoT_Marketplace && docker compose -f infra/docker-compose.yml up -d --build publisher` を再実行します。
publisher の `extra_hosts` は `infra/docker-compose.yml` に設定済みなので、追加は不要です。

### C. `owner_mismatch` (403)

**原因**: `/marketplace/register` に渡した `seller_eth_addr` と、on-chain の
Merchandise.getOwner() が違う。deploy に使った Hardhat account と別のアドレスを
`seller_eth_addr` に指定したときに起こる。

**対処**: `seller_eth_addr` に、Merchandise を deploy した Hardhat signer
(例: Account #1 = `iotOwner`) のアドレスを指定する。

### D. Hardhat console で `try { ... } catch (e) { ... }` が SyntaxError

```
Uncaught SyntaxError: missing ) after argument list
```

**原因**: Hardhat console (Node REPL) は、入力が 1 つの文として完結した時点で
評価する。`try { ... }` のブロックが閉じた行で文が完結したと判断されるため、
`catch` を次の行に書くと構文エラーになる (括弧が閉じていない `deploy(...)` の
ような複数行の呼び出しは問題ない)。

**対処**: 1 行にまとめて書く。例:

```javascript
let r; try { r = await (await pubKey.registerKey("[buyer-key]")).wait(); console.log("ok:", r.status); } catch (e) { console.log("err:", e.message); }
```

### E. Hardhat ノードを再起動した後、`getState()` 等が 0x で返り `BAD_DATA` になる

```
ERROR: could not decode result data (value="0x", info={ "method": "getState", ...
```

**原因**: Hardhat ノードを `Ctrl+C` 等で停止すると **on-chain state が
全消去**される。すでにデプロイ済の PubKey / IoTMarket / Merchandise も
コードがなくなる。`provider.getCode(addr)` が `"0x"` (= 長さ 2) を返す
アドレスは contract が存在しないことを示す。

**対処**:

```javascript
// 主要アドレスにコードがあるか確認
const c1 = await ethers.provider.getCode("0x5FbDB2315678afecb367f032d93F642f64180aa3");  // PubKey
const c2 = await ethers.provider.getCode("0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512");  // IoTMarket
console.log({ pubKey: c1.length, ioTMarket: c2.length });
// 2 が出ていたら消えている
```

消えていれば、`.exit` でコンソールを抜けてから `npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost`
で再デプロイ。

注意: ノード再起動後の chain には、前セッションで自分が手動デプロイした
Merchandise (script の決定的アドレス以外) は復元されない。**chain reset
後は script デプロイ由来の 5 種だけを使う**のが安全。

### F. チェーン再デプロイ後、Account #2 の購入が `0x295f0a57` で revert する

**原因**: PubKey contract が再デプロイされて state がリセットされたため、
buyer (Account #2) の公開鍵登録が消えている。Merchandise.purchase() 内
で呼ばれる `i_pubKey.getPubKey(tx.origin)` が `PubKey__NotRegistered` (`0x295f0a57`) で revert する
([Stage 5 のトラブル対応 B](marketplace-vc-bridge.md#b-purchase-が-0x295f0a57-で-revert) と同じ)。

**対処**: PubKey を再登録する。

```javascript
const [_, __, buyer] = await ethers.getSigners();
const pk = await ethers.getContractAt("PubKey", "0x5FbDB2315678afecb367f032d93F642f64180aa3", buyer);
let r; try { r = await (await pk.registerKey("[buyer-key]")).wait(); console.log("ok:", r.status); } catch (e) { console.log("err:", e.message); }
```

seller (Account #1) を使う場合も同様に登録すること。

### G. Bridge ログに新しい Purchase event が出ない / bridge が止まる / wallet が `/issuer/token` で 400

**原因**: bridge は起動時に IoTMarket から Merchandise の一覧を読み、そのときの block 番号から先だけを見ます。Hardhat を再起動するとチェーンが空になり、block 番号も 0 に戻ります。

- 動いている bridge は、古い block 番号より先しか見ないので、新しい Purchase event を拾いません
- bridge を作り直しても、IoTMarket がまだ deploy されていなければ、一覧を読めずに `bridge: fatal` を出して終了します (exit 1)。自動では再起動しません

wallet の 400 は、旧チェーンで発行された pre_authorized_code を再利用しようとして、publisher 側で使用済みと扱われたときに出ます。

**対処**: 次の順にやり直します。

1. コントラクトを deploy し直します ([E](#e-hardhat-ノードを再起動した後getstate-等が-0x-で返り-bad_data-になる) の手順)。buyer の公開鍵も登録し直します ([F](#f-チェーン再デプロイ後account-2-の購入が-0x295f0a57-で-revert-する) の手順)
2. publisher を作り直す場合 (下の 400 が続く場合) は、ここで作り直します。publisher のメモリ上の SellerToken と seller の登録が消えるので、Step S2 からやり直します
3. Step S3 で Merchandise を deploy して登録し直し、Step S4 で seller binding をやり直します。新しいチェーンでは Merchandise のアドレスも変わるので、S3 で表示された値を使います。publisher を作り直していなくても S4 は必要です
4. S3 のあとで bridge を作り直します。bridge は作り直した時点の Merchandise だけを監視するので、S3 より前に作り直すと、新しい Merchandise を監視しません

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml --profile mv-bridge up -d --force-recreate bridge
sleep 5
docker logs iw3ip-mv-bridge 2>&1 | tail -5
```

期待する出力 (Merchandise の数は、deploy script の 5 個に S3 の分を足した数になります):

```text
bridge: listening to 6 merchandise(s)
bridge: starting poll from block <現在の番号>
bridge: started rpc=http://host.docker.internal:8545 market=0x... publisher=http://publisher:8080
```

`bridge: fatal` が出ている場合は、IoTMarket がまだ deploy されていません。手順 1 に戻ります。

その後、再度 `Merchandise.purchase()` を実行すると新しい Purchase event を捕捉できます。古い deeplink は破棄して、新しい `bridge: claim ok` 行から取り直してください。

wallet の 400 が続く場合は、Hardhat を起動し直したあとに同じ購入をして、取引の ID (tx hash) が前回と同じになっている可能性があります。publisher は同じ取引 ID には前回の使用済みのコードを返すので、手順 2 で publisher も作り直します。

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml --profile mv-bridge up -d --force-recreate publisher
```

このあと Step S2〜S4 をやり直し、最後に上の手順で bridge を作り直します。取り込んだデータ (Stage 6 Step E1 で書き込んだ行) も消えるので、S5 で行を見たい場合は Stage 6 Step E1 の書き込みもやり直します。

### H. その他

[Stage 5 のトラブルシューティング](marketplace-vc-bridge.md#トラブルシューティング)
(MetaMask chainId キャッシュ、`BRIDGE_PUBLIC_PUBLISHER_URL` 未設定 等)
も参照してください。

---

## 限界 / 将来課題

- **on-chain ガード**: 現状 `IoTMarket.registerMerchandise()` 自体は
  publisher 検証なしでも通る。on-chain でのガードは今後の課題 (Stage 8 以降)
- **EIP-712 署名**: SellerVC 経由の seller_did と eth address の紐付けは
  publisher が信頼する方式 (Stage 5/6 の eth_did_bound と同じ MVP レベル)
- **dataset_id をどこで宣言するか**: Stage 6 case B で `additionalInfo` の
  `dataset_id` 値を取れるようにしたが、`/marketplace/register` 側はクライアント
  POST に依存。将来は publisher 自身が `getAllAdditionalInfo()` を呼んで
  突合する案がある

## 次に進む

次は [スマホアプリで購入から閲覧まで (Stage A)](marketplace-mobile-app.md) です。購入から閲覧までを、スマホの画面だけで進めます。

- Hardhat ノード (ターミナル A) は、止めずに動かしたままにします。Stage A は、Stage 6 の Step E0 でデプロイしたコントラクトと、登録した公開鍵を使います。
- publisher と bridge は動かしたままで構いません。Stage A の Step M0 で作り直します。
- iot-market-ui は、Stage A の Step M0 で Ctrl+C で止めてから起動し直します。
- Metro は動かしたままにします。

## 関連

- [SellerVC — マーケット出品ガバナンス VC (Stage 7 設計仕様)](../design/seller-vc-spec.md)
- [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md)
- [ServiceVC で書いたデータを PurchaseViewerVC で読み出す (Stage 6)](marketplace-vc-end-to-end.md)
- [機器間での連携認証 (Service VC / Stage 4 prep)](ha-ssi-service.md)
- [データ提供者の認証 (Consent VC / Stage 1)](ha-ssi-wallet.md)
