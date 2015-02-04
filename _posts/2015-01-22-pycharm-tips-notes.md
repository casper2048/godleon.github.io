---
layout: post
title:  "PyCharm 使用技巧筆記"
date:   2015-01-31 15:23:00
comments: true
categories: [python]
tags: [Python]
---

[PyCharm](https://www.jetbrains.com/pycharm/) 真的是寫 Python 的神器阿...簡直可以媲美 M$ Visual Studio .NET

就它的功能來對比它的收費標準，算是很便宜阿!!

而 PyCharm 目前我也還在邊用邊學的狀態，以下就慢慢來來記錄一下一些好用的功能，有發現甚麼特別的功能就回來補!

程式碼撰寫
=========

### Optimize Imports

**功能**：把程式碼中的 import module 做整理排序，讓 module 的引用井然有序

**使用方式**：`Ctrl + Alt + O`


### Auto Import Module

**功能**：自動將程式碼中尚未 import 的 module 加入，使用

**使用方式**：在程式碼中尚未 import 的 module 上按下 `Alt + Enter`


### Global Search

**功能**：搜尋專案中程式碼有無指定的搜尋文字

**使用方式**：在程式碼中連按兩下 `Shift`

### Debug @ Django Server

Debug 在寫程式中真的是很重要的功能阿! 下好中斷點，觀察變數的內容變化，來進行 debug 實在是很稀鬆平常的事，但在開發 Django 專案時要怎麼做呢? 要透過以下步驟設定：

1. Edit Run/Debug Configurations，勾選 `Single instance only`

2. 按 `Shift + F9` 以 debug 模式啟動 Django server

接著程式就會在中斷點處停住囉!



工具視窗
=======

### 專案視窗 (Project Window)

用來檢視專案目前的整體結構，快速切換專案視窗的 Hot Key 為 `Alt + 1`
