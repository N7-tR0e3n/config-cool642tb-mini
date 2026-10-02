# cool642tb-mini DYA Studio 対応手順書

## 概要

本ドキュメントは、cool642tb-mini キーボードを DYA Studio に対応させるための手順をまとめたものです。DYA Studio は ZMK Studio の拡張版で、Web ブラウザから USB 経由でキーボードの設定をリアルタイムに変更できる機能を提供します。

## DYA Studio 対応レベル

本手順書では **Level 2** 対応を行います。Level 2 では以下の拡張機能が利用可能になります:

- **ZMK Studio RPC の拡張**: より大きなバッファサイズで高度な通信に対応
- **Input Processors**: トラックボールの automouse や scroll を柔軟に制御
- **最新の ZMK 機能**: `CONFIG_ZMK_POINTING` など、最新の API を使用
- **カスタムプロトコル対応**: DYA Studio 独自の拡張プロトコルに対応

### Level 2 クイックスタート (要点のみ)

既に ZMK の知識がある方向けの簡潔な手順:

1. **west.yml**: `cormoran/zmk` の `main+dya` ブランチを使用
2. **build.yaml**: ボード名を `xiao_ble//zmk` に変更
3. **_R.conf**: 以下を追加
   - `CONFIG_ZMK_POINTING=y` (ZMK_MOUSE の代わり)
   - `CONFIG_ZMK_STUDIO=y` + `CONFIG_ZMK_STUDIO_LOCKING=n`
   - `CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256`
4. **_R.overlay**: Input Processors を設定
   - `#include <input/processors.dtsi>`
   - `trackball_listener` で `zip_temp_layer` と `zip_xy_to_scroll_mapper` を定義
5. **build.yml**: `uses: cormoran/zmk/.github/workflows/build-user-config.yml@main+dya`
6. ビルド → フラッシュ → https://studio.dya.cormoran.works/ で接続

詳細な手順は以下のセクションを参照してください。

## cool642tb-mini の構成

- **ボード**: Seeeduino XIAO BLE (nRF52840)
- **構成**: 分割キーボード (左右 2 台)
- **右半体**: PMW3610 トラックボール搭載、USB 経由で DYA Studio 接続
- **左半体**: EC11 ロータリーエンコーダー搭載
- **総キー数**: 43 キー (左: 24 キー、右: 19 キー)

## 前提条件

- GitHub アカウント
- cool642tb-mini のハードウェアが組み立て済み
- 基本的な ZMK ファームウェアの知識

---

## Level 2 移行チェックリスト

既存の ZMK Studio 設定から Level 2 に移行する場合、以下の変更が必要です:

| 変更項目 | 従来 (Level 1) | Level 2 | 対象ファイル |
|---------|---------------|---------|------------|
| ボード名 | `seeeduino_xiao_ble` | `xiao_ble//zmk` | `build.yaml` |
| ポインティング API | `CONFIG_ZMK_MOUSE=y` | `CONFIG_ZMK_POINTING=y` | `cool642tb-mini_R.conf` |
| RPC バッファ | (デフォルト) | `CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256` | `cool642tb-mini_R.conf` |
| Input Processors | なし | `<input/processors.dtsi>` インクルード | `cool642tb-mini_R.overlay` |
| Automouse | `CONFIG_PMW3610_AUTOMOUSE_*` | `&zip_temp_layer 4 800` | `cool642tb-mini_R.overlay` |
| Scroll | レイヤー固定 | `&zip_xy_to_scroll_mapper` | `cool642tb-mini_R.overlay` |

### 新規セットアップの場合

本手順書の手順 1 から順に実施してください。Level 2 の設定がすべて含まれています。

---

## 手順 1: West マニフェストの設定

DYA Studio に対応した ZMK フォークを使用するため、`config/west.yml` を以下のように設定します。

### ファイル: `config/west.yml`

```yaml
manifest:
  remotes:
    - name: cormoran
      url-base: https://github.com/cormoran
  projects:
    - name: zmk
      remote: cormoran
      revision: main+dya
      import:
        file: app/west.yml
    - name: zephyr
      remote: cormoran
      revision: v4.1.0+zmk-fixes+nrf-half-duplex-uart
      clone-depth: 1
      import:
        name-blocklist:
          - ci-tools
          - hal_altera
          - hal_cypress
          - hal_infineon
          - hal_microchip
          - hal_nxp
          - hal_openisa
          - hal_xtensa
          - hal_st
          - hal_ti
          - loramac-node
          - mcuboot
          - mcumgr
          - net-tools
          - openthread
          - edtt
          - trusted-firmware-m
  
  self:  
    path: config
```

