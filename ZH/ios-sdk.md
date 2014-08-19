# Cubie SDK for iOS

## 如何執行範例程式 

### 建立一個新的 Cubie App

到 [Cubie Developer 管理介面][1] 註冊一個開發帳號，成功後登入後點擊 `Create New App` 創建一個新的 應用程式，名稱為 `Demo`

按 **Details...** 修改應用程式資訊，在 iOS 的 App Bundle ID 欄位輸入 `com.cubie.openapi.demo2`

### 執行範例程式

下載最新版的 `cubie-sdk-ios-a.b.c.zip` 和 `cubie-sdk-ios-demo-frameworks-a.b.c.zip`，解開在同一個目錄，並把 `cubie-sdk-ios-a.b.c` 改名為 `cubie-sdk-ios`，用 XCode 打開 `cubi-sdk-ios-demo-frameworks-a.b.c` 下的 demo2.xcodeproj

修改 `demo2-Info.plist` 把以下兩個 Key 的 `[cubie_app_key]` 用 [Cubie Developer 管理介面][2] 中 App Details 裹的 App Key 取代 

* `CubieAppKey` 
* `URL types` -> `Item 0` -> `URL Schemes` -> `Item 0` 

![Info.plist][3]

在 iOS device 上執行 demo2 看否能登入並發送訊息

## ＳＤＫ使用說明

### 前置作業

參考上方 **建立一個新的 Cubie App** ，然後在 XCode 建立一個新的專案。

參考上方 **執行範例程式** 修改 `Info.plist` 加入新的 key `CubieAppKey` 和 `URL Schemes`

Cubie SDK iOS 版本可以使用 [CocoaPods][4] 的方式來管理，或是直接加入 [Frameworks][5]

#### CocoaPods

下載 `cubie-sdk-ios-a.b.c.zip` 後解壓並放在和專案同一層的目錄，把 `cubie-sdk-ios-a.b.c` 改名為 `cubie-sdk-ios`

[安裝 CocoaPods][6] 後，在專案目錄中加入一個名為 `Podfile` 的檔案，內容如下：

```
pod 'CubieSDK', :path => '../cubie-sdk-ios'
```

執行 `pod install` 後就可以用 XCode 打開產生出來 `[your project name].xcworkspace` 檔案開始使用 Cubie SDK

#### Frameworks

下載 `cubie-sdk-ios-a.b.c.zip` 後解壓並放在和專案同一層的目錄，把 `cubie-sdk-ios-a.b.c` 改名為 `cubie-sdk-ios`

在 XCode 打開新專案後選擇 **TARGETS** -> **General** 在 **Linked Frameworks and Libraries** 子頁中按 **+** 號，在跳出來的對話框中按 **Add Other...**

瀏覽到 cubie-sdk-ios/Frameworks 下選擇 `CubieSDK.framework`，此外假如你的專案中沒有 [AFNetworking][7], [CocoaLumberjack][8], [JSONKit][9] 的話，也請一併選擇這三個 frameworks

![frameworks][10]

這樣就可以開使用 CubieSDK

### 連結 Cubie

為了方便使用 Cubie SDK 的功能，`UIViewController` 的子類別可以使用 `UIViewController+CBSession`，這個 category 已經實作好和 Cubie 連結的一般邏輯，包括呼叫 `connect:` 和 `disconnect:` 來讓使用者連結或解除 Cubie 連結。另外 `UIViewController` 的子類別應在 `viewDidAppear` 中呼叫 `onCBSessionOpen:onCBSessionClose:` 假如使用者已連結 Cubie 的話，onOpen 會被呼叫，這時候可以使用 `CBService` 去取得使用者的資訊或傳送訊息等服務；假如使用者斷開連結的話，`onClose` 會被呼叫，此時應把使用者導引回到一個尚未連結 Cubie 時的畫面。

為了能接收從 Cubie 程式回傳過來的連結資訊，必須要在你的 `UIApplicationDelegate` 中 override `application:openURL:sourceApplication:annotation:` 如下：

```
- (BOOL) application:(UIApplication*) application openURL:(NSURL*) url sourceApplication:(NSString*) sourceApplication annotation:(id) annotation
{
    return [Cubie handleOpenUrl:url sourceApplication:sourceApplication];
}
```

此外，Cubie SDK 有提供連結 Cubie 時使用的按鈕，可以直接使用以下方法獲得 `UIButton`：

```
UIButton* connectButton = [Cubie buttonWithCubieStyle];
```

![connect with cubie button][11]

