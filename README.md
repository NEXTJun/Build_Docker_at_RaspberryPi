# Docker Project on Raspberry Pi


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

不同平台有不同的ssh工具, 如果像筆者主要作業系統是Windows10~~為了玩遊戲~~, 可下載[PuTTY]或[TeraTerm]等ssh工具, 只要知道樹莓派的ip位置和帳號密碼, 在同一個網域下就可以遠端登入操作。

[PuTTY]: https://www.putty.org/
[TeraTerm]: https://ttssh2.osdn.jp/index.html.en

#### (1) IP位置查詢
```shell
ifconfig
```
通常市售的路由器分配的ip會是192.x.x.x, 因路由器分配的IP可能會有變動, 如果確認裝置是長時間使用的, 可至路由器設定樹莓派成固定IP。

---

## 三、 Docker基本操作

### (一) Docker介紹
要維持軟體環境的一致性是很困難的是, 在社群分享專案時, 時常會因為別人的電腦少裝package或設定有誤, 導致重現困難, 如果使用虛擬機的話, 可以確立一個相同的執行環境。但傳統上的虛擬機, 建構環境上需要從硬體層開始虛擬化, 往往構建出的image相當大, 運行起來也很耗CPU資源。

Docker是一個好用的虛擬容器技術, 從作業系統層開始虛擬化, 不但輕量化image的檔案大小, 也減少消耗資源。這意味著, 只要建構好Docker image並分享出去, 其他人將image下載並執行後, 就能簡單的複製出成果, 甚至能大量製造運行, 不用再瑣碎的一步步打指令做安裝設定。

Docker有幾個基礎物件概念

+ Image(映像檔)
+ Container(容器)
+ Repository(倉庫)
+ ~~Networks、Volumes、Plugins等, 等我搞清楚後再補充~~ 

#### (1) Image(映像檔)
基礎的唯讀範本, 並包含可運行的最小系統 

好比使用指令docker pull ubuntu:latest, 我們可以得到一個最基礎的ubuntu運行環境

#### (2) Container(容器)
從image建立的執行實例, 且不同的container互相不會影響

好比使用指令sudo docker create –it ubuntu:latest, 我們可以建立一個ubuntu運行環境, 再進入容器內去構建驗證自己的程序

