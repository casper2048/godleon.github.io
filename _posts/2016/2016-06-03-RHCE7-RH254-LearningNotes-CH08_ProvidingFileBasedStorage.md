---
layout: post
title:  "[RHCE7] RH254 Chapter 8 Providing File-Based Storage Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 8 Providing File-Based Storage 留下的內容"
date: 2016-06-03 04:30:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

8.1 Exporting NFS File Systems
==============================

## 8.1.2 NFS exports

- 提供 NFS 服務需要先安裝 `nfs-utils` 套件

- 設定檔位於 `/etc/exports`(也可以在 `/etc/exports.d` 目錄中增加 `*.exports` 檔案)

- 無法同時使用 NFS & Samba 提供同一個目錄的檔案分享

- 設定分享可以使用 IP, IP/network, FQDN, Host Name .... 等等

- 比較常用的屬性包含：`ro`(real-only), `rw`(read-write), `no_root_squash`(client 連入後的操作維持 root 權限，預設會變成 `nfsnobody`)

設定範例：

```bash
# 允許 cliet(diskless.example.com) 以 root 的身分進行 read-write
/myshare  diskless.example.com(rw,no_root_squash)
```

## 8.1.3 Configuring an NFS export

以下簡單說明設定一個 NFS 服務的方式：(以設定 `/myshare` 目錄為例，`serverX` & `desktopX`)

### NFS server 端

```bash
# 啟動 nfs-server.service
$ sudo systemctl enable nfs-server.service
$ sudo systemctl start nfs-server.service

# 設定 NFS 分享，並套用設定
$ sudo mkdir /myshare
$ sudo echo "/myshare  desktopX(rw)" | sudo tee --append /etc/exports
$ sudo exportfs -r

# 設定防火牆
$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --reload
```

### NFS client 端

```bash
$ sudo mkdir /mnt/nfsexport
$ sudo mount serverX:/myshare /mnt/nfsexport
```

---------------------------------------------

Practice: Exporting NFS File Systems
====================================

## 目標

1. NFS server 分享 `/nfsshare` 目錄，提供 `read-write` 權限給 `nfsnobody`

2. NFS client 使用 `/mnt/nfsshare` 目錄掛載遠端目錄，重開機會自動連結

## 實作過程

### NFS server

```bash
# 新增目錄 & 變更權限
$ sudo mkdir /nfsshare
$ sudo chown nfsnobody /nfsshare

$ echo "/nfsshare  desktopX(rw)" | sudo tee /etc/exports.d/myshare.exports
$ sudo exportfs -r

$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --reload

$ sudo systemctl enable nfs-server.service
$ sudo systemctl restart nfs-server.service
```

### NFS client

```bash
# 掛載 remote NFS share folder
$ sudo mkdir /mnt/nfsshare
$ echo "server0:/nfsshare /mnt/nfsshare nfs defaults 0 0" | sudo tee --append /etc/fstab
$ sudo mount -a

# 查詢目前掛載狀況
$ sudo df -h
Filesystem         Size  Used Avail Use% Mounted on
......
server0:/nfsshare   10G  3.1G  7.0G  31% /mnt/nfsshare

# 驗證
$ echo "Hello NFS" | sudo tee /mnt/nfsshare/123
Hello NFS
$ cat /mnt/nfsshare/123
Hello NFS
```

---------------------------------------------

8.2 Protecting NFS Exports
==========================

## 8.2.1 Kerberos-enabled NFS exports

連接 remote NFS 是不需要經過驗證程序的，僅透過來源的 FQDN or IP address 來限制存取，為了補強這個部份，NFS server 一共提供了以下幾種方式加強檔案存取上的安全性：

- `none`: 所有人的存取都會轉成 **nfsnobody** 的身份執行

- `sys`: 使用系統帳號(此為預設值)，NFS server 會使用 NFS client 傳來的 UID & GID

- `krb5`: 使用 Kerberos V5 取代本地 UNIX UIDs 及 GIDs 進行身份認證

