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

## 10. PWA Viewer（スマホ + PC 共通 UX）

§3〜§9 までは「ターミナルで `/verifier/request` の URL を叩く → token を回収 →
`/platform/data` に curl」という開発者目線のフローでした。
publisher 内蔵の **PWA Viewer**（`/buyer/start` + `/viewer`）を使うと、
スマホでも PC でも **「ブラウザでページを開けばあとは自動」** という体験になります。

```
[iPhone Safari] ─ /buyer/start                            [publisher]
   │  ↓ 自動で deeplink                                       │
   │  iw3ip-wallet 起動 → Tier 3 提示 ───────────────────────► │ mint ViewerToken
   │  ↑ redirect_uri=/viewer?vt=...                            │
   │  Safari 戻る → /viewer が image/video 自動表示             │

[PC Chrome] ─ /buyer/start                                [publisher]
   │  ↓ QR 表示 + ロングポーリング                              │
   │      QR を iPhone で読む → ウォレット → Tier 3 提示 ──────► │
   │  ↑ /verifier/status から viewer_url を取得                 │
   │  PC ブラウザが /viewer に自動遷移 → 画像表示                │
```

### 10.1 起動

特別な準備は不要です。`/buyer/start?ds=<dataset_id>` を **同じ URL でスマホでも PC でも** 開けば、
ページが UA を見て挙動を切り替えます：

```
iPhone Safari:  http://192.168.68.53:8080/buyer/start?ds=home/event/possible_littering
PC Chrome:      http://192.168.68.53:8080/buyer/start?ds=home/event/possible_littering
```

### 10.2 同一デバイス（iPhone）の挙動

1. 上記 URL を Safari で開く
2. ページが内部で `/verifier/request` を叩いて deeplink を取得
3. `window.location = deeplink` で **iw3ip-wallet が自動起動**
4. ウォレットで「購入閲覧（Tier 3 / 動画まで）」を選んで提示
5. wallet が `redirect_uri=/viewer?vt=...&ds=...` を受け取る
6. **Safari に自動で戻り、画像/動画が描画される**

curl で URL をコピペする手順は **不要**になります。

### 10.3 異デバイス（PC + iPhone）の挙動

1. PC ブラウザで上記 URL を開く
2. ページに **大きな QR コード**が表示される（中身は OID4VP deeplink）
3. iPhone のカメラ or ウォレットで QR を読む → wallet が起動 → 提示
4. PC のページは裏で `/verifier/status?state=...` を 2 秒ごとにロングポーリング
5. wallet 提示が完了すると `viewer_url` がレスポンスに乗る → PC が自動遷移
6. **PC ブラウザに同じ画像/動画が描画される**

### 10.4 Viewer ページの中身

`/viewer?vt=<viewer_token>&ds=<dataset_id>` は次を表示します：

- 上部に **Tier バッジ**（`event` / `event+image` / `event+image+video`）
- `image_url` を `<img>` でインライン表示
- `video_url` を `<video controls>` で再生可能
- `image_cid` がある場合（案 C ON）はクリッカブルリンクで `/ipfs/<cid>` 経由表示
- 末尾の `details` で生レスポンス JSON を確認可能

ViewerToken の TTL（60 秒）が切れた場合は 401 と共に「再提示してください」のメッセージが出ます。

### 10.5 Tier 別の見え方（実機スクリーンショット）

#### ウォレット側：3 ティアが別カードとして並ぶ

iPhone の iw3ip-wallet（Sphereon mobile-wallet fork）に PurchaseViewerVC を 3 枚受領すると、
Tier 別の表示名で 3 つの異なるカードとして並びます。

<figure markdown>
![iw3ip-wallet credential list with 3 PurchaseViewerVC tiers](images/data-user-vc-tiered/wallet-tier-cards.png){ width=320 }
<figcaption>
ウォレットの credential 一覧（縦スクロール）。同じ VCT
<code>https://iw3ip.example/credentials/PurchaseViewerVC/v1</code>
を共有しつつ、 <code>credential_configuration_id</code>
（<code>PurchaseViewerVC.full</code> / <code>.access</code> / <code>.event</code>）の差で
3 種類の display name が出ます。
</figcaption>
</figure>

