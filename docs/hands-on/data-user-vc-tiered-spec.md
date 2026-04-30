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

文字列マッチは **大文字化したうえで完全一致**（スペース込み）。
未知のラベルは default = **5** にフォールバックします。

```
entityType (uppercase exact match):
  GOVERNMENTORGANIZATION | POLICE       -> 35
  ENTERPRISE                            -> 20
  RESEARCH ORGANIZATION                 -> 15  (← スペース込みで一致)
  unknown                               -> 5

purpose (uppercase exact match):
  CRIME SEARCH                          -> 25  (← スペース込みで一致)
  TRAFFIC MANAGEMENT                    -> 20
  RESEARCH                              -> 15
  unknown                               -> 5

legalCompliance == true                 -> +15
dataHandlingPolicy == "ISO27001"        -> +15
misuseRecord == false                   -> +10
misuseRecord == true                    -> -10
```

### 実機で観測したスコア例

| プロファイル | entityType | purpose | legal | policy | misuse | score |
|---|---|---|---|---|---|---|
| Tier 3 (full) | `GovernmentOrganization` | `CrimeSearch` | true | ISO27001 | false | **80** (35+5+15+15+10) |
| Tier 2 (access) | `Enterprise` | `Research` | true | ISO27001 | false | **75** (20+15+15+15+10) |
| Tier 1 (denied) | `Enterprise` | `Research` | false | Other | true | **25** (20+15+0+0-10) |

`CrimeSearch` の purpose ラベルは Solidity 側の `"CRIME SEARCH"`
（スペース込み）と一致しないため、現在の Python 実装ではフォールバック値
**5** になります（だから 35 + 25 ではなく 35 + 5 で 80）。
ハンズオンでは「ティア境界が再現できれば良い」のでこれで十分機能しますが、
将来 ZK / オンチェーン同期を入れるときは UI の入力ラベルを正規化する
小修正が要ります（フォロー TODO）。

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

## tier 拡張: 意味レベルでの段階化（VLM）

ここまでの §1〜§9 は「**メディアの欠落**」によるアクセス制御です（Tier 2 は
動画キーが消える、Tier 1 は画像も動画も消える）。次の段階として、**同じ素材から
情報処理（VLM 推論 + 顔/PII ブラー）を経由した派生データを生成し、tier 別に
出し分ける**設計に拡張します。Tier 1 は「素材を全く渡さない」のではなく
「**プライバシー情報を抜いた要約テキスト**」を渡すようになり、low trust の
受信者でも「何が起きたか」を知ることはできるが「誰がやったか」は分からない、
という新しい段階が生まれます。

### 新しい tier 定義

| Tier | アクセスレベル | 例 (score) | 出力する派生データ |
|---|---|---|---|
| **3** Full | `full` | 政府機関 + crime + ISO27001 (80) | 生の image / video + 全テキスト派生 |
| **2** Access | `access` | 企業 + research + ISO27001 (75) | **顔/PII ブラー済 image** + **詳細テキスト**（人名・物体名あり） |
| **1** Summary | `summary`（新） | 企業 + research のみ (60〜) | **概要テキストのみ**（PII redact 済、image 無し） |
| 0 Denied | `denied` | 不適格 (<60) | claim 自体を拒否 |

`access_level` の値域に **`summary`** を追加します（既存の `denied` のセマンティクスは
変更なし — score 60 未満は claim 拒否のまま）。

### 派生データの schema

`/platform/data` のレスポンス（rows 内の各オブジェクト）に次のキーを追加します:

| キー | 内容 | 露出する tier |
|---|---|---|
| `image_url_redacted` | 顔・人物・ナンバープレート等をブラーした image の URL | 2 + 3 |
| `image_cid_redacted` | 同 IPFS CID（案 C 有効時） | 2 + 3 |
| `description_full` | VLM が生成した詳細記述（人名・固有名詞あり） | 2 + 3 |
| `description_summary` | VLM が生成した概要（PII redact 済） | 1 + 2 + 3 |
| `description_model` | 推論に使った VLM のモデル ID + バージョン（監査用） | 全 tier |
| `description_generated_at` | 推論時刻（ISO8601） | 全 tier |

既存の `image_url` / `image_cid` / `video_url` / `video_cid` は **Tier 3 のみ**で
露出するように変更します（旧 Tier 2 で見えていた `image_url` は
`image_url_redacted` に置き換わる）。

### 新しい allowed_views ラベル

```
allowed_views ⊆ {
  "event",                # 既存: メタデータ
  "image",                # 既存: 生 image_url / image_cid (Tier 3 のみ)
  "video",                # 既存: 生 video_url / video_cid (Tier 3 のみ)
  "image_redacted",       # 新: image_url_redacted / image_cid_redacted (Tier 2+)
  "description_full",     # 新: description_full (Tier 2+)
  "description_summary",  # 新: description_summary (Tier 1+)
}
```

### tier → allowed_views マッピング（更新）

```
"full"    -> ["event", "image", "video",
              "image_redacted",
              "description_full", "description_summary"]
"access"  -> ["event",
              "image_redacted",
              "description_full", "description_summary"]
"summary" -> ["event",
              "description_summary"]
"denied"  -> []
```

