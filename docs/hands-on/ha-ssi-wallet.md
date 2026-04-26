# スマホSSIウォレットサンプル（Phase 2）

!!! note "このページは構築予定のハンズオンです"
    本ページは、Sphereon mobile-wallet を ertlnagoya 配下へ fork した
    `iw3ip-wallet` を用いた VC 検証デモの設計ドラフトです。
    対応するアプリ・バックエンド実装は準備中です。

## 目的

スマホの SSI ウォレットで受け取った Consent VC を、IW3IP バックエンドが
OID4VP で検証し、検証が通った要求に対してのみイベントデータを共有する
流れを体験します。

従来の [HA x SSI Publisher サンプル](ha-ssi-publisher.md) では Consent VC
相当のポリシー JSON を `/consents` に直接登録していました。本サンプルでは、
**ウォレットが保持する VC を提示 → 検証 → allowed/denied** という
本来の SSI モデルを扱います。

パイプライン:

`スマホウォレット (VC保持) -> QR/Deeplink -> OID4VP Verifier -> Platform API -> イベント共有`

## このページで分かること

- ウォレットに VC を発行（OID4VCI）し、提示（OID4VP）する基本フロー
- Verifier 側の Presentation Definition（PEX）とウォレット側の応答対応
- `did:jwk` / `did:key` と SD-JWT VC を用いた軽量構成
- `allowed` / `denied` が VC の内容と `purpose` によって決まる様子

## つまずきやすい点

- Presentation Definition の `input_descriptors` と VC の claim が食い違うと、
  ウォレットが「提示可能な VC がない」と判定する
- DID メソッドが issuer / wallet / verifier でずれると署名検証に失敗する
- SD-JWT VC・W3C VC-JWT・mdoc は非互換。1 形式に固定する
- スマホと PC が別ネットワークだと QR 経由の redirect が届かない
  （[スマホ閲覧アプリ](mobile-viewer.md) と同じ前提）

## 公式リンク

- Sphereon mobile-wallet（upstream）: <https://github.com/Sphereon-Opensource/mobile-wallet>
- `@sphereon/pex`（Presentation Exchange）: <https://github.com/Sphereon-Opensource/pex>
- OpenID for Verifiable Presentations 仕様: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- OpenID for Verifiable Credential Issuance 仕様: <https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html>
- SD-JWT VC: <https://www.ietf.org/archive/id/draft-ietf-oauth-sd-jwt-vc-latest.html>

## 前提

- Docker / Docker Compose が使える
- PC とスマホが同じ LAN に接続されている
- スマホに `iw3ip-wallet`（fork 版）のビルドをインストールできる
  （TestFlight / 内部配布 APK / Expo dev build のいずれか）
- [HA x SSI Publisher サンプル](ha-ssi-publisher.md) の構成を一度起動できている

## 関連リポジトリ

- `iw3ip-wallet` (fork 予定): `https://github.com/ertlnagoya/iw3ip-wallet`（準備中）
  - upstream: Sphereon-Opensource/mobile-wallet
  - ブランチ方針: `main` は upstream 追従、`iw3ip/*` で IW3IP 固有改変
- `iw3ip-verifier`（Publisher 内に追加する OID4VP エンドポイント、準備中）

## 最短ルート

最初は次の 5 手順を想定しています。

1. Publisher 側で Issuer / Verifier プロファイルを起動する
2. スマホで `iw3ip-wallet` を開き、Issuer の QR から Consent VC を受領する
3. PC 画面に表示される Verifier QR をスマホで読み取る
4. ウォレットが要求に合致する VC を選んで提示する
5. Publisher 側 `/audit/logs` に `allow` が残ることを確認する

その後の分岐:

- 拒否ケースを見たい場合: `purpose` が合わない要求に変えて再提示する
- 失効を見たい場合: VC を revoke → 同じ提示で `denied` になることを確認する
- Phase 1 との違いを見たい場合: `/consents` への直接登録ではなく
  提示経由になった点を監査ログで比較する

## 工程別の目次

<details class="iw3ip-toc-details" open>
  <summary>準備: Publisher / Issuer / Verifier を起動する</summary>
  <p>まず Publisher に OID4VCI / OID4VP 相当のエンドポイントを追加したプロファイルで起動します。</p>
  <ol>
    <li><a href="#1-起動">起動</a></li>
    <li><a href="#2-presentation-definitionを確認">Presentation Definition を確認</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 1: VC を発行してウォレットに保存する</summary>
  <p>Issuer の QR からスマホウォレットへ Consent VC を発行し、保有状態を確認します。</p>
  <ol>
    <li><a href="#3-ウォレットを起動">ウォレットを起動</a></li>
    <li><a href="#4-issuer-qrで発行">Issuer QR で発行</a></li>
  </ol>
</details>

<details class="iw3ip-toc-details">
  <summary>確認 2: VC を提示し allowed と denied を比較する</summary>
  <p>Verifier QR を読み取り、`purpose` が一致する場合と一致しない場合の判定差を確認します。</p>
  <ol>
    <li><a href="#5-verifier-qrで提示">Verifier QR で提示</a></li>
    <li><a href="#6-拒否ケース">拒否ケース</a></li>
    <li><a href="#7-監査ログ確認">監査ログ確認</a></li>
  </ol>
</details>

## 読み進め方

Phase 2 のうち、Consent VC を「登録済みポリシー」ではなく
「ウォレット提示物」として扱う段階です。既存の
[HA x SSI Publisher サンプル](ha-ssi-publisher.md) と
[環境・防災イベント共有サンプル](environment-disaster.md) を一度通してから
本ページに進むと、差分が理解しやすくなります。