#### Viewer 側：提示したティアでレスポンスが切り替わる

iPhone（iw3ip-wallet）で end-to-end を流したときの `/viewer` 画面のスクリーンショットです。
バッジの色 + コンテンツの欠け方で、ティア境界が一目で分かります。

| Tier 3（gov / Full） | Tier 2（enterprise / Access） | Tier 1（default / Event-only） |
|---|---|---|
| ![Tier 3 viewer](images/data-user-vc-tiered/viewer-tier-3-full.png) | ![Tier 2 viewer](images/data-user-vc-tiered/viewer-tier-2-access.png) | ![Tier 1 viewer](images/data-user-vc-tiered/viewer-tier-1-event.png) |
| 🟢 `tier: event+image+video` | 🟠 `tier: event+image` | ⚪️ `tier: event` |
| 画像 + 動画プレーヤー両方 | 画像のみ、動画プレーヤーなし | テキストとタイムスタンプのみ |
| `image_cid` / `image_url` / `video_url` / `video_duration_sec` 全部 | `image_cid` / `image_url` のみ | すべての media キー欠落 |

3 枚は同じデータセット（`home/event/possible_littering`）に対して、提示する PurchaseViewerVC のティアを
変えて取得した `/platform/data` の結果です。**サーバ側のデータは同一**で、どのキーが
受信者にレスポンスされるかが ViewerToken の `allowed_views` で決まる、というのが Stage T の本質です。

### 10.6 PC + スマホ両対応のメリット

| 観点 | 案 A〜C（手動 curl） | PWA Viewer |
|---|---|---|
| スマホで簡単に確認 | ✗（URL コピペ）| ✓ |
| PC で確認 | ✗（PC にウォレットなし）| ✓（QR + ロングポーリング） |
| アプリ追加インストール | iw3ip-wallet（スマホのみ） | iw3ip-wallet のみ（PC は不要） |
| 失効後の再取得 | curl やり直し | ページリロード |
| 公開デモ | 手順説明が長い | URL を 1 つ渡すだけ |

### 10.7 トラブルシュート

| 症状 | 対処 |
|---|---|
| iPhone で deeplink が起動しない | Safari → ウォレットで開く リンクをタップ。Safari の「アプリ起動許可」を確認 |
| PC で QR が表示されない | ブラウザが CDN（`cdn.jsdelivr.net`）に到達できるか確認。オフライン環境ではエラーになる |
| PC のロングポーリングが終わらない | wallet 側で正しい VC を提示できているか `docker compose logs publisher` で `/verifier/response` の 200 を確認 |
| `/viewer` が 401 | ViewerToken TTL 60 秒切れ。`/buyer/start` から再開 |
| `/buyer/start` に「データセットが一致しません」バナー | §10.8 の deny UX を参照 |

### 10.8 deny UX（提示拒否時の振る舞い）

verifier が VC 提示を拒否すると、`/verifier/status` レスポンスに `reason` コードと
`human_message_ja` / `human_message_en` の人間可読メッセージが乗ります。
`/buyer/start` ページはロングポーリング中にこれを検出して、QR の代わりに赤バナーと
「購入画面から再提示」リンクを表示します（実装: `publisher/app/ssi/verifier_routes.py`）。

主要な reason コードと表示文（JA / EN）:

| reason | JA バナー文言 | EN |
|---|---|---|
| `dataset_mismatch` | 提示された VC のデータセットが、要求されたデータセットと一致しません。 | The presented VC is bound to a different dataset. |
| `action_not_allowed` | 提示された VC では、このデータの読み取り権限がありません。 | The presented VC does not include the required action (read). |
| `purpose_mismatch` | 提示された VC の許可目的に、今回の用途が含まれていません。 | The presented VC's allowed_purposes does not cover this purpose. |
| `missing_entityType` 等 | DataUserVC に *XXX* が含まれていません。 | DataUserVC is missing *XXX*. |
| `verification_failed` | VC の署名検証に失敗しました。 | VC verification failed. |

