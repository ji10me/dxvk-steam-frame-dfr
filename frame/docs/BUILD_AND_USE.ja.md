# ビルドと利用手順

## 1. 前提

既にFrame単体のWindows版VRChatがVR表示できる環境を対象にします。
このソースだけではFEX、Proton、SteamVRやVRChatの導入は完了しません。
GPUドライバーは`VK_KHR_fragment_shading_rate`のattachment機能を、
OpenVRランタイムは`GetEyeTrackedFoveationCenter`を提供する必要があります。

この実装は2つのプロセスで構成されます。

1. Linux ARM64のブリッジがOpenVR Backgroundから左右の視線中心を取得する。
2. Windows x86-64のDXVKがGAZ2ファイルを読み、VRSマップを描画へ適用する。

視線が無効、古い、または対応しない描画passの場合は通常のシェーディングへ戻ります。

## 2. DXVK DLLをビルド

[DXVKのビルド説明](../../README.upstream.md#build-instructions)にある依存関係を用意します。
Linux上のMinGW-w64、Meson、Ninjaなどを使う場合の例です。

```sh
git submodule update --init --recursive
meson setup build-frame --cross-file build-win64.txt --buildtype release \
  -Denable_d3d8=false -Denable_d3d9=false -Denable_d3d10=false
meson compile -C build-frame
```

主な成果物は`build-frame/src/d3d11/d3d11.dll`と`build-frame/src/dxgi/dxgi.dll`です。
LLVM-MinGWを使う場合は、使用するコンパイラーに合わせたMeson cross fileを別に指定してください。
ビルドしたDLLはWindows x86-64用で、FrameではFEX側で実行されます。
CPUマップ計算をARM64の常駐処理へ移す実装ではありません。

## 3. OpenVRブリッジをビルド

ブリッジはFrame上、または互換性のあるLinux ARM64環境でビルドします。
APIを含むValveの[OpenVR SDK](https://github.com/ValveSoftware/openvr)を別途取得します。
公開準備時に確認したSDK revisionは`0924064316de3effbcd1acf1e309182a2deb1c05`です。
SDKと実行時SteamVRのインターフェースが対応する必要があります。

```sh
mkdir -p frame/external frame/bin
git clone https://github.com/ValveSoftware/openvr.git frame/external/openvr
git -C frame/external/openvr checkout 0924064316de3effbcd1acf1e309182a2deb1c05
OPENVR_SDK="$PWD/frame/external/openvr"
OPENVR_LIB_DIR="$OPENVR_SDK/lib/linuxarm64"
c++ -std=c++17 -O2 -Wall -Wextra -Werror \
  -I"$OPENVR_SDK/headers" -Iframe/include frame/tools/frame_gaze_bridge.cpp \
  -L"$OPENVR_LIB_DIR" -Wl,-rpath,"$OPENVR_LIB_DIR" -lopenvr_api -pthread \
  -o frame/bin/frame-gaze-bridge
```

`frame/external`と`frame/bin`はGit対象外です。
Linux x86-64で独立検証する場合は対応するSDKライブラリーディレクトリへ変更します。

## 4. 私有のProton環境へDLLを配置

既存の動作環境とは別に用意した、書き換え可能な私有ProtonのDXVKだけを差し替えます。
元DLLは正確に復元できるよう保存しておきます。
一般的な配置先は`files/lib64/wine/dxvk/`ですが、使用するProton版で実際の場所を確認してください。
実行中のゲームのDLLや、複数ゲームが使う共有Protonへ直接上書きしないでください。
Prefix内へハードリンクされたDLLを上書きすると元ファイルも変わるので、新しいファイルとして置き換えます。

FEX/Protonの起動方法は環境ごとに異なるため、このフォークには自動配置スクリプトを含めていません。
DLLを配置しただけで使われるとは限らず、起動時にその私有Proton/DXVKが選ばれていることを確認します。

## 5. DFRを有効にする

Windows側DXVKへ、次の環境変数を渡します。

| 変数 | 設定 |
| --- | --- |
| `DXVK_GAZE_VRS` | `1`で有効、`0`で無効 |
| `DXVK_GAZE_FILE` | ブリッジが書くファイルのWine側パス |
| `DXVK_GAZE_DOT` | 通常`0`、位置合わせ時だけ`1` |
| `DXVK_GAZE_FULL_AREA` / `DXVK_GAZE_MID_AREA` | 中心・中間領域を調整する場合の値。上流の定義を引き継ぐ |

ブリッジの出力先は実行ユーザーの
`$HOME/.local/share/frame-pcvr/gaze/frame-gaze-v1.bin`です。
Wineの通常の`Z:`マッピングなら、DXVK側のパスは
`Z:\home\YOUR_USER\.local\share\frame-pcvr\gaze\frame-gaze-v1.bin`のようになります。
`YOUR_USER`を自分のユーザー名へ置き換え、両プロセスから同じ実体が見えることを確認します。

FEXではホストのSteam起動オプションだけではWindowsゲストへ届かない場合があります。
その環境のFEX app configの`Env`設定と、Steamコンテナーへの設定引き継ぎも確認してください。
この公開フォークは、個別端末のFEX設定を自動で変更しません。

VRChatを起動し、Linux側で対象プロセスを確認します。

```sh
pgrep -a -x VRChat.exe
```

確認したPIDを指定して、SteamVRと同じユーザーでブリッジを実行します。
親ディレクトリを用意し、ブリッジが専用の非公開ディレクトリとファイルを作成します。

```sh
mkdir -p "$HOME/.local/share/frame-pcvr"
VRCHAT_PID=12345  # 自分の対象プロセスのPIDへ置き換える
frame/bin/frame-gaze-bridge "$VRCHAT_PID" 900 --flip-y
```

900は実行秒数です。1〜1800秒を受け付け、期限・ゲーム終了・scene focus喪失で終了します。
期限で終了した場合は、引き続き使う対象PIDを確認して再実行します。
自動起動や無期限の常駐サービスは作成しません。
ブリッジはゲームPID・開始時刻・Steam App IDを確認し、別プロセスへ追従しません。
別のVR scene/sessionを作成せず、ゲームのメモリーへ接続しません。
終了時には視線を無効化し、直前の視線値を消去します。生の視線をログへ記録しません。

## 6. 視線位置を合わせる

`DXVK_GAZE_DOT=1`で、対応するSBS passに位置確認用の点を表示できます。
上下が逆の場合はブリッジの`--flip-y`を切り替えます。
同時にDXVK側の上下反転を重ねないでください。

必要な場合だけ`--calibration GAIN_X GAIN_Y OFFSET_X OFFSET_Y`を追加します。
未指定ではゲイン1・オフセット0です。各自で位置を確認して設定します。
確認後は`DXVK_GAZE_DOT=0`へ戻します。

## 7. 無効化・復元

ブリッジをCtrl+Cで終了し、ゲスト側の`DXVK_GAZE_VRS=0`でDFRを無効にできます。
元のDXVKへ戻す場合はゲームを終了した後、保存した元DLLを復元して通常構成で起動します。
このリポジトリから端末全体の電力設定、クロック、解像度、アバター制限を変更する処理はありません。

## 8. 独立テスト

```sh
python3 -m unittest discover -s frame/tests -p 'test_*.py' -v
mkdir -p frame/bin
c++ -std=c++17 -O2 -Isrc/dxvk frame/tools/frame_foveation_bench.cpp -o frame/bin/map-bench
frame/bin/map-bench 128
```

ベンチマーク出力はCPUマップ計算の時間です。ゲームFPSやGPU描画時間ではありません。
