---
layout: post
title:  "學習 Linux Command Line 筆記(4) - 撰寫 shell script"
description: "學習 Linux Command Line 中 shell script 的撰寫"
date:   2015-03-16 21:10:00
published: true
comments: true
categories: [linux]
tags: [Linux, Bash]
---

啟動專案 (Starting A Project)
=============================

``` bash
#!/bin/bash

# Program to output a system information page

TITLE="System Information Report for $HOSTNAME"
CURRENT_TIME=$(date +"%x %r %Z")
TIME_STAMP="Generated $CURRENT_TIME, by $USER"

cat << _EOF_
<html>
    <head>
        <title>$TITLE</title>
    </head>
    <body>
        <H1>$TITLE</H1>
        <p>$TIME_STAMP</p>
    </body>
</html>
_EOF_
```

---------------------------------------

流程控制：IF (Flow Control: Branching With if)
==============================================


使用 **$?** 判斷程式執行的結束狀態：

``` bash
$ ls -d /usr/bin/
/usr/bin/
$ echo $?
0

$ ls -d /bin/usr
ls: cannot access /bin/usr: No such file or directory
$ echo $?
2
```

test / 加強版 test / (()) 的混搭使用

``` bash
#!/bin/bash

INT=-5
if [[ "$INT" =~ ^-?[0-9]+$ ]]; then
    if [ $INT -eq 0 ]; then
        echo "INT is zero"
    else
        if [ $INT -lt 0 ]; then
            echo "INT is negative"
        else
            echo "INT is positive"
        fi

        if [ $((INT % 2)) -eq 0 ]; then
            echo "INT is even."
        else
            echo "INT is odd."
        fi
    fi
else
    echo "INT is not an integer." >&2
    exit 1
fi
```

``` bash
# 建立 temp 目錄並進入 temp
# 若 temp 目錄已存在就會發生錯誤，不執行後面的指令
$ mkdir temp && cd temp

# 檢查 temp 目錄是否存在，若不存在則執行後面指令建立目錄
$ [ -d temp ] || mkdir temp
```

---------------------------------------

讀取鍵盤輸入 (Reading Keyboard Input)
=====================================

### 讀取多個輸入

#### 存入多個變數中

``` bash
echo "Enter one or more values > "
read var1 var2 var3
echo "var1 = $var1"
echo "var2 = $var2"
echo "var3 = $var3"
```

``` bash
$ ./multi-input-2
Enter one or more values >
ab cd ef
var1 = ab
var2 = cd
var3 = ef

# 輸入數量不足
$ ./multi-input-2
Enter one or more values >
ab
var1 = ab
var2 =
var3 =

# 輸入數量超過原有變數
$ ./multi-input-2
Enter one or more values >
ab cd ef gh
var1 = ab
var2 = cd
var3 = ef gh
```

#### 未指定接收變數，則全部內容都會存入 **$REPLY** 變數中

``` bash
echo -n "Enter one or more values > "
read
echo "REPLY = $REPLY"
```

``` bash
$ ./multi-input-1
Enter one or more values > var1 var2 var3 var4
REPLY = var1 var2 var3 var4
```

### 限定時間的輸入

限定 10 秒內必須輸入，否則則出現 timeout 錯誤

``` bash
if read -t 10 -sp "Enter secret passphrase > " secret_pass; then
    echo -e "\nSecret passphrase = $secret_pass"
else
    echo -e "\nInput timed out" >&2
    exit 1
fi
```

### 測試輸入資料是否合法

``` bash
# 測試是否屬於合法的檔名
$ [[ "filename.yml" =~ ^[-[:alnum:]\._]+$ ]] && echo true || echo false
true
$ [[ "&filename.yml" =~ ^[-[:alnum:]\._]+$ ]] && echo true || echo false
false

# 測試是否屬於浮點數
$ [[ "2.9" =~ ^-?[[:digit:]]*\.[[:digit:]]+$ ]] && echo true || echo false
true
$ [[ "2" =~ ^-?[[:digit:]]*\.[[:digit:]]+$ ]] && echo true || echo false
false

# 測試是否屬於整數
$ [[ "1" =~ ^-?[[:digit:]]+$ ]] && echo true || echo false
true
$ [[ "1.5" =~ ^-?[[:digit:]]+$ ]] && echo true || echo false
false
```

