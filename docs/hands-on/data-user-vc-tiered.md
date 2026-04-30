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

## 0b. 実データ（画像 / 動画）統合の選び方

このハンズオンには 3 種類の「データ実体の運び方」があります。教育・実用バランスで
**案 B**（publisher 内蔵の HTTP メディア・ゲートウェイ）を推奨。実機 e2e 検証も
案 B で動作確認済みです。

| 案 | image / video の出処 | 実体取得 | 工数 | 用途 |
|---|---|---|---|---|
| A | プレースホルダ CID 文字列のみ | 不可 | 即時 | tier 投影の挙動だけ確認 |
| **B** | publisher 配下 `/media/<sha256>.<ext>` 経由で配信 | ブラウザ / iPhone Safari でそのまま表示 | 同梱済み | デモ / ハンズオン |
| **C**（推奨） | ローカル kubo IPFS daemon で content-addressed 配信 | publisher の `/ipfs/<cid>` リバースプロキシ + 任意の公開 gateway | 同梱済み（`--profile ipfs` で起動） | 本番に近い分散デモ |

このページの §2〜§7 は基本フロー（DataUserVC + tier 投影）です。
**案 B の実データ統合は §8**、**案 C の IPFS 統合は §9** で扱います。
案 C は案 B の上位互換で、`/media/upload` のレスポンスに `cid` が増えるだけなので
provider 側スクリプトの差分は最小です。

## 1. サービス起動

```bash
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml up -d publisher hardhat bridge mosquitto
```

ヘルスチェック:

```bash
curl -s localhost:8080/healthz | jq .
curl -s localhost:8080/.well-known/openid-credential-issuer \
  | jq '.credential_configurations_supported | keys'
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
#    80
```

### 3b. Tier 2 — 画像まで

```bash
# data_user_attrs.entityType: "Enterprise", purpose: "Research"
# allowed_views: ["event", "image"], access_level: "access", score: 75
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
```

スマートコントラクト `DataUserVerifier.sol` の重みと、`trust_score.py` の
重みが一致することは、テストの 4 ケース（GovernmentOrganization / Enterprise /
低スコア / score>=80 でも entity が違うと full にならない）で確認します。

## 7. 実機検証で観測した値

iPhone（iw3ip-wallet）で end-to-end を流すと、3 ティアそれぞれで次の値が
ViewerToken のミントログ（`viewer_token_issued ... views=...`）と
`/platform/data` の応答に出ます。

| Tier | DataUserVC profile | trust_score | views | image_cid | video_cid | video_duration_sec |
|---|---|---|---|---|---|---|
| **3** gov | gov + crime + ISO27001 | 80 | `event+image+video` | あり | あり | あり |
| **2** ent | enterprise + research + ISO27001 | 75 | `event+image` | あり | **なし** | **なし** |
| **1** low | `data_user_attrs` 省略 | n/a (default) | `event` | **なし** | **なし** | **なし** |

ウォレットでは Tier 別に **3 種類の異なる表示名**で `PurchaseViewerVC` が
並びます（`PurchaseViewerVC.full / .access / .event`）。同じ VCT でも
display name が異なるため、ユーザはどのカードがどのティアか一目で
区別できます。

!!! tip "実機検証で詰まったところ"
    実機で初回試行すると次の症状が出やすいです。直近の 2 PR
    （`fix/stage-t-purchase-viewer-binding`、`feat/stage-t-tier-display-and-projection`）
    にすべてフォロー済みなので、最新の `main` で進めれば回避できます。

    - 「No Available Credential」が出る → PurchaseViewerVC の
      plain claims に `subject_id` が必要（修正済み）
    - 3 枚の VC の見分けが付かない → tier-aware
      credential_configuration_id で `display.name` が分岐（修正済み）
    - `/simulate/publish` で `image_cid` を渡しても `/platform/data` の
      投影に出ない → pipeline で top-level に hoist（修正済み）
    - `/marketplace/claim` の `deeplink` を直接ウォレットで開く
      （`/issuer/offer?claim_id=...` ではなく）と claim とのバインドが
      確実に保たれる

## 8. 実データ統合（案 B：HTTP メディア・ゲートウェイ）

§7 までで「ティア毎に `image_cid` / `video_cid` の **キーが消える / 残る**」
ことは確認できました。次に、**実体の画像 / 動画**まで含めて end-to-end を
通します。提供側にダミー JPEG / MP4 を生成 → publisher の `/media/upload` に
POST → 戻る URL を payload の `image_url` / `video_url` に乗せる、という流れです。

### 8.1 提供側スクリプトを 1 発で動かす

```bash
cd ~/program/Blockchain_IoT_Marketplace
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080
```

中身は次の 3 ステップを順に実行します：

1. 1×1 JPEG / MP4 fixture を生成（`fixtures/` 配下）
2. `POST /media/upload` で 2 つアップロード（sha256 で dedup）
3. `image_url` / `video_url` 付きイベントを `/simulate/publish` に送る

スクリプト出力に `image_url` / `video_url` の `http://192.168.68.53:8080/media/...`
形式の URL が表示されます。

### 8.2 受信側（既存の Tier 3 / 2 / 1 フロー）

§3〜§7 の流れをそのまま使います。Tier 別に `/platform/data` を取得すると：

- **Tier 3 (gov full)** → `event` + `image_url` + `video_url` + `video_duration_sec` 全部
- **Tier 2 (enterprise)** → `event` + `image_url` のみ。`video_url` キーは欠落
- **Tier 1 (default)** → `event` のみ

iPhone Safari で `image_url` をタップ → 1×1 JPEG が表示されます。
カスタム画像 / 動画を試したいときは：

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080 \
  --image /path/to/snapshot.jpg \
  --video /path/to/clip.mp4 \
  --video-duration-sec 12
