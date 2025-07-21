---
title: "[筆記] LVM & Storage Pool 初始化設定"
categories:
  - 筆記
tags:
  - VM
  - QEMU/KVM
toc: true
toc_label: Index
---

硬碟分區+ LVM + storage pool 設定流程

## Install

```bash
sudo apt update
sudo apt install libvirt-daemon-system libvirt-clients virt-manager gir1.2-spiceclientgtk-3.0 qemu-kvm
sudo apt install thin-provisioning-tools
sudo usermod -aG libvirt $(whoami)
newgrp libvirt
```

## 流程示意

```text
【第一層：物理硬體 (Hardware)】
   └─ 物理硬碟 (/dev/sda, /dev/sdb)
      │
      └─ 【第二層：清除流程 (確保硬碟乾淨)】
           ├─ 使用指令清除分區表和檔案系統簽章，例如：
           │     ├─ wipefs -a /dev/sda
           │     ├─ dd if=/dev/zero of=/dev/sda bs=1M count=100
           │     └─ 確認硬碟已無分區：fdisk -l /dev/sda
           │
           └─ 注意事項：
                 ├─ 有資料的硬碟也能分區，但新分區會破壞原資料結構
                 └─ 建議正式使用前清除殘留資料，避免誤判與混亂

      │
      └─ 【第三層：分區 (Partitions)】
           └─ 利用分區工具（fdisk、parted）將物理硬碟分割成分區
                └─ 產生物理分區，如 /dev/sda1、/dev/sda2、/dev/sdb1 等

      │ └─ 【第四層：LVM 基礎設施 (在主機 OS 中操作)】
           └─ 實體卷 (PVs)
                └─ 使用 pvcreate 將分區初始化成實體卷 (PV)
                     └─ 例如：pvcreate /dev/sda1 /dev/sdb1
                │
                └─ 使用 vgcreate 組合實體卷成卷組 (VG)
                     └─ 例如：vgcreate vg_vms /dev/sda1 /dev/sdb1
                │
                └─ 卷組 (VG) 範例：
                     ┌──────────────────────────────┐
                     │  卷組 (VG): vg_vms          │  <== 統一的儲存資源池
                     └──────────────────────────────┘

                │
                ├───────────┐ (分支一：為 Directory Pool 準備)
                │           │
                │           ▼ (手動使用 lvcreate + mkfs + mount)
                │        ┌───────────────────────────────────┐
                │        │ 邏輯卷 (LV) -> 檔案系統 (ext4) -> 掛載點 │
                │        │ e.g., /dev/vg_vms/lv_for_isos -> /isos  │
                │        └───────────────────────────────────┘
                │
                └───────────┐ (分支二：為 LVM Pool 準備)
                            │
                            ▼ (保持原始狀態，不格式化)
                         ┌───────────────────────────────────┐
                         │ 剩餘未格式化的原始儲存空間          │
                         │ (由 vg_vms 直接提供)              │
                         └───────────────────────────────────┘

【第五層：Libvirt 儲存池 (虛擬化管理層操作)】
   │
   ├─ (對應分支一) ─> 【Directory Pool: 'iso-pool'】
   │                    │ (類型: dir)
   │                    │ (目標路徑: /isos)
   │                    │ (用途: 存放 .iso、.qcow2 等映像檔)
   │                    └──────────────────────────────────┘
   │
   └─ (對應分支二) ─> 【LVM Pool: 'vms-lvm-pool'】
                        │ (類型: lvm)
                        │ (來源: vg_vms)
                        │ (用途: 自動為 VM 生成 LV 作為虛擬磁碟)
                        └──────────────────────────────────┘
                           │
                           ▼

【第六層：虛擬機 (VMs) 的使用】
   │
   ├─ ISO 映像檔 (例如 ubuntu.iso)
   │    └─ 從 【Directory Pool: 'iso-pool'】 中選取並掛載到 VM 用於安裝。
   │
   └─ 虛擬機 (例如 anfu-vm-01)
        │
        └─ 建立 VM 時，從 【LVM Pool: 'vms-lvm-pool'】 中請求虛擬磁碟 (LV)
           │
           ▼
        ┌──────────────────────────────────────────────────┐
        │  虛擬磁碟 (由 Libvirt 自動建立新的 LV)              │
        │  例如: /dev/vg_vms/anfu-vm-01.img                 │
        └──────────────────────────────────────────────────┘
```