**期待される動作（実機未確認・Stage T 実装ベース）:**

1. PC で `/buyer/start?ds=home/event/possible_littering` を開く → QR が出る
2. iPhone のウォレットで、**異なる dataset にバインドされた** PurchaseViewerVC（例: `home/event/another_dataset`）をわざと選んで提示
3. publisher は `/verifier/response` を受け取り、`{"verified": false, "reason": "dataset_mismatch"}` を記録
4. PC の `/verifier/status` ロングポーリングが `status: "denied"` + `human_message_ja` を返す
5. PC ページは QR を `<div class="deny-banner">提示された VC のデータセットが、要求されたデータセットと一致しません。</div>` に置き換え、`/buyer/start?ds=...` への再提示リンクを表示
6. 同時に `docker compose logs publisher` 側で `_write_audit(action="presentation", reason="dataset_mismatch", verified="false")` の監査ログ行が出る

実機で確認する場合は §10.5 のスクリーンショット手順を流用し、Tier 3 提示時に
**dataset 引数だけ別データセットに差し替えた** `/issuer/offer?vc_kind=PurchaseViewerVC&claim_id=<別 claim>` を発行して
ウォレットに 4 枚目のカードを入れた状態で QR を読むと再現します。

## 11. PWA Provider（データ提供者向け PWA）

§10 までは「**受信側** が PWA Viewer でデータを取得する」フローでした。
§11 では対称となる「**提供側** が PWA で SSI 認証を済ませてデータを供出する」
フローを扱います。同じ publisher コンテナの `/provider/start` + `/provider`
+ `/provider/publish` だけで完結し、`provider_with_media.py` の curl 操作を
ブラウザ単体に置き換えます。

```
[PC ブラウザ] ─ /provider/start                       [publisher]
   │  ↓ QR 表示 + ロングポーリング                         │
   │      QR を iPhone で読む → ウォレット → SellerVC 提示 ─►│ mint SellerToken
   │  ↑ /verifier/status で seller_token + licensed_datasets
   │  PC ページが「許可データセット」一覧を出す
   │  ↓ 1 つ選んで「アップロードへ進む」                     │
   │  /provider?pt=<seller_token>&ds=<dataset_id>          │
   │     ファイル選択 / カメラ撮影 / ブラウザ録画 ─────────►│ /media/upload
   │     ↑ URL + (CID) を払い出し                          │
   │     「Publish」ボタン                                  │
   │     POST /provider/publish (Bearer SellerToken) ─────►│ use_seller_token
   │                                                        │ → process_message
   │  ↑ {"status":"allowed", ...} を表示                     │
```

`/provider/start` は SellerVC 用の **OID4VP ループ**で、§10 の `/buyer/start`
と同じ実装パターンです。違いは `vc_kind=SellerVC` を要求する点と、成功時に
`SellerToken` を発行する点です。

### 11.1 起動

特別な準備は不要です（`docker compose ... up -d publisher` が動いていれば OK）。
PC ブラウザで:

```
http://192.168.68.53:8080/provider/start?ds=home/event/possible_littering
```

を開きます。`ds=` は **ヒント**で、SellerVC の検証は dataset スコープではないので
省略しても動きますが、画面ヘッダに表示されるので付けておくと作業が分かりやすいです。

### 11.2 SellerVC を提示する（OID4VP）

PC では QR が表示されます。iPhone のウォレットで読み取って **SellerVC** を提示
してください（PurchaseViewerVC ではなく SellerVC である点に注意）。

提示が成功すると、PC ページが次の状態に切り替わります：

- 「SellerVC 提示が承認されました」の緑バナー
- **出品許可データセット一覧**（SellerVC の `licensed_datasets[]` がそのまま並ぶ）
- `seller_id` と SellerToken の有効期限（既定 24 時間）
- 1 つ選んで **「アップロードへ進む →」** ボタン

ボタンを押すと `/provider?pt=<seller_token>&ds=<選んだデータセット>` に遷移します。

