# Pocket Gemma：Gemma 4 E2B・E4B画像認識の調査

2026年10月14日。アプリで確認した不具合を修正しました。実際のモデルを使うiOSシミュレーターで両モデルの画像認識が成功しています。iPhone実機での成功と、使用中の端末の直接の失敗原因はまだ確認できていません。

1. **原因と根拠**

   1.0.10の画像送信処理には `needsVision ? .gpu : settings.backend` があり、CPUでテキストが動いている場合も画像の送信でメイン推論をGPUに切り替えていました。画像の有無で利用者の設定を変更しないよう修正しました。

   画像のトークン予算は会話に設定するだけで、エンジン初期化前の `ExperimentalFlags.visualTokenBudget` に設定されていませんでした。そのため70への再試行でも、初期化した画像エンコーダーの容量は下がっていませんでした。今回の標準モデルは初期値280なので、この省略だけが通常の280設定でのエラーの原因だとは断定できません。

   エンジン再利用の判定には画像予算とspeculative decodingの変更が含まれておらず、キャッシュもこれらの構成を分離していませんでした。再利用判定とキャッシュの構成をそろえました。利用者の端末で古いキャッシュが直接の原因だったかは未確認です。

   iOS版ライブラリのGPU検証では、画像の読み込みより前のメインデコーダー初期化でMetalが「texture binding has argument index 31 that is greater than 30」を返しました。それが「Failed to modify graph with delegate」として伝わります。ただし、このシミュレーターではGPUのテキストも同じ段階で失敗します。利用者の実機ではテキストが動いているため、これをその端末の画像エンコーダーの失敗原因として断定しません。

