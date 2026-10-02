# cool642tb-mini ZMK Firmware Config

## プロジェクト概要

Seeeduino XIAO BLE (nRF52840) ベースのカスタム分割キーボード **cool642tb-mini** 向け ZMK ファームウェア設定リポジトリ。

- 右半体にトラックボール (PMW3610) を搭載
- 左半体にロータリーエンコーダー (EC11) を搭載
- ZMK Studio 対応 (右半体 USB 経由)

## ディレクトリ構造

```
config-cool642tb-mini/
├── build.yaml                          # GitHub Actions ビルドマトリクス
├── README.md
├── zephyr/module.yml                   # Zephyr モジュール定義 (board_root: config)
├── firmware/                           # 事前ビルド済み .uf2 ファイル
│   ├── cool642tb-mini_L-seeeduino_xiao_ble-zmk.uf2
│   ├── cool642tb-mini_R-seeeduino_xiao_ble-zmk.uf2
│   └── settings_reset-seeeduino_xiao_ble-zmk.uf2
└── config/
    ├── west.yml                        # ZMK 依存関係 (カスタムフォーク参照)
    ├── cool642tb-mini.json             # ZMK Studio 用物理レイアウト定義
    ├── cool642tb-mini.keymap           # キーマップ定義
    └── boards/shields/Test/
        ├── cool642tb-mini.dtsi         # 共通ハードウェア定義 (matrix, kscan, encoder)
        ├── cool642tb-mini.zmk.yml      # シールドメタデータ
        ├── cool642tb-mini_L.overlay    # 左半体: col-gpios, エンコーダー有効化
        ├── cool642tb-mini_L.conf       # 左半体: EC11, バッテリー設定
        ├── cool642tb-mini_R.overlay    # 右半体: col-gpios, PMW3610 SPI 設定
        ├── cool642tb-mini_R.conf       # 右半体: PMW3610, ZMK Studio, BLE 設定
        ├── Kconfig.shield              # SHIELD_ROBA_L / SHIELD_ROBA_R 定義
        └── Kconfig.defconfig           # デフォルト設定 (キーボード名、分割設定)
```

## ZMK 依存関係 (config/west.yml)

| 依存 | リポジトリ | ブランチ/リビジョン |
|------|-----------|------------------|
| ZMK 本体 | `na-ka-no/zmk` | `for-cool642tb_mini` |
| PMW3610 ドライバー | `na-ka-no/zmk-pmw3610-driver` | `main` |

カスタムフォークを使用しているため、公式 ZMK の最新機能との差異に注意。

## ビルド設定 (build.yaml)

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: cool642tb-mini_R
    snippet: studio-rpc-usb-uart   # ZMK Studio 用
  - board: seeeduino_xiao_ble
    shield: cool642tb-mini_L
  - board: seeeduino_xiao_ble
    shield: settings_reset
```

GitHub Actions (`.github/workflows/build.yml`) で `zmkfirmware/zmk` の `build-user-config.yml@v0.3.0` を使用してビルド。

## ハードウェア仕様

### キーマトリクス
- **行**: 4行、`xiao_d` ピン 1/2/3/6 (col2row、プルダウン)
- **列 (左半体)**: `xiao_d` 10/9/8/7 + `gpio0` 10/9 (6列)
- **列 (右半体)**: `xiao_d` 10/9/8/7 + `gpio0` 10 (5列、col-offset=6)
- **総キー数**: 43キー

### トラックボール (右半体)
- **センサー**: PMW3610 (SPI)
- **SCK/MOSI/MISO**: gpio0 ピン 5/4/4
- **CS**: gpio0 ピン 9 (ACTIVE_LOW)
- **IRQ**: gpio0 ピン 2 (ACTIVE_LOW, PULL_UP)
- **CPI**: 600、オートマウスレイヤー: 4、スクロールレイヤー: 5

### エンコーダー (左半体)
- **EC11**: `xiao_d` ピン 0 (A) / 5 (B)、24ステップ
- **デフォルト動作**: スクロールアップ/ダウン (default_layer)

## キーマップ構成

### レイヤー一覧

| # | 名前 | 主な用途 |
|---|------|---------|
| 0 | default_layer | QWERTY、エンコーダー: スクロール |
| 1 | FUNCTION | 数字行、記号、音量、スクショ |
| 2 | NUM | シフト記号 (`!@#$%` 等) |
| 3 | layer_3 | F1〜F12、矢印、Bluetooth切替 |
| 4 | MOUSE | マウスボタン (MB1/MB2/MB3) |
| 5 | SCROLL | スクロール専用 |
| 6 | layer_6 | 未割り当て |
| 7 | layer_7 | 未割り当て |

### コンボ定義

| キー位置 | 出力 | 用途 |
|---------|------|------|
| 12 + 13 | `LANG2` | 英数切替 |
| 18 + 19 | `LANG1` | かな切替 |
| 38 + 41 | `mo 3` | レイヤー3 一時有効化 |

### 特記事項
- `&mt` の flavor は `balanced`、quick-tap-ms は `0`
- エンコーダーは FUNCTION レイヤーで音量調整 (`C_VOL_UP`/`C_VOL_DN`)
- layer_3 エンコーダー: タブ切替 (`LC(PAGE_UP)`/`LC(PAGE_DOWN)`)

## PMW3610 設定 (cool642tb-mini_R.conf)

重要なパラメータ:
- `CONFIG_PMW3610_CPI=600`
- `CONFIG_PMW3610_ORIENTATION_180=y`
- `CONFIG_PMW3610_INVERT_SCROLL_X=y`
- `CONFIG_PMW3610_SCROLL_TICK=16`
- `CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264`
- `CONFIG_PMW3610_REST1_SAMPLE_TIME_MS=20`
- `CONFIG_PMW3610_POLLING_RATE_125_SW=y`
- `CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=800`
- `CONFIG_PMW3610_MOVEMENT_THRESHOLD=2`
- `CONFIG_PMW3610_SMART_ALGORITHM=y`

## ZMK Studio

- 右半体で `CONFIG_ZMK_STUDIO=y`、`CONFIG_ZMK_STUDIO_LOCKING=n`
- `build.yaml` の右半体ビルドに `snippet: studio-rpc-usb-uart` を指定
- 物理レイアウトは `config/cool642tb-mini.json` で定義

## ファームウェア書き込み

1. XIAO BLE をダブルクリックでブートローダーモードに入れる
2. 対応する `.uf2` ファイルをドライブにドラッグ＆ドロップ
3. 左右それぞれに書き込む (右 → 左 の順を推奨)
4. ペアリングリセットが必要な場合は `settings_reset.uf2` を使用