### 重要なポイント

- **ZMK フォーク**: `cormoran/zmk` の `main+dya` ブランチを使用
- **Zephyr**: DYA 対応版の Zephyr を使用 (`v4.1.0+zmk-fixes+nrf-half-duplex-uart`)
- **理由**: 公式 ZMK では DYA Studio 機能が含まれていないため、カスタムフォークが必要

---

## 手順 2: GitHub Actions ワークフローの設定

DYA 対応のビルドワークフローを使用するため、`.github/workflows/build.yml` を設定します。

### ファイル: `.github/workflows/build.yml`

```yaml
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: cormoran/zmk/.github/workflows/build-user-config.yml@main+dya
```

### 重要なポイント

- **正しい構文**: `{owner}/{repo}/.github/workflows/{filename}@{ref}` の形式に従う
- **誤った例**: ~~`cormoran/zmk/main+dya/.github/workflows/build-user-config.yml@v0.3.0`~~
  - `main+dya` をパスに含めない
- **ref の指定**: `@main+dya` でブランチを指定、または `@v0.3.0` などのタグを指定

---

## 手順 3: ビルドマトリクスの設定 (Level 2)

`build.yaml` で DYA Studio Level 2 対応のボード名と snippet を設定します。

### ファイル: `build.yaml`

```yaml
---
include:
 - board: xiao_ble//zmk              # Level 2: xiao_ble//zmk を使用
   shield: cool642tb-mini_R
   snippet: studio-rpc-usb-uart      # DYA Studio 用 snippet
 - board: xiao_ble//zmk              # Level 2: xiao_ble//zmk を使用
   shield: cool642tb-mini_L
 - board: xiao_ble//zmk              # Level 2: xiao_ble//zmk を使用
   shield: settings_reset
```

### Level 2 での変更点

#### ボード名の変更

| 従来 (Level 1) | Level 2 |
|---------------|---------|
| `seeeduino_xiao_ble` | `xiao_ble//zmk` |

- **`xiao_ble//zmk`**: cormoran/zmk の `app/boards/seeed/xiao_ble/xiao_ble_zmk.dts` で定義される ZMK 専用バリアント
- **最適化内容**: バッテリー電圧測定、フラッシュパーティション、UF2 ブートローダー設定が ZMK 向けに調整済み
- **互換性**: Seeeduino XIAO nRF52840 と同一ハードウェア

#### snippet の設定

- **snippet**: `studio-rpc-usb-uart` を右半体のビルドに追加
  - USB-UART 経由で Studio と通信するための設定
  - Level 2 の拡張 RPC プロトコルに対応
- **左半体**: snippet 不要 (BLE 経由でペアリングされる)
- **デバッグ用**: `# snippet: zmk-usb-logging` をコメントアウトで残しておくと便利

---

## 手順 4: 右半体のファームウェア設定 (Level 2)

DYA Studio Level 2 を有効化するため、右半体の `.conf` ファイルを設定します。

### ファイル: `config/boards/shields/Test/cool642tb-mini_R.conf`

```conf
# キーボード名
CONFIG_ZMK_KEYBOARD_NAME="cool642tb-mini"

# BLE 設定
CONFIG_ZMK_BLE=y
CONFIG_BT_DEVICE_NAME_MAX=30

# === Level 2: ポインティングデバイス設定 ===
CONFIG_ZMK_POINTING=y            # Level 2: CONFIG_ZMK_MOUSE の後継
CONFIG_SPI=y
CONFIG_INPUT=y

# PMW3610 トラックボール
CONFIG_PMW3610=y
CONFIG_PMW3610_CPI=600
CONFIG_PMW3610_ORIENTATION_180=y
CONFIG_PMW3610_SCROLL_TICK=16
CONFIG_PMW3610_INVERT_SCROLL_X=y
CONFIG_PMW3610_POLLING_RATE_125_SW=y
CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=800
CONFIG_PMW3610_SMART_ALGORITHM=y

# エンコーダー (EC11)
CONFIG_EC11=y
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y

# バッテリー報告
CONFIG_ZMK_BATTERY_REPORTING=y

# === DYA Studio Level 2 設定 ===
CONFIG_ZMK_STUDIO=y                      # DYA Studio を有効化
CONFIG_ZMK_STUDIO_LOCKING=n              # Studio ロック機能を無効化

# Level 2: 拡張 RPC バッファ
CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256    # フレームデータ転送バッファ拡張
CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=256    # 受信バッファ拡張 (オプション)
```