- `krb5i`: 使用 Kerberos V5 進行身份認證，資料完整性檢查，以防止數據被篡改

- `krb5p`: 使用 Kerberos V5 進行身份認證，資料完整性檢查及 NFS 傳輸加密，以防止數據被篡改，這是最安全的方式

以上可以選一個使用，也可以同時使用多個；而 NFS client 連接時就必須額外指定 `sec=method` 參數來符合安全設定的要求。

此外，NFS server 上還需要額外執行 `nfs-secure-server.service` system unit，且 NFS client 也需要額外執行 `nfs-secure.service` 來負責進行 Kerberos 相關的認証工作。

> 要設定 Kerberos 認証，最重要的是 `/etc/krb5.keytab` 是絕對不能少的!

## 8.2.2 Configuring a Kerberos-enabled NFS server

### NFS server

```bash
# 取得 krb5.keytab 檔案，並確認 SELinux context type
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/server0.keytab
$ ls -lZ /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab

# 啟動 nfs-secure-server.service
$ sudo systemctl enable nfs-secure-server.service
ln -s '/usr/lib/systemd/system/nfs-secure-server.service' '/etc/systemd/system/nfs.target.wants/nfs-secure-server.service'
$ sudo systemctl restart nfs-secure-server.service
$ sudo systemctl status nfs-secure-server.service
nfs-secure-server.service - Secure NFS Server
   Loaded: loaded (/usr/lib/systemd/system/nfs-secure-server.service; enabled)
   Active: active (running) since Wed 2016-06-01 05:06:44 JST; 6s ago
  Process: 1901 ExecStart=/usr/sbin/rpc.svcgssd $RPCSVCGSSDARGS (code=exited, status=0/SUCCESS)
 Main PID: 1904 (rpc.svcgssd)
   CGroup: /system.slice/nfs-secure-server.service
           └─1904 /usr/sbin/rpc.svcgssd

Jun 01 05:06:44 server0.example.com systemd[1]: Starting Secure NFS Server...
Jun 01 05:06:44 server0.example.com systemd[1]: Started Secure NFS Server.

# 設定 NFS share
$ sudo mkdir /securedexport
$ echo "/securedexport  *.example.com(sec=krb5p,rw)" | sudo tee --append /etc/exports
$ sudo exportfs -r

# 設定防火牆
$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --reload
```

### NFS client

```bash
# 取得 krb5.keytab 檔案，並確認 SELinux context type
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/desktop0.keytab
$ ls -lZ /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab

# 安裝 nfs-utils 套件
$ sudo yum -y install nfs-utils

# 啟動 nfs-secure.serivce
$ sudo systemctl enable nfs-secure.service
ln -s '/usr/lib/systemd/system/nfs-secure.service' '/etc/systemd/system/nfs.target.wants/nfs-secure.service'
$ sudo systemctl restart nfs-secure.service
$ sudo systemctl status nfs-secure.service
nfs-secure.service - Secure NFS
   Loaded: loaded (/usr/lib/systemd/system/nfs-secure.service; enabled)
   Active: active (running) since Wed 2016-06-01 05:16:18 JST; 6s ago
  Process: 1977 ExecStart=/usr/sbin/rpc.gssd $RPCGSSDARGS (code=exited, status=0/SUCCESS)
 Main PID: 1978 (rpc.gssd)
   CGroup: /system.slice/nfs-secure.service
           └─1978 /usr/sbin/rpc.gssd

Jun 01 05:16:18 desktop0.example.com systemd[1]: Started Secure NFS.

# 建立目錄並掛載 NFS remote shared folder
$ sudo mkdir /mnt/securedexport
$ sudo mount -o sec=krb5 server0:/securedexport /mnt/securedexport
```

## 8.2.3 SELinux and labeled NFS

### NFS server

為了讓 NFS server 可以支援 SELinux context，必須指定使用 `NFS 4.2`：

