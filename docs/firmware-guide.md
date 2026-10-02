# cool642tb-mini ファームウェア設定ファイル解説

## ファイル構成

```
cool642tb-mini/
├── build.yaml                              # ビルドターゲット定義
├── zephyr/module.yml                       # Zephyr モジュール設定
├── .github/workflows/build.yml            # GitHub Actions ワークフロー
└── config/
    ├── west.yml                            # ZMK 依存関係マニフェスト
    ├── cool642tb-mini.keymap               # キーマップ定義
    ├── cool642tb-mini.json                 # ZMK Studio 用物理レイアウト
    └── boards/shields/Test/
        ├── cool642tb-mini.dtsi             # 共通ハードウェア定義
        ├── cool642tb-mini.zmk.yml          # シールドメタデータ
        ├── Kconfig.shield                  # シールド識別子定義
        ├── Kconfig.defconfig               # デフォルト Kconfig 値
        ├── cool642tb-mini_L.overlay        # 左半体デバイスツリー
        ├── cool642tb-mini_L.conf           # 左半体 Kconfig 設定
        ├── cool642tb-mini_R.overlay        # 右半体デバイスツリー
        └── cool642tb-mini_R.conf           # 右半体 Kconfig 設定
```

---

## `config/west.yml` — ZMK 依存関係マニフェスト

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: na-ka-no
      url-base: https://github.com/na-ka-no

  projects:
    - name: zmk
      remote: na-ka-no
      revision: for-cool642tb_mini
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: na-ka-no
      revision: main

  self:
    path: config
```

**役割**: `west` ツールが使用する依存関係の設定ファイル。ビルドに必要なリポジトリを定義します。

| 項目 | 内容 |
|------|------|
| `remotes` | GitHub の取得元を名前付きで定義。`zmkfirmware` は公式、`na-ka-no` はカスタムフォーク |
| `zmk` プロジェクト | na-ka-no の ZMK フォークを使用。`for-cool642tb_mini` ブランチはこのキーボード専用のカスタマイズが含まれる |
| `import: app/west.yml` | ZMK 本体の依存関係（Zephyr、HAL 等）を自動的に引き込む |
| `zmk-pmw3610-driver` | トラックボールセンサー PMW3610 の ZMK ドライバ。na-ka-no のフォーク版を使用 |
| `self: path: config` | このリポジトリ自体のパスを `config/` と宣言 |

---

## `build.yaml` — ビルドターゲット定義

```yaml
include:
  - board: seeeduino_xiao_ble
    shield: cool642tb-mini_R
    snippet: studio-rpc-usb-uart
  - board: seeeduino_xiao_ble
    shield: cool642tb-mini_L
  - board: seeeduino_xiao_ble
    shield: settings_reset
```

**役割**: GitHub Actions がビルドする対象の組み合わせ（ボード × シールド）を列挙します。

| ターゲット | 用途 |
|-----------|------|
| `cool642tb-mini_R` + `studio-rpc-usb-uart` | 右半体（セントラル）。ZMK Studio が USB 経由で接続可能 |
| `cool642tb-mini_L` | 左半体（ペリフェラル） |
| `settings_reset` | ペアリング情報をフラッシュからクリアする専用ファームウェア |

> `snippet: zmk-usb-logging` はコメントアウト中。有効にするとシリアルデバッグログが出力されます。

---

## `.github/workflows/build.yml` — GitHub Actions ワークフロー

```yaml
on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3.0
```

**役割**: push・PR・手動実行をトリガーに ZMK ファームウェアをビルドします。

- ZMK 公式の再利用可能ワークフロー `build-user-config.yml@v0.3.0` を呼び出すだけのシンプルな構成
- このワークフローが `build.yaml` を読み取り、3 つのターゲットを順番にビルドします
- 成果物（`.uf2` ファイル）は GitHub Actions の Artifacts としてダウンロード可能になります

---

## `zephyr/module.yml` — Zephyr モジュール設定

```yaml
name: cool642tb-mini
build:
  settings:
    board_root: config
```

**役割**: このリポジトリを Zephyr モジュールとして認識させる定義ファイル。

- `board_root: config` により、Zephyr のビルドシステムがボード・シールド定義を `config/boards/` 以下から探すようになります
- これがないと `cool642tb-mini_R` / `_L` シールドが見つかりません

---

## `config/boards/shields/Test/cool642tb-mini.zmk.yml` — シールドメタデータ

```yaml
file_format: "1"
id: cool642tb-mini
name: cool642tb-mini
type: shield
url: https://github.com/na-ka-no/zmk-config-cool642tb-mini
requires: [seeeduino_xiao_ble]
features:
  - keys
