# データ提供者の認証 (Consent VC / Stage 1)

スマホ SSI ウォレット (iw3ip-wallet) で VC を受け取り、その VC を提示してデータ操作を許可してもらう流れを試します。Part 2 (機能拡張) で VC を使う最初のハンズオンです。Stage 0 ([検知イベントだけを共有する](webcam-event-sharing.md)、[防災イベントを共有する](environment-disaster.md)) で `/consents` に JSON を登録していた部分を、VC の提示に置き換えます。

> **やること**: スマホで Consent VC を受け取り、提示して書き込みを許可してもらう
>
> **前提**: [Home Assistant のデータを取り込む (HA SSI Publisher)](ha-ssi-publisher.md) または [実機なしで一連を動かす (HA Demo Simulator)](ha-demo-simulator.md) を済ませてあること、できれば Stage 0 ([検知イベントだけを共有する](webcam-event-sharing.md) または [防災イベントを共有する](environment-disaster.md))
>
> **使うもの**: PC + スマホ (iw3ip-wallet 入り)
>
> **所要時間**: 約 60 分

!!! note "使用するウォレット"
    本ページでは、Sphereon mobile-wallet を fork した
    [`iw3ip-wallet`](https://github.com/ertlnagoya/iw3ip-wallet) を使います。

!!! tip "dataset の選択"
    本ハンズオンの例は `home/env/temperature` で書かれています。Stage 0
    ([検知イベントだけを共有する](webcam-event-sharing.md)) と同じカメライベント
    `home/event/possible_littering` でも同じ手順で動作するので、その場合は
    dataset と purpose を読み替えてください。Stage の番号は
    [ハンズオンの概要](index.md#part-phase-stage) で説明しています。

## 目的

スマホの SSI ウォレットで受け取った Consent VC を、IW3IP バックエンドが
OID4VP で検証し、検証が通った要求に対してのみデータを共有する
流れを試します。

従来の [Home Assistant のデータを取り込む (HA SSI Publisher)](ha-ssi-publisher.md) では Consent VC
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
  （[スマホでマーケットを開いて購入する (v1)](mobile-viewer.md) と同じ前提）

## 公式リンク

- Sphereon mobile-wallet（upstream）: <https://github.com/Sphereon-Opensource/mobile-wallet>
- `@sphereon/pex`（Presentation Exchange）: <https://github.com/Sphereon-Opensource/pex>
- OpenID for Verifiable Presentations 仕様: <https://openid.net/specs/openid-4-verifiable-presentations-1_0.html>
- OpenID for Verifiable Credential Issuance 仕様: <https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html>
- SD-JWT VC: <https://datatracker.ietf.org/doc/draft-ietf-oauth-sd-jwt-vc/>

## 前提

- Docker / Docker Compose が使える
- PC とスマホが同じ LAN に接続されている
- コマンド中の `$HOST_IP` と URL 中の `<HOST_IP>` は、PC の LAN IP を表します。使うターミナルごとに、最初に `export HOST_IP=<PC の LAN IP>` を実行してください (IP の調べ方は [ハンズオンの概要](index.md#host-ip) を参照)。ブラウザやスマホに入力する URL の `<HOST_IP>` は、同じ IP に置き換えます
- スマホに `iw3ip-wallet`（fork 版）のビルドをインストールできる
  （TestFlight / 内部配布 APK / Expo dev build のいずれか）
- [Home Assistant のデータを取り込む (HA SSI Publisher)](ha-ssi-publisher.md) または [実機なしで一連を動かす (HA Demo Simulator)](ha-demo-simulator.md) で publisher を一度起動できている

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
5. Publisher 側 `/audit/logs` に `allow` (`reason` は `ok`) が残ることを確認する

その後の分岐:

- 拒否ケースを見たい場合: `purpose` が合わない要求に変えて再提示する
- 失効 (revoke): 未実装です。考え方は [拡張ヒント](#拡張ヒント) を参照してください
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

- `input_descriptors` に `dataset_id` と `allowed_purposes` の制約が含まれる
- 形式が `vc+sd-jwt` であること

## 3. ウォレットを起動

`iw3ip-wallet` をスマホで起動し、初回ログインを完了します。
初期鍵は `did:jwk` として生成されます。

起動直後は、ウォレット自身のカード (Holder details) だけが表示されます。

![起動直後のウォレット](images/ha-ssi-wallet/05-holder-details.png){ width="220" }

## 4. Issuer QR で発行

PC ブラウザで以下を開きます。

```txt
http://<HOST_IP>:8080/issuer/offer?type=ConsentVC&dataset_id=home/env/temperature&purpose=research
```

表示された QR をスマホウォレットで読み取ります。初めての相手 (publisher) なので確認の画面が出ます。「はい、続行」を選び、続く確認にも「はい、続行」と答えます。ハンズオンの publisher はウォレットが信頼する一覧に入っていないため、「低い信頼レベル」と表示されます。

| 相手の確認 | 続行の確認 |
|---|---|
| ![相手の確認](images/ha-ssi-wallet/04-trust-prompt.png){ width="220" } | ![続行の確認](images/ha-ssi-wallet/03-trust-confirm-modal.png){ width="220" } |

発行される VC の内容が表示されるので、受け取ります。保存されると、一覧に「IW3IP 同意クレデンシャル」が加わります。

| 受け取る VC の内容 | 保存後の一覧 |
|---|---|
| ![受け取る VC の内容](images/ha-ssi-wallet/02-credential-offer.png){ width="220" } | ![保存後の一覧](images/ha-ssi-wallet/01-credential-list.png){ width="220" } |

## 5. Verifier QR で提示

PC ブラウザで以下を開きます。

```txt
http://<HOST_IP>:8080/verifier/request?dataset_id=home/env/temperature&purpose=research
```

表示された QR をウォレットで読み取ります。要求の内容と、提示する VC の項目が表示されるので、確認して「共有」を選びます。

![提示の確認画面](images/ha-ssi-wallet/09-vp-information-request.png){ width="220" }

検証の結果はウォレットに返り、PC のブラウザには表示されません。PC では監査ログで結果を確かめます。

```bash
curl 'http://localhost:8080/audit/logs?limit=3'
```

いちばん上 (最新) の記録の `action` が `allow`、`reason` が `ok` なら成功です。publisher のログにも、PolicyToken を発行した行が出ます。

```bash
docker compose -f infra/docker-compose.yml logs publisher 2>&1 | grep policy_token_issued
```

参考: ウォレットに返る応答は次の形です。

```json
{
  "status": "allowed",
  "dataset_id": "home/env/temperature",
  "vc_kind": "ConsentVC",
  "policy_token": "VHA9X1d...",
  "policy_token_jti": "9b2fc3...",
  "expires_in": 300
}
```

`policy_token` は §8 で使う短命の認可トークンです。
有効期間は 5 分・単回消費で、`/platform/ingest` を 1 回呼び出すと無効化されます。

## 6. 拒否ケース

`purpose` を `marketing` に変えて同じ提示を実行します。

```txt
http://<HOST_IP>:8080/verifier/request?dataset_id=home/env/temperature&purpose=marketing
```

PC では、§5 と同じく監査ログで確かめます。いちばん上の記録の `action` が `deny`、`reason` が `purpose_mismatch` になります。

```bash
curl 'http://localhost:8080/audit/logs?limit=3'
```

参考: ウォレットに返る応答は次の形です。

```json
{
  "status": "denied",
  "dataset_id": "home/env/temperature",
  "reason": "purpose_mismatch",
  "human_message_ja": "提示された VC の許可目的に、今回の用途が含まれていません。",
  "human_message_en": "The presented VC's allowed_purposes does not cover this purpose."
}
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
- HA SSI Publisher の監査ログと比べ、ポリシー判定の結果に加えて
  「誰がどの VC を提示したか」が残る

ウォレット側にも履歴が残ります。VC のカードを開くと、受け取った時刻と、どの相手にいつ提示したかを確認できます。

![ウォレットの履歴](images/ha-ssi-wallet/10-card-activity.png){ width="220" }

## 8. PolicyToken でデータを書き込む

§5 で得た `policy_token` を `Authorization: Bearer` ヘッダに付けて
`/platform/ingest` を呼び出すと、検証済みの VC にもとづいてデータを書き込めます。
従来の `/consents` JSON 登録経路はヘッダ無しで従来通り動きます。

PolicyToken の性質:

- 有効期限: 5 分（`expires_in` で返却）
- 単回消費: 1 回 `/platform/ingest` を通すと無効化される
- スコープ: 発行時の `dataset_id` と一致する body のみ受け付ける
- 形式: 不透明文字列（サーバ側 in-memory 管理）

リクエスト例:

```bash
TOKEN=<policy_token>  # publisher ログの policy_token_issued 行の token= の値 (この節の最後を参照)

curl -X POST http://$HOST_IP:8080/platform/ingest \
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
curl "http://$HOST_IP:8080/audit/logs?limit=5"
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

トークンは `/verifier/response` のレスポンスとしてウォレットに返るため、
PC のブラウザには表示されません。提示が終わったら publisher のログを確認し、
`policy_token_issued jti=... token=... dataset=... ttl=...s` の行にある
`token=` の後ろの値をコピーして、上の `<policy_token>` に使います。

```bash
docker compose -f infra/docker-compose.yml logs publisher 2>&1 | grep policy_token_issued | tail -n 1
```

§5 から 5 分を過ぎた場合や、`policy_token_expired` が返る場合は、トークンの期限が切れています。§5 の提示をやり直し、上のコマンドで出た最新の行 (最後の行) の `token=` の値を使います。

## 次のステップ

このハンズオン (Stage 1) で扱ったのは **書き込み単回認可** (ConsentVC + PolicyToken)
だけです。ほかの認可の形は、次のページで扱います。

- [データ閲覧者の認証 (Viewer VC / Stage 3)](ha-ssi-viewer.md) — 読み出し。TTL 内は何度でも使える (ViewerVC)
- [機器間での連携認証 (Service VC / Stage 4 prep)](ha-ssi-service.md) — 機器からの連続書き込み (ServiceVC)
- [購入後に証明書をウォレットへ渡す (Bridge / Stage 5)](marketplace-vc-bridge.md) — 購入と結びついた読み出し (PurchaseViewerVC)
- [4 種類の証明書を 1 動線でつなぐ (Stage 6)](marketplace-vc-end-to-end.md) — Stage 1〜5 の総合演習
- [出品者の身元を裏付ける (Seller VC / Stage 7)](marketplace-seller-vc.md) — 出品者の身元の管理

## 拡張ヒント

- **失効 (未実装)**: 現在の publisher には revoke のエンドポイントがありません。
  Status List 2021 などで失効リストを提供し、revoke 後に同じ提示が `denied` になるよう拡張する
- **SIOPv2 による holder 認証**: 提示と同時に holder の DID を検証する
- **Phase 3 連携**: [AI に手順を考えさせる (LLM Planner)](llm-planner.md) の plan 実行前に
  VC 提示を要求し、知能統合層での権限チェックに拡張する

## トラブル時

- 症状: ウォレットが「提示可能な VC がない」と表示
  - 確認: Presentation Definition と VC の `vct` / claim が一致しているか
- 症状: 署名検証エラー
  - 確認: issuer / verifier / wallet の DID メソッドが揃っているか
  - 確認: 時刻ずれ（`iat` / `exp`）が大きすぎないか
- 症状: QR が開けない
  - 確認: スマホと PC が同じ LAN か（[スマホでマーケットを開いて購入する (v1)](mobile-viewer.md) 参照）
