# DataUserVC × 段階アクセス（Phase 2 拡張）

受信者の信頼度 (DataUserVC) に応じて、見せる中身を Tier 3 / Tier 2 / Tier 1 の 3 段階に切り替えます。後半（§12〜§13）では、映像を意味的中間表現に変換し、受信者の信頼度に応じて出力を変えるレンダリング（trust-aware rendering）までを扱います。全体で長めのハンズオンです。

> **やること**: 信頼度ごとに動画 / 画像 / 要約だけを出し分ける動作の確認
>
> **前提**: [HA SSI Wallet](ha-ssi-wallet.md) と [USB ウェブカメライベント共有](webcam-event-sharing.md)
>
> **使うもの**: PC + スマホ (iw3ip-wallet)
>
> **所要時間**: 90 分くらい

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

本ページのコマンド例は、PC の LAN IP を `192.168.68.53`、教材リポジトリを
`~/program/Blockchain_IoT_Marketplace` に clone した前提で示します。自分の環境の値に
読み替えてください。

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
**案 B の実データ統合は §8**、**案 C の IPFS 統合は §9** で扱います。
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

## 3. /marketplace/claim を 3 通り呼び出す

`merchandise_id` には、[USBウェブカメライベント共有サンプル](webcam-event-sharing.md) で
出品したものを使います。あらかじめ 1 件出品しておき、コマンド例の `M-0001` を
その `merchandise_id` に読み替えてください。

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

3a のコマンドの `data_user_attrs` で、`entityType` を `Enterprise`、`purpose` を
`Research` に変えて実行します。期待する結果は次のとおりです。

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
PurchaseViewerVC を発行し、ウォレットで取得してから提示します。

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

## 8. 実データ統合（案 B：HTTP メディア・ゲートウェイ）

§7 までで、ティアごとに `image_cid` / `video_cid` のキーが応答に含まれるかどうかが
変わることを確認しました。ここでは**画像 / 動画の実体**も含めて、提供から受信までを
通して実行します。提供側でダミーの JPEG / MP4 を生成し、publisher の `/media/upload` に
POST して、返ってきた URL を payload の `image_url` / `video_url` に入れます。

### 8.1 提供側スクリプトを実行する

```bash
cd ~/program/Blockchain_IoT_Marketplace
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080
```

このスクリプトは次の 3 ステップを順に実行します。

1. 1×1 JPEG / MP4 fixture を生成（`fixtures/` 配下）
2. `POST /media/upload` で 2 つアップロード（sha256 が同じファイルは重複保存しない）
3. `image_url` / `video_url` 付きイベントを `/simulate/publish` に送る

スクリプト出力に `image_url` / `video_url` の `http://192.168.68.53:8080/media/...`
形式の URL が表示されます。

### 8.2 受信側（既存の Tier 3 / 2 / 1 フロー）

§3〜§7 の流れをそのまま使います。Tier 別に `/platform/data` を取得すると、次のようになります。

- **Tier 3 (gov full)** → `event` + `image_url` + `video_url` + `video_duration_sec` 全部
- **Tier 2 (enterprise)** → `event` + `image_url` のみ。`video_url` キーは欠落
- **Tier 1 (default)** → `event` のみ

iPhone Safari で `image_url` をタップすると、1×1 JPEG が表示されます。
自分で用意した画像 / 動画を使う場合は、次のように指定します。

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080 \
  --image /path/to/snapshot.jpg \
  --video /path/to/clip.mp4 \
  --video-duration-sec 12
```

### 8.3 案 B の制限

- `image_url` / `video_url` は、`image_cid` / `video_cid`（案 A 用）と同じ対応表で投影されるので、併用できます
- コンテンツアドレシング（内容のハッシュ値をアドレスにする方式）ではありません。URL を知っていれば誰でも GET できます
- publisher は単一インスタンスで、レプリカや pinning（IPFS 上でデータを保持し続ける設定）はありません

コンテンツアドレシングと分散保存が必要な場合は **案 C（IPFS）** に
進みます。`/media/upload` のレスポンスの形式は同じ（`{url, sha256, content_type, byte_size, cid, ipfs_gateway_url}`）
なので、提供側スクリプトは変更せず、保存先の backend だけが切り替わります。

## 9. 案 C：ローカル kubo IPFS daemon で分散配信

案 B では、単一の publisher インスタンスが blob（画像 / 動画のバイト列）を配信しました。
案 C では、同じ blob を**コンテンツアドレシング（CID）**で IPFS ネットワーク上に置きます。
IPFS のノードには kubo（IPFS の Go 実装）を使います。
受信者は CID を使って**任意の IPFS gateway**（publisher 内蔵の `/ipfs/<cid>`、
公開 `https://ipfs.io/ipfs/<cid>` など）から取得できるので、publisher が停止しても
データは失われません。

### 9.1 kubo を一緒に起動する

`docker-compose.yml` の `ipfs` profile を有効にして起動します。

```bash
cd ~/program/Blockchain_IoT_Marketplace
export IPFS_API_URL=http://ipfs:5001
export IPFS_GATEWAY_URL=http://ipfs:8080

docker compose -f infra/docker-compose.yml --profile ipfs up -d --force-recreate publisher ipfs
docker compose -f infra/docker-compose.yml ps ipfs
# iw3ip-ipfs container が Up になっていれば OK
```

publisher の設定に `IPFS_API_URL=http://ipfs:5001` が反映されたことを確認します。

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer >/dev/null
docker compose -f infra/docker-compose.yml exec publisher \
  python -c "from publisher.app.config import Settings; \
             print('IPFS_API_URL=', Settings().ipfs_api_url); \
             print('IPFS_GATEWAY_URL=', Settings().ipfs_gateway_url)"
```

### 9.2 アップロード時に CID が返ることを確認

`provider_with_media.py` のレスポンスに `cid` と `ipfs_gateway_url` が増えます。
`/tmp/stage_t_demo.jpg` は例なので、手元のファイルのパスに読み替えてください。

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://192.168.68.53:8080 \
  --image /tmp/stage_t_demo.jpg \
  --video /tmp/stage_t_demo.jpg
```

期待する出力は次のとおりです（`cid` フィールドが `bafk...` で始まります）。

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

provider が payload に `image_cid` / `video_cid` を自動的に入れるので、
受信側の §3〜§7 のフローはそのまま動きます。

### 9.3 受信側：CID でも URL でも取れる

Tier 3 / 2 の `/platform/data` 応答に `image_cid` と `image_url` の両方が出ます。

```bash
curl -s -H "authorization: Bearer $TOK_GOV" \
  "http://192.168.68.53:8080/platform/data?dataset_id=home/event/possible_littering" \
  | jq '.rows[0] | {image_cid, image_url, ipfs_gateway: ("http://192.168.68.53:8080/ipfs/"+.image_cid)}'
```

iPhone Safari でいずれかを開いてください。

| 取得方法 | URL 例 |
|---|---|
| publisher の HTTP gateway | `http://192.168.68.53:8080/media/<sha256>.jpg`（案 B 互換） |
| publisher の IPFS proxy | `http://192.168.68.53:8080/ipfs/<cid>` |
| 公開 IPFS gateway | `https://ipfs.io/ipfs/<cid>`（インターネット接続が必要） |

最後の**公開 IPFS gateway** で取得できれば、publisher が停止していても CID だけで
データを取得できること（コンテンツアドレシングが機能していること）を確認できます。

### 9.4 案 C の利点と注意

利点:

- **コンテンツアドレシング**: CID は中身のハッシュです。内容を改竄すると CID が変わるので検出でき、複数の gateway から同じデータを取得できます
- **publisher が停止してもデータが残る**: 他の IPFS ピアにレプリカがあれば取得できます
- **案 B と互換**: レスポンスには `cid` と `ipfs_gateway_url` が増えるだけです

注意:

- kubo daemon が停止していると、`/media/upload` のレスポンスは `cid: null` になります（案 B と同じ動作にフォールバックします）
- 公開 gateway 経由の取得では、IPFS ネットワークへの伝播を待つ時間（分単位）がかかることがあります
- Web3.Storage / Pinata などの pinning service と組み合わせると永続性が上がります（未対応で、今後の課題です）

### 9.5 トラブルシュート

| 症状 | 対処 |
|---|---|
| `cid: null` がレスポンスに返る | `docker compose ... ps ipfs` で kubo container が Up か確認。停止していたら `docker compose ... --profile ipfs up -d ipfs` で起動する |
| `/ipfs/<cid>` が 502 | publisher から `http://ipfs:8080` に到達できない。Docker network 共有を確認 |
| `/ipfs/<cid>` が 404 | `IPFS_GATEWAY_URL` が空。`.env` か `export` 設定を確認 |
| 公開 gateway で取得できない | NAT 配下の場合、kubo がピアに見えていない。`ipfs swarm peers` でピア接続を確認 |