### Level 2 での変更点

#### 1. ポインティングデバイス API の更新

| 従来 (Level 1) | Level 2 |
|---------------|---------|
| `CONFIG_ZMK_MOUSE=y` | `CONFIG_ZMK_POINTING=y` |

- **理由**: ZMK の公式 API が `ZMK_MOUSE` から `ZMK_POINTING` に変更されました
- **後方互換性**: `ZMK_MOUSE=y` でも動作しますが、Level 2 では明示的に新 API を使用します
- **利点**: Input Processors と統合された最新の機能が利用可能

#### 2. ZMK Studio RPC バッファの拡張

```conf
CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256    # デフォルト: 128
CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=256    # デフォルト: 128
```

- **用途**: DYA Studio のカスタムプロトコルモジュールとの高度な通信に対応
- **効果**: より大きなデータフレーム (キーマップ、設定データ) の送受信が可能
- **トレードオフ**: RAM 使用量が増加 (nRF52840 では問題なし)

#### 3. cool642tb-mini 固有の設定

- **PMW3610 トラックボール**: 右半体に搭載されているため、関連設定を含める
- **CONFIG_ZMK_STUDIO=y**: DYA Studio 機能を有効化
- **CONFIG_ZMK_STUDIO_LOCKING=n**: 編集ロック機能を無効化 (開発中は推奨)

---

## 手順 5: Input Processors の設定 (Level 2)

Level 2 では、Input Processors を使用してトラックボールの動作を柔軟に制御できます。

### ファイル: `config/boards/shields/Test/cool642tb-mini_R.overlay`

```dts
#include <input/processors.dtsi>  // Level 2: Input Processor のインクルード

&pmw3610 {
    status = "okay";
    sck-pin  = <5>;
    mosi-pin = <4>;
    miso-pin = <4>;
    cs-gpios = <&gpio0 9 GPIO_ACTIVE_LOW>;
    irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
};

/ {
    // Level 2: Input Listener と Processor の定義
    trackball_listener: input_listener_trackball {
        compatible = "zmk,input-listener";
        device = <&pmw3610>;
        
        // automouse: レイヤー 4 に 800ms 自動切替
        input-processors = <&zip_temp_layer 4 800>;
        
        // スクロールモード: レイヤー 5 で XY 移動をスクロールに変換
        scroller {
            layers = <5>;
            input-processors = <&zip_xy_to_scroll_mapper>;
        };
    };
};
```

### Level 2 の Input Processors 機能

#### 1. `zip_temp_layer` (Automouse)

```dts
input-processors = <&zip_temp_layer 4 800>;
```

- **機能**: トラックボール移動を検出すると、自動的にレイヤー 4 (MOUSE) に切り替わる
- **パラメータ**:
  - `4`: 切り替え先のレイヤー番号
  - `800`: タイムアウト (ms)、800ms 操作がないと元のレイヤーに戻る
- **cool642tb-mini での用途**: マウスボタン (MB1/MB2/MB3) を自動的にアクティブ化

#### 2. `zip_xy_to_scroll_mapper` (Scroll Mode)

```dts
scroller {
    layers = <5>;
    input-processors = <&zip_xy_to_scroll_mapper>;
}
```

- **機能**: 特定のレイヤー (5) がアクティブな時、XY 移動をスクロールに変換
- **パラメータ**:
  - `layers = <5>`: レイヤー 5 (SCROLL) でのみ有効
- **cool642tb-mini での用途**: スクロール専用モードの実装

#### 3. その他の利用可能な Input Processors (Level 2)

| Processor | 機能 |
|-----------|------|
| `&zip_scaler` | 入力値のスケーリング (感度調整) |
| `&zip_transform` | 座標変換 (回転、反転) |
| `&zip_code_mapper` | 入力イベントをキーコードにマッピング |

### 従来の実装との違い

| 項目 | 従来 (Level 1) | Level 2 (Input Processors) |
|------|---------------|---------------------------|
| Automouse | ドライバー内蔵 | `zip_temp_layer` で制御 |
| Scroll モード | `CONFIG_PMW3610_*` | `zip_xy_to_scroll_mapper` |
| 柔軟性 | 固定設定 | オーバーレイで自由にカスタマイズ可能 |
| レイヤー連携 | 限定的 | 任意のレイヤーと連携可能 |

