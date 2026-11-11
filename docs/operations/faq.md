# FAQ

進め方や位置付けに関する質問をまとめています。実行エラーの切り分けは [トラブルシュート](troubleshooting.md) を参照してください。

## Q. どのサンプルから始めるべきですか？

A. 実機なしで Phase 1 から順に確認する場合は [実機なしで一連を動かす (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) から始めてください。Phase 3 のデモを 1 つだけ動かすなら、1 コマンドで起動できる `assistant-demo`（[AI に手順を考えさせる (LLM Planner)](../hands-on/llm-planner.md)）が最短です。実機を使う場合は、手元の機材に合わせて [AI カメラで人物を検知する (HUSKYLENS2)](../hands-on/huskylens2.md) か [USB カメラでポイ捨てを検知する (OpenCV)](../hands-on/webcam.md) から始めます。

## Q. 実機やスマホが無くても進められますか？

A. Part 1 は、スマホで見る手順 (1.4) を除いて PC だけで進められます。[実機なしで一連を動かす (HA Demo Simulator)](../hands-on/ha-demo-simulator.md) は実機なしで動き、HUSKYLENS2 と USB ウェブカメラのページにも mock モードがあります。Part 2 のうちウォレットを使うハンズオンでは、スマホが必要です。

## Q. 実際のお金や暗号資産は必要ですか？

A. 不要です。ブロックチェーンは手元の PC で動かす Hardhat のローカルチェーンで、購入に使う ETH はテスト用のものです。

## Q. Part、Phase、Stage は何が違いますか？

A. Part はこのサイトでの学習順、Phase は基盤の発展段階、Stage は Part 2 の VC を中心とするハンズオンに付けた通し番号です (準備の Stage 0 と、画像の来歴を署名する Stage P も含みます。Stage P の P0 は VC を使わずに単独で動き、P1〜P3 は publisher とチェーンにつないで Tier ごとの ViewerToken を使います)。対応は [ハンズオン](../hands-on/index.md#part-phase-stage) にまとめています。