```

### 8.3 案 B の制限

- ✅ `image_cid` / `video_cid`（案 A 用）と同じテーブルで投影される（共存可能）
- ❌ コンテンツアドレシング（実体ハッシュ）ではない — URL を知っている人は誰でも GET 可能
- ❌ 単一インスタンス：レプリカ / pinning なし

「コンテンツアドレシング + 分散保存」が要る場合は **案 C（IPFS / Web3.Storage）** に
進みます。`/media/upload` のレスポンス shape は同じ（`{url, sha256, content_type, byte_size, cid, ipfs_gateway_url}`）
なので、提供側スクリプトは変更不要のまま backend だけ差し替わります。

## 9. 案 C：ローカル kubo IPFS daemon で分散配信

案 B では publisher が単一インスタンスで blob を配信しました。案 C では
**コンテンツアドレシング（CID）**で同じ blob を IPFS network 上に置きます。
受信者は CID を **任意の IPFS gateway**（publisher 内蔵の `/ipfs/<cid>`、
公開 `https://ipfs.io/ipfs/<cid>` など）で取得できるので、publisher が落ちても
データは消えません。

### 9.1 kubo を一緒に起動する

`docker-compose.yml` の `ipfs` profile を有効化するだけです：

```bash
cd ~/program/Blockchain_IoT_Marketplace
export IPFS_API_URL=http://ipfs:5001
export IPFS_GATEWAY_URL=http://ipfs:8080

docker compose -f infra/docker-compose.yml --profile ipfs up -d --force-recreate publisher ipfs
docker compose -f infra/docker-compose.yml ps ipfs
# iw3ip-ipfs container が Up になっていれば OK
```

publisher 側の起動ログに `IPFS_API_URL=http://ipfs:5001` が反映されたことを確認：

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer >/dev/null
docker compose -f infra/docker-compose.yml exec publisher \
  python -c "from publisher.app.config import Settings; \
             print('IPFS_API_URL=', Settings().ipfs_api_url); \
             print('IPFS_GATEWAY_URL=', Settings().ipfs_gateway_url)"
```

### 9.2 アップロード時に CID が返ることを確認

`provider_with_media.py` のレスポンスに `cid` と `ipfs_gateway_url` が増えます：

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080 \
  --image /tmp/stage_t_demo.jpg \
  --video /tmp/stage_t_demo.jpg
```

期待出力（`cid` フィールドが `bafy...` で始まる）：

```json
[upload] {
  "image": {
    "url": "http://192.168.68.53:8080/media/<sha256>.jpg",
    "sha256": "...",
    "content_type": "image/jpeg",
    "byte_size": 7645,
    "cid": "bafkreigb...",
    "ipfs_gateway_url": "http://192.168.68.53:8080/ipfs/bafkreigb..."
  },
  ...
}
```

provider が payload に `image_cid` / `video_cid` を自動的に乗せるので、
受信側の §3〜§7 のフローはそのまま動きます。

### 9.3 受信側：CID でも URL でも取れる

Tier 3 / 2 の `/platform/data` 応答に `image_cid` と `image_url` の両方が出ます：

```bash
curl -s -H "authorization: Bearer $TOK_GOV" \
  "http://192.168.68.53:8080/platform/data?dataset_id=home/event/possible_littering" \
  | jq '.rows[0] | {image_cid, image_url, ipfs_gateway: ("http://192.168.68.53:8080/ipfs/"+.image_cid)}'
```

iPhone Safari でいずれかを開いてください：

| 取得方法 | URL 例 |
|---|---|
| publisher の HTTP gateway | `http://192.168.68.53:8080/media/<sha256>.jpg`（案 B 互換） |
| publisher の IPFS proxy | `http://192.168.68.53:8080/ipfs/<cid>` |
| 公開 IPFS gateway | `https://ipfs.io/ipfs/<cid>` ※外部 NW があれば |

最後の **公開 IPFS gateway** は publisher が落ちていても CID で取得できる
（つまり真にコンテンツアドレシング済み）ことを示します。

### 9.4 案 C の利点と注意

- ✅ **コンテンツアドレシング済み**：CID = 中身のハッシュ。改竄不可、複数 gateway で重複取得可能
- ✅ **publisher が落ちても資産が残る**：他の IPFS ピアにレプリカがあれば取得可能
- ✅ **案 B 互換**：レスポンス shape は `cid`、`ipfs_gateway_url` が増えただけ
- ⚠️ kubo daemon が落ちると `/media/upload` のレスポンスは `cid: null` で返る（案 B 動作にフォールバック）
- ⚠️ 公開 gateway 経由の取得は IPFS network の伝播待ち（分単位）が発生することがある
- 🔜 Web3.Storage / Pinata 等の pinning service と組み合わせると永続性が上がる（フォロー TODO）

### 9.5 トラブルシュート

| 症状 | 対処 |
|---|---|
| `cid: null` がレスポンスに返る | `docker compose ... ps ipfs` で kubo container が Up か確認。落ちていたら `docker compose ... --profile ipfs up -d ipfs` |
| `/ipfs/<cid>` が 502 | publisher から `http://ipfs:8080` に到達できない。Docker network 共有を確認 |
| `/ipfs/<cid>` が 404 | `IPFS_GATEWAY_URL` が空。`.env` か `export` 設定を確認 |
| 公開 gateway で取得できない | NAT 配下の場合、kubo がピアに見えていない。`ipfs swarm peers` でピア接続を確認 |

## 次に進む

- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) — Phase 2 wallet を立ち上げる
- [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) — 設計根拠
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) — 商品化の前段
