## 1. Cable Connection  
### (1). Host OS Internet Connection  
1. CX7 QSFP ports for fronthaul and backhaul connections.  
2. RJ45 port for the host OS internet connection.  

### (2). E2E(End to End) Test Connection  
1. CX7 fronthaul port#0 or port#1 must be connected to the fronthaul switch.  
2. Make sue the PTP is configured to use the port connected to the fronthaul switch.  

![說明](images/image_0.png)

---
## 2. Disable Secure Boot  
1. Reboot and press Esc to enter the UEFI BIOS menu.  
2. Use right arrow key to navigate to Security tab.  
3. Use down arrow key to navigate to Secure Boot menu and press Enter.  

![說明](images/image_1.png)

4. Down arrow to select Disable and press Enter.  

![說明](images/image_2.png)

5. Press F4 to save and exit.  

---
## 3. DGX Spark First-Time Setup  

![說明](images/image_3.png)

### 1. GPU  
code :  
$ lspci | grep -i nvidia  
Function: Check whether the system has an NVIDIA device installed (typically an NVIDIA GPU).  

(1). lspci : List all PCI / PCIe devices, such as graphics cards (GPU), network cards (NIC), SSD controllers, USB controllers, and RAID cards.  
(2). | : Pass the output of the previous command to the next command for processing.  
(3). grep -i nvidia : Search for text containing "nvidia".  
grep : Search text.  
-i : Ignore uppercase and lowercase differences.  

Output:  
000f:01:00.0 VGA compatible controller : NVIDIA Corporation Device 2e12 (rev a1)  

(1) 000f:01:00.0 : PCIe device address, format : Domain:Bus:Device.Function  
(2) VGA compatible controller : Display controller (GPU)  
(3) NVIDIA Corporation : Manufacturer : NVIDIA  
(4) Device 2e12 : Device ID  

## 2. NIC  

code :  
lspci | grep -i mellanox  
Function: List all PCIe devices, then display only Mellanox-related hardware.  

(1) mellanox : Mellanox devices, such as ConnectX NICs, InfiniBand, SmartNICs, and RDMA devices.  

Output:  
4 Output
2 ConnectX-7 cards, each with two ports
2 NICs × 2 ports = 4 Ethernet functions


---
## 4. Configure the Network Interfaces (For the following steps)  
Purpose : Ensure that you have the proper netplan config for your local network.  
The network interface names could change after reboot  
--> Create a persistent net link files under /etc/systemd/network, one for each interface.  
Target : To ensure persistent network interface names after reboot  

### (1). Run to check for network devices and look for the entries.  
--> To find the MAC address of the CX7 NIC.  

![說明](images/image_4.png)

code :  
$ sudo apt-get install jq -y  
功能: 安裝 jq 工具  
(1) sudo : 以管理員權限執行  
(2) apt-get install : 安裝軟體  
(3) jq : 一個專門處理 JSON 格式資料的工具，Linux 查硬體時常輸出 JSON，所以 jq 很重要。  
(4) -y : 自動回答 yes  

code :  
$ sudo lshw -json -C network  
功能: 取得所有網卡的詳細資料（用 JSON 格式）  
| jq '.[] | "\(.product), MAC: \(.serial)"'  
功能: 把 JSON 轉成人類看得懂的文字   
| grep "ConnectX-7"  
功能: 只留下 ConnectX-7

(1) lshw : list hardware ， 列出電腦硬體資訊。    
(2) -json : 輸出 JSON 格式，方便 jq 處理。  
(3) -C network : 只列出 network 類別，Ethernet card, NIC, Mellanox 和 Wi-F.  
(4) .[] : 把 JSON 裡「每一個網卡」拿出來  
(5) \(.product) : 取出「網卡型號」  
    \(.serial) : 取出「MAC 位址」 

Output:  
所有 ConnectX-7 網卡 + 每個 port 的 MAC

### (2). Create files at /etc/systemd/network/ with the desired name for the interface and the MAC address found in the previous step.  
功能: 把每張 Mellanox 網卡（用 MAC 位址辨識）固定重新命名成 aerial100~103  

![說明](images/image_5.png)

code:  
(1):  
sudo nano /etc/systemd/network/20-aerial100.link  
作用: 用 nano 編輯一個 link 規則檔  
(2):   
[Match]  
MACAddress=4c:bb:47:ww:ww:ww  
作用: 找到 MAC 是這個的網卡  
(3):  
[Link]  
Name=aerial100  
作用: 把這張卡改名叫 aerial100  

