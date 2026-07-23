# ローカル VLM でカメラデータを意味づけして流通する (Part 3)

ラップトップ PC と USB ウェブカメラの映像を、PC 上で動く**ローカル VLM（視覚言語モデル）**で解析して意味テキストを付与し、その AI 加工データを IoT データ流通基盤で配布するまでを体験します。画像そのものを外部クラウドに送らず、解析もローカルで完結させる点が要点です。

> **やること**: ウェブカメラ画像をローカル VLM で解析し、生成した意味データ（説明文）を基盤上で流通させる
>
> **前提**: [環境構築](../setup/index.md) と [最短起動](../setup/quickstart.md) が済んでいること
>
> **使うもの**: ラップトップ PC + USB ウェブカメラ + Docker（VLM を含む）
>
> **所要時間**: 40〜60 分（初回はモデルの pull に時間がかかります）

!!! note "このページの位置付け"
    Part 3（知能統合）のハンズオンです。カメラ取り込みの基礎は
    [USB ウェブカメラサンプル](webcam.md)、VLM による tier 別の出し分けは
    [DataUserVC × 段階アクセス](data-user-vc-tiered.md) の §12 で扱います。
    本ページは「**カメラ → ローカル VLM → 意味データ流通**」を 1 本の流れとして
    体験することに集中します。

## 目的

生の画像を配るのではなく、ローカル VLM が生成した「**意味づけされた派生データ**」を流通させます。詳細記述（`description_full`）と、個人情報を除いた要約（`description_summary`）を作り分け、AI をローカルで使うことでプライバシーとデータ主権を保つ設計を体験します。

## このページで分かること

- ローカル VLM（Ollama）を基盤に組み込み、画像から意味データを生成する流れ
- 生画像ではなく「意味づけした派生データ」を流通させる考え方
- 個人情報を含む詳細記述と、redact 済み要約の作り分け
- 推論に使ったモデルや失敗時の degrade 通知（`processing_warnings`）が監査可能であること

## 全体像

```
┌───────────────┐   snapshot.jpg   ┌──────────────────────────┐
│ USB ウェブカメラ │ ───────────────▶ │ publisher (--profile vlm) │
│ + ラップトップ   │                  │  └─ Ollama (llava/moondream)│
└───────────────┘                  └───────────┬──────────────┘
                                               │ description_full / summary
                                               ▼
                                     ┌────────────────────┐
                                     │ /platform/data で受信 │
                                     │ (意味データを流通)     │
                                     └────────────────────┘
```

## 前提

- [最短起動](../setup/quickstart.md) の v1 スタックを起動できる
- USB ウェブカメラが PC から認識されている（[USB ウェブカメラサンプル](webcam.md) を一度試していると理解が早い）
- Docker で `--profile vlm` が使える（初回はモデル pull で数 GB のダウンロード）

## 最短ルート

最初は次の 4 手順で十分です。

1. `--profile vlm` を付けてサービスを起動し、モデルの pull を待つ
2. ウェブカメラから 1 枚だけ静止画を撮る
3. その画像を基盤に投入し、ローカル VLM に解析させる
4. `/platform/data` で `description_*`（意味データ）が流通していることを確認する

## 1. VLM 付きでサービスを起動する

VLM は opt-in です。`--profile vlm` を付けると Ollama サービスと、モデルを事前取得する `vlm-pull` が一緒に起動します。

```bash
cd ~/program/Blockchain_IoT_Marketplace
export VLM_BACKEND=ollama
# ラップトップの CPU だけで動かす場合は軽量な moondream を推奨。
# 品質重視かつ時間に余裕があれば llava（既定）でも可。
export VLM_MODEL=moondream
export IMAGE_REDACTION_BACKEND=opencv
docker compose -f infra/docker-compose.yml --profile vlm up -d \
  publisher hardhat bridge mosquitto vlm vlm-pull
```

`vlm-pull` がモデルを取得し終わるまで待ちます（初回のみ、数 GB）。

```bash
docker compose -f infra/docker-compose.yml logs -f vlm-pull
# モデル取得が終わったらログから抜ける
```

publisher の起動を確認します。

```bash
curl -s http://localhost:8080/health | jq .
# -> {"status": "ok", ...}
```

!!! tip "モデルの選び方"
    `llava` は記述が詳しい一方、CPU のみだと 1 枚あたり数分かかります。`moondream`
    は 1.7 GB 程度と軽く、CPU でも数十秒〜数分で応答します。まずは `moondream`
    で流れを通し、余裕があれば `VLM_MODEL=llava` に切り替えて品質差を比べてください。
    応答が遅い環境では `VLM_TIMEOUT_SEC`（既定 180 秒）を延ばします。