## 1. 起動

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet up --build -d
```

確認:

```bash
curl http://localhost:8080/health
curl http://localhost:8080/.well-known/openid-credential-issuer
```

## 2. Presentation Definition を確認

```bash
curl http://localhost:8080/verifier/presentation-definitions/consent-temperature
```

確認ポイント:

- `input_descriptors` に `dataset_id` と `purpose` の制約が含まれる
- 形式が `vc+sd-jwt` であること

## 3. ウォレットを起動

`iw3ip-wallet` をスマホで起動し、初回ログインを完了します。
初期鍵は `did:jwk` として生成されます。

## 4. Issuer QR で発行

PC ブラウザで以下を開きます。

```txt
http://<PCのLAN_IP>:8080/issuer/offer?type=ConsentVC&dataset_id=home/env/temperature&purpose=research
```

表示された QR をスマホウォレットから読み取り、提示同意 → VC を保存します。

## 5. Verifier QR で提示

PC ブラウザで以下を開きます。

```txt
http://<PCのLAN_IP>:8080/verifier/request?dataset_id=home/env/temperature&purpose=research
```

表示された QR をウォレットで読み取り、該当する VC を選択して提示します。

期待結果:

```json
{
  "status": "allowed",
  "dataset_id": "home/env/temperature",
  "policy_token": "VHA9X1d...",
  "policy_token_jti": "9b2fc3...",
  "expires_in": 300
}
```

`policy_token` は次節 §8 で使う短命の認可トークンです。
有効期間は 5 分・単回消費で、`/platform/ingest` を 1 回呼び出すと無効化されます。

## 6. 拒否ケース

`purpose` を `marketing` に変えて同じ提示を実行します。

期待結果:

```json
{"status":"denied","dataset_id":"home/env/temperature","reason":"purpose_mismatch"}
```

ウォレットが同じ VC を持っていても、Verifier 側の要求条件に合わなければ
拒否される、という点がポイントです。

## 7. 監査ログ確認

```bash
curl http://localhost:8080/audit/logs?limit=10
```

確認ポイント:

- `presentation_verified` イベントが `allow` / `deny` と共に記録される
- `holder_did`、`vc_hash`、`purpose` が残る
- HA x SSI Publisher サンプルの監査ログと比べ、ポリシー判定ではなく
  「誰がどの VC を提示したか」が追加で残る

## 8. 共有データを取得する

§5 で得た `policy_token` を `Authorization: Bearer` ヘッダに付けて
`/platform/ingest` を呼び出すと、検証済みの VC に対応するデータ共有が成立します。
従来の `/consents` JSON 登録経路はヘッダ無しで従来通り動きます。

PolicyToken の性質:

- 有効期限: 5 分（`expires_in` で返却）
- 単回消費: 1 回 `/platform/ingest` を通すと無効化される
- スコープ: 発行時の `dataset_id` と一致する body のみ受け付ける
- 形式: 不透明文字列（サーバ側 in-memory 管理）

リクエスト例:

```bash
TOKEN=<policy_token>  # §5 の verifier レスポンスから取得

curl -X POST http://<PCのLAN_IP>:8080/platform/ingest \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":"home/env/temperature","purpose":"research","value":21.4}'
```

期待結果:

```json
{"status":"received","count":1}
```

監査ログには PolicyToken 消費イベントが追加されます。

```bash
curl 'http://<PCのLAN_IP>:8080/audit/logs?limit=5'
```

```json
{
  "action": "allow",
  "raw_topic": "platform/ingest",
  "reason": "policy_token_consumed:9b2fc3...",
  "dataset_id": "home/env/temperature",
  "purpose": "research",
  "holder_did": "did:jwk:...",
  "presentation_verified": "allow"
}
```

エラーケース:

| 状況 | HTTP | `detail` |
| --- | --- | --- |
| 同じトークンで 2 回目を呼ぶ | 403 | `policy_token_already_consumed` |
| 期限切れ | 401 | `policy_token_expired` |
| body の `dataset_id` がトークンと一致しない | 403 | `policy_token_dataset_mismatch` |
| 未知のトークン | 401 | `policy_token_unknown` |

トークンを取り出すには、ウォレットからの提示完了時に
publisher コンテナのログ（`/verifier/response` のレスポンスボディ）を確認するか、
スマホ側で表示される完了画面を読み取ります。
ハンズオンでは publisher ログから JSON を拾って curl に渡すのが手早いです。

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet logs -f publisher | grep policy_token
```

## 拡張ヒント

- **失効**: Status List 2021 を `/verifier/status` で提供し、revoke 後に
  同じ提示が `denied` になることを確認する
- **SIOPv2 による holder 認証**: 提示と同時に holder の DID を検証する
- **Phase 3 連携**: [LLM Planner ハンズオン](llm-planner.md) の plan 実行前に
  VC 提示を要求し、知能統合層での権限チェックに拡張する

## トラブル時

- 症状: ウォレットが「提示可能な VC がない」と表示
  - 確認: Presentation Definition と VC の `vct` / claim が一致しているか
- 症状: 署名検証エラー
  - 確認: issuer / verifier / wallet の DID メソッドが揃っているか
  - 確認: 時刻ずれ（`iat` / `exp`）が大きすぎないか
- 症状: QR が開けない
  - 確認: スマホと PC が同じ LAN か（[スマホ閲覧アプリ](mobile-viewer.md) 参照）