NOTE:  
後面的文件都會假設：
(1):aerial00、aerial01 是拿來接 RU / fronthaul  
(2):而且 aerial00 專門拿來做 PTP（時間同步）  

### (3). Apply the change

![說明](images/image_6.png)

code:  
$ sudo netplan apply  
功能: 套用（啟用）你目前設定的網路配置。  
(1): netplan : Ubuntu 的網路管理工具，用來設定 IP 位址 、 DHCP / static IP 、 gateway 、 DNS 、 網卡設定。  
(2): apply : 套用設定，把設定檔 → 變成實際網路狀態。  

---
## 5. Disable Auto Upgrade  

1. Purpose : prevents the installed version of the low latency kernel from being accidentally changed with a subsequent software upgrade.  
--> Edit the system file, and change the “1” to “0” for both lines.

![說明](images/image_7.png)

code:  
$ sudo nano /etc/apt/apt.conf.d/20auto-upgrades  
功能:打開 Ubuntu 的「自動更新設定檔」來修改內容  
APT::Periodic::Update-Package-Lists "0";  
作用: 不自動更新「套件列表」，" 1 " 是要 ， " 0 " 是不要  
APT::Periodic::Unattended-Upgrade "0";  
作用: 不要自動安裝更新，" 1 " 是半自動 ， " 0 " 是完全不自動

(1): nano : 開啟文字編輯器  
(2): /etc/apt/apt.conf.d/20auto-upgrades : Ubuntu 自動更新設定檔  


2. Disable the fwupd-refresh timer   
--> Prevent fwupdmgr from automatically checking for any updates  

![說明](images/image_8.png)

code:  
$ sudo systemctl mask fwupd-refresh.timer  
功能: 關閉 firmware 自動更新

(1): systemctl : 控制 systemd 服務  
(2): mask : 完全封鎖服務（最強關閉)  
(3): fwupd-refresh.timer : 定期檢查 firmware 更新  

## 6. Install NVIDIA Optimized Ubuntu Kernel

1. Install the NVIDIA optimized Ubuntu kernel

$ sudo apt update    
$ sudo apt install -y linux-image-6.17.0-1014-nvidia  
#NOTE: This will install the specific kernel version, not the latest NVIDIA optimized kernel.

code:  
$ sudo apt update  
功能:確認目前有哪些軟體可以安裝  

(1): update : 讓系統知道目前套件庫裡有哪些版本可以安裝 

code:
$ sudo apt install -y linux-image-6.17.0-1014-nvidia  
功能: 安裝 NVIDIA 指定版本的 Linux 核心

(1): linux-image-6.17.0-1014-nvidialinux-image-6.17.0-1014-nvidia :  
要安裝的 Linux Kernel 套件名稱

2. Update grub to change the default boot kernel  

#The version to use here depends on the latest version that was installed with the previous command.  

$ sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for DGX OS GNU\/Linux>DGX OS GNU\/Linux, with Linux 6.17.0-1014-nvidia"/' /etc/default/grub

code:
$ sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for DGX OS GNU\/Linux>DGX OS GNU\/Linux, with Linux 6.17.0-1014-nvidia"/' /etc/default/grub  
功能: 把 Linux 開機預設 kernel 改成 NVIDIA 指定的 6.17.0-1014-nvidia，確保每次開機都固定進6.17.0-1014-nvidia  

(1): sed : Linux 文字替換工具  
(2): -i : 直接修改檔案（in-place)  
(3): 's/舊文字/新文字/' : 搜尋取代語法  
(4): ^GRUB_DEFAULT=.* : 所有以 GRUB_DEFAULT= 開頭的行，「 ^ 」表示行開頭，「 GRUB_DEFAULT= 」表示指定設定名稱，「 .* 」表示後面任何內容。  
(5): GRUB_DEFAULT="Advanced options for DGX OS GNU/Linux>DGX OS GNU/Linux, with Linux 6.17.0-1014-nvidia" : 將 「 Linux 6.17.0-1014-nvidia 」作為預設開機項目。在 sed 裡，「 / 」 是特殊符號，「 \/ 」才代表真正的 「 / 」。  
(6): /etc/default/grub : GRUB 設定檔，GRUB = Linux 開機管理器。

整體流程 :  

修改 GRUB 設定  
↓  
指定預設 kernel  
↓  
下次開機固定進 NVIDIA kernel  