## 10. PWA Viewer（スマホ + PC 共通 UX）

§3〜§9 では、ターミナルから `/verifier/request` を呼び出して token を取得し、
`/platform/data` に curl でアクセスするという、開発者向けの手順を使いました。
publisher 内蔵の **PWA Viewer**（`/buyer/start` + `/viewer`。PWA は Progressive Web App の略）を使うと、
スマホでも PC でも、ブラウザでページを開くだけで提示から表示までが自動で進みます。

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

特別な準備は不要です。`/buyer/start?ds=<dataset_id>` を **同じ URL でスマホでも PC でも** 開くと、
ページが User-Agent（ブラウザの種類を示す情報）を見て動作を切り替えます。

```
iPhone Safari:  http://192.168.68.53:8080/buyer/start?ds=home/event/possible_littering
PC Chrome:      http://192.168.68.53:8080/buyer/start?ds=home/event/possible_littering
```

### 10.2 同一デバイス（iPhone）の挙動

1. 上記 URL を Safari で開く
2. ページが内部で `/verifier/request` を呼び出して deeplink を取得
3. `window.location = deeplink` で **iw3ip-wallet が自動起動**
4. ウォレットで「購入閲覧（Tier 3 / 動画まで）」を選んで提示
5. ウォレットが `redirect_uri=/viewer?vt=...&ds=...` を受け取る
6. **Safari に自動で戻り、画像/動画が描画される**

curl で URL をコピーして貼り付ける手順は不要です。

### 10.3 異デバイス（PC + iPhone）の挙動

1. PC ブラウザで上記 URL を開く
2. ページに **大きな QR コード**が表示される（中身は OID4VP deeplink）
3. iPhone のカメラかウォレットで QR を読むと、ウォレットが起動するので VC を提示する
4. PC のページは裏で `/verifier/status?state=...` を 2 秒ごとにロングポーリング
5. ウォレットでの提示が完了するとレスポンスに `viewer_url` が含まれ、PC のページが自動で遷移する
6. **PC ブラウザに同じ画像/動画が描画される**

### 10.4 Viewer ページの中身

`/viewer?vt=<viewer_token>&ds=<dataset_id>` は次を表示します。

- 上部に **Tier バッジ**（`event` / `event+image` / `event+image+video`）
- `image_url` を `<img>` でインライン表示
- `video_url` を `<video controls>` で再生可能
- `image_cid` がある場合（案 C が有効な場合）は、`/ipfs/<cid>` へのリンクを表示
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

iPhone（iw3ip-wallet）で一連の流れを実行したときの `/viewer` 画面のスクリーンショットです。
バッジの色と、表示されるコンテンツの違いから、どのティアで取得したかが分かります。

| Tier 3（gov / Full） | Tier 2（enterprise / Access） | Tier 1（default / Event-only） |
|---|---|---|
| ![Tier 3 viewer](images/data-user-vc-tiered/viewer-tier-3-full.png) | ![Tier 2 viewer](images/data-user-vc-tiered/viewer-tier-2-access.png) | ![Tier 1 viewer](images/data-user-vc-tiered/viewer-tier-1-event.png) |
| 緑のバッジ `tier: event+image+video` | オレンジのバッジ `tier: event+image` | 灰色のバッジ `tier: event` |
| 画像 + 動画プレーヤー両方 | 画像のみ、動画プレーヤーなし | テキストとタイムスタンプのみ |
| `image_cid` / `image_url` / `video_url` / `video_duration_sec` 全部 | `image_cid` / `image_url` のみ | すべての media キー欠落 |

3 枚は同じデータセット（`home/event/possible_littering`）に対して、提示する PurchaseViewerVC のティアを
変えて取得した `/platform/data` の結果です。**サーバ側のデータは同一**で、受信者に返す
キーは ViewerToken の `allowed_views` で決まります。

### 10.6 PC + スマホ両対応のメリット

| 観点 | 案 A〜C（手動 curl） | PWA Viewer |
|---|---|---|
| スマホで簡単に確認 | ✗（URL をコピーして貼り付け）| ✓ |
| PC で確認 | ✗（PC にウォレットなし）| ✓（QR + ロングポーリング） |
| アプリ追加インストール | iw3ip-wallet（スマホのみ） | iw3ip-wallet のみ（PC は不要） |
| 失効後の再取得 | curl やり直し | ページリロード |
| 公開デモ | 手順説明が長い | URL を 1 つ渡すだけ |

### 10.7 トラブルシュート

| 症状 | 対処 |
|---|---|
| iPhone で deeplink が起動しない | Safari → ウォレットで開く リンクをタップ。Safari の「アプリ起動許可」を確認 |
| PC で QR が表示されない | ブラウザが CDN（`cdn.jsdelivr.net`）に到達できるか確認。オフライン環境ではエラーになる |
| PC のロングポーリングが終わらない | ウォレット側で正しい VC を提示できているか `docker compose logs publisher` で `/verifier/response` の 200 を確認 |
| `/viewer` が 401 | ViewerToken TTL 60 秒切れ。`/buyer/start` から再開 |
| `/buyer/start` に「データセットが一致しません」バナー | §10.8 の deny UX を参照 |

### 10.8 deny UX（提示拒否時の振る舞い）

verifier が VC 提示を拒否すると、`/verifier/status` レスポンスに `reason` コードと
`human_message_ja` / `human_message_en`（人が読むためのメッセージ）が含まれます。
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

**期待される動作（実機では未確認で、実装から読み取った内容です）:**

1. PC で `/buyer/start?ds=home/event/possible_littering` を開く → QR が出る
2. iPhone のウォレットで、**異なる dataset にバインドされた** PurchaseViewerVC（例: `home/event/another_dataset`）をわざと選んで提示
3. publisher は `/verifier/response` を受け取り、`{"verified": false, "reason": "dataset_mismatch"}` を記録
4. PC の `/verifier/status` ロングポーリングが `status: "denied"` + `human_message_ja` を返す
5. PC ページは QR を `<div class="deny-banner">提示された VC のデータセットが、要求されたデータセットと一致しません。</div>` に置き換え、`/buyer/start?ds=...` への再提示リンクを表示
6. 同時に `docker compose logs publisher` 側で `_write_audit(action="presentation", reason="dataset_mismatch", verified="false")` の監査ログ行が出る

実機で確認する場合は、§10.5 と同じ環境で、**別のデータセットに対する claim** から
`/issuer/offer?vc_kind=PurchaseViewerVC&claim_id=<別 claim>` を発行し、ウォレットに 4 枚目の
カードを入れます。その状態で QR を読み、4 枚目のカードを提示すると再現します。

## 11. PWA Provider（データ提供者向け PWA）

§10 までは、**受信側**が PWA Viewer でデータを取得するフローでした。
§11 では、**提供側**が PWA 上で SSI 認証を済ませてデータを提供するフローを
扱います。同じ publisher コンテナの `/provider/start`、`/provider`、
`/provider/publish` だけで動作し、§8 の `provider_with_media.py` で行った操作を
ブラウザだけで実行できます。

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

`/provider/start` は SellerVC を提示する **OID4VP のフロー**で、§10 の `/buyer/start`
と同じ実装パターンです。違いは `vc_kind=SellerVC` を要求する点と、成功時に
`SellerToken` を発行する点です。

### 11.1 起動

特別な準備は不要です（`docker compose ... up -d publisher` で publisher が動いていれば使えます）。
PC ブラウザで次の URL を開きます。

```
http://192.168.68.53:8080/provider/start?ds=home/event/possible_littering
```

`ds=` は表示用の**ヒント**です。SellerVC の検証はデータセット単位ではないので
省略しても動きますが、画面ヘッダに表示されるので、付けておくと作業中のデータセットが分かります。

### 11.2 SellerVC を提示する（OID4VP）

PC では QR が表示されます。iPhone のウォレットで読み取って **SellerVC** を提示
してください。ここで提示するのは SellerVC です。PurchaseViewerVC を選ばないよう注意してください。

提示が成功すると、PC ページが次の状態に切り替わります。

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

`/provider` ページには、**データ実体を提供する方法が 3 つ**あります。
どれを選んでも `/media/upload` を経由して同じ URL/CID の配信処理を通るので、
受信側の `/viewer` での見え方は変わりません。

| モード | 動作 | 推奨 |
|---|---|---|
| 📁 ファイルから選ぶ | OS のファイルピッカー。既存のファイルをそのまま選択 | PC / スマホ両用 |
| 📷 カメラで撮影 | `<input capture="environment">`。iPhone Safari ではカメラが直接起動。PC ではファイルピッカーにフォールバック | iPhone での即時撮影 |
| 🔴 ブラウザで録画 | `MediaRecorder` + `getUserMedia({video,audio})`。「録画開始」→ ライブプレビュー → 「停止」で WebM/VP9 として自動アップロード。Firefox は VP8 にフォールバック。Safari 16 以下のみ MP4 にフォールバック（Safari 17+ は WebM/VP9 をネイティブ対応） | PC のウェブカメラ |