!!! note "deny UX"
    SellerVC に `seller_id` や `licensed_datasets` が欠けている場合、
    `/verifier/status` から返る `human_message_ja` が画面の赤バナーに出ます
    （reason コード: `missing_seller_id` / `missing_licensed_datasets`）。
    その場で別の VC で再試行できるよう、リロードリンクが添えられています。

### 11.3 データを提供する 3 つの方法

`/provider` ページは **データ実体の提供方法を 3 つ用意**しています。
どれを選んでも `/media/upload` 経由で同じ URL/CID 配信ロジックに乗るので、
受信側の `/viewer` から見れば違いはありません。

| モード | 動作 | 推奨 |
|---|---|---|
| 📁 ファイルから選ぶ | OS のファイルピッカー。既存のファイルをそのまま選択 | PC / スマホ両用 |
| 📷 カメラで撮影 | `<input capture="environment">`。iPhone Safari ではカメラが直接起動。PC ではファイルピッカーにフォールバック | iPhone での即時撮影 |
| 🔴 ブラウザで録画 | `MediaRecorder` + `getUserMedia({video,audio})`。「録画開始」→ ライブプレビュー → 「停止」で WebM/VP9（Safari は MP4 にフォールバック）として自動アップロード | PC のウェブカメラ |

3 モードはすべて同一の `uploadBlob()` パイプラインを通り、
プレビュー → `POST /media/upload` → 結果パネル表示 → Publish 有効化、という流れは共通です。
SHA-256 で重複排除されるので、同じ動画を複数回撮ってもストレージは増えません。

!!! tip "ブラウザ録画のメリット"
    PC のウェブカメラから「**カメラ → SSI 認証 → Publish**」をブラウザ 1 画面で
    完結できるので、デモやワークショップで「来歴付きデータの提供」を 30 秒で
    見せられます。USB ウェブカメラ + MQTT のハンズオン（[USB ウェブカメラ
    イベント共有サンプル](webcam-event-sharing.md)）が「常時ストリーミング」に
    対し、こちらは「人が今この瞬間に撮って来歴を付ける」ユースケースです。

録画で停止を押すと `video_duration_sec` フォームに **実測秒数が自動入力**される
ので、手動で値を合わせる必要はありません。

### 11.4 イベント発行と licensed_datasets ゲート

アップロードが完了すると Publish ボタンが有効になります。フォームには:

- `topic` — 既定で `homeassistant/event/<dataset の末尾セグメント>` が入る
- `purpose` — 既定 `community_cleaning`
- `camera_id`、`video_duration_sec`
- `extra payload keys`（任意の JSON マージ）

を入れて **Publish event** を押します。ブラウザは `Authorization: Bearer
<seller_token>` を付けて `/provider/publish` に POST します。

サーバー側では:

1. `schemas.normalize(topic, payload)` で **dataset_id を topic から導出**
2. `ssi_state.use_seller_token(token, dataset_id=<resolved>)` で
   **`licensed_datasets[]` に `dataset_id` が含まれること**を検証
3. パスしたら `processor.process_message` を呼ぶ（`/simulate/publish` と同じ経路）
4. レスポンスに `seller_token_jti` と `register_count` を上乗せして返す

`SellerToken` は **multi-use**（`/marketplace/register` と同じ仕様）なので、
**1 回の SellerVC 提示で複数のイベントを連続発行できます**。`register_count` が
増えていくのを画面下部の details パネルで確認できます。

deny ケースの挙動:

| 失敗パターン | HTTP | reason |
|---|---|---|
| `Authorization` ヘッダなし | 401 | `missing_authorization_header` |
| 不明な SellerToken | 401 | `seller_token_unknown` |
| TTL 切れ SellerToken | 401 | `seller_token_expired` |
| `topic` から resolve した dataset が `licensed_datasets[]` に無い | 403 | `seller_token_dataset_not_licensed` |
| `topic` が `normalize()` で受け付けられない | 400 | `unsupported_topic:...` |

すべて `/audit/logs` に `action=deny` で記録されます。

### 11.5 受信側で確認

