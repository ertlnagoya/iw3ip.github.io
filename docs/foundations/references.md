# 参考文献・発展学習資料

サイト内の学習を終えたあとに参照する補助資料です。基本理解とハンズオンはサイト内の説明だけで進められます。

一度にすべてを読む必要はありません。サイト内で全体像を把握した上で、社会課題や技術仕様を詳しく調べる際に参照してください。

## 社会課題を理解する資料

### プライバシーと防犯・見守り

顔識別機能付きの防犯カメラについては、法的な整理と検討課題が個人情報保護委員会の資料 [1] にまとめられています。防犯カメラの設置をプライバシーとの調和から論じた研究 [2] と、地方自治体による公共空間への大規模な設置事業の実態を調べた研究 [3] は、カメラの設置と運用に伴う課題を考える手がかりになります。

### 行方不明者・落とし物・公共的情報共有

行方不明者の届出の件数と内訳は、警察庁の統計資料 [4] で確かめられます。落とし物については、オンラインでの手続 [5] と届出・検索 [6] が警察庁から案内されています。

## 技術を学ぶ資料

### ブロックチェーン基礎

スマートコントラクトを含むブロックチェーンの仕組みは、Ethereum の開発者向け文書 [7] で体系的に学べます。ブロックをハッシュでつなぐ考え方の背景は、Bitcoin の論文 [8] で確かめられます。

### Hardhat

本サイトで使う Hardhat の機能は公式の文書 [9] に、最初の使い方は入門の文書 [10] にまとめられています。

### SSI / DID / VC

DID と VC の仕様は、W3C の DID Core [11] と Verifiable Credentials Data Model 2.0 [12] で定められています。SSI に関する標準化と実装の取り組みは、Decentralized Identity Foundation [13] で追うことができます。

### 実装に近い読み物

API を公開するときに注意すべき脆弱性は OWASP API Security Top 10 [14] に、アクセスのたびに検証するゼロトラストの考え方は NIST SP 800-207 [15] にまとめられています。

## 論文（本サイト掲載）

IW3IP に関する発表論文は [論文](../publications.md) のページにまとめています。

## 推奨する使い方

1. まず本サイト内の基礎章と Hands-on を完了する
2. 次に、社会課題の資料を読んで「なぜこの基盤が必要か」を確認する
3. その後、技術資料で各要素技術を詳しく調べる
4. 最後に、論文で研究的背景を体系的に追う

## 図版について

- 本サイトの図は主に独自作成（Mermaid）です。
- 論文図版を利用する場合は、著作権・利用条件・出典明記を必ず確認してください。

## 参考文献

[1] 個人情報保護委員会, 「顔識別機能付き防犯カメラの利用に関する法的整理と検討課題」, 2022, <https://www.ppc.go.jp/files/pdf/20220128_shiryou-2_kentoukadai.pdf>

[2] 千葉尚路, 樋野公宏, 「プライバシーと調和する都市空間の防犯カメラ設置のあり方に関する研究 論点・裁判例の整理と新聞記事分析」, 都市計画報告集, Vol.16, No.2, pp.124–128, 2017. DOI: [10.11361/reportscpij.16.2_124](https://doi.org/10.11361/reportscpij.16.2_124)

[3] 村中大輝, 雨宮護, 大山智也, 「地方自治体による公共空間への防犯カメラ大規模設置事業の取り組み実態と評価」, 都市計画論文集, Vol.51, No.3, pp.357–364, 2016. DOI: [10.11361/journalcpij.51.357](https://doi.org/10.11361/journalcpij.51.357)

[4] 警察庁, 「令和6年における行方不明者届受理等の状況」, <https://www.npa.go.jp/publications/statistics/safetylife/R6_yukuefumeishakouhoushiryou2.pdf>

[5] 警察庁, 「オンラインでの申請等の案内」, <https://www.npa.go.jp/policies/application/shinseisys/>

[6] 警察庁, 「落とし物の届出・検索」, <https://www.npa.go.jp/bureau/soumu/ishitsubutsu/ishitsu-todokedekensaku.html>

[7] Ethereum Foundation, "Ethereum development documentation," <https://ethereum.org/en/developers/docs/>

[8] S. Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System," 2008, <https://bitcoin.org/bitcoin.pdf>

[9] Nomic Foundation, "Hardhat 2 Documentation," <https://v2.hardhat.org/docs>

[10] Nomic Foundation, "Getting started with Hardhat (Hardhat 2)," <https://v2.hardhat.org/hardhat-runner/docs/getting-started>

[11] W3C, "Decentralized Identifiers (DIDs) v1.0," W3C Recommendation, 2022, <https://www.w3.org/TR/did-1.0/>

[12] W3C, "Verifiable Credentials Data Model v2.0," W3C Recommendation, 2025, <https://www.w3.org/TR/vc-data-model-2.0/>

[13] Decentralized Identity Foundation, <https://identity.foundation/>

[14] OWASP Foundation, "OWASP API Security Top 10," <https://api-security.owasp.org/>

[15] S. Rose, O. Borchert, S. Mitchell, and S. Connelly, "Zero Trust Architecture," NIST Special Publication 800-207, 2020. DOI: [10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207)
