# ZMK フォーク移行ガイド

na-ka-no/zmk@for-cool642tb_mini → cormoran/zmk@main+dya

---

## フォーク概要比較

| 項目 | na-ka-no | cormoran |
|------|---------|---------|
| ベース | zmkfirmware/zmk | zmkfirmware/zmk |
| upstream との差 | 173 commits 遅延・29 ahead | 56 commits 遅延・62 ahead |
| ボード定義場所 | `app/boards/arm/seeeduino_xiao_ble/` | `app/boards/seeed/xiao_ble/` |
| ボード名 | `seeeduino_xiao_ble` | `xiao_ble//zmk` |
| ZMK Studio RPC | 基本対応 | 拡張対応（Custom Protocol 対応） |
| Input Processors | なし（古い upstream ベース） | あり（`zip_temp_layer` 等） |
| バッテリー計算 | カスタムキャリブレーション済み | ZMK 標準値 |
| 有線 Split | なし | あり（`ZMK_SPLIT_RELAY_EVENT`） |

cormoran フォークは upstream により近く（遅延が少ない）、機能追加も多い。

---

## 1. ボード名の変更（`build.yaml`・必須）

cormoran フォークではボード定義が `seeeduino_xiao_ble` から `xiao_ble//zmk` に変わっています。

```yaml
# 変更前（na-ka-no）
board: seeeduino_xiao_ble

# 変更後（cormoran）
board: xiao_ble//zmk
```

`xiao_ble//zmk` は `app/boards/seeed/xiao_ble/xiao_ble_zmk.dts` で定義される ZMK 専用バリアントです。Seeeduino XIAO nRF52840 と同一ハードウェアですが、ZMK 向けの設定（バッテリー電圧測定・フラッシュ・UF2 等）が最適化されています。

---

## 2. `CONFIG_ZMK_MOUSE` の廃止（`_R.conf`・推奨変更）

cormoran フォークでは `ZMK_MOUSE` は非推奨となり、`ZMK_POINTING` が後継です。後方互換のため `ZMK_MOUSE=y` でも動作しますが、明示的に置き換えが推奨されます。

```kconfig
# 変更前
CONFIG_ZMK_MOUSE=y

# 変更後
CONFIG_ZMK_POINTING=y
```

---

## 3. Input Processors の追加（overlay・必須）

na-ka-no フォークは古い upstream をベースにしており、ZMK の Input Processor 機能がありません。cormoran フォークでは正式に利用可能です。PMW3610 ドライバー移行に伴い、automouse と scroll を Input Processor で実装する際に必要です。

```c
// 利用可能になる Input Processor（overlay で使用）
#include <input/processors.dtsi>

&trackball_listener {
    input-processors = <&zip_temp_layer 4 800>;

    scroller {
        layers = <5>;
        input-processors = <&zip_xy_to_scroll_mapper>;
    };
};
```

---

## 4. バッテリー計算の変更（注意・動作確認が必要）

### na-ka-no フォークのカスタマイズ内容

na-ka-no フォークでは以下の 3 ファイルを変更していました。

| ファイル | 変更内容 |
|---------|---------|
| `battery.c` | 電圧しきい値を独自値に変更（3360/2940mV、独自計算式） |
| `battery_common.c` | 5 点移動平均を追加（残量表示のちらつき抑制） |
| `battery_voltage_divider.c` | NiMH 計算関数 → Li-Ion 計算関数に変更 |

### cormoran フォークの状態

| ファイル | 内容 |
|---------|------|
| `battery.c` | ZMK 標準値（4200/3450mV、標準計算式） |
| `battery_common.c` | 移動平均なし（ZMK 標準実装） |
| `battery_voltage_divider.c` | Li-Ion 計算関数を使用（DTS 定義の閾値を参照） |

### 影響

- `battery_voltage_divider.c` の Li-Ion 対応は cormoran フォークでも済んでいる
- 一方、na-ka-no が調整した**電圧しきい値のキャリブレーション**と**移動平均**は失われる
- バッテリー残量の表示精度が変わる可能性がある
- `CONFIG_ZMK_BATTERY_REPORTING_FETCH_MODE_LITHIUM_VOLTAGE=y` は引き続き有効

> **確認推奨**: 移行後にバッテリー残量表示を実機で確認してください。不正確な場合は cormoran フォークへのパッチ適用を検討してください。

---

## 5. ZMK Studio RPC の拡張（`_R.conf`・DYA Studio 向け）

cormoran フォークでは ZMK Studio の RPC が拡張されており、DYA Studio のカスタムプロトコルモジュールと連携できます。

```kconfig
# cormoran フォークで追加された設定
CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256     # フレームデータ転送バッファ拡張
CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=256     # 受信バッファ拡張（必要に応じて）
```

---

## 6. `ZMK_SPLIT_RELAY_EVENT`（`_R.conf`・PMW3610 連携で必要な場合あり）

cormoran フォークで追加された、セントラル → ペリフェラル方向へのイベント中継機能です。BLE 分割キーボードの場合、通常は不要です。ただし `CONFIG_ZMK_PMW3610_SPLIT_RPC_RELAY=y`（PMW3610 の Studio RPC をペリフェラルに中継する場合）と組み合わせて使う場合に必要になることがあります。

cool642tb-mini ではトラックボールがセントラル（右半体）側にあるため、PMW3610 の Split Relay は不要です。

---

## 7. 変更が不要なファイル

| ファイル | 理由 |
|---------|------|
| `cool642tb-mini.dtsi` | ハードウェア定義は変更不要 |
| `Kconfig.shield` | `SHIELD_ROBA_R/L` の定義はそのまま有効 |
| `Kconfig.defconfig` | 左右の役割定義は変更不要 |
| `cool642tb-mini_L.overlay` | 変更不要 |
| `cool642tb-mini_L.conf` | 変更不要 |
| `cool642tb-mini.keymap` | `CONFIG_ZMK_STUDIO_LOCKING=n` で常時解錠のため `&studio_unlock` 不要 |
| `zephyr/module.yml` | `board_root: config` はそのまま有効 |
| `cool642tb-mini.zmk.yml` | 変更不要（`requires: [seeeduino_xiao_ble]` は参照目的のため影響なし） |

---

## 変更サマリー

| ファイル | 変更内容 | 重要度 |
|---------|---------|--------|
| `build.yaml` | `seeeduino_xiao_ble` → `xiao_ble//zmk` | 必須 |
| `config/west.yml` | na-ka-no → cormoran remote、ZMK revision 変更 | 必須 |
| `_R.conf` | `ZMK_MOUSE` → `ZMK_POINTING`、Studio RPC 設定追加 | 推奨 |
| `_R.overlay` | Input Processor 追加（PMW3610 ドライバー移行と併せて） | 必須（PMW3610 移行時） |
| バッテリー精度 | キャリブレーション喪失の可能性 | 実機確認が必要 |

---

## na-ka-no フォーク固有の変更で失われるもの

| 機能 | na-ka-no での実装 | cormoran での状態 | 対応方法 |
|------|----------------|----------------|---------|
| バッテリー電圧しきい値の調整 | `battery.c` カスタム値 | ZMK 標準値（4200/3450mV） | 実機確認後、必要なら手動でパッチ適用 |
| バッテリー残量の移動平均 | `battery_common.c` 5 点平均 | なし（瞬時値） | 必要なら `battery_common.c` を手動でパッチ適用 |