`/provider` ページの末尾には、対象データセットの `/buyer/start` への
リンクが自動生成されています。別タブで開いて Tier 3 の PurchaseViewerVC を
提示すると、たった今 Publish した画像 / 動画が `/viewer` に表示されます
（§10.5 のスクリーンショットと同じ画面）。

提供 → 認証 → 発行 → 受信 → 検証 のループを **ブラウザ 2 タブで完結**できます。

!!! success "実機検証済み (2026-04-30)"
    iPhone Safari で撮影 → 提供 → Publish した **`video/quicktime`** の動画を、
    macOS Safari の `/viewer` で **`<video>` タグ経由でインライン再生**できることを
    確認しました（§11.8.A 参照）。同じ wallet が SellerVC（提供側）と
    PurchaseViewerVC.full（受信側）の両方を保持し、提示先のページで自動的に
    使い分けられます。スクリーンショット:
    `images/data-user-vc-tiered/provider/A-macsafari-viewer-tier3.jpg`
    （差分パッチで配置予定）。

### 11.6 `/provider/start` と `/buyer/start` の対応関係

| 観点 | `/buyer/start`（§10） | `/provider/start`（§11） |
|---|---|---|
| 提示する VC | PurchaseViewerVC | **SellerVC** |
| 検証成功で発行 | ViewerToken (TTL 60s) | **SellerToken** (TTL 24h) |
| 用途 | `/platform/data` の Bearer | **`/provider/publish` の Bearer** |
| 単発 / 複数 | multi-use（読み取りは継続的） | **multi-use**（1 回の認証で複数 publish） |
| dataset スコープ | VC が dataset_id にバインド | SellerVC は `licensed_datasets[]` の集合 |
| deny メッセージ | §10.7 共通形 | 同じ `human_message_ja/en` 形式 |

### 11.7 トラブルシュート

| 症状 | 対処 |
|---|---|
| `/provider/start` で「SellerVC のオファーを受け取っていない」 | 先に `POST /issuer/offer?vc_kind=SellerVC&seller_id=...&licensed_datasets=...` でウォレットに SellerVC を入れる必要があります |
| iPhone Safari で「📷 カメラで撮影」がファイルピッカーになる | iOS のバージョンによっては `accept="image/*,video/*" capture="environment"` が組み合わせで効かないことがあります。`accept="video/*"` にすると確実にカメラが直起動。 |
| 「🔴 ブラウザで録画」のボタンが反応しない | `getUserMedia` は HTTPS / `localhost` でしか動きません。LAN の IP（`http://192.168.x.x`）で開いた場合はブラウザがマイク/カメラ権限を拒否します。`localhost:8080` か HTTPS 経由でアクセスしてください |
| `/provider/publish` が 403 `seller_token_dataset_not_licensed` | SellerVC の `licensed_datasets[]` に `topic` から導出される dataset_id が含まれていません。例: `topic=homeassistant/event/possible_littering` → dataset_id は `home/event/possible_littering` |
| Publish レスポンスに `status: send_error` | publisher の `PLATFORM_API_URL` が到達不能。SellerToken のゲートは通っており、`register_count` も上がっているはずです（処理は許可、配信が失敗）|
| `/provider/publish` が 401 `seller_token_unknown` | `SSIStateStore` はインメモリのため publisher container を再起動すると全 token が wipe されます。`/provider/start` から再度 SellerVC を提示すれば新しい SellerToken で復旧します（SellerVC 自体は wallet に残っているので追加発行は不要）。ハンズオンで container を再起動する場合はこの再提示を 1 回挟みます |
| ウォレットで「Retrieving access token failed: 400 / Error Screen」 | OID4VCI offer は **single-use**。Sphereon 系 wallet は内部で `/issuer/token` をリトライすることがあり、2 回目の呼び出しが `invalid_grant` で 400 を返します。**1 回目の呼び出しでは VC は wallet に既に発行済み**なので、エラー画面を Dismiss してウォレットの credential 一覧を確認してください。新しいカードが追加されているはずです |

### 11.8 実機検証ログ

§11.3 の 3 入力モードについて、4 環境で end-to-end を回した結果をここに残します。
スクリーンショットは `docs/hands-on/images/data-user-vc-tiered/provider/`
配下に置いてあります。

