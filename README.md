# Pocket Gemma

iPhone・iPadで対応するGemma・Qwenを端末内で使うSwiftUIチャットアプリです。

## AltStore Classicで更新

Sourcesに以下のURLを一度追加してください。

https://raw.githubusercontent.com/taotaohoyo/pocket-gemma-build/main/altstore-source.json

最新版のunsigned IPAは[Releases](https://github.com/taotaohoyo/pocket-gemma-build/releases)で配布します。
同じApple IDで上書き更新すると、保存済みのモデルと会話を引き継げます。
無料Apple IDの署名期限のRefreshは引き続きAltStoreで行います。
AltStore PAL向けの公証済みアプリではありません。

## 公開ソースと確認

PocketGemma-source.zipは監査したソース、ビルド設定、生成アセットと合成テスト用ファイルのみです。
元の非公開リポジトリの履歴、利用者の会話・写真・動画、モデル本体、署名情報は含みません。
ActionsのBuild iPhone appを手動実行すると、標準macOSランナーでコア・UIテストを実行してunsigned IPAを作成します。
Actions添付の保持は1日ですが、ReleasesのIPAと更新ソースは継続して利用できます。
UIテストは合成の返信を使います。実機AI推論・iOS 27との完全な外観比較を確認したものではありません。
負荷・速さのゲージは端末とモデルサイズに基づく比較の目安です。

1.0.10では画像GPU処理、Word・スキャンPDFの本文、送信後の入力消去とキーボード終了、端末自動選択、モデル別アイコン、Gemma 4 12B（実験対応）を追加しました。iCloud Driveは任意の自動バックアップで、別の端末では履歴を復元します。CloudKitのリアルタイム同期ではありません。無料署名で利用するユーザー選択フォルダ方式です。

1.0.11ではE2B・E4Bの画像推論の構成とエラー後の復帰を修正しました。検証済みの経路は「モデル構成 → Accelerator → CPU」です。モデルや会話の削除・再ダウンロードは不要です。実機は未確認で、GPUの既知の制約があります。[日本語の調査報告](VISION-INVESTIGATION-JA.md)と[詳細な記録](VISION-INVESTIGATION.md)を参照してください。UIテストと別に、実モデルを使うiOSシミュレーターの推論検証を行っています。

最終の実モデル検証はiOS 26.2シミュレーターで11件成功、失敗0、スキップ0です。E2B・E4Bとも不正画像のエラー後に正常画像へ回答しました。iPhone実機の成功とGPUメインの互換性は未確認です。
