---
layout: post
title:  "[RHCE7] RH254 Chapter 10 Providing Apache HTTPD Web Service Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 10 Providing Apache HTTPD Web Service 留下的內容"
date: 2016-06-07 04:25:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

10.1 Configuring Apache HTTPD
=============================

## 10.1.1 Introduction to Apache HTTPD

- **HTTP** 使用 `80/TCP`；**HTTPS** 使用 `443/TCP`

- groupinstall `web-server` 會同時安裝 `httpd` & `httpd-manual`

- `httpd-tools` 套件中包含了一個進行 benchmark & 壓力測試的工具，稱為 `ab`

## 10.1.2 Basic Apache HTTPD configuration

web server 的設定檔位於 `/etc/httpd/conf/httpd.conf`：

```bash
ServerRoot "/etc/httpd" # Aapche 設定檔的主要目錄
Listen 80
Include conf.modules.d/*.conf   # 可以把設定檔模組化放在此目錄
User apache   # httpd daemon 會以 apache:apache 的身分執行
Group apache
ServerAdmin root@localhost  # 網頁顯示錯誤時，頁面下方的聯絡人
<Directory /> # 指的是硬碟的 "/" 目錄
    AllowOverride none
    Require all denied  # 拒絕存取，回傳 HTTP/1.1 406 Forbidden 訊息
</Directory>
DocumentRoot "/var/www/html"  # web server 網頁的根目錄
<Directory "/var/www">
    AllowOverride None
    # Allow open access:
    Require all granted  # 允許存取
</Directory>
<Directory "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
<IfModule dir_module>   # ifmodule 載入，裡面的參數才會生效
    DirectoryIndex index.html
</IfModule>
<Files ".ht*">  # 對於 ".htaccess" & ".htpasswd" 這一類的敏感資料不提供存取
    Require all denied
</Files>
ErrorLog "logs/error_log"   # 相對路徑，需要與上方的 "ServerRoot" 參數搭配
LogLevel warn
<IfModule log_config_module>
    # 自訂 log 格式 (其中 "combined" & "common" 為代表 log 格式的變數)
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%h %l %u %t \"%r\" %>s %b" common
    <IfModule logio_module>
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
    </IfModule>

    CustomLog "logs/access_log" combined  # 以 "combined" 所定義的 log 格式進行儲存
</IfModule>
<IfModule alias_module>
    ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"
</IfModule>
<Directory "/var/www/cgi-bin">
    AllowOverride None
    Options None
    Require all granted
</Directory>
<IfModule mime_module>
    TypesConfig /etc/mime.types
    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz
    AddType text/html .shtml
    AddOutputFilter INCLUDES .shtml
</IfModule>
AddDefaultCharset UTF-8   # 網頁編碼
<IfModule mime_magic_module>
    MIMEMagicFile conf/magic
</IfModule>
EnableSendfile on
IncludeOptional conf.d/*.conf   # 其他設定檔可以放到此目錄下(副檔名必須是 ".conf")
```

## 10.1.3 Starting Aapche HTTPD

啟動 httpd：

```bash
$ sudo systemctl enable httpd.service
$ sudo systemctl start httpd.service
$ sudo firewall-cmd --permanent --add-service=http --add-service=httpd
$ sudo firewall-cmd --reload
```

預設 SELinux 僅允許 httpd 與特定的 port 進行連結，可透過 `sudo semanage port -l | grep '^http'` 查詢，若要使用其他自訂的 port 提供 httpd 服務，就要修改 port context 為 `http_port_t`

### Using an alternative document root

若要修改設定檔中的 `DocumentRoot` 設定到其他目錄，有幾個重點需要注意：

1. 目錄需要提供 `apache` user & group 讀取的權限，以 `/var/www/html` 來說，權限為 `root:root 755`

2. SELinux context type 需要設定為 `httpd_sys_content_t`，httpd daemon 才能讀取目錄中的檔案內容：`sudo semanage fcontext -a -t httpd_sys_content_t '/new/location(/.*)?'`