| シナリオ | 環境 | 状態 | 観測値 |
|---|---|---|---|
| **A** iPhone カメラ撮影（`capture="environment"`） | iPhone Safari (iOS 18.x) | ✅ 検証済 (2026-04-30) | upload `video/quicktime` 273KB → Publish `status=allowed` → 受信側 `/viewer` で `video/quicktime` がインライン再生（macOS Safari） |
| **B** PC ブラウザ録画（MediaRecorder） | macOS Chrome (latest) | ⏳ 未検証 | codec: 期待値 `video/webm;codecs=vp9,opus` |
| **C** Firefox VP8 フォールバック | macOS Firefox (latest) | ⏳ 未検証 | codec: 期待値 `video/webm;codecs=vp8,opus` または `video/webm` |
| **D** macOS Safari MP4 フォールバック | macOS Safari (latest) | ⏳ 未検証 | codec: 期待値 `video/mp4` |

#### 実機検証で見つかったバグ（A シナリオ初回試行）

A の検証 1 回目で **実装側のバグ 2 件 + 運用上の挙動 2 件**が顕在化しました。
ユニットテストでは検出できないタイプ（実機 UA / iPhone 固有のファイル形式 /
インメモリ state / wallet 側のリトライ挙動）で、実機検証の本来の価値を示す
具体例です。修正済みの 2 件は最新 main で解消されています。

