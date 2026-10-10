# 信頼度に応じて見せる中身を変える (DataUserVC / Stage T)

受信者の信頼度 (DataUserVC) に応じて、見せる中身を Tier 3 / Tier 2 / Tier 1 の 3 段階に切り替えます。画像 / 動画の実体の配信（§8〜§11）は [画像・動画を段階アクセスで配信する](data-user-vc-tiered-media.md) で、映像を意味的中間表現に変換し、受信者の信頼度に応じて出力を変えるレンダリング（trust-aware rendering、§12〜§13）は [意味レベルで段階化する](data-user-vc-tiered-semantic.md) で扱います。3 ページ全体で長めのハンズオンです。

> **やること**: 信頼度ごとに動画 / 画像 / 要約だけを出し分ける動作の確認
>
> **前提**: [HA SSI Wallet](ha-ssi-wallet.md) と [USB ウェブカメライベント共有](webcam-event-sharing.md)
>
> **使うもの**: PC + スマホ (iw3ip-wallet)
>
> **所要時間**: 60 分くらい（続編の 2 ページは別に 90 分と 60〜90 分）

このハンズオンは、[スマホSSIウォレットサンプル](ha-ssi-wallet.md) と
[USBウェブカメライベント共有サンプル](webcam-event-sharing.md) を踏まえ、
**データ受信者の信頼属性に応じてカメラ視野を 3 段階に絞る**流れを確認します。

設計の根拠は [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) に
まとめています。先に読むと理解しやすくなります。

処理の流れは次のとおりです。

`Wallet -> DataUserVC 提示 -> trustScore 算出 -> /marketplace/claim -> PurchaseViewerVC -> /platform/data に image/video が出る/出ない`

## 最短ルート

1. publisher / hardhat / bridge を起動する
2. **DataUserVC** を 3 種類のプロファイルで発行する（gov-full / enterprise-access / low-deny）
3. 各プロファイルで `/marketplace/claim` を呼び出して `allowed_views` を比較する
4. PurchaseViewerVC を提示して `/platform/data` の出力差を見る

## このページで分かること

- DataUserVC を発行・提示する OID4VCI（VC の発行プロトコル）/ OID4VP（VC の提示プロトコル）の流れ
- `entityType / purpose / legalCompliance / dataHandlingPolicy / misuseRecord`
  の組み合わせで `full / access / denied` がどう変わるか
- `/platform/data` の出力に `image_cid` / `video_cid` が現れる / 消える条件

## つまずきやすい点

- `data_user_attrs` を省くと、既定で **`event` のみ**になります（image/video は出ません）
- score >= 80 でも、`entityType` が `Enterprise` のままでは **`full` になりません**
- ViewerToken の発行後に `data_user_attrs` を変えても、**発行済みの token には反映されません**

## 前提

- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) を一度動かしている
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) を理解している
- Docker / Docker Compose が使える
- `curl`, `jq` が使える

本ページのコマンド例は、教材リポジトリを `~/program/Blockchain_IoT_Marketplace` に clone した前提で示します。