### 取得使用者的個人資訊
參考範例：
```
[CBService requestMe:^(CBUser* user, NSError* error) {
    if (!error) {
        // update UI ...
    }
}];
```
向 Cubie Server 要求使用者的頭像網址、暱稱等個人資訊，成功後回傳一 `CBUser` 物件。 
### 取得好友名單
參考範例：
先準備一個 `CBFriendList` 的成員變數

```
@property (nonatomic, strong) CBFriendList* friendList;
```

每次呼叫 `requestFriends` 時帶入 `friendList`（一開始為 null）會向 Cubie server 要更多的好友，結果可以在 callback 中的 `updatedFriendList` 拿到，此時應把它更新到 `friendList`，下次呼叫時再次傳入。
```
[CBService requestFriends:friendList pageSize:pageSize 
    done:^(CBFriendList* updatedFriendList, NSError* error) {
        if (!error) {
            friendList = updatedFriendList;
            // update UI ...
        }
         
}];
```
`pageSize` 表示向 Cubie Server 要求的朋友個數

### 傳送訊息

參考範例：

```
[CBService sendMessage:cbMessage to:friendUid
    done:^(NSError* error) {
        if (!error) {
            // notify user  ...
        }
  }];
```

* `friendUid` : 欲傳送對象的 `uid`
可由好友名單中選取ㄧ `CBFriend`，透過 `friend.uid` 取得 `uid`。

* `cbMessage` : 欲傳送的訊息
每一則訊息是由 **文字**，**圖片**，**應用程式連結**，**應用程式按鈕** 四種元素混搭組成，建立一則 `CBMessage` 的方法是使用 `CBMessageBuilder` 的設定函式：
    * `notification:(NSString*) notification`: 設定提示訊息，**請注意此欄位必填！**
    * `text:(NSString*) text`: 設定純文字內容
    * `image:(NSString*) imageUrl`: 設定圖片的網址（必需要是能公開下載）
    * `appLink:(NSString*) linkText`: 設定連結文字
    * `appLink:(NSString*) linkText action:(CBMessageActionParams*) actionParams`: 設定連結文字，傳入 `actionParams` 來決定使用者點開後的行為
    * `appButton:(NSString*) buttonText`: 設定按鈕上的文字
    * `appButton:(NSString*) buttonText action:(CBMessageActionParams*) actionParams`: 設定按鈕上的文字，傳入 `actionParams` 來決定使用者點開後的行為
    
CBMessage 的四個元素如下圖：

![CBMessage][12]

Notification 出現的地方包括 status bar, notification panel, 還有聊天室列表中

![local notification][13]

![enter image description here][14]

### 將交易記錄告知 Cubie Server
參考範例：
```
[CBService createTransaction:orderId
    itemName:itemName
    currency:currency
    price:itemPrice
    purchaseDate:purchasDate
    extraData:nil
    done:^(NSError* error) {
        if (error) {
            // schedule retry
        }
}];
```
如果是使用 App Store In-App Purchase 的話，呼叫時機應在使用者已完成購買， `paymentQueue:updatedTransactions:` 被呼叫而且 `SKPaymentTransaction.transactionState == SKPaymentTransactionStatePurchased` 時，請參考 [In-App Purchase Programming Guide 中  Delivering Products][15] 內容敘述

* `orderId`: [`SKPaymentTransaction.transactionIdentifier`][16]
* `itemName`: [`SKPaymentTransaction.payment.productIdentifier`][17]
* `currency`: 透過 [`SKProductsRequest`][18] 取得 SKProduct 後的 [`[product.priceLocale objectForKey:NSLocaleCurrencyCode]`][19]
* `itemPrice`: 透過 [`SKProductsRequest`][20] 取得 SKProduct 後的 [`product.price`][21]
* `purchasDate`:[`SKPaymentTransaction.transactionIdentifier`][22]
* `extraData`: 額外資料，若無額外資料請填 nil

## FAQ

### CubieAppKey not defined in Info.plist

請參考 **執行範例程式** 中修改 `Info.plist` 的步驟，確認 `CubieAppKey` 和 [Cubie Developer 管理介面][1] 中的 App Key 一致。

### url scheme cubie-xxxxxxxxxxxxxxx not defined in Info.plist

請參考 **執行範例程式** 中修改 `Info.plist` 的步驟， 確認 `URL types` -> `URL Schemes` 中有一個 item 是 `cubie-[app_key]`，其中 app key 和 `CubieAppKey` 一致。

### invalid app key

請參考 **執行範例程式** 中修改 `Info.plist` 的步驟，確認 `CubieAppKey` 和 [Cubie Developer 管理介面][1] 中的 App Key 一致。

### invalid app signature

請確認 [Cubie Developer 管理介面][1] 中的應用資訊裹 `App Bundle ID` 和你的專案的一致。

