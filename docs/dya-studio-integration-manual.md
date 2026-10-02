# DYA Studio 対応マニュアル（cool642tb-mini 向け）

DYA Studio の Developer Guide（https://studio.dya.cormoran.works/developer-guide）および
DYA2 キーボードの実装を参照して作成した、cool642tb-mini 専用の対応手順書です。

---

## DYA Studio の機能レベル

DYA Studio への対応は 3 段階に分かれています。

| レベル | 内容 | 対応コスト |
|--------|------|-----------|
| **Level 1** | ZMK Studio 互換（キーマップ編集） | 低 |
| **Level 2** | DYA Studio 拡張機能（トラックボール・BLE・診断等） | 中 |
| **Level 3** | カスタム設定 UI（独自ハードウェア設定） | 高 |

cool642tb-mini では **Level 1 ＋ Level 2** を目標とします。

---

## 前提：フォーク切り替え

Level 2 以上の対応には cormoran 氏のフォークへの切り替えが必要です。
詳細は以下のドキュメントを参照してください。

- `docs/zmk-fork-migration.md` — ZMK フォーク移行ガイド
- `docs/pmw3610-driver-migration.md` — PMW3610 ドライバー移行ガイド

---

## Level 1：ZMK Studio 基本対応

ZMK Studio の**キーマップ編集機能**のみを使えるようにする最小構成です。
現在の cool642tb-mini は既に ZMK Studio に対応しています。

### 必要な設定（確認用）

**`cool642tb-mini_R.conf`**
```kconfig
CONFIG_ZMK_STUDIO=y          # ZMK Studio 有効化
CONFIG_ZMK_STUDIO_LOCKING=n  # ロック機能を無効化（常時編集可能）
```

**`build.yaml`**
```yaml
- board: seeeduino_xiao_ble
  shield: cool642tb-mini_R
  snippet: studio-rpc-usb-uart  # USB 経由での Studio 接続
```

**`cool642tb-mini.dtsi`**
```c
chosen {
    zmk,physical-layout = &roba_physical_layout;  // 物理レイアウト指定
};
```

### 接続方法

1. 右半体を USB ケーブルで PC に接続
2. Chrome または Edge で https://zmk.studio を開く
3. 「Connect」→ USB シリアルポートを選択

---

## Level 2：DYA Studio 拡張機能

DYA Studio（https://studio.dya.cormoran.works）の拡張機能を使えるようにします。

### 2-1. フォークと依存関係の更新

**`config/west.yml`** を以下に書き換えます。

```yaml
manifest:
  version: 1.2
  defaults:
    remote: cormoran
    revision: main
  remotes:
    - name: cormoran
      url-base: https://github.com/cormoran
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    # west コマンド拡張（west zmk-build を提供）
    - name: zmk-west-commands
      import: true

    # ZMK 本体（cormoran フォーク）
    - name: zmk
      revision: main+dya
      clone-depth: 1
      import:
        file: app/west.yml

    # Zephyr をバージョン固定
    - name: zephyr
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

    # PMW3610 ドライバー（DYA Studio 対応版）
    - name: zmk-driver-pmw3610-with-custom-studio-rpc

    # --- 以下、有効化したい機能に応じて追加 ---

    # BLE 接続管理
    - name: zmk-module-ble-management

    # ランタイムコンボ編集
    - name: zmk-feature-runtime-combo

    # ランタイムマクロ編集
    - name: zmk-feature-runtime-macro

    # 電源・スリープ設定
    - name: zmk-module-settings-rpc

    # ファームウェア診断
    - name: zmk-feature-watchdog

    # デバイス情報表示
    - name: zmk-feature-device-info

    # キーチャター診断
    - name: zmk-feature-kscan-diagnostics

    # トラックボールのランタイム入力設定
    - name: zmk-module-runtime-input-processor

  self:
    path: config
```

### 2-2. ビルドターゲットの更新

**`build.yaml`** のボード名を変更します。

```yaml
include:
  - board: xiao_ble//zmk            # seeeduino_xiao_ble から変更
    shield: cool642tb-mini_R
    snippet: studio-rpc-usb-uart
  - board: xiao_ble//zmk
    shield: cool642tb-mini_L
  - artifact: cool642tb-mini_R-settings_reset
    board: xiao_ble//zmk
    shield: cool642tb-mini_R
    cmake-args: -DCONFIG_ZMK_SETTINGS_RESET_ON_START=y
  - artifact: cool642tb-mini_L-settings_reset
    board: xiao_ble//zmk
    shield: cool642tb-mini_L
    cmake-args: -DCONFIG_ZMK_SETTINGS_RESET_ON_START=y
```