---

## 手順 6: 物理レイアウト定義の作成

DYA Studio で正しいキー位置を表示するため、物理レイアウトを JSON で定義します。

### ファイル: `config/cool642tb-mini.json`

```json
{
  "layouts": {
    "default_layout": {
      "name": "default_layout",
      "layout": [
        { "row": 0, "col":  0, "x":      0, "y": 0 },
        { "row": 0, "col":  1, "x":  1.003, "y": 0 },
        { "row": 0, "col":  2, "x":  2.005, "y": 0 },
        ...
      ]
    }
  }
}
```

### cool642tb-mini のレイアウト

- **行数**: 4 行 (row: 0-3)
- **列数**: 左半体 6 列 (col: 0-5) + 右半体 5 列 (col: 6-10、col-offset=6)
- **座標**: 各キーの物理的な位置を `x`, `y` で定義
- **トラックボール**: キーとしては定義しない (入力デバイスとして別扱い)

---

## 手順 7: ハードウェア定義の確認

### 左半体オーバーレイ (`cool642tb-mini_L.overlay`)

```dts
// 列ピン定義 (6列)
col-gpios = <&xiao_d 10 GPIO_ACTIVE_HIGH>,
            <&xiao_d  9 GPIO_ACTIVE_HIGH>,
            <&xiao_d  8 GPIO_ACTIVE_HIGH>,
            <&xiao_d  7 GPIO_ACTIVE_HIGH>,
            <&gpio0  10 GPIO_ACTIVE_HIGH>,
            <&gpio0   9 GPIO_ACTIVE_HIGH>;

// エンコーダー
&encoder {
    status = "okay";
};
```

### 右半体オーバーレイ (`cool642tb-mini_R.overlay`)

```dts
#include <input/processors.dtsi>  // Level 2: Input Processors

// 列ピン定義 (5列、col-offset=6)
col-gpios = <&xiao_d 10 GPIO_ACTIVE_HIGH>,
            <&xiao_d  9 GPIO_ACTIVE_HIGH>,
            <&xiao_d  8 GPIO_ACTIVE_HIGH>,
            <&xiao_d  7 GPIO_ACTIVE_HIGH>,
            <&gpio0  10 GPIO_ACTIVE_HIGH>;

// PMW3610 トラックボール (SPI)
&pmw3610 {
    status = "okay";
    sck-pin  = <5>;    // gpio0 5
    mosi-pin = <4>;    // gpio0 4
    miso-pin = <4>;    // gpio0 4
    cs-gpios = <&gpio0 9 GPIO_ACTIVE_LOW>;
    irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
};

// Level 2: Input Listener (詳細は手順 5 を参照)
/ {
    trackball_listener: input_listener_trackball {
        compatible = "zmk,input-listener";
        device = <&pmw3610>;
        input-processors = <&zip_temp_layer 4 800>;  // Automouse
        
        scroller {
            layers = <5>;
            input-processors = <&zip_xy_to_scroll_mapper>;  // Scroll
        };
    };
};
```

**Level 2 の追加要素**:
- `<input/processors.dtsi>` のインクルード
- `trackball_listener` による Input Processors の定義
- 詳細な設定方法は **手順 5** を参照してください

---

## 手順 8: ファームウェアのビルドとフラッシュ

### 8.1 GitHub Actions でビルド

1. 変更を GitHub リポジトリにプッシュ
2. Actions タブでビルドの完了を確認
3. ビルドアーティファクトから `.uf2` ファイルをダウンロード
   - `cool642tb-mini_L-seeeduino_xiao_ble-zmk.uf2`
   - `cool642tb-mini_R-seeeduino_xiao_ble-zmk.uf2`

### 8.2 ファームウェアの書き込み

1. **右半体から書き込む** (推奨順序)
   - XIAO BLE の RESET ボタンを素早く 2 回押す
   - ブートローダーモードに入る (緑色 LED 点滅)
   - `XIAO-SENSE` ドライブがマウントされる
   - `cool642tb-mini_R-seeeduino_xiao_ble-zmk.uf2` をドラッグ＆ドロップ
   - 自動的に再起動する

2. **左半体を書き込む**
   - 同様の手順で `cool642tb-mini_L-seeeduino_xiao_ble-zmk.uf2` を書き込む

### 8.3 BLE ペアリング

- 両半体の電源を入れると、自動的にペアリングされる
- ペアリングに失敗する場合は、`settings_reset.uf2` を両半体に書き込んでリセット

