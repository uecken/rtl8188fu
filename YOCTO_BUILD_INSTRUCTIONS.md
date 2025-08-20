# Yocto Linux環境でのRTL8188FUドライバビルド手順

## 概要
このドキュメントは、Yocto Linux環境（ARMv7l、カーネル6.6.48）でのRTL8188FUドライバのビルドとインストール手順を説明します。

## ターゲット環境
- **OS**: Yocto Linux
- **アーキテクチャ**: ARMv7l
- **カーネルバージョン**: 6.6.48
- **ビルド日**: Thu Aug 29 15:33:59 UTC 2024

## 方法1: ターゲットデバイス上での直接ビルド（推奨）

### 1. 必要なパッケージのインストール

```bash
# パッケージマネージャーの更新
opkg update

# ビルドツールのインストール
opkg install build-essential
opkg install make
opkg install gcc
opkg install git
opkg install linux-headers-$(uname -r)
opkg install dkms

# または、Yocto環境で利用可能なパッケージマネージャーを使用
# apt-get update && apt-get install build-essential make gcc git linux-headers-$(uname -r) dkms
```

### 2. ソースコードの取得

```bash
# 作業ディレクトリの作成
mkdir -p /tmp/rtl8188fu-build
cd /tmp/rtl8188fu-build

# ソースコードのクローン
```

### 3. ドライバのビルド

```bash
# クリーンビルド
make clean

# ビルド実行
make

# ビルド結果の確認
ls -la *.ko
```

### 4. ドライバのインストール

```bash
# ドライバのインストール
sudo make install

# または
sudo cp rtl8188fu.ko /lib/modules/$(uname -r)/kernel/drivers/net/wireless/
sudo depmod -a
```

### 5. ドライバのロード

```bash
# 既存のドライバをアンロード（存在する場合）
sudo modprobe -r rtl8188fu

# ドライバをロード
sudo modprobe rtl8188fu

# ロード確認
lsmod | grep rtl8188fu
```

## 方法2: Yoctoビルドシステムでの統合

### 1. レイヤーの作成

```bash
# Yoctoプロジェクトディレクトリに移動
cd /path/to/your/yocto/project

# 新しいレイヤーを作成
bitbake-layers create-layer meta-rtl8188fu
bitbake-layers add-layer meta-rtl8188fu
```

### 2. レシピファイルの作成

`meta-rtl8188fu/recipes-kernel/rtl8188fu/rtl8188fu_1.0.bb`を作成：

```bitbake
DESCRIPTION = "RTL8188FU WiFi driver with txpower fixes"
LICENSE = "GPL-2.0"
LIC_FILES_CHKSUM = "file://LICENSE;md5=801f80980d171dd6425610833a22dbe6"

inherit module

SRC_URI = "git://github.com/uecken/rtl8188fu.git;protocol=https;branch=master"
SRCREV = "44d722e074a16bf32ca01accee325d2c"

S = "${WORKDIR}/git"

# カーネルモジュールとしてビルド
RPROVIDES_${PN} += "kernel-module-rtl8188fu"

# 依存関係
DEPENDS += "virtual/kernel"

# インストール先
FILES_${PN} += "${base_libdir}/modules/${KERNEL_VERSION}/kernel/drivers/net/wireless/rtl8188fu.ko"

# モジュールの自動ロード
KERNEL_MODULE_AUTOLOAD += "rtl8188fu"

# ビルド設定
EXTRA_OEMAKE += "KERNEL_SRC=${STAGING_KERNEL_DIR}"

do_install_append() {
    install -d ${D}${base_libdir}/modules/${KERNEL_VERSION}/kernel/drivers/net/wireless/
    install -m 0644 ${S}/rtl8188fu.ko ${D}${base_libdir}/modules/${KERNEL_VERSION}/kernel/drivers/net/wireless/
}
```

### 3. ローカル設定ファイルの更新

`conf/local.conf`に以下を追加：

```bitbake
# RTL8188FUドライバをイメージに含める
IMAGE_INSTALL_append = " rtl8188fu"
```

### 4. ビルドの実行

```bash
# ビルド実行
bitbake rtl8188fu

# または、イメージ全体をビルド
bitbake core-image-minimal
```

## 方法3: DKMSを使用した管理

### 1. DKMSレシピの作成

`meta-rtl8188fu/recipes-kernel/rtl8188fu/rtl8188fu-dkms_1.0.bb`を作成：

```bitbake
DESCRIPTION = "RTL8188FU WiFi driver with DKMS support"
LICENSE = "GPL-2.0"
LIC_FILES_CHKSUM = "file://LICENSE;md5=801f80980d171dd6425610833a22dbe6"

inherit dkms

SRC_URI = "git://github.com/uecken/rtl8188fu.git;protocol=https;branch=master"
SRCREV = "44d722e074a16bf32ca01accee325d2c"

S = "${WORKDIR}/git"

# DKMS設定
DKMS_MODULE_NAME = "rtl8188fu"
DKMS_MODULE_VERSION = "1.0"

# 依存関係
DEPENDS += "virtual/kernel"

# インストール先
FILES_${PN} += "/usr/src/${DKMS_MODULE_NAME}-${DKMS_MODULE_VERSION}/"
```