- コマンド中の `$HOST_IP` と URL 中の `<HOST_IP>` は、PC の LAN IP を表します。使うターミナルごとに、最初に `export HOST_IP=<PC の LAN IP>` を実行してください (IP の調べ方は [ハンズオンの概要](index.md#host-ip) を参照)。ブラウザやスマホに入力する URL の `<HOST_IP>` は、同じ IP に置き換えます

## 0b. 実データ（画像 / 動画）統合の選び方

このハンズオンでは、画像 / 動画の実体を受信者に届ける方法を 3 種類用意しており、
本ページでは案 A / 案 B / 案 C と呼びます。学習のしやすさと実用性の両面から
**案 B**（publisher 内蔵の HTTP メディア・ゲートウェイ）を推奨します。実機で提供から
受信までを通して実行する（end-to-end）検証も、案 B で済んでいます。

| 案 | image / video の出処 | 実体取得 | 工数 | 用途 |
|---|---|---|---|---|
| A | プレースホルダ CID 文字列のみ | 不可 | 即時 | tier 投影の挙動だけ確認 |
| **B**（推奨） | publisher 配下 `/media/<sha256>.<ext>` 経由で配信 | ブラウザ / iPhone Safari でそのまま表示 | 同梱済み | デモ / ハンズオン |
| C | ローカル kubo IPFS daemon で content-addressed 配信 | publisher の `/ipfs/<cid>` リバースプロキシ + 任意の公開 gateway | 同梱済み（`--profile ipfs` で起動） | 本番に近い分散デモ |

表中の「tier 投影」は、受信者の Tier に応じて `/platform/data` の応答に含めるキーを
選別する処理を指します。

このページの §2〜§7 は基本フロー（DataUserVC + tier 投影）です。
**案 B の実データ統合は [§8](data-user-vc-tiered-media.md#8-実データ統合案-bhttp-メディアゲートウェイ)**、**案 C の IPFS 統合は [§9](data-user-vc-tiered-media.md#9-案-cローカル-kubo-ipfs-daemon-で分散配信)** で扱います。
案 C は案 B の機能をすべて含みます。`/media/upload` のレスポンスに `cid` が増えるだけなので、
provider 側スクリプトは変更せずに使えます。

## 1. サービス起動

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml up -d publisher bridge mosquitto
```

ヘルスチェック:

```bash
curl -s localhost:8080/health | jq .
curl -s localhost:8080/.well-known/openid-credential-issuer \
  | jq '.credential_configurations_supported | keys'
# -> ["ConsentVC", "DataUserVC", "PurchaseViewerVC", "SellerVC", "ServiceVC", "ViewerVC"]
```

`DataUserVC` がリストに出ていれば、起動は完了です。

## 2. 3 種類の DataUserVC オファーを作る

DataUserVC の発行ページを PC のブラウザで開きます。表示された QR コードをスマホのウォレットで読み取り、DataUserVC として保存します。3 つのプロファイルそれぞれについて行います。

### 2a. Tier 3（full）プロファイル — 政府機関 + 犯罪捜査 + ISO27001

```
http://<HOST_IP>:8080/issuer/offer?type=DataUserVC&entity_type=GovernmentOrganization&purpose=CrimeSearch&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false
```

### 2b. Tier 2（access）プロファイル — 企業 + 研究 + ISO27001

```
http://<HOST_IP>:8080/issuer/offer?type=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false
```

### 2c. Tier 1（denied）プロファイル — 企業 + 研究 + ポリシーなし + 濫用記録あり

```
http://<HOST_IP>:8080/issuer/offer?type=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true
```

## 3. /marketplace/claim を 3 通り呼び出す

`/marketplace/claim` は、購入が成立したことを publisher に伝える API です。通常は bridge が購入のイベントを検知して呼び出しますが、ここでは tier の違いだけを確認するため、curl で直接呼び出します。`data_user_attrs` に DataUserVC と同じ属性を渡すと、publisher が trustScore を計算し、その購入で閲覧できる範囲を決めます。

`merchandise_address`、`buyer_eth_addr`、`tx_hash`、`dataset_id` の 4 つは必須です。ここでは実際の購入を伴わないので、`tx_hash` には呼び出しごとに異なる任意の文字列を入れます (同じ `tx_hash` で再度呼ぶと、前回と同じ claim が返ります)。

### 3a. Tier 3 — 動画まで開く

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{
    "merchandise_address": "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
    "buyer_eth_addr": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "tx_hash": "0xdemo-tier3",
    "dataset_id": "home/event/possible_littering",
    "data_user_attrs": {
      "entityType": "GovernmentOrganization",
      "purpose": "CrimeSearch",
      "legalCompliance": true,
      "dataHandlingPolicy": "ISO27001",
      "misuseRecord": false
    }
  }' | tee /tmp/claim.json | jq '{claim_id, offer_url}'
```

応答には `claim_id`、`offer_url` (PurchaseViewerVC の発行ページ)、`deeplink` が含まれます。決まった tier は、発行される PurchaseViewerVC の種類として deeplink に入っています。次のコマンドで取り出せます。

```bash
jq -r .deeplink /tmp/claim.json \
  | python3 -c "import sys,json,urllib.parse; print(json.loads(urllib.parse.unquote(sys.stdin.read().split('credential_offer=',1)[1]))['credential_configuration_ids'])"
# -> ['PurchaseViewerVC.full']
```

| 種類 | 閲覧できる範囲 (`allowed_views`) | trustScore |
|---|---|---|
| `PurchaseViewerVC.full` | `event` / `image` / `video` | 80 以上で、政府機関など |
| `PurchaseViewerVC.access` | `event` / `image` | 60 以上 |
| `PurchaseViewerVC.event` | `event` のみ | 上記以外、または `data_user_attrs` なし |

### 3b. Tier 2 — 画像まで

3a のコマンドの `data_user_attrs` で、`entityType` を `Enterprise`、`purpose` を `Research` に、`tx_hash` を `0xdemo-tier2` に変えて実行します。trustScore は 75 で、deeplink の種類は `['PurchaseViewerVC.access']` になります。

### 3c. Tier 1 — `data_user_attrs` を省く既定値

```bash
curl -s -X POST localhost:8080/marketplace/claim \
  -H 'content-type: application/json' \
  -d '{
    "merchandise_address": "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
    "buyer_eth_addr": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
    "tx_hash": "0xdemo-tier1",
    "dataset_id": "home/event/possible_littering"
  }' | tee /tmp/claim.json | jq '{claim_id, offer_url}'
```

deeplink の種類は `['PurchaseViewerVC.event']` になります。

## 4. PurchaseViewerVC を発行 → 提示 → /platform/data

§3 の応答の `offer_url` (ホスト名の部分を `<HOST_IP>` に置き換えます) を PC のブラウザで開き、QR コードをウォレットで読み取って、各 claim の PurchaseViewerVC を受け取ります。ウォレットには tier ごとに別のカードとして表示されます。その後、[Stage 5](marketplace-vc-bridge.md) の Step 8 と同じ手順で提示します。
提示が受理されると ViewerToken が発行されます。この ViewerToken で `/platform/data` を
呼び出すと、プロファイルごとに次の差が出ます。

| プロファイル | `event` | `image_cid` | `video_cid` |
|---|---|---|---|
| 3a Tier 3 (gov full) | あり | あり | あり |
| 3b Tier 2 (enterprise) | あり | あり | **なし** |
| 3c Tier 1 (default) | あり | **なし** | **なし** |

取得した ViewerToken をシェル変数 `VIEWER_TOKEN` に入れて、次を実行します。

```bash
curl -s -H "authorization: Bearer $VIEWER_TOKEN" \
     'localhost:8080/platform/data?dataset_id=home/event/possible_littering' | jq .
```

Tier 2 / Tier 1 では、`image_cid` / `video_cid` のキー自体が応答に含まれないことを
確認します（値が `null` になるのではなく、キーごと省かれます）。

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
```

スマートコントラクト `DataUserVerifier.sol` の重みと、`trust_score.py` の
重みが一致することは、テストの 4 ケース（GovernmentOrganization / Enterprise /
低スコア / score>=80 でも entity が違うと full にならない）で確認します。

## 7. 実機検証で観測した値

iPhone のウォレット（iw3ip-wallet。Sphereon mobile-wallet の fork）で一連の流れを
通して実行すると、3 ティアそれぞれで次の値が
ViewerToken の発行ログ（`viewer_token_issued ... views=...`）と
`/platform/data` の応答に出ます。

| Tier | DataUserVC profile | trust_score | views | image_cid | video_cid | video_duration_sec |
|---|---|---|---|---|---|---|
| **3** gov | gov + crime + ISO27001 | 80 | `event+image+video` | あり | あり | あり |
| **2** ent | enterprise + research + ISO27001 | 75 | `event+image` | あり | **なし** | **なし** |
| **1** low | `data_user_attrs` 省略 | n/a (default) | `event` | **なし** | **なし** | **なし** |

ウォレットでは Tier 別に **3 種類の異なる表示名**で `PurchaseViewerVC` が
並びます（`PurchaseViewerVC.full / .access / .event`）。VCT（VC の種類を表す識別子）は
同じでも display name が異なるため、ユーザはどのカードがどのティアかを
区別できます。

!!! tip "実機検証で詰まったところ"
    実機で初めて試すと、次の症状が出ることがあります。いずれも教材リポジトリの
    2 つのブランチ（`fix/stage-t-purchase-viewer-binding`、
    `feat/stage-t-tier-display-and-projection`）で修正済みなので、最新の `main` を
    使えば発生しません。

    - ウォレットに「No Available Credential」と表示される。PurchaseViewerVC の
      plain claims に `subject_id` が必要でした（修正済み）
    - 3 枚の VC を見分けられない。Tier ごとに `credential_configuration_id` を分け、
      `display.name` が変わるようにしました（修正済み）
    - `/simulate/publish` で `image_cid` を渡しても `/platform/data` の応答に出ない。
      pipeline が `image_cid` を payload の最上位（top-level）に移すようにしました（修正済み）
    - `/marketplace/claim` の応答に含まれる `deeplink` をウォレットで直接開くと、
      claim との対応付けが確実に保たれます。`/issuer/offer?claim_id=...` を
      自分で呼び出す方法より確実です

## 次に進む

- [画像・動画を段階アクセスで配信する](data-user-vc-tiered-media.md) — 続きの §8〜§11（HTTP メディア・ゲートウェイ、IPFS、PWA Viewer、PWA Provider）
- [意味レベルで段階化する](data-user-vc-tiered-semantic.md) — 続きの §12〜§13（VLM による意味レベルの段階化、意味的中間表現）
- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) — Phase 2 wallet を立ち上げる
- [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) — 設計根拠
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) — イベントを条件付きで共有する
