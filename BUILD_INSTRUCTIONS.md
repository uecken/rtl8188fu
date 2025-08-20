# RTL8188FU ドライバ - ビルド手順と変更点

## 概要
このドキュメントは、RTL8188FUドライバのtxpower設定問題を修正し、ARMv7l向けにビルドするための手順を説明します。

## 重要な変更点

### 1. cfg80211_rtw_set_txpower関数の実装
**ファイル**: `os_dep/linux/ioctl_cfg80211.c`

**変更内容**:
- 未実装だった`cfg80211_rtw_set_txpower`関数を実装
- mBmからdBmへの変換処理を追加
- `registry_priv`の`target_tx_pwr_2g`配列を更新
- `target_tx_pwr_valid`フラグを設定
- 現在のチャンネルでtxpowerを再設定

**実装された機能**:
```c
static int cfg80211_rtw_set_txpower(struct wiphy *wiphy,
    struct wireless_dev *wdev,
    enum nl80211_tx_power_setting type, int mbm)
{
    _adapter *padapter = (_adapter *)rtw_netdev_priv(wdev->netdev);
    struct registry_priv *regsty = adapter_to_regsty(padapter);
    PHAL_DATA_TYPE pHalData = GET_HAL_DATA(padapter);
    int power_dbm;
    
    // mBmからdBmに変換
    power_dbm = mbm / 100;
    
    // target_tx_pwr_2g配列を更新
    regsty->target_tx_pwr_2g[RF_PATH_A][OFDM] = power_dbm;
    regsty->target_tx_pwr_2g[RF_PATH_A][CCK] = power_dbm;
    regsty->target_tx_pwr_valid = _TRUE;
    
    // 現在のチャンネルでtxpowerを再設定
    if (check_fwstate(&padapter->mlmepriv, _FW_LINKED)) {
        PHY_SetTxPowerLevel8188F(padapter, pHalData->CurrentChannel);
    }
    
    return 0;
}
```

### 2. cfg80211_rtw_get_txpower関数の修正
**ファイル**: `os_dep/linux/ioctl_cfg80211.c`

**変更内容**:
- 固定値の12ではなく、実際の設定値を返すように修正
- dBmからmBmへの変換処理を追加

**修正された機能**:
```c
static int cfg80211_rtw_get_txpower(struct wiphy *wiphy,
    struct wireless_dev *wdev,
    int *dbm)
{
    _adapter *padapter = (_adapter *)rtw_netdev_priv(wdev->netdev);
    struct registry_priv *regsty = adapter_to_regsty(padapter);
    int current_power;
    
    // registry_privから現在のtxpowerを取得
    if (regsty->target_tx_pwr_valid == _TRUE) {
        current_power = regsty->target_tx_pwr_2g[RF_PATH_A][OFDM];
    } else {
        // デフォルト値として12dBmを返す
        current_power = 12;
    }
    
    // dBmをmBmに変換して返す
    *dbm = current_power * 100;
    
    return 0;
}
```

### 3. PHY_GetTxPowerIndex_8188F関数の修正
**ファイル**: `hal/rtl8188f/rtl8188f_phycfg.c`

**変更内容**:
- `target_tx_pwr`が設定されている場合はそれを使用するように修正
- dBmからtxpower indexへの変換処理を追加

**修正された機能**:
```c
u8 PHY_GetTxPowerIndex_8188F(
    IN PADAPTER pAdapter,
    IN u8 RFPath,
    IN u8 Rate,
    IN CHANNEL_WIDTH BandWidth,
    IN u8 Channel
)
{
    PHAL_DATA_TYPE pHalData = GET_HAL_DATA(pAdapter);
    struct registry_priv *regsty = adapter_to_regsty(pAdapter);
    s8 txPower = 0, powerDiffByRate = 0, limit = 0;
    BOOLEAN bIn24G = _FALSE;
    
    // target_tx_pwrが設定されている場合はそれを使用
    if (regsty->target_tx_pwr_valid == _TRUE) {
        txPower = regsty->target_tx_pwr_2g[RFPath][Rate];
    } else {
        txPower = (s8) PHY_GetTxPowerIndexBase(pAdapter, RFPath, Rate, BandWidth, Channel, &bIn24G);
    }
    
    // 残りの処理は変更なし
    powerDiffByRate = PHY_GetTxPowerByRate(pAdapter, BAND_ON_2_4G, ODM_RF_PATH_A, RF_1TX, Rate);
    limit = PHY_GetTxPowerLimit(pAdapter, pAdapter->registrypriv.RegPwrTblSel, (u8)(!bIn24G), pHalData->CurrentChannelBW, RFPath, Rate, pHalData->CurrentChannel);
    
    powerDiffByRate = powerDiffByRate > limit ? limit : powerDiffByRate;
    txPower += powerDiffByRate;
    txPower += PHY_GetTxPowerTrackingOffset(pAdapter, RFPath, Rate);
    
    if (txPower > MAX_POWER_INDEX)
        txPower = MAX_POWER_INDEX;
    
    return txPower;
}
```

### 4. リンク確立時のtxpower適用
**ファイル**: `core/rtw_mlme_ext.c`