簡單理解就是 `pvs 就是硬碟抽象成的物件, 然後可以選擇這些物件創建 vgs (pool) , 之後根據需要從vgs創建 適合大小的lv or 整個vgs轉成lvm`

- lvm 無法被vm直接使用, 需轉成storage pool, 之後就可以選擇pool 分割出一個volume, vm就可以選擇這個volume開機
- lv 分割後可以直接格式化 然後mount作為 filesystem 目錄使用, 直接存qcow2 就可以放vm檔案, 但須自己管理檔案, 還有一個作法是把該dir可以定義成storage pool, 可以更方便管理映像檔

## 磁碟分區

分區目的是

- 切割硬碟空間
- 標記類型碼,跟作業系統說這塊磁區用哪種方式處理 e.g. lvm, linux file system , windows ntfs , EFI 系統分區(製作系統安裝碟)

以上資訊會寫在硬碟的開頭,可以理解成這硬碟的metadata 稱為分區表, 紀錄資訊例如 磁區0~x 是lvm, 磁區 x-y 是 ntfs ..., 但這分區表有兩種格式

- MBR: 舊格式
- GPT: 目前主流

補充: GPT在硬碟結尾還會有一份 備份分區表

### 分區表格式比較

目前主流應該都是GPT , MBR是比較舊的類型(現在硬碟很容易就超過2T,比較不適用)

| 特性             | MBR (Master Boot Record)     | GPT (GUID Partition Table)                         |
| :--------------- | :--------------------------- | :------------------------------------------------- |
| **硬碟容量限制** | 最大支援 2TB                 | 理論上最大支援 9.4 ZB                              |
| **分區數量限制** | 最多 4 個主要分區            | 預設最多 128 個分區                                |
| **可靠性**       | 分區表僅存於磁碟開頭，無備份 | 分區表在磁碟頭尾各有一份備份                       |
| **啟動方式**     | 配合傳統 BIOS (Legacy)       | 配合現代 UEFI                                      |
| **分區識別**     | 使用 32-bit 磁碟簽章         | 為每個分區使用 GUID (全域唯一識別碼)               |
| **作業系統支援** | 幾乎所有系統                 | 現代 64-bit 作業系統 (如 Windows 7+, macOS, Linux) |

### 基本相關指令

```bash
lsblk
# 顯示目前物理硬碟狀態(無論有無掛載)

df
# 顯示掛載硬碟狀態
```

### 從當前硬碟重新初始化

- unmount

假設要做一些操作, 若有被掛載, 可能會有一些issue, 需保證MOUNTPOINTS 需為空白

```bash
lsblk
#NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
#sda      8:0    0  1.8T  0 disk
#├─sda1   8:1    0  800G  0 part /data1
#└─sda2   8:2    0    1T  0 part /data2/backups

umount /data1
umount /data2/backups

or

umount /dev/sda1
umount /dev/sda2
```

若被PID佔用

```bash
fuser -kv /data1
# 強制中止PID
```

- 清除磁區資料

```bash
blkdiscard -v /dev/sda
## ssd 專用 對設備壽命佳

dd if=/dev/zero of=/dev/sda bs=4M status=progress
## hdd
```

### 建立分區

可先用需用lsblk 查找硬碟代號 e.g. `nvme0n1` `sda`

`sgdisk` 是GPT專用的現代硬碟工具

```bash
# 變數化設備名稱
NEW_DISK="/dev/sdx"
sudo sgdisk --new=1:0:0 --typecode=1:8e00 "${NEW_DISK}"
# -n, --new=<partition:start:end>
# -t, --typecode=<partition:code>
# 8e00 是 Linux LVM 的 GPT GUID 代碼


sudo partprobe /dev/sdx
# 分區後, 讓系統重新解析分區
# or reboot 建議重開比較穩
```

