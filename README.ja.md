# DXVK — Steam Frame DFR

[English](README.md) | 日本語

Steam Frame内で描画するWindows版VRChat向けに、**Dynamic Foveated Rendering（DFR）**
を組み込んだ実験的なDXVKフォークです。
Vulkanのfragment shading rate attachmentを使い、視線付近の細かさを保ちながら
周辺領域のシェーディング処理を減らします。

DXVKと、ReinaS-64892・poprox24さんらによる視線追従VRSの実装を基にしています。
公開ブランチは`steam-frame-dfr`です。
元コードのrevisionと残る依存は[出典情報](frame/PROVENANCE.md)に記載しています。

## Frame向けの変更

- OpenVR Backgroundのネイティブプロセスで、`GetEyeTrackedFoveationCenter`から左右の視線中心を取得。
- NDCからUVへの変換、任意の上下反転、利用者ごとの位置補正を通じて既存のGAZ2形式へ接続。
- タイル型GPUのsecondary command buffer経路を維持するVRSアップロードとdynamic state設定。
- layered shading-rate attachment非対応時は、対応するsingle-layer経路を利用。
  非対応の両眼layered passは通常のシェーディングを継続。
- 円の境界を求めて連続領域を埋め、領域を再利用するCPUマップ計算。
- マップ互換性・境界・異常入力・視線変換・GAZ2形式を確認する独立テスト。

元実装の非同期worker、GAZ2、VRS資源管理は引き継いでいます。

## 使い方

既にFrame単体でWindows版VRChatをVR表示できる環境を対象にします。
このリポジトリはDFRソースと視線ブリッジを提供します。
FEX・Proton・SteamVR・VRChatの導入は別途必要です。

1. このブランチからWindows x86-64用の`d3d11.dll`と`dxgi.dll`をビルド。
2. 元DLLを復元用に保存し、別途用意した書き換え可能な私有Protonへ配置。
3. Linux ARM64用OpenVR視線ブリッジをビルドして実行。
4. Windows側DXVKへ`DXVK_GAZE_VRS=1`と正しい`DXVK_GAZE_FILE`を渡す。
   両プロセスから同じGAZ2ファイルが見える必要があります。

DLLの配置、FEXの環境変数引き継ぎ、視線位置合わせ、復元は
[ビルドと利用手順](frame/docs/BUILD_AND_USE.ja.md)にまとめています。
ブリッジはOpenVR Backgroundとして動作し、別のscene sessionを作成しません。
視線が無効または古い場合は通常のシェーディングへ戻ります。

### 必要なもの

- 動作済みのSteam Frame単体PCVR環境。
- `VK_KHR_fragment_shading_rate`のattachment機能に対応するVulkanドライバー。
- `GetEyeTrackedFoveationCenter`を提供するOpenVRランタイムと、対応するSDK。
- Windows x86-64 DLLをビルドするためのDXVKの依存関係。
- 視線ブリッジ用のLinux ARM64 C++17コンパイラーとOpenVRライブラリー。
- 独立テスト用のPython 3とC++17コンパイラー。

### 設定

| 変数 | 用途 |
| --- | --- |
| `DXVK_GAZE_VRS` | `1`で有効、`0`で無効 |
| `DXVK_GAZE_FILE` | ブリッジが書くGAZ2ファイルのWine側パス |
| `DXVK_GAZE_DOT` | 対応するSBS passの位置確認用の点。通常`0`、確認時だけ`1` |

領域の調整とブリッジの校正は[利用手順](frame/docs/BUILD_AND_USE.ja.md)を参照してください。
FEXへのゲスト環境変数の渡し方はランチャーに依存し、ホストのSteam起動オプションだけでは
反映されない場合があります。

## ビルド手順

submoduleを含めて公開ブランチを取得します。

```sh
git clone --branch steam-frame-dfr --recurse-submodules https://github.com/ji10me/dxvk-steam-frame-dfr.git
cd dxvk-steam-frame-dfr
```

### DLLのビルド

DXVKの依存関係とMinGW-w64コンパイラーを用意した場合の例です。

```sh
meson setup build-frame --cross-file build-win64.txt --buildtype release \
  -Denable_d3d8=false -Denable_d3d9=false -Denable_d3d10=false
meson compile -C build-frame
```

主な成果物は`build-frame/src/d3d11/d3d11.dll`と`build-frame/src/dxgi/dxgi.dll`です。
これらはWindows x86-64用で、FrameではFEX側で実行されます。
マップ計算をARM64の別サービスへ移す実装ではありません。

コンパイラーなどの条件は[元のDXVKビルド説明](README.upstream.md#build-instructions)、
ネイティブOpenVRブリッジのビルドコマンドは[Frame向け手順](frame/docs/BUILD_AND_USE.ja.md)を参照してください。

### 独立テスト

```sh
python3 -m unittest discover -s frame/tests -p 'test_*.py' -v
```

テストはVRランタイムやゲームへ接続しません。

## 検証状況

元の実機構成ではVRChatへ適用し、本人が描画と視線位置を確認しています。
公開する描画ソースは、動作確認済みanalytic map版とライセンス表示コメント以外が一致します。
公開準備時には新しいチェックアウトでDLLビルドと3つの独立テストに合格しました。
確認範囲と未検証部分は[検証記録](frame/docs/VALIDATION.ja.md)に記載しています。

CPUマップ計算の単体ベンチマークを、ゲーム全体のFPS改善率とは扱いません。
効果は場面・CPU負荷・フレーム制御によって変わります。

## 出典とライセンス

- [DXVK](https://github.com/doitsujin/dxvk)：変換レイヤーと基礎描画実装。
- [ReinaS-64892さんのDFRブランチ](https://github.com/ReinaS-64892/dxvk/tree/dfr)と
  [vrchat-dfr-linux](https://github.com/ReinaS-64892/vrchat-dfr-linux)：関連するDFR実装。
  実際に取り込んだパッチのrevisionは[PROVENANCE](frame/PROVENANCE.md)を参照してください。
- poprox24さんと元実装の貢献者：継承したコードと権利表示。
- ji10me：上記のFrame向け追加・調整。

DXVKとFrame向け追加コードは **zlib/libpng（SPDX: Zlib）** です。
[LICENSE](LICENSE)と[frame/LICENSE](frame/LICENSE)を参照してください。
GAZ2共有ヘッダーは元の **MIT** 表示を保持し、元実装の案内を
[DFR-UPSTREAM.txt](frame/licenses/DFR-UPSTREAM.txt)に収録しています。
別プロジェクトの単体PCVRキットにある個人評価用の制限は、この公開DFRフォークには適用しません。

公開範囲はDFRのソース・テスト・文書・ライセンスです。
実機ログ、認証情報、視線記録、動画デコード修正、独自配布できないランタイムのバイナリーは含めません。
元のDXVK READMEは[README.upstream.md](README.upstream.md)に保持しています。
