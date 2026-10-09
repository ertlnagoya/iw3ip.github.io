# 意味レベルで段階化する (VLM と意味的中間表現 / Stage T)

[信頼度に応じて見せる中身を変える](data-user-vc-tiered.md)（§0〜§7）と [画像・動画を段階アクセスで配信する](data-user-vc-tiered-media.md)（§8〜§11）の続きです。映像を意味的中間表現に変換し、受信者の信頼度に応じて出力を変えるレンダリング（trust-aware rendering）までを扱います。節番号は 1 ページ目からの通し番号（§12〜§13）です。

> **やること**: VLM による推論と顔 / PII のブラー処理で派生データを生成して tier 別に出し分け、意味的中間表現 (SIR) による信頼度別レンダリングを動かす
>
> **前提**: [信頼度に応じて見せる中身を変える](data-user-vc-tiered.md) の §0〜§7 と、[画像・動画を段階アクセスで配信する](data-user-vc-tiered-media.md) の §8〜§11 を済ませていること
>
> **使うもの**: PC + スマホ (iw3ip-wallet)、`--profile vlm` で起動する publisher（Ollama）

## 12. 意味レベルの段階化（VLM + 顔ブラー）

§1〜§11（[1 ページ目](data-user-vc-tiered.md)と [2 ページ目](data-user-vc-tiered-media.md)）のアクセス制御は、**メディアのキーを応答から省く**方式でした（Tier 2 では
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

[§2](data-user-vc-tiered.md#2-3-種類の-datauservc-オファーを作る) の 3 種類に **summary tier 用**を追加します。

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

VLM 拡張を有効にすると tier は 4 段階になり、summary が Tier 1、denied が Tier 0 になります。[§2c](data-user-vc-tiered.md#2c-tier-1deniedプロファイル--企業--研究--ポリシーなし--濫用記録あり) の「Tier 1（denied）」は、VLM 拡張を使わない 3 段階の場合の呼び方で、同じプロファイルを指します。

```bash
curl -s -X POST 'localhost:8080/issuer/offer?vc_kind=DataUserVC&entity_type=Enterprise&purpose=Research&legal_compliance=false&data_handling_policy=Other&misuse_record=true' | jq .
```

### 12.4 4 通りの `/marketplace/claim` と `/platform/data` 比較

[§3〜§4](data-user-vc-tiered.md#3-marketplaceclaim-を-3-通り呼び出す) と同じ要領で、4 通りそれぞれについて claim、PurchaseViewerVC の提示、ViewerToken の取得、
`/platform/data` の取得を行います。VLM profile が有効なとき、レスポンスに含まれるキーは次のように変わります。

| プロファイル | `event` | `image_url` | `video_url` | `image_url_redacted` | `description_full` | `description_summary` |
|---|---|---|---|---|---|---|
| 12.3.a Tier 3 (full) | あり | あり | あり | あり | あり | あり |
| 12.3.b Tier 2 (access) | あり | **なし** | **なし** | あり | あり | あり |
| 12.3.c Tier 1 (summary) | あり | **なし** | **なし** | **なし** | **なし** | あり |
| 12.3.d Tier 0 (denied) | claim 自体が `access_level: "denied"` で拒否 |

profile が無効なら従来どおり（[§4](data-user-vc-tiered.md#4-purchaseviewervc-を発行--提示--platformdata) と同じ）3 段階の tier 投影になり、新しいキーは応答に含まれません。
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

[§11](data-user-vc-tiered-media.md#11-pwa-providerデータ提供者向け-pwa) の `/provider` ページからアップロードして Publish した画像は、profile vlm が
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

[§11.8](data-user-vc-tiered-media.md#118-実機検証ログ) と同じ形式で、VLM + 顔ブラーの実機検証ログを残します。

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

[§11.3](data-user-vc-tiered-media.md#113-データを提供する-3-つの方法) でメディアをアップロードした直後に、`/provider` ページに **「1.5 意味的中間表現を確認」**
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
追加されます。OFF（既定）では [§10](data-user-vc-tiered-media.md#10-pwa-viewerスマホ--pc-共通-ux) と同じ従来の tier 投影で表示します。ON では次のように動作します。

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

[§11.8](data-user-vc-tiered-media.md#118-実機検証ログ) / §12.10 と同じ形式で記録します。

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

- 戻る: [画像・動画を段階アクセスで配信する](data-user-vc-tiered-media.md) — §8〜§11（画像・動画の配信）
- 最初から: [信頼度に応じて見せる中身を変える](data-user-vc-tiered.md) — §0〜§7（DataUserVC の発行と tier による出し分け）