### 使用者同意連結 Cubie 後回到程式中，但呼叫 UIViewController(CBSession) 的 connect: 時所帶的 onOpen 沒有被呼叫

### 呼叫 [CBservice sendMessage:to:done:] 時得到 CBMessageIsEmptyError

CBMessage 的 **文字**，**圖片**，**應用程式連結**，**應用程式按鈕** 不能全部為空，另外 `notification` **為必填欄位**。

這參考 **連結 Cubie** 確認有在 `UIApplicationDelegate` 的 `application:openURL:sourceApplication:annotation:` 呼叫 `Cubie` 的 `handleOpenUrl:sourceApplication:`。

  [1]: https://lh5.googleusercontent.com/-GZ09mm_hs0c/U-nQEWT-H4I/AAAAAAAAAbA/CyQHyjMf_og/s0/Screen%252520Shot%2525202014-08-12%252520at%2525204.26.08%252520PM.png "plist"
  [2]: https://github.com/AFNetworking/AFNetworking
  [3]: https://lh5.googleusercontent.com/-GZ09mm_hs0c/U-nQEWT-H4I/AAAAAAAAAbA/CyQHyjMf_og/s0/Screen%252520Shot%2525202014-08-12%252520at%2525204.26.08%252520PM.png "plist"
  [4]: http://cocoapods.org/
  [5]: https://developer.apple.com/technologies/ios/cocoa-touch.html
  [6]: http://guides.cocoapods.org/using/getting-started.html
  [7]: https://github.com/AFNetworking/AFNetworking
  [8]: https://github.com/CocoaLumberjack/CocoaLumberjack
  [9]: https://github.com/johnezang/JSONKit
  [10]: https://lh3.googleusercontent.com/-cS5wDGEKoW8/U-nXRalI9bI/AAAAAAAAAbQ/gAERA829Xtc/s0/Screen%252520Shot%2525202014-08-12%252520at%2525204.48.37%252520PM.png "frameworks"
  [11]: https://lh4.googleusercontent.com/-1gTiIF4wzGA/U_F07qXUxOI/AAAAAAAAAc8/7k0AcUDLA_w/s0/iOS%252520Simulator%252520Screen%252520shot%252520Aug%25252018%25252C%2525202014%25252C%25252011.35.32%252520AM.png "connect with cubie button"
  [12]: https://lh6.googleusercontent.com/-17QcHtVX4NI/U_Ftdw7lbLI/AAAAAAAAAcU/VQMgxlqMkcE/s0/iOS%252520Simulator%252520Screen%252520shot%252520Aug%25252018%25252C%2525202014%25252C%25252010.59.54%252520AM.png "CBMessage"
  [13]: https://lh3.googleusercontent.com/-GiFLZYBUrLE/U_FttN3lNsI/AAAAAAAAAcc/Up69U4NT33A/s0/iOS%252520Simulator%252520Screen%252520shot%252520Aug%25252018%25252C%2525202014%25252C%25252011.03.17%252520AM.png "local notification"
  [14]: https://lh6.googleusercontent.com/-fIhZCXTW0Rs/U_FtywwRCOI/AAAAAAAAAck/-5aT1QwKk8M/s0/iOS%252520Simulator%252520Screen%252520shot%252520Aug%25252018%25252C%2525202014%25252C%25252011.00.34%252520AM.png "chat group last message"
  [15]: https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Chapters/DeliverProduct.html#//apple_ref/doc/uid/TP40008267-CH5-SW4
  [16]: https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/Reference/Reference.html#//apple_ref/occ/instp/SKPaymentTransaction/transactionIdentifier
  [17]: https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentRequest_Class/Reference/Reference.html#//apple_ref/occ/instp/SKPayment/productIdentifier
  [18]: https://developer.apple.com/library/mac/documentation/StoreKit/Reference/SKProductsRequest/Reference/Reference.html
  [19]: https://developer.apple.com/Library/ios/documentation/StoreKit/Reference/SKProduct_Reference/Reference/Reference.html#//apple_ref/occ/instp/SKProduct/priceLocale
  [20]: https://developer.apple.com/library/mac/documentation/StoreKit/Reference/SKProductsRequest/Reference/Reference.html
  [21]: https://developer.apple.com/Library/ios/documentation/StoreKit/Reference/SKProduct_Reference/Reference/Reference.html#//apple_ref/occ/instp/SKProduct/price
  [22]: https://developer.apple.com/library/ios/documentation/StoreKit/Reference/SKPaymentTransaction_Class/Reference/Reference.html#//apple_ref/occ/instp/SKPaymentTransaction/transactionDate