> 詳細設定可以參考 httpd_selinux(8)

### Allowing write access to a DocumentRoot                                                                                                                                                                                                                                                                                                               
預設僅有 root 可以對 DocumentRoot 指定的目錄進行寫入，但假設如果要提供 `webmasters` 群組也有寫入權限呢?

可以透過 ACL 來達成：

```bash
$ sudo setfacl -R -m g:webmasters:rwX /var/www/html
$ sudo setfacl -R -m d:g:webmaster:rwX /var/www/html
```

可以透過設定 sticky bit 的方式：

```bash
$ sudo mkdir -p -m 2775 /new/docroot
$ sudo chgroup webmasters /new/docroot
```

----------------------------------------------------

Practice: Configuring a Web Server
==================================

## 目標

1. 安裝 Apache web server，啟動 HTTP 服務

2. 首頁顯示 `Hello Class!`

3. 可以從 `http://serverX.example.com/manual` 檢視文件資訊

## 實作過程

```bash
$ sudo yum -y groupinstall web-server

$ sudo echo "Hello Class!" | sudo tee /var/www/html/index.html
$ sudo echo "Hello Class\!" | sudo tee /var/www/html/index.html

$ sudo systemctl enable httpd.service
$ sudo systemctl start httpd.service

$ sudo firewall-cmd --permanent --add-service=http
$ sudo firewall-cmd --reload
```

----------------------------------------------------

10.2 Configuring & Troubleshooting Virtual Hosts
================================================

virtual host 的功能是可以讓同一台 web server 以不同的 ip or hostname 為基礎，同時對多個 domain 提供不同的內容

## 10.2.2 Configuring virtual hosts

virtual host 的設定是使用 `<VirtualHost>` 區段來進行設定，可以不修改預設設定檔，直接編輯新檔案放到 `/etc/httpd/conf.d` 目錄下

假設有個檔案 **/etc/httpd/conf.d/site1.conf** 如下：

```bash
<Directory "/src/site1/www">  # 設定目錄權限
    Require all granted
    AllowOverride None
</Directory>

# 針對三部主機的 DocumentRoot 進行定！
<VirtualHost 192.168.0.1:80>  # 若 remote client 使用網址 http://192.168.0.1:80 就會從這個設定提供服務
    ServerName    site1.example.com # 用在 named-bsaed virtual hosting
    DocumentRoot  /src/site1/www    # 網站根目錄，僅有在此 virtual host 中有效
    ServerAdmin   webmaster@site1.example.com
    ErrorLog      "logs/site1_error_log"  # error log 存放位置
    CustomLog     "logs/site1_access_log"   combined  # access 存放位置 & 格式
</VirtualHost>
```

> 若使用 `<VirtualHost *:80>` 表示會監聽 **80/TCP**，會讓主要的設定失效(若是有的話)，另外 `_default` = `*`

> 當有多個 .conf 設定檔存在時，會依照數字 -> 字母順序進行載入，若要設定優先被載入，可以用 `00-default.conf` 的方式命名

## 10.2.3 Troubleshooting virtual hosts

設定 virtual host 遇到問題時，可用以下方式嘗試排除：

- 為每個 virtual host 設定不同的 DocumentRoot，透過網頁顯示內容來判別

- 為每個 virtual host 設定不同的 error/access log

- 檢查一下 conf 設定檔的載入順序是否造成問題

- 一個一個關閉 virtual host 找原因

- 使用 `sudo journalctl UNIT=httpd.service` 檢視 httpd.service 相關的 log 紀錄

----------------------------------------------------

Practice: Configuring a Vurtual Host
====================================

## 目標

1. 在 serverX 中啟動 web server

2. 從 `wwwX.example.com` 連進來的 client，顯示 `/srv/wwwX.example.com/www` 的內容

3. 從其他 domain 連進來的 client，顯示 `/srv/default/www` 的內容

## 實作過程