可先用需用lsblk 查找硬碟代號 e.g. `nvme0n1` `sda`

`sgdisk` 是GPT專用的現代硬碟工具

```bash
# 變數化設備名稱
NEW_DISK="/dev/sdx"
sudo sgdisk --new=1:0:0 --typecode=1:8e00 "${NEW_DISK}"
# -n, --new=<partition:start:end>
# -t, --typecode=<partition:code>
# 8e00 是 Linux LVM 的 GPT GUID 代碼


sudo partprobe /dev/sdx
# 分區後, 讓系統重新解析分區
# or reboot 建議重開比較穩
```

### typecode對照表

typecode `代碼:分區類型碼` 標記該磁區用途

每個硬碟代號 e.g. sda , nvme0n1 (不同兩顆硬碟), 分區代號都是分開的 前者有1,2,3 後者也可以1,2,3

`8300為預設代碼` ext4 linux 標準檔案系統

| 代碼 (Code) | GPT 名稱                 | 用途                                            |
| :---------- | :----------------------- | :---------------------------------------------- |
| **`8300`**  | **Linux filesystem**     | **(預設值)** 標準 Linux 檔案系統 (ext4, XFS 等) |
| **`8e00`**  | **Linux LVM**            | 用於 LVM 的實體卷冊 (PV)                        |
| **`8200`**  | **Linux swap**           | 用於 Linux 交換空間 (虛擬記憶體)                |
| **`ef00`**  | **EFI System Partition** | 用於 UEFI 啟動的系統分區 (ESP)                  |
| `0700`      | Microsoft basic data     | 用於 Windows NTFS, FAT32 等                     |

### --new 規則

```text
第1個數字: 分區代號 , 0 請自動尋找最低的可用分區編號 , 若是1 則是指定分區代號1
第2個數字: 起始磁區 , 0 表示自動尋找適當的起始位置
第3個數字: 結束磁區 , 0 表示使用全部可用空間
```

其他範例

```bash
sgdisk --new=0:0:+500G /dev/sdx
# 建立一個 500GB 的分區

sgdisk --new=0:0:+100G --new=2:0:0 /dev/sdx
# 建立兩個分區，第一個 100GB，第二個使用剩下所有空間

sgdisk --new=0:0:0 --typecode=1:8e00 /dev/sda
# 把sda 全部建立成一個個lvm 格式的硬碟, 自動抓取分區代號, 與磁區起始位置, 然後全部建立
# Creating new GPT entries in memory.
# Warning: The kernel is still using the old partition table.
# The new table will be used at the next reboot or after you
# run partprobe(8) or kpartx(8)
# The operation has completed successfully.
```

## LVM 初始化

流程 PVS, VGS , LV or LVM

### PVS

Physical Volumes: LVM的的基本單位, 簡單說可以把 硬碟空間抽象成lvm的單位

- 檢視

```bash
pvs
# 查看被分區成 pvs 的基本單位
#PV             VG        Fmt  Attr PSize    PFree
#/dev/nvme0n1p3 ubuntu-vg lvm2 a--  <950.82g <850.82g
#/dev/sda       local-lvm lvm2 a--    <1.82t  512.00m

#PV: 實體卷的路徑。可以是整顆硬碟 (/dev/sdc) 或一個分割區 (/dev/sda2)。
#VG: 這個實體卷目前屬於哪個「卷組」(Volume Group)。
#PSize: 這個實體卷的總大小。
#PFree: 這個實體卷上還有多少空間未被分配給卷組。
```

- 創建

創建PVS, 先檢視格式化後的磁碟分區代號

```bash
lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   1.8T  0 disk
└─sda1                      8:1    0   1.8T  0 part
```

```bash
pvcreate /dev/sda1
# Physical volume "/dev/sda1" successfully created.
pvs
#  PV             VG        Fmt  Attr PSize    PFree
#  /dev/nvme0n1p3 ubuntu-vg lvm2 a--  <950.82g <850.82g
#  /dev/sda1                lvm2 ---    <1.82t   <1.82t
```