## 2. ウェブカメラから 1 枚撮る

USB ウェブカメラから静止画を 1 枚だけ取得し、`snapshot.jpg` として保存します。

```python
# capture_snapshot.py
import cv2

cap = cv2.VideoCapture(0)          # カメラが複数あるときは 0 を 1, 2... に変える
ok, frame = cap.read()
cap.release()
if not ok:
    raise SystemExit("カメラから取得できませんでした（index や占有状況を確認）")
cv2.imwrite("snapshot.jpg", frame)
print("saved snapshot.jpg")
```

```bash
python capture_snapshot.py
```

!!! note "カメラが無い / 開けない場合"
    任意の JPEG 画像を `snapshot.jpg` として代用できます。カメラの index や占有で
    失敗する場合は [USB ウェブカメラサンプル](webcam.md) の `mock` モードや
    トラブル項目を参照してください。

## 3. 画像を基盤に投入してローカル VLM に解析させる

撮影した画像を publisher のメディアゲートウェイに投入します。VLM プロファイルが有効なので、publisher が Ollama を呼び出して説明文を生成します。

```bash
python examples/hands_on/data_user_vc_tiered/provider_with_media.py \
  --base-url http://localhost:8080 \
  --image snapshot.jpg
```

publisher は取り込んだ画像に対し、ローカル VLM で 2 種類の説明文を生成します（CPU だと 1 枚あたり数十秒〜数分かかります）。

- `description_full` … 人名・物体名・文字なども含む詳細な記述
- `description_summary` … 個人情報を除いた概要（PII redact 済み）

## 4. 意味データが流通しているか確認する

`/platform/data` を取得し、VLM が生成した意味データが乗っていることを確認します。

```bash
curl -s http://localhost:8080/platform/data | jq .
```

見る点:

| キー | 内容 |
|---|---|
| `description_full` | 詳細記述（人名・固有名詞あり） |
| `description_summary` | 概要（PII redact 済み） |
| `description_model` | 推論に使った VLM のモデル ID（監査用） |
| `description_generated_at` | 推論時刻（ISO8601、監査用） |
| `processing_warnings` | 推論やブラーが失敗したステップ（例 `vlm_unavailable`） |

生の画像を配らずに、**意味づけされたデータが基盤上で流通している**ことを確認できれば、このハンズオンのコアは完了です。信頼度に応じて `description_full` / `description_summary` を出し分ける tier 制御は [DataUserVC × 段階アクセス §12](data-user-vc-tiered.md) で扱います。

## 5. 確認ポイント

- `description_summary` が、生画像を出さずに内容の意味を伝えている
- `description_model` と `description_generated_at` に推論の出所と時刻が記録されている
- クラウドに画像を送らず、ローカル VLM だけで意味データを生成・流通できている

## 考えてみよう

- 生の画像そのものではなく「意味づけした派生データ」を流通させると、プライバシーとデータ主権の観点で何が変わるでしょうか。
- ローカル実行（クラウド API を使わない）ことの利点と制約（速度・精度・運用）を整理してみてください。

## 発展課題

- **自然言語での検索・加工**: 蓄積した意味データに対し、LLM で自然言語検索や要約を行う（[LLM Planner](llm-planner.md) / [地域安全アシスタント](regional-safety-assistant.md)）。
- **機器の遠隔操作**: 解析結果を条件に、`plan` と `execute` を分けて機器を操作する（同上 Part 3）。
- **ラズベリーパイ + ラズパイカメラ**: 撮影はラズパイ、VLM 推論は PC / ホスト側、という分担で実機化する（本ページはまずラップトップで完結させる構成です）。

## よくある問題

- 初回のモデル pull が重く時間がかかる → `vlm-pull` のログで取得完了を待つ。回線に余裕のあるときに実施する。
- CPU で推論が遅い / タイムアウトする → 軽量な `VLM_MODEL=moondream` を使う、`VLM_TIMEOUT_SEC` を延ばす。
- `processing_warnings` に `vlm_unavailable` が出る → `vlm` サービスが起動しているか、モデル pull が完了しているかを確認（`docker compose ... ps` / `logs`）。
- カメラが開けない → カメラ index の変更や占有プロセスの停止（[USB ウェブカメラサンプル](webcam.md) を参照）。

## 関連ページ

- 前提: [USB ウェブカメラサンプル](webcam.md) / [最短起動](../setup/quickstart.md)
- 深掘り: [DataUserVC × 段階アクセス（§8 メディア統合・§12 VLM tier）](data-user-vc-tiered.md) / [LLM Planner](llm-planner.md)
- 設計: [DataUserVC × 段階アクセス制御 仕様（tier 拡張: 意味レベルの段階化）](data-user-vc-tiered-spec.md)
