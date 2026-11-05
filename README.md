# Steam Frame向け Dynamic Foveated Rendering

Steam Frameでローカル描画するWindows版VRChat向けに調整した、**実験的なDXVKフォーク**です。
視線の付近を細かく、周辺を粗く描くVulkan attachment VRSを使います。
映像を別のPCから転送する構成でなくても、描画側でDFRを適用できます。

公開ブランチは **`steam-frame-dfr`** です。
PCVR環境全体の導入キットではなく、既に単体PCVRが動いている環境で使うDFR部分のソースです。

## Frame向けの追加内容

- FrameのOpenVR `GetEyeTrackedFoveationCenter` から左右の視線中心を取得するBackgroundプロセス。
- NDCからUVへの変換、上下反転、必要に応じたゲイン・中心位置の補正。
- タイル型GPUのsecondary command bufferを維持するVRS処理。
- layered shading-rate attachment非対応時の分岐。非対応の両眼layered passは通常の描画を継続。
- 円の境界を求めて連続した領域を埋めるCPUマップ計算。既存のmap領域を再利用する。
- 視線変換、GAZ2パケット、境界・異常入力・旧計算との互換性を確認する独立テスト。

DXVK、ReinaS-64892/vrchat-dfr-linux、poprox24などの実装を基にしています。
非同期worker、GAZ2、VRS資源管理などの元コードは残っています。
詳細な出典は[PROVENANCE](frame/PROVENANCE.md)、元のDXVKの説明は[README.upstream.md](README.upstream.md)を参照してください。

## ビルド・使い方

[日本語のビルド・導入・使い方](frame/docs/BUILD_AND_USE.ja.md)を参照してください。

```sh
git clone --branch steam-frame-dfr --recurse-submodules https://github.com/ji10me/dxvk-steam-frame-dfr.git
cd dxvk-steam-frame-dfr
python3 -m unittest discover -s frame/tests -p 'test_*.py' -v
```

テストにはPython 3とC++17コンパイラーが必要です。テストはVRランタイムへ接続しません。
DLLのビルドには、さらにDXVKのビルド依存関係が必要です。
OpenVRブリッジはFrameのLinux ARM64側、DXVK DLLはWindows x86-64側で動きます。

## 動作確認の範囲

元の実機構成で、Frame内のVRChatでDFRを適用し、本人が視線追従と表示を確認しています。
この公開版は同じ描画コードを切り出し、出典・ライセンス表示とディレクトリ構成を整えたものです。
マップ計算の単体速度差をゲーム全体のFPS倍率とは扱っていません。
公開チェックアウトでの検証内容と未検証部分は[VALIDATION](frame/docs/VALIDATION.md)に記載しています。

このリポジトリにハードウェア動画デコード修正、端末の個別設定、アカウント情報、
実機ログ、視線記録、ゲームやProtonの実行ファイルは含めていません。

## ライセンス

DXVKとFrame向けの追加コードは **zlib/libpng（SPDX: Zlib）** です。
元の[LICENSE](LICENSE)と、追加部分の[frame/LICENSE](frame/LICENSE)を参照してください。
GAZ2共有ヘッダーは元の **MIT** 表示を保持しています。
元実装のライセンス案内は[DFR-UPSTREAM.txt](frame/licenses/DFR-UPSTREAM.txt)に同梱しています。
この公開DFRフォークに、別プロジェクトの個人評価専用・再配布禁止条件は適用しません。