```bash
$ echo 'RPCNFSDARGS="-V 4.2"' | sudo tee --append /etc/sysconfig/nfs

$ sudo systemctl restart nfs-server
$ sudo systemctl restart nfs-secure-server
```

### NFS client

在 client side 的部分，掛載的設定同樣也要變更：

```bash
# 設定以 v4.2 掛載
$ sudo mount -o sec=krb5p,v4.2 serverX:/securedexport /mnt/securedexport
```

最後要驗證在 serverX 中設定的 SELinux context 會不會正確的帶到 NFS client，可直接建立一個新檔案進行驗證：

```bash
# NFS server 端
$ sudo touch /securedexport/selinux.txt
$ sudo chcon -t public_content_t /securedexport/selinux.txt

# NFS client 端
$ ls -lZ /mnt/securedexport
```

---------------------------------------------

Practice：Protecting NFS Exports (很重要)
========================================

## 目標

1. NFS server 分享目錄 `/securenfs`，並使用 `Kerberos` + `SELinux Label`

2. NFS client 使用 `/mnt/secureshare` 進行掛載

3. krb5 keytab 分別位於 `http://classroom.example.com/pub/keytabs/serverX.keytab` & `http://classroom.example.com/pub/keytabs/desktopX.keytab`

## 實作過程

### Prerequisite

需要在 server & desktop 端都執行 `lab nfskrb5 setup` 以預先建置相關環境

### NFS server

1、取得 krb5 keytab

```bash
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/server0.keytab

# 確認 SELinux context type
$ ls -Z /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab
```

2、啟用 SElinux label 功能 for NFS

```bash
$ echo 'RPCNFSDARGS="-V 4.2"' | sudo tee --append /etc/sysconfig/nfs
RPCNFSDARGS="-V 4.2"

$ sudo systemctl enable nfs-server.service
$ sudo systemctl restart nfs-server.service

$ sudo systemctl enable nfs-secure-server.service
$ sudo systemctl restart nfs-secure-server.service
```

3、設定 NFS share

```bash
$ sudo mkdir /securenfs
$ echo "/securenfs  desktop0(sec=krb5p,rw)" | sudo tee --append /etc/exports
$ sudo exportfs -rv
exporting desktop0.example.com:/securenfs

# 設定防火牆
$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --reload
```

### NFS client

1、取得 krb5 keytab

```bash
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/desktop0.keytab

# 確認 SELinux context type
$ ls -Z /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab
```

2、安裝相關套件 & 啟動 nfs-secure.service

```bash
$ sudo yum -y install nfs-utils

$ sudo systemctl enable nfs-secure.service
$ sudo systemctl restart nfs-secure.service
```

3、設定掛載遠端目錄

```bash
$ sudo mkdir /mnt/secureshare
$ echo "server0:/securenfs  /mnt/secureshare  nfs  defaults,v4.2,sec=krb5p  0  0" | sudo tee --append /etc/fstab
$ sudo mount -a
```

### 驗證

#### server

```bash
$ echo "Hello World" | sudo tee /securenfs/testfile.txt
$ sudo chcon -t public_content_t /securenfs/testfile.txt
$ sudo chown ldapuser0 /securenfs/testfile.txt
$ ls -Z /securenfs/testfile.txt
-rw-r--r--. ldapuser0 root unconfined_u:object_r:public_content_t:s0 /securenfs/testfile.txt
```

#### client

以 ldapuserX 的身份登入後：

```bash
# client 端同樣可以看到正確的 context type
$ ls -Z /mnt/secureshare/testfile.txt
-rw-r--r--. ldapuser0 root unconfined_u:object_r:public_content_t:s0 /mnt/secureshare/testfile.txt

# 寫入測試
$ echo "I can write" >> /mnt/secureshare/testfile.txt
$ ls -Z /mnt/secureshare/testfile.txt
-rw-r--r--. ldapuser0 root unconfined_u:object_r:public_content_t:s0 /mnt/secureshare/testfile.txt
$ cat /mnt/secureshare/testfile.txt
Hello World
I can write
```

