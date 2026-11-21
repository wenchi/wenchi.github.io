---
title: "關於 iOS 的 Singleton 及實作方式"
date: 2025-06-18 10:00:00 +0800
categories: [iOS, Design Patterns]
tags: [Singleton, Swift, Objective-C, Thread Safety, GCD, dispatch_once, Design Patterns]
---

## Singleton

Singleton（單例）模式的原則是：一個類別必須保證只有一個實體存在，並提供一個全域的存取點。不管是以何種方式實現，這個設計模式在軟體開發都很常見。

iOS 有幾種作法可以讓開發者自己做出符合單例的類別，實作時要特別注意兩件事：

1. **Thread Safety**：多個 thread 同時第一次取用時，也只能建立出一個實體。
2. **防止從外部建立實體**：其他人不能繞過存取點，自己再建立一個。

如果使用 Swift 開發，有官方推薦的做法。

---

## 實作邏輯

實作邏輯是當要取用實體時，判斷實體是否已存在：不存在時初始化一個並回傳；已存在時，直接回傳已存在的實體。

不論是 Objective-C 或是 Swift，我們都根據語言的特性實作，並防止在 multithread 的情況下，singleton 機制失效，保證 thread safety。

## Singleton 與 Shared Instance

Apple 的框架裡，有很多類別提供了一個共用的實體，例如：

- `UIApplication.shared`
- `UserDefaults.standard`
- `NotificationCenter.default`
- `FileManager.default`

但它們不全是嚴格的 singleton。

- **`UIApplication` 是嚴格的 singleton**：一個 App 只能有一個實體，自己再建立一個會直接丟出例外。
- **`UserDefaults`、`NotificationCenter`、`FileManager` 是「shared instance」模式**：它們提供一個方便取用的共用實體，但你仍然可以自己建立新的實體，例如 `UserDefaults(suiteName:)` 用來和 App Extension 共用資料、`NotificationCenter()` 建立獨立的通知中心、`FileManager()` 搭配自己的 delegate 使用。

所以，實作自己的類別時，要先想清楚需要的是哪一種：

- 真的「只能有一個」（例如代表整個 App 狀態的物件），就要實作嚴格的 singleton，阻止外部建立實體。
- 只是「大部分時候用同一個就好」，提供一個 `shared` 就夠了，不需要阻止外部建立，也比較方便寫測試。

---

## 實作程式碼

### 使用 Objective-C 實作

#### 初步實作單例

實作的程式碼其實很簡短，但是在 multithread 的情況下，這種作法會失效：如果實體還沒有建立完成，就有多個 thread 同時進入，每個 thread 都會建立實體，singleton 機制就失效了。

<script src="https://gist.github.com/KennC/7c20483051029c24e07a81c312a7fa7a.js"></script>

#### 讓單例保持 Thread Safety

在 Objective-C 中，我們可以利用 GCD 的 `dispatch_once` 保護 singleton 機制。`dispatch_once` 裡的 block 在整個 App 執行期間只會執行一次，即使多個 thread 同時呼叫也一樣。我們把實體初始化放在 block 裡面，就可以確保 singleton 機制不會失效。

<script src="https://gist.github.com/KennC/749da1537d59fd8e2b32efd7a502851c.js"></script>

#### 防止從外部建立實體

另外，由於 Objective-C 有許多建立實體的方式，我們也嘗試使用幾種方式來取得 `APIService` 實體，並檢查是否為同一個：

<script src="https://gist.github.com/KennC/c6663fe3a95a5e8bbb6b02b15c198aee.js"></script>

我們發現使用 `alloc` 或是 `new` 取得的實體，都是新的實體，只有使用存取點 `[APIService sharedInstance]` 拿到的才會是同一個實體。

**早期的做法：覆寫 `allocWithZone:`**

網路上常見的做法是覆寫 `allocWithZone:`，讓 `alloc` 也回傳同一個實體：

<script src="https://gist.github.com/KennC/8036df6d93898931b1cb9877f1501d24.js"></script>

這樣不論是用 `sharedInstance`、`alloc` 或 `new`，取得的都是同一個實體。但這個做法有副作用：`[[APIService alloc] init]` 雖然拿到同一個實體，**`init` 卻會再被執行一次**。如果 `init` 裡有初始化屬性，singleton 原本的狀態就會被重設，而且呼叫的人完全不會察覺。

**建議的做法：`NS_UNAVAILABLE`**

比較好的做法，是在編譯時期就禁止外部使用 `init` 與 `new`。在 header 裡宣告：

```objc
// APIService.h
@interface APIService : NSObject

+ (instancetype)sharedInstance;

- (instancetype)init NS_UNAVAILABLE;
+ (instancetype)new NS_UNAVAILABLE;

@end
```

因為 `init` 被標記為不可使用，類別內部改用一個私有的初始化方法：

```objc
// APIService.m
@implementation APIService

+ (instancetype)sharedInstance {
    static APIService *instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] initPrivate];
    });
    return instance;
}

- (instancetype)initPrivate {
    return [super init];
}

@end
```