### 2-3. GitHub Actions ワークフローの更新

**`.github/workflows/build.yml`** を DYA2 方式に変更します。

```yaml
name: Build ZMK firmware

on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: zmkfirmware/zmk-build-arm:stable
    name: Build
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Cache west modules
        uses: actions/cache@v4
        continue-on-error: true
        env:
          cache_name: cache-west-modules
        with:
          path: |
            zmk/
            zephyr/
            modules/
            zmk-west-commands/
            zmk-driver-pmw3610-with-custom-studio-rpc/
          key: ${{ runner.os }}-build-${{ env.cache_name }}-${{ hashFiles('config/west.yml') }}
          restore-keys: |
            ${{ runner.os }}-build-${{ env.cache_name }}-

      - name: Init
        run: |
          git config --global --add safe.directory '*'
          west init -l config
          west update --narrow
          west zephyr-export

      - name: Build
        run: |
          ZMK="${PWD}/zmk/app"
          CFG="${PWD}/config"
          MOD="${PWD}/zmk-driver-pmw3610-with-custom-studio-rpc"
          west build -s "$ZMK" -d build/cool642tb-mini_R -b xiao_ble//zmk -S studio-rpc-usb-uart -p always -- \
            -DSHIELD=cool642tb-mini_R -DZMK_CONFIG="$CFG" "-DZMK_EXTRA_MODULES=$MOD"
          west build -s "$ZMK" -d build/cool642tb-mini_L -b xiao_ble//zmk -p always -- \
            -DSHIELD=cool642tb-mini_L -DZMK_CONFIG="$CFG" "-DZMK_EXTRA_MODULES=$MOD"
          west build -s "$ZMK" -d build/cool642tb-mini_R-settings_reset -b xiao_ble//zmk -p always -- \
            -DSHIELD=cool642tb-mini_R -DZMK_CONFIG="$CFG" "-DZMK_EXTRA_MODULES=$MOD" -DCONFIG_ZMK_SETTINGS_RESET_ON_START=y
          west build -s "$ZMK" -d build/cool642tb-mini_L-settings_reset -b xiao_ble//zmk -p always -- \
            -DSHIELD=cool642tb-mini_L -DZMK_CONFIG="$CFG" "-DZMK_EXTRA_MODULES=$MOD" -DCONFIG_ZMK_SETTINGS_RESET_ON_START=y

      - name: Copy artifacts
        shell: sh -x {0}
        run: |
          mkdir -p artifacts
          find ./build -type f -path '*/zephyr/zmk.uf2' \
          | while read -r f; do
              name=$(basename "$(dirname "$(dirname "$f")")")
              mv "$f" "artifacts/${name}.uf2"
            done

      - name: Archive artifacts
        uses: actions/upload-artifact@v4
        with:
          name: cool642tb-mini
          path: artifacts
```

### 2-4. 右半体ハードウェア設定（overlay）

**`cool642tb-mini_R.overlay`** を書き換えます。
PMW3610 ドライバーを cormoran 版に移行し、automouse・scroll を input processor で実装します。

```c
#include "cool642tb-mini.dtsi"
#include <zephyr/dt-bindings/input/input-event-codes.h>
#include <input/processors.dtsi>

&default_transform {
    col-offset = <6>;
};

&kscan0 {
    col-gpios
        = <&xiao_d 10 GPIO_ACTIVE_HIGH>
        , <&xiao_d 9  GPIO_ACTIVE_HIGH>
        , <&xiao_d 8  GPIO_ACTIVE_HIGH>
        , <&xiao_d 7  GPIO_ACTIVE_HIGH>
        , <&gpio0  10 GPIO_ACTIVE_HIGH>
        ;
};

/ {
    trackball_listener: trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball_r>;
        // トラックボール動作時に MOUSE レイヤー（4）を 800ms 間有効化
        input-processors = <&zip_temp_layer 4 800>;

        // SCROLL レイヤー（5）がアクティブなとき XY をスクロールに変換
        scroller {
            layers = <5>;
            input-processors = <&zip_xy_to_scroll_mapper>;
        };
    };
};

&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK,  0, 5)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 4)>,
                    <NRF_PSEL(SPIM_MISO, 0, 4)>;
        };
    };
    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK,  0, 5)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 4)>,
                    <NRF_PSEL(SPIM_MISO, 0, 4)>;
            low-power-enable;
        };
    };
};

&xiao_serial { status = "disabled"; };

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 9 GPIO_ACTIVE_LOW>;

    trackball_r: trackball@0 {
        status = "okay";
        compatible = "cormoran,pmw3610";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
        cpi = <600>;
        evt-type = <INPUT_EV_REL>;
        x-input-code = <INPUT_REL_X>;
        y-input-code = <INPUT_REL_Y>;
        settings-id = "trackball";
    };
};
```