---------------------------------------------

8.3 Providing SMB File Shares
=============================

SAMBA 4.0 之後，就可以完整取代 AD 的功能了!

## 8.3.2 SMB file sharing with Samba

要提供 samba 檔案分享，有以下幾個步驟需要設定

### 1、安裝 samba

```bash
$ sudo yum -y install samba
```

### 2、分享目錄的設定

首先當然要建立分享目錄：`sudo mkdir /sharedpath`

接著是目錄權限的設定，這包含兩個部份：

#### 一般使用者權限

此部份權限的設定取決於 client 會用那一位使用者身份進行存取，要設定相對應的權限才可正常使用(**至少要可以 read 才可以 mount**)

#### SELinux contexts & Booleans

分享目錄中所有的內容的 SELinux context type 都需要設定成 `samba_share_t`，samba process 才可以正常的存取檔案

比較快速的方法是設定 default context type 給分享目錄：(就可以透過 `restorecon` 快速設定 context type)

```bash
$ sudo semanage fcontext -a -t samba_share_t '/sharedpath(/.*)?'

# 一次將分享目錄下的的資料全部設定為 samba_share_t
$ sudo restorecon -vvFR /sharedpath
```

此外，Samba 同樣也可以接受以下兩個 context type：

- `public_content_t`(**read-only**)

- `public_content_rw_t`(**read-write**，但需要搭配 `smbd_anon_write` SELinux Boolean 選項的開啟)

### 3、設定 samba (/etc/samba/smb.conf)

`/etc/samba/smb.conf` 是 smaba 的主要設定檔，包含了好幾個 section：(設定完後可透過 `testparm` 指令驗證是語法上是否有錯誤)

#### [global]

- `workgroup`：等同 windows 的工作群組

- `security`：控制 client 如何與 samba server 進行認証，例如：`security = user`(使用本機 samba 帳號)

- `hosts allow`：允許的來源 client，不指定就表示所有來源的 client 都可以連 (也可以在獨立的 section 中設定)，以下設定都是合法的：(**設定多個時可用空白隔開**)
  - `172.25.0.0/24`
  - `172.25.0.0/255.255.255.0`
  - `172.25.0.` (等於 **172.25.0.0/24**)
  - `172.25.` (等於 **172.25.0.0/16**)
  - `2001:db8:0:1::/64`
  - `desktopX.example.com`
  - `.example.com` (指定 domain)

#### file share sections

samba 可以一次設定多個檔案分享的 section，以下是比較重要的設定參數：

- `path`：分享的路徑

- `writable = [yes|no]`：若 yes，表示所有認證過的使用者都有權限 read-write；但若是 no，則僅有在 `write list` 設定中的使用者才可以 read-write(**用逗號隔開，若前面帶上 "@" 表示指定 group**)

- `write list`：若 `writable = no`，則僅有 write list 設定的使用者才可以寫入

- `valid users`：指定那些帳號可以掛載，沒指定就不允許(**用逗號隔開，若前面帶上 "@" 表示指定 group**)

- `read only = [yes|no]`：是否唯讀("**read only = no**" 效果等同於 "**writable = yes**")

- `browseralbe = [yes|no]`：是否隱藏分享目錄(隱藏的話，網路芳鄰功能找不到)

#### home section

分享使用者家目錄出來，給各自的使用者掛載使用

> 必須設定 `setsebool -P samba_enable_home_dirs=on` 才可以啟用分享家目錄功能

### 4、設定相關的 Linux 使用者

1. 若要建立 samba-only 的使用者，建議將 login shell 設定為 **/sbin/nologin**：`sudo useradd -s /sbin/nologin [USER_NAME]`

2. samba 密碼與系統密碼其實是分開的，因此要額外安裝設定 samba 密碼的套件：`sudo yum -y install samba-client`

以下是 `smbpasswd` 的簡單使用說明：

