# データマーケット モバイルアプリ (Stage A: 案 A の最小版)

!!! abstract "素人でも使える PWA 体験"
    Phase 2 で構築した backend をそのまま使い、`iot-market-ui` を
    iPhone/Android のホーム画面アプリとして体験するハンズオンです。
    コマンドライン操作なしで、購入から VC 受領、データ閲覧までを
    スマホ画面のタップで進めます。Stage 8 (信頼モデルの強化) と並行し、
    Stage 9 (案 B / RN 統合アプリ) への足掛かりです。

## 目的

- iot-market-ui を **PWA (Progressive Web App)** として体験する
- 既存の deeplink 連携 (publisher / iw3ip-wallet / MetaMask Mobile) を
  「素人が脱落しない 1 動線」に磨いた状態で確認する
- 購入履歴を端末側にローカル保存し、`/my-data` から後日アクセスできる
  ことを確認する

## このページで分かること

- iPhone Safari の **「ホーム画面に追加」** で IoT マーケットがアプリ化される
- 購入後 `/purchased/[txHash]` 画面の deeplink ボタンを押すと
  iw3ip-wallet が起動して VC を受領 → ブラウザに復帰
- `/my-data` で過去の購入が一覧表示される
- `/welcome` の 3 ステップで初回ユーザでも詰まらないフロー

## つまずきやすい点

- iOS Safari の PWA 制約: 一部 API (push 通知、Bluetooth 等) が制限される。
  本ハンズオンの範囲では問題なし
- Universal Links / App Links が wallet 側で設定されていないと、
  受領完了後のブラウザ自動復帰が起きない。**復帰しなくても手動で
  Safari に戻れば動線は途切れません**
- localStorage は端末ローカル。同じ wallet を別端末で使うと履歴は
  共有されない (この限界はハンズオンで明記)

## 公式リンク

- [VC Architecture Overview](../design/vc-architecture-overview.md) — 全体像
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md) — 購入動線の元ハンズオン
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md) — Seller 側の動線

## 前提

- Stage 5 / 6 / 7 の e2e 検証が済んでいる
- publisher + bridge + Hardhat + iot-market-ui が起動できる
- iw3ip-wallet (iPhone) が動く
- LAN IP は `192.168.68.53` で示すのであなたの環境に読み替え

## 全体像

```
[初回起動]
   iPhone Safari → http://192.168.68.53:5173/welcome
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
- iPhone Safari で開いたとき、 `Add to Home Screen` でアプリ化できる

### 操作

ターミナル D (iot-market-ui):

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only

cd iot-market-ui
cat > .env.local <<EOF
VITE_RPC_URL=http://192.168.68.53:8545
VITE_PUBLISHER_URL=http://192.168.68.53:8080
EOF
npm run dev -- --host 0.0.0.0 --port 5173
```

### 期待出力
```
VITE v5.x.x  ready in xxx ms
➜  Network: http://192.168.68.53:5173/
```

iPhone Safari で `http://192.168.68.53:5173/welcome` を開くと、3 ステップ
オンボーディング画面が出る。

---

## Step M1. ホーム画面に追加 (PWA install)

### 何を確認するか
- iOS Safari の **共有メニュー** から **「ホーム画面に追加」** で
  アプリアイコンが作られる
- アプリを起動すると Safari の URL バーが出ない standalone モード

### 操作 (iPhone)

1. Safari で `http://192.168.68.53:5173/` を開く
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
    自動的に install プロンプトが出る (manifest があるため)。
    手動でも メニュー → 「アプリをインストール」から追加可。

---

## Step M2. 初回オンボーディング `/welcome`

### 何を確認するか
- 3 ステップ表示で MetaMask + iw3ip-wallet 準備を案内
- 「もう入っています」で次に進める
- 最終的に「マーケットへ進む」で `/` に遷移

### 操作 (iPhone, ホーム画面アプリ起動 or Safari)

`http://192.168.68.53:5173/welcome` を開く → 表示される 3 ステップを
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
2 回 「もう入っています」をタップすれば `/` に着く。

---

## Step M3. 購入動線 (Stage 5/6/7 の動線をモバイルで再確認)

### 何を確認するか
- 既存の `/merchandise/[address]` ページで MetaMask Mobile in-app
  browser を使うと、購入が完了し `/purchased/[txHash]` に自動遷移する
- `/purchased/[txHash]` に **「ウォレットで開く」** が出ている

### 操作 (iPhone, MetaMask Mobile)