#### (3) Repository(倉庫)
存放image檔案的場所, 可以從倉庫pull映像檔到本機端, 或push上存放的伺服器, 官方管理的公開repository為[Docker Hub](https://hub.docker.com/)

### (二) Docker指令介紹
由於docker主要還是用cli操作, 這邊記錄下常用的幾個指令
*  [info - 查找](#info-查找)
*  [ls - 列舉](#ls-列舉)
*  [delete - 刪除](#delete-刪除)
*  [create - 建立](#create-建立)
*  [operate - 操作](#operate-操作)

### info 查找
#### (1) 查看版本號
```shell
docker version
```

#### (2) 查看現行資訊
```shell
docker info
```

#### (3) 在Repository中搜尋Image
```shell
docker search [image名稱]
```

#### (4) 從Repository下載Image到Local
```shell
docker pull [image名稱]
```
### ls 列舉
#### (1) 列出Local內存在的Image
```shell
docker image ls
or
docker images
```

#### (2) 列出Local內正在運行中的Container
```shell
docker container ls
or
docker ps
```
`-a` 列出包含已停止的所有存在

### delete 刪除
#### (1) 刪除Local內存在的Image
```shell
docker rmi [image名稱]
or
docker image rm [image名稱]
```

#### (2) 刪除Local內所有已停止的Container
```shell
docker container prune -f
or
docker container rm $(docker ps -aq)
```

#### (3) 刪除Local內所有未使用的Image
```shell
docker image prune -f -a
or
docker image rm $(docker images -q)
```

### create 建立
#### (1) 建立並啟動Container
```shell
docker run [Image]
```
Example: 

`docker run -d -p 80:80 nginx` ,

`docker run -it --rm -v ~/app:/app --name my_container ubuntu /bin/bash`
            
`-d` 建立後移至背景執行

`-it` 啟動後可使用互動模式

`-p` 將本機端的port進行轉發, 用法為 `-p <本機端port>:<docker端port>`

`-v` 創建共享資料夾, 用法為 `-v <本地資料夾絕對路徑>:<docker內資料夾的絕對路徑>`

`--rm` 關閉後container自動刪除, 通常用於做簡單驗證。如未帶此參數, container會暫存等待再次啟動

`--name` 設定container的名稱, 用法為 `--name [名稱]`

`/bin/bash`或`bash` 帶此參數後, 可進入容器內的cli進行互動, 而非執行image預設的指令, 可使用`Ctrl+D`或`exit`離開

#### (2) 建立但不啟動Container
```shell
docker create [image]
```

#### (3) 根據該目錄底下, 名為Dockerfile的設定, 建立起Image
```shell
docker build [Dockerfile位置]
```
Example: 

`docker build -t nginx:test .`

`-t, --tag=[名稱]` 為image命名

### operate 操作
#### (1) 啟動已建立的Container
```shell
docker start [container名稱]
```

#### (2) 停止已建立的Container
```shell
docker stop [container名稱]
```

#### (3) 重啟已建立的Container
```shell
docker restart [container名稱]
```

#### (4) 強制停止已建立的Container
```shell
docker kill [container名稱]
```

#### (5) 重新命名已建立的Container
```shell
docker rename [container舊名稱] [container新名稱]
```

#### (6) 複製檔案至已建立的Container
```shell
docker cp [本機端檔案位置] [container名稱]:[container內檔案放置位置]
```

#### (7) 從外部操作運行中的Container
```shell
docker exec [container名稱] [command]
```
Example: 

`docker exec -it 4103ef793ded bash`
            
`-it` 啟動後可使用互動模式

`/bin/bash`或`bash` 帶此參數後, 可進入容器內的cli進行互動, 可使用`Ctrl+D`或`exit`離開

### (三) Dockerfile建立
雖然Docker可以當虛擬機操作, 但Docker的主要目的是將程序的環境做腳本化。 當我們完成一個專案, 且需要做部屬時, 會使用Dockerfile技術, 建立該專案運行的環境, 並將應用程式內容置入該環境中, 整合成為image。 日後只需要將該專案的image建立成container, 即可快速部屬出應用程式。

以下將列出撰寫Dockerfile常用的幾個指令

Example: 
```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.7-slim

# Record the maintainer information
MAINTAINER John john@myemail.com

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN apt-get update && apt-get upgrade -y && pip install pip -U && pip install -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

`FROM`：做為基底所使用到的Docker Image名稱, 如建立時無預載, 會自動從官方的repository下載

`MAINTAINER`：留下撰寫和維護Dockerfile的負責人姓名和聯絡資訊

`WORKDIR`：創立工作目錄

`COPY`：複製資料至指定目錄, 用於將本機端專案放置至docker內

`RUN`：執行Linux指令, 安裝和設定Image所需程序

`EXPOSE`：設定docker端的對外端口, 執行docker run 時可將本機端對接此端口

`ENV`：設定環境變數

`CMD`：執行docker run 時的預設指令

---

## 四、 Docker簡單測試
為測試Docker on Raspberry Pi, 可依循以下步驟進行驗證

### (一) 驗證測試
#### 1. OS環境架設
根據 [環境架構](#二、-環境架構), 將OS燒錄至樹莓派內, 並做好相關設定

#### 2. 檔案載入
將此專案app資料夾內的檔案存放到樹莓派內, 可使用WinSCP、FTP、git clone等方式, 連同app資料夾置於家目錄底下`~/app`

#### 3. 建立Image
app資料夾內含有[(三) Dockerfile建立](#(三)-Dockerfile建立)的Dockerfile Example範例, 使用指令 `docker build -t test ~/app` 建立image

#### 4. 建立並執行Container
執行指令 `docker run -d -p 80:80 test` , 啟動網頁服務 

#### 5. 連線確認
開啟瀏覽器輸入樹莓派ip位置, 確認畫面是否出現 **Hello World!**
