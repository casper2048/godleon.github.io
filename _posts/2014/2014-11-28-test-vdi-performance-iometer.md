---
layout: post
title:  "使用 IOMeter 測試 VDI 效能"
date:   2014-11-28 10:55:00
comments: true
categories: [virtualization]
tags: [Virtualization, VMware]
---

最近公司正在評估購買 storage 用來做虛擬化，由於大部分會用在提供使用者桌面，因此 VDI 的效能必須要好，使用者的操作體驗才會不錯

網路上找了一下，一堆測試軟體，考量到可調整測試的彈性，決定以 [IOMeter](http://www.iometer.org/) 來進行測試


測試情境
========

從參考資料可得知，一般的 Desktop VM(文章中寫到是 Windows 7 為例)在操作的過程中，對磁碟的讀寫狀況大概會是如下：

1. 80% Write / 20% Read

2. 80% Random / 20% Sequential

3. Read/Write Block Size 範圍落在 512B ~ 1 MB，為 4K 居多


測試環境準備
===========

建議使用一台全新的 VM 用來做測試，不要安裝額外軟體，只要將 VMTools 裝好即可，且為了將環境單純化，可在 VM 內建立第二個 Virtual Disk 作為測試之用。


IOMeter 測試參數設定
===================

### 主畫面

VM 包含了幾個 vCPU，在畫面中就會出現幾個 Worker，這裡可以不用更動，一般建議設定 4 個 worker 來進行測試。

> 所以可以設定 4 個 vCPU 給 VM

### Disk Targets

- **Target**：選擇第二個 Virtual Disk

- **Maximum Disk Size**：建議設定 41943040 (20GB)；若不符合需求，也可上下調整

- **\# of Outstanding I/Os**：設定 16，模擬同時有 16 個 application 同時對 storage 進行存取，在一般的桌面環境算是一個還合理的數據

### Access Specifications

建立一個新的設定，名稱可自訂(假設為 `VDI`)，參數內容如下：

- **Transfer Request Size**：4 Kilobutes

- **Percent Read/Write Distribution**：80% Write / 20% Read

- **Percent Random/Sequential Distribution**：80% Random / 20% Sequential

- **Align I/O on**：128 Kilobytes

設定完後，按下 `OK` 即可儲存設定。

接著回到主畫面中，將設定 `VDI` 加入到 **Assigned Access Specifications**。

### Test Setup

這邊可以設定測試時間 `Run Time`，可隨個人喜好自行設定。

參考資料中的文章寫到設定 30 秒即可，為了取得一個合理一點的平均值，我會設定在執行 30 分鐘以上。

### Results Display

這邊可以設定測試結果的更新時間，從 1 秒 ~ 無限大(不更新)都可以；另外 `Display` 區塊中還可以設定即時要看到的資訊。

但其實這些資訊最後都會輸出成 csv，可用 Excel 觀看。


執行測試
========

以上都設定完後，按下主畫面上的綠色旗標，就可以開始測試了!


參考資料
========

- [VDI Performance - Using Iometer to Simulate a Desktop Workload | Atlantis Computing Blog](http://blog.atlantiscomputing.com/2013/08/how-to-use-iometer-to-simulate-a-desktop-workload/)

- [Iometer | Benjr.tw](http://benjr.tw/370)