```

**役割**: ZMK ツールチェーンがこのシールドを識別するためのメタデータ。

- `requires: [seeeduino_xiao_ble]` — このシールドは Seeeduino XIAO BLE ボードと組み合わせて使用することを宣言
- `features: [keys]` — キースキャン機能を持つシールドであることを示す

---

## `config/boards/shields/Test/Kconfig.shield` — シールド識別子定義

```kconfig
config SHIELD_ROBA_R
    def_bool $(shields_list_contains,cool642tb-mini_R)

config SHIELD_ROBA_L
    def_bool $(shields_list_contains,cool642tb-mini_L)
```

**役割**: ビルド時にどちらの半体をビルドしているかを判定する Kconfig シンボルを定義します。

- `shields_list_contains` 関数でシールド名を確認し、`SHIELD_ROBA_R` / `SHIELD_ROBA_L` を `y` にセット
- この値を `Kconfig.defconfig` で参照することで、左右で異なる設定を適用できます

---

## `config/boards/shields/Test/Kconfig.defconfig` — デフォルト Kconfig 値

```kconfig
if SHIELD_ROBA_R
config ZMK_KEYBOARD_NAME
    default "cool642tb-mini_R"
config ZMK_SPLIT
    default y
config ZMK_SPLIT_ROLE_CENTRAL
    default y
endif

if SHIELD_ROBA_L
config ZMK_KEYBOARD_NAME
    default "cool642tb-mini_L"
config ZMK_SPLIT
    default y
endif
```

**役割**: 左右ごとのデフォルト設定値を自動的に適用します。

| 設定 | 右半体 | 左半体 |
|------|--------|--------|
| `ZMK_KEYBOARD_NAME` | `cool642tb-mini_R` | `cool642tb-mini_L` |
| `ZMK_SPLIT` | `y`（分割キーボード有効） | `y` |
| `ZMK_SPLIT_ROLE_CENTRAL` | `y`（セントラル = USB/BLE ホスト側） | 設定なし（ペリフェラル） |

---

## `config/boards/shields/Test/cool642tb-mini.dtsi` — 共通ハードウェア定義

**役割**: 左右両半体が共通で使うハードウェア構成（キーマトリクス、エンコーダー等）を定義します。左右の overlay からこのファイルを `#include` します。

### 物理レイアウト（ZMK Studio 用）

```c
roba_physical_layout: roba_physical_layout {
    compatible = "zmk,physical-layout";
    display-name = "Default";
    transform = <&default_transform>;
    kscan = <&kscan0>;
    keys = < ... 43 キーの座標 ... >;
};
```

- ZMK Studio がキー位置を GUI で表示するための座標情報（単位: 0.01u）
- 座標は `(x, y)` で指定し、左右の中央にギャップ（親指キー部）があります

### キーマトリクス変換

```c
default_transform: keymap_transform_0 {
    compatible = "zmk,matrix-transform";
    columns = <11>;
    rows = <4>;
    map = <
        RC(0,0)...RC(0,10)
        RC(1,0)...RC(1,10)
        RC(2,0)...RC(2,10)
        RC(3,0)...RC(3,10)
    >;
};
```

- 4 行 × 11 列の論理マトリクス（左右合計 43 キー）
- `RC(行, 列)` 形式でキーの物理位置と論理位置を対応付けます

### GPIO キースキャン

```c
kscan0: kscan {
    compatible = "zmk,kscan-gpio-matrix";
    diode-direction = "col2row";
    row-gpios
        = <&xiao_d 1 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>  // 行0
        , <&xiao_d 2 ...>                                   // 行1
        , <&xiao_d 3 ...>                                   // 行2
        , <&xiao_d 6 ...>;                                  // 行3
};
```

- col2row ダイオード配線（列→行方向）
- 行は `xiao_d` 1/2/3/6 ピン（XIAO BLE の D1〜D3, D6）
- 列は左右 overlay でそれぞれ定義

### ロータリーエンコーダー

```c
left_encoder: encoder_left {
    compatible = "alps,ec11";
    a-gpios = <&xiao_d 0 ...>;  // D0 ピン
    b-gpios = <&xiao_d 5 ...>;  // D5 ピン
    steps = <24>;               // 1 回転 24 ステップ
    status = "disabled";        // デフォルト無効（左 overlay で有効化）
};

right_encoder: encoder_right {
    status = "disabled";        // 右半体エンコーダーは常に無効
};
```

### センサー登録（`sensors` ノード）

```c
sensors: sensors {
    compatible = "zmk,keymap-sensors";
    sensors = <&left_encoder &right_encoder>;
    triggers-per-rotation = <10>;
};
```