### VGS

Volume Groups: 可以理解成創建一個pool, 內含各硬碟被分區成LVM單位

`一般來說一個硬碟只會有一個vgs,  或是把多個硬碟合併成一個vgs`

- 檢視vgs

```bash
vgs
#VG        #PV #LV #SN Attr   VSize    VFree
#ubuntu-vg   1   1   0 wz--n- <950.82g <850.82g

#VG: 卷組的名稱。
##PV: 這個卷組是由幾個實體卷 (PVs) 組成的。 (vg_data 是由 2 個 PV 組成)
##LV: 這個卷組上已經切出了幾個「邏輯卷」(lvg)。
#VSize: 這個卷組的總大小 (所有 PV 大小的總和)。
#VFree: 這個卷組中還有多少可用空間可以讓你切出新的邏輯卷 (lvg)。

pvs
#  PV             VG        Fmt  Attr PSize    PFree
#  /dev/nvme0n1p3 ubuntu-vg lvm2 a--  <950.82g <850.82g
#  /dev/sda1                lvm2 ---    <1.82t   <1.82t
# 可以看到若尚未設置vgs vg是空的

```

- 創建vgs

```bash
vgcreate anfu_vg /dev/sda1
#Volume group "anfu_vg" successfully created

pvs
#  PV             VG        Fmt  Attr PSize    PFree
#  /dev/nvme0n1p3 ubuntu-vg lvm2 a--  <950.82g <850.82g
#  /dev/sda1      anfu_vg   lvm2 a--    <1.82t   <1.82t
# vg 會非空白
vgs
#  VG        #PV #LV #SN Attr   VSize    VFree
#  anfu_vg     1   0   0 wz--n-   <1.82t   <1.82t
#  ubuntu-vg   1   1   0 wz--n- <950.82g <850.82g
# vgs 會顯示在列表

```

- 擴展vgs

```bash
vgextend anfu_vg /dev/sda2
# 假設有一個新的pvs
# 可以這樣擴展 vgs的大小
```

### 方案a. LV

Logical Volumes: 可以理解成選定一個pool(vgs) 拿出需要`固定大小`的 pvs, 輸出成 lv

`這種是固定大小方案`

lv = vgs 切一部分容量成 lv, 這邊可以直接格式化成filesystem mount(這邊需要自己管理檔案,若存vm可用qcow2), filesystem也可以抽象成storage pool, 但實際上適用於管理裡面的映像檔

- 檢視lvg

```bash
lvg
#LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
#ubuntu-lv ubuntu-vg -wi-ao---- 100.00g
```

- 創建lvg

假設我要從某vgs 切出一塊 lvg

```bash
lvcreate -n another_lv -L 200G anfu_vg

# -n or --name  lvg name
# -L or --size  lvg size

lvcreate --name will_tainan --size 200G anfu_vg
#  Logical volume "will_tainan" created.

```

- 擴展lvg

擴展路徑規則 `/dev/<vgs name>/<lvg name>`

```bash
lvextend -L +8G /dev/anfu_vg/will_tainan
# 假設我要擃展 will_tainan lvg的大小

```

lvg 切出固定大小後 就是要格式化, 之後用mount掛載成目錄, 就可以當作一般硬碟用

```bash
sudo mkfs.ext4 /dev/ubuntu-vg/test_vm_storage
```

若正在掛載, 需更新掛載時設定的檔案系統大小

路徑規則 `/dev/<vgs name>/<lvg name>`

```bash
resize2fs /dev/anfu_vg/will_tainan
```

- 刪除lvg

確認不能被掛載, MOUNTPOINTS 需為空

```bash
lsblk
#NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
#sda                         8:0    0   1.8T  0 disk
#└─sda1                      8:1    0   1.8T  0 part
#  └─anfu_vg-will_tainan   252:1    0   200G  0 lvm  /mnt/test
#nvme0n1                   259:0    0 953.9G  0 disk
#├─nvme0n1p1               259:1    0     1G  0 part /boot/efi
#├─nvme0n1p2               259:2    0     2G  0 part /boot
#└─nvme0n1p3               259:3    0 950.8G  0 part
#  └─ubuntu--vg-ubuntu--lv 252:0    0   100G  0 lvm  /

```