```bash
$ sudo yum -y groupinstall web-server

# 新增目錄 & index.html
$ sudo mkdir -p /srv/{www0.example.com,default}/www
$ echo "www.example.com" | sudo tee /srv/www0.example.com/www/index.html
$ echo "default" | sudo tee /srv/default/www/index.html

# 設定 SELinux context type
$ sudo semanage fcontext -a -t httpd_sys_content_t '/srv(/.*)?'
$ sudo restorecon -vvRF /srv/
restorecon reset /srv/www0.example.com context unconfined_u:object_r:var_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /srv/www0.example.com/www context unconfined_u:object_r:var_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /srv/default context unconfined_u:object_r:var_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /srv/default/www context unconfined_u:object_r:var_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /srv/www0.example.com/www/index.html context unconfined_u:object_r:httpd_sys_content_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /srv/default/www/index.html context unconfined_u:object_r:httpd_sys_content_t:s0->system_u:object_r:httpd_sys_content_t:s0

# 設定 virtual host
$ sudo bash -c 'cat <<EOF > /etc/httpd/conf.d/00-www0.example.com.conf
<VirtualHost *:80>
  ServerName www0.example.com
  DocumentRoot /srv/www0.example.com/www
  CustomLog "logs/www0.example.com-vhost.log" combined
</VirtualHost>
<Directory /srv/www0.example.com/www>
  require all granted
</Directory>
EOF'
$ sudo bash -c 'cat <<EOF > /etc/httpd/conf.d/00-default.conf
<VirtualHost _default_:80>
  DocumentRoot /srv/default/www
  CustomLog "logs/default-vhost.log" combined
</VirtualHost>
<Directory /srv/default/www>
  require all granted
</Directory>
EOF'

# 啟動 httpd.service & 防火牆
$ sudo systemctl enable httpd.service
$ sudo systemctl restart httpd.service
$ sudo firewall-cmd --permanent --add-service=http
$ sudo firewall-cmd --reload
```

----------------------------------------------------

10.3 Configuring HTTPS
======================

## 10.3.1 Transport Layer Security

關於 TLS，可以參考以下資料：

