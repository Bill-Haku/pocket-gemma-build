# Pocket Gemma 1.0.12：実機ログに基づく画像推論の切り分け

この報告は1.0.11のiPhone実機で得られたエラーを受けた追加調査です。実機の成功をシミュレーターの結果で代用しません。最終のIPA・実モデル検証を完了した結果を末尾に記載します。

## 確定できた範囲

言語生成GPUは実機で動作しています。一方、画像GPU構成は `conversation_creation` で `vision_litert_compiled_model_executor.cc:313`、`litert_compiled_model.h:1961`、`Failed to modify graph with delegate` に到達しています。

公式0.18.0の前者は画像エンコーダーの `CompiledModel::Create` の結果を受け取る位置です。後者はネイティブ `CreateCompiledModel` のエラーを伝える位置です。画像側のグラフ・デリゲート構築の失敗まで絞れますが、この一般的なエラーだけでは失敗デリゲートや演算子は確定しません。70と280の両方で失敗しているため、トークン削減を根本対策として扱いません。空きメモリの数値だけで、原因がメモリ不足であるとも断定しません。

これは前回シミュレーターで得られた、言語モデル側のMetal初期化エラーとは別の段階です。

## アプリ側の修正

- `GenerationSettings.swift`：画像エンコーダーのCPU/GPU設定を言語生成から独立。旧JSONに新しい項目がなくても読み込めるようにしました。旧設定の言語GPU・トークン数・プロンプト等を維持し、画像側の初期値だけCPUにします。
- `InferenceEnginePlan.swift`：保存した画像backendを本番のEngineConfigへ渡し、構成ごとのエンジン再利用・キャッシュ分離を維持。
- `GenerationConfigView.swift`：画像側の独立スイッチとGPU/GPU・GPU/CPU・CPU/CPUのプリセット。プリセットはモデル・トークン数・サンプリングを変更しません。
- `ChatStore.swift`：エラー後に自動で70へ変更する再試行を撤去。元の構成の結果を保持。
- `GemmaService.swift`、`NativeLogCapture.swift`、`NativeLogEvidence.swift`：処理段階・メモリ・利用できるstderrの運用ログを取得し、失敗と結び付いたデリゲート名、明記された演算子名、委譲ノード番号、ソース位置を分離。ネイティブstderrは元の出力先にも転送します。ログのない名前を設定から推測しません。
- `InferenceDiagnostics.swift`：直近24回の比較を端末内に保存・共有。OSLog、フィルターで除外した行、ネイティブが返さなかった詳細は取得できません。会話本文や画像を診断に含めません。
- `record-runtime-build.py`、IPA検証：実際に組み込まれたframeworkをハッシュ化し、Mach-OのCPU・プラットフォーム・SDK・最低OSを記録。記録のSHA256とIPA内のバイト列を再照合します。ハッシュの範囲はAltStoreが再署名する前のunsigned IPAです。

エラーの通知・元のメッセージを維持しています。画像機能の無効化、OCRによる代用、モデルのバイナリ編集、根拠のないランタイムのダウングレードは行っていません。

## ランタイムとモデル

2026-10-14に確認した公式最新安定版はLiteRT-LM **0.18.0**。SwiftPMの公式iOSアーカイブのchecksumは `d765b99592d4ec3d0c9e2bd69469454af06c834861340672da1891c0c121c347` です。

1.0.11の実際のIPA内のCLiteRTLMはARM64/iOS、最低OS15.0、ネイティブSDK26.4.0でした。SHA256は `02d99220cba485d5a781dcb64d92025211c22aefe55fab6fab4f973fad6e51be`。アプリのビルドSDKとネイティブライブラリのビルドSDKを区別します。SwiftPMのバージョン・ソースrevision・アーカイブchecksum・実バイナリを照合しますが、公式バイナリには完全なコンパイラフラグや独立した全構成部品のバージョンを自己申告する情報がありません。WORKSPACEのLiteRT revisionは公開タグのソース宣言として記録し、バイナリの自己申告バージョンとは扱いません。

モデルは差し替えていません。両モデルのLITERTLM 1.5.0コンテナには画像エンコーダー・画像アダプター等があり、画像signatureは70/140/280です。アダプターはモデル指定でCPUとなり、画面の画像backendはエンコーダーの設定です。

| モデル | SHA256（維持） |
|---|---|
| E2B | `181938105e0eefd105961417e8da75903eacda102c4fce9ce90f50b97139a63c` |
| E4B | `0b2a8980ce155fd97673d8e820b4d29d9c7d99b8fa6806f425d969b145bd52e0` |

## Gallery・関連Issueとの比較

Galleryの公開Android実装は言語・画像backendを独立に選び、画像とテキストを構造化した入力で送ります。Pocket Gemmaも公式Swiftの `.imageFile` と `.text` を使っています。今回、backendの独立選択を本番アプリにも反映しました。