```bash
sudo umount /mnt/test
# 先確認不能被掛載狀態


lvremove /dev/anfu_vg/will_tainan
# 刪除lvg, 會回到vgs的容量


vgs
#  VG        #PV #LV #SN Attr   VSize    VFree
#  anfu_vg     1   1   0 wz--n-   <1.82t   <1.53t
#  ubuntu-vg   1   1   0 wz--n- <950.82g <850.82g
lvg
#  LV          VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
#  will_tainan anfu_vg   -wi-a----- 300.00g
#  ubuntu-lv   ubuntu-vg -wi-ao---- 100.00g
lvremove /dev/anfu_vg/will_tainan
#Do you really want to remove and DISCARD active logical volume anfu_vg/will_tainan? [y/n]: Y
#  Logical volume "will_tainan" successfully removed.

```

### 方案b. lvm

Logic Volume Manage: 這種是把一個pvs 直接抽象成物件,不分容量,

`若是整個vgs 要給vm使用, 就不用抽象成lv, 而是整包vgs當作一個lvm, 最後轉成storage pool, 之後vm要多少 直接跟切出固定大小, vm用這個volume開機`

lvm = vgs 直接打包, 沒有所謂lvm的物件 lvm=整包vgs的lv 概念 , 這邊也可以抽象成 storage pool, 這邊切出volume實際上底層就是切出一個lv

## storage pool manage

- lvm 資源無法直接被vm使用 用需要轉成storage pool

- lv 可轉可不轉, 一般會直接把lv格式化, 然後mount 當作file system使用, vm可直接存qcow2 在這個目錄, 但需自己管理檔案, 也支援把某目錄轉成storage pool, 主要是可以管理該目錄下的映像檔, 比較少用

- 確認來源

```bash
sudo vgs
#  VG        #PV #LV #SN Attr   VSize    VFree
#  anfu_vg     1   1   0 wz--n-   <1.82t    1.62t
#  ubuntu-vg   1   1   0 wz--n- <950.82g <850.82g

```

- 創建pool

```bash
sudo virsh pool-define-as lvm-pool lvm --source-name <vg_name> --target /dev/<vg_name>
```

lvm 設定 storage pool

```bash
sudo virsh pool-define lvm-pool.xml
```

`這個若lvm pool綁定vgs, 則是沒有容量設定的,`

vim lvm-pool.xml

這邊會需要path 基本上就是`/dev/<vg_name>`

lvm pool

```text
<pool type='logical'>
  <name>anfu_lvm_pool</name>
  <source>
    <name>anfu_vg</name>
    <format type='lvm2'/>
  </source>
  <target>
    <path>/dev/anfu_vg</path>
  </target>
</pool>

```

`lv格式化後 mount成一般目錄, 可將目錄轉成storage pool, 方便管理映像檔`

```text
<pool type='dir'>
  <name>default</name>
  <target>
    <path>/var/lib/libvirt/images</path>
  </target>
</pool>
```

```bash
sudo virsh pool-define lvm-pool.xml
#Pool anfu_lvm_pool defined from lvm-pool.xml

cat lvm-pool.xml
#<pool type='logical'>
#  <name>anfu_lvm_pool</name>
#  <source>
#    <name>anfu_vg</name>
#    <format type='lvm2'/>
#  </source>
#  <target>
#    <path>/dev/anfu_vg</path>
#  </target>
#</pool>

```

- 驗證

```bash
sudo virsh pool-list --all
```

- 啟動

```bash
sudo virsh pool-start anfu_lvm_pool
sudo virsh pool-autostart anfu_lvm_pool
```

- 檢視設定檔

```bash
sudo virsh pool-dumpxml anfu_lvm_pool
```

```bash
sudo lvdisplay
# list 所有lv 路徑
```
