# ハンズオン

実際に手を動かして IW3IP を理解する章です。
**基本** で全体像を体験し、**機能拡張** で安全な共有や段階アクセスを深掘りし、**知能統合** で AI 連携まで見せます。

> はじめに [環境構築](../setup/index.md) を済ませてからこのページに進んでください。

## 基本 と 機能拡張 の関係

このハンズオンは、マーケットプレイス周辺の機能を **基本 (v1)** と **機能拡張 (v2)** の 2 段階で説明します。
両者は **置き換え関係ではなく追加レーン** です。基本だけでもデータの売買は完結します。

### 基本 (v1) — まず動く形

- データを **マーケットプレイスに出品** し、**MetaMask で購入**、**暗号化 IPFS から復号** して受け取る
- 必要なのは PC と MetaMask のみ
- ハンズオン Part 1 はこの v1 経路を組み立てる練習です

### 機能拡張 (v2) — 安全・条件付き・段階的に

- 購入したあと、**スマホ SSI ウォレット** が **VC** を受け取る
- 受信側は VC を提示してデータを取得する。何のために使うか (purpose) や信頼度に応じて、見せる中身を段階的に変える
- ハンズオン Part 2 はこの v2 経路と、その上に乗る同意・段階アクセスを学びます

```
購入 (v1 と v2 で共通)
    ↓
    ├── v1 lane: 暗号化 IPFS URI → 復号 → データ        ← Part 1 で扱う
    └── v2 lane: bridge → PurchaseViewerVC → wallet     ← Part 2 で扱う
                  → ViewerToken → /platform/data
```

詳しい設計は [Marketplace VC Bridge — v1 / v2 設計仕様](../design/marketplace-vc-bridge-spec.md)。

## 全体地図

<div class="iw3ip-phase-grid">
  <div class="iw3ip-phase-card iw3ip-phase-1">
    <div class="iw3ip-phase-kicker">Part 1 / 基本 (v1)</div>
    <h3>📡 IoT データを取って共有する</h3>
    <p>カメラやセンサからデータを取り、見て、マーケットプレイスで売買する基本の経路を組み立てます。</p>
    <p class="iw3ip-phase-links">
      <a href="ha-demo-simulator.md">HA Demo Simulator</a> /
      <a href="huskylens2.md">HUSKYLENS2</a> /
      <a href="webcam.md">USB Webcam</a> /
      <a href="ha-ssi-publisher.md">HA Publisher</a> /
      <a href="mobile-viewer.md">Mobile Viewer</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-2">
    <div class="iw3ip-phase-kicker">Part 2 / 機能拡張 (v2)</div>
    <h3>🧭 同意・条件付き共有・段階アクセス</h3>
    <p>VC とウォレットを使って、目的や信頼度に応じた共有を学びます。途中で止めても OK です。</p>
    <p class="iw3ip-phase-links">
      <a href="ha-ssi-wallet.md">HA SSI Wallet</a> /
      <a href="webcam-event-sharing.md">Webcam Event Sharing</a> /
      <a href="environment-disaster.md">Environment Disaster</a> /
      <a href="marketplace-vc-bridge.md">VC Bridge</a> /
      <a href="data-user-vc-tiered.md">DataUserVC Tiered</a>
    </p>
  </div>
  <div class="iw3ip-phase-card iw3ip-phase-3">
    <div class="iw3ip-phase-kicker">Part 3 / 知能統合</div>
    <h3>🧠 AI に要求を解釈させる</h3>
    <p>人の要求を AI が解釈し、Part 1/2 で集めたデータを材料に判断・処理する応用段階です。</p>
    <p class="iw3ip-phase-links">
      <a href="regional-safety-assistant.md">Regional Safety Assistant</a> /
      <a href="llm-planner.md">LLM Planner</a>
    </p>
  </div>
</div>

## 進む順 — 自分のコースを選ぶ

| 目的 | 進む順 |
|---|---|
| まず動かしたい | [HA Demo Simulator](ha-demo-simulator.md) のみ (15 分) |
| 基本を一通り | Part 1 全部 (60–90 分) |
| 安全な共有まで | Part 1 → Part 2 の 2.1〜2.3 (+ 60 分) |
| マーケット連携の v2 まで | Part 1 → Part 2 の 2.4 まで (+ 30 分) |
| 段階アクセス・semantic まで | Part 2 の 2.5 まで (+ 30 分) |
| AI 統合まで | Part 3 まで (+ 30 分) |

各ハンズオンページの先頭には **「このハンズオンで分かること」「前提」「使うもの」「所要時間」** を載せています。途中で詰まったら [トラブルシュート](../operations/troubleshooting.md) を参照。

---

## Part 1: 基本 — IoT データの基本経路

### この Part のゴール

- **データがどこで生まれて、どこへ届くか** を目で追えるようになる
- **マーケットプレイス v1 経路** (暗号化 IPFS 受け渡し) で売買を一往復できる

### 1.1 まず動かす — 実機なしで全体感をつかむ

実機がなくても、Home Assistant のシミュレータでハンズオン全体を体験できます。
最初に触るのに最適です。

- [Home Assistant Demo Simulator サンプル](ha-demo-simulator.md)