2. **エンジンとモデル**

   推論は公式LiteRT-LM Swift **0.18.0**です。[公式リリース](https://github.com/google-ai-edge/LiteRT-LM/releases/tag/v0.18.0)と最新リリースAPIを確認し、調査時点で最新の安定版でした。この症状の修正を明記した新しいリリースは見つかっていません。根拠のないバージョン変更や独自ネイティブバイナリーへの交換はしていません。

   モデルはHugging Faceのlitert-community配布の通常版 `.litertlm` です。E2Bは2,588,147,712 bytes、E4Bは3,659,530,240 bytes。完全なファイルをダウンロードし、アプリの固定SHA-256と一致することを確認してテストしました。正確なリビジョン・ハッシュは[詳細報告](VISION-INVESTIGATION.md)に記録しています。

   両方のLITERTLM 1.5.0コンテナにVision Encoder、Vision Adapter、tokenizerとデコーダーがあります。画像エンコーダーは同じ224,137,936 bytesで、優先する活性値はFP16、アダプターはCPUです。実際のエクスポートにある画像予算は**70・140・280**で、存在しない560・1120を選べないようにしました。GPU専用の別配布ファイルは画像コンポーネントを持たないため、画像対応の代替品として使用していません。

   E2Bの公式モデルカードには重みの混合2/4/8 bit量子化が記載されています。E4Bの全テンソルの量子化形式を個別に検証したわけではありません。ファイルサイズをそのままピークメモリ量とは扱っていません。実機のピークメモリと応答速度は未計測です。

3. **修正ファイル**

   | ファイル | 変更 |
   |---|---|
   | `PocketGemma/Services/GemmaService.swift` | CPU・GPU設定の維持、初期化前の画像予算、構成に基づく再初期化、エラー後の復帰 |
   | `Packages/PocketCore/Sources/PocketCore/InferenceEnginePlan.swift` | モデルSHA、backend、画像予算、コンテキスト、MTPを含む再利用・キャッシュ判定 |
   | `Packages/PocketCore/Sources/PocketCore/ModelSpec.swift`、`GenerationSettings.swift` | E2B・E4Bで選べる画像予算を実際のエクスポートに制限 |
   | `PocketGemma/Services/ChatStore.swift` | delegate・テンソル確保エラーに限り、返答前に70の新しいエンジンで一度再試行 |
   | `PocketGemma/Views/InferenceDiagnostics.swift`、`ModelView.swift` | 端末内の実行段階・構成・メモリとネイティブエラーの診断／任意の共有 |
   | `PocketGemma/Views/GenerationConfigView.swift` | 有効な画像予算とCPU・GPUの説明 |
   | `PocketGemma/Views/ChatView.swift`、`ChatComposer.swift` | 回帰テストで見つかった日本語・絵文字送信時のキーボード終了の競合を修正 |
   | コアテスト、`NativeTests/NativeVisionTests.swift`、各検証スクリプト | 構成変更、実モデルの画像・テキスト・切り替え・不正画像後の復帰を確認 |

   画像認識の無効化、画像をOCRや分類だけで置き換える処理、エラーを成功に見せる処理は追加していません。モデルの保存先、アプリID、会話履歴とモデル一覧を維持しています。

4. **Galleryとの比較、既知の不具合**

   [Googleの担当者の説明](https://github.com/google-ai-edge/gallery/issues/634)と公開ツリーを確認しました。GalleryのiOSアプリのSwiftソースは公開されておらず、公開予定も示されていません。最新のiOS版で使うバイナリー・ビルドフラグ・モデルのハッシュとの完全比較はできません。

   公開Android実装は言語・画像のbackendを別々に扱い、画像と文章を構造化メッセージにして渡します。Pocket Gemmaも公式Swiftの `.imageFile` と `.text` を使います。画像用の特殊トークンを自作していません。写真の一時URLが失効する前にコピーし、ImageIOで向きを適用し、最大768pxのJPEGへ変換します。このアプリのインポーターと推論サービスを実モデルのテストに使用しました。

   [#2979](https://github.com/google-ai-edge/LiteRT-LM/issues/2979)は、同じE2BファイルのCPU画像経路でXNNPACK Node 1480がreshape／テンソル確保に失敗する報告です。[#1933](https://github.com/google-ai-edge/LiteRT-LM/issues/1933)は0.10.1のE4B画像prefillでネイティブのnullポインターが発生する報告です。バージョンと段階が異なり、どちらも今回の実機エラーと同じ原因だとは確認できていません。両Issueとも調査時点で未解決です。

5. **テストと残る制約**

   [先に完了したiOS実推論検証](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38401334155)は、iOS 26.2／arm64シミュレーター、公式iOSライブラリと完全なモデルを使用して10件すべて成功、失敗0、スキップ0でした。UIのサンプル返信を画像認識の結果として扱っていません。
   実推論テストの設定はコンテキスト2,048、出力128トークン、TopK 1、TopP 1、Temperature 0です。アプリの初期値4,096を含む全設定の組み合わせと、実機のメモリ制限を検証済みとはしていません。

   | 確認 | E2B | E4B |
   |---|---|---|
   | CPUメイン＋CPU画像、予算70／280 | 赤い四角に回答 | 赤い四角に回答 |
   | CPUメイン＋GPU画像、予算70 | 赤い四角に回答 | 赤い四角に回答 |
   | アプリのインポーターとCPUサービス | 画像回答＋テキスト回答5 | 画像回答＋テキスト回答5 |
   | E2B→E4B→E2B切り替え | 切り替え後も画像とテキスト8に回答 | 切り替え後も画像とテキスト8に回答 |
   | 不正JPEGのエラー後、正常JPEGへ復帰 | エラーを返し、次の画像に回答。クラッシュなし | 最終検証でもエラー後に赤い四角へ回答。クラッシュなし |

   Macの実モデル検証も10件成功しましたが、MacのGPUはWebGPU経路なのでiPhoneのMetalの成功の証拠とは扱っていません。GPUメインを使うiOSシミュレーターの14件は前述のMetal初期化エラーで失敗しました。

   [最終のiOS実推論テスト](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38432709791)は、E4Bの不正画像後の復帰を追加して**11件成功・失敗0・スキップ0**でした。最終IPAと同じコミット `4c4a52ae10b46eaaab8f704022076e07535cace3` のコードです。初回はGitHubがmacOS実行環境を確保できず、テスト前に止まったため、コードを変更せず再実行しました。

   [最終IPAビルド](https://github.com/taotaohoyo/pocket-gemma-build/actions/runs/38432594875)も成功し、コア40件・iPhone操作17件・iPad初回起動1件とIPA構造・モデル検証メモリの検査を通っています。ダウンロードしたIPAは1.0.11（12）、19,913,510 bytes、ARM64のiPhone実機用で、CLiteRTLMを含みます。モデル本体・検証アプリ・合成の添付・署名情報を含みません。UIの18件成功はCIログを読み戻して確認しました。大きなUI動画・画像アーカイブはブラウザーがダウンロードをブロックしたため、今回はローカルでの目視確認をしていません。実推論の記録とIPAはダウンロードしてハッシュを照合済みです。

   CPUは遅くなる可能性がありますが、画像自体をモデルへ渡して推論しています。実機GPUの互換性、実機のメモリ上限・速度、利用者の端末での直接の失敗箇所は未確認です。ライブラリ内のSIGSEGVをSwiftの例外処理だけで捕捉することはできません。GPUの再現したシェーダーエラーはライブラリ側で対象端末のリソース上限に対応する必要があります。

更新後は、使用するモデルの「構成」→「モデル構成」→「Accelerator」で**CPU**を選ぶと、今回検証したアプリの経路を使えます。モデルの再ダウンロードや会話削除は不要です。CPUでも実機で失敗する場合は「モデルと設定」→「推論の診断」に残る段階とネイティブエラーを確認して追加調査する必要があります。