| # | 症状 | 原因 | 修正 / 対応 |
|---|---|---|---|
| 1 | `/provider/start` で `HTTP 404 no_presentation_definition_for_dataset` | c1 の page-side JS が `dataset_id=<hint>` を `/verifier/request` に渡していた。verifier は SellerVC を `*` 配下にしか登録していないため lookup miss | [Blockchain_IoT_Marketplace#41](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/41) で `dataset_id="*"` 固定に修正済み |
| 2 | アップロード時に `HTTP 415 unsupported media type` | iPhone Safari の `<input capture>` は QuickTime `.MOV`（`video/quicktime`）で保存するが、`media_routes._ALLOWED_EXT` に含まれていなかった | [Blockchain_IoT_Marketplace#42](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/42) で `.mov` 追加 |
| 3 | Publish 直前に `HTTP 401 seller_token_unknown` | `SSIStateStore` がインメモリのため container 再ビルドで全 token wipe。`/provider/start` から OID4VP し直せば復旧 | 仕様（運用上の特性）。ハンズオンでは「container 起動後に SellerVC 提示」が前提だと §11.7 で案内 |
| 4 | wallet 側に「Retrieving access token failed: 400」エラー画面 | OID4VCI offer は **single-use** だが Sphereon wallet が `/issuer/token` をリトライし、2 回目が `invalid_grant` で 400。実際には 1 回目で VC は wallet に入っている | wallet 側の挙動。dismiss して wallet の credential 一覧を見れば PurchaseViewerVC.full が入っている。§11.7 に案内追加候補 |

各シナリオの再現手順と確認ポイントは下記の通りです。検証する人は各サブセクションを
通しで実施し、`status` 列を ✅ に更新したうえで観測値とスクリーンショットを
このページに追記します（差分パッチ歓迎）。

#### 共通の前提

すべてのシナリオで:

1. `docker compose -f infra/docker-compose.yml up -d publisher hardhat bridge mosquitto` が稼働
2. publisher の `licensed_datasets` に `home/event/possible_littering` を含む SellerVC を一枚 wallet に持っている
3. publisher のホストは PC からも iPhone からも到達可能（同じ LAN 推奨）

「ブラウザで録画」モード (B/C/D) は `getUserMedia` の制約上 **`localhost`
または HTTPS でしか動かない**点に注意。LAN IP (`http://192.168.x.x:8080`) で
開いた場合はブラウザがマイク/カメラ権限を拒否するため、PC ブラウザは必ず
**publisher と同じマシンで `http://localhost:8080`** で開いてください。

#### A. iPhone カメラ撮影（`capture="environment"`）

**目的**: `<input type="file" accept="image/*,video/*" capture="environment">`
が iOS Safari でカメラを直接起動することを確認する。

**手順**:

1. iPhone Safari で `http://<publisher-host>:8080/provider/start?ds=home/event/possible_littering` を開く
2. ウォレットで SellerVC を提示 → `/provider?pt=...&ds=...` に遷移
3. **「📷 カメラで撮影」** の input をタップ
4. iOS のシートで「ビデオを撮影」が **デフォルト**で出るか、もしくは Safari が
   そのままカメラに遷移するかを確認
5. 5〜10 秒の動画を撮影 → 戻る → 自動的に `/media/upload` に POST される
6. 「Publish event」を押して 200 が返ることを確認

**実機での観測 (2026-04-30, iPhone Safari, iOS 18.x)**:

- [x] 「📷 カメラで撮影」をタップ → 「ファイルを選択」 → iOS シートで「ビデオを撮影」を選択 → カメラ起動 ✅
  - **注意**: シートに「写真を撮る / ビデオを撮影 / フォトライブラリ / ファイルを選択」の選択肢が並ぶ（カメラ直起動ではない）。`accept="image/*,video/*"` + `capture` は「カメラ優先」というヒントに留まる仕様
- [x] アップロード結果: `content_type: video/quicktime` / `byte_size: 273897` / `sha256=11367cf4cb1b...` ✅
  - 期待は `video/mp4` だったが、iPhone Safari が録画を **QuickTime (`.MOV`)** で保存することが判明
- [x] Publish レスポンス: `status: allowed`、`dataset_id=home/event/possible_littering`、`seller_token_jti=49bf45c467a65ddc`、`register_count=1` ✅
- [x] 受信側 `/viewer` (macOS Safari, Tier 3 PurchaseViewerVC.full): 緑バッジ `tier: event+image+video` + `<video>` タグでインライン再生成功 ✅
  - **macOS Safari は `video/quicktime` をネイティブ再生できる**（重要なデータポイント — Stage T 案 B が QuickTime を扱える証明）

**スクリーンショット (差分パッチで配置予定)**:

```
images/data-user-vc-tiered/provider/A-iphone-404-original.png       # 修正前: 404 no_presentation_definition_for_dataset
images/data-user-vc-tiered/provider/A-iphone-415-mov-rejected.png   # 修正前: 415 .MOV unsupported
images/data-user-vc-tiered/provider/A-iphone-401-token-unknown.png  # 運用上: container 再ビルドで token wipe
images/data-user-vc-tiered/provider/A-iphone-after-publish.png      # 成功: Published 緑バナー + register_count=1
images/data-user-vc-tiered/provider/A-macsafari-viewer-tier3.jpg    # 受信側: /viewer で .MOV インライン再生
```

**iOS バージョン依存の注意**: iOS 18.x では `accept="image/*,video/*"` +
`capture="environment"` の組み合わせで **カメラ直起動にはならず**、
「ビデオを撮影 / 写真ライブラリ / ファイルを選択」の選択肢が並ぶシートが出ます。
カメラ直起動を強制したい場合は `accept="video/*"` のみに narrow する必要があり、
ファイル選択の柔軟性とのトレードオフです。今回は併存設計のまま採用（操作回数 +1
の代わりに既存ファイル選択も可能）。

#### B. PC ブラウザ録画 — Chrome（VP9）

**目的**: MediaRecorder + `getUserMedia` 経路で PC ウェブカメラから直接録画
→ アップロード → Publish が動くこと、Chromium 系では VP9 が選択されることを
確認する。

**手順**:

1. 同じ PC で publisher を起動した状態で、Chrome で `http://localhost:8080/provider/start?ds=home/event/possible_littering` を開く
2. iPhone wallet で SellerVC を提示（QR を読む / 同 LAN にいる前提）
3. 成功パネルから dataset を選んで `/provider` に遷移
4. **「🔴 ブラウザで録画」** セクションの「録画開始」を押す
5. ブラウザのカメラ/マイク権限ダイアログで **許可**
6. ライブプレビュー `<video>` にウェブカメラ映像が出ることを確認
7. 5〜10 秒待って「停止 & アップロード」
8. `recStatus` が「録画完了 (Ns) — アップロード中…」→「録画完了 (Ns)」と遷移
9. アップロード結果に `content_type: video/webm` が出るのを確認
10. `video_duration_sec` フォームに **実測秒数が自動入力**されているか確認
11. Publish 成功

**確認ポイント (codec 確認)**:

DevTools コンソールで:

```js
['video/webm;codecs=vp9,opus','video/webm;codecs=vp8,opus','video/webm','video/mp4']
  .filter(m => MediaRecorder.isTypeSupported(m))
```

→ 期待値: `["video/webm;codecs=vp9,opus", "video/webm;codecs=vp8,opus", "video/webm"]`
（先頭 = 実際に使われるもの）

**スクリーンショット (TBD)**:

```
images/data-user-vc-tiered/provider/B-chrome-permission.png         # カメラ権限ダイアログ
images/data-user-vc-tiered/provider/B-chrome-recording.png          # 録画中（赤バナー + プレビュー）
images/data-user-vc-tiered/provider/B-chrome-uploaded.png           # 録画完了 + URL/CID
images/data-user-vc-tiered/provider/B-chrome-publish-ok.png         # Publish 成功
images/data-user-vc-tiered/provider/B-chrome-devtools-codec.png     # DevTools の codec 判定
```

#### C. PC ブラウザ録画 — Firefox（VP8 フォールバック）

**目的**: Firefox でも録画動線が成立すること、VP9 が無い場合に VP8 へ
フォールバックすることを確認する。

**手順**: B と同じ手順を Firefox で実施。

**確認ポイント**:

DevTools の同じスニペットで MediaRecorder.isTypeSupported を叩き、
**`vp9,opus` が `false` なのに `vp8,opus` または `webm` が `true`** であることを
確認。アップロードされたファイルの `content_type` も `video/webm` になっていれば成功。

**スクリーンショット (TBD)**:

```
images/data-user-vc-tiered/provider/C-firefox-permission.png
images/data-user-vc-tiered/provider/C-firefox-recording.png
images/data-user-vc-tiered/provider/C-firefox-publish-ok.png
images/data-user-vc-tiered/provider/C-firefox-devtools-codec.png    # vp9=false, vp8=true
```

#### D. macOS Safari（MP4 フォールバック）

**目的**: WebM 系がサポートされない Safari で MP4 にフォールバックすること
を確認する。

**手順**: B と同じ手順を macOS Safari (16+) で実施。

**確認ポイント**:

`MediaRecorder.isTypeSupported('video/webm;codecs=vp9,opus')` が `false`、
`MediaRecorder.isTypeSupported('video/mp4')` が `true` になり、アップロード結果の
`content_type` が `video/mp4` になることを確認。

**Safari 固有の注意**:

- Safari 14.1 未満では `MediaRecorder` 自体が無く、`recStatus` が「このブラウザは
  録画に対応していません」になる。その場合は §11.7 のトラブルシュートに該当バージョン
  情報を追記してください。
- マイク/カメラ権限は **アドレスバー左の Safari 設定アイコン**から付与する必要が
  あることがある。

**スクリーンショット (TBD)**:

```
images/data-user-vc-tiered/provider/D-safari-permission.png
images/data-user-vc-tiered/provider/D-safari-recording.png
images/data-user-vc-tiered/provider/D-safari-publish-ok.png
images/data-user-vc-tiered/provider/D-safari-devtools-codec.png     # mp4=true
```

#### 検証完了後のチェックリスト

すべてのシナリオで:

- [ ] スクリーンショットを `docs/hands-on/images/data-user-vc-tiered/provider/`
      に上記のパスで配置
- [ ] §11.8 冒頭のステータステーブルを ⏳ → ✅ に更新
- [ ] 実機で観測した codec 値・OS バージョン・特異な挙動を該当サブセクションに追記
- [ ] §11.7 トラブルシュートに、検証中に発見した新しい症状があれば追記

## 次に進む

- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) — Phase 2 wallet を立ち上げる
- [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) — 設計根拠
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) — 商品化の前段