---------------------------------------

流程控制：while & until (Flow Control: Looping With while / until)
==============================================

while 搭配 read 從外部檔案取得資料輸入

``` bash
while read distro version release; do
        printf "Distro: %s\tVersion: %tReleased: %s\n" $distro $version $release
done < distros.txt
```

將排序好的資料 pipe 給 while 進行處理

``` bash
sort -k 1,1 -k 2n distros.txt | while read distro version release; do
        printf "Distro: %s\tVersion: %s\tReleased: %s\n" $distro $version $release
done
```

---------------------------------------

位置參數 (Positional Parameters)
================================

### 使用 shift 指令將參數往前推移

``` bash
count=1
while [[ $# -gt 0 ]]; do
        echo "Argument $count = $1"
        count=$((count + 1))
        shift
done
```

執行結果：

``` bash
$ ./posit-param2 1 2 3
Argument 1 = 1
Argument 2 = 2
Argument 3 = 3
```

### 群組管理位置參數

$*, "$*", $@, "$@" 的差異比較：

``` bash
print_params() {
        echo "\$1 = $1"
        echo "\$2 = $2"
        echo "\$3 = $3"
}

pass_params() {
        echo -e "\n" '$* :'; print_params $*
        echo -e "\n" '"$*" :'; print_params "$*"
        echo -e "\n" '$@ :'; print_params $@
        echo -e "\n" '"$@" :'; print_params "$@"
}

pass_params "hello" "shell script"
```

執行結果：

``` bash
 $* :
$1 = hello
$2 = shell
$3 = script

 "$*" :
$1 = hello shell script
$2 =
$3 =

 $@ :
$1 = hello
$2 = shell
$3 = script

 "$@" :
$1 = hello
$2 = shell script
$3 =
```

---------------------------------------

字串與數字 (Strings And Numbers)
================================

### 空變數的延展

變數無值則取代：<font color='red'>**${parameter:-word}**</font>

``` bash
$ foo=

$ echo $foo

# 若沒有資料則以內容輸出
$ echo ${foo:-"substitude valie if unset"}
substitude valie if unset

# $foo 依然沒有內容
$ echo $foo

$ foo=bar
$ echo ${foo:-"substitude valie if unset"}
bar
```

變數無值則設定變數值：<font color='red'>**${parameter:=word}**</font>

``` bash
$ foo=
$ echo ${foo:="default valie if unset"}
default valie if unset

# 此時 $foo 變數已經有內容存在
$ echo $foo
default valie if unset
```

變數為空值則發生錯誤：<font color='red'>**${parameter:?word}**</font>

``` bash
$ foo=
# 若 $foo 為空值，則發生錯誤
$ echo ${foo:?"paramter is empty"}
-bash: foo: paramter is empty
$ echo $?
1

$ foo=bar
$ echo ${foo:?"paramter is empty"}
bar
[vagrant@localhost LinuxCommandLine]$ echo $?
0
```

變數有值則取代：<font color='red'>**${parameter:+word}**</font>

``` bash
$ foo=
$ echo ${foo:+"substitude valie if set"}

$ foo=bar
$ echo ${foo:+"substitude valie if set"}
substitude valie if set
```

### 回傳變數名稱延展

列出環境變數中以 **BASH** 開頭的項目：

``` bash
$ echo ${!BASH*}
BASH BASHOPTS BASHPID BASH_ALIASES BASH_ARGC BASH_ARGV BASH_CMDS BASH_COMMAND BASH_LINENO BASH_SOURCE BASH_SUBSHELL BASH_VERSINFO BASH_VERSION
```

### 字串處理