1. MetaMask Mobile を開く → 内蔵ブラウザ (右下のグローブアイコン) で
   `http://192.168.68.53:5173/merchandise/0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9`
   を開く (PR #20 deploy 由来の Merchandise #1)
2. 「Connect your wallet!」をタップして MetaMask 接続
3. 「Purchase」ボタンを押し、MetaMask 側で **Confirm**
4. 数秒後、自動で `/purchased/<txHash>` に遷移し QR + 「ウォレットで開く」
   ボタンが表示される

!!! note "PC ブラウザでの代替"
    iPhone での MetaMask Mobile 操作が難しい場合、PC Chrome の
    MetaMask 拡張で同じ URL を開いて購入してもよい。tx hash は同じく
    `/purchased/[txHash]` に流れる。`/my-data` の履歴は iPhone 端末
    のみに保存されるので、最終確認は iPhone で `/my-data` を開く。

---

## Step M4. iw3ip-wallet で VC 受領

### 何を確認するか
- `/purchased/[txHash]` の deeplink で wallet が起動 → 確認画面
- 承認すると wallet 内に PurchaseViewerVC が保存

### 操作 (iPhone)

`/purchased/<txHash>` で **「ウォレットで開く」** をタップ。

iw3ip-wallet が起動して「IW3IP 購入閲覧クレデンシャル」確認画面 → 承認。

### 期待結果
- wallet の VC 一覧に新しいエントリ
- claim に `merchandise_address`, `tx_hash`, `buyer_eth_addr`, `dataset_id`
- (任意) 自動的に Safari に復帰 = Universal Links が wallet 側で
  設定されていれば。設定されていなければホームボタンで Safari に戻る

---

## Step M5. `/my-data` で履歴 + データ閲覧

### 何を確認するか
- `localStorage` に Step M3 の購入が保存され、`/my-data` で一覧表示
- 「閲覧チケットを取得」をタップで wallet 提示画面が開く
- 提示後、ViewerToken を `/my-data` のフォームに貼ると JSON 表示

### 操作

1. iPhone のホーム画面アプリで IW3IP を起動
2. ハンバーガーメニュー → **「購入履歴 / データを見る」**
3. Step M3 の購入が表示されている (dataset_id, merchandise, tx, 日時)
4. **「閲覧チケットを取得」** をタップ → wallet で提示
5. PC 側 `docker logs publisher | grep viewer_token_issued` で token を確認
6. その token を `/my-data` の「ViewerToken (Bearer)」欄に貼り付け
7. **「取得実行」** で JSON が表示

### 期待 JSON

```json
{
  "dataset_id": "home/env/temperature",
  "count": ...,
  "read_count": 1,
  "seller_did": "did:jwk:...",
  "rows": [...]
}
```

!!! note "現状の限界 (Stage 9 で解消予定)"
    ViewerToken の受け渡しが**手動 (ペースト)** な点が現状の最大の
    UX 課題です。本格的な「タップで完結」は案 B (Stage 9 = ネイティブ
    アプリで OID4VP クライアント内蔵) で解決します。

---

## 完了判定マトリクス

| 工程 | 確認項目 | 出力例 |
| --- | --- | --- |
| M0 | PWA shell が動く | `manifest.webmanifest` を Safari の DevTools で確認 |
| M1 | ホーム画面アイコン | iPhone ホームに `IW3IP` アイコン |
| M2 | onboarding 3 ステップ | 「マーケットへ進む」で `/` に遷移 |
| M3 | 購入後遷移 | `/purchased/[txHash]` で QR + deeplink |
| M4 | wallet で VC 受領 | claim に merchandise/tx/eth |
| M5 | `/my-data` で履歴表示 + データ取得 | `seller_did` 入りの JSON |

---

## トラブルシューティング

### A. ホーム画面に追加が選択肢に出ない

**原因**: Safari でないブラウザを使っている、または manifest 取得失敗。

**対処**:
- Safari (iOS 標準) で開いていることを確認
- `http://192.168.68.53:5173/manifest.webmanifest` を直接開いて JSON が
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

**原因**: 購入時に `/purchased/[txHash]` で `localStorage` 書き込みが
失敗した (private browsing モード等)。

**対処**:
- Safari の private browsing をオフ
- 別の購入を実行して再度 `/my-data` を確認

### E. その他

[Stage 5 トラブルシューティング](marketplace-vc-bridge.md#トラブルシューティング)
+ [Stage 7](marketplace-seller-vc.md) のものが引き続き有効。

---

## 限界 (Stage 9 で解消予定)

- **ViewerToken のペースト**: 自動取得には wallet 側に publisher API
  叩く実装が必要 (案 B = Stage 9)
- **アプリ間スイッチング**: 3 つのアプリ (PWA / MetaMask / iw3ip-wallet)
  を行き来する。本物のシングルアプリ体験は案 B
- **オフライン対応**: PWA だが service worker は pass-through。チェーン
  read は常にオンライン
- **iOS の通知 / バックグラウンド処理**: iOS PWA の制約で限定的

## 関連

- [VC Architecture Overview](../design/vc-architecture-overview.md)
- [Stage 5: Marketplace × Wallet bridge](marketplace-vc-bridge.md)
- [Stage 6: 4-VC end-to-end](marketplace-vc-end-to-end.md)
- [Stage 7: Marketplace Seller VC](marketplace-seller-vc.md)