---

## 手順 9: DYA Studio への接続

### 9.1 接続手順

1. **右半体を USB ケーブルで PC に接続**
2. **ブラウザで DYA Studio にアクセス**
   - URL: https://studio.dya.cormoran.works/
3. **「Connect」ボタンをクリック**
4. **シリアルポート選択ダイアログで「cool642tb-mini」を選択**
5. **接続完了**
   - キーボードのレイアウトが表示される
   - リアルタイムでキーマップを編集可能

### 9.2 cool642tb-mini 固有の注意点

- **USB 接続は右半体のみ**: 右半体に DYA Studio 機能が組み込まれている
- **左半体は BLE 接続**: 左半体は右半体経由で制御される
- **トラックボール**: DYA Studio 上では編集不可 (ファームウェアレベルの設定)
- **エンコーダー**: キーマップで rotation-sensor を編集可能

---

## 手順 10: キーマップの編集

### DYA Studio でできること

- **キーの再割り当て**: ドラッグ＆ドロップでキーを変更
- **レイヤーの編集**: 複数レイヤーの切り替えと編集
- **モディファイアキーの設定**: `&mt` (Mod-Tap) などの動作設定
- **コンボの設定**: 複数キー同時押しの定義
- **エンコーダーの設定**: 回転時の動作を変更

### cool642tb-mini の現在のレイヤー構成

| # | 名前 | 用途 |
|---|------|------|
| 0 | default_layer | QWERTY 配列、エンコーダー: スクロール |
| 1 | FUNCTION | 数字行、記号、音量制御、スクリーンショット |
| 2 | NUM | シフト記号 (`!@#$%` 等) |
| 3 | layer_3 | F1〜F12、矢印キー、Bluetooth プロファイル切替 |
| 4 | MOUSE | マウスボタン (MB1/MB2/MB3) |
| 5 | SCROLL | スクロール専用レイヤー |
| 6-7 | 未使用 | 将来の拡張用 |

### コンボ設定

| キー位置 | 出力 | 用途 |
|---------|------|------|
| 12 + 13 | `LANG2` | 英数切替 (macOS/Windows) |
| 18 + 19 | `LANG1` | かな切替 (macOS/Windows) |
| 38 + 41 | `mo 3` | レイヤー 3 一時有効化 |

---

## トラブルシューティング

### DYA Studio に接続できない

**症状**: ブラウザでシリアルポートが表示されない

**解決策**:
1. Chrome または Edge ブラウザを使用 (Firefox は非対応)
2. 右半体の USB ケーブルを接続し直す
3. ファームウェアが正しくビルドされているか確認:
   ```conf
   CONFIG_ZMK_STUDIO=y
   CONFIG_ZMK_STUDIO_LOCKING=n
   ```
4. `snippet: studio-rpc-usb-uart` が build.yaml に含まれているか確認

### BLE ペアリングが失敗する

**症状**: 左右の半体が接続されない

**解決策**:
1. `settings_reset.uf2` を両半体に書き込む
2. PC の Bluetooth 設定から「cool642tb-mini」を削除
3. 両半体の電源を入れ直す
4. 自動的にペアリングされるまで待つ (30 秒程度)

### トラックボールが動作しない

**症状**: カーソルが動かない、スクロールしない

**解決策**:
1. PMW3610 の SPI ピン設定を確認:
   - CS: gpio0 9
   - IRQ: gpio0 2
   - SCK/MOSI/MISO: gpio0 5/4/4
2. `CONFIG_PMW3610=y` が cool642tb-mini_R.conf に含まれているか確認
3. `CONFIG_ZMK_MOUSE=y` が有効か確認
4. センサーの物理的な接続を確認

### エンコーダーが反応しない

**症状**: 左半体のエンコーダーを回しても何も起こらない

**解決策**:
1. `CONFIG_EC11=y` が cool642tb-mini_R.conf (または L.conf) に含まれているか確認
2. エンコーダーのピン設定を確認:
   - A: xiao_d 0
   - B: xiao_d 5
3. キーマップでセンサーバインディングが設定されているか確認:
   ```dts
   sensor-bindings = <&encoder_msc_down_up>;
   ```

### ビルドが失敗する

**症状**: GitHub Actions でビルドエラーが発生

**解決策**:
1. build.yml の構文を確認:
   ```yaml
   uses: cormoran/zmk/.github/workflows/build-user-config.yml@main+dya
   ```
2. west.yml の revision が正しいか確認:
   ```yaml
   revision: main+dya
   ```