`full` は **生も派生も全部**見える上位互換。`access` は生メディア無し・ブラー画像
ありの中間。`summary` は **テキストのみ**で、画像/動画は redacted も含めて一切なし。

### VLM パイプライン契約

publisher 内蔵で `--profile vlm` を ON にすると、`/provider/publish` の処理経路に
次のステップが挟まります。

```
provider page
  ├── /media/upload で image を保存（既存）
  └── POST /provider/publish
        └── pipeline.process_message
              ├── (新) VLMClient.describe(image_url)
              │     -> {description_full, description_summary, model, generated_at}
              ├── (新) ImageRedactor.blur_pii(image_url)
              │     -> {image_url_redacted, image_cid_redacted}
              ├── 既存: consent / policy 評価
              └── 既存: platform_client.send(envelope)
```

VLM 不在（profile 無効 or 推論失敗）時の **degrade ポリシー**:

| 条件 | publisher の挙動 |
|---|---|
| `--profile vlm` 無効 | キーを **生成しない**（旧仕様と同じ — Tier 1/2 は従来の挙動にフォールバック） |
| profile 有効 + VLM 呼び出し失敗 | description キーは省略、`image_url_redacted` のみ生成 試行（顔ブラー単体は VLM 不要なため）|
| profile 有効 + 顔ブラー失敗 | redacted 画像キー省略、description のみ。Tier 2 受信者には image 無しで text のみ届く |
| profile 有効 + 両方成功 | 全キー生成 |

degrade 時はレスポンスに **`processing_warnings: ["vlm_unavailable", ...]`** を
乗せて受信側が状況を把握できるようにします。

### VLM の実装選択肢

- **MVP**: ローカル LLaVA (Ollama 経由 / `ollama run llava` を compose service として起動)
- **将来**: より大きなマルチモーダル LLM（Qwen2-VL 等）への差し替え、または external API
  への切り替えを `VLM_BACKEND=ollama|openai|anthropic` 環境変数で制御
- **stub モード** (`VLM_BACKEND=stub`): テスト用。決定的にダミー文字列を返す

### 顔/PII ブラーの実装選択肢

- **MVP**: OpenCV Haar cascade で顔検出 → Gaussian blur → 同 sha256 重複排除で再保存
- **拡張候補**: ナンバープレート（OCR ベース）、画面（鋭い矩形検出）、IDバッジ等を
  順次追加。検出器は `RedactionPipeline` の plug-in として差し込めるよう interface を
  切る

入力画像の content_type が動画の場合、MVP では **Tier 2 用の redacted は生成しない**
（動画フレーム抽出 + フレーム別ブラーは将来課題）。Tier 3 のみで動画を露出し、
Tier 2 は description_full テキストのみで済ませます。

### redact 失敗の検出（研究課題）

VLM が `description_summary` から PII を完全には抜けないケースは構造的に発生し得ます。
本仕様では **検出責任を post-check ステージに分離**し、初期実装は研究 TODO として
記録するに留めます。

検出案:

1. `description_full` から固有名詞（人名・組織名・場所名）を NER で抽出
2. `description_summary` の中に同じ単語が残っていないか diff
3. 残っていれば既知 PII 辞書 / 国別人名辞書と照合し、確度高なら
   `processing_warnings: ["redaction_leak_suspected"]` を立てる
4. 警告ありの行は audit 経路で人手レビュー queue に積む

これは hands-on の MVP には含めません（spec で意図だけ明文化）。本格運用時は
別途 PR で実装する想定です。

### 監査 (audit log) への影響

`audit/logs` に `description_model` と `description_generated_at` が
記録されることで、**どの VLM 出力が誰にいつ何の tier で届いたか**が再現可能に
なります。VLM がアップグレードされた前後で同じ画像から異なる説明が生成された
場合、generated_at で分離できる前提です。

### テスト方針（追加分）

`tests/test_data_user_vc_tiered_vlm.py` を新設、次のケースをカバー:

1. `--profile vlm` 無効時に既存の Tier 投影が回帰しない（5）
2. stub VLM backend で `description_*` キーが期待通りの shape で出る（3）
3. tier 別の `/platform/data` 投影マトリクス（4）
   - Tier 3: 生キー + 派生キー全部
   - Tier 2: image_redacted + description_full + description_summary、生 image/video なし
   - Tier 1: description_summary のみ
   - Tier 0 (denied): claim 自体が 403
4. VLM 呼び出し失敗時の degrade（2）
   - VLM 失敗 → 顔ブラー成功 → description キー無しで Tier 2 が image_redacted のみ
   - 両方失敗 → Tier 1/2 両方とも description_summary 無し（純粋に degraded）
5. `processing_warnings` の流出経路（2）

合計 +16 ケース、既存 88 と独立に管理します（profile-OFF が回帰しないことが
最重要）。

### 将来課題（spec 上で明記）

- 動画 → フレーム抽出 → VLM 集約による Tier 2/1 動画派生
- VLM 出力の **再現性保証**（同じ image + 同じプロンプトで同じ description が
  返る backend のみ permit するか、generated_at で諦めるか）
- redact post-check の実装（前述）
- VLM 推論の **キャッシュ層**（同じ sha256 image に対して同じ description を
  再利用 — 現状の image dedup と対称な扱い）
- VLM 推論の **コスト課金**（DataUserVC tier に応じて呼び出し回数を制限する等）