3 つのモードはすべて同じ `uploadBlob()` の処理を通り、
プレビュー → `POST /media/upload` → 結果パネル表示 → Publish 有効化、という流れは共通です。
SHA-256 で重複排除されるので、同じファイルを複数回アップロードしてもストレージは増えません。

!!! tip "ブラウザ録画のメリット"
    PC のウェブカメラでの撮影から SSI 認証、Publish までをブラウザ 1 画面で
    実行できるので、デモやワークショップで「来歴付きデータの提供」を 30 秒ほどで
    見せられます。USB ウェブカメラ + MQTT のハンズオン（[USB ウェブカメラ
    イベント共有サンプル](webcam-event-sharing.md)）は常時ストリーミングする構成でしたが、
    こちらは人がその場で撮影したデータに来歴を付けて提供する使い方です。

録画で停止を押すと `video_duration_sec` フォームに **実測秒数が自動入力**される
ので、手動で値を合わせる必要はありません。

### 11.4 イベント発行と licensed_datasets ゲート

アップロードが完了すると Publish ボタンが有効になります。フォームの次の項目を
確認・入力して **Publish event** を押します。

- `topic` — 既定で `homeassistant/event/<dataset の末尾セグメント>` が入る
- `purpose` — 既定 `community_cleaning`
- `camera_id`、`video_duration_sec`
- `extra payload keys`（任意の JSON マージ）

ブラウザは `Authorization: Bearer <seller_token>` を付けて `/provider/publish` に
POST します。

サーバー側では次の順に処理します。

1. `schemas.normalize(topic, payload)` で **dataset_id を topic から導出**
2. `ssi_state.use_seller_token(token, dataset_id=<resolved>)` で
   **`licensed_datasets[]` に `dataset_id` が含まれること**を検証
3. 検証に通ったら `processor.process_message` を呼ぶ（`/simulate/publish` と同じ経路）
4. レスポンスに `seller_token_jti` と `register_count` を加えて返す

`SellerToken` は有効期限内なら繰り返し使えます（**multi-use**。`/marketplace/register` と同じ仕様）。
そのため、**1 回の SellerVC 提示で複数のイベントを続けて発行できます**。`register_count` が
増えていく様子は、画面下部の details パネルで確認できます。

拒否（deny）される場合の応答は次のとおりです。

| 失敗パターン | HTTP | reason |
|---|---|---|
| `Authorization` ヘッダなし | 401 | `missing_authorization_header` |
| 不明な SellerToken | 401 | `seller_token_unknown` |
| TTL 切れ SellerToken | 401 | `seller_token_expired` |
| `topic` から導出した dataset が `licensed_datasets[]` に無い | 403 | `seller_token_dataset_not_licensed` |
| `topic` が `normalize()` で受け付けられない | 400 | `unsupported_topic:...` |

すべて `/audit/logs` に `action=deny` で記録されます。

### 11.5 受信側で確認

`/provider` ページの末尾には、対象データセットの `/buyer/start` への
リンクが自動生成されています。別タブで開いて Tier 3 の PurchaseViewerVC を
提示すると、たった今 Publish した画像 / 動画が `/viewer` に表示されます
（§10.5 のスクリーンショットと同じ画面）。

提供、認証、発行、受信、検証の一連の流れを、**ブラウザの 2 つのタブだけ**で実行できます。

!!! success "実機検証済み (2026-04-30)"
    iPhone Safari で撮影 → 提供 → Publish した **`video/quicktime`** の動画を、
    macOS Safari の `/viewer` で **`<video>` タグ経由でインライン再生**できることを
    確認しました（§11.8 のシナリオ A を参照）。同じウォレットが SellerVC（提供側）と
    PurchaseViewerVC.full（受信側）の両方を保持し、提示先のページに応じて自動的に
    使い分けられます。スクリーンショットは
    `images/data-user-vc-tiered/provider/A-macsafari-viewer-tier3.jpg` にあります。

### 11.6 `/provider/start` と `/buyer/start` の対応関係

| 観点 | `/buyer/start`（§10） | `/provider/start`（§11） |
|---|---|---|
| 提示する VC | PurchaseViewerVC | **SellerVC** |
| 検証成功で発行 | ViewerToken (TTL 60s) | **SellerToken** (TTL 24h) |
| 用途 | `/platform/data` の Bearer | **`/provider/publish` の Bearer** |
| 単発 / 複数 | multi-use（読み取りは継続的） | **multi-use**（1 回の認証で複数 publish） |
| dataset スコープ | VC が dataset_id にバインド | SellerVC は `licensed_datasets[]` の集合 |
| deny メッセージ | §10.8 の共通形式 | 同じ `human_message_ja/en` 形式 |

### 11.7 トラブルシュート

| 症状 | 対処 |
|---|---|
| `/provider/start` で「SellerVC のオファーを受け取っていない」 | 先に `POST /issuer/offer?vc_kind=SellerVC&seller_id=...&licensed_datasets=...` でウォレットに SellerVC を入れる必要があります |
| iPhone Safari で「📷 カメラで撮影」がファイルピッカーになる | iOS のバージョンによっては `accept="image/*,video/*" capture="environment"` が組み合わせで効かないことがあります。`accept="video/*"` だけにすると、カメラが直接起動します |
| 「🔴 ブラウザで録画」のボタンが反応しない | `getUserMedia` は HTTPS / `localhost` でしか動きません。LAN の IP（`http://192.168.x.x`）で開いた場合はブラウザがマイク/カメラ権限を拒否します。`localhost:8080` か HTTPS 経由でアクセスしてください |
| `/provider/publish` が 403 `seller_token_dataset_not_licensed` | SellerVC の `licensed_datasets[]` に `topic` から導出される dataset_id が含まれていません。例: `topic=homeassistant/event/possible_littering` → dataset_id は `home/event/possible_littering` |
| Publish レスポンスに `status: send_error` | publisher の `PLATFORM_API_URL` が到達不能。SellerToken のゲートは通っており、`register_count` も上がっているはずです（処理は許可、配信が失敗）|
| `/provider/publish` が 401 `seller_token_unknown` | `SSIStateStore` はインメモリのため publisher container を再起動するとすべての token が消えます。`/provider/start` から再度 SellerVC を提示すれば新しい SellerToken で復旧します（SellerVC 自体はウォレットに残っているので追加発行は不要です）。ハンズオンで container を再起動する場合はこの再提示を 1 回挟みます |
| ウォレットで「Retrieving access token failed: 400 / Error Screen」 | OID4VCI の offer は 1 回しか使えません（**single-use**）。Sphereon 系のウォレットは内部で `/issuer/token` をリトライすることがあり、2 回目の呼び出しは `invalid_grant` で 400 になります。**1 回目の呼び出しで VC はウォレットに発行済み**なので、エラー画面を Dismiss してウォレットの credential 一覧を確認してください。新しいカードが追加されています |

### 11.8 実機検証ログ

§11.3 の 3 つの入力モードについて、4 つの環境で提供から受信までを通して実行した結果を記録します。
スクリーンショットは `docs/hands-on/images/data-user-vc-tiered/provider/`
配下に置いています。

| シナリオ | 環境 | 状態 | 観測値 |
|---|---|---|---|
| **A** iPhone カメラ撮影（`capture="environment"`） | iPhone Safari (iOS 18.x) | 検証済 (2026-04-30) | upload `video/quicktime` 273KB → Publish `status=allowed` → 受信側 `/viewer` で `video/quicktime` がインライン再生（macOS Safari） |
| **B** PC ブラウザ録画（MediaRecorder） | macOS Chrome 147 | codec のみ確認済 (2026-04-30) | サポート 4 種類: `vp9,opus` / `vp8,opus` / `webm` / `mp4` → 選好順で **VP9 を選択**。録画 + Publish は wallet IP 切替後に検証 |
| **C** Firefox VP8 フォールバック | macOS Firefox 139 | **検証済 (2026-04-30)** | サポート 2 種類: `vp8,opus` / `webm` (VP9 なし — Firefox の MediaRecorder は VP9 未対応) → **VP8 を選択**。録画 → アップロード → Publish 完走、`media_uploaded ext=.webm bytes=89909` (WebM/VP8 89KB) + `POST /provider/publish 200 OK`。`pickRecorderMime()` の VP8 fallback 経路を実機で確認できました |
| **D** macOS Safari WebM/VP9（当初の想定は MP4 フォールバック） | macOS Safari 17+ | **検証済 (2026-04-30)** | サポート 4 種類: `vp9,opus` / `vp8,opus` / `webm` / `mp4` → 選好順で **VP9 を選択**（MP4 ではない）。録画 → アップロード → Publish 完走、`seller_token_issued ertl-bcd-final` + `POST /provider/publish 200 OK`。**Safari 17+ は WebM/VP9 にネイティブ対応**しているため、当初の仕様で想定した「Safari は MP4 にフォールバックする」は Safari 16 以下にだけ当てはまります |