```bash
# 新增 fred 為 samba user 並設定密碼
$ sudo smbpasswd -a fred

# 移除 samba user
$ sudo smbpasswd -x fred
```

### 5、啟動 samba & 開啟防火牆

啟動 samba 服務需要啟動 `smb`(**TCP/445** & **TCP/139**) & `nmb`(**UDP/137** & **UDP/138**) 兩個 daemon：

```bash
$ sudo systemctl enable smb.service nmb.service
$ sudo systemctl restart smb.service nmb.service
```

> 若有修改 /etc/samba/smb.conf，可執行 `sudo systemctl reload smb.service nmb.service` 快速套用變更設定

開啟防火牆設定：

```bash
$ sudo firewall-cmd --permanent --add-service=samba
$ sudo firewall-cmd --reload
```

### 6、client 驗證(掛載 samba 分享)

1. 首先必須確認安裝 `cifs-utils` 套件：`sudo yum -y install cifs-utils`

2. 需使用 samba 帳號密碼(**同系統帳號，但非系統密碼**) 進行掛載：(假設 samba 帳號密碼皆為 `student`)

```bash
$ sudo bash -c 'cat << EOF > /root/student.smb
username=student
password=student
EOF'

$ sudo mkdir /mnt/myshare
$ echo "//serverX/myshare  /mnt/myshare  cifs  credentials=/root/student.smb  0 0" | sudo tee --append /etc/fstab
$ sudo mount -a
```

> 若要直接掛載也可以使用 `sudo mount -o credentials=/root/student.smb //serverX/myshare /mnt/myshare`

---------------------------------------------

Practice: Providing SMB File Shares
===================================

## 目標

1. 建立 samba 分享 `/smbshare`，設定分享名稱為 `smbshare`，工作群組為 `mycompany`

2. `marketing` 群組有 read-write 的權限；非 `marketing` 群組僅有唯讀權限

3. 建立可存取 samba-only 的使用者 `brian`，密碼 `redhat`，並隸屬於 `marketing` 群組

4. 建立可存取 samba-only 的使用者 `rob`，密碼 `redhat`，但不屬於 `marketing` 群組

## 實作過程

### server

```bash
# 安裝 samba 套件
$ sudo yum -y install samba samba-client

# 新增相關使用者
$ sudo groupadd -r marketing
$ sudo useradd -s /sbin/nologin -G marketing brian
$ sudo useradd -s /sbin/nologin rob
$ sudo smbpasswd -a brian
$ sudo smbpasswd -a rob

# 設定本地端使用者權限
$ sudo mkdir /smbshare
$ sudo setfacl -R -m g:marketing:rwX /smbshare
$ sudo setfact -R -m d:g:marketing:rwX /smbshare

# 設定 default SELinux context
$ sudo semanage fcontext -a -t samba_share_t '/smbshare(/.*)?'
$ sudo restorecon -vvRF /smbshare

# 編輯設定檔
$ sudo vim /etc/samba/smb.conf
[global]
...
workgroup = mycompany
security = user
passdb backend = tdbsam
...
[smbshare]
path = /smbshare
valid users = @marketing,rob
write list = @marketing

# 啟動服務
$ sudo systemctl enable smb.service nmb.service
$ sudo systemctl restart smb.service nmb.service

# 開啟防火牆
$ sudo firewall-cmd --permanent --add-service=samba
$ sudo firewall-cmd --reload
```

### client

