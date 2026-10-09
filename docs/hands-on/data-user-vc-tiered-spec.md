# 段階アクセスの設計仕様

データ受信者（buyer/recipient）が保持する **DataUserVC** の信頼属性に応じて、
Publisher が公開するカメラ視野を **イベント／画像／動画** の 3 段階に
動的に絞り込むための設計文書です。

このページでは、既存の実装のうち**どこを固定し、どこを差し替えるか**を定義します。
ハンズオン手順は [DataUserVC × 段階アクセス](data-user-vc-tiered.md) を参照してください。

## このページで分かること

- 既存 5 種類の VC（ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC）と、
  6 種類目として追加する **DataUserVC** の関係
- Li らのスマートコントラクト `ssi/contracts/DataUserVerifier.sol`（[論文](../publications.md) 1 の著者による実装）に書かれた trustScore（信頼度スコア）算出ロジックを
  Phase 2 の Publisher にどのように取り込むか
- 「VC 検証 → trustScore → allowed_views」という単方向データフロー
- データ供給側（HA / RaspberryPi / USB Webcam）と Publisher の責務分離

## つまずきやすい点

- 「DataUserVC でカメラ映像が見える／見えない」を**オンチェーンのスマートコントラクト**で判定すると、
  オフラインでは失敗します。今回の実装は **Publisher 側の Python で評価**します
- アクセスレベルは trustScore（属性ごとの点数の和）だけでは決まりません。`full` になるのは、
  entityType が高信頼の区分（`GovernmentOrganization` / `Police`）で、かつ score>=80 のときだけです
- ViewerVC / PurchaseViewerVC が持つ `allowed_views` は **Token を発行する時点**で
  確定し、以後は変えません（発行後に権限を引き上げません）

## 目的

Phase 2 で実現済みの「VC 提示 → 検証 → token mint → /platform/data でイベント取得」
というフローに対し、**「何を取得できるか」を VC 属性で段階化**します。

- entityType（GovernmentOrganization / Police / Enterprise / ResearchOrganization）
- purpose（CrimeSearch / TrafficManagement / Research）
- legalCompliance（true / false）
- dataHandlingPolicy（ISO27001 / その他）
- misuseRecord（過去の濫用記録）

これらを Li らのスマートコントラクト `DataUserVerifier.sol` と同じ重み付けで
スコア化し、`full` / `access` / `denied` の 3 段階に分類します。

## この仕様の前提

差し替える対象は **Publisher 内の VC 検証層 + token 層 + projection 層（応答に含めるキーを選別する層）** だけです。
以下はそのまま再利用します。

- 既存 5 種類の VC（ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC / SellerVC）
- OID4VCI（VC 発行）/ OID4VP（VC 提示）/ DCQL（提示要求の問い合わせ言語）のエンドポイント
- bridge ↔ Hardhat の Purchase event 経路
- Webcam / HA / Demo simulator のデバイス側コード（変更なし）

