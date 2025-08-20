
■環境
uname -a
Linux moment-ff1-8f1a85bc 6.6.48 #1 SMP PREEMPT Thu Aug 29 15:33:59 UTC 2024 armv7l GNU/Linux
RFIC: RTL8188FTV
WiFi Module: RTL8188FTM
Driver:/lib/modules/6.6.48/extra/rtl8188fu.ko
md5sum rtl8188fu.ko rtl8188fu.ko
44d722e074a16bf32ca01accee325d2c  rtl8188fu.ko

■問題
下記コマンドで送信電力を20dBmに設定しても、txpowerは12.00 dBmのままとなる。
ip link set wlan0 down
iw dev wlan0 set txpower fixed 20mBm
ip link set wlan0 up
        Interface wlan0
                ifindex 67
                wdev 0x1e00000001
                addr 4c:a3:8f:1a:85:bc
                type managed
                txpower 12.00 dBm

                