```bash
# 安裝 samba 相關套件
$ sudo yum -y install cifs-utils

# 建立掛載目錄
$ sudo mkdir /mnt/{brian,rob}

# 設定 samba credentials
$ sudo bash -c 'cat << EOF > /root/brian.smb
username=brian
password=redhat
EOF'
$ sudo bash -c 'cat << EOF > /root/rob.smb
username=rob
password=redhat
EOF'

# 設定掛載
$ echo "//server0/smbshare  /mnt/brian  cifs  credentials=/root/brian.smb  0  0" | sudo tee --append /etc/fstab
$ echo "//server0/smbshare  /mnt/rob  cifs  credentials=/root/rob.smb  0  0" | sudo tee --append /etc/fstab
$ sudo mount -a

# 驗證
$ sudo df -h
Filesystem          Size  Used Avail Use% Mounted on
......
//server0/smbshare   10G  3.1G  7.0G  31% /mnt/brian
//server0/smbshare   10G  3.1G  7.0G  31% /mnt/rob

# brian 的目錄可以寫入資料
$ sudo touch /mnt/brian/123
# rob 的目錄無法寫入資料
$ sudo touch /mnt/rob/123
touch: cannot touch '/mnt/rob/123': Permission denied
```

---------------------------------------------

8.4 Performing a Multiuser SMB Mount
====================================

RHEL7 提供了 `cifscreds` 工具(需安裝 `cifs-utils` 套件)，目的是為了同時讓多個使用者使用 samba share session(應該也可以用來驗證 multiple client 設定是否正確)，以下有幾點需要注意：

1. 僅有在該掛載的 session 中有效(**session keyring** 的概念)

2. client 掛載 samba 分享時需要加上 `multiuser` & `sec=ntlmssp` 兩個選項設定才會生效

3. 使用 cifscreds 時若沒指定使用者，會預設默認為 client 系統使用者；若系統不存在該使用者，則要額外加上 `-u USER_NAME` 進行設定

4. cifscreds 工具一共有四個功能：
  - `add`：新增 samba 認證到 session keyring 中
  - `udpate`：更新 session keyring
  - `clear`：指定清除 session keyring 中特定使用者
  - `clearall`：清除 session keyring 中所有使用者

## 使用示範

使用情境：

- `root` 已經掛載 `//serverX/myshare` 到目錄 `/mnt/multiuser`

- 使用者 `frank`(read-write) & `jane`(read-only) 想透過 cifscreds 使用這個 samba share session 來存取資料

```bash
# 驗證 frank 的權限，可正常 read-write
[frank@desktopX ~]$ cifscreds add serverX
Password: redhat
[frank@desktopX ~]$ echo "Frank was here" > /mnt/multiuser/frank.txt


# 驗證 jane 的權限，僅有 read-only
[jane@desktopX ~]$ cifscreds add serverX
Password: redhat
[jane@desktopX ~]$ echo "Jane was here" > /mnt/multiuser/jane.txt   #發生 permission denied 錯誤
```

---------------------------------------------

Practice: Performing a Multiuser SMB Mount
==========================================

## 目標

- desktop0 永久掛載 `//server0/smbshare` 到 `/mnt/multiuser` 目錄下，**以 multiple user 的型式掛載**

- 建立 `/root/smb-multiuser.txt` 作為 samba credential 檔案，帳號 = `brian`，密碼 = `redhat`

- 在 server0 中有另一個 samba user `rob`(密碼為 `redhat`) 存在，在 desktop0 中也有相對應的 system user，驗證 rob 僅有 read-only 的權限

## 實作過程

```bash
[student@desktopX ~]$ sudo yum -y install cifs-utils

[student@desktopX ~]$ sudo bash -c 'cat <<EOF > /root/smb-multiuser.txt
username=brian
password=redhat
EOF'

[student@desktopX ~]$ sudo mkdir /mnt/multiuser
[student@desktopX ~]$ echo "//server0/smbshare  /mnt/multiuser  cifs  credentials=/root/smb-multiuser.txt,multiuser,sec=ntlmssp  0  0" | sudo tee --append /etc/fstab
[student@desktopX ~]$ sudo mount -a
[student@desktopX ~]$ sudo df -h
Filesystem          Size  Used Avail Use% Mounted on
.....
//server0/smbshare   10G  3.1G  7.0G  31% /mnt/multiuser

# 切換到使用者 brian 做測試
$ su - brian
Password: redhat

[brian@desktop0 ~]$ cifscreds add server0
Password: redhat

[brian@desktop0 ~]$ echo "Hello World" > /mnt/multiuser/brian.txt
[brian@desktop0 ~]$ cat /mnt/multiuser/brian.txt
Hello World

# 切換到使用者 rob 做測試
$ su - rob
Password: redhat

[rob@desktop0 ~]$ cifscreds add server0
Password: redhat

# 僅有 read-only 權限
[rob@desktop0 ~]$ echo "Hello World" > /mnt/multiuser/rob.txt
-bash: /mnt/multiuser/rob.txt: Permission denied

[rob@desktop0 ~]$ cat /mnt/multiuser/brian.txt
Hello World
```

