+++
title = "關於 iOS 的 Singleton 及實作方式"
date = 2022-02-09T22:20:24+08:00
draft = false
toc = true
[taxonomies]
tags = ["iOS", "Design Pattern"]
[extra]
summary = "Singleton 原則是單例對象必須保證只有一個實例存在。不管是以何種方式實現，這個設計模式在軟體開發常見。甚至有些內建的類別已經是單例。iOS 有幾種作法可以讓開發者自己做出符合單例的類別，特別要注意 Thread Safety 即可。如果使用 Swift 開發，有官方推薦單行做法。"
+++

<!-- more -->

Singleton 原則是單例對象必須保證只有一個實例存在。不管是以何種方式實現，這個設計模式在軟體開發常見。甚至有些內建的類別已經是單例。iOS 有幾種作法可以讓開發者自己做出符合單例的類別，特別要注意 Thread Safety 即可。如果使用 Swift 開發，有官方推薦單行做法。


## iOS 已實作 Singleton 的類別

Apple 官方的類別有些已實作 singleton，例如：

- UIApplication
- UserDefaults
- NotificationCenter
- FileManager

提供的全局訪問點就是 shared、standard、defualt，例如：

- UIApplication.shared
- UserDefaults.standard
- NotificationCenter.defualt
- FileManager.defualt

---

## 實作邏輯

實作邏輯是當要取用類別時，判斷該類別是否已存在，不存在時初始化一個並回傳。已存在時，直接回傳已存在的實體。
不論是 Objective-C 或是 Swift，我們都根據語言的特性實作，並防止進入 multithread 時，singleton 機制失效。保證 thread safety。

---

## 實作程式碼
### 使用 Objective-C 實作

- 初步實作單例

    實作的程式碼其實很簡短，但是當這個類別進入 multithread 時，此種作法會失效。因為如果在實體還沒有建立完成，而卻有多個線程進入了，每個線程都會產生實體，這個 singleton 機制就失效了。 
    <script src="https://gist.github.com/KennC/7c20483051029c24e07a81c312a7fa7a.js"></script>
- 讓單例保持 Thread Safety

    在 Objective-C 中，我們可以利用 GCD 的 dispatch_once 特性，保護 singleton 機制。因為 dispatch_once 裡頭的 block 只會執行一次，我們把實體初始化放在 block 裡面，就可以確保我們的 singleton 機制不會失效。 
    <script src="https://gist.github.com/KennC/749da1537d59fd8e2b32efd7a502851c.js"></script>

- 覆寫初始化方法(Option)

    另外由於有許多創建方式，我們也嘗試使用幾種來取得 APIService 實體，並檢查是否為同一個。一般來說有經驗的開發者知道使用的物件是單例，都會直接呼叫訪問點，故此步驟是 Option。
    <script src="https://gist.github.com/KennC/c6663fe3a95a5e8bbb6b02b15c198aee.js"></script>

    我們發現使用 alloc 或是 new 取得的實體，都是新的實體，只有使用訪問點 [APIService sharedSingleton] 拿到的才會是同一個實體，如果又要確保其他方式產生的也會是同一個實體，我們可以覆寫 allocWithZone 就可以達到效果。
    <script src="https://gist.github.com/KennC/8036df6d93898931b1cb9877f1501d24.js"></script>

    這樣不論是用 sharedSingleton、alloc 或 new 取得的實體就會都是同一個實體。即是個有實作 singleton 的類別。也有 thread safety。

### 使用 Swift 實作

- 直接把 Objective-C 改成 Swift

    在 Swift 裡，直接將 Objective-C 翻譯成 Swift，但這樣做就可惜使用 Swift。
    <script src="https://gist.github.com/KennC/8c6e032bc97cf9cde6d45b4dfb2d59cc.js"></script>

- 更 Swifty 的單例
    <script src="https://gist.github.com/KennC/06a476ea732ace6872b840a3e5681c23.js"></script>
    我們會發現沒有使用 dispatch_once，因為在[蘋果官方部落格](https://developer.apple.com/swift/blog/?id=7)有這一段話可以讓我們不用自己寫 dispatch_once。
    > The lazy initializer for a global variable (also for static members of structs and enums) is run the first time that global is accessed, and is launched as dispatch_once to make sure that the initialization is atomic. This enables a cool way to use dispatch_once in your code: just declare a global variable with an initializer and mark it private.

    官方已明確指出這些方式都會放入 dispatch_once 初始化，我們就可以在符合官方的規則之下寫出更簡潔的程式。也就是官方推薦的單行單例作法。

- 官方推薦的 Swift 寫法
    
    在 [Cocoa Design Patterns](https://developer.apple.com/documentation/swift/cocoa_design_patterns#//apple_ref/doc/uid/TP40014216-CH7-ID177) 介紹常見的模式裡，[Managing a Shared Resource Using a Singleton](https://developer.apple.com/documentation/swift/cocoa_design_patterns/managing_a_shared_resource_using_a_singleton) 非常清楚的告知開發者，只要這樣寫即可。
    <script src="https://gist.github.com/KennC/94001b5a60ff43f4a937f48888553641.js"></script>

    若有另外要一起寫進單例的，就使用此寫法。
    <script src="https://gist.github.com/KennC/082e6ff72f5b66dbefe6b7001468909a.js"></script>

- 覆寫初始化方法(Option)

    基本上到這裡，已經使用官方推薦的寫法，也不會出現多線程做出多個實體的問題，算是大功告成。如果想要實作"自行創建實體"的特點，我們可以用 private 覆寫 init。想要在別的地方創建這個實體，就會出現 error 擋住。與上方 Objective-C 原理相同，也一樣有經驗的開發者知道使用的物件是單例，都會直接呼叫訪問點，故此步驟也是 Option。
    <script src="https://gist.github.com/KennC/d662238b0c33dd18d6af36dbef402c15.js"></script>