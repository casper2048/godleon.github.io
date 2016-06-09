---
layout: post
title:  "[RHCE7] RH254 Chapter 11 Writing Bash Scripts Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 11 Writing Bash Scripts 留下的內容"
date: 2016-06-05 11:40:00
published: false
comments: true
categories: [linux]
tags: [Linux, RHCE, RH254]
---

11.1 Bash Shell Scripting Basics
===============================

## 11.1.2 Creating and executing Bash shell scripts

- bash shell script  的內容開頭要加上 `#!/bin/bash` 宣告為 bash script

- shell scripr 要有 `execute` 的權限才可以使用 `./script_name` 的方式執行，不然也可以用 `bash script_name` 的方式執行

## 11.1.3 Displaying output

若要在 shell script 產生 standard error，可使用類似以下方式：

```bash
echo "This is a error message" >&2
```

## 11.1.5 Using Bash shell expansion features

使用 `$[]` 可以進行簡單的數學運算：

```bash
$ echo $[1+2]
3
$ COUNT=10; echo $[$[${COUNT} + 1] + 2]
13

$ COUNT=10; echo $[++COUNT]
11
```

## 11.1.6 Iterating with the for loop

```bash
$ for h in host1 host2 host3; do echo $h; done
host1
host2
host3
$ for h in host{1,3}; do echo $h; done
host1
host3
$ for h in host{1..3}; do echo $h; done
host1
host2
host3
$ for h in host{a..c}; do echo $h; done
hosta
hostb
hostc
$ for e in $(seq 2 2 8); do echo $e; done
2
4
6
8
```

## 11.1.7 Troubleshooting shell script bugs

可透過以下兩個方式讓 shell script 執行時有完整的輸出以方便 debug：

- 在 shell script 開頭加上 `#!/bin/bash -x`

- 以 `bash -x <SCRIPT_NAME>` 的方式執行程式

> 也可以在 shell script 透過 `set -x` 開啟 debug 功能，再透過 `set +x` 把 debug 功能關閉

-------------------------------------------

Practice: Writing Bash Scripts
==============================

## 目標

1. 一一備份 MariDB 中所有的資料庫

2. 回報每一個資料庫備份檔的大小

## 實作過程

安裝 & 啟動 MariaDB：

```bash
# 安裝 & 啟動 MariaDB
$ sudo yum -y groupinstall mariadb mariadb-client
$ sudo systemctl enable mariadb.service
$ sudo systemctl restart mariadb.service
```

撰寫 shell script：

```bash
$ vi backup_db.sh
#!/bin/bash
DBUSER=root
FMT_OPTIONS='--skip-column-names -E'
COMMAND='show databases'
BACKUP_DIR=~/dbbackup

mkdir -p ${BACKUP_DIR}
for db in $(mysql ${FMT_OPTIONS} -u ${DBUSER} -e "${COMMAND}"  | grep -v ^* | grep -v schema)
do
  mysqldump -u ${DBUSER} ${db} > ${BACKUP_DIR}/${db}.dump
done

TOTAL=0
for dump in ${BACKUP_DIR}/*
do
  SIZE=$(stat --printf "%s\n" ${dump})
  TOTAL=$[$TOTAL + $SIZE]
done

for dump in ${BACKUP_DIR}/*
do
  SIZE=$(stat --printf "%s\n" ${dump})
  echo "${dump}, ${SIZE}, $[100 * ${SIZE} / ${TOTAL}]%"
done
```

執行 shell scripts：

```bash
$ bash backup_db.sh
/home/student/dbbackup/mysql.dump, 514664, 99%
/home/student/dbbackup/test.dump, 1261, 0%
```

-------------------------------------------

Lab: Writing Bash Scripts
=========================

先以 root 身分在 server 端執行 `lab bashbasic setup`

## 目標

1. 撰寫 script `/usr/local/sbin/mkaccounts`

2. 可以讀取 `/tmp/support/newusers` 中的檔案並建立使用者(格式為 `FirstName:LastName:不重要:SupportTier`)

3. 建立使用者名稱為 FirstName 第一個字母 + LastName，comment 的部分則是 Full Name

4. 根據 Support Tier 的人數建立報表，格式如下：(`"Tier No","人數","百分比"`)

```bash
"Tier 1","22","44%"
"Tier 2","15","30%"
"Tier 3","13","26%"
```

## 實作過程

```bash
[root@serverX ~]# vi /usr/local/sbin/mkaccounts
#!/bin/bash

NEWUSERSFILE="/tmp/support/newusers"

TIER_1=0
TIER_2=0
TIER_3=0
TOTAL_NUMBER=$(cat ${NEWUSERSFILE} | wc -l)
for usr in $(cat ${NEWUSERSFILE})
do
  FIRST_NAME=$(echo "${usr}" | cut -d: -f1)
  LAST_NAME=$(echo "${usr}" | cut -d: -f2)
  SUPPORT_TIER=$(echo "${usr}" | cut -d: -f4)

  case "${SUPPORT_TIER}" in
    1)
      TIER_1=$[${TIER_1} + 1]
      ;;
    2)
      TIER_2=$[${TIER_2} + 1]
      ;;
    3)
      TIER_3=$[${TIER_3} + 1]
      ;;
  esac

  ACCOUNT_FNAME=$(echo ${FIRST_NAME} | cut -c 1)
  USER_ACCOUNT=$(echo "${ACCOUNT_FNAME}${LAST_NAME}" | tr '[:upper:]' '[:lower:]')
  useradd -c "${FIRST_NAME} ${LAST_NAME}" ${USER_ACCOUNT}
done

echo "\"Tier 1\",\"${TIER_1}\",\"$[${TIER_1} * 100 / ${TOTAL_NUMBER}]%\""
echo "\"Tier 2\",\"${TIER_2}\",\"$[${TIER_2} * 100 / ${TOTAL_NUMBER}]%\""
echo "\"Tier 3\",\"${TIER_3}\",\"$[${TIER_3} * 100 / ${TOTAL_NUMBER}]%\""


[root@serverX ~]# chmod +x /usr/local/sbin/mkaccounts
```