- [SSL/TLS協議運行機制的概述 - 台灣 MakeHub](https://makehub.tw/communication/http/ssl-tls)

- [聊聊HTTPS和SSL/TLS協議 - 壹讀](https://read01.com/677n5A.html)

## 10.3.2 Configuring TLS certificates

要設定一個提供 TLS 加密的 virtual host，有以下幾個步驟需要完成：

1. 取得簽核過的憑證(certificate)

2. 安裝 Apache 擴充模組來支援 TLS

3. 使用步驟一取得的憑證，進行 virtual host 的 TLS 相關設定

### 1、取得憑證

先不考慮如何產生憑證，假設憑證已經存在，應該會有那些檔案?

1. `/etc/pki/tls/private/<FQDN>.key`：private key，必須要最高權限(`root:root 0600/0400`)來保護，並搭配 SELinux content type `cert_t`，此檔案絕對不能外流

2. `/etc/pki/tls/certs/<FQDN>.crt`：public key，由 CA 所發，與 private key 搭配用來加解密 web server & client 之間的通訊，儲存權限為 `root:root 0644`，並搭配 SELinux content type `cert_t`

3. `/etc/pki/tls/certs/<CA_FQDN>.crt`：用來提供中繼 CA 資訊的憑證

### 2、安裝 Apache 擴充模組

安裝 `mod_ssl` 套件：`sudo yum -y install mod_ssl`

安裝完後，就會 listen 443/TCP，且會有一個預設設定檔存在於 `/etc/httpd/conf.d/ssl.conf`

### 3、設定 virtual host TLS

```bash
$ sudo grep '^[^#]' /etc/httpd/conf.d/ssl.conf
Listen 443 https  # listen 443/TCP
SSLPassPhraseDialog exec:/usr/libexec/httpd-ssl-pass-dialog # 詢問 certificate 密碼用的程式
SSLSessionCache         shmcb:/run/httpd/sslcache(512000)
SSLSessionCacheTimeout  300
SSLRandomSeed startup file:/dev/urandom  256
SSLRandomSeed connect builtin
SSLCryptoDevice builtin
<VirtualHost _default_:443>   # 指定此 virtial host 處理 443/TCP 的 request
  ErrorLog logs/ssl_error_log
  TransferLog logs/ssl_access_log
  LogLevel warn
  SSLEngine on  # 指定開啟 TLS
  SSLProtocol all -SSLv2 -SSLv3   # 指定 server & client 溝通用的協定，去除 SSLv2 & SSLv3 (這兩個較不安全)
  SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5  # 指定加密方式(指定不要用 aNull & MD5)
  SSLCertificateFile /etc/pki/tls/certs/localhost.crt   # 被 CA 簽核過的 certificate
  SSLCertificateKeyFile /etc/pki/tls/private/localhost.key  # private key
  SSLCertificateChainFile /etc/pki/tls/certs/example-ca.crt # 中繼 CA 的 certificate
  <Files ~ "\.(cgi|shtml|phtml|php3?)$">
      SSLOptions +StdEnvVars
  </Files>
  <Directory "/var/www/cgi-bin">
      SSLOptions +StdEnvVars
  </Directory>
  BrowserMatch "MSIE [2-5]" \
           nokeepalive ssl-unclean-shutdown \
           downgrade-1.0 force-response-1.0
  CustomLog logs/ssl_request_log \
            "%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \"%r\" %b"
</VirtualHost>
```

----------------------------------------------------

Practice: Configuring a TLS-enabled Virtual Host
================================================

## 目標

1. 設定兩個 virtual host，分別為服務 `https://www0.example.com`(`/srv/www0/www`) & `https://webapp0.example.com`(`/src/webapp0/www`)

2. 都必須支援 TLS

3. **https://wwwX.example.com** 的憑證相關檔案位於 `http://classroom.example.com/pub/tls/certs/www0.crt` & `http://classroom.example.com/pub/tls/private/www0.key`

4. **https://webapp0.example.com** 的憑證相關檔案位於 `http://classroom.example.com/pub/tls/certs/webapp0.crt` & `http://classroom.example.com/pub/tls/private/webapp0.key`

5. 兩個 virtaul host 共用的 CA 中繼憑證位於 `http://classroom.example.com/pub/example-ca.crt`

## 實作過程

```bash
# 包含連 mod_ssl 會一起安裝
$ sudo yum -y groupinstall web-server

# 新增 virtual host 所會使用的目錄 & 首頁檔案
$ sudo mkdir -p /srv/{www0,webapp0}/www
$ echo "www0.example.com" | sudo tee /srv/www0/www/index.html
$ echo "webapp0.example.com" | sudo tee /srv/webapp0/www/index.html

# 設定 SELinux content type
$ sudo semanage fcontext -a -t httpd_sys_content_t '/srv(/.*)?'
$ sudo restorecon -vvRF /srv/

# 取得 www0.example.com 的憑證相關檔案
$ sudo wget -O /etc/pki/tls/certs/www0.crt http://classroom.example.com/pub/tls/certs/www0.crt
$ sudo wget -O /etc/pki/tls/private/www0.key http://classroom.example.com/pub/tls/private/www0.key
$ sudo chmod 0400 /etc/pki/tls/private/www0.key
$ ls -Z /etc/pki/tls/private/www0.key /etc/pki/tls/certs/www0.crt
-r--------. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/private/www0.key
-rw-r--r--. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/certs/www0.crt

# 取得 webapp0.example.com 的憑證相關檔案
$ sudo wget -O /etc/pki/tls/certs/webapp0.crt http://classroom.example.com/pub/tls/certs/webapp0.crt
$ sudo wget -O /etc/pki/tls/private/webapp0.key http://classroom.example.com/pub/tls/private/webapp0.key
$ sudo chmod 0400 /etc/pki/tls/private/webapp0.key
$ sudo ls -Z /etc/pki/tls/private/webapp0.key /etc/pki/tls/certs/webapp0.crt
-rw-r--r--. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/certs/webapp0.crt
-r--------. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/private/webapp0.key

# 取得中繼 CA 檔案
$ sudo wget -O /etc/pki/tls/certs/example-ca.crt http://classroom.example.com/pub/example-ca.crt
$ sudo ls -Z /etc/pki/tls/certs/example-ca.crt
-rw-r--r--. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/certs/example-ca.crt

# 設定 virtual host (www0.example.com)
$ sudo bash -c 'cat <<EOF > /etc/httpd/conf.d/ssl-www0.conf
<VirtualHost *:443>
  ServerName www0.example.com
  DocumentRoot /srv/www0/www
  ErrorLog logs/ssl_www0_error_log
  TransferLog logs/ssl_www0_access_log
  LogLevel warn
  SSLEngine on
  SSLProtocol all -SSLv2 -SSLv3
  SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5
  SSLCertificateFile /etc/pki/tls/certs/www0.crt
  SSLCertificateKeyFile /etc/pki/tls/private/www0.key
  SSLCertificateChainFile /etc/pki/tls/certs/example-ca.crt
</VirtualHost>

<Directory /srv/www0/www>
  Require all granted
</Directory>
EOF'

# 設定 virtual host (webapp0.example.com)
$ sudo bash -c 'cat <<EOF > /etc/httpd/conf.d/ssl-webapp0.conf
<VirtualHost *:443>
  ServerName webapp0.example.com
  DocumentRoot /srv/webapp0/www
  ErrorLog logs/ssl_webapp0_error_log
  TransferLog logs/ssl_webapp0_access_log
  LogLevel warn
  SSLEngine on
  SSLProtocol all -SSLv2 -SSLv3
  SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5
  SSLCertificateFile /etc/pki/tls/certs/webapp0.crt
  SSLCertificateKeyFile /etc/pki/tls/private/webapp0.key
  SSLCertificateChainFile /etc/pki/tls/certs/example-ca.crt
</VirtualHost>

<Directory /srv/webapp0/www>
  Require all granted
</Directory>
EOF'

# 啟動 httpd 服務 & 開啟防火牆
$ sudo systemctl enable httpd.service
$ sudo systemctl restart httpd.service
$ sudo firewall-cmd --permanent --add-service=https
$ sudo firewall-cmd --reload
```

----------------------------------------------------

10.4 Integrating Dynamic Web Content
====================================

## 10.4.1 Common Gateway Interface

要讓 Apache 可以執行 CGI，要在 `/etc/httpd/conf/httpd.conf` 設定檔中加上：

> ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"

另外還要進行以下設定：

- CGI script 要以 apache:apache 的身分執行

- CGI script 的 SELinux context type 必須是 `httpd_sys_script_exec_t`

- 必須提供一個 `Directory` block 設定，搭配v `Options None` & `Require all granted` 設定

## 10.4.2 Serving dynamic PHP content

- 要安裝 `mod_php` 套件

- 加上以下的 configuration 讓 php 生效

```bash
<FilesMatch \.php$>
  SetHandler application/x-httpd-php
</FilesMatch>
DirectoryIndex index.php
```

## 10.4.3 Serving dynamic Python content

- python & httpd 現在支援 WSGI(Web Server Gateway Interface)

- 需要額外安裝 `mod_wsgi` 套件

- 在 `VirtualHost` block 中增加 `WSGIScriptAlias` 設定

- WSGI application 必須以 `apache:apache` 的身分執行，且 SELinux context type 必須是 `httpd_sys_content_t`

```bash
# 所有到 http://servername/myapp 的 request 都會由 /srv/myapp/www/myapp.py 統一處理
WSGIScriptAlias   /myapp   /srv/myapp/www/myapp.py
```

## 10.4.4 Database connectivity

- 若要讓 apache 存取遠端的 database，必須要把 SELinux Boolean `httpd_can_network_connect_db` 設定為 `1`

- 若遠端的 database port 非標準的 port，則必須要把 SELinux Boolean `httpd_can_network_connect` 設定為 `1`

----------------------------------------------------

Practice: Configuring a Web Application
=======================================

## 目標

1. 讓 remote client 可以透過網址 `http://serverX.example.com` 連到網頁

2. 首頁程式為 PHP 開發，檔案位於 `/var/www/html/index.php`

## 實作過程

```bash
# 要讓 php 正常執行 & 可以連結資料庫，必須安裝 mod_php & php & php-mysql 套件
$ sudo yum -y install mod_php php php-mysql

$ sudo firewall-cmd --permanent --add-service=http
$ sudo firewall-cmd --reload

$ sudo systemctl enable httpd.service
$ sudo systemctl restart httpd.service
```

----------------------------------------------------

Lab: Configuring a Web Application
==================================

## 目標

> 先執行 `lab webapp setup`

1. 一個支援 `https` 的 `python`(WSGI) 網站 `https://webappX.example.com`

2. 相關檔案如下：

- TLS certificate: `http://classroom/pub/tls/certs/webappX.crt`

- TLS private key: `http://classroom/pub/tls/private/webappX.key`

- TLS CA certificate: `http://classroom/pub/example-ca.crt`

- Python application: `/home/student/webapp.wsgi`

## 實作過程

```bash
$ sudo yum -y groupinstall web-server
$ sudo yum -y install mod_wsgi

# 放到 /srv/xxx/www 目錄的檔案，搭配 restorecon 可以把 SELinux context type 自動變成 httpd_sys_content_t
$ sudo mkdir -p /srv/webapp/www
$ sudo cp ~/webapp.wsgi /srv/webapp/www/
$ sudo ls -lZ /srv/webapp/www/webapp.wsgi
-rw-r--r--. root root unconfined_u:object_r:var_t:s0   /srv/webapp/www/webapp.wsgi
$ sudo restorecon -vvRF /srv/
restorecon reset /srv/webapp context unconfined_u:object_r:var_t:s0->system_u:object_r:var_t:s0
restorecon reset /srv/webapp/www context unconfined_u:object_r:var_t:s0->system_u:object_r:httpd_sys_content_t:s0
restorecon reset /srv/webapp/www/webapp.wsgi context unconfined_u:object_r:var_t:s0->system_u:object_r:httpd_sys_content_t:s0

# 取得憑證 & private key
$ sudo wget -O /etc/pki/tls/certs/webapp.crt http://classroom/pub/tls/certs/webapp0.crt
$ sudo wget -O /etc/pki/tls/private/webapp.key http://classroom/pub/tls/private/webapp0.key
$ sudo chmod 0400 /etc/pki/tls/private/webapp.key
$ sudo wget -O /etc/pki/tls/certs/example-ca.crt http://classroom/pub/example-ca.crt
$ sudo ls -Z /etc/pki/tls/certs/webapp.crt /etc/pki/tls/private/webapp.key /etc/pki/tls/certs/example-ca.crt
-rw-r--r--. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/certs/example-ca.crt
-rw-r--r--. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/certs/webapp.crt
-r--------. root root unconfined_u:object_r:cert_t:s0  /etc/pki/tls/private/webapp.key

$ sudo bash -c 'cat <<EOF > /etc/httpd/conf.d/webapp.conf
<VirtualHost _default_:443>
ServerName webapp0.example.com
ErrorLog logs/ssl_webapp_error_log
TransferLog logs/ssl_webapp_access_log
LogLevel warn
SSLEngine on
SSLProtocol all -SSLv2 -SSLv3
SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5
SSLCertificateFile /etc/pki/tls/certs/webapp.crt
SSLCertificateKeyFile /etc/pki/tls/private/webapp.key
SSLCertificateChainFile /etc/pki/tls/certs/example-ca.crt
WSGIScriptAlias / /srv/webapp/www/webapp.wsgi
</VirtualHost>

<Directory /srv/webapp/www>
    Require all granted
</Directory>
EOF'

# 啟動 httpd.service & 開啟防火牆
$ sudo systemctl enable httpd.service
$ sudo systemctl restart httpd.service
$ sudo firewall-cmd --permanent --add-service=https
$ sudo firewall-cmd --reload
```
