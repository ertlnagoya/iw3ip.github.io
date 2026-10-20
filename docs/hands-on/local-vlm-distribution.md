# ローカル VLM でカメラデータを意味づけ

ラップトップ PC と USB ウェブカメラの映像を、PC 上で動く**ローカル VLM（視覚言語モデル）**で解析して意味テキストを付与し、その AI 加工データを IoT データ流通基盤で配布するまでを行います。画像を外部クラウドに送らず、解析もローカルで完結させます。

> **やること**: ウェブカメラ画像をローカル VLM で解析し、生成した意味データ（説明文）を基盤上で流通させる
>
> **前提**: [環境構築](../setup/index.md) と [最短起動](../setup/quickstart.md) が済んでいること
>
> **使うもの**: ラップトップ PC + USB ウェブカメラ + Docker（VLM を含む）
>
> **所要時間**: 約 40〜60 分（初回はモデルの pull に時間がかかります）

!!! note "このページの位置付け"
    Part 3（知能統合）のハンズオンです。カメラ取り込みの基礎は
    [USB カメラでポイ捨てを検知する (OpenCV)](webcam.md)、VLM による tier 別の出し分けは
    [意味レベルで段階化する](data-user-vc-tiered-semantic.md#12-意味レベルの段階化vlm--顔ブラー) の §12 で扱います。
    本ページは「**カメラ → ローカル VLM → 意味データ流通**」を 1 つの流れとして
    扱います。

## 目的

生の画像を配るのではなく、ローカル VLM が生成した「**意味づけされた派生データ**」を流通させます。詳細記述（`description_full`）と、個人情報を除いた要約（`description_summary`）を作り分け、AI をローカルで使うことでプライバシーとデータ主権を保つ設計を確認します。

## このページで分かること

- ローカル VLM（Ollama）を基盤に組み込み、画像から意味データを生成する流れ
- 生画像ではなく「意味づけした派生データ」を流通させる考え方
- 個人情報を含む詳細記述と、個人情報を除いた (redact 済みの) 要約の作り分け
- 推論に使ったモデルと、VLM が使えず品質を落として処理した場合の通知（`processing_warnings`）を監査できること

## 全体像

```
┌───────────────┐   snapshot.jpg   ┌──────────────────────────┐
│ USB ウェブカメラ │ ───────────────▶ │ publisher (--profile vlm) │
│ + ラップトップ   │                  │  └─ Ollama (llava/moondream)│
└───────────────┘                  └───────────┬──────────────┘
                                               │ description_full / summary
                                               ▼
                                     ┌────────────────────┐
                                     │ /platform/ingest へ │
                                     │ (意味データを流通)     │
                                     └────────────────────┘
```

## 前提

- [最短起動](../setup/quickstart.md) の v1 スタックを起動できる
- [最短起動](../setup/quickstart.md) の IPFS を起動したままの場合は、先に止めておく (`cd ~/program/Blockchain_IoT_Marketplace/ipfs && docker compose down`)。IPFS と publisher がどちらもポート 8080 を使うためです
- USB ウェブカメラが PC から認識されている（[USB カメラでポイ捨てを検知する (OpenCV)](webcam.md) を一度試していると理解が早い）
- 教材リポジトリのトップで Python 3 の仮想環境を作り、OpenCV を入れておく (`cd ~/program/Blockchain_IoT_Marketplace && python3 -m venv .venv && source .venv/bin/activate && pip install "opencv-python<5"`)。新しく開いたターミナルでも `source ~/program/Blockchain_IoT_Marketplace/.venv/bin/activate` を実行します。Homebrew の Python などでは、venv の外で `pip install` するとエラーになります
- Docker で `--profile vlm` が使える（初回はモデル pull で数 GB のダウンロード）

## 最短ルート

最初は次の 4 手順で十分です。

1. `--profile vlm` を付けてサービスを起動し、モデルの pull を待つ
2. ウェブカメラから 1 枚だけ静止画を撮る
3. その画像を基盤に投入し、ローカル VLM に解析させる
4. `/platform/ingest` で、流通した行に `description_*`（意味データ）が付いていることを確認する

## 1. VLM 付きでサービスを起動する

VLM は既定では起動しません。`--profile vlm` を付けると Ollama サービスと、モデルを事前取得する `vlm-pull` が一緒に起動します。教材リポジトリのトップで実行してください（以下は `~/program/` に clone した場合の例です）。

```bash
cd ~/program/Blockchain_IoT_Marketplace
export VLM_BACKEND=ollama
# ラップトップの CPU だけで動かす場合は軽量な moondream を推奨。
# 品質重視かつ時間に余裕があれば llava（既定）でも可。
export VLM_MODEL=moondream
export IMAGE_REDACTION_BACKEND=opencv
PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml --profile vlm up --build -d \
  publisher mosquitto vlm vlm-pull
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

USB ウェブカメラから静止画を 1 枚だけ取得し、`snapshot.jpg` として保存します。下のスクリプトを教材リポジトリのトップ (`~/program/Blockchain_IoT_Marketplace`) に `capture_snapshot.py` として保存し、そこで実行します。OpenCV を使うので、前提で作った venv を有効にします。

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
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
python3 capture_snapshot.py
```

!!! note "カメラが無い / 開けない場合"
    任意の JPEG 画像を `snapshot.jpg` として代用できます。カメラの index や占有で
    失敗する場合は [USB カメラでポイ捨てを検知する (OpenCV)](webcam.md) の `mock` モードや
    トラブル項目を参照してください。

## 3. ローカルモデルに解析させて「意味」を得る

撮影した画像を publisher の `/semantic/analyze` に送ると、ローカルモデルが画像を解析し、**生のピクセルを返さずに**構造化された意味データ（SIR: Semantic Intermediate Representation）だけを返します。ウォレットや VC は不要で、すぐに試せます。

```bash
cd ~/program/Blockchain_IoT_Marketplace
curl -s -X POST http://localhost:8080/semantic/analyze \
  -F "file=@snapshot.jpg" \
  -F "source_device_id=laptop-webcam-01" \
  | jq '{scene_summary, people, sensitive_regions: [.sensitive_regions[].type], events: [.events[].type], privacy_risk_score, analyzer_version}'
```

既定のスタブでは画像によらず次の固定値が返ります。`SEMANTIC_ANALYZER_BACKEND=vision` で起動すると値が画像に応じて変わります。

```json
{
  "scene_summary": "研究室の入口付近に1人の人物がいる（mock出力）",
  "people": {"count": 1, "identities": []},
  "sensitive_regions": ["face", "text", "unknown_sensitive"],
  "events": ["person_detected"],
  "privacy_risk_score": 0.78,
  "analyzer_version": "mock-semantic-analyzer/v1"
}
```

!!! note "アナライザの選択"
    既定は決定的なスタブです。実際の OpenCV 顔検出などを使うには、publisher 起動時に
    `SEMANTIC_ANALYZER_BACKEND=vision` を設定します。§1 のとおり `VLM_BACKEND` を設定して起動していると、
    次の配信ステップで VLM による説明文（`description_*`）も付与されます。

## 4. 意味データを基盤に流通させる

画像を publisher に投入し、その画像を指すイベントを発行して、AI 由来の意味データを流通パイプラインに載せます。次節の演習プログラムの解答 (`answer_program.py`) に `--distribute` を付けて実行すると、consent 登録 → `/media/upload` → `/simulate/publish` を一括で行います。§5 の確認の前に、§2 で撮った `snapshot.jpg` で実行しておきます。

```bash
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
python3 examples/hands_on/local_vlm_distribution/answer_program.py \
  --base-url http://localhost:8080 --image snapshot.jpg --distribute
```


§1 のとおり `VLM_BACKEND` を設定して起動していれば、publisher はこのとき VLM を呼び出し、行に `description_full` / `description_summary` / `description_model` / `description_generated_at` を付与します。VLM や顔ブラーが失敗したときだけ、`processing_warnings` (`vlm_unavailable` など) も付きます。

!!! info "受信側での取得には ViewerToken が必要"
    付与された `description_*` を受信側で取り出すには、ViewerToken が必要です。ViewerToken は、
    `/marketplace/claim` で発行された PurchaseViewerVC を提示して得ます。どの tier の PurchaseViewerVC になるかは、claim に渡す `data_user_attrs` で決まります ([信頼度に応じて見せる中身を変える](data-user-vc-tiered.md#3-marketplaceclaim-を-3-通り呼び出す) の §3)（`/platform/data` は素の GET では 401 になります）。信頼度
    ごとに `description_full` / `description_summary` を出し分ける流れは
    [意味レベルで段階化する](data-user-vc-tiered-semantic.md#12-意味レベルの段階化vlm--顔ブラー) の §12 で扱います。本ページで扱うのは
    「カメラ → ローカルモデルで意味づけ → 基盤へ流通」までです。

## 5. 確認ポイント

- `/semantic/analyze` が生画像を返さず、構造化された意味データ（SIR）だけを返している
- クラウドに画像を送らず、ローカルモデルだけで意味づけできている
- `VLM_BACKEND` を設定して起動したとき、配信した行に `description_*`（説明文とモデル ID）が付与される。§1 で `PLATFORM_INGEST_READ_ENABLED=true` を付けて起動しているので、次のコマンドで最後に流通した行を確かめられる (ViewerToken なしで中身が見えるのは、この確認用の設定のときだけ)

    ```bash
    curl -s http://localhost:8080/platform/ingest | jq '.[-1] | {description_full, description_summary, description_model, processing_warnings}'
    ```

## 考えてみよう

- 生の画像そのものではなく「意味づけした派生データ」を流通させると、プライバシーとデータ主権の観点で何が変わるでしょうか。
- ローカル実行（クラウド API を使わない）ことの利点と制約（速度・精度・運用）を整理してみてください。

## 演習で手を動かす

初めての人でも段階的に進められるよう、演習プログラムを用意しています。**まず全体構造を読み（Step A）→ 雛形の穴埋め（Step B）→ 自分でゼロから機能追加（Step C）**の順に進めてください。

演習用プログラム:

- [問題用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/local_vlm_distribution/problem_program.py)（TODO 付き）
- [解答用プログラム](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/local_vlm_distribution/answer_program.py)
- [演習の説明（全体構造の解説つき）](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/hands_on/local_vlm_distribution/README.md)

### Step A. 全体構造とプログラムを読む

README の「Big picture」で、3 つの関数の役割を押さえます。`analyze_frame()`（`/semantic/analyze` に送って SIR を得る）、`summarize_sir()`（SIR を 1 行に要約）、`distribute_frame()`（consent 登録 → アップロード → 発行）。「画像は外に出さず、意味だけを出す」流れを理解するのが目的です。

### Step B. 雛形を穴埋めする（課題）

`problem_program.py` の 2 つの TODO を実装します。

- **TODO 1 `summarize_sir()`**: SIR から人数・物体数・機微領域数・シーン要約を読み、1 行の要約を返す
- **TODO 2 `build_event()`**: アップロードした画像（`image_url`）を運ぶイベントペイロードを組み立てる

```bash
cd ~/program/Blockchain_IoT_Marketplace
source .venv/bin/activate
python3 examples/hands_on/local_vlm_distribution/problem_program.py \
  --base-url http://localhost:8080 --image snapshot.jpg
```

`answer_program.py` と見比べて確認します。`--distribute` を付けると流通まで実行します。

!!! note "`--distribute` がタイムアウトしたとき"
    `/simulate/publish` は VLM の推論が終わってから応答します。CPU だけの PC では推論に数十秒から数分かかるため、プログラム側の待ち時間を超えて `TimeoutError` などで終わることがあります (待ち時間は既定 600 秒で、環境変数 `HANDS_ON_HTTP_TIMEOUT` で変えられます。古い教材リポジトリでは 10 秒なので `git pull` します)。publisher 側の処理は続いているので、行は推論が終わったあとに記録されます。次のコマンドで、`raw_topic` が `homeassistant/event/possible_littering` で `action` が `allow` の行が増えたことを確認します（`/platform/data` で確かめるには ViewerToken が必要です）。

    ```bash
    curl -s localhost:8080/audit/logs \
      | jq '[.[] | select(.raw_topic == "homeassistant/event/possible_littering")][0] | {ts, action, reason}'
    ```

### Step C. ゼロから機能を足す（発展課題）

雛形なしで、次のいずれかを自分で実装します。

- **要約の高度化**: `sensitive_regions[].type` を列挙し、`privacy_risk_score` が高いとき警告する
- **フレーム選別**: 一定間隔で N 枚撮り、最も privacy_risk の高い 1 枚だけ流通させる
- **新しい派生フィールド**: SIR から計算した値（例 `people_count`）をイベント `data` に足す
- **独自アナライザ**: `publisher/app/semantic_analyzer.py` に新しい `SemanticAnalyzer` を実装し、`SEMANTIC_ANALYZER_BACKEND` で差し替える

## さらに先へ（機能拡張の方向）

- **自然言語での検索・加工**: 蓄積した意味データに対し、LLM で自然言語検索や要約を行う（[AI に手順を考えさせる (LLM Planner)](llm-planner.md) / [地域安全アシスタントを動かす](regional-safety-assistant.md)）。
- **機器の遠隔操作**: 解析結果を条件に、`plan` と `execute` を分けて機器を操作する（同上 Part 3）。
- **ラズベリーパイ + ラズパイカメラ**: 撮影はラズパイ、推論は PC / ホスト側、という分担で実機化する（本ページはまずラップトップで完結させる構成です）。

## よくある問題

- 初回のモデル pull が重く時間がかかる → `vlm-pull` のログで取得完了を待つ。回線に余裕のあるときに実施する。
- CPU で推論が遅い / タイムアウトする → 軽量な `VLM_MODEL=moondream` を使う、`VLM_TIMEOUT_SEC` を延ばす。
- `processing_warnings` に `vlm_unavailable` が出る → `vlm` サービスが起動しているか、モデル pull が完了しているかを確認（`docker compose -f infra/docker-compose.yml --profile vlm ps` / `docker compose -f infra/docker-compose.yml logs vlm-pull`）。
- カメラが開けない → カメラ index の変更や占有プロセスの停止、macOS ではターミナルのアプリへのカメラの許可（[USB カメラでポイ捨てを検知する (OpenCV)](webcam.md) を参照）。

## 関連ページ

- 前提: [USB カメラでポイ捨てを検知する (OpenCV)](webcam.md) / [最短起動](../setup/quickstart.md)
- 詳細: DataUserVC × 段階アクセス（[§8 メディア統合](data-user-vc-tiered-media.md#8-実データ統合案-bhttp-メディアゲートウェイ)・[§12 VLM tier](data-user-vc-tiered-semantic.md#12-意味レベルの段階化vlm--顔ブラー)） / [AI に手順を考えさせる (LLM Planner)](llm-planner.md)
- 設計: [段階アクセスの設計仕様（tier 拡張: 意味レベルの段階化）](data-user-vc-tiered-spec.md)