### 1.2 デバイスからデータを取る

カメラやセンサからデータを取り出して publisher に送る経路を作ります。
お手元の機材に合わせてどれか 1 つ選んでください。

- [HUSKYLENS2 サンプル](huskylens2.md) — 専用 AI カメラを使う
- [USB ウェブカメラサンプル](webcam.md) — USB カメラ + OpenCV
- [HA × SSI Publisher サンプル](ha-ssi-publisher.md) — Home Assistant 連携

### 1.3 スマホで見る

集めたデータをスマホブラウザで閲覧します。

- [Mobile Viewer サンプル](mobile-viewer.md)

### 1.4 マーケットプレイスで売買 (v1 lane)

出品 → 購入 → 暗号化 IPFS から復号して受け取り、までの一往復。
ここまでが **基本 (v1)** の完了点です。

- iot-market-ui で出品する
- MetaMask で購入する
- IPFS に保存された暗号化データを復号して受け取る

詳しい起動手順は [最短起動](../setup/quickstart.md) にあります。

### Part 1 の成功判定

- データが publisher に届いていることを `/platform/ingest` のログで確認できた
- マーケットで出品 → 購入 → 受信が一往復できた

---

## Part 2: 機能拡張 — 同意・条件付き共有・段階アクセス

### この Part のゴール

- **「全部見せる」ではなく「条件付きで見せる」** がなぜ必要か理解する
- **VC とウォレット** でその条件付き共有を実装する
- **段階アクセス (Tier 3 / 2 / 1)** で、信頼度に応じて見せる中身を変える

### 2.1 同意 (Consent VC) を入れる

共有の前に「この目的なら共有していい」という同意 VC を発行・提示する流れを学びます。

- [HA × SSI Wallet サンプル](ha-ssi-wallet.md)

### 2.2 イベント共有 — 生データではなく結果だけ

カメラの生映像ではなく「人物検出: あり」「ゴミ捨て: あり」のような **イベント** だけを送る方式を体験します。

- [USB ウェブカメライベント共有サンプル](webcam-event-sharing.md)
- [環境・防災イベント共有サンプル](environment-disaster.md)

### 2.3 役割の分離 — 閲覧と書き込みを分ける

データを「見るだけ」のロールと「書き込みだけ」のロールを VC で分けます。

- [HA × SSI Viewer サンプル](ha-ssi-viewer.md)
- [HA × SSI Service サンプル](ha-ssi-service.md)

### 2.4 マーケットプレイス × ウォレット (v2 lane)

購入後の受け渡しを暗号化 IPFS ではなく **PurchaseViewerVC + ViewerToken** で行う v2 経路を組み立てます。

- [Marketplace VC Bridge ハンズオン](marketplace-vc-bridge.md) — bridge を立てる
- [Marketplace VC end-to-end (Stage 6)](marketplace-vc-end-to-end.md) — 購入から閲覧までフルパス
- [Marketplace Seller VC (Stage 7)](marketplace-seller-vc.md) — 出品者側の VC
- [Marketplace Mobile App (Stage A)](marketplace-mobile-app.md) — 専用モバイルアプリ

### 2.5 信頼度に応じた段階アクセス (Stage T)

利用者の **DataUserVC** に基づき、Tier 3 (動画+全部) / Tier 2 (画像+派生) / Tier 1 (要約のみ) の 3 段階で見せ方を変えます。
最後に **意味的中間表現** + **trust-aware rendering** まで深掘りします。

- [DataUserVC Tiered Access ハンズオン](data-user-vc-tiered.md)
- [DataUserVC Tiered Access 仕様](data-user-vc-tiered-spec.md)

### Part 2 の成功判定

- Consent / DataUserVC / PurchaseViewerVC のいずれかをスマホ wallet に取り込めた
- `purpose` や信頼度を変えると、`/platform/data` の応答が `allow` / `deny` / 内容違いに切り替わる
- audit log に「誰が・いつ・どの purpose で・何を見たか」が残っている

---

## Part 3: 知能統合 — AI に要求を解釈させる

### この Part のゴール

- 人の自由文の要求 (例:「最近の異常を教えて」) を AI が **plan** に分解する
- plan を **execute** で実行し、Part 1/2 で集めたデータを材料に応答を返す
- frontend demo まで一気通貫で見せる

### 3.1 地域安全アシスタント

Part 1/2 で蓄積された「いつ・どこで・何が起きた」イベントを材料に、自由文の要求に応答するデモです。

- [Regional Safety Assistant サンプル](regional-safety-assistant.md)

### 3.2 LLM Planner

要求を **plan → execute → UI** に分解する設計の体験です。

- [LLM Planner ハンズオン](llm-planner.md)
- [LLM Planner 置き換え仕様](llm-planner-spec.md)

### Part 3 の成功判定

- 自由文の要求が plan ステップに分解されて実行される
- 実行結果が frontend demo に表示される
- planner_diagnostics でどう判断したかを追える

---

## 用語が分からなくなったら

[環境構築のミニ辞書](../setup/index.md) に戻ってください。
出てくる用語のほぼ全てを 1 行で説明しています。