3. **Level 2**: build.yaml のボード名を確認:
   ```yaml
   board: xiao_ble//zmk  # seeeduino_xiao_ble ではない
   ```
4. Actions のログでエラーメッセージを確認
5. シールドファイル (`.overlay`, `.conf`) の構文エラーをチェック

### Input Processors が動作しない (Level 2)

**症状**: Automouse やスクロールモードが機能しない

**解決策**:
1. overlay で `<input/processors.dtsi>` をインクルードしているか確認:
   ```dts
   #include <input/processors.dtsi>
   ```
2. `trackball_listener` が正しく定義されているか確認
3. レイヤー番号が keymap と一致しているか確認:
   - Automouse: レイヤー 4 (MOUSE)
   - Scroll: レイヤー 5 (SCROLL)
4. `CONFIG_ZMK_POINTING=y` が有効か確認 (CONFIG_ZMK_MOUSE ではない)

### バッテリー残量表示が不正確 (Level 2)

**症状**: Level 2 移行後、バッテリー残量の表示が不安定または不正確

**原因**: cormoran/zmk では ZMK 標準のバッテリー計算を使用しており、従来のカスタムキャリブレーションが失われている

**解決策**:
1. 実機でバッテリー残量を確認し、許容範囲内か判断
2. 不正確な場合:
   - zmk-fork-migration.md の「バッテリー計算の変更」セクションを参照
   - na-ka-no フォークのパッチを cormoran フォークに適用
   - または、DTS で電圧しきい値を調整:
     ```dts
     &vbatt {
         full-ohms = <2000000>;
         output-ohms = <820000>;
     };
     ```

---

## まとめ

cool642tb-mini を DYA Studio Level 2 に対応させるための主要なポイント:

### Level 2 固有の設定

1. **ボード名の更新**: `seeeduino_xiao_ble` → `xiao_ble//zmk`
2. **ポインティング API の更新**: `CONFIG_ZMK_MOUSE=y` → `CONFIG_ZMK_POINTING=y`
3. **拡張 RPC バッファ**: `CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256`
4. **Input Processors**: `zip_temp_layer` と `zip_xy_to_scroll_mapper` で高度な制御

### 基本設定

5. **カスタム ZMK フォーク使用**: `cormoran/zmk` の `main+dya` ブランチ
6. **右半体で Studio 有効化**: `CONFIG_ZMK_STUDIO=y` + `snippet: studio-rpc-usb-uart`
7. **物理レイアウト定義**: `cool642tb-mini.json` でキー配置を定義
8. **GitHub Actions 設定**: 正しい構文でワークフローを参照
9. **トラックボール対応**: PMW3610 + Input Processors で柔軟な制御
10. **エンコーダー対応**: EC11 の設定を維持

### Level 2 の利点

- **最新の ZMK 機能**: 公式 ZMK に近いコードベースで、最新機能が利用可能
- **柔軟な Input 制御**: Input Processors により、オーバーレイで動作をカスタマイズ可能
- **拡張プロトコル対応**: DYA Studio の高度な機能 (カスタムプロトコルモジュール) に対応
- **将来性**: ZMK の進化に追従しやすい実装

これらの設定により、Web ブラウザから USB 接続でキーマップをリアルタイム編集できるだけでなく、最新の ZMK 機能を活用した高度なカスタマイズが可能になります。

---

## 参考リンク

### DYA Studio 関連

- **DYA Studio**: https://studio.dya.cormoran.works/
- **DYA Studio 開発者ガイド**: https://studio.dya.cormoran.works/developer-guide
- **DYA Studio Level 2**: https://studio.dya.cormoran.works/developer-guide/level-2

### ZMK 関連

- **ZMK 公式ドキュメント**: https://zmk.dev/
- **cormoran/zmk リポジトリ**: https://github.com/cormoran/zmk
- **PMW3610 ドライバー**: https://github.com/na-ka-no/zmk-pmw3610-driver

### cool642tb-mini 関連ドキュメント

- **ZMK フォーク移行ガイド**: `docs/zmk-fork-migration.md`
  - na-ka-no/zmk → cormoran/zmk への移行時の詳細
  - バッテリー計算、Input Processors、ボード定義の変更点
  - Level 2 で失われる機能と対処方法

---

**作成日**: 2026-10-02  
**対象バージョン**: cormoran/zmk `main+dya` ブランチ (Level 2)  
**cool642tb-mini リビジョン**: dya-studio-2
