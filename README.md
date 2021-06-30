# 大二筆記-
常見的虛擬化術語:
* Host： 就是虛擬機器所在的那部實體主機
* VM： 就是虛擬機器硬體的意思
* Guest OS： 在 VM 上面所安裝的獨立的作業系統
* Client： 就是你的工作機，跟上述的環境無關！舉例來說，你目前在使用的這部 Windows 工作機，可以透過 remote-viewer 連線到 VM 上，那麼這個系統就可以稱為 client 端的系統。

經常使用到的軟體：
* KVM： 整合到 Linux 核心，是最重要的虛擬化技術，可以虛擬出 CPU 的硬體
* qemu： 相對於 KVM，qemu 則主要在虛擬出各項週邊設備，包括磁碟、網卡、USB、顯卡、音效等
* libvirtd： 提供使用者一個管理 VM 的服務
* virt-manager： 有點類似圖形界面，可以搭配 libvirtd 進行虛擬機器的管理。
* virsh： 終端機界面的管理指令。

## 安裝及啟動
```cmd
yum install qemu-kvm qemu-img virt-manager libvirt libvirt-client virt-install virt-viewer bridge-utils

systemctl start libvirtd
systemctl enable libvirtd
systemctl status libvirtd
● libvirtd.service - Virtualization daemon
   Loaded: loaded (/usr/lib/systemd/system/libvirtd.service; enabled; vendor pr>
   Active: active (running) since Thu 2021-07-01 02:17:48 CST; 1s ago
     Docs: man:libvirtd(8)
           https://libvirt.org
 Main PID: 83842 (libvirtd)

```
---

## 觀察目前啟動的VM

```
virsh list [--all]
 Id    名稱                         狀態
----------------------------------------------------

virsh net-list [--all]
 名稱               狀態     自動啟動  Persistent
----------------------------------------------------------
 default              啟用     yes           yes

```
預設不會有 VM ，而預設會啟用一個名為『 default 』的橋接器！提供一個 VM 內部可以連線的網路狀態！

---

## 關閉與取消定義的虛擬機器、橋接器等
在確認了有 default 這個橋接器，觀察Port狀態
```
netstat -tulnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN      5250/dnsmasq
udp        0      0 192.168.122.1:53        0.0.0.0:*                           5250/dnsmasq
udp        0      0 0.0.0.0:67              0.0.0.0:*                           5250/dnsmasq

```
* Port 53: 提供 VM 向外部查詢 DNS
* port 67: 是提供VMDHCP自動取得網路參數

---
 
## 觀察目前網路卡資訊
```
ip link show
virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500....
virbr0-nic: <BROADCAST,MULTICAST> mtu 1500....
```
發現出現了virbr0、virbr0-nic兩張介面卡。
* virbr0：會自動加入一個192.168.122.X/24的網段給虛擬機器使用，並且使用的是 NAT 的機制，因此使用這個 virbr0 橋接器連結的系統， 就可以自動的透過你的 host 上網了。

觀察設定檔察看預設網路設定值
由於網路轉遞是 NAT 技術，而 DHCP 的範圍則是 192.168.122.2 ~ 192.168.122.254 之間！ 至於 DHCP server 的 IP 則是 192.168.122.1 喔

```
vim /etc/libvirt/qemu/networks/default.xml
<network>
  <name>default</name>
  <uuid>d77a80ea-c1cf-44d1-9157-a71ce0f46b0a</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:60:8e:8f'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>
```
之後我們會自行分配網路，不需要用到libvirtd所提供的DHCP，之後自行建立網路橋接器。
那目前得先把VM(domain)、網路介面(netdomain)關閉，這部分就能使用libvirtd所提供的指令virsh。


```
virsh list --all
#如果沒有機器啟動，就不需要進行關閉的動作

#正常關閉
virsh shutdown 名稱
#強制關閉
virsh destroy 名稱
#取消定義
virsh undefine 名稱
```
```
virsh net-list --all

 名稱               狀態     自動啟動  Persistent
----------------------------------------------------------
 default              啟用     yes           yes

#關閉網路
virsh net-destroy 名稱
#取消網路定義
virsh net-undefine 名稱

#再觀察一次
virsh net-list --all

 名稱               狀態     自動啟動  Persistent
----------------------------------------------------------
```
---
##實作

```
#將 /etc/libvirt/qemu/networks/default.xml 備份到 /root/virtual/ 目錄內
mkdir /root/virtual
cp -a /etc/libvirt/qemu/networks/default.xml /root/virtual

#列出目前所有的虛擬網路橋接器
virsh list --all
virsh net-list --all

#關閉網路
virsh net-destroy 名稱(default)

#完整刪除，及需要取消網路定義
virsh net-undefine 名稱(default)

#查詢監聽port，是否已經刪除
netstat -tunlp | grep 53

```