字串長度：<font color='red'>**${#parameter}**</font>

取得變數中的部分字串內容：

- <font color='red'>**${parameter:offset}**</font>

- <font color='red'>**${#parameter:offset:length}**</font>

> 若 offset 為負值，則表示從字串結尾開始算(負值必須前置空白)

``` bash
$ foo="this string is long."

$ echo ${#foo}
20

$ echo ${foo:5}
string is long.

$ echo ${foo:5:6}
string

$ echo ${foo: -5}
long.

$ echo ${foo: -5:2}
lo
```

移除在 parameter 中 pattern 定義的字串**前面**部分：

- <font color='red'>**${parameter#pattern}**</font> (移除比對到最短的字串)

- <font color='red'>**${parameter##pattern}**</font> (移除比對到最長的字串)

``` bash
$ foo=file.txt.zip

$ echo ${foo#*.}
txt.zip

$ echo ${foo##*.}
zip
```

移除在 parameter 中 pattern 定義的字串**後面**部分：

- <font color='red'>**${parameter%pattern}**</font> (移除比對到最短的字串)

- <font color='red'>**${parameter%%pattern}**</font> (移除比對到最長的字串)

``` bash
$ foo=file.txt.zip

$ echo ${foo%.*}
file.txt

$ echo ${foo%%.*}
file
```

搜尋並替換字串：

- <font color='red'>**${parameter/pattern/string}**</font> (第一個比對相符被替換)

- <font color='red'>**${parameter//pattern/string}**</font> (所有比對相符皆替換)

- <font color='red'>**${parameter/#pattern/string}**</font> (前面第一個比對相符被替換)

- <font color='red'>**${parameter/%pattern/string}**</font> (後面第一個比對相符被替換)

``` bash
$ foo=JPG.JPG

$ echo ${foo/JPG/jpg}
jpg.JPG

$ echo ${foo//JPG/jpg}
jpg.jpg

$ echo ${foo/#JPG/jpg}
jpg.JPG

$ echo ${foo/%JPG/jpg}
JPG.jpg
```

### 基數

二進位、八進位、十進位、十六進位的處理：

``` bash
$ echo $((21))
21
$ echo $((021))
17
s$ echo $((0x21))
33
$ echo $((2#111111))
63
```

---------------------------------------

陣列 (Arrays)
=============

### 指派陣列

``` bash
# 單值
$ a[1]=foo
$ echo ${a[1]}
foo

# 多值
$ days=(Sun Mon Tue Wed Thu Fri Sat)
$ echo ${days[0]}
Sun
$ days=([0]=Sun [1]=Mon [2]=Tue [3]=Wed [4]=Thu [5]=Fri [6]=Sat)
```

### 陣列操作

``` bash
$ animal=("a dog" "a cat")
$ for i in ${animal[*]}; do echo $i; done
a
dog
a
cat

$ for i in ${animal[@]}; do echo $i; done
a
dog
a
cat

$ for i in "${animal[*]}"; do echo $i; done
a dog a cat
# 最能反映陣列真實資料狀況(使用雙引號 + @)
$ for i in "${animal[@]}"; do echo $i; done
a dog
a cat

# 陣列元素指派不完整
$ foo=([2]=a [4]=b [6]=c)
$ for i in "${foo[@]}"; do echo $i; done
a
b
c
$ for i in "${!foo[@]}"; do echo $i; done
2
4
6

# 陣列末端加入元素
$ foo=(a b c)
$ echo ${foo[@]}
a b c
$ foo+=(d e f)
$ echo ${foo[@]}
a b c d e f

# 陣列排序
$ a=(f e d c b a)
$ a_sorted=($(for i in "${a[@]}"; do echo $i; done | sort))
$ echo ${a_sorted[@]}
a b c d e f

# 刪除陣列
foo=(a b c d e f)
$ echo ${foo[@]}
a b c d e f
$ unset "foo[2]"
$ echo ${foo[@]}
a b d e f
$ unset foo
$ echo ${foo[@]}

$ foo=(a b c d e f)
$ foo=
$ echo ${foo[@]}
b c d e f
$ foo=A
$ echo ${foo[@]}
A b c d e f
```

---------------------------------------

EXOTICA 雜項
============

### Subshells

因為 pipeline 的命令會在 subshell 中執行，因此可以透過 Process Substitution 的方式來處理 subshell 中變數回到 shell 會消失的問題

``` bash
$ echo "foo" | read
$ echo $REPLY

$ read < <(echo "foo")
$ echo $REPLY
foo
```

---------------------------------------

參考資料
========

- [The Linux Command Line by William E. Shotts, Jr.](http://linuxcommand.org/tlcl.php)