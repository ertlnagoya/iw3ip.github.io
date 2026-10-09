# データ提供者の認証 (Consent VC / Stage 1)

スマホ SSI ウォレット (iw3ip-wallet) で VC を受け取り、その VC を提示してデータ操作を許可してもらう流れを試します。Part 2 (機能拡張) の最初のハンズオンです。

> **やること**: スマホで Consent VC を受け取り、提示して書き込みを許可してもらう
>
> **前提**: [HA SSI Publisher サンプル](ha-ssi-publisher.md) または [HA Demo Simulator](ha-demo-simulator.md) を済ませてあること
>
> **使うもの**: PC + スマホ (iw3ip-wallet 入り)
>
> **所要時間**: 約 60 分

!!! note "使用するウォレット"
    本ページでは、Sphereon mobile-wallet を fork した
    [`iw3ip-wallet`](https://github.com/ertlnagoya/iw3ip-wallet) を使います。

!!! tip "dataset の選択"
    本ハンズオンの例は `home/env/temperature` で書かれています。Stage 0
    ([webcam-event-sharing](webcam-event-sharing.md)) と同じカメライベント
    `home/event/possible_littering` でも同じ手順で動作するので、その場合は
    dataset と purpose を読み替えてください。Stage の番号は
    [ハンズオン概要](index.md#part-phase-stage) で説明しています。

## 目的

スマホの SSI ウォレットで受け取った Consent VC を、IW3IP バックエンドが
OID4VP で検証し、検証が通った要求に対してのみデータを共有する
流れを試します。

従来の [HA x SSI Publisher サンプル](ha-ssi-publisher.md) では Consent VC
相当のポリシー JSON を `/consents` に直接登録していました。本サンプルでは、
**ウォレットが保持する VC を提示 → 検証 → allowed/denied** という
本来の SSI モデルを扱います。

このページで使う用語:

- **OID4VCI** (OpenID for Verifiable Credential Issuance): VC を発行してウォレットに渡す手順の仕様
- **OID4VP** (OpenID for Verifiable Presentations): ウォレットが VC を提示する手順の仕様
- **Issuer / Verifier**: VC を発行する側 / 提示された VC を検証する側。本ハンズオンではどちらも publisher が担う
- **Presentation Definition (PD)**: Verifier が「どの VC のどの項目を提示してほしいか」を記述した要求。PEX (Presentation Exchange) 仕様で定義されている
- **SD-JWT VC**: 必要な項目だけを選んで開示できる JWT 形式の VC
- **did:jwk**: 公開鍵 (JWK) をそのまま識別子にした DID
- **PolicyToken**: VC の提示が検証された後に publisher が発行する、短命の認可トークン

パイプライン:

`スマホウォレット (VC保持) -> QR/Deeplink -> OID4VP Verifier -> Platform API -> イベント共有`

## このページで分かること

- ウォレットに VC を発行（OID4VCI）し、提示（OID4VP）する基本フロー
- Verifier 側の Presentation Definition（PEX）とウォレット側の応答対応
- `did:jwk` / `did:key` と SD-JWT VC を用いた軽量構成
- `allowed` / `denied` が VC の内容と `purpose` によって決まる様子

## よくある問題

- Presentation Definition の `input_descriptors` と VC の claim が食い違うと、
  ウォレットが「提示可能な VC がない」と判定する
- DID メソッドが issuer / wallet / verifier でずれると署名検証に失敗する
- SD-JWT VC・W3C VC-JWT・mdoc には互換性がないため、1 つの形式に固定する
- スマホと PC が別ネットワークだと QR 経由の redirect が届かない
  （[スマホ閲覧アプリ](mobile-viewer.md) と同じ前提）

## 公式リンク

- Sphereon mobile-wallet（upstream）: <https://github.com/Sphereon-Opensource/mobile-wallet>
- `@sphereon/pex`（Presentation Exchange）: <https://github.com/Sphereon-Opensource/pex>
- OpenID for Verifiable Presentations 仕様: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- OpenID for Verifiable Credential Issuance 仕様: <https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html>
- SD-JWT VC: <https://datatracker.ietf.org/doc/draft-ietf-oauth-sd-jwt-vc/>

## 前提

- Docker / Docker Compose が使える
- PC とスマホが同じ LAN に接続されている
- スマホに `iw3ip-wallet`（fork 版）のビルドをインストールできる
  （TestFlight / 内部配布 APK / Expo dev build のいずれか）
- [HA x SSI Publisher サンプル](ha-ssi-publisher.md) の構成を一度起動できている

## 関連リポジトリ

- `iw3ip-wallet`: <https://github.com/ertlnagoya/iw3ip-wallet>
  - upstream: Sphereon-Opensource/mobile-wallet
  - ブランチ方針: `main` は upstream 追従、`iw3ip/*` で IW3IP 固有改変
- [Blockchain_IoT_Marketplace](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace): publisher (OID4VCI / OID4VP のエンドポイントを含む。`ssi-wallet` プロファイルで起動)

## 最短ルート

最初は次の 5 手順で十分です。

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
拒否されます。

## 7. 監査ログ確認

```bash
curl 'http://localhost:8080/audit/logs?limit=10'
```

確認ポイント:

- `presentation_verified` イベントが `allow` / `deny` と共に記録される
- `holder_did`、`vc_hash`、`purpose` が残る
- HA x SSI Publisher サンプルの監査ログと比べ、ポリシー判定の結果に加えて
  「誰がどの VC を提示したか」が残る

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
  -d '{
    "dataset_id":"home/env/temperature",
    "purpose":"research",
    "value":21.4
  }'
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
ハンズオンでは publisher ログから JSON を取得して curl に渡すのが簡便です。

```bash
docker compose -f infra/docker-compose.yml --profile ssi-wallet logs -f publisher | grep policy_token
```

## 次のステップ

このハンズオン (Stage 1) で扱ったのは **書き込み単回認可** (ConsentVC + PolicyToken)
だけです。シリーズの他のステージで他の認可形態を扱います:

- [Stage 3: SSI ビューワサンプル](ha-ssi-viewer.md) — 読み出し多回認可 (ViewerVC)
- [Stage 4 prep: SSI サービスサンプル](ha-ssi-service.md) — M2M 連続書き込み (ServiceVC)
- [Stage 5: マーケット連携 v2](marketplace-vc-bridge.md) — 購入連動 read (PurchaseViewerVC)
- [Stage 6: 4-VC end-to-end](marketplace-vc-end-to-end.md) — Stage 1〜5 の総合演習
- [Stage 7: SellerVC](marketplace-seller-vc.md) — 出品身元のガバナンス

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