**役割**: 「どのセンサーを、何番目として、どの頻度でキーマップに届けるか」を定義するノードです。このノードがなければエンコーダーをキーマップで使用できません。

具体的には以下の 2 つのことを行っています。

**1. 使用するセンサーデバイスの宣言と順序付け**

```c
sensors = <&left_encoder &right_encoder>;
```

ここに列挙した順番が、キーマップ内 `sensor-bindings` のインデックスと対応します。

```c
// インデックス 0（left_encoder）に encoder_msc_down_up を割り当て
sensor-bindings = <&encoder_msc_down_up>;
```

**2. 1 回転あたりのイベント発火回数の指定**

```c
triggers-per-rotation = <10>;
```

EC11 の物理ステップ数（24）とは別に、1 回転で何回キーマップイベントを発生させるかを決めます。`10` なら約 2.4 ステップごとに 1 イベントが発火します。

| プロパティ | 値 | 内容 |
|-----------|-----|------|
| `compatible` | `"zmk,keymap-sensors"` | ZMK のセンサー管理ノードとして認識させる |
| `sensors` | `<&left_encoder &right_encoder>` | 登録するセンサーを宣言順に列挙。インデックス 0 が左エンコーダー、インデックス 1 が右エンコーダー |
| `triggers-per-rotation` | `10` | 1 回転あたり 10 回のキーマップイベントを発火（24 ステップ ÷ 10 ≒ 2.4 ステップで 1 イベント） |

---

## `config/boards/shields/Test/cool642tb-mini_L.overlay` — 左半体デバイスツリー

```c
#include "cool642tb-mini.dtsi"

&kscan0 {
    col-gpios
        = <&xiao_d 10 GPIO_ACTIVE_HIGH>  // D10 = P0.10（NFC pin）
        , <&xiao_d 9  GPIO_ACTIVE_HIGH>  // D9  = P0.27
        , <&xiao_d 8  GPIO_ACTIVE_HIGH>  // D8  = P1.13
        , <&xiao_d 7  GPIO_ACTIVE_HIGH>  // D7  = P1.12
        , <&gpio0  10 GPIO_ACTIVE_HIGH>  // P0.10（NFC pin）
        , <&gpio0  9  GPIO_ACTIVE_HIGH>; // P0.09（NFC pin）
};

&left_encoder {
    status = "okay";  // エンコーダーを有効化
};
```

**役割**: 左半体固有のハードウェア設定。

| 設定 | 内容 |
|------|------|
| `col-gpios` | 左半体 6 列分の列 GPIO ピン定義。うち 2 つ（P0.09/P0.10）は NFC ピンを GPIO として転用 |
| `left_encoder` | ロータリーエンコーダーを有効化（右半体では無効のまま） |

> `gpio0 10` と `gpio0 9`（P0.09/P0.10）を GPIO として使うため、`_L.conf` で `CONFIG_NFCT_PINS_AS_GPIOS=y` が設定されています。

---

## `config/boards/shields/Test/cool642tb-mini_L.conf` — 左半体 Kconfig 設定

```kconfig
CONFIG_NFCT_PINS_AS_GPIOS=y          # NFC ピン (P0.09/P0.10) を GPIO として使用
CONFIG_EC11=y                         # EC11 ロータリーエンコーダードライバ有効化
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y  # エンコーダー割り込みをグローバルスレッドで処理

CONFIG_BT_DEVICE_NAME_MAX=30         # BT デバイス名の最大長を 30 文字に拡張

CONFIG_ZMK_BATTERY_REPORTING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=n
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=n
CONFIG_ZMK_BATTERY_REPORTING_FETCH_MODE_LITHIUM_VOLTAGE=y  # 電圧値からバッテリー残量を計算
```

**役割**: 左半体（ペリフェラル）固有の Kconfig 設定。

| 設定 | 内容 |
|------|------|
| `NFCT_PINS_AS_GPIOS` | P0.09/P0.10 の NFC 機能を無効化して GPIO として利用可能にする |
| `EC11` | ロータリーエンコーダードライバを有効化 |
| `BATTERY_REPORTING` | 自身（左半体）のバッテリー残量を BLE 経由で報告する機能を有効化 |
| `SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=n` | 右半体（セントラル）が左半体のバッテリー残量を PC へ代わりに通知（プロキシ）する機能を無効化。左半体のバッテリーは PC に報告しない |
| `SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=n` | 右半体が左半体のバッテリー残量を定期取得する機能を無効化 |
| `FETCH_MODE_LITHIUM_VOLTAGE` | リチウム電池の電圧曲線からバッテリー残量をパーセントで算出 |

