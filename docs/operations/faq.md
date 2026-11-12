# FAQ

進め方や位置付けに関する質問をまとめています。実行エラーの切り分けは [トラブルシュート](troubleshooting.md) を参照してください。

## Q. どのサンプルから始めるべきですか？

A. 実機があってもなくても、[事前準備 (ツールのインストール)](../setup/prerequisites.md) のあと、[実機なしで一連を動かす (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) (Part 1 の 1.1) → [最短起動 (スタックを動かす)](../setup/quickstart.md) → Part 1 の 1.2 の順に進みます。1.2 では、手元の機材に合わせて [AI カメラで人物を検知する (HUSKYLENS2)](../hands-on/huskylens2.md)、[USB カメラでポイ捨てを検知する (OpenCV)](../hands-on/webcam.md)、[Home Assistant のデータを取り込む (HA SSI Publisher)](../hands-on/ha-ssi-publisher.md) から 1 つ選びます。HA SSI Publisher を選ぶ場合は、最短起動を 1.2 のあと (1.3 の前) に行います。1 コマンドで起動できる `assistant-demo`（[AI に手順を考えさせる (LLM Planner)](../hands-on/llm-planner.md)）は Part 3 の内容で、初めての人には不要です。

## Q. 実機やスマホが無くても進められますか？

A. Part 1 は、スマホで見る手順 (1.4、任意) を除いて PC だけで進められます。[実機なしで一連を動かす (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) は実機なしで動き、HUSKYLENS2 と USB ウェブカメラのページにも mock モードがあります。Part 2 のうちウォレットを使うハンズオンでは、スマホが必要です。1.4 と Part 2 の Stage A では、スマホに MetaMask モバイルも入れます。

## Q. 実際のお金や暗号資産は必要ですか？

A. 不要です。ブロックチェーンは手元の PC で動かす Hardhat のローカルチェーンで、購入に使う ETH はテスト用のものです。

## Q. Part、Phase、Stage は何が違いますか？

A. Part はこのサイトでの学習順、Phase は基盤の発展段階、Stage は Part 2 の VC を中心とするハンズオンに付けた通し番号です (準備の Stage 0 と、画像の来歴を署名する Stage P も含みます。Stage P の P0 は VC を使わずに単独で動き、P1〜P3 は publisher とチェーンにつないで Tier ごとの ViewerToken を使います)。対応は [ハンズオン](../hands-on/index.md#part-phase-stage) にまとめています。