##手動建立NAT方式的橋接器：
你的虛擬機器想要連線到 Internet 有兩種方式：
* 透過 Host 的 NAT 轉遞，取得 private IP 即可
* 直接透過 bridge 的功能，直接設定對外的 IP 取得方式即可。

qemu 也提供了兩種基本的橋接器給我們使用的:
* 透過 NAT，例如剛剛的 default 網路界面
* 透過直接 forward 到外部實體網卡上！就是直接 bridge 功能！

透過 NAT 的橋接方式
* 綁訂到 Server 的 IP 假設為 192.168.10.254/24 這一個
* 假設有啟動 DHCP 服務，同時提供的動態 IP 範圍在 192.168.10.1~192.168.10.100
* 假設 Host 看到的網路界面名稱就稱為 virbr1 好了。
```
vim /root/virtual/qnet.xml

<network>
    <name>qnet</name>
    <forward mode='nat'/>
    <bridge name='virbr1' stp='on' delay='0'/>
    <mac address='52:54:00:66:ff:0c'/>
    <ip address='192.168.10.254' netmask='255.255.255.0'>
        <dhcp>
            <range start='192.168.10.1' end='192.168.10.100'/>
        </dhcp>
    </ip>
</network>
```
##手動建立橋接器，直接連結到實體網卡上面：
```
vim /root/virtual/qforward.xml

<network>
  <name>qforward</name>
  <forward dev='eno1' mode='bridge'>
    <interface dev='eno1'/>
  </forward>
</network>

#查看狀態
virsh net-create qforward.xml
 virsh net-list
 名稱               狀態     自動啟動  Persistent
----------------------------------------------------------
 qforward             啟用     no            no
 qnet                 啟用     no            no
```
---
##設計虛擬磁碟
* 虛擬磁碟可以是實體磁碟、可以是檔案、可以是 LVM 裝置等等
* 虛擬磁碟可以使用 qemu-img 來建置
* 常見的虛擬磁碟格式，主要為 qcow2 與 raw ，其餘不要考慮
* raw 格式最快，但是得先預留出磁碟容量，因此不建議
* qcow2 還在持續發展中，速度與 raw 已經差不多，而且檔案系統用多少，算多少。

```
基本qemu指令查詢
qemu-img --help


#qemu-img製作一個40G的虛擬磁碟
cd /vmdisk
qemu-img create -f qcow2 /vmdisk/centos8.ver10.img 40G

qemu info centos8.ver10.img

image: centos8.img
file format: qcow2
virtual size: 40 GiB (42949672960 bytes)
disk size: 4.64 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false

```
---
##下載所需vm系統

自行查找所需iso
---

##建立XML檔案
```
virt-install --name centos8.v1 \
	--cpu host --vcpus 4 --memory 4096 --memballoon virtio \
	--clock offset=utc \
	--controller virtio-scsi \
	--disk /vmdisk/centos8.ver10.img,cache=writeback,io=threads,device=disk,bus=virtio \
	--network network=qnet,model=virtio \
	--graphics spice,port=5910,listen=0.0.0.0,password=centos8 \
	--cdrom /vmdisk/iso/CentOS-8.3.2011-x86_64-dvd1.iso \
	--video qxl \
	--dry-run --print-xml \
    > /vmdisk/centos8.ver10.xml
```
##修改XML檔
* 把74行以後的都刪除，保留1~73行
* 第23行加入
```
<on_poweroff>destroy</on_poweroff>
<on_reboot>restart</on_reboot>
<on_crash>restart</on_crash>
```
* 第44行刪除或註解
```<controller type="virtio-scsi" index="0"/>```
* 第56行改成
```source network="qnet"/>```
* 65行
```
<graphics type="spice" port="5911" tlsPort="-1" listen="0.0.0.0" passwd="centos7">
將其中的 tlsPort="-1"刪除(沒有刪除會強制加密)
```
---
XML