### 2-5. 右半体 Kconfig 設定（conf）

**`cool642tb-mini_R.conf`** を書き換えます。

```kconfig
CONFIG_ZMK_KEYBOARD_NAME="cool642tb-mini"
CONFIG_ZMK_POINTING=y               # ZMK_MOUSE から変更
CONFIG_NFCT_PINS_AS_GPIOS=y
CONFIG_ZMK_BLE=y
CONFIG_BT_DEVICE_NAME_MAX=30
CONFIG_SPI=y
CONFIG_INPUT=y

# PMW3610 センサー基本設定
CONFIG_PMW3610=y
CONFIG_PMW3610_SMART_ALGORITHM=y
CONFIG_PMW3610_INVERT_Y=y           # ORIENTATION_180 + INVERT_X=n の等価設定
CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264
CONFIG_PMW3610_REST1_SAMPLE_TIME_MS=20
CONFIG_PMW3610_REPORT_INTERVAL_MIN=8  # 125Hz 相当

# PMW3610 DYA Studio 対応
CONFIG_ZMK_PMW3610_STUDIO_RPC=y
CONFIG_ZMK_PMW3610_CUSTOM_SETTINGS=y
CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256

# エンコーダー（dtsi 定義との整合のため）
CONFIG_EC11=y
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y

# バッテリー
CONFIG_ZMK_BATTERY_REPORTING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=n
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=n

# ZMK Studio
CONFIG_ZMK_STUDIO=y
CONFIG_ZMK_STUDIO_LOCKING=n

# ── 以下、有効化したい機能に応じて追加 ──────────────

# BLE 接続管理（ブラウザから接続先を管理）
# CONFIG_ZMK_BLE_MANAGEMENT=y
# CONFIG_ZMK_BLE_MANAGEMENT_STUDIO_RPC=y

# ランタイムコンボ編集
# CONFIG_ZMK_RUNTIME_COMBO=y
# CONFIG_ZMK_RUNTIME_COMBO_STUDIO_RPC=y

# ランタイムマクロ編集
# CONFIG_ZMK_RUNTIME_MACRO=y
# CONFIG_ZMK_RUNTIME_MACRO_STUDIO_RPC=y
# CONFIG_ZMK_BEHAVIOR_LOCAL_ID_TYPE_CRC16=y

# 電源・スリープ設定
# CONFIG_ZMK_SETTINGS_RPC=y
# CONFIG_ZMK_SETTINGS_RPC_STUDIO=y

# ファームウェア診断（クラッシュ記録）
# CONFIG_ZMK_WATCHDOG=y
# CONFIG_ZMK_WATCHDOG_STUDIO_RPC=y

# デバイス情報（ビルド情報・起動時間等）
# CONFIG_ZMK_DEVICE_INFO=y
# CONFIG_ZMK_DEVICE_INFO_STUDIO_RPC=y

# キーチャター診断
# CONFIG_ZMK_KSCAN_DIAGNOSTICS=y
# CONFIG_ZMK_KSCAN_DIAGNOSTICS_STUDIO_RPC=y

# トラックボールのランタイム入力設定
# CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR=y
# CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC=y

# スタック拡張（ランタイムマクロ・コンボを使う場合）
# CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048
# CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=256
```

---

## 各モジュールの機能と有効化手順

### BLE 接続管理（`zmk-module-ble-management`）

DYA Studio から Bluetooth 接続先の管理ができます。

- ペアリング済みデバイスの一覧・切り替え
- プロファイルへのカスタム名付与
- 不要なペアリングの削除

```kconfig
CONFIG_ZMK_BLE_MANAGEMENT=y
CONFIG_ZMK_BLE_MANAGEMENT_STUDIO_RPC=y
```

---

### ランタイムコンボ編集（`zmk-feature-runtime-combo`）

ファームウェアの再ビルドなしにコンボを追加・編集できます。

```kconfig
CONFIG_ZMK_RUNTIME_COMBO=y
CONFIG_ZMK_RUNTIME_COMBO_STUDIO_RPC=y
CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=128
CONFIG_ZMK_STUDIO_RPC_CUSTOM_SUBSYSTEM_REQUEST_PAYLOAD_MAX_BYTES=96
CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048
```

---

### ランタイムマクロ編集（`zmk-feature-runtime-macro`）

ファームウェアの再ビルドなしにマクロを作成・編集できます。

