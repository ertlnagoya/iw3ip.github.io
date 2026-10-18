# スマホアプリで購入から閲覧まで (Stage A)

Phase 2 の backend をそのまま使い、`iot-market-ui` をスマホのホーム画面アプリ (PWA: Progressive Web App) として使います。コマンドラインなしで購入から VC 受領、閲覧までタップで進められます。

> **やること**: PWA として iot-market-ui をスマホで使い、購入と VC 取得を進める
>
> **前提**: [Marketplace VC Bridge](marketplace-vc-bridge.md) (Stage 5)
>
> **使うもの**: PC + スマホ (iw3ip-wallet) + MetaMask モバイル
>
> **所要時間**: 約 30 分

!!! abstract "コマンドラインを使わない PWA 版"
    Phase 2 で構築した backend をそのまま使い、`iot-market-ui` を
    iPhone/Android のホーム画面アプリとして使うハンズオンです。
    コマンドライン操作なしで、購入から VC 受領、データ閲覧までを
    スマホ画面のタップで進めます。本ページの PWA 版を「案 A」、ウォレット機能を
    組み込んだネイティブアプリ (React Native) を「案 B」と呼びます。案 B は Stage 9
    として計画しています。

!!! tip "Stage 0 と同じデータ"
    Stage A の購入対象は Stage 6 case B 以降の deploy script で登録された
    Merchandise リストから選びます。`home/event/possible_littering` (Merchandise #2)
    を選べば、Stage 0 (Stage の一覧は
    [ハンズオンの概要](index.md#part-phase-stage) を参照)
    [webcam-event-sharing](webcam-event-sharing.md) と同じカメライベントが、
    購入経由 (PurchaseViewerVC) で読めるところまで確認できます。
    `home/event/flood_risk_high` (Merchandise #4) は PurchaseViewerVC の
    Presentation Definition が登録されていないため、閲覧まで進めません。

## 目的

- iot-market-ui を **PWA (Progressive Web App)** として使う
- 既存の deeplink 連携 (publisher / iw3ip-wallet / MetaMask Mobile) を
  初学者が迷わず進められる 1 つの流れとして確認する
- 購入履歴を端末側にローカル保存し、`/my-data` から後日アクセスできる
  ことを確認する

## このページで分かること

- iPhone Safari の **「ホーム画面に追加」** で IoT マーケットがアプリ化される
- 購入後 `/purchased/[txHash]` 画面の deeplink ボタンを押すと
  iw3ip-wallet が起動して VC を受領 → ブラウザに復帰
- `/my-data` で過去の購入が一覧表示される
- `/welcome` の 3 ステップで、初めての利用者でも準備を進められること

## つまずきやすい点

- iOS Safari の PWA 制約: 一部 API (push 通知、Bluetooth 等) が制限される。
  本ハンズオンの範囲では問題なし
- Universal Links / App Links が wallet 側で設定されていないと、
  受領完了後のブラウザ自動復帰が起きない。**復帰しなくても手動で
  Safari に戻れば動線は途切れません**
- localStorage は端末ローカル。同じ wallet を別端末で使うと履歴は
  共有されない

## 先に読むページ

- [VC Architecture Overview](../design/vc-architecture-overview.md) — 全体像
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md) — 購入動線の元ハンズオン
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md) — Seller 側の動線

## 前提

- Stage 5 / 6 / 7 のハンズオンを済ませている。その後に Hardhat を起動し直した場合は、先に [Stage 6 の Step E0](marketplace-vc-end-to-end.md#step-e0-前提環境の起動-stage-5-と共通) (ノードの起動、デプロイ、公開鍵の登録、bridge の作り直し) をやり直す
- publisher + bridge + Hardhat + iot-market-ui が起動できる
- iw3ip-wallet (iPhone) が動く
- コマンド中の `$HOST_IP` と URL 中の `<HOST_IP>` は、PC の LAN IP を表します。使うターミナルごとに、最初に `export HOST_IP=<PC の LAN IP>` を実行してください (IP の調べ方は [ハンズオンの概要](index.md#host-ip) を参照)。ブラウザやスマホに入力する URL の `<HOST_IP>` は、同じ IP に置き換えます
- 教材リポジトリは `~/program/Blockchain_IoT_Marketplace`、ウォレットは `~/program/iw3ip-wallet` に clone した前提で、Mac と iPhone を使う手順を示します

## 全体像

```
[初回起動]
   iPhone Safari → http://<HOST_IP>:5173/welcome
        ↓ 3 ステップ完了 → 「マーケットへ進む」
[ホーム画面追加]
   Safari メニュー → ホーム画面に追加
   → IW3IP アプリアイコンが追加 (standalone display)
[起動]
   アイコンタップ → iot-market-ui の "/" が standalone で開く
[購入]
   商品選択 → MetaMask Mobile in-app browser で確認 → tx 送信
   → 自動で /purchased/[txHash] に遷移
   → 「ウォレットで開く」ボタン → iw3ip-wallet 起動 → VC 受領
[履歴 + 閲覧]
   ハンバーガーメニュー → 「購入履歴 / データを見る」
   → /my-data で過去の購入一覧
   → 「閲覧チケットを取得」 → wallet で提示
   → ViewerToken をペースト → データ表示
```

---

## Step M0. PWA 化された iot-market-ui を起動

### 何を確認するか
- Vite dev server に PWA manifest と service worker が組み込まれている
  (service worker が登録されるのは https か localhost で開いたときだけです。`http://<HOST_IP>:5173` では登録されませんが、ホーム画面への追加はできます)
- iPhone Safari で開いたとき、 `Add to Home Screen` でアプリ化できる

### 操作

ターミナル D (iot-market-ui):

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only
# pull したコードで publisher と bridge を作り直す (iot-market-ui からの呼び出しには publisher の CORS 対応が必要)
docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d

cd iot-market-ui
cat > .env.local <<EOF
VITE_RPC_URL=http://$HOST_IP:8545
VITE_PUBLISHER_URL=http://$HOST_IP:8080
EOF
npm run dev -- --host 0.0.0.0 --port 5173
```

### 期待出力
```
VITE v5.x.x  ready in xxx ms
➜  Network: http://<HOST_IP>:5173/
```

iPhone Safari で `http://<HOST_IP>:5173/welcome` を開くと、3 ステップ
オンボーディング画面が出ます。

---

## Step M1. ホーム画面に追加 (PWA install)

### 何を確認するか
- iOS Safari の **共有メニュー** から **「ホーム画面に追加」** で
  アプリアイコンが作られる
- アプリを起動すると Safari の URL バーが出ない standalone モード

### 操作 (iPhone)

1. Safari で `http://<HOST_IP>:5173/` を開く
2. 画面下の **共有ボタン** (□に↑) をタップ
3. **「ホーム画面に追加」** を選択
4. 名前 (例: `IW3IP`) を確認して **「追加」**
5. ホーム画面に IW3IP アイコンが出る → タップで起動

### 期待結果

- ホーム画面にアイコン (デフォルト favicon) が表示
- アイコン起動時、URL バーやタブバーがない standalone 表示
- アプリのページ遷移はそのまま動く (`/`, `/welcome`, `/my-data`,
  `/seller`, `/merchandise/[address]`)

!!! note "Android (Chrome) の場合"
    `http://<HOST_IP>:5173` は https ではないため service worker が登録されず、
    インストールを促す表示は自動では出ません。
    メニュー → 「ホーム画面に追加」から追加してください。

---

## Step M2. 初回オンボーディング `/welcome`

### 何を確認するか
- 3 ステップ表示で MetaMask + iw3ip-wallet 準備を案内
- 「もう入っています」で次に進める
- 最終的に「マーケットへ進む」で `/` に遷移

### 操作 (iPhone, ホーム画面アプリ起動 or Safari)

`http://<HOST_IP>:5173/welcome` を開く → 表示される 3 ステップを
読みながらタップ。

### 期待表示

```
STEP 1 / 3 - お財布アプリ (MetaMask)
[MetaMask を入手する] [もう入っています →]

STEP 2 / 3 - 閲覧チケット用アプリ (IW3IP Wallet)
[もう入っています →]

STEP 3 / 3 - 準備完了
[マーケットへ進む] [購入履歴を見る]
```

ハンズオン参加者は Stage 1〜7 で既に両方のアプリを設定済のはずなので
「もう入っています」を 2 回タップすると STEP 3 に進みます。続けて
「マーケットへ進む」をタップすると `/` に移動します。

---

## Step M3. 購入動線 (Stage 5/6/7 の動線をモバイルで再確認)

### 何を確認するか
- 既存の `/merchandise/[address]` ページで MetaMask Mobile in-app
  browser を使うと、購入が完了し `/purchased/[txHash]` に自動遷移する
- `/purchased/[txHash]` に **「ウォレットで開く」** が出ている

### 操作 (iPhone, MetaMask Mobile)

0. MetaMask Mobile を準備する (初回のみ。[Stage 5 の Step 5](marketplace-vc-bridge.md#step-5-metamask-設定) と同じ設定です)
    - ネットワークを追加: Network Name `Hardhat localhost`、RPC URL `http://<HOST_IP>:8545`、Chain ID `31337`、Currency `ETH`
    - 追加後、`Hardhat localhost` に切り替える
    - アカウントメニュー → 「秘密鍵をインポート」→ Account #2 の秘密鍵 (Stage 5 の Step 1 でメモしたもの) を入れる。PubKey を登録済みのアカウント (Stage 5 の Step 3) でないと購入できません
1. MetaMask Mobile を開く → 内蔵ブラウザ (右下のグローブアイコン) で
   `http://<HOST_IP>:5173/merchandise/0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0`
   を開く (deploy script が作る Merchandise #0。#1 は Stage 5 で購入済みのことがあるため使わない)
2. (省略可) ハンバーガーメニュー → 「Connect your Wallet」をタップして MetaMask 接続。
   スマートフォンの画面幅では右上の「Connect your wallet!」ボタンは表示されません。手順 3 のボタンを押すと自動で接続されるので、この手順は飛ばしても構いません
3. 「Purchase!」ボタンを押し、MetaMask 側で **Confirm**
4. 数秒後、自動で `/purchased/<txHash>` に遷移し QR + 「ウォレットで開く」
   ボタンが表示される

この購入履歴は、購入に使った MetaMask Mobile の内蔵ブラウザの `localStorage` に保存されます。
Step M4・M5 も、この内蔵ブラウザで続けてください。

!!! note "PC ブラウザでの代替"
    iPhone での MetaMask Mobile の操作が難しい場合は、PC の Chrome の
    MetaMask 拡張で同じ URL を開いて購入しても構いません。購入後は同じく
    `/purchased/[txHash]` に遷移します。ただし `/my-data` の履歴は購入に
    使った端末にだけ保存されます。

---

## Step M4. iw3ip-wallet で VC 受領

### 何を確認するか
- `/purchased/[txHash]` の deeplink で wallet が起動 → 確認画面
- 承認すると wallet 内に PurchaseViewerVC が保存

### 操作 (iPhone)

`/purchased/<txHash>` で **「ウォレットで開く」** をタップ。

iw3ip-wallet が起動して「IW3IP 購入閲覧（Tier 1 / イベントのみ）」確認画面 (英語表示では「IW3IP Purchase Viewer (Event-only)」) → 承認。

### 期待結果
- wallet の VC 一覧に新しいエントリ
- claim に `merchandise_address`, `tx_hash`, `buyer_eth_addr`, `dataset_id`
- 承認後は、アプリの切り替えで MetaMask Mobile (内蔵ブラウザ) に戻る

---

## Step M5. `/my-data` で履歴 + データ閲覧

### 何を確認するか
- `localStorage` に Step M3 の購入が保存され、`/my-data` で一覧表示
- 「閲覧チケットを取得」をタップで wallet 提示画面が開く
- 提示後、ViewerToken を `/my-data` のフォームに貼ると JSON 表示

### 操作

1. MetaMask Mobile の内蔵ブラウザで `http://<HOST_IP>:5173/my-data` を開く
   (ハンバーガーメニュー → **「購入履歴 / データを見る」** でも可)
2. Step M3 の購入が表示されている (dataset_id, merchandise, tx, 日時)
3. **「閲覧チケットを取得」** をタップ → 開いたページの **「ウォレットで開く」** をタップ → wallet で PurchaseViewerVC を選んで提示
4. PC 側 `docker logs iw3ip-publisher 2>&1 | grep viewer_token_issued | tail -1` で token を確認
   (publisher のログは標準エラーに出るため `2>&1` が必要)
5. `/my-data` に戻り、その購入の **「このデータを見る」** をタップして「ViewerToken (Bearer)」欄を表示
6. その token を「ViewerToken (Bearer)」欄に貼り付け
7. **「取得実行」** で JSON が表示

!!! note "ViewerToken の有効期限は 60 秒"
    手順 3〜7 は続けて行ってください。PC で確認した token は、ユニバーサルクリップボードや AirDrop で iPhone に渡します。
    `viewer_token_expired` が出たら、手順 3 からやり直してください。

### 期待 JSON

```json
{
  "dataset_id": "home/env/temperature",
  "count": ...,
  "read_count": 1,
  "seller_did": "unknown",
  "allowed_views": ["event"],
  "rows": [...]
}
```

Merchandise #0 は seller 登録をしていないので、`seller_did` は `"unknown"` です。
`seller_did` に seller の `did:jwk:...` が入るのは、Stage 7 の Step S3〜S4 で新しく
deploy・登録した Merchandise を、その後 publisher を作り直さずに購入した場合だけです
(seller の対応付けは publisher のメモリにあり、作り直すと消えます。Stage 7 で使った
Merchandise は購入済みなので、もう一度は買えません)。

!!! note "現状の限界 (Stage 9 で解消予定)"
    ViewerToken を手動で貼り付ける必要がある点が、現状の使い勝手の
    課題です。タップだけで完結させるには、OID4VP クライアントを組み込んだ
    ネイティブアプリ (案 B、Stage 9) が必要です。

---

## 完了判定マトリクス

| 工程 | 確認項目 | 出力例 |
| --- | --- | --- |
| M0 | PWA shell が動く | `manifest.webmanifest` を Safari の DevTools で確認 |
| M1 | ホーム画面アイコン | iPhone ホームに `IW3IP` アイコン |
| M2 | onboarding 3 ステップ | 「マーケットへ進む」で `/` に遷移 |
| M3 | 購入後遷移 | `/purchased/[txHash]` で QR + deeplink |
| M4 | wallet で VC 受領 | claim に merchandise/tx/eth |
| M5 | `/my-data` で履歴表示 + データ取得 | `dataset_id` と `rows` を含む JSON |

---

## トラブルシューティング

### A. ホーム画面に追加が選択肢に出ない

**原因**: Safari でないブラウザを使っている、または manifest 取得失敗。

**対処**:

- Safari (iOS 標準) で開いていることを確認
- `http://<HOST_IP>:5173/manifest.webmanifest` を直接開いて JSON が
  表示されるか確認
- `http://localhost` ではなく LAN IP で開いていることを確認

### B. アイコンタップで起動するもブラウザバーが出てしまう

**原因**: manifest の `display: "standalone"` が反映されていない、または
Safari のキャッシュ。

**対処**: ホーム画面のアイコンを長押しで削除 → Safari で再追加。

### C. 「ウォレットで開く」をタップしても wallet が起動しない

**原因**: `openid-credential-offer://` scheme が wallet で登録されていない。

**対処**:

- iw3ip-wallet が起動済か確認
- Stage 5 のトラブルシューティング C (wallet キャッシュ、Metro bundler)
  を確認

### D. `/my-data` の履歴が空になる

**原因**: 次のどちらかです。

- 購入時に `/purchased/[txHash]` で `localStorage` 書き込みが失敗した (private browsing モード等)
- publisher が古く CORS 未対応のため、`/purchased/[txHash]` からの `/marketplace/claim` が失敗した (履歴は claim が成功したときだけ保存される)

**対処**:

- Safari の private browsing をオフ
- `git pull` 後に `cd ~/program/Blockchain_IoT_Marketplace && docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d` で publisher を作り直す
- 別の購入を実行して再度 `/my-data` を確認

### E. その他

[Stage 5 トラブルシューティング](marketplace-vc-bridge.md#トラブルシューティング)
と [Stage 7](marketplace-seller-vc.md) のトラブルシューティングも参照してください。

---

## 限界 (Stage 9 で解消予定)

- **ViewerToken の貼り付け**: 自動取得には、wallet 側に publisher API を
  呼び出す実装が必要 (案 B = Stage 9)
- **アプリ間スイッチング**: 3 つのアプリ (PWA / MetaMask / iw3ip-wallet)
  を行き来する。1 つのアプリで完結させるのは案 B
- **オフライン対応**: PWA だが service worker は pass-through。チェーン
  read は常にオンライン
- **iOS の通知 / バックグラウンド処理**: iOS PWA の制約で限定的

## 関連

- [VC Architecture Overview](../design/vc-architecture-overview.md)
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md)
- [Stage 6: 4-VC end-to-end](marketplace-vc-end-to-end.md)
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md)
