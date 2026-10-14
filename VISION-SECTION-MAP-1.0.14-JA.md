# Pocket Gemma 1.0.14 (15): セクション読み込みの診断ログ

1.0.13で見えなかったLiteRT-LMのセクション読み込み警告を保持する、小さな診断更新です。

`NativeLogEvidence.swift` に提供された許可リストの3項目を追加しました。
`Failed to map section`、`Section not found:`、`not found. Skipping` です。
通常の推論と最小再現テストの両方で、既存のstderrキャプチャを通じて記録されます。
会話JSON・prompt/responseの除外、パス等の伏せ字、64行/8192バイトの上限は維持しています。

公式0.18.0の `LitertLmLoader::GetSectionBuffer` は、セクション不在でもmmap失敗でも
`nullopt` を返します。`GetTFLiteModel` が空のバッファを返し、
`ModelResourcesLitertLm::GetTFLiteModel` は `not found in the model` にします。
この最後の文言だけから、モデル欠落やmmap失敗を確定できません。
警告自体も任意のコンポーネントについて出るため、該当試行・コンポーネントと併せて読みます。

参照:
- [公式0.18.0 loader](https://github.com/google-ai-edge/LiteRT-LM/blob/v0.18.0/runtime/util/litert_lm_loader.cc#L346)
- [公式0.18.0 model resources](https://github.com/google-ai-edge/LiteRT-LM/blob/v0.18.0/runtime/components/model_resources_litert_lm.cc#L129)

追加テストは、分割されたstderrと改行なしの末尾を含む3種類の警告の保持、
元のエラー理由とパス伏せ字の両立、許可語が含まれる会話JSON・prompt/responseの除外です。
警告からXNNPACKや演算子を推測して埋める処理は追加していません。

モデルのSHA256、公式LiteRT-LM 0.18.0、CPUスレッド数/フラグ、バックエンド、
メモリチェック、画像処理、VisionRuntimeBridge、保存済み設定を維持しています。
GPUからCPUへの自動再試行は有効化していません。
Nightlyは試験していません。実機の部品テスト結果を見てから判断します。

実機検証はこの環境では実施できません。更新後は次の順で確認してください。
1. E4Bのエンコーダー単体をXNNPACK／組み込みCPUで比較。
2. 同じE4Bのアダプター単体も両方で比較。
3. 「テストA」を元と同じ構成で再実行し、nativeLogを含む比較JSONを共有。
4. 通常のテキスト推論を確認。

組み込みCPUだけ成功しても、画像会話全体が動いたとは扱いません。
両方の失敗や成功も、合成入力と通常会話の条件差・失敗段階を併せて評価します。
実機の画像認識解消を確認した版ではありません。

## 今回の検証

コア58件（追加した警告の保持と会話除外の2件を含む）、iPhone操作18件、iPad起動1件が成功しました。
これらはmacOSランナーのコアテストとiOSシミュレーターのUIテストです。
今回の1.0.14では実モデルの推論テストを再実行していません。
1.0.13での11件の実モデル検証は、[元の検証記録](https://github.com/taotaohoyo/pocket-gemma-build/releases/download/v1.0.13/NativeVision-1.0.13-verification.json)に保持しています。
過去のシミュレーター成功を1.0.14の実機成功として数えません。

IPAのARM64/iPhoneOS形式、version/build、同梱の公式CLiteRTLMバイナリと合成JPEGのSHA256を照合しました。
同梱ネイティブSHA256は1.0.13と同じです。AltStore再署名前の照合であり、端末内バイナリの読み戻しではありません。

IPA SHA256: `0417329215dde24c6f7f7ae01756efed9f1309b8b6d9b0258335dfa8542a54fe`。
アプリソースZIP SHA256: `4681e93a60391e106560d91fa773ee088b0d19d1778e3292ec96973ce3c0c5fc`。
ビルドcommit: `74bd92a0d35c0742bf4c37d54e08009156cab407`。
[ビルドとテスト](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38543809798)。

Nightly・実機・Gallery実バイナリの比較は未実施です。
既存の1.0.13の部品テストは、次の実機検証にそのまま使えます。
ソース上で確認した警告の経路と、利用者のテストAで実際にmmapが失敗したかという結論は分けています。
この環境にはテストA/B/Cの比較JSONがないため、原因の確定には実機ログが必要です。