#### 実機検証で見つかったバグ（A シナリオ初回試行）

シナリオ A の 1 回目の検証で、**実装のバグ 2 件と運用上の挙動 2 件**が見つかりました。
実機の User-Agent、iPhone 固有のファイル形式、インメモリの state、ウォレットの
リトライ動作に起因するもので、いずれもユニットテストでは検出できませんでした。
バグ 2 件は最新の main で修正済みです。

| # | 症状 | 原因 | 修正 / 対応 |
|---|---|---|---|
| 1 | `/provider/start` で `HTTP 404 no_presentation_definition_for_dataset` | `/provider/start` ページの JavaScript が `dataset_id=<hint>` を `/verifier/request` に渡していた。verifier は SellerVC を `*` 配下にしか登録していないため、検索に失敗していた | [Blockchain_IoT_Marketplace#41](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/41) で `dataset_id="*"` 固定に修正済み |
| 2 | アップロード時に `HTTP 415 unsupported media type` | iPhone Safari の `<input capture>` は QuickTime `.MOV`（`video/quicktime`）で保存するが、`media_routes._ALLOWED_EXT` に含まれていなかった | [Blockchain_IoT_Marketplace#42](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/42) で `.mov` 追加 |
| 3 | Publish 直前に `HTTP 401 seller_token_unknown` | `SSIStateStore` がインメモリのため、container を再ビルドするとすべての token が消える。`/provider/start` から SellerVC を提示し直せば復旧する | 仕様どおりの動作です。ハンズオンでは container の起動後に SellerVC を提示する必要があり、§11.7 に記載しています |
| 4 | ウォレットに「Retrieving access token failed: 400」のエラー画面が出る | OID4VCI の offer は 1 回しか使えない（**single-use**）が、Sphereon 系のウォレットが `/issuer/token` をリトライし、2 回目が `invalid_grant` で 400 になる。実際には 1 回目で VC はウォレットに入っている | ウォレット側の挙動です。エラー画面を閉じて credential 一覧を見ると PurchaseViewerVC.full が入っています。§11.7 に記載しています |

各シナリオの再現手順と確認内容は以下のとおりです。検証する人は各項を
通して実施し、上の表の「状態」列を「検証済」に更新して、観測値とスクリーンショットを
このページに追記してください。

#### 共通の前提

すべてのシナリオで次を前提とします。

1. `docker compose -f infra/docker-compose.yml up -d publisher bridge mosquitto` が稼働
2. `licensed_datasets` に `home/event/possible_littering` を含む SellerVC を 1 枚ウォレットに持っている
3. publisher のホストは PC からも iPhone からも到達可能（同じ LAN 推奨）

「ブラウザで録画」モード (B/C/D) は、`getUserMedia` の制約により **`localhost`
または HTTPS でしか動きません**。LAN IP (`http://192.168.x.x:8080`) で
開くとブラウザがマイク/カメラの権限を拒否するため、PC ブラウザでは必ず
**publisher と同じマシンで `http://localhost:8080`** を開いてください。

#### A. iPhone カメラ撮影（`capture="environment"`）

**目的**: `<input type="file" accept="image/*,video/*" capture="environment">`
が iOS Safari でカメラを直接起動するかを確認します。

**手順**:

1. iPhone Safari で `http://<publisher-host>:8080/provider/start?ds=home/event/possible_littering` を開く
2. ウォレットで SellerVC を提示 → `/provider?pt=...&ds=...` に遷移
3. **「📷 カメラで撮影」** の input をタップ
4. iOS のシートで「ビデオを撮影」が **デフォルト**で出るか、もしくは Safari が
   そのままカメラに遷移するかを確認
5. 5〜10 秒の動画を撮影 → 戻る → 自動的に `/media/upload` に POST される
6. 「Publish event」を押して 200 が返ることを確認

**実機での観測 (2026-04-30, iPhone Safari, iOS 18.x)**:

- [x] 「📷 カメラで撮影」をタップ → 「ファイルを選択」 → iOS シートで「ビデオを撮影」を選択 → カメラ起動
  - **注意**: シートに「写真を撮る / ビデオを撮影 / フォトライブラリ / ファイルを選択」の選択肢が並び、カメラは直接起動しません。`accept="image/*,video/*"` + `capture` の指定は、「カメラ優先」というヒントとして扱われます
- [x] アップロード結果: `content_type: video/quicktime` / `byte_size: 273897` / `sha256=11367cf4cb1b...`
  - `video/mp4` を期待していましたが、iPhone Safari は録画を **QuickTime (`.MOV`)** で保存することが分かりました
- [x] Publish レスポンス: `status: allowed`、`dataset_id=home/event/possible_littering`、`seller_token_jti=49bf45c467a65ddc`、`register_count=1`
- [x] 受信側 `/viewer` (macOS Safari, Tier 3 PurchaseViewerVC.full): 緑バッジ `tier: event+image+video` + `<video>` タグでインライン再生成功
  - **macOS Safari は `video/quicktime` をネイティブ再生できます**。案 B（HTTP メディア・ゲートウェイ）で QuickTime 形式を扱えることを確認できました

**スクリーンショット**:

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
カメラを直接起動させるには `accept="video/*"` だけに絞る必要がありますが、
そうすると既存の画像ファイルを選べなくなります。現在の実装は両方を指定したままに
しており、操作が 1 回増える代わりに既存ファイルも選べます。

#### B. PC ブラウザ録画 — Chrome（VP9）

**目的**: MediaRecorder + `getUserMedia` を使い、PC ウェブカメラから直接録画
→ アップロード → Publish が動くこと、Chromium 系では VP9 が選択されることを
確認します。

**手順**:

1. 同じ PC で publisher を起動した状態で、Chrome で `http://localhost:8080/provider/start?ds=home/event/possible_littering` を開く
2. iPhone のウォレットで SellerVC を提示（QR を読む / 同 LAN にいる前提）
3. 成功パネルから dataset を選んで `/provider` に遷移
4. **「🔴 ブラウザで録画」** セクションの「録画開始」を押す
5. ブラウザのカメラ/マイク権限ダイアログで **許可**
6. ライブプレビュー `<video>` にウェブカメラ映像が出ることを確認
7. 5〜10 秒待って「停止 & アップロード」
8. `recStatus` が「録画完了 (Ns) — アップロード中…」→「録画完了 (Ns)」と遷移
9. アップロード結果に `content_type: video/webm` が出るのを確認
10. `video_duration_sec` フォームに **実測秒数が自動入力**されているか確認
11. Publish 成功

**確認内容（codec）**:

DevTools コンソールで次を実行します。

```js
['video/webm;codecs=vp9,opus','video/webm;codecs=vp8,opus','video/webm','video/mp4']
  .filter(m => MediaRecorder.isTypeSupported(m))
```

期待値は `["video/webm;codecs=vp9,opus", "video/webm;codecs=vp8,opus", "video/webm"]` です
（先頭の要素が実際に使われます）。

**スクリーンショット（未配置）**:

```
images/data-user-vc-tiered/provider/B-chrome-permission.png         # カメラ権限ダイアログ
images/data-user-vc-tiered/provider/B-chrome-recording.png          # 録画中（赤バナー + プレビュー）
images/data-user-vc-tiered/provider/B-chrome-uploaded.png           # 録画完了 + URL/CID
images/data-user-vc-tiered/provider/B-chrome-publish-ok.png         # Publish 成功
images/data-user-vc-tiered/provider/B-chrome-devtools-codec.png     # DevTools の codec 判定
```

#### C. PC ブラウザ録画 — Firefox（VP8 フォールバック）

**目的**: Firefox でも録画から Publish までが動くこと、VP9 が無い場合に VP8 へ
フォールバックすることを確認します。

**手順**: B と同じ手順を Firefox で実施します。

**確認内容**:

DevTools で B と同じスニペットを実行して MediaRecorder.isTypeSupported を呼び出し、
**`vp9,opus` が `false` で、`vp8,opus` または `webm` が `true`** であることを
確認します。アップロードされたファイルの `content_type` も `video/webm` になっていれば成功です。

**スクリーンショット（未配置）**:

```
images/data-user-vc-tiered/provider/C-firefox-permission.png
images/data-user-vc-tiered/provider/C-firefox-recording.png
images/data-user-vc-tiered/provider/C-firefox-publish-ok.png
images/data-user-vc-tiered/provider/C-firefox-devtools-codec.png    # vp9=false, vp8=true
```

#### D. macOS Safari (Safari 17+ は WebM/VP9, Safari 16 以下は MP4 fallback)

**目的**: Safari 17 以降と 16 以下で codec の選択が分かれることを
確認します。**実機検証 (2026-04-30) で Safari 17+ は WebM/VP9 にネイティブ対応している
ことが分かった**ため、「Safari 17 以降も MP4 にフォールバックする」という当初の想定は当てはまりません。

**手順**: B と同じ手順を macOS Safari で実施します。

**確認内容**:

| Safari バージョン | サポート codec | 選ばれる codec | アップロード `content_type` |
|---|---|---|---|
| **17+** (modern) | vp9, vp8, webm, mp4 | **VP9** | `video/webm` |
| 16 以下 | mp4 のみ | MP4 | `video/mp4` |

`MediaRecorder.isTypeSupported('video/webm;codecs=vp9,opus')` の結果で動作が分かれます。

- Safari 17+ で `true` → VP9 が選ばれる（Chrome と同じ挙動）
- Safari 16 以下で `false` → MP4 にフォールバック

DevTools コンソールで実際の codec 一覧を出力すると、どちらに分かれたかを確認できます。

**Safari 固有の注意**:

- Safari 14.1 未満では `MediaRecorder` 自体が無く、`recStatus` が「このブラウザは
  録画に対応していません」になります。その場合は §11.7 のトラブルシュートに該当バージョン
  の情報を追記してください。
- マイク/カメラ権限を **アドレスバー左の Safari 設定アイコン**から付与する必要が
  ある場合があります。

**スクリーンショット（未配置）**:

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
- [ ] §11.8 冒頭の表の「状態」列を「検証済」に更新
- [ ] 実機で観測した codec 値・OS バージョン・特異な挙動を該当サブセクションに追記
- [ ] §11.7 トラブルシュートに、検証中に発見した新しい症状があれば追記

## 12. 意味レベルの段階化（VLM + 顔ブラー）

§1〜§11 のアクセス制御は、**メディアのキーを応答から省く**方式でした（Tier 2 では
動画のキー、Tier 1 では画像のキーも含まれません）。§12 では、同じ素材に
VLM（Vision Language Model。画像を入力に取れる言語モデル）による推論と、
顔 / PII（個人を特定できる情報）のブラー処理を適用して**派生データを生成し、tier 別に
出し分ける**仕組みを動かします。これにより Tier 1 の受信者にも、
**プライバシー情報を除いた要約テキスト**を渡せます。

設計の根拠は [DataUserVC × 段階アクセス制御 仕様 §「tier 拡張: 意味レベルでの段階化」](data-user-vc-tiered-spec.md#tier-vlm)
を参照してください。

### 12.1 新しい tier 定義

| Tier | アクセスレベル | 例 (score) | 出力する派生データ |
|---|---|---|---|
| **3** Full | `full` | 政府機関 + crime + ISO27001 (80) | 生 image / video + 顔ブラー画像 + 詳細文 + 概要文 |
| **2** Access | `access` | 企業 + research + ISO27001 (75) | **顔ブラー済 image** + **詳細文**（人名・物体名あり）+ 概要文 |
| **1** Summary | `summary`（新） | 企業 + 不明な purpose + legalCompliance のみ (50〜59) | **概要文のみ**（PII redact 済、image 無し） |
| 0 Denied | `denied` | 不適格 (<50) | claim 自体を拒否 |

`access` と `denied` の間に、新しい `summary` tier が加わります。VLM profile が有効なときだけ、
score 50〜59 が `summary` になります（profile が無効なら、従来どおり 60 未満は `denied` です）。
`/platform/data` のレスポンスに次のキーが増えます:

| キー | 内容 | 露出する tier |
|---|---|---|
| `image_url_redacted` | 顔・人物・ナンバープレート等をブラーした image URL | 2 + 3 |
| `image_cid_redacted` | 同 IPFS CID（案 C 有効時） | 2 + 3 |
| `description_full` | VLM が生成した詳細記述（人名・固有名詞あり） | 2 + 3 |
| `description_summary` | VLM が生成した概要（PII redact 済） | 1 + 2 + 3 |
| `description_model` | 推論に使った VLM のモデル ID + バージョン（監査用） | 全 tier |
| `description_generated_at` | 推論時刻（ISO8601） | 全 tier |
| `processing_warnings` | 推論やブラーが失敗したステップを列挙（§12.7 の degrade 通知） | 全 tier |

### 12.2 `--profile vlm` で起動

VLM と顔ブラーは **opt-in**（明示的に有効にしたときだけ動く機能）です。既定（profile 無効）では
§1〜§11 と同じ 3 段階の tier 投影で動きます。

```bash
cd ~/program/Blockchain_IoT_Marketplace
# profile vlm を ON にする。Ollama service と vlm-pull (llava の事前 pull)
# が一緒に立ち上がる
docker compose -f infra/docker-compose.yml --profile vlm up -d \
  publisher bridge mosquitto vlm vlm-pull
```

publisher の環境変数を VLM 用に切り替えます（compose ファイルが受け取る環境変数を
シェルで `export` するか、`.env` に書きます）。

```bash
export VLM_BACKEND=ollama
export IMAGE_REDACTION_BACKEND=opencv
docker compose -f infra/docker-compose.yml --profile vlm up -d publisher
```

`vlm-pull` が `llava` モデルを pull し終わるまで待ちます（初回のみ。数 GB あります）。

```bash
docker compose -f infra/docker-compose.yml logs -f vlm-pull
# -> "vlm model llava ready" が出たら抜ける
```

publisher が起動していることを確認します。

```bash
curl -s http://localhost:8080/health | jq .
# -> {"status": "ok", ...}
```

profile を無効にした場合と有効にした場合で同じデータセットの `/platform/data`
を取得すると、有効な場合にだけ `description_*` キーが含まれます。

### 12.3 4 種類の DataUserVC オファー

§2 の 3 種類に **summary tier 用**を追加します。

#### 12.3.a Tier 3（full）— §2a と同じ

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=GovernmentOrganization&purpose=CrimeSearch&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

#### 12.3.b Tier 2（access）— §2b と同じ

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=true&data_handling_policy=ISO27001&misuse_record=false' | jq .
```

#### 12.3.c Tier 1（summary, 新）— 企業 + 不明な purpose + legal compliance のみ

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=unknown&legal_compliance=true&data_handling_policy=other&misuse_record=false' | jq .
# score = 20 + 5 + 15 + 0 + 10 = 50 -> summary (VLM profile ON のときのみ)
```

#### 12.3.d Tier 0（denied）— §2c と同じ

VLM 拡張を有効にすると tier は 4 段階になり、summary が Tier 1、denied が Tier 0 になります。§2c の「Tier 1（denied）」は、VLM 拡張を使わない 3 段階の場合の呼び方で、同じプロファイルを指します。

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true' | jq .
```

### 12.4 4 通りの `/marketplace/claim` と `/platform/data` 比較

§3〜§4 と同じ要領で、4 通りそれぞれについて claim、PurchaseViewerVC の提示、ViewerToken の取得、
`/platform/data` の取得を行います。VLM profile が有効なとき、レスポンスに含まれるキーは次のように変わります。

| プロファイル | `event` | `image_url` | `video_url` | `image_url_redacted` | `description_full` | `description_summary` |
|---|---|---|---|---|---|---|
| 12.3.a Tier 3 (full) | あり | あり | あり | あり | あり | あり |
| 12.3.b Tier 2 (access) | あり | **なし** | **なし** | あり | あり | あり |
| 12.3.c Tier 1 (summary) | あり | **なし** | **なし** | **なし** | **なし** | あり |
| 12.3.d Tier 0 (denied) | claim 自体が `access_level: "denied"` で拒否 |

profile が無効なら従来どおり（§4 と同じ）3 段階の tier 投影になり、新しいキーは応答に含まれません。
この状態で 12.3.c の claim を送ると `denied` になります（summary tier は profile が有効なときだけ
使われます）。

### 12.5 顔/PII ブラーの確認方法

Tier 2 で受信した `image_url_redacted` の URL をブラウザで開くと、
**元画像（`image_url`）と同じ構図で、人物の顔がぼかされた**画像が返ります。

| 元 (`image_url`, Tier 3 のみ) | ブラー後 (`image_url_redacted`, Tier 2+) |
|---|---|
| ![pre-redaction](images/data-user-vc-tiered/vlm/V4-original.jpg){ width="300" } | ![post-redaction](images/data-user-vc-tiered/vlm/V4-redacted.jpg){ width="300" } |

publisher は内部で次の順に処理します。

1. 元画像を `/media/<sha>.<ext>` から fetch
2. OpenCV Haar cascade で顔検出
3. 検出された顔領域に Gaussian blur (kernel 51×51) を適用
4. 同じ extension で再エンコード → publisher 自身の `/media/upload` に POST
5. 戻ってきた URL/CID を `image_url_redacted` / `image_cid_redacted` に設定

同じ画像に対する 2 回目以降のリクエストでは、sha256 による重複排除が働き、
**ブラー済み画像の生成も省略**されます。

!!! note "MVP（最小構成の実装）の制限"
    Haar cascade は **正面顔のみ**検出します。横顔・遮蔽された顔・低解像度の顔は
    検出されず、ブラーがかかりません。ナンバープレート / IDバッジ / 画面 (鋭い矩形) などの検出器の追加は、
    [仕様の将来課題](data-user-vc-tiered-spec.md#tier-vlm) に記録しています。

### 12.6 VLM 出力の確認

Tier 2 / 3 で取得した `description_full` と Tier 1 の `description_summary` を
並べると、**人名・固有名詞が除かれているか**を確認できます。

```bash
# Tier 3 でフルキー取得
curl -s -H "authorization: Bearer $VIEWER_TOKEN_TIER3" \
     'localhost:8080/platform/data?dataset_id=home/event/possible_littering' \
  | jq '.rows[0] | {description_full, description_summary, description_model}'
```

期待される出力例:

```json
{
  "description_full": "John Smith dropped a Coca-Cola bottle near the Hibiya station entrance at around 14:32.",
  "description_summary": "An adult dropped a piece of litter near a public location during the afternoon.",
  "description_model": "ollama/llava"
}
```

`description_full` には人名（"John Smith"）・ブランド（"Coca-Cola"）・場所（"Hibiya
station"）が残り、`description_summary` には「an adult / public location /
afternoon」のような **一般化された語だけ**が出ているはずです。

!!! warning "redact 失敗の可能性"
    LLaVA などの汎用 VLM は確率的に動作するので、**`description_summary` から
    PII を完全には除けない場合があります**。
    たとえば、プロンプトの指示に反して人物の服の色や髪型を残す、特定の建物名を
    一般名詞と認識して残す、といった例です。
    仕様では、漏れの検出を後段のチェック処理（post-check）に分け、**研究課題**として
    記録しています（[仕様の「redact 失敗の検出」](data-user-vc-tiered-spec.md#tier-vlm)）。
    実運用では、Tier 1 の出力を人手でレビューする工程を挟むのが安全です。

### 12.7 degrade ケース（VLM / blur が停止したとき）

VLM か顔ブラーのどちらかが失敗しても publish は止まりません。機能を縮退（degrade）させて
処理を続け、`processing_warnings[]` で受信側に状況を通知します。

VLM だけを停止する場合:

```bash
docker compose -f infra/docker-compose.yml stop vlm
# /provider/publish を投げる → /platform/data で
# processing_warnings: ["vlm_unavailable"] が出る
# image_url_redacted は生成される (顔ブラーは VLM 不要のため)
# description_* キーは欠落
```

両方を停止する場合:

```bash
docker compose -f infra/docker-compose.yml stop vlm
# 加えて publisher の IMAGE_REDACTION_BACKEND を空にして再起動
# /platform/data で processing_warnings: ["vlm_unavailable", "redaction_unavailable"]
# Tier 2 / Tier 1 受信者は派生キー無し、生キーのみ (= profile OFF と等価)
```

degrade 時も `image_url` / `video_url` は **そのまま残る**ので、
受信側は Tier に応じた投影で、生のキーか派生キーのどちらかを必ず受け取ります。
受信側は、期待したキーの一部が無いときに `processing_warnings` を確認すれば状況が分かります。

### 12.8 既存 §11 (PWA Provider) との関係

§11 の `/provider` ページからアップロードして Publish した画像は、profile vlm が
有効ならそのまま VLM パイプラインを通ります。**provider 側のページに変更はなく**、
派生データはサーバー側で自動的に生成されます。受信側の `/viewer` は、
**Tier 2 で生のキーが無い場合に `image_url_redacted` を 🔒 バッジ付きで表示**し、
`description_full` と `description_summary` を緑とオレンジで並べて表示します。
`processing_warnings[]` がある行には警告バナーを表示します
（[Blockchain_IoT_Marketplace#48](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/48)
で実装済み）。

### 12.9 トラブルシュート

| 症状 | 対処 |
|---|---|
| `vlm-pull` が `Error: pull model manifest: file does not exist` | ネットワーク（image registry）に到達できていません。コンテナ内から `curl https://registry.ollama.ai` で確認してください。または `VLM_MODEL` を別の名前（`llava:7b` 等）に変更します |
| 1 回目の `/provider/publish` がタイムアウト | LLaVA の初回起動（cold start。モデルを VRAM にロードする処理）に 30〜60 秒かかります。`vlm-pull` のログに model ready が出ているか確認してください。出ていれば 2 回目以降は速くなります |
| `description_full` / `description_summary` が同じ内容 | LLaVA が prompt を区別していない可能性があります。`docker compose logs vlm` で `/api/generate` の呼び出しが 2 回別々に来ているかを確認してください。同じ画像でも prompt が違うので、戻り値は別々になるはずです |
| `image_url_redacted` の画像で顔がぼけていない | Haar cascade が検出するのは正面顔だけです。横顔・斜め顔にはブラーがかかりません。必要なら DNN 系の検出器に差し替えます |
| Profile OFF でも `description_*` キーが出る | 不具合です。profile が無効なのに従来の tier 投影になっていない可能性があります。`tests/test_data_user_vc_tiered_vlm.py::test_pipeline_no_injectors_keeps_legacy_envelope` が pass することを確認し、再現する場合は issue を立ててください |
| `processing_warnings: ["vlm_unavailable"]` が常時出る | Ollama が応答していないか、`VLM_API_URL` が間違っています。`docker compose exec publisher curl http://vlm:11434/api/version` で疎通を確認してください |

### 12.10 実機検証ログ

§11.8 と同じ形式で、VLM + 顔ブラーの実機検証ログを残します。

| シナリオ | 環境 | 状態 | 観測結果 |
|---|---|---|---|
| **V1** Tier 3 で生 + 派生キー揃う | macOS Chrome + Ollama (llava) | 部分検証 (2026-04-30) | pipeline ログで `vlm_describe_done full_len=280 summary_len=172` + `opencv_blur_faces detected=1` を確認。**`/platform/data` 経由の Tier 別投影は、OID4VP のフローが必要なため未実施** |
| **V2** Tier 2 で生キー欠落 / 派生キーあり | macOS Chrome | 未検証 | OID4VP のフローが必要なため未実施 |
| **V3** Tier 1 (summary) で text のみ | macOS Chrome | 未検証 | OID4VP のフローが必要なため未実施 |
| **V4** 顔ブラー視覚確認 | StyleGAN2 顔画像（人物識別不可な合成画像）→ publish → Tier 2 で受信 | **検証済 (2026-04-30)** | OpenCV が顔 1 つを検出し、Gaussian blur (51×51) を適用して、別ファイルとして再アップロード。顔を認識できない程度までぼけている。<br>[元画像](images/data-user-vc-tiered/vlm/V4-original.jpg) → [ブラー後](images/data-user-vc-tiered/vlm/V4-redacted.jpg) |
| **V5** description_full vs summary の品質 | StyleGAN2 sample | 部分検証 | VLM の 2 段階のプロンプトがどちらも完了（`full_len=280`, `summary_len=172`）。**実際のテキストの比較は、ViewerToken 経由の取得が必要なため未実施** |
| **V6** VLM を停止した状態の degrade | timeout 60s で実質 vlm_unavailable | 検証済 (2026-04-30) | `vlm describe failed: ollama call failed: timed out` → pipeline が `processing_warnings: ["vlm_unavailable"]` を出力し、image_url_redacted は問題なく生成。その後タイムアウトを延長した（下の表の 2）が、degrade の基本動作は確認できている |
| **V7** redact 失敗事例の収集 | 大規模 sample | 研究課題 | 仕様の post-check の実装が前提。今回の実機検証では未着手 |

#### 実機検証で見つかった追加バグ / 制約

V1〜V6 の実施中に、**運用上の問題が 3 件**見つかりました。

| # | 症状 | 原因 / 対応 |
|---|---|---|
| 1 | `/provider/publish` 後に `vlm describe failed: ollama call failed: timed out` | CPU 上の LLaVA-7B 推論は **1 stage あたり約 186 秒**かかります（1×1 ピクセル画像でも同じです）。describe() は 2 stage なので、合計で約 6 分必要です |
| 2 | 当初の既定値 VLM_TIMEOUT_SEC=60 では完了しない | [Blockchain_IoT_Marketplace#45](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/45) で既定値を 180 秒に変更しました。CPU 環境ではさらに `VLM_TIMEOUT_SEC=600` の指定が必要です |
| 3 | ハンズオンで使うには、CPU 上の LLaVA-7B は処理時間が長すぎる | 軽量モデル（`bakllava`, `moondream`）か、GPU 環境 / 外部 API の利用を推奨します |

#### moondream で再検証した結果 (2026-04-30 追補)

ollama に **`moondream`**（1.7 GB。llava の約 1/3 のサイズ）を入れて、同じパイプラインを実行した結果です。

| 指標 | llava | moondream |
|---|---|---|
| モデルサイズ | 4.7 GB | **1.7 GB** |
| describe() 完走時間（CPU） | ~6 分 | **~21 秒〜5 分**（プロンプトと cold/warm 状態に依存） |
| 現在の実装の長い 2 段階プロンプト | 期待通りの長文 (`full_len=280, summary_len=172`) | **断片のみ** (`full_len=3, summary_len=10`) |
| シンプルなプロンプト ("Describe this image.") | 動作するが冗長 | **高品質**: "A man with a beard and glasses... blurred green landscape" |

**結論**: 軽量 VLM (moondream) は大幅に速い一方で、**現在の長い 2 段階プロンプトにはうまく応答しません**。
プロンプトをモデルごとに調整する（モデルごとに適した長さ・言い回しを別の dict で持つ）か、
全モデル共通の短いプロンプトに統一する必要があります。

**短いプロンプトに統一**すると、redact の強さを細かく指定できなくなります。そのため、**モデルごとに調整した
プロンプトの dict を `vlm_client.py` に持つ**方針とし、今後の課題として記録します。

#### V4 視覚比較

| 元画像（合成） | OpenCV Haar cascade ブラー後 |
|---|---|
| ![original](images/data-user-vc-tiered/vlm/V4-original.jpg){ width="300" } | ![redacted](images/data-user-vc-tiered/vlm/V4-redacted.jpg){ width="300" } |
| 顔の細部（目・鼻・口）を判別できる | 顔の中央の矩形領域が Gaussian blur で判別できない。背景・髪・耳・あごは元のまま |

合成画像（StyleGAN2 出力）を使ったので、**実在の人物の PII は含まれていません**。

実装と、backend を差し替えるための接続点（plug-in seam）は、ブランチ `feat/stage-t-vlm-tier-real-backends`
([Blockchain_IoT_Marketplace#44](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/44))
で追加し、ユニットテスト（155 件すべて pass）で確認済みです。タイムアウトの延長は
[#45](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/45) で対応済みです。

## 13. 意味的中間表現と信頼度別レンダリング (semantic-tier 拡張)

§12 の VLM 段階化は、**同じ素材から派生テキストと顔ブラー画像**を生成して
信頼度別に出し分けるモデルでした。§13 ではこれを拡張し、

  **「映像 → 意味的中間表現 (SIR: Semantic Intermediate Representation) → 信頼度ポリシー → トラスト対応レンダラ → 出力」**

という 4 層パイプラインを使います。**プライバシーリスクのある領域 (顔・テキスト・画面・
白板・書類・名札・IDカード・ナンバープレート + unknown_sensitive) を bbox（bounding box。矩形領域）単位で
構造化し、信頼度ごとに何を見せるかを宣言的に決定**します。原映像が低信頼ユーザに
渡る経路がコード上に存在しない設計です（fail-closed by construction。失敗したときや
判定できないときは、開示しない側に倒れます）。

詳細な設計根拠は
[DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) を参照してください
（§13 に対応する章は仕様ページにまだ無く、追加を予定しています）。

### 13.1 アーキテクチャ概観

```
[iPhone Safari の /provider]   §1〜§2 と同じフローでフレームをアップロード
       │  POST /media/upload
       ▼
[publisher SemanticAnalyzer]   /semantic/analyze
       │  ├─ MockSemanticAnalyzer   (テスト・MVP)
       │  ├─ VisionSemanticAnalyzer (OpenCV Haar + MSER)
       │  └─ Apple Vision / Core ML / VLM (将来課題)
       ▼
[Semantic Intermediate Representation (SIR) JSON]
       │  正規化座標の bbox + sensitive_regions[] + events[] +
       │  privacy_risk_score + analyzer_version
       ▼
[TrustPolicyEngine]            /semantic/render or /semantic/render_url
       │  ViewerTrustLevel ∈ {anonymous,low,medium,high,owner,admin}
       │  → DisclosurePolicy (allowed_outputs + mask_regions + audit_required)
       ▼
[TrustAwareRenderer]
       │  textSummary / eventList / redactedImage / lowResolutionImage /
       │  maskedVideoFrame / originalFrame
       ▼
[Viewer Output]
```

各層は独立してテストできます。ポリシー判定や VLM 出力に誤りがあっても
**安全側（隠す側）に倒れる**ことを、層の構造で保証します。

### 13.2 信頼度マッピング

`ViewerTrustLevel` (6 段階) と `OutputKind` (6 種) の対応表:

| Level | textSummary | eventList | redactedImage | lowResolutionImage | maskedVideoFrame | originalFrame |
|---|---|---|---|---|---|---|
| `anonymous` | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ |
| `low` | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ |
| `medium` | ✓ | ✓ | **✓** | **✓** | ✗ | ✗ |
| `high` | ✓ | ✓ | ✓ | ✓ | **✓** | ✗ |
| `owner` | ✓ | ✓ | ✓ | ✓ | ✓ | **✓** (audit) |
| `admin` | ✓ | ✓ | ✓ | ✓ | ✓ | **✓** (audit) |

`unknown_sensitive` は **HIGH 以下で常にマスク**します（HIGH だけは
`SEMANTIC_ALLOW_UNKNOWN_AT_HIGH=true` を明示的に設定すると解除できます）。

### 13.3 起動

VLM tier (§12) を有効にした publisher に **`SEMANTIC_ANALYZER_BACKEND` 環境変数**
を加えます:

```bash
cd ~/program/Blockchain_IoT_Marketplace
SEMANTIC_ANALYZER_BACKEND=vision \
  VLM_BACKEND=ollama VLM_MODEL=moondream IMAGE_REDACTION_BACKEND=opencv \
  docker compose -f infra/docker-compose.yml --profile vlm up -d publisher
```

| 環境変数 | 値 | 効果 |
|---|---|---|
| `SEMANTIC_ANALYZER_BACKEND` | 空 / `stub` | MockSemanticAnalyzer (決定論的、3 region 固定出力) |
| `SEMANTIC_ANALYZER_BACKEND` | `vision` | OpenCV Haar cascade + MSER |
| `SEMANTIC_ALLOW_UNKNOWN_AT_HIGH` | `true` | HIGH ティアのみ unknown_sensitive を unmask |

### 13.4 /provider §1.5 「分析を実行」パネル

§11.3 でメディアをアップロードした直後に、`/provider` ページに **「1.5 意味的中間表現を確認」**
セクションが現れます（見出しの「§1.5」は `/provider` ページ内の番号で、本ページの節番号とは別です）。
Publish の前に、受信者からどう見えるかを試しに確認する（dry-run）ための UI です。

1. アップロード完了 → 「分析を実行」ボタンが有効化
2. ボタン押下 → **`POST /semantic/analyze`** で SIR を取得
3. 検出された `sensitive_regions` を **赤 bbox オーバーレイ**で元画像に重ねて表示
4. 4 つのカード (anonymous / low / medium / high) で **各信頼度の見え方**をプレビュー
   - granted_kinds, text_summary, ポリシーが許可した場合はマスク済画像を inline 表示
5. 提供者は実際の Publish の前に「medium の受信者には何が見えるか」を実機で確認可能

この確認により、アナライザの誤検知や過検出をイベント発行前に見つけられます。
Publish ボタンとは独立して動くので、確認だけして Publish せずにやり直すこともできます。

### 13.5 /viewer 🔬 トグル — 信頼度別レンダリング

受信側 `/viewer` の上部に **「🔬 意味的レンダリングを使う (実験)」** チェックボックスが
追加されます。OFF（既定）では §10 と同じ従来の tier 投影で表示します。ON では次のように動作します。

1. ページ load 時に `/platform/data` の `allowed_views` から信頼度を導出
   ```
   video         → high
   image | image_redacted → medium
   description_* → low
   event のみ / 不明 → anonymous (fail-closed)
   ```
2. 各 row で **`POST /semantic/render_url`** を呼び、サーバ側で fetch + analyze + policy + render を 1 往復で実行
3. 結果を表示:
   - `[<level>] kinds: redactedImage + textSummary + eventList`
   - 許可された image_b64 を inline `<img>` で展開
   - text_summary, event_list, audit_required バナー、policy rationale (debug)

トグル状態は localStorage に保存されるので、リロードしても選択が維持されます。
**OFF のときの動作は従来とまったく同じ**なので、§12 までの手順には影響しません。

### 13.6 fail-closed の設計箇所

| 箇所 | 振る舞い |
|---|---|
| `_coerce_trust_level()` | 不明な値 / None → ANONYMOUS |
| `TrustPolicyEngine.evaluate()` | `privacy_risk_score >= 0.9` で image kinds 剥奪 |
| `TrustPolicyEngine.evaluate_safe()` | 内部例外 → `allowed_outputs=frozenset()` (空) |
| `_compute_mask_plan()` | face/text/screen/whiteboard/document/id_card/name_tag/`unknown_sensitive` を MEDIUM 以下で常時マスク |
| `TrustAwareRenderer.render()` | ORIGINAL_FRAME に対して `trust_level in (OWNER, ADMIN)` を二重チェック |
| `_render_masked()` | cv2 / decode / encode 失敗 → text fallback |
| `SemanticIntermediateRepresentation.empty()` | `privacy_risk_score=1.0` でアナライザ失敗を fail-closed 状態として表現 |
| `/semantic/analyze` | 失敗時 `empty()` SIR を返す（raw bytes 一切返さない）|
| `/semantic/render_url` | URL fetch 失敗 → `empty()` SIR + text-only |

### 13.7 サーバ側監査ログ

`/semantic/render` および `/semantic/render_url` は、次のいずれかに当たる場合に
`audit_log` に行を追加します。

- ポリシーが `audit_required=True`（OWNER / ADMIN ティア）
- レンダラが実際に `image_bytes` を返した（信頼度問わず）

監査行に記録されるフィールド:

| フィールド | 値 |
|---|---|
| `ts` | UTC ISO8601 |
| `action` | `semantic/render` または `semantic/render_url` |
| `subject_did` | `sir.source_device_id` |
| `purpose` | `semantic_render` |
| `reason` | `trust=<level>;kinds=<list>;image=yes|no;audit_required=<bool>;rationale=<policy>` |
| `message_hash` | `sir.frame_id` |
| `presentation_verified` | `owner_or_admin` または `policy_only` |

**記録しないもの**: 原フレームのバイト列、base64 化されたレンダリング画像、SIR から
派生する顔エンコーディング・人名等。`image_url` は記録します（publisher 内
`/media/<sha>.<ext>` のみで、URL 単独では権限のない受信者が画像を取得できない設計
のため）。

ANONYMOUS / LOW での text-only 呼び出しは web access log の範囲とし、audit テーブルには
記録しません（audit テーブルの記録を「実際の開示」に絞るためです）。

### 13.8 API リファレンス

#### `POST /semantic/analyze`

- 入力: multipart `file` フィールド + `source_device_id` フォーム値
- 出力: SIR JSON (`{frame_id, source_device_id, captured_at, scene_summary,
  objects, people, sensitive_regions, events, privacy_risk_score, analyzer_version}`)
- セキュリティ: 入力バイト列を応答に**含めません**。アナライザ失敗時は
  `SemanticIntermediateRepresentation.empty()` を返します。

#### `POST /semantic/render`

- 入力: JSON `{trust_level?: string, sir: object, image_url?: string}`
- 出力: `{trust_level, granted_kinds[], text_summary, event_list[], rationale,
  audit_required, image_b64?, image_content_type?}`
- セキュリティ: `trust_level` が不明な場合は ANONYMOUS として扱います。`image_url` は publisher 内の
  URL だけを fetch します。低信頼ユーザには image_b64 を付与しません。

#### `POST /semantic/render_url`

- 入力: JSON `{trust_level?: string, image_url: string, source_device_id?: string}`
- 出力: `/semantic/render` と同じ
- 動作: サーバ側で fetch + analyze + render を 1 回の呼び出しで実行します。fetch または analyze に
  失敗した場合は、`empty()` SIR を使って fail-closed で応答します。

### 13.9 既存 VLM tier (§12) との関係

| 観点 | §12 VLM tier | §13 semantic pipeline |
|---|---|---|
| 信頼度の決定方法 | DataUserVC の `allowed_views` フィールドが claim 段階で決まる | `allowed_views` から実行時に派生 + 既定で legacy 動作 |
| 派生データの形式 | テキスト (description_full / summary) + 顔ブラー画像 | 構造化 SIR (bbox + sensitive_regions + events) |
| マスク粒度 | 顔のみ (`image_url_redacted`) | 顔 + テキスト + 画面 + 白板 + 書類 + 名札 + ID + プレート + unknown_sensitive |
| 受信側での切替 | claim mint 時に固定 | `/viewer` トグルで実行時切替可能 |
| 失敗時 | `processing_warnings: ["vlm_unavailable"]` | `empty()` SIR + text-only fallback |
| 監査ログ | image_redactor / vlm_client 個別 | `/semantic/render*` の audit hook が一元的に記録 |

両者は**併用できます**。`/viewer` の 🔬 トグルは、§12 までの tier 投影をそのまま残し、
ON にしたときだけ semantic pipeline 経由の表示に切り替えます。

### 13.10 実機検証ログ

§11.8 / §12.10 と同じ形式で記録します。

| シナリオ | 環境 | 状態 | 検証ポイント |
|---|---|---|---|
| **S1** /provider §1.5 panel | iPhone Safari (実画像) | **検証済 (2026-04-30)** | iPhone カメラ撮影画像をアップロード → §1.5 「分析を実行」 → analyzer=`vision-opencv`、`face` (conf 0.85) + `unknown_sensitive` (0.50) を検出、bbox 赤枠が画像上にオーバーレイ。`privacy_risk_score=0.90`、4-tier preview cards (anonymous / low / medium / high) もそれぞれ disclosure を表示。<br>[S1 screenshot](images/data-user-vc-tiered/semantic/S1-provider-analyze.png) |
| **S2** /viewer 🔬 toggle | iPhone Safari + Tier 3 の PurchaseViewerVC | **検証済 (2026-04-30)** | Tier 3 の PurchaseViewerVC (`PurchaseViewerVC.full`、tier=`event+image+video+image_redacted+description_full+...`) を発行し、iPhone Safari の /viewer で 1 行に対して 🔬 toggle を切替: **OFF** = legacy 表示で生写真 + `vlm_unavailable` 派生データ警告。**ON** = `/semantic/render_url` 経由で `derived trust level: high`、`[high] kinds: textSummary + eventList`、「1人の人物 + 2件のセンシティブ領域」「person_detected: 1人の人物が検出された」を表示し、生写真は出さない（信頼度に応じた表示の切り替えを確認）。<br>[toggle OFF](images/data-user-vc-tiered/semantic/S2-viewer-toggle-off.png) / [toggle ON](images/data-user-vc-tiered/semantic/S2-viewer-toggle-on.png) |
| **S3** OWNER tier audit log | publisher 単独 (curl) | **検証済 (2026-04-30)** | OWNER 信頼度で `/semantic/render_url` を呼ぶと image_b64 + 全 kinds が返り、`/audit/logs` に `semantic/render_url` 行が記録される (`trust=owner;kinds=textSummary+redactedImage+eventList+originalFrame+maskedVideoFrame+lowResolutionImage;image=yes;audit_required=True`) |
| **S4** Vision analyzer 顔検出 | publisher (`SEMANTIC_ANALYZER_BACKEND=vision`) | **検証済 (2026-04-30)** | 合成顔画像で `/semantic/analyze` を呼び出すと SIR に `face` (confidence 0.85, bbox: x=0.14, y=0.19, w=0.70, h=0.70) + `unknown_sensitive` (MSER 由来) が含まれる。privacy_risk_score=0.9 (face 0.4 + unknown 0.5)|
| **S5** Fail-closed: VisionAnalyzer cv2 不在 | publisher (`sys.modules['cv2']=None` simulate) | **検証済 (2026-04-30)** | container 内で cv2 / numpy を欠落させた状態で `VisionSemanticAnalyzer.analyze()` を呼ぶと `SemanticAnalyzerError("vision backend requires cv2 + numpy")` が raise され、route 側で `SemanticIntermediateRepresentation.empty()` (objects=0, sensitive=0, events=0, **privacy_risk_score=1.0**) に demote。trust=anonymous/low/medium/**high** すべて renderer が `high_risk_score(1.00)>=ceiling(0.9); image kinds dropped` の rationale で image を落とし、`textSummary + eventList` のみ返す。OWNER のみ `audit_required` path で image_bytes を許可 (期待動作)。 |

実装は
[Blockchain_IoT_Marketplace#49](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/49)
（パイプライン本体）、
[#50](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/50)
（/provider §1.5）、
[#51](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/51)
（/viewer 🔬）、
[#52](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/pull/52)
（audit hook）で行い、ユニットテスト（190 件すべて pass）で確認済みです。

## 次に進む

- [スマホSSIウォレットサンプル](ha-ssi-wallet.md) — Phase 2 wallet を立ち上げる
- [DataUserVC × 段階アクセス制御 仕様](data-user-vc-tiered-spec.md) — 設計根拠
- [USBウェブカメライベント共有サンプル](webcam-event-sharing.md) — 商品化の前段
