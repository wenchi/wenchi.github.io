+++
title = "關於 iOS 的記憶體管理 ( Reference Counting )"
date = 2022-01-24T22:20:20+08:00
draft = false
toc = true
[taxonomies]
tags = ["iOS"]
[extra]
summary = "Reference Counting 是記憶體管理的方式，將該物件的參照次數儲存起來，每次該物件被參照到，就將參照次數 +1，被釋放時就 -1，參照次數變成 0 時，立刻移除物件，該物件佔的記憶體會被歸還，有基本的觀念後，接著就分為手動管理記憶體(MRC)與自動管理記憶體(ARC)。"
+++
<!-- more -->

## Reference Counting
Reference Counting 是記憶體管理的方式，將該物件的參照次數儲存起來，每次該物件被參照到，就將參照次數 +1，被釋放時就 -1，參照次數變成 0 時，立刻移除物件，該物件佔的記憶體會被歸還，如下圖。![rc01](/media/rc01.webp)
有基本的觀念後，接著就分為手動管理記憶體(MRC)與自動管理記憶體(ARC)。

* ### 手動管理記憶體 Manual Reference Count (MRC)
    工程師自己寫程式，控制物件的參照次數
    * Objective-C 的參照以及對應語法，如表格： 
      | 物件操作          | Objective-C  |
      | :--------------|:-----|
      | 產生物件、參照物件    | alloc/new/copy/mutableCopy |
      | 參照物件    | retain |
      | 釋放物件  | release |   
      | 移除物件  | dealloc | 
  
  * 需要特別注意，以下這兩種情形，會造成 App 立即閃退
    * release 非自己參照的物件
    * release 已經 release 的物件

  
* ### 自動管理記憶體 Automatic Reference Count (ARC)
    本質還是 Reference count，但改由系統幫我們做管理，工程師不用自己寫語法參照或釋放。ARC 自動幫變數加上所有權修飾符。所有權修飾符 
    * __strong
    * __weak
    * __unsafe_unretained
    * __autoreleasing
    
    需要特別注意，如果兩個擁有 __strong 修飾符的物件，互相參照，會造成記憶體遺漏的問題。__weak 比起其他修飾符，更常出現在程式碼裡面。例如：宣告 Delegate 的時候、block 中操作變數, etc.
---
## Deinitialization
實際開發時，其實沒有那麼多時間去觀察每一個資源有沒有被完整釋放，可以先在類別裡觀察 ```deinit{}```，印出某些字串。如果沒有印出來，再去細看是哪裡抓住資源，造成記憶體遺漏。[ 參考官方 Deinitialization 文件 ](https://docs.swift.org/swift-book/LanguageGuide/Deinitialization.html)

---
## 設置檔案為 ARC 或 MRC
從 2011 開始的 Xcode 4.2 默認所有的檔案都是 ARC，現在應該是非常少的專案還在用 MRC 了，但有時實驗性質，還是會修改一下。

  `-fobjc-arc` => file (open) object automatic reference counting

  `-fno-objc-arc` => file no object automatic reference counting

---

## 參考資料
* [Pro Multithreading and Memory Management for iOS and OS X: with ARC, Grand Central Dispatch, and Blocks](https://www.tenlong.com.tw/products/9781430241164)

* [Automatic Reference Counting](https://docs.swift.org/swift-book/LanguageGuide/AutomaticReferenceCounting.html)