```kconfig
CONFIG_ZMK_RUNTIME_MACRO=y
CONFIG_ZMK_RUNTIME_MACRO_STUDIO_RPC=y
CONFIG_ZMK_BEHAVIOR_LOCAL_ID_TYPE_CRC16=y
CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=192
CONFIG_ZMK_STUDIO_RPC_CUSTOM_SUBSYSTEM_REQUEST_PAYLOAD_MAX_BYTES=192
CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048
CONFIG_ZMK_CUSTOM_SETTINGS_LARGE_VALUE_MAX_SIZE=256
```

---

### 電源・スリープ設定（`zmk-module-settings-rpc`）

DYA Studio からアイドルタイムアウト・スリープタイムアウトを変更できます。

```kconfig
CONFIG_ZMK_SETTINGS_RPC=y
CONFIG_ZMK_SETTINGS_RPC_STUDIO=y
```

---

### ファームウェア診断（`zmk-feature-watchdog`）

スレッドフリーズやハードフォルトを検出・記録します。DYA Studio でインシデントログを確認できます。

```kconfig
CONFIG_ZMK_WATCHDOG=y
CONFIG_ZMK_WATCHDOG_STUDIO_RPC=y
```

---

### デバイス情報（`zmk-feature-device-info`）

ビルド情報・MCU ID・起動時間・ZMK 設定状態を DYA Studio で確認できます。

```kconfig
CONFIG_ZMK_DEVICE_INFO=y
CONFIG_ZMK_DEVICE_INFO_STUDIO_RPC=y
```

---

### キーチャター診断（`zmk-feature-kscan-diagnostics`）

キーの発火頻度・チャターを DYA Studio でリアルタイムに可視化できます。

```kconfig
CONFIG_ZMK_KSCAN_DIAGNOSTICS=y
CONFIG_ZMK_KSCAN_DIAGNOSTICS_STUDIO_RPC=y
```

---

### トラックボールのランタイム入力設定（`zmk-module-runtime-input-processor`）

ポインタ感度・軸反転・回転などをリビルドなしで DYA Studio から調整できます。

```kconfig
CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR=y
CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC=y
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048
```

---

### トラックボール詳細設定（`zmk-driver-pmw3610-with-custom-studio-rpc`）

DYA Studio からトラックボールの CPI・軸反転・省電力タイミングをリアルタイム変更できます。センサーフレームのキャプチャ・診断も利用可能です。

```kconfig
CONFIG_ZMK_PMW3610_STUDIO_RPC=y
CONFIG_ZMK_PMW3610_CUSTOM_SETTINGS=y
CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256
```

---

## 接続方法

1. 右半体を USB ケーブルで PC に接続
2. Chrome または Edge で https://studio.dya.cormoran.works を開く
3. 「Connect」→「USB Serial」→ シリアルポートを選択
4. 接続後、各機能のパネルが表示される

> **注意**: DYA Studio は Chrome/Edge のみ対応です。Firefox・Safari は非対応です。

---

## 変更ファイル一覧

| ファイル | Level 1 | Level 2（最小） | Level 2（全機能） |
|---------|:-------:|:--------------:|:---------------:|
| `config/west.yml` | — | 必須 | 必須 |
| `build.yaml` | — | 必須 | 必須 |
| `.github/workflows/build.yml` | — | 必須 | 必須 |
| `cool642tb-mini_R.overlay` | — | 必須 | 必須 |
| `cool642tb-mini_R.conf` | 既存確認 | 必須 | 必須 |
| `cool642tb-mini_L.conf` | — | — | — |
| `cool642tb-mini.dtsi` | — | — | — |
| `cool642tb-mini.keymap` | — | — | — |

---

## 注意事項

### バッテリー残量の精度

cormoran の ZMK フォークでは na-ka-no フォークで行われたバッテリー電圧キャリブレーションが適用されていません。移行後にバッテリー残量表示がずれる可能性があります（詳細は `docs/zmk-fork-migration.md` を参照）。

### スクロール方向

PMW3610 ドライバー移行後、スクロール方向が変わる場合があります。
`cool642tb-mini_R.overlay` の `scroller` ノードに `zip_scroll_scaler` を追加して調整してください。

```c
scroller {
    layers = <5>;
    input-processors = <&zip_xy_to_scroll_mapper>,
                       <&zip_scroll_scaler 1 16>;  // スクロール感度を調整
};
```

### センサー向き

`CONFIG_PMW3610_ORIENTATION_180=y` は cormoran ドライバーに存在しません。
代わりに `CONFIG_PMW3610_INVERT_Y=y` を使用してください。向きが合わない場合は `CONFIG_PMW3610_INVERT_X=y` や `CONFIG_PMW3610_SWAP_XY=y` を組み合わせて調整してください。