```
<domain type="kvm">
  <name>centos8.v1</name>
  <!--uuid>42560c85-a72d-4174-889f-2aa35d65da07</uuid-->
  <memory>4194304</memory>
  <currentMemory>4194304</currentMemory>
  <vcpu>4</vcpu>
  <os>
    <type arch="x86_64" machine="pc-i440fx-rhel7.6.0">hvm</type>
    <boot dev="cdrom"/>
    <boot dev="hd"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <vmport state="off"/>
  </features>
  <cpu mode="host-model"/>
  <clock offset="utc">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2" cache="writeback" io="threads"/>
      <source file="/vmdisk/centos8.ver10.img"/>
      <target dev="vda" bus="virtio"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/vmdisk/iso/CentOS-8.1.1911-x86_64-dvd1.iso"/>
      <target dev="hda" bus="ide"/>
      <readonly/>
    </disk>
    <!--controller type="virtio-scsi" index="0"/-->
    <controller type="usb" index="0" model="ich9-ehci1"/>
    <controller type="usb" index="0" model="ich9-uhci1">
      <master startport="0"/>
    </controller>
    <controller type="usb" index="0" model="ich9-uhci2">
      <master startport="2"/>
    </controller>
    <controller type="usb" index="0" model="ich9-uhci3">
      <master startport="4"/>
    </controller>
    <interface type="network">
      <source network="qnet"/>
      <mac address="52:54:00:18:19:84"/>
      <model type="virtio"/>
    </interface>
    <console type="pty"/>
    <channel type="spicevmc">
      <target type="virtio" name="com.redhat.spice.0"/>
    </channel>
    <input type="tablet" bus="usb"/>
    <graphics type="spice" port="5910" listen="0.0.0.0" passwd="centos8">
      <image compression="off"/>
    </graphics>
    <sound model="ich6"/>
    <video>
      <model type="qxl"/>
    </video>
    <redirdev bus="usb" type="spicevmc"/>
    <redirdev bus="usb" type="spicevmc"/>
    <memballoon model="virtio"/>
  </devices>
</domain>
```
---
##啟動虛擬機
```
virsh create /vmdisk/centos8.ver10.xml
```
---
##觀察虛擬機

```
virsh list

 Id    名稱                         狀態
----------------------------------------------------
 4     centos8.v01                    執行中
```
##放行iptables
```
vim /root/firewall.sh
iptables -A INPUT -s IP -p tcp --dport 5910 -j ACCEPT
```

##Remote-Viewer
spice://YourIP:Port

##效能調校
 CPU 核心與執行緒的對應:
 每台實體機不同
```
cpupower monitor

 CPU| C3   | C6   | PC3  | PC6   || C7   | PC2  | PC7   || C0   | Cx   | Freq  || POLL | C1   | C1E  | C3   | C6
   0|  0.20| 97.18|  0.13| 89.54||  0.00|  1.02|  0.00||  0.33| 99.67|  3014||  0.00|  0.00|  0.02|  0.00| 99.64
   4|  0.20| 97.17|  0.13| 89.54||  0.00|  1.02|  0.00||  0.67| 99.33|  1834||  0.00|  0.00|  0.07|  0.22| 99.04
   1|  0.02| 98.72|  0.13| 89.54||  0.00|  1.02|  0.00||  0.20| 99.80|  2374||  0.00|  0.00|  0.04|  0.00| 99.74
   5|  0.02| 98.72|  0.13| 89.54||  0.00|  1.02|  0.00||  0.22| 99.78|  2299||  0.00|  0.00|  0.05|  0.00| 99.72
   2|  0.18| 95.78|  0.13| 89.54||  0.00|  1.02|  0.00||  2.85| 97.15|  3595||  0.00|  0.00|  0.01|  0.00| 97.12
   6|  0.18| 95.78|  0.13| 89.54||  0.00|  1.02|  0.00||  0.31| 99.69|  2220||  0.00|  0.32|  0.10|  0.20| 99.07
   3|  0.03| 97.93|  0.13| 89.54||  0.00|  1.02|  0.00||  0.49| 99.51|  1890||  0.00|  0.00|  0.02|  0.00| 99.48
   7|  0.03| 97.93|  0.13| 89.54||  0.00|  1.02|  0.00||  0.30| 99.70|  2009||  0.00|  0.00|  0.01|  0.00| 99.68

```
##所以修改XML
* 原本的:
```
<vcpu>4</vcpu>
   <os>
   <type arch="x86_64" machine="pc-i440fx-rhel7.6.0">hvm</type>
   <boot dev="cdrom"/>
   <boot dev="hd"/>
   </os>
   <features>
   <acpi/>
   <apic/>
   <vmport cccstate="off"/>
   </features>
   <cpu mode="host-model"/>
```
* 建議修改:
```
<vcpu placement='static'>4</vcpu>
<cputune>
    <!--分別對應在不同的運算核心上面，使用 4~7 或 0~3 均可！ -->
    <vcpupin vcpu='0' cpuset='4'/>   
    <vcpupin vcpu='1' cpuset='5'/>
    <vcpupin vcpu='2' cpuset='6'/>
    <vcpupin vcpu='3' cpuset='7'/>
</cputune>

<!--將指令 bypass 給 host CPU！ -->
<cpu mode='host-passthrough'>                         
    <arch>x86_64</arch>
    <model>SandyBridge-IBRS</model>
    <vendor>Intel</vendor>
    <microcode version='47'/>
   <!--使用與 host 相同的設計，只是 threads 設計為 1 個-->
    <topology sockets='1' cores='4' threads='1'/> 
</cpu>
```