---------------------------------------------

Lab: Providing File-Based Storage
=================================

## 目標

### NFS share

1. 在 serverX 上設定 NFS 分享服務，分享 `/krbnfs` 目錄，並搭配 `krb5p` 安全性認証 & SELinux Label

2. Kerberos Keytab 檔案分別在 `http://classroom.example.com/pub/keytabs/server0.keytab` & `http://classroom.example.com/pub/keytabs/desktop0.keytab`

3. 允許 `desktopX` 對 NFS 分享進行 `read-write` 的存取

4. desktop0 掛載 NFS 分享在本地端的 `/mnt/securespace` 目錄

### Samba share

1. 在 serverX 上設定 SAMBA 分享服務，分享 `/sambaspace` 目錄，分享名稱為 `smbspace`，位於 `salesdep` 工作群組

2. 附屬群組為 `sales` 的使用者擁有 `read-write` 的權限

3. 不屬於 `sales` 群組的使用者僅有 `read-only` 的權限

4. 建立一個 samba-only 的使用者 `frank`，屬於 `sales` 群組，密碼為 `redhat`

5. 建立一個 samba-only 的使用者 `martin`，不屬於 `sales` 群組，密碼為 `redhat`

6. 在 desktop0 的本地目錄 `/mnt/salesshare` 上，以 `multiuser` 的模式掛載 samba 分享

7. 掛載使用 `/root/smb-multiuser.txt` 認証檔，並搭配使用者 `frank` 的身份

## 實作過程

### NFS share

#### 1. NFS server

```bash
# 取得 kerberos keytab & 確認 SELinux context type
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/server0.keytab
$ ls -Z /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab

# 新增 SELinux label for NFS 設定
$ sudo sed -i 's/^RPCNFSDARGS=.*/RPCNFSDARGS="-V 4.2"/g' /etc/sysconfig/nfs

# 啟用 nfs-secure-server.service & nfs-server.service
$ sudo systemctl enable nfs-secure-server.service nfs-server.service
$ sudo systemctl restart nfs-secure-server.service nfs-server.service

# 分享 NFS 目錄
$ sudo mkdir /krbnfs
$ echo "/krbnfs  desktop0(sec=krb5p,rw)" | sudo tee --append /etc/exports
$ sudo exportfs -rv

# 開啟防火牆 for NFS
$ sudo firewall-cmd --permanent --add-service=nfs
$ sudo firewall-cmd --reload
```

#### 2. NFS client

```bash
# 取得 kerberos keytab & 確認 SELinux context type
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/desktop0.keytab
$ ls -Z /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab

# 安裝 nfs client 相關套件
$ sudo yum -y install nfs-utils

# 啟用 nfs-secure.service
$ sudo systemctl enable nfs-secure.service
$ sudo systemctl restart nfs-secure.service
$ sudo systemctl status nfs-secure.service

# 掛載 & 驗證
$ sudo mkdir /mnt/securespace
$ echo "server0:/krbnfs  /mnt/securespace  nfs  defaults,v4.2,sec=krb5p  0  0" | sudo tee --append /etc/fstab
$ sudo mount -a
$ sudo df -h
Filesystem       Size  Used Avail Use% Mounted on
....
server0:/krbnfs   10G  3.1G  7.0G  31% /mnt/securespace
```

### Samba share

#### 1. Samba server