> 分割キーボードでは PC と直接通信するのは右半体（セントラル）のみです。左半体（ペリフェラル）のバッテリー残量を PC に伝えるには、右半体が代理で通知（プロキシ）する仕組みが必要ですが、ここでは両機能とも無効にしており、**左半体のバッテリー残量は PC に報告されません。**

---

## `config/boards/shields/Test/cool642tb-mini_R.overlay` — 右半体デバイスツリー

**役割**: 右半体固有のハードウェア設定（キー列 GPIO + PMW3610 トラックボール SPI 接続）。

### キー列 GPIO（右半体は 5 列）

```c
&default_transform {
    col-offset = <6>;  // マトリクスの 6 列目以降を右半体が担当
};

&kscan0 {
    col-gpios
        = <&xiao_d 10 GPIO_ACTIVE_HIGH>  // D10 = P0.10
        , <&xiao_d 9  GPIO_ACTIVE_HIGH>  // D9  = P0.27
        , <&xiao_d 8  GPIO_ACTIVE_HIGH>  // D8  = P1.13
        , <&xiao_d 7  GPIO_ACTIVE_HIGH>  // D7  = P1.12
        , <&gpio0  10 GPIO_ACTIVE_HIGH>; // P0.10
};
```

`col-offset = <6>` により、右半体のキーは論理マトリクスの列 6〜10 にマッピングされます。

### トラックボールリスナー

```c
trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;
};
```

PMW3610 からの入力イベント（相対座標）を ZMK の入力システムに接続します。`automouse-layer` と `scroll-layers` の処理はドライバー側で行われます。

### SPI ピン設定（PMW3610 接続）

```c
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK,  0, 5)>,  // D5  = P0.05（SCK）
                    <NRF_PSEL(SPIM_MOSI, 0, 4)>,  // D4  = P0.04（3 線共用）
                    <NRF_PSEL(SPIM_MISO, 0, 4)>;  // D4  = P0.04（3 線共用）
        };
    };
    // spi0_sleep は省電力時の同一ピン設定（low-power-enable）
};

&xiao_serial { status = "disabled"; };  // UART を無効化して SPI ピンと競合を回避
```

PMW3610 は MOSI/MISO を同じピン（P0.04）で共用する 3 線 SPI 接続です。

### PMW3610 センサーノード

```c
&spi0 {
    compatible = "nordic,nrf-spim";
    cs-gpios = <&gpio0 9 GPIO_ACTIVE_LOW>;  // CS = P0.09（NFC pin）

    trackball: trackball@0 {
        compatible = "pixart,pmw3610";
        spi-max-frequency = <2000000>;                          // 最大 2MHz
        irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>; // IRQ = P0.02
        automouse-layer = <4>;  // トラックボール動作時に MOUSE レイヤー（4）を自動有効化
        scroll-layers = <5>;    // SCROLL レイヤー（5）でスクロールモードに切替
    };
};
```

| ピン | 用途 | GPIO |
|------|------|------|
| SCK | SPI クロック | P0.05 (D5) |
| MOSI/MISO | SPI データ（共用） | P0.04 (D4) |
| CS | チップセレクト | P0.09 (NFC pin) |
| IRQ | 動作検出割り込み | P0.02 (D0) |

---

## `config/boards/shields/Test/cool642tb-mini_R.conf` — 右半体 Kconfig 設定

**役割**: 右半体（セントラル）固有の Kconfig 設定。トラックボール・ZMK Studio・BLE 等を設定します。

### 基本設定

```kconfig
CONFIG_ZMK_KEYBOARD_NAME="cool642tb-mini"  # BLE デバイス名
CONFIG_ZMK_MOUSE=y                          # マウス機能有効化
CONFIG_NFCT_PINS_AS_GPIOS=y                 # NFC ピンを GPIO として使用
CONFIG_ZMK_BLE=y                            # BLE 通信有効化
CONFIG_SPI=y                                # SPI バス有効化
CONFIG_INPUT=y                              # 入力サブシステム有効化
```

### PMW3610 センサー設定