### 2. DKMSのインストール

```bash
# DKMSパッケージのインストール
opkg install dkms

# または
apt-get install dkms
```

### 3. DKMSでの管理

```bash
# モジュールの追加
sudo dkms add -m rtl8188fu -v 1.0

# モジュールのビルド
sudo dkms build -m rtl8188fu -v 1.0

# モジュールのインストール
sudo dkms install -m rtl8188fu -v 1.0

# ステータス確認
sudo dkms status
```

## テスト手順

### 1. ドライバの動作確認

```bash
# ドライバがロードされているか確認
lsmod | grep rtl8188fu

# 無線インターフェースの確認
iwconfig
ip link show

# カーネルログの確認
dmesg | grep rtl8188fu
```

### 2. txpower設定のテスト

```bash
# 現在のtxpowerを確認
iw dev wlan0 get txpower

# txpowerを20dBmに設定
iw dev wlan0 set txpower fixed 20mBm

# 設定が反映されたか確認
iw dev wlan0 get txpower

# デバッグログの確認
dmesg | grep -i txpower
```

### 3. Wi-Fi接続テスト

```bash
# スキャン実行
iw dev wlan0 scan

# ネットワークに接続
iw dev wlan0 connect "SSID_NAME" key 0:password

# 接続状態確認
iw dev wlan0 link

# IPアドレスの取得
dhclient wlan0
```

## トラブルシューティング

### よくある問題と解決策

#### 1. ビルドエラー

```bash
# エラー: "No such file or directory"
# 解決策: カーネルヘッダーの確認
ls -la /usr/src/linux-headers-$(uname -r)
find /usr/src -name "linux-headers*"

# エラー: "KERNEL_SRC not found"
# 解決策: カーネルソースパスの設定
export KERNEL_SRC=/usr/src/linux-headers-$(uname -r)
make KERNEL_SRC=$KERNEL_SRC
```

#### 2. ドライバのロードエラー

```bash
# エラー: "Unknown symbol"
# 解決策: カーネルバージョンの確認
uname -r
modinfo rtl8188fu.ko | grep depends

# エラー: "Invalid module format"
# 解決策: カーネルとの互換性確認
file rtl8188fu.ko
```

#### 3. txpower設定が反映されない

```bash
# 解決策: ドライバの再ロード
sudo modprobe -r rtl8188fu
sudo modprobe rtl8188fu

# 解決策: デバッグログの確認
dmesg | grep rtl8188fu
dmesg | grep -i txpower

# 解決策: ファームウェアの確認
ls -la /lib/firmware/rtlwifi/
```

#### 4. Yocto固有の問題

```bash
# エラー: "Recipe not found"
# 解決策: レイヤーの確認
bitbake-layers show-layers

# エラー: "No provider for rtl8188fu"
# 解決策: レシピファイルの確認
find . -name "*rtl8188fu*"

# エラー: "Failed to build"
# 解決策: ビルドログの確認
tail -f tmp/log/cooker/*.log
```

## 設定ファイル

### 1. udevルールの作成

`/etc/udev/rules.d/99-rtl8188fu.rules`を作成：

```
# RTL8188FU WiFi adapter
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="rtl8188fu", NAME="wlan0"
```

### 2. モジュール設定

`/etc/modules-load.d/rtl8188fu.conf`を作成：

```
# RTL8188FU WiFi driver
rtl8188fu
```

### 3. ネットワーク設定

`/etc/network/interfaces`に追加：

```
auto wlan0
iface wlan0 inet dhcp
    wireless-essid "YOUR_SSID"
    wireless-key "YOUR_PASSWORD"
```

## パフォーマンス最適化

### 1. 電源管理の設定

```bash
# 電源管理を無効化（安定性向上）
echo 'options rtl8188fu power_save=0' | sudo tee /etc/modprobe.d/rtl8188fu.conf
```

### 2. バッファサイズの調整

```bash
# ネットワークバッファの調整
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf
sysctl -p
```

## 注意事項

1. **カーネルバージョン**: カーネル6.6.48との互換性を確認してください。

2. **アーキテクチャ**: ARMv7lアーキテクチャ専用のビルドです。

3. **Yocto環境**: Yocto環境では、パッケージマネージャーが`opkg`または`apt-get`の場合があります。

4. **ファームウェア**: 必要に応じてファームウェアファイルを`/lib/firmware/rtlwifi/`に配置してください。

5. **テスト**: 本番環境に適用する前に、十分にテストしてください。

## ライセンス

このドライバはGPL-2.0ライセンスの下で提供されています。

## サポート

問題が発生した場合は、以下を確認してください：
- カーネルログ: `dmesg`
- システムログ: `journalctl`
- ビルドログ: Yocto環境の`tmp/log/`ディレクトリ 