```bash
$ sudo yum -y install samba samba-client

# 建立 samba-only 使用者 & 設定密碼
$ sudo groupadd sales
$ sudo useradd -s /sbin/nologin -G sales frank
$ sudo useradd -s /sbin/nologin martin
$ echo "frank:redhat" | sudo chpasswd
$ echo "martin:redhat" | sudo chpasswd
$ (echo redhat; echo redhat) | sudo smbpasswd -a frank
$ (echo redhat; echo redhat) | sudo smbpasswd -a martin
$ sudo pdbedit -L
frank:1001:
martin:1002:

# 建立目錄 & 設定 SELinux context
$ sudo mkdir /sambaspace
$ sudo setfacl -R -m g:sales:rwX /sambaspace
$ sudo setfacl -R -m d:g:sales:rwX /sambaspace
$ sudo semanage fcontext -a -t samba_share_t '/sambaspace(/.*)?'
$ sudo restorecon -vvRF /sambaspace
$ sudo getfacl /sambaspace
getfacl: Removing leading '/' from absolute path names
# file: sambaspace
# owner: root
# group: root
user::rwx
group::r-x
group:sales:rwx
mask::rwx
other::r-x
default:user::rwx
default:group::r-x
default:group:sales:rwx
default:mask::rwx
default:other::r-x
$ ls -ldZ /sambaspace
drwxrwxr-x+ root root system_u:object_r:samba_share_t:s0 /sambaspace

# 編輯 /etc/samba/smb.conf
$ sudo vi /etc/samba/smb.conf
$ sudo cat /etc/samba/smb.conf
[global]
workgroup = salesdep
.....
security = user
passdb backend = tdbsam
[smbspace]
path = /sambaspace
write list = @sales

# 啟動 smb.service & nmb.service 服務
$ sudo systemctl enable smb.service nmb.service
$ sudo systemctl restart smb.service nmb.service
$ sudo systemctl status smb.service nmb.service

# 啟用防火牆
$ sudo firewall-cmd --permanent --add-service=samba
$ sudo firewall-cmd --reload
```

#### 2. Samba client

```bash
# 安裝 cifs 相關套件
$ sudo yum -y install cifs-utils

$ sudo bash -c 'cat << EOF > /root/smb-multiuser.txt
username=frank
password=redhat
EOF'

# 設定掛載目錄
$ sudo mkdir /mnt/salesshare
$ echo "//server0/smbspace  /mnt/salesshare  cifs  credentials=/root/smb-multiuser.txt,multiuser,sec=ntlmssp  0  0" | sudo tee --append /etc/fstab
$ sudo mount -a
$ sudo df -h
Filesystem          Size  Used Avail Use% Mounted on
....
server0:/krbnfs      10G  3.1G  7.0G  31% /mnt/securespace
//server0/smbspace   10G  3.1G  7.0G  31% /mnt/salesshare

# 驗證
$ sudo groupadd sales
$ sudo useradd -G sales frank
$ sudo useradd martin
$ echo "frank:redhat" | sudo chpasswd
$ echo "martin:redhat" | sudo chpasswd
$ su - frank
Password: redhat
$ cifscreds add server0
Password: redhat
[frank@desktop0 ~]$ echo "Hello I am Frank" > /mnt/salesshare/frank.txt
[frank@desktop0 ~]$ cat /mnt/salesshare/frank.txt
Hello I am Frank
[frank@desktop0 ~]$ exit
logout
$ su - martin
Password: redhat
[martin@desktop0 ~]$ cifscreds add server0
Password: redhat
[martin@desktop0 ~]$ echo "Hello I am Martin" > /mnt/salesshare/martin.txt
-bash: /mnt/salesshare/martin.txt: Permission denied
[martin@desktop0 ~]$ cat /mnt/salesshare/frank.txt
Hello I am Frank
```

---------------------------------------------

參考資料
=======

- [NFS Server 架設](http://dywang.csie.cyut.edu.tw/dywang/rhce7/node55.html)