**Li らの `DataUserVerifier.sol` は、Phase 2 の Publisher が評価ロジックの基準として参照するもの**
で、オンチェーン呼び出しはしません（理由は[なぜオンチェーン呼び出しをしないか](#なぜオンチェーン呼び出しをしないか)を参照）。

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
発行するときの判断材料**として使います。

図中の post-PEX checks は、提示された VC が要求条件に合うことを照合した後に行う追加の検証を指します。

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
結果は `DataUserVerifier.sol` と一致させ、テストで両者を突き合わせます。

### 重み

文字列は **大文字化したうえで完全一致**（スペースも含む）で照合します。
未知のラベルは既定値の **5** にフォールバックします。

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
**5** になります（そのため 35 + 25 にはならず、35 + 5 を含む合計 80 になります）。
ハンズオンではティアの境界を再現できれば足りるので、このままで問題ありません。
将来、ゼロ知識証明（ZK）やオンチェーンとの同期を入れるときには、UI の入力ラベルを
正規化する修正が必要です（未対応で、今後の課題です）。

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

`DataUserVC` のときは 5 属性すべてが必須です。欠けている場合は 400 を返します。

### `POST /verifier/request_object`

`vc_kind=DataUserVC` を受け付けます。DCQL の `claims` で上記の 5 属性を要求します。

### `POST /verifier/*`（提示エンドポイント）

`DataUserVC` が提示されたときの動作は次のとおりです。

- post-PEX checks で 5 属性が claim に存在するか検証
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

`data_user_attrs` を省略した場合は、`allowed_views=["event"]` の最小権限で claim します。

### `GET /platform/data`

`ViewerToken.allowed_views` に応じて、出力するフィールドを選びます。

| allowed_views に含まれる | 出力フィールド |
|---|---|
| `event` | 既存 event JSON 全体 |
| `image` | `image_cid` 追加 |
| `video` | `video_cid`, `video_duration_sec` 追加 |

`allowed_views` に含まれないビューのキーは**応答から省きます**（値を null にするのではなく、キーごと出力しません）。

## なぜオンチェーン呼び出しをしないか

`DataUserVerifier.sol` は研究プロトタイプで、以下の事情があります。

- VC フォーマットが W3C VC v1 JSON-LD（Phase 2 は SD-JWT VC）
- DID 方式が `did:key`（Phase 2 は `did:jwk`）
- gas / 確認待ちが OID4VP のフローに合わない
- ハンズオン環境（Hardhat ローカル）では永続性が壊れやすい

そこで **ロジック（重みとしきい値）だけを Python に再実装**し、
スマートコントラクト側は、将来の本番環境へ移行するときに使う候補として残します。

`trust_score.py` は **`DataUserVerifier.sol` のミラー**として保守し、
変更時はテストで両者が一致することを確認します。

## ssi-ui の扱い

`ssi-ui/` 配下の Next.js 試験 UI は **非推奨（deprecated）** とし、
今後の機能追加は Phase 2 wallet（`iw3ip-wallet`）+ Publisher 側に集約します。

## テスト方針

`tests/test_data_user_vc_tiered.py` に次の 13 ケースを置きます（括弧内はケース数）。

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

テストは合計 88 件（既存 75 + 新規 13）で、既存テストに失敗はありません。

## 将来拡張

- DataUserVC を **Verifiable Presentation** にして複数 VC を束ねる
- `DataUserVerifier.sol` を Polygon zkEVM などで再運用し、
  `trust_score.py` をフォールバックに変更
- `allowed_views` を時間帯ベースに拡張（夜間のみ video など）

## tier 拡張: 意味レベルでの段階化（VLM）  {#tier-vlm}

ここまでの設計（[ハンズオン](data-user-vc-tiered.md)の §1〜§7 と、[続きのページ](data-user-vc-tiered-media.md)の §8〜§9 に対応）は、**メディアのキーを応答から省く**方式の
アクセス制御です（Tier 2 では動画のキー、Tier 1 では画像と動画のキーが含まれません）。
次の段階として、同じ素材に VLM（Vision Language Model。画像を入力に取れる言語モデル）による推論と
顔 / PII（個人を特定できる情報）のブラー処理を適用して**派生データを生成し、tier 別に
出し分ける**設計に拡張します。Tier 1 には**プライバシー情報を除いた要約テキスト**を
渡します。信頼度の低い受信者は「何が起きたか」を知ることができますが、
「誰がやったか」は分かりません。

### 新しい tier 定義

| Tier | アクセスレベル | 例 (score) | 出力する派生データ |
|---|---|---|---|
| **3** Full | `full` | 政府機関 + crime + ISO27001 (80) | 生の image / video + 全テキスト派生 |
| **2** Access | `access` | 企業 + research + ISO27001 (75) | **顔/PII ブラー済 image** + **詳細テキスト**（人名・物体名あり） |
| **1** Summary | `summary`（新） | 企業 + 不明な purpose + legalCompliance のみ (50〜59) | **概要テキストのみ**（PII redact 済、image 無し） |
| 0 Denied | `denied` | 不適格 (<50) | claim 自体を拒否 |

`access_level` の値域に **`summary`** を追加します。VLM profile が有効なときだけ、score 50〜59 が
`summary` になります。profile が無効な場合の `denied` の扱いは変わらず、score 60 未満は claim を拒否します。

### 派生データの schema

`/platform/data` のレスポンス（rows 内の各オブジェクト）に次のキーを追加します:

| キー | 内容 | 露出する tier |
|---|---|---|
| `image_url_redacted` | 顔・人物・ナンバープレート等をブラーした image の URL | 2 + 3 |
| `image_cid_redacted` | 同 IPFS CID（[ハンズオン §9](data-user-vc-tiered-media.md#9-案-cローカル-kubo-ipfs-daemon-で分散配信) の案 C、IPFS 配信が有効な場合） | 2 + 3 |
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

`full` では**生のデータも派生データもすべて**見えます。`access` では生のメディアは見えず、
ブラー画像と説明文が見えます。`summary` は**テキストのみ**で、画像 / 動画はブラー済みのものも含めて見えません。

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

VLM が使えない場合（profile が無効、または推論に失敗）は、機能を縮退（degrade）させて処理を続けます。条件ごとの挙動は次のとおりです。

| 条件 | publisher の挙動 |
|---|---|
| `--profile vlm` 無効 | キーを **生成しない**（旧仕様と同じ — Tier 1/2 は従来の挙動にフォールバック） |
| profile 有効 + VLM 呼び出し失敗 | description キーは省略、`image_url_redacted` のみ生成 試行（顔ブラー単体は VLM 不要なため）|
| profile 有効 + 顔ブラー失敗 | redacted 画像キー省略、description のみ。Tier 2 受信者には image 無しで text のみ届く |
| profile 有効 + 両方成功 | 全キー生成 |

degrade 時はレスポンスに **`processing_warnings: ["vlm_unavailable", ...]`** を
含めて、受信側が状況を把握できるようにします。

### VLM の実装選択肢

- **MVP**（最小構成の実装）: ローカル LLaVA (Ollama 経由 / `ollama run llava` を compose service として起動)
- **将来**: より大きなマルチモーダル LLM（Qwen2-VL 等）への差し替え、または external API
  への切り替えを `VLM_BACKEND=ollama|openai|anthropic` 環境変数で制御
- **stub モード** (`VLM_BACKEND=stub`): テスト用。決定的にダミー文字列を返す

### 顔/PII ブラーの実装選択肢

- **MVP**: OpenCV Haar cascade で顔検出 → Gaussian blur → 同 sha256 重複排除で再保存
- **拡張候補**: ナンバープレート（OCR ベース）、画面（鋭い矩形検出）、IDバッジ等を
  順次追加。検出器は `RedactionPipeline` の plug-in として差し込めるよう interface を
  切る

入力の content_type が動画の場合、MVP では **Tier 2 用の redacted は生成しません**
（動画のフレーム抽出とフレームごとのブラーは将来課題です）。動画は Tier 3 にだけ出力し、
Tier 2 には description_full のテキストだけを渡します。

### redact 失敗の検出（研究課題）

VLM は、`description_summary` から PII を完全には除けない場合があります。
本仕様では**漏れの検出を後段のチェック処理（post-check）に分け**、初期実装には含めず
研究課題として記録します。

検出方法の案は次のとおりです。

1. `description_full` から固有名詞（人名・組織名・場所名）を NER で抽出
2. `description_summary` の中に同じ単語が残っていないか diff
3. 残っていれば既知 PII 辞書 / 国別人名辞書と照合し、確度高なら
   `processing_warnings: ["redaction_leak_suspected"]` を立てる
4. 警告ありの行は audit 経路で人手レビュー queue に積む

これはハンズオンの MVP には含めず、仕様に意図だけを記載します。本格運用時に
別途実装する想定です。

### 監査 (audit log) への影響

`audit/logs` に `description_model` と `description_generated_at` が
記録されるので、**どの VLM 出力が誰にいつ何の tier で届いたか**を後から追跡できます。
VLM のアップグレード前後で同じ画像から異なる説明が生成された場合は、
generated_at で区別します。

### テスト方針（追加分）

`tests/test_data_user_vc_tiered_vlm.py` を新設し、次のケースを確認します（括弧内はケース数）。

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

合計 16 ケースを追加し、既存の 88 件とは独立に管理します。このうち、profile が無効なときに
従来の動作が変わらないことの確認を最も重視します。

### 将来課題（spec 上で明記）

- 動画 → フレーム抽出 → VLM 集約による Tier 2/1 動画派生
- VLM 出力の **再現性保証**（同じ image + 同じプロンプトで同じ description が
  返る backend のみ permit するか、generated_at で諦めるか）
- redact post-check の実装（前述）
- VLM 推論の **キャッシュ層**（同じ sha256 image に対して同じ description を
  再利用 — 現状の image dedup と対称な扱い）
- VLM 推論の **コスト課金**（DataUserVC tier に応じて呼び出し回数を制限する等）
