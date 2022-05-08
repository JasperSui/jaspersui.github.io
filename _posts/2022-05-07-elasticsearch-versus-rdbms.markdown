---
layout:     post
title:      "Elasticsearch 和 RDBMS 面對篩選多欄位資料需求的差別"
subtitle:   ""
catalog:    true
date:       2022-05-07 08:00:00
author:     "Jasper"
header-img: "img/post-bg-2015.jpg"
tags:
    - Elasticsearch
    - RDBMS
    - Search Engine
---
## 前言

最近有遇到需求是想要在眾多資料中，去同時透過很多欄位篩選出資料，且這些欄位的型別並不受限，使用者在篩選時還要可以依照不同型別的欄位給出更細部的篩選條件，最重要的是，**要篩選哪些欄位是由使用者自行決定的，組合方式可能有上千種。**這就是這篇文章誕生緣由，接下來會和大家一起去走過技術選型的流程，如果有更好的想法也歡迎在留言分享！

## 介紹

![img](https://i.imgur.com/mP5nWWv.png)

[Certified Kubernetes Administrator (CKA)](https://www.cncf.io/certification/cka/) 是由 [Cloud Native Computing foundation (CNCF)](https://www.cncf.io/) 所發行的證書，目前的報名費用是 **$375** 美元，不定時會有一些折扣碼釋出，但今年的黑色星期五就很罕見的沒有佛心折扣碼，只有一些固定打 8 折或 85 折的折扣碼，但下面會提到我是在哪邊去買這個考試資格的，總共只花了 6500 台幣，算起來大概打了 62 折，能大幅降低荷包失血的程度 XD。

CKA 提供瞭解各個 Kubernetes 基礎元件組成，以及清楚 Kubernetes 架構和運作模式的「管理者/開發者」來進行測驗，考試時除了 Terminal 分頁以外，還可以再開一個[官方 Kubernetes Docs](https://kubernetes.io/docs/home/) 的分頁來查詢，考試時間 2 個小時，我遇到的題目總共有 17 題，體感上來說題目的順序並不是照著難度來排的，考試配分的比重我覺得和官方提供的[配分表](https://www.cncf.io/certification/cka/)蠻相似的：

![img](https://i.imgur.com/ifFZxvK.png)

本篇適合粗略瞭解**容器化概念 (Containerized)**的讀者，不管你之前有沒有接觸過 Kubernetes，相信這篇文章應該能帶給你一些收穫，會和大家分享一些考試的技巧，以及我如何從只會在工作環境使用一些 Kubernetes 簡單指令到拿到 CKA 證照的！

<hr>

## 考前準備

#### Udmey 課程

在這裡推薦任何不管有無經驗想要考 CKA 的讀者，花個 380 塊台幣跟著 Udemy 上的課程 [Certified Kubernetes Administrator (CKA) with Practice Tests](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/) 來準備。如果你完全沒有經驗的話，那這篇能夠帶你去從 Kubernetes 基礎的設計從廣而深去學習；如果你是平常就有在使用 Kubernetes 來做日常開發的工程師，那這個課程能夠更穩固你的知識，甚至是在之後需求和架構上的設計都能因爲更全面地瞭解 Kubernetes 而有更多元彈性的解決方案！

這個課程的講師 **Mumshad Mannambeth** 除了 CKA 以外也有提供 CKAD 的課程，整個課程上完的感覺優點如下：

1. 價格便宜，Udemy 時常會有折扣價所以大約 10 美元左右就可以買到了
2. 上課節奏是很平穩的，講師口音不明顯，語速偶爾會稍快但也可以練習一些英聽
3. 對於一些前置的基礎概念包括 **Docker**, **Networking**, etc. 都會有小節來幫你補足先備知識
4. 學習完各個元件、知識後都有 **Practice Tests** 可以練習，最後還有三次的 *Mock Exam* 可以做
5. **提供免費的 Web 環境 (KodeKloud) 讓你無限次地去做 Practice Tests 來熟悉操作**

![img](https://i.imgur.com/JNFCnOZ.png)


最後一點是我認為最重要也是這個課程最優質的一點，因為有一個環境能夠讓你方便快速的實戰練習各個小節學到的知識和技巧是很重要的。CKA 在你大概懂 Kubernetes 的架構之後，考點就剩下你對 `Terminal`、`Kubernertes CLI` 的熟練度了，如果你能夠在測試環境去完成各個元件的操作 (`Pod`, `Service`, `Deployment` ...)，那在真正上考場的時候也就不需要擔心了，這邊提供一份之前考試時使用的[書籤資料夾](https://github.com/JasperSui/cka-bookmarks-cheatsheet)的檔案，裡面已經有各種常見且分好類的文檔連結，只要先提前匯入這個書籤檔到瀏覽器裡，這樣在看到類似的題目時可以直接用書籤進到文檔頁面，會省去很多時間。

<br>

#### 考試碼購買

官方原價從 **$300** 漲到 **$375** 了，漲幅實在是有點大，但有效期也從原本的一年變為三年，所以也算是不無小補。 CNCF 之前時常會在一些節慶的時候提供蠻優惠的折扣碼，我那時候比較靠近黑五，據往年經驗文是會有到 **40% off**，但我怎麼等都沒有看到一些像樣的折扣，後來我就去找看看有沒有其他管道可以買到比較便宜的折扣碼，當時是到[迎棧科技的 CKA 認證課程](https://www.inwinstack.com/zh/training/cka/)下面的 *CKA 考試碼*表單購買，那時候買的價錢是 6500 台幣，但是剛剛看活動好像已經沒有了，建議有興趣的讀者可以直接打電話問看看他們，或許還有機會。

<br>

#### 準備時間

我大概是在 8 月下旬的時候買上面的課程決定要考 CKA 的，每天用的時間就是平日中午吃飯從配 Youtube 換成配 Udemy，一天取決於小節長度，大約都可以看 2 ~ 4 個小節左右，假日則是看個 1 ~ 2 個小時的課程，再邊補齊平日看完但沒做完 Practice Tests 的地方，最後在 11 月中旬的時候就順利考到證照啦！

其實我準備的時間算是拉得很長的，主要原因是不想要學得太趕，把每個地方都學透徹了慢慢學才繼續走下一步，加上某些不熟悉的地方會練習比較多次，像是 **ETCD 備份**、**Cluster 升級** 之類的，平常如果你和我一樣接觸到的都是 Cloud Platform (`GKE`, `EKS`, `AKS`) 幫你管理 Kubernetes 平台的話，那這些東西對你來說會相較於使用 `Pod`、`Deployment`、`Service`、`Ingress` 來得陌生，而這些考題的比分佔比也是相較簡單的題目多蠻多的，加上這幾塊因為流程其實雖然繁瑣，但最根本的那幾個指令和步驟都是在文檔上面清清楚楚的寫出來了，所以我認為這些都是很好拿分的考點，只需要多熟悉操作幾次，基本上看到就是送分題。

在 11 月初的時候因為已經安排考試時間了，所以我在這段期間就是有意識地去不斷重作一些自己認為不熟的章節，包括 Mock Exam 三題也是刷了好幾遍，盡可能讓自己去熟悉實際操作，考試當下看到題目的時候才不會因為緊張而忘記指令，並且讀熟 [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) 不管是考試還是日常使用都會有非常大的幫助。

這邊也推薦你看這部 [Kubernetes Certification Tips - How to Pass Certified Kubernetes Administrator Exam? Time Management](https://www.youtube.com/watch?v=wgfjXHw7uPs)，一樣是 Udemy 講師做的影片，裡面有教你考證照的一些技巧，其中包括答題時的時間管理，非常值得一看！

<br>

#### 我的程度可以應考了嗎？

我想這會是許多人考試前最大的疑惑，常常會想去找一些考古題可能來猜題，結果會發現其實好多題目自己還不會，在考試前幾天就更緊張。

但我要說的會是只要你看完 Udemy 的課程並紮實地練習**每一個** Practice tests，同時最後那三個 Mock Exam 不斷地做，做到你可以看到題目就反射出這題相對應的步驟大概會是怎樣，那我想要直接上場考試是沒問題的，如果你還是有點怕，那這時候再去看一些考古題然後如果有遇到不會的，一樣養成習慣只透過 **Kubernetes Documentation** 來尋找解答，最好是紀錄當下看到題目的思路，以及後來是怎麼解決的，這樣可以有效幫助你在考場上遇到不會的題目，一樣可以照著自己習慣的 SOP 來走，最後就只需要一點運氣加上自信，準備迎接挑戰！

如果你已經買了考試碼，那還有一個練習的管道是官方提供的服務 [Killer.sh](https://killer.sh/)，裡面會有兩次的模擬考使用，因為我這邊沒有使用到所以沒辦法提供太多的建議，詳細的規則可以再去官方網站查詢一下，聽說難度是相較於真正考試還難，多練習總是好的！

<br>

#### 預約考試

在兌換完你的考試碼後，官方會寄出一封教你如何[預約考試的指引守則](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/scheduling-or-rescheduling-an-exam)：

![img](https://i.imgur.com/iLt3yNA.png)

預約前必須先對你要考試的設備進行[檢查](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/exam-preparation-checklist)，檢查完之後才能夠預約，下面是檢查設備的畫面。

![img](https://i.imgur.com/rBUdF6P.png)

信上說的其實就是叫你提前 24 小時預約，還有說明如果要取消退費的一些注意事項，提前預約考試非常重要，因為他們預約網站非常之卡，而你最快只能預約 24 小時以後的考試，所以絕對不要壓線去預約自己想要的時段，不然有很大的機率會預約不到。

預約會需要檢查瀏覽器，上次我想用 Edge 沒辦法過，所以還是用 Chrome 去通過它們的設備檢查，接下來要準備你的身份證明文件，我是用**護照**，因為上面有英文名字，在驗證過程中也會需要你填入一些個人資訊，**填入的英文名字要完全和證件上的相符才會讓你繼續開始考試**，所以這邊務必要確認名字是對得上的。

當你預約完之後會有另一封考前注意事項的信，以上提到的所有細節都可以在 [Linux Foundation Docs](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook) 中找到，包括[考試環境(桌面、地板、飲用水)的規定、個人行為的規定](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/exam-rules-and-policies)，以下為一些常用的連結傳送門：

- [註冊考試教學](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/exam-registration)
- [個人身份驗證注意事項](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/candidate-identification-and-authentication)
- [退費政策](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/exam-refund-policy)
- [My Portal 使用教學](https://docs.linuxfoundation.org/tc-docs/certification/lf-candidate-handbook/my-portal)

<hr>

## 考試期間

#### 考試環境

建議可以在考試開始之前 30 分鐘就定位，先把整個環境整理乾淨，確保不會被外在的人事物影響，最後沈澱一下自己的心靈準備應戰，大約在考試時間開始前 15 分鐘就可以加入考試了。

考試開始之前你會進到一個畫面，然後等考官進來之後會透過右下角的聊天視窗和你對談，會先確認你的攝影機、麥克風有沒有問題，中間過程就照著考官的指示操作即可，記得當考官問完話或是說完話的時候要記得用文字訊息回個 OK 之類的才能進到下一步。

接著會請你以不阻擋你的臉的方式拿著你的身份證明文件對著攝影機，確認完設備和身份沒問題之後會請你用攝影機環繞整個桌面、桌下、牆壁和門的環境，要確保桌下也是乾淨的。

<br>

#### 考試畫面

![img](https://i.imgur.com/GXLi5XK.png)

考試畫面其實和 **KodeKloud** 提供的練習環境長得差不多，一樣左邊是題目敘述，右邊會是 Terminal。

值得一提的是官方有提供 `Flag` 功能，可以先把自己有疑惑或是不確定的題目標記起來，之後可以再隨時回來作答，這個功能非常有用，尤其當你在一進到考場時可能沒有辦法那麼沉著的解題，一開始可能還是會很緊張，這時候我會建議都直接先標記起來，之後再慢慢做，當你解到幾題比較簡單的題目而且做完之後，會比較冷靜，這時候再回頭去慢慢解那些因為緊張而沒辦法好好答出來的題目就好。

我當時是遇到 17 題題目，到我第一輪跑完所有題目 (包括標記住還沒解完的題目)時，大概還有一個小時左右，這時候我才來專心去解那些標起來的題目，直到我解完所有的題目大概還剩下 20 ~ 30 分鐘左右，所以整體來說時間算是蠻充裕的。

<hr>

## 考試後

恭喜你完成了一個不簡單的任務！接下來就是在 24 小時左右會收到考試結果的通知信，我是在考完將近 22 小時候收到 **Linux Foundation** 寄來的證照考核結果信件的！

<img src="https://i.imgur.com/9B1DIgR.png" style="width:700px">

除了 **Linux Foundation** 上面的證照和徽章以外，同時也可以去申請 [Credly](https://info.credly.com/) 這個網站的帳號，看起來他可以幫你統整一些你獲得的證照，也可以線上一鍵幫你檢查你的證照是不是還在有效期限內，我個人是用這個網站提供的[個人網址](https://www.credly.com/badges/34544d07-2012-46b8-a76c-a5caaf102443/public_url)放在我的 LinkedIn 上來方便其他人檢視。

用起來的效果大概會像是像這樣，進來之後可以看到右上角有個 Verify 的按鈕，會自動幫你去發行該證照的基金會做驗證：

<img src="https://i.imgur.com/lXhbmvJ.png" style="width:700px">
<br>
<img src="https://i.imgur.com/ZKYo3Al.png" style="width:350px">

<hr>

## 總結

在當初決定要不要考 CKA 的時候，讓我猶豫的只有證照費用有點稍貴 XD，但後來想想覺得如果能用幾千塊來輔助我更全面地學習 Kubernetes 以及證明我對於 Kubernetes 的掌握程度，那其實好像還蠻划算的。

原先在工作日常使用上其實就是用一些前人寫好的 Script，還有簡單的 `Get`, `Descirbe`, `Delete` 這種類似的操作而已，沒有系統性地學習這個工具，遇到問題常常也就是單就錯誤訊息來 Google，但其實只要有深入瞭解過 Kubernetes 是怎麼運作的，稍微對它有點概念，在工作上一定會有很大的幫助。

有下定決心考這個證照對我個人來說是可以推進我每天持續學習的，透過每天中午的短暫時間來把 Udemy 課程的各個小節上完，到真正要準備考試前兩週的全力準備，整體算下來也是不短的過程。當我真正拿到這個證照後，回頭再去想一些之前遇到的不管是架構上的設計還是功能需求，其實都有機會可以用 Kubernetes 來更簡單地達到我們的目的。

建議如果是想要精進 Kubernetes 的讀者可以從學習 CKA 當作入門，接下來可以去看 CKAD 甚至是 CKS，證照不一定要考，我覺得能夠讓自己有動力去把該學的學完，那其實你也可以把課上完就好，只是想說既然都學得差不多了那就順便考一下好了 XD

祝大家都順利拿到證照！如果考完發現有證照有改制或是有哪邊錯誤的地方歡迎提出來，感謝閱讀！

## 參考

[Cloud Native Computing Foundation](https://www.cncf.io/)<br>
[Linux Foundation Documentation](https://www.docs.linuxfoundation.org/)<br>