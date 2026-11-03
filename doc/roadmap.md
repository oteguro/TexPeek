# TexPeek ロードマップ

## 目的

macOS の Finder でゲーム用テクスチャを確認できるツールを作る。
最初の対象は DDS とし、Finder のサムネイル、Quick Look プレビュー、
単体のビューアーアプリを提供する。

## 基本方針

- 本体アプリは Swift を中心に実装する。
- Finder のサムネイルには `QLThumbnailProvider` を使用する。
- スペースキーで開くプレビューには `QLPreviewProvider` を使用する。
- DDS の解析・デコード・画像変換を共通モジュールにまとめ、アプリと両拡張から利用する。
- BC 圧縮の展開には外部ライブラリを使用する。第一候補は [bcdec](https://github.com/iOrange/bcdec)。
- 初版は CPU でデコードして `CGImage` に変換する。Metal による高速化は必要性を確認してから検討する。
- DDS の既定アプリを TexPeek に変更することを、サムネイルやプレビュー利用の条件にしない。
- 本体アプリの常駐を必要としない。

## 想定構成

| 要素 | 役割 |
| --- | --- |
| TexPeek アプリ | ファイルを開いて表示する最小限のビューアーと、拡張のホスト |
| 共通デコードモジュール | DDS ヘッダー解析、ミップ選択、BC 展開、表示用画像の生成 |
| Thumbnail 拡張 | Finder が要求するサイズのサムネイルを生成 |
| Preview 拡張 | Quick Look 向けの静止画像プレビューを生成 |
| bcdec | C のライブラリとして組み込み、BC ブロックを展開 |

bcdec は DDS コンテナ全体の読み込みを担当しない。
DDS ヘッダーの解析、データ範囲の検証、ブロックの配置、画像の端の処理は共通モジュールが担当する。

## 初版の対応範囲

- 2D テクスチャを表示する。
- 従来の DDS ヘッダーと DX10 拡張ヘッダーを解析する。
- 非圧縮の RGBA/BGRA 8bit、BC1・BC2・BC3・BC4・BC5・BC7 を対象とする。
- 非圧縮形式は対応するビットマスク・DXGI 形式を明示し、未知の形式はエラーにする。
- BC4 はグレースケール、BC5 は R/G チャンネルを表示する方針とする。
- 法線マップの Z 成分復元など、用途を推測する変換は初版では行わない。
- 透明部分には市松模様を表示する。
- sRGB と linear、ストレートアルファと乗算済みアルファの扱いを整理し、表示時の変換を統一する。
- サムネイルでは、適切なミップが存在すれば利用してデコード量を減らす。
- キューブマップ・配列・3D テクスチャは初版では未対応として扱い、通常の 2D と誤認しない。

## 実装ステップ

### 1. プロジェクト基盤

- [ ] 対象 macOS バージョン、対応 CPU、Xcode の要件を決める。
- [ ] 本体アプリと 2 種類の Quick Look 拡張のターゲットを作る。
- [ ] 共通モジュールと C ライブラリの接続方法を決める。
- [ ] bcdec のライセンスを確認し、使用するリビジョンを固定してライセンス表記を同梱する。
- [ ] DDS の既存 UTI と競合の有無を調べ、必要な imported type 宣言を追加する。
- [ ] 両拡張の `QLSupportedContentTypes` に対応 UTI を登録する。
- [ ] 本体は `Viewer` / `LSHandlerRank = Alternate` を基本とし、既定アプリを変更する処理は追加しない。

`Alternate` は開くアプリの候補として登録する設定であり、既定アプリにならないことを保証するものではない。
他の対応アプリがない環境も含め、Launch Services の動作を確認する。

### 2. DDS デコーダー

- [ ] ヘッダー、形式、寸法、ミップ数を取得する。
- [ ] データ長・オフセット・サイズ計算を検証し、整数オーバーフローを防ぐ。
- [ ] 初版対象の非圧縮形式を表示用ピクセルへ変換する。
- [ ] bcdec で対象 BC 形式を展開する。
- [ ] 4 の倍数でない寸法や、4×4 未満の画像を正しく処理する。
- [ ] 色空間とアルファの処理を含めて `CGImage` を生成する。
- [ ] 要求サイズに応じたミップ選択と、処理量・メモリ使用量の上限を設ける。
- [ ] 未対応形式・破損ファイルを識別できるエラーを返す。

### 3. 最小限のビューアーアプリ

- [ ] 「開く」とドラッグ＆ドロップで DDS を読み込む。
- [ ] ウィンドウに合わせた表示、拡大縮小、原寸表示を提供する。
- [ ] 透明部分の市松模様を表示する。
- [ ] 画像サイズ・圧縮形式・ミップ数を表示する。
- [ ] 未対応形式や読み込み失敗の理由を表示する。

### 4. Finder サムネイル

- [ ] `QLThumbnailProvider` から共通モジュールを呼び出す。
- [ ] 要求サイズと表示スケールに合わせ、縦横比を保持した画像を返す。
- [ ] 大きな画像や多数のファイルでの生成時間・メモリ使用量を確認する。
- [ ] DDS の既定アプリが別のアプリでもサムネイルが表示されることを確認する。

### 5. Quick Look プレビュー

- [ ] `QLPreviewProvider` でデータベースのプレビュー拡張を実装する。
- [ ] 共通モジュールで生成した画像を `QLPreviewReply` で返す。
- [ ] Finder でスペースキーを押して表示・切り替え・終了できることを確認する。
- [ ] 本体アプリが終了した状態でも、両拡張が動作することを確認する。

### 6. 検証と配布

- [ ] 対象形式ごとの小さなテスト画像を用意し、出典・ライセンスを記録する。
- [ ] 基準画像と比較し、色・アルファ・チャンネル・向きを確認する。
- [ ] DX10 ヘッダー、ミップ、端数寸法、破損データ、巨大な寸法値を検証する。
- [ ] 対応対象の macOS / CPU でビルドと実機表示を確認する。
- [ ] サンドボックス内のファイルアクセス、拡張の登録・有効化を確認する。
- [ ] 他の DDS 対応アプリや Quick Look 拡張との共存を確認する。
- [ ] 配布経路を決め、必要な署名・公証とインストール手順を整える。

## 初版後の候補

- BC6H と浮動小数点 HDR 画像の表示、トーンマッピング・露出調整。
- キューブマップの展開表示、配列要素・3D スライスの選択。
- RGBA チャンネル切り替え、ミップレベル選択、法線マップ表示。
- KTX / KTX2 など、DDS 以外のゲーム用テクスチャ形式。
- 計測に基づく並列化や Metal による高速化。

Quick Look 内に操作 UI が必要になった場合は、
`NSViewController` と `QLPreviewingController` によるプレビューへの移行を検討する。
共通デコーダーはそのまま利用できる構成にする。

## 参考資料

- [Apple: QLThumbnailProvider](https://developer.apple.com/documentation/quicklookthumbnailing/qlthumbnailprovider)
- [Apple: QLPreviewProvider](https://developer.apple.com/documentation/quicklookui/qlpreviewprovider)
- [Apple: Defining file and data types for your app](https://developer.apple.com/documentation/uniformtypeidentifiers/defining-file-and-data-types-for-your-app)
- [Microsoft: Programming Guide for DDS](https://learn.microsoft.com/en-us/windows/win32/direct3ddds/dx-graphics-dds-pguide)
- [bcdec](https://github.com/iOrange/bcdec)
