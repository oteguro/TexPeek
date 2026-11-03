# テスト用テクスチャ

## flower-rgba8

実写風のピンクのコスモス画像。TexPeek の非圧縮 DDS 表示を確認するための初期サンプル。

| ファイル | 用途 |
| --- | --- |
| `flower-rgba8.dds` | Finder / Quick Look / 本体ビューアーで表示する対象 |
| `flower-rgba8.png` | DDS と同じ画素を保存した表示比較用画像 |

- 寸法: 256 × 256 ピクセル
- 形式: 非圧縮 RGBA8、各チャンネル 8bit
- バイト順: R, G, B, A（行は上から順）
- ヘッダー: 従来形式の DDS（DX10 拡張なし）
- ピクセル形式: `DDPF_RGB | DDPF_ALPHAPIXELS`、32bit
- マスク: R=`0x000000FF`, G=`0x0000FF00`, B=`0x00FF0000`, A=`0xFF000000`
- 行ピッチ: 1,024 バイト
- ミップマップ: なし（ベース画像のみ）
- アルファ: 全画素 255（不透明）。透明度の検証用サンプルではない
- DDS サイズ: 262,272 バイト（128 バイトのヘッダーと 262,144 バイトの画素データ）
- 色の扱い: sRGB を想定した表示用画像。従来形式のこの DDS ヘッダーには sRGB の明示情報はない

2026-11-03 に組み込み image_gen で生成した画像を、Lanczos フィルターで 256 × 256 に縮小し、
RGBA8 の DDS と PNG に変換した。Pillow で DDS を再読み込みし、PNG の画素との完全一致を確認済み。

## BC 圧縮サンプル

すべて `flower-rgba8.png` を入力とした 256 × 256、ミップなしの DDS。
BC 圧縮は非可逆のため、元画像と画素は完全一致しない。

| ファイル | 格納形式 | 内容 | ファイルサイズ |
| --- | --- | --- | --- |
| `flower-bc1.dds` | BC1 / DXT1 | RGB、不透明 | 32,896 バイト |
| `flower-bc2.dds` | BC2 / DXT3 | RGB + 明示的 4bit アルファ | 65,664 バイト |
| `flower-bc3.dds` | BC3 / DXT5 | RGB + 補間アルファ | 65,664 バイト |
| `flower-bc4.dds` | BC4_UNORM / DXGI 80 | グレースケール 1 チャンネル | 32,916 バイト |
| `flower-bc5.dds` | BC5_UNORM / DXGI 83 | 元画像の R/G のみ | 65,684 バイト |
| `flower-bc6h.dds` | BC6H_UF16 / DXGI 95 | linear RGB、符号なし浮動小数点 | 65,684 バイト |
| `flower-bc7.dds` | BC7_UNORM_SRGB / DXGI 99 | sRGB の RGB + アルファ | 65,684 バイト |

- BC1〜BC3 は従来の FourCC ヘッダー、BC4〜BC7 は DX10 拡張ヘッダーを使用。
- BC4 の入力は Pillow の `L` 変換による輝度（約 0.299R + 0.587G + 0.114B）。
  格納されるのは 1 チャンネルであり、白黒表示には R を RGB に複製する必要がある。
- BC5 は R/G をそのまま格納し、比較画像では B=0 として表示。法線の Z 成分復元は行っていない。
- BC6H は入力の sRGB を linear RGB に変換し、RGBA FP16 の入力から RGB を圧縮。
  元画像が SDR のため値は 0〜1 の範囲で、HDR の明るさや負値を検証するサンプルではない。
  比較画像では linear RGB を sRGB に戻している。BC6H にアルファはない。
- 入力のアルファはすべて 255。BC1〜BC3 の展開結果も 255。
  BC7 の展開結果には非可逆圧縮による 254〜255 の量子化誤差がある。
- BC1〜BC3 のヘッダーには sRGB の明示情報はない。表示時は元画像と同じ sRGB として扱う。

### 生成と検証

2026-11-03 に [ISPCTextureCompressor の Python バインディング](https://github.com/K0lb3/ispc_texcomp_py)
`ispc-texcomp 1.0.1` を使用して生成。
BC6H は `slow`、BC7 は `alpha_slow` プロファイルを使用した。
BC4 は R8、BC5 は RG8 のパック済み入力を渡している。
BC2 は、不透明の BC1 カラーブロックに全ビット 1 の明示的アルファブロックを付けて生成。
カラーブロックが BC2 と同じ 4 色モードで解釈できる端点順であることを確認した。

全 7 形式を独立した [bcdec](https://github.com/iOrange/bcdec)（ヘッダーのバージョン表記 0.985）で展開し、
元の各チャンネルと比較した。DDS のヘッダー、寸法、形式コード、データ長も確認済み。
圧縮・検証ツールは一時環境で実行しており、アプリの依存関係には追加していない。

`flower-bc1-preview.png` 〜 `flower-bc7-preview.png`（BC6 は `flower-bc6h-preview.png`）は
各 DDS の圧縮データを展開した比較用画像。
`flower-bc-comparison.png` は元画像と全 7 形式の一覧。
これらの確認は DDS の内容の検証であり、Finder の標準対応や TexPeek の実装完了を示すものではない。

## 元画像の生成プロンプト

> Use case: photorealistic-natural. Asset type: square flower image for a 256x256 DDS texture preview test. Primary request: a realistic close-up photograph of a pink cosmos flower with a detailed golden yellow center, natural petals and green foliage softly blurred behind it. Composition: one complete flower prominently centered, square frame, readable at 256 pixels. Lighting: soft natural daylight, realistic colors and fine petal texture. Constraints: photograph-like, no text, no watermark, no border, opaque background.