```kconfig
CONFIG_PMW3610=y                            # PMW3610 ドライバー有効化
CONFIG_PMW3610_CPI=600                      # カーソル感度（DPI）
CONFIG_PMW3610_CPI_DIVIDOR=1                # CPI 分割なし（実効 CPI = 600）
CONFIG_PMW3610_ORIENTATION_180=y            # センサーを 180° 回転（実装向きの補正）
CONFIG_PMW3610_INVERT_X=n                   # X 軸反転なし
CONFIG_PMW3610_INVERT_SCROLL_X=y            # スクロール X 軸を反転
CONFIG_PMW3610_SCROLL_TICK=16               # 16 カウントで 1 スクロールステップ
CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264   # 動作モード → REST1 移行時間（ms）
CONFIG_PMW3610_REST1_SAMPLE_TIME_MS=20      # REST1 のサンプリング周期（20ms）
CONFIG_PMW3610_POLLING_RATE_125_SW=y        # ポーリングレート 125Hz（SW モード）
CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=800     # オートマウス タイムアウト 800ms
CONFIG_PMW3610_MOVEMENT_THRESHOLD=2         # オートマウス起動の最小移動量（2 カウント）
CONFIG_PMW3610_SMART_ALGORITHM=y            # 表面トラッキング精度向上アルゴリズム
```

### ZMK Studio 設定

```kconfig
CONFIG_ZMK_STUDIO=y           # ZMK Studio によるランタイムキーマップ編集を有効化
CONFIG_ZMK_STUDIO_LOCKING=n   # ロック機能を無効化（常に編集可能）
```

### エンコーダー・バッテリー設定

```kconfig
CONFIG_EC11=y                                        # EC11 ドライバ（dtsi 定義との整合のため）
CONFIG_ZMK_BATTERY_REPORTING=y                       # 自身（右半体）のバッテリー残量を報告
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=n   # 左半体のバッテリーを PC へプロキシしない
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=n # 左半体のバッテリーを定期取得しない
```

> 分割キーボードでは PC と直接通信するのは右半体（セントラル）のみです。左半体（ペリフェラル）のバッテリー残量を PC に伝えるには、右半体が代理で通知（プロキシ）する仕組みが必要ですが、ここでは両機能とも無効にしており、**左半体のバッテリー残量は PC に報告されません。**

---

## `config/cool642tb-mini.keymap` — キーマップ定義

**役割**: 全レイヤーのキーバインドを定義するメインファイル。

### グローバル設定

```c
#define MOUSE 4
#define SCROLL 5
#define NUM 6
#define ZMK_POINTING_DEFAULT_SCRL_VAL 80  // &msc のスクロール量デフォルト値

&mt {
    flavor = "balanced";  // ホールド/タップの判定方式
    quick-tap-ms = <0>;   // クイックタップ無効
};
```

### コンボ定義

| キー位置 | 出力 | 用途 |
|---------|------|------|
| 12 + 13（D + F） | `LANG2` | 英数切替（macOS: かな→英数） |
| 18 + 19（J + K） | `LANG1` | かな切替（macOS: 英数→かな） |
| 38 + 41（親指左 + 右） | `mo 3` | レイヤー 3 一時有効化 |

### エンコーダー動作

```c
encoder_msc_down_up {
    compatible = "zmk,behavior-sensor-rotate";
    bindings = <&msc SCRL_UP>, <&msc SCRL_DOWN>;
    tap-ms = <20>;
};
```

時計回り → スクロールアップ、反時計回り → スクロールダウン。

### レイヤー一覧

| # | 名前 | 概要 | エンコーダー |
|---|------|------|-------------|
| 0 | `default_layer` | QWERTY 配列。`lt 4 F` でホールド中 MOUSE レイヤー | スクロール↑↓ |
| 1 | `FUNCTION` | 数字行・記号・音量・スクショ等 | 音量↑↓ |
| 2 | `NUM` | シフト記号（`!@#$%` 等） | — |
| 3 | `layer_3` | F1〜F12・矢印・Bluetooth 切替 | タブ切替（Ctrl+PageUp/Down） |
| 4 | `MOUSE` | マウスボタン（MB1/MB2/MB3） | — |
| 5 | `SCROLL` | スクロール専用（全 `&trans`） | — |
| 6 | `layer_6` | 未割り当て | — |
| 7 | `layer_7` | 未割り当て | — |

### デフォルトレイヤー（layer 0）の特記キー

| キー | 動作 |
|------|------|
| `lt 4 F` | F キーをホールドで MOUSE レイヤー（4）に移行 |
| `lt 5 K` | K キーをホールドで SCROLL レイヤー（5）に移行 |
| `lt 2 LANG2` | LANG2 ホールドで NUM レイヤー（2）に移行 |
| `lt 1 LANG1` | LANG1 ホールドで FUNCTION レイヤー（1）に移行 |
| `mt LSHIFT Z` | Z をタップで Z、ホールドで左 Shift |
| `mt LEFT_ALT BACKSPACE` | Backspace タップ、ホールドで左 Alt |
| `mt LCTRL SPACE` | Space タップ、ホールドで左 Ctrl |
| `mt RSHFT ENTER` | Enter タップ、ホールドで右 Shift |
