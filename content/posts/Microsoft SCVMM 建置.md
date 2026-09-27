---
title: Microsoft SCVMM 建置
toc: true
date: 2026-09-20
showTableOfContents: "true"
---
## 目錄
## 一、準備環境

| 套件                                   | 說明                    |
| ------------------------------------ | --------------------- |
| SCVMM 2022 iso                       |                       |
| MS SQL Developer 2022                |                       |
| ADK 10.1.26100.2454 （2024 年 12 月）    | 勾選Deployment Tools 就好 |
| ADK 10.1.26100.2454 的Windows PE 附加元件 |                       |

## 二、安裝說明
1. 安裝前機器必須登入Domain，先需SQL Sever 2022以及 PE ADK，在SCVMM 2022 又有一個 bug 讀不到 PE ADK  `C:\Program Files (x86)\Windows Kits\10\Assessment and Deployment Kit\Deployment Tools\WSIM\amd64` 下的dll檔，必須把dll複製出來搬到上一層WSIM資料夾中

![](static/Pasted%20image%2020260927104605.png)

2. 做進階設定部分，這邊留下預設值

![](static/Pasted%20image%2020260927104613.png)

3. 安裝完成

![](static/Pasted%20image%2020260927104618.png)