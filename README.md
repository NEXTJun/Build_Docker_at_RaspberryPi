# Docker Project in Raspberry Pi


## 一、 前言

做了許多項Arduino, ESP8266之類的MCU Project, 已經有種看到社群分享Non-OS的專案, 大腦能立即浮現從硬體到軟體的各種材料和實作細節的反應。以Prototype的前提下, 對技術開始存在種倦怠感, 是時候該往OS及Server端邁進。

基於社群技術分享的幾個經驗守則：照步驟實作可重現、適當的降低上手門檻, 此次內容選用的技術為：

+ 硬體：樹莓派
+ 作業系統：Linux
+ 發行版：HypriotOS
+ 虛擬機技術：Docker

選用這套組合的目的是硬體入手簡單、社群資源豐富, 且建構在虛擬機上, 可以減少其他單晶片電腦和linux發行版會遇到的套件安裝衝突問題, 他日要構建叢集系統或微服物會方便很多。

此份文件一部分算是本人的筆記, 也會寫到些linux的操作步驟, 日後會以此內容做基礎架構其他專案, 不定期更新。

~~希望別像MCU專案一樣, 分享完一陣煙火後就沒在維護~~

整體花費上, 因採用的多為開源技術, 最貴的就硬體樹莓派, 在還沒搞到控制板對接下, 材料費約1500, 算是把精力集中在軟體面上的專案。

>## 專案難度: ★★★☆☆

---

## 二、 環境架構

### (一) 環境版本

#### 1. 硬體
+ Raspberry Pi 3

#### 2. 軟體
+ HypriotOS Version 1.12.3
    + Linux Kernel 4.19.118-v7
    + Docker Version 19.03.12

### (二) 作業系統燒錄

Linux作業系統發行版：[HypriotOS]
+ 預設使用者帳號：pirate
+ 預設使用者密碼：hypriot

HypriotOS對PC玩家來說, 是個沒聽過的發行版, 但他是個專門用在ARM單晶片電腦內建Docker的輕量化發行版, 配上樹莓派, 可以很方便的大量佈置便宜的Server。至少省去我很多在不同arm架構下linux發行版, 還要debug docker安裝遇到的各種問題。

樹莓派的OS安裝相當簡單, OS提供者會釋出img檔, 將image燒錄進SD卡後, SD卡插入樹莓派後即可運行。燒錄工具有許多種, 筆者在Win10下常用的程式為：[Win32DiskImager]

[HypriotOS]: https://blog.hypriot.com/
[Win32DiskImager]: https://sourceforge.net/projects/win32diskimager/

詳細的燒錄步驟可參考網路上的教學, 請自行參考附上的網路資源。

+ [[ Raspberry Pi ] Win32 Disk Imager 燒錄 SD 卡教學](https://oranwind.org/-raspberry-pi-win32-disk-imager-shao-lu-sd-qia-jiao-xue/)
+ [為樹莓派燒錄作業系統(Raspbian, Ubuntu) (Windows, Mac)](https://medium.com/@bob910078/%E7%82%BA%E6%A8%B9%E8%8E%93%E6%B4%BE%E7%87%92%E9%8C%84%E4%BD%9C%E6%A5%AD%E7%B3%BB%E7%B5%B1-raspbian-ubuntu-windows-mac-5a8fcc0abdd1)

### (三) 網路設定

不管是linux套件更新安裝, 或是從git pull事先寫好的shell script, 連上網路算是最先要處理的事項, 由於HypriotOS各版本間存在部分網路設定差異, 而且cli的環境, 對習慣Windows有圖形化介面的使用者來說很不熟悉, 這邊還是一步步將要打的指令紀錄上。

#### 1. 無線網路連線設定
在HypriotOS 1.12.3中, 預設開啟有線網路連接, 卻關掉無線網路連線的設定, 最要命的是沒有內建nmtui這個好用的網路設定工具。 如果像筆者一樣依賴wifi連線的, 請按照以下步驟一一輸入指令註冊和設定開機連線。

#### (1) 建立無線網路連線連線目標設定檔, 並開啟編輯器
```shell
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

#### (2) 輸入連線資訊, 連線資訊的客製化內容有提供到很細節, 詳細可看此[資源](https://w1.fi/cgit/hostap/plain/wpa_supplicant/wpa_supplicant.conf)
```shell
ctrl_interface=/var/run/wpa_supplicant
update_config=1

network={
ssid="YOUR_NETWORK_NAME"
psk="YOUR_NETWORK_PASSWORD"
}
```

#### (3) 修改無線網路預設開機連線程序設定檔, 並開啟編輯器
```shell
sudo nano /etc/network/interfaces.d/50-cloud-init.cfg
```

#### (4) 增加連線程序設定
```shell
auto wlan0
iface wlan0 inet manual
wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp
```
以下為設定各行的意義

+ 自動啟動第一個無線網卡
> auto wlan0
+ 連線資訊採用手動設定
> iface wlan0 inet manual
+ 載入連線資訊(基地台名和密碼等)
> wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
+ DHCP自動分配IP
> iface default inet dhcp

#### (5) 啟用有線網路連線熱插拔功能, 避免出現"A start job is running for Raise network interface"的開機長時間等待
把有線網路設定eth0設定, 從
```shell
auto eth0
```
修改為
```shell
allow-hotplug eth0
```

#### (6) 重啟網路服務
```shell
sudo service networking restart
```

#### (7) 確認連線狀態
```shell
ifconfig
```
如果在wlan0底下存在192.x.x.x, 表示已成功和路由器溝通, 並取得分配ip

#### (8) 確認網路通暢
```shell
sudo ping 8.8.8.8
```
如果有回傳data, 表示與外網溝通成功, 用ctrl+c退出即可

#### 2. 開通ssh連線設定
每次使用樹莓派都要接上hdmi是很麻煩的事, 我們並未使用桌面gui, 僅是在cli環境下操作。 因此，為了能夠更方便的使用樹莓派, 並因應未來大量裝置操作的問題, 可使用ssh工具做遠端登入操作。 在1.12.3版已預設開啟ssh, 這邊暫不敘述樹莓派開啟功能的步驟。

如果像筆者主要作業系統是Windows10, 可下載[PuTTY]或[TeraTerm]等ssh工具, 只要知道樹莓派的ip位置和帳號密碼, 在同一個網域下就可以遠端登入操作。

[PuTTY]: https://www.putty.org/
[TeraTerm]: https://ttssh2.osdn.jp/index.html.en

#### (1) IP位置查詢
```shell
ifconfig
```
通常市售的路由器會分配的ip位置為192.x.x.x

---