**変更内容**:
- `mlmeext_joinbss_event_callback`関数でリンク確立時にtxpower設定を適用

**追加された機能**:
```c
void mlmeext_joinbss_event_callback(_adapter *padapter, int join_res)
{
    // 既存のコード...
    
    // Apply txpower setting if configured via cfg80211
    if (regsty->target_tx_pwr_valid == _TRUE) {
        DBG_871X("Applying cfg80211 txpower setting on link established\n");
        PHY_SetTxPowerLevel8188F(padapter, pHalData->CurrentChannel);
    }
}
```

## ビルド手順

### ARMv7lデバイス上での直接ビルド（推奨）

#### 1. 必要なパッケージのインストール
```bash
sudo apt update
sudo apt install build-essential git dkms linux-headers-$(uname -r) -y
```

#### 2. ソースコードの取得
```bash
git clone https://github.com/uecken/rtl8188fu.git
cd rtl8188fu
```

#### 3. ドライバのビルド
```bash
make clean
make
```

#### 4. ドライバのインストール
```bash
sudo make install
```

#### 5. ドライバのロード
```bash
sudo modprobe rtl8188fu
```

### WSL環境でのクロスコンパイル（実験的）

#### 1. WSL環境のセットアップ
```bash
# Ubuntu 22.04をインストール
wsl --install Ubuntu-22.04

# システムを更新
sudo apt update && sudo apt upgrade -y

# ビルドツールをインストール
sudo apt install build-essential git dkms linux-headers-generic -y

# ARMv7l向けのクロスコンパイルツールをインストール
sudo apt install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf -y
```

#### 2. カーネルヘッダーのインストール
```bash
# 最新のカーネルヘッダーをインストール
sudo apt install linux-headers-6.8.0-65-generic -y

# カーネルヘッダーのパスを確認
ls -la /usr/src/linux-headers-6.8.0-65-generic/
```

#### 3. Makefileの修正
Makefileに以下の変更が含まれていることを確認：
- ARMv7l向けのクロスコンパイル設定
- カーネルソースパスの設定
- ARMv7l向けのコンパイラフラグ

#### 4. クロスコンパイルの実行
```bash
make clean
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- KERNEL_SRC=/usr/src/linux-headers-6.8.0-65-generic
```

**注意**: WSL環境でのクロスコンパイルは、アーキテクチャの不一致により問題が発生する可能性があります。

## テスト手順

### 1. ドライバの動作確認
```bash
# ドライバがロードされているか確認
lsmod | grep rtl8188fu

# 無線インターフェースの確認
iwconfig
```

### 2. txpower設定のテスト
```bash
# 現在のtxpowerを確認
iw dev wlan0 get txpower

# txpowerを20dBmに設定
iw dev wlan0 set txpower fixed 20mBm

# 設定が反映されたか確認
iw dev wlan0 get txpower
```

### 3. 接続テスト
```bash
# Wi-Fiネットワークに接続
sudo iw dev wlan0 connect "SSID_NAME"

# 接続状態を確認
iw dev wlan0 link

# txpower設定が維持されているか確認
iw dev wlan0 get txpower
```

## トラブルシューティング

### よくある問題と解決策

#### 1. ビルドエラー
```bash
# エラー: "No such file or directory"
# 解決策: カーネルヘッダーが正しくインストールされているか確認
ls -la /usr/src/linux-headers-$(uname -r)

# エラー: "unrecognized command-line option"
# 解決策: クロスコンパイラのバージョンを確認
arm-linux-gnueabihf-gcc --version
```

#### 2. ドライバのロードエラー
```bash
# エラー: "Unknown symbol"
# 解決策: カーネルバージョンとの互換性を確認
uname -r
modinfo rtl8188fu.ko | grep depends
```

#### 3. txpower設定が反映されない
```bash
# 解決策: ドライバを再ロード
sudo modprobe -r rtl8188fu
sudo modprobe rtl8188fu

# 解決策: デバッグログを確認
dmesg | grep rtl8188fu
```

## ファイル変更一覧

| ファイル | 変更内容 |
|---------|---------|
| `os_dep/linux/ioctl_cfg80211.c` | cfg80211_rtw_set_txpower関数の実装、cfg80211_rtw_get_txpower関数の修正 |
| `hal/rtl8188f/rtl8188f_phycfg.c` | PHY_GetTxPowerIndex_8188F関数の修正 |
| `core/rtw_mlme_ext.c` | リンク確立時のtxpower適用処理の追加 |
| `Makefile` | ARMv7l向けのクロスコンパイル設定の追加 |

## 注意事項

1. **アーキテクチャの互換性**: ARMv7l向けのドライバは、ARMv7lアーキテクチャのデバイスでのみ動作します。

2. **カーネルバージョン**: ドライバは、ターゲットデバイスのカーネルバージョンと互換性がある必要があります。

3. **クロスコンパイルの制限**: WSL環境でのクロスコンパイルは、アーキテクチャの不一致により問題が発生する可能性があります。

4. **テスト環境**: 本番環境に適用する前に、テスト環境で十分にテストしてください。

## ライセンス

このドライバはGPL-2.0ライセンスの下で提供されています。

## 貢献

バグ報告や機能要求は、GitHubのIssuesページで報告してください。 