Galleryの現行iOSソースは公開されていないとメンテナーが説明しています。そのため、iOSアプリの実際のバイナリ、ビルドフラグ、モデルSHA256との完全な照合はできません。Androidのデフォルト画像GPUが、この実機でも正解であるとは推定しません。

Issue #2979は別revisionの実機でCPU/XNNPACKのreshape、委譲ノード1480、画像送信時のtensor allocation失敗を報告しています。今回のGPU・会話作成時のエラーと同一原因であるとは確定しません。#2370は0.12.0でSTABLEHLO_COMPOSITE不足や、編集したE4B bundleのXNNPACK分割を報告しており、モデル形式・runtimeが異なります。これを根拠に現在のモデルを編集しません。CPU構成で同じ署名が出れば、次の診断で名前と段階を比較できます。委譲ノード1480は、そのまま原因となるモデル演算子の番号ではありません。

ライブラリ側のデリゲート・演算子の問題が実機で再現した場合は、アプリのbackend選択で回避可能かと、ライブラリの演算子登録・実装・Metal/XNNPACKの修正が必要かを分けて扱う必要があります。0.18.0のリリース説明には、この実機の症状の解消を明記した項目はありません。

## 次の実機比較

1. 1.0.12へ更新し、同じモデルを使います。再ダウンロードは不要です。
2. モデル構成で言語生成GPU、画像CPUを選び、元と同じトークン数のまま、新しい会話で同じ画像を送信します。
3. 必要に応じてGPU/GPU、CPU/CPUも新しい会話で比較します。試す順番によってモデル・トークン数を変えないでください。
4. 「モデルと設定 → 推論の診断 → 比較結果と診断を共有」で成功・失敗を含む結果を確認できます。

実機のGPU/CPU成功、特定の失敗演算子、iOS版Galleryとのバイナリ一致は、まだ未検証です。

参照：[公式0.18.0画像実装](https://github.com/google-ai-edge/LiteRT-LM/blob/v0.18.0/runtime/executor/vision_litert_compiled_model_executor.cc)、[公式Swift設定](https://github.com/google-ai-edge/LiteRT-LM/blob/v0.18.0/swift/Config.swift)、[公式Package.swift](https://github.com/google-ai-edge/LiteRT-LM/blob/v0.18.0/Package.swift)、[リリース](https://github.com/google-ai-edge/LiteRT-LM/releases/tag/v0.18.0)、[Issue #2979](https://github.com/google-ai-edge/LiteRT-LM/issues/2979)、[Issue #2370](https://github.com/google-ai-edge/LiteRT-LM/issues/2370)、[Gallery #634](https://github.com/google-ai-edge/gallery/issues/634)。


## 最終検証

| 構成（言語 / 画像エンコーダー） | E2B | E4B |
|---|---|---|
| GPU / GPU | シミュレーターの言語GPU初期化で停止 | 同左 |
| GPU / CPU | シミュレーターの言語GPU初期化で停止 | 同左 |
| CPU / CPU | 赤い四角の画像について回答 | 赤い四角の画像について回答 |

比較はContext 4096 / Vision 280、同じ合成JPEG、同じモデルSHA256で行いました。モデルごとの画像の後にテキストの計算問題へ回答し、E2B→E4B→E2Bの切り替えと、両モデルの不正JPEG→正常JPEGも検証しました。7件成功・失敗0・スキップ0です。この7件には、GPU制約の検出やログ取得自体のテストも含むため、「7件すべてがGPU画像認識に成功した」という意味ではありません。

アプリのコア51件、iPhone操作17件、iPad初回起動1件も成功。最初の候補ではファイル選択とスキャンPDFのUIテストが失敗しましたが、最終候補で両方成功しました。これらの判定条件を緩める変更は行っていません。

アプリソースZIPのSHA256は `d1a08b5ce7d5b5755ad47b9de778dc215b906053b301a15f5a14528155345110`。IPAは `d491e4856d13c0a23ce7677f247bb05c044bf9576d7fe73e648d023a939b6d3f` です。IPAビルドcommitは `e1bad9befa3b8f553115b6a6526248c384963f94`、ネイティブ検証commitは `8507ec6b552b3fea5b27a9aa66696078b9b83fe5`。両commitでアプリのソースZIPが同一であることを匿名読み取りで照合しました。後者では検証用のJSON書き出しと検証アプリのバージョン表示だけを修正しています。

[IPAビルド](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38448414222)、[実モデル検証](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38452549006)、リリース添付の `PocketGemma-1.0.12-build.json` と `NativeVision-1.0.12-verification.json` に照合情報を含みます。大きなUI映像artifactの全体はダウンロードせず、GitHubの実行ログで18件の成功を確認しています。最新UI映像の目視確認や実機検証を行ったとは報告しません。
