# 地域安全アシスタントサンプル（Phase 3）

この Hands-on は、既存の [USBウェブカメライベント共有サンプル（Phase 2）](webcam-event-sharing.md) をさらに発展させ、**人間の要求を AI が解釈し、必要なイベント収集・条件判定・機器操作へ分解する**流れを体験するためのものです。

Phase 2 では `possible_littering` のようなイベントを共有しました。  
Phase 3 では、その先として、次の流れを扱います。

`人間の要求 -> planner -> 実行計画 -> イベント評価 -> 機器操作`

## この Hands-on で学べること

- 自然言語の要求を構造化された実行計画へ変換する考え方
- Phase 2 のイベント共有を、Phase 3 の判断・制御へ拡張する方法
- `plan` と `execute` を分けて設計する理由
- 実行履歴を後から確認できるように残す重要性

## 前提

- Docker / Docker Compose が使える
- `curl` が使える
- ソースコードリポジトリ `Blockchain_IoT_Marketplace` の `codex/phase3-safety-assistant-sample` ブランチを使う

参考:

- [Docker 公式サイト](https://docs.docker.com/get-docker/)
- [FastAPI 公式サイト](https://fastapi.tiangolo.com/)

## 対応するソースコード

この Hands-on では、次のファイルを使います。

- [assistant/app/main.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase3-safety-assistant-sample/assistant/app/main.py)
- [assistant/app/planner.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase3-safety-assistant-sample/assistant/app/planner.py)
- [assistant/app/evaluator.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase3-safety-assistant-sample/assistant/app/evaluator.py)
- [assistant/app/actuator.py](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase3-safety-assistant-sample/assistant/app/actuator.py)
- [examples/phase3_request_park_safety.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase3-safety-assistant-sample/examples/phase3_request_park_safety.json)
- [examples/phase3_events_park_safety.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/codex/phase3-safety-assistant-sample/examples/phase3_events_park_safety.json)

今回は Phase 3 用の `問題用プログラム / 解答用プログラム` ではなく、**最小実装そのもの**を読む形にしています。  
理由は、Phase 3 では「planner」「evaluator」「actuator」のモジュール境界自体が学習対象だからです。

## 想定シナリオ

利用者は次のように依頼します。

> 公園北側でポイ捨てや危険行動が増えていたら教えて。必要なら照明をつけて管理者に通知して。

この要求に対して、assistant は次のように考えます。

1. 対象場所は `park-north`
2. 注目するイベントは `possible_littering` と `suspicious_activity`
3. しきい値を超えたら `light_on` と `send_notification` を実行する

## 1. assistant サービスを起動

`Blockchain_IoT_Marketplace` リポジトリで次を実行します。

```bash
docker compose -f infra/docker-compose.yml --profile assistant up --build -d assistant
```

確認:

```bash
curl http://localhost:8090/health
```

期待結果:

```json
{"status":"ok","service":"assistant"}
```

もし `Connection refused` が出る場合:

- コンテナ起動に時間がかかっている可能性があります
- `docker ps` で `assistant` が `Up` か確認してください
- `docker compose -f infra/docker-compose.yml --profile assistant logs assistant` でログも確認できます

## 2. 計画生成を確認

まず、自然言語要求がどのような計画に変換されるかを見ます。

```bash
curl -X POST http://localhost:8090/assistant/plan \
  -H 'Content-Type: application/json' \
  -d @examples/phase3_request_park_safety.json
```

期待結果の例:

```json
{
  "status": "planned",
  "plan": {
    "intent": "monitor_public_safety",
    "target_area": "park-north",
    "watch_events": ["possible_littering", "suspicious_activity"],
    "actions": ["light_on", "send_notification"]
  }
}
```

確認ポイント:

- `target_area` が `park-north` になっている
- `watch_events` に `possible_littering` が入っている
- `actions` に `light_on` と `send_notification` が入っている

## 3. 実行を確認

次に、サンプルイベントを使って実際に判定と制御を走らせます。

```bash
curl -X POST http://localhost:8090/assistant/execute \
  -H 'Content-Type: application/json' \
  -d @examples/phase3_request_park_safety.json
```

期待結果の例:

```json
{
  "status": "executed",
  "execution": {
    "request_text": "公園北側でポイ捨てや危険行動が増えていたら教えて。必要なら照明をつけて管理者に通知して。",
    "evaluation": {
      "triggered": true
    },
    "actions_executed": [
      {"action": "light_on", "status": "executed"},
      {"action": "send_notification", "status": "executed"}
    ]
  }
}
```

この結果は、サンプルイベントファイルに `possible_littering` が 3 件含まれているため、しきい値を満たしていることを意味します。

## 4. 実行履歴を確認

```bash
curl http://localhost:8090/assistant/executions
```

確認ポイント:

- 元の `request_text`
- 生成された `plan`
- 評価結果 `evaluation`
- 実行された `actions_executed`

が 1 つの履歴として残ります。

## 5. Phase 2 と Phase 3 の違いを整理

Phase 2 では、主に次を扱いました。

- イベントを共有する
- Consent や policy で `allow` / `deny` を判定する

Phase 3 では、その上に次が追加されます。

- 人間の要求を解釈する
- 必要なイベントを選ぶ
- 判定条件をまとめる
- 条件成立時に機器操作を行う

つまり、Phase 3 は **「イベントを共有する」から「要求に応じて判断し、行動する」へ進む段階**です。

## 6. 成功判定

この Hands-on では、次を確認できれば成功です。

- `/assistant/plan` が自然言語要求を計画へ変換する
- `/assistant/execute` が `triggered: true` を返す
- `light_on` と `send_notification` が `executed` になる
- `/assistant/executions` で履歴を確認できる

## 7. よくあるつまずき

### `examples/phase3_request_park_safety.json` が見つからない

- 実行ディレクトリが `Blockchain_IoT_Marketplace` のルートか確認してください

### 期待した action が実行されない

- `examples/phase3_events_park_safety.json` のイベント件数がしきい値未満の可能性があります
- `possible_littering` の件数と `location` を確認してください

### どこが AI なのか分かりにくい

この最小実装では、planner はまだルールベースです。  
ただし、**自然言語要求を構造化計画へ変換するモジュール境界**を先に作っているため、将来的に LLM や外部 planner に差し替えやすくしています。

## 8. 停止

```bash
docker compose -f infra/docker-compose.yml --profile assistant down
```

## 次の学習

- Phase 2 からつなげて理解したい場合: [USBウェブカメライベント共有サンプル（Phase 2）](webcam-event-sharing.md)
- Phase 1 の基礎から見直したい場合: [HA x SSI Publisherサンプル（Phase 1）](ha-ssi-publisher.md)
