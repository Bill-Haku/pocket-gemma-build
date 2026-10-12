# Pocket Gemma

iPhoneで対応するローカルAIモデルを使う、SwiftUIのチャットアプリです。

## AltStore用IPAのビルド

Actionsの **Build iPhone app** を手動実行します。標準のmacOSランナーで
コアテスト・UI操作テストを実行し、unsigned iPhone IPAを作成します。
成功した実行の **PocketGemma-AltStore** 成果物にIPAが含まれます。
署名とインストールはAltStore側で行います。成果物の保存期間は1日です。

## ソース

`PocketGemma-source.zip` にSwiftソース、ビルド設定、アセット、テストが
入っています。ワークフローはZIPを展開してからビルドします。
モデル本体・ユーザーの会話履歴・写真・署名証明書は含みません。
UI操作テストはサンプルの会話で実行し、実際のAI推論とは別に検証します。

必要環境は、対応するmacOSとXcode 26以降、XcodeGenです。
ZIPを展開したMacでは `brew install xcodegen` と
`bash scripts/build-ios.sh` でunsigned IPAを作成できます。

詳細は `docs/PUBLIC-BUILD.md` を参照してください。
