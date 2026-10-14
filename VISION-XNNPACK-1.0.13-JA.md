# Pocket Gemma 1.0.13：XNNPACK reshapeの切り分け

## 実機ログで確定したこと

1.0.12 (13) のGPU/CPUは、会話作成を通過し、画像prefillで失敗しています。画像アダプターもCPUです。失敗したデリゲートはXNNPACKで、委譲ノードは2374。画像1枚、Vision 280、Context 4096、MTP無効でも失敗し、テキスト入力は成功しています。

公式ソースでは、XNNPACKが外部入力の形状を設定した後、`xnn_reshape_runtime()` の成功以外の戻り値を一律にTFLiteエラーへ変換しています。その後の `AllocateTensors` エラーは、このPrepare失敗を包む結果です。ログ中の「Failed to allocate tensors」だけでは、ヒープ確保失敗やiOSのメモリ制限超過を確定できません。

最も直接的な問題箇所は **CPU/XNNPACKの実行準備・内部reshape** です。ただし、XNNPACK内部の戻り値、演算子、形状、確保バイト数が現在のログにないため、形状不整合・実装やCPU命令の互換性・一時メモリ確保のどれかは未確定です。今回の版は根本原因が確定した修正版とは報告しません。

参照：[XNNPACK delegate Prepare](https://github.com/google-ai-edge/LiteRT/blob/26895c9fbcc25c43faa8c1a98cd1fd28951602c3/tflite/delegates/xnnpack/xnnpack_delegate.cc#L1526)。

## Node 2374とモデルの形状

変更していない公式モデルを、コンテナの範囲読み取りとTFLite FlatBufferの参照で調べました。

| 項目 | E2B / E4Bの画像エンコーダー |
|---|---|
| 画像モデル形式 | TFLite、LITERTLM 1.5.0コンテナ内 |
| 画像エンコーダーのサイズ | 224,137,936 bytes |
| signature | vision_70 / vision_140 / vision_280 |
| 各入口グラフの演算子数 | 1477 |
| 各入口グラフのテンソル数 | 1729 |
| STABLEHLO_COMPOSITE | 入口ごとに112個、RMS norm用の内部グラフを参照 |
| コンポーネント全体のsubgraph数 | 339 |
| 280の画像入力 | FLOAT32 [1,2520,768] |
| 280の位置入力 | INT32 [1,2520,2] |
| 280の出力 | BOOL [1,280]、FLOAT32 [1,280,768] |
| 70 / 140のパッチ容量 | 630 / 1260 |

各入口グラフにはFLOAT32テンソル1302個、INT8テンソル352個、INT32テンソル54個、BOOLテンソル21個があります。量子化の例として、attentionの[768,768]テンソルはINT8、スケール768個、zero pointはすべて0、量子化軸0でした。FLOAT32入出力とINT8量子化が混在する形式です。コンテナの `prefer_activation_type=fp16` という希望値から、CPU側の全テンソルがFP16だとは判断しません。

モデル上の1477個の演算子に2374というインデックスを直接当てることはできません。実行時には複合演算の展開や委譲でグラフと採番が変わります。`TfLiteXNNPackDelegate` のノードは複数演算子をまとめた計算領域です。内部の失敗演算子は未特定です。現在の公開LM APIからその領域の全元ノード一覧は取得できません。

モデルの画像アダプターは別のTFLiteセクションで、CPUというbackend制約を持ちます。画面の「画像backend」はエンコーダーの指定であり、アダプターをGPUへ変更したとは扱いません。単独テストでは、各セクションの実際の入力・出力レイアウトをC APIから記録します。モデルのメタデータ、量子化、重みは変更しません。

最終の単独実行で抽出部分のSHA256も照合しました。E2BとE4Bのエンコーダーは形状が共通ですが、セクション全体のバイト列は異なります。同じサイズ・形状だけを理由に同じ重みだとは扱いません。

| 部分 | E2B SHA256 | E4B SHA256 |
|---|---|---|
| エンコーダー | `6b2f6475a7b25e1954e73bc0b994dd852d3ecc0a1abfcf05e29f7861c3a2b99a` | `075c1ab8cfc598eb7c2821159c5a1293d851c4dcde5ea3a782b178c6a5df0af8` |
| アダプター | `b759f1d7332a06fff57261aa66137b33834d6ec347906ee7f10749621d671584` | `babd89a552fb90f1838d497c8141c2ea93d8acddd39d3fe8c6bef7e6d038049f` |

アダプターは両モデルともFLOAT32入力 `[1,280,768]`。出力はE2Bが `[1,280,1536]`、E4Bが `[1,280,2560]` で、各言語モデルへの接続寸法は異なります。最終のCPU/CPU直接JPEG検証では、正方形入力の実際のリサイズが768×768・2304パッチであることをネイティブINFOログから確認しました。280は出力のvisual token budgetで、画像パッチ数そのものではありません。

## ランタイムとGalleryとの比較

2026-10-14に確認した最新公式安定版はLiteRT-LM **0.18.0** です。リリース説明にこのGemma 4 iOS/XNNPACKエラーの修正を明記した項目はなく、関連Issueも解決済みの公式互換構成を提示していません。

- Pocket Gemma：公式SwiftPM 0.18.0、revision `b2f686e2ed4718fb84ec398a61dd59ca0f0aff27`、公式XCFramework checksum `d765b99592d4ec3d0c9e2bd69469454af06c834861340672da1891c0c121c347`。
- unsigned 1.0.12 IPAのARM64 CLiteRTLM実バイナリ：61,591,392 bytes、SHA256 `02d99220cba485d5a781dcb64d92025211c22aefe55fab6fab4f973fad6e51be`。SDK26.4.0、最低iOS15.0。AltStore再署名前の照合であり、端末内の再署名済みバイナリを読み出した結果ではありません。
- 公式0.18.0のソース宣言はLiteRT revision `26895c9fbcc25c43faa8c1a98cd1fd28951602c3`。完全なネイティブコンパイラフラグやXNNPACKの独立した実バイナリrevisionを自己申告する情報はありません。ソース宣言と実バイナリ内のバージョンを混同しません。
- Galleryの公開リポジトリは現在もAndroidのみ。公開iOS allowlistはGemma 3n等を掲載しており、現行Gemma 4 iOSアプリの実際のモデルSHA256・ランタイムバイナリ・ビルド設定を照合できません。
- 公開Android実装には言語・画像backendの独立指定と構造化画像入力があります。Pocket Gemmaも公式Swiftの画像＋テキスト入力を使用しています。ただし、Androidの実装やGalleryの「GPU」という表示だけでiOSの内部画像backendが同一とは判断できません。

Issue #2979は、同じE2B SHA256のJPEG直接入力でもCPU/XNNPACKが失敗した実機報告で、写真選択処理を唯一の原因にする根拠はありません。#2370は0.12.0と編集済みE4B bundleを含む別構成です。そこで報告された162演算子の委譲漏れを、現在の未編集E4Bにそのまま適用しません。古いcommitへの変更を勧めるコメントも、具体的な互換revisionと再現結果を示していません。

別の既知事例[LiteRT #9215](https://github.com/google-ai-edge/LiteRT/issues/9215)では、rank 7のFULLY_CONNECTEDをXNNPACKが受け入れ、Prepareで失敗しています。今回の未変更の画像エンコーダーを全339グラフで調べると、FULLY_CONNECTEDの入力・出力はrank 3、全テンソルの最大rankは4でした。その特定のrank 7再現を今回の根本原因とは扱いません。また[cache伝播のIssue #2152](https://github.com/google-ai-edge/LiteRT-LM/issues/2152)については、公式0.18.0のC APIで言語・画像・音声へのSetCacheDir呼び出しを確認し、同じ修正を重ねて適用する根拠はありません。

参照：[公式リリース](https://github.com/google-ai-edge/LiteRT-LM/releases/tag/v0.18.0)、[Issue #2979](https://github.com/google-ai-edge/LiteRT-LM/issues/2979)、[Issue #2370](https://github.com/google-ai-edge/LiteRT-LM/issues/2370)、[Gallery iOSソース公開の回答](https://github.com/google-ai-edge/gallery/issues/634)、[iOS allowlist](https://github.com/google-ai-edge/gallery/blob/4928788ef07cdf2194fc435a0f446cf3b997bd5d/model_allowlists/ios_1_0_0.json)。

## 今回実装した診断

- `VisionDiagnosticRunner.swift` / `VisionProbeView.swift`：同梱の512×512と1200×1600の合成JPEGを公式Conversation APIへ直接送ります。Photos、AttachmentStorage、Swift側のリサイズ・再圧縮を使いません。Context4096、Vision280、MTP/Thinking無効を固定し、言語・画像backendを個別に選べます。
- `VisionRuntimeBridge`：既存の公式バイナリに含まれる公開LiteRT APIで、画像エンコーダーとアダプターを別々に実行します。各モデルの検証済みセクションをバイト単位でそのままコピーし、抽出部分のSHA256も記録します。通常の言語エンジンを解放した状態で、合成テンソルをXNNPACKと組み込み最適化CPUで比較できます。
- 組み込みCPUの `LiteRtCpuKernelModeBuiltin` はCPUデリゲートを使わない公式オプションです。ただし、この設定はSwift LMのEngineConfigには公開されていません。**単独コンポーネントの成功は、画像会話の成功ではありません。** 本番会話に適用するには、互換性を検証したLM側の設定公開やネイティブビルドが必要です。
- `NativeLogEvidence.swift` / `NativeLogCapture.swift`：runtime reshapeとexternal-value reshapeを区別し、利用できる内部XNNPACK演算のエラーを保持。INFOで取得できるリサイズ後のパッチ数も記録します。ログのない演算子名やパッチ数は推測して埋めません。
- `InferenceMemoryCapture.swift` / `InferenceDiagnostics.swift`：200ms間隔のアプリ使用メモリのピークと利用可能メモリの最小値を記録。サンプル間の瞬間的なピークとOSのjetsam制限は未観測です。
- `GemmaService.swift` / `ChatStore.swift`：検証中の並列推論を防止し、通常のエンジンを解放。会話履歴、保存済み設定、モデルの重みを変更しません。通常の画像送信にも寸法・利用可能なパッチログ・メモリ観測を追加しました。

エラーを隠したり、OCRへ置き換えたり、画像入力を無効にしたりしていません。トークン削減、添付枚数制限の変更、モデルの書き換え、メモリ制限の無条件解除も行っていません。

## 参考パッチの扱い

提示された自動CPU/CPU再試行は、実機でCPU/CPUが成功した場合に検討できるアプリ側の回避策です。今回の画像側はすでにCPUなので、言語GPUをCPUにしただけでXNNPACKの不具合が直るとは断言できません。最初の `NativeLogCapture.finish()` の後に同じcaptureを使うと、2回目のログが欠けます。固定300msの待機も、ネイティブ処理の終了を保証する根拠にはなりません。

この版では構成を自動で変えず、CPU/CPUを明示的に比較できるようにしました。将来採用する場合も、要求構成と実行構成、試行ごとのログ、キャンセル、最終コールバックの終了、成功条件を分離して検証する必要があります。

## 実機で確認する項目

1. E4Bで「同梱JPEG・正方形」を言語GPU/画像CPUで実行し、同じJPEGをCPU/CPUで比較。
2. 同じ構成で縦長JPEGを比較。画像1枚、Context4096、Vision280を維持。
3. E4Bのエンコーダー単体をXNNPACK／組み込みCPUで比較し、アダプターも同様に比較。
4. E2Bでも同じ検証を実施。「比較結果と診断を共有」で成功・失敗の両方を確認。
5. 検証後、通常のテキストと画像の会話を試す。元の保存済みbackend設定を引き続き使用します。

内部XNNPACKログが得られない場合は、その失敗statusと領域の元ノード・shapeを出すネイティブソース側の計測が必要です。組み込みCPUが成功しても、Galleryと同じ実装になったとは報告しません。

## 自動検証

最終IPAのコア56件、iPhone操作18件、iPad初回起動1件が成功。ネイティブ11件も成功し、失敗・スキップは0です。E2B/E4Bの正方形・縦長の直接JPEG入力はCPU/CPUで赤い四角について回答しました（4試行）。エンコーダーとアダプターのXNNPACK／組み込みCPU単独実行は両モデルで成功し、出力にNaN/Infinityはありませんでした（8試行）。

通常のテキスト、モデル切り替え、不正JPEGのエラー後の復帰も検証しました。GPU/GPUとGPU/CPUの計4試行は、シミュレーターの言語Metal初期化制限で停止しています。実機のimage_prefill/XNNPACKの再現や解消ではありません。実機未検証を維持します。

単体結果には `testedComponent` と `inputPath=isolated_component_synthetic_tensors` を付け、JPEGの会話とは分けています。数値tensor型はLiteRT enumの1=FLOAT32、2=INT32、6=BOOLです。最初の2候補は新設橋渡しのC++／Swiftコンパイルで失敗し、修正後の最終候補で検証しています。コンパイル失敗をモデル推論の失敗とは数えません。

IPA SHA256: `0362e826837e7fe582d80be7f328b3ec32dd5ad74c9af68e5ee83ef8eb43f351`。アプリソースZIP SHA256: `004fbe1867fe96c0e7e3f36e2d64eb306fef57d86343f4bd00bf03fdbc3a4e51`。ビルドcommit: `f10ba1b12094864c5f757f3447394d1eeed94108`、ネイティブ検証commit: `f10ba1b12094864c5f757f3447394d1eeed94108`。同じアプリソースを使用しています。

[IPAビルド](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38481814900)、[実モデル検証](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38481615668)とリリース添付の照合JSONに実行・SHA256・構成別結果を記録しました。実機・最新UI動画の目視確認を行ったとは報告しません。
