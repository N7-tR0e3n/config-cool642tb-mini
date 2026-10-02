# PMW3610 ドライバー移行ガイド

na-ka-no/zmk-pmw3610-driver → cormoran/zmk-driver-pmw3610-with-custom-studio-rpc

---

## ドライバー概要比較

| 項目 | na-ka-no（現在） | cormoran（移行先） |
|------|----------------|-----------------|
| フォーク元 | kumamuk-git/zmk-pmw3610-driver | 独自実装 |
| compatible 文字列 | `pixart,pmw3610` | `cormoran,pmw3610` |
| automouse 制御 | DTS プロパティ（ドライバー内部） | ZMK input processor |
| scroll 制御 | DTS プロパティ（ドライバー内部） | ZMK input processor |
| DYA Studio 対応 | なし | センサー診断・ランタイム設定変更 |
| CPI 指定方法 | Kconfig | DTS プロパティ |
| センサー向き指定 | `ORIENTATION_0/90/180/270` | `INVERT_X` / `INVERT_Y` / `SWAP_XY` |

---

## 1. `cool642tb-mini_R.overlay` の変更

### インクルードの追加

```c
// 追加が必要
#include <zephyr/dt-bindings/input/input-event-codes.h>
#include <input/processors.dtsi>
```

### `trackball` ノードの変更

```c
// 変更前（na-ka-no）
trackball: trackball@0 {
    status = "okay";
    compatible = "pixart,pmw3610";
    reg = <0>;
    spi-max-frequency = <2000000>;
    irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
    automouse-layer = <4>;
    scroll-layers = <5>;
};

// 変更後（cormoran）
trackball_r: trackball@0 {
    status = "okay";
    compatible = "cormoran,pmw3610";
    reg = <0>;
    spi-max-frequency = <2000000>;
    irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
    cpi = <600>;
    evt-type = <INPUT_EV_REL>;       // 必須プロパティ
    x-input-code = <INPUT_REL_X>;   // 必須プロパティ
    y-input-code = <INPUT_REL_Y>;   // 必須プロパティ
    settings-id = "trackball";      // DYA Studio での識別名（推奨）
};
```

### `trackball_listener` ノードの変更

automouse と scroll の制御がドライバーから input processor へ移動します。

```c
// 変更前（na-ka-no）
/ {
    trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;
    };
};

// 変更後（cormoran）
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
```

---

## 2. `cool642tb-mini_R.conf` の変更

### 削除する設定

| 設定 | 削除理由 |
|------|---------|
| `CONFIG_ZMK_MOUSE=y` | 名称変更（後継あり） |
| `CONFIG_PMW3610_CPI=600` | DTS プロパティ `cpi = <600>` に移動 |
| `CONFIG_PMW3610_CPI_DIVIDOR=1` | cormoran ドライバーに存在しない |
| `CONFIG_PMW3610_ORIENTATION_180=y` | 存在しない（個別フラグで代替） |
| `CONFIG_PMW3610_INVERT_X=n` | ORIENTATION_180 との組み合わせで使用していたため整理 |
| `CONFIG_PMW3610_INVERT_SCROLL_X=y` | 存在しない（scroll は input processor 側で制御） |
| `CONFIG_PMW3610_SCROLL_TICK=16` | 存在しない（input processor 側で制御） |
| `CONFIG_PMW3610_POLLING_RATE_125_SW=y` | 名称変更（後継あり） |
| `CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=800` | 存在しない（overlay の `zip_temp_layer 4 800` に移行） |
| `CONFIG_PMW3610_MOVEMENT_THRESHOLD=2` | 存在しない |

### 変更・追加する設定

```kconfig
# ZMK_MOUSE の後継
CONFIG_ZMK_POINTING=y

# ORIENTATION_180=y + INVERT_X=n の等価設定（Y のみ反転）
CONFIG_PMW3610_INVERT_Y=y

# POLLING_RATE_125_SW の後継（8ms = 125Hz 相当）
CONFIG_PMW3610_REPORT_INTERVAL_MIN=8

# DYA Studio 対応（新規追加）
CONFIG_ZMK_PMW3610_STUDIO_RPC=y
CONFIG_ZMK_PMW3610_CUSTOM_SETTINGS=y
CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256
```

### 引き継げる設定（変更不要）

```kconfig
CONFIG_PMW3610=y
CONFIG_PMW3610_SMART_ALGORITHM=y
CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264
CONFIG_PMW3610_REST1_SAMPLE_TIME_MS=20
```

---

## 3. センサー向きの変換

現在の設定 `ORIENTATION_180=y` ＋ `INVERT_X=n` は「Y 軸のみ反転」を意味します。
cormoran ドライバーでは `ORIENTATION_*` がないため、個別フラグで再現します。

| 元の設定 | 実際の動作 | cormoran での等価設定 |
|---------|----------|-------------------|
| `ORIENTATION_180=y` ＋ `INVERT_X=n` | Y のみ反転 | `PMW3610_INVERT_Y=y`（X はデフォルトで非反転） |

> **注意**: 実際にビルドして動作確認後、向きが合わない場合は `PMW3610_INVERT_X`・`PMW3610_SWAP_XY` を組み合わせて調整してください。

---

## 4. スクロール感度の調整

na-ka-no ドライバーの `CONFIG_PMW3610_SCROLL_TICK=16`（16 カウントで 1 スクロールステップ）は input processor で再現できます。

感度が合わない場合は `scroller` ノードに `zip_scroll_scaler` を追加してください。

```c
scroller {
    layers = <5>;
    input-processors = <&zip_xy_to_scroll_mapper>,
                       <&zip_scroll_scaler 1 16>;  // 1/16 に減速
};
```

---

## 5. `config/west.yml` の変更

```yaml
# 変更前
projects:
  - name: zmk
    remote: na-ka-no
    revision: for-cool642tb_mini
    import: app/west.yml
  - name: zmk-pmw3610-driver
    remote: na-ka-no
    revision: main

# 変更後
projects:
  - name: zmk-west-commands
    import: true
  - name: zmk
    revision: main+dya
    import:
      file: app/west.yml
  - name: zephyr
    revision: v4.1.0+zmk-fixes+nrf-half-duplex-uart
    clone-depth: 1
    import:
      name-blocklist: [ci-tools, hal_altera, hal_cypress, hal_infineon,
                       hal_microchip, hal_nxp, hal_openisa, hal_xtensa,
                       hal_st, hal_ti, loramac-node, mcuboot, mcumgr,
                       net-tools, openthread, edtt, trusted-firmware-m]
  - name: zmk-driver-pmw3610-with-custom-studio-rpc
```

> `remotes` の `na-ka-no` を `cormoran` に変更し、`defaults.remote: cormoran` を設定してください。

---

## 6. `build.yaml` のボード名変更

```yaml
# 変更前
board: seeeduino_xiao_ble

# 変更後
board: xiao_ble//zmk
```

---

## 変更一覧サマリー

| ファイル | 変更内容 |
|---------|---------|
| `_R.overlay` | compatible 変更、DTS プロパティ追加・削除、input processor 追加 |
| `_R.conf` | 10 項目削除、5 項目追加・変更 |
| `config/west.yml` | フォーク元変更、zephyr ピン留め追加 |
| `build.yaml` | ボード名変更 |
| `cool642tb-mini.dtsi` | 変更不要 |
| `cool642tb-mini_L.*` | 変更不要 |
| `cool642tb-mini.keymap` | 変更不要（`CONFIG_ZMK_STUDIO_LOCKING=n` で Studio 常時解錠のため `&studio_unlock` 不要） |