這樣其他人寫 `[[APIService alloc] init]` 或 `[APIService new]` 時，會直接編譯失敗，只能透過 `sharedInstance` 取得實體。

### 使用 Swift 實作

#### 直接把 Objective-C 改成 Swift（已無法使用）

早期有人會直接將 Objective-C 的寫法翻譯成 Swift：

<script src="https://gist.github.com/KennC/8c6e032bc97cf9cde6d45b4dfb2d59cc.js"></script>

但這只能在 Swift 1、2 使用。**從 Swift 3 開始，`dispatch_once` 已經從 Swift 中移除**，這段程式碼現在無法編譯。而且 Swift 本身就有更簡潔的做法，不需要自己寫。

#### Swift 1.0 時代的寫法

<script src="https://gist.github.com/KennC/06a476ea732ace6872b840a3e5681c23.js"></script>

這裡多包了一層 `struct`，是因為 Swift 1.2 以前，class 還不支援 `static` 儲存屬性，只能借用 struct 的 static 屬性。

我們會發現沒有使用 `dispatch_once`，因為在 <a href="https://developer.apple.com/swift/blog/?id=7" target="_blank" rel="noopener noreferrer">Swift blog</a> 有這一段說明：

> The lazy initializer for a global variable (also for static members of structs and enums) is run the first time that global is accessed, and is launched as dispatch_once to make sure that the initialization is atomic.

也就是說，**Swift 保證全域變數與 static 屬性的初始化，只會在第一次存取時執行一次，而且是 thread-safe 的**。這篇 blog 是 2014 年的文章，現在底層的實作已經不是 `dispatch_once`，但這個保證沒有改變，所以我們不需要自己處理。

#### 官方推薦的 Swift 寫法

Apple 的官方文件 <a href="https://developer.apple.com/documentation/swift/managing-a-shared-resource-using-a-singleton" target="_blank" rel="noopener noreferrer">Managing a Shared Resource Using a Singleton</a> 非常清楚的告知開發者，只要使用 `static let` 即可：

<script src="https://gist.github.com/KennC/94001b5a60ff43f4a937f48888553641.js"></script>

若初始化時有其他設定要一起做，就使用 closure 的寫法：

<script src="https://gist.github.com/KennC/082e6ff72f5b66dbefe6b7001468909a.js"></script>

#### 加上 private init

到這裡，已經不會出現多個 thread 建立出多個實體的問題。但其他人仍然可以寫 `APIService()` 自己建立實體，所以還不算嚴格的 singleton。

將 `init` 宣告為 `private`，在其他地方建立實體時，就會出現編譯錯誤，原理與上方 Objective-C 的 `NS_UNAVAILABLE` 相同：

<script src="https://gist.github.com/KennC/d662238b0c33dd18d6af36dbef402c15.js"></script>

如果你需要的是嚴格的 singleton，這一步是必要的；如果只是 shared instance，就可以不加。

---

## Singleton 的 Thread Safety 只保證「建立」

要特別注意，上面所有的做法，保證的都是「**只會建立一個實體**」。但 singleton 裡面的**狀態**並不會因此變成 thread-safe。

```swift
final class Counter {
    static let shared = Counter()
    var value = 0
    private init() {}
}

DispatchQueue.concurrentPerform(iterations: 1000) { _ in
    Counter.shared.value += 1 // data race
}
```

多個 thread 同時讀寫 `Counter.shared.value`，一樣會有 data race。Singleton 是全域共用的物件，反而更容易被多個 thread 同時存取。保護共享資料的方法（serial queue、barrier、`actor`），可以參考 [iOS 並行處理：從 GCD 到 Swift Concurrency]({% post_url 2025-11-01-gcd %})。

## Swift 6 的 Singleton

在 Swift 6 的嚴格並行檢查下，上面 `Counter` 的寫法會直接**編譯錯誤**：`static let shared` 是全域共用的，如果它的型別不是 `Sendable`，編譯器會認為它可能被多個 thread 同時存取。

常見的解法有三種：

**1. 沒有可變狀態：符合 `Sendable`**

如果 singleton 所有的屬性都是 `let`，且型別都是 `Sendable`，可以宣告成 `final class` 並符合 `Sendable`：

```swift
final class APIService: Sendable {
    static let shared = APIService()
    let baseURL = URL(string: "https://api.example.com")!
    private init() {}
}
```

**2. 只在 main thread 使用：`@MainActor`**

和 UI 相關的狀態，直接限定在 main thread 存取：

```swift
@MainActor
final class AppState {
    static let shared = AppState()
    var isLoggedIn = false
    private init() {}
}
```

**3. 需要在多個 thread 存取可變狀態：`actor`**

```swift
actor TokenStore {
    static let shared = TokenStore()
    private var token: String?

    func update(_ newToken: String) {
        token = newToken
    }

    func current() -> String? {
        token
    }
}

await TokenStore.shared.update("abc")
```

`actor` 會保證內部的狀態同一時間只會被一個任務存取，從外面呼叫時需要加上 `await`。
