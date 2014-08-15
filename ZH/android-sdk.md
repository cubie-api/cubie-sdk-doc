# Cubie SDK for Android

## 如何執行範例程式

### 建立一個新的 Cubie App

到 [Cubie Developer 管理介面][1] 註冊一個開發帳號，成功後登入後點擊 `Create New App` 創建一個新的 應用程式，名稱為 `Demo`

按 **Details...** 修改應用程式資訊，在 Android 的 App Package 欄位輸入 `com.cubie.sdk.demo`

App Signatures 的產生方式如下

### 產生 App Signature

在 Eclipse 中選取 Eclipse -> Preferences... -> Android -> Build 找出所使用 debug keystore 的位置，假如有用 Custom debug keystore 的話就以這個為主，否則就是用 Default debug keystore

![Eclipse Preferences Android Build][2]

#### 以 Windows 為例
假設上述 keystore 的路徑為 C:\Users\deveoper\.android\debug.keystore 
在命令提示視窗中輸入以下指令：（`keytool` 的位置是在 `%JAVA_HOME%\bin` 裹）

```
keytool -exportcert -alias androiddebugkey -keystore C:\Users\deveoper\.android\debug.keystore | openssl sha1 -binary | openssl base64
```

#### 以 Mac OSX 為例
假設上述 keystore 的路徑為 /Users/developer/.android/debug.keystore 
在終端機中輸入以下指令：（`keytool` 的位置是在 `$JAVA_HOME/bin` 裹）

```
keytool -exportcert -alias androiddebugkey -keystore /Users/developer/.android/debug.keystore | openssl sha1 -binary | openssl base64
```

輸入密碼 `android` （沒有問 keystore 的密碼話就是 keystore 的路徑不對）後把印出來的結果複製貼回 [Cubie Developer 管理介面][3] 中 App Details 的 Android App Signatures 欄位中 

### 匯入 SDK 和範例程式到 eclipse

下載最新版的 cubie-sdk-android-a.b.c.zip 和 cubie-sdk-android-demo-a.b.c.zip

![Eclipse File Import Android][4]

解開後在 Eclipse 中選擇 File -> Import.. -> Android -> Existing Android Code Into Workspace 匯入 `cubie-sdk-android-demo` 和 `cubie-sdk-android`

### 執行範例程式

打開 `cubie-sdk-android-demo` 的 `res/values/strings.xml` 把其中 `[cubie_app_key]` 的地方用 [Cubie Developer 管理介面][5] 中 App Details 裹的 App Key 取代

假設 App Key 為 `abcdefghijklmnopqrstu`
把 `strings.xml` 的 `cubie_app_key` 和 `cubie_return_url_scheme` 改成

```
    <string name="cubie_app_key">abcdefghijklmnopqrstu</string>
    <string name="cubie_return_url_scheme">cubie-abcdefghijklmnopqrstu</string>
```

執行 cubie-sdk-android-demo 看否能登入並發送訊息

----------

## ＳＤＫ使用說明

### 前置作業

參考上方 **建立一個新的 Cubie App** ，然後在 Eclipse 建立一個新的專案。由於 Cubie SDK 是 Android library project，想在新專案使用的話，必須在專案設定中指定 SDK 的位置。匯入 `cubie-sdk-android` 到 eclipse 後，右鍵點選新專案進入 properties 後，在左邊的列表選擇 Android，然後在右下方按 **Add...** 選擇 `cubie-sdk-android`。

![library project][6]

參考上方 **執行範例程式** 中設定 `strings.xml` 的方法，修改 `cubie_app_key` 和 `cubie_return_url_scheme`，並且在 `AndroidManifest.xml` 中加入一個 SDK 和 Cubie Server 間溝通的 `ConnectCubieActivity`（已包含在 SDK 中，你的程式裹不需有這個檔案）：

```
<activity android:name="com.cubie.openapi.sdk.ConnectCubieActivity" />
```

並且定義 app key 為 `strings.xml` 中的 `cubie_app_key`：

```
<meta-data android:name="com.cubie.openapi.sdk.AppKey" 
           android:value="@string/cubie_app_key" />
```

如果打算要傳送可以讓對方開啟應用程式的訊息，還要在你想被開啟的 activity 中加入以下 intent filter：

```
<activity android:name="..." >
    <intent-filter>
        <action   android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data     android:host="@string/cubie_return_url_host"
                  android:scheme="@string/cubie_return_url_scheme" />
    </intent-filter>
</activity>
```

### 連結 Cubie

為了方便使用 Cubie SDK 的功能，Activity 可以全部繼承 `CubieBaseActivity`，這個 Activity 已經實作好和 Cubie 連結的一般邏輯，包括呼叫 `connect()` 和 `disconnect()` 來讓使用者連結或解除連結 Cubie，假如使用者已連結 Cubie 的話，`onSessionOpen()` 會被呼叫，這時候可以使用 `Cubie.getService()` 去取得使用者的資訊或傳送訊息等服務；假如使用者斷開連結的話，`onSessionClose()` 會被呼叫，此時應把使用者導引回到一個尚未連結 Cubie 時的畫面。

如果不想要繼承 `CubieBaseActivity`（可能因為已繼承別的 Activity）可以參考 `CubieBaseActivity` 中如何使用 `CubieActivityHelper`。

### 取得使用者的個人資訊
參考範例：

```
Cubie.getService().requestMe(new CubieServiceCallback<CubieUser>() {
    public void done(CubieUser user, CubieException e) { 
        updateUI(user);
    }
});
```

`CubieService.requestMe` 向 Cubie Server 要求使用者的頭像網址、暱稱等個人資訊，成功後回傳一 `CubieUser` 物件。   

### 取得好友名單
參考範例：
先準備一個 `CubieFriendList` 的成員變數

```
private CubieFriendList cubieFriendList;`
```

每次呼叫 `requestFriends` 時帶入 `cubieFriendList`（一開始為 null）會向 Cubie server 要更多的好友，結果可以在 callback 中的 `updatedFriendList` 拿到，此時應把它更新到 `cubieFriendList`，下次呼叫時再次傳入。

```
Cubie.getService().requestFriends(cubieFriendList, pageSize, new CubieServiceCallback<CubieFriendList>() {
    public void done(CubieFriendList updatedFriendList, CubieException e) {
        cubieFriendList = updatedFriendList;
        updateUI();
    }
});
```
* pageSize表示向Cubie Server要求的朋友個數

* CubieFriendList 包含兩個成員函式：
    * `getAllFriends()`：取得當前已載入的朋友列表（`List<CubieFriend>`）
    * `hasMore()`：判斷朋友列表是否完全載入

### 傳送訊息
參考範例：

```
Cubie.getService().sendMessage(receiverUid,
                               cubieMessage,
                               new CubieServiceCallback<CubieSendAck>() {
    public void done(CubieSendAck sendAck, CubieException e) {...}
});
```
- `receiverUid`: 欲傳送對象的 uid
可由好友名單中選取ㄧ CubieFriend friend, 透過 `friend.getUid()` 取得 uid。

- `cubieMessage`: 欲傳送的訊息
每一則訊息是由 **文字**，**圖片**，**應用程式連結**，**應用程式按鈕** 四種元素混搭組成，建立一則 `CubieMessage` 的方法是使用 `CubieMessageBuilder` 的設定函式：

    * `setNotification(String notification)`: 設定提示訊息，**請注意此欄位必填！** 請參考下方圖片
    * `setText(String text)`: 設定純文字內容
    * `setImage(String text)`: 設定圖片的網址（必需要是能公開下載）
    * `setAppLink(String text)`: 設定連結文字
    * `setAppLink(String text, CubieMessageActionParams actionParams)`: 設定連結文字，傳入 `actionParams` 來決定使用者點開後的行為
    * `setAppButton(String linkText)`: 設定按鈕上的文字
    * `setAppButton(String linkText, CubieMessageActionParams actionParams)`: 設定按鈕上的文字，傳入 `actionParams` 來決定使用者點開後的行為

Notification 出現的地方包括 status bar, notification panel, 還有聊天室列表中

![notification on status bar][7]

![notification panel][8]

![chat room last text][9]

### 將交易記錄告知 Cubie Server
參考範例：

```
Cubie.getService().createTransaction(request, new CubieServiceCallback<Void>() {
    public void done(Void object, CubieException e) {...}
});
```

request 為ㄧ `CubieTransactionRequest` 物件，參考如下：

```
CubieTransactionRequest request = new CubieTransactionRequest(orderId,
    productId,
    itemPrice,
    purchaseTime,
    extraData);
```

如果是使用 Google Play In-app Billing 的話，呼叫時機應在使用者已完成購買，程式收到 Google Play 回傳結果的 `onActivityResult` 中，請參考 [Implementing In-app Billing 文件中 Purchasing an Item ][10] 內容敘述

* `orderId`: [`INAPP_PURCHASE_DATA`][11] 的 orderId
* `productId`: [`INAPP_PURCHASE_DATA`][12] 的 productId
* `itemPrice`: 物品價格（可用 [getSkuDetails()][13] 取得 `price`，或寫死在程式中）
* `purchaseTime`: [`INAPP_PURCHASE_DATA`][14] 的 `purchaseTime`
* `extraData`: 額外資料，若無額外資料請填 null

## 常見錯誤

### 發生 CubieAppKeyNotDefinedException

請確認 `AndroidManifest.xml` 有定義 

```
<meta-data android:name="com.cubie.openapi.sdk.AppKey" 
           android:value="@string/cubie_app_key" />
```

### 發生 ConnectCubieActivityNotDefinedException

請確認 `AndroidManifest.xml` 有定義

```
<activity android:name="com.cubie.openapi.sdk.ConnectCubieActivity" />
```
    
### cannot find app by return url

請確認 `strings.xml` 的 `cubie_return_url_scheme` 設定正確，並且有 activity 去處理以下這個 intent filter

```
<activity android:name="..." >
    <intent-filter>
        <action   android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data     android:host="@string/cubie_return_url_host"
                  android:scheme="@string/cubie_return_url_scheme" />
    </intent-filter>
</activity>
```

### invalid app key

請確認 `strings.xml` 的 `cubie_app_key` 設定正確

### invalid app signature

請參 **產生 App Signature** 的章節 ，另外也可在程式裹面用以下方式印出 app signature，可用來和 keytool 所產生的結果比對是否一致

```
private void printSignature(final Context context) throws Exception
{
    PackageInfo packageInfo = context.getPackageManager()
        .getPackageInfo(context.getPackageName(), PackageManager.GET_SIGNATURES);
    MessageDigest md;
    md = MessageDigest.getInstance("SHA");
    md.update(packageInfo.signatures[0].toByteArray());
    final String keyHash = new String(Base64.encode(md.digest(), Base64.NO_WRAP));
    System.out.println("signature hash:" + keyHash);
}
```


  [1]: https://dev.cubie.com/developer/
  [2]: https://lh3.googleusercontent.com/-qRKykal36OE/U7oWRsdCQRI/AAAAAAAAAZ4/4z2doxKQDFw/s0/Screen%252520Shot%2525202014-07-07%252520at%25252011.34.39%252520AM.png "Screen Shot 2014-07-07 at 11.34.39 AM.png"
  [3]: http://dev.cubie.com/
  [4]: https://lh4.googleusercontent.com/-naK8Y_91StM/U7oT-1dSuSI/AAAAAAAAAZs/Tx2Upz7zKYA/s0/Screen%252520Shot%2525202014-07-07%252520at%25252011.27.08%252520AM.png "Screen Shot 2014-07-07 at 11.27.08 AM.png"
  [5]: http://dev.cubie.com/
  [6]: https://lh6.googleusercontent.com/-IS73m5sDuIQ/U-BEt3uSC4I/AAAAAAAAAas/9LrtPLnKULk/s0/Screen%252520Shot%2525202014-08-05%252520at%25252010.35.11%252520AM.png "library project"
  [7]: https://lh4.googleusercontent.com/-1vYacclNhlk/U-3IRookA8I/AAAAAAAAAbo/94-VsuX3qvc/s0/device-2014-08-15-163431.png "notification on status bar"
  [8]: https://lh4.googleusercontent.com/-Fl0GFlEP1VQ/U-3IXSGFJSI/AAAAAAAAAbw/3swZynOa4j8/s0/device-2014-08-15-163457.png "notification panel"
  [9]: https://lh5.googleusercontent.com/-gERUtslPBCY/U-3IdmjRgJI/AAAAAAAAAb4/c2auKYNczXk/s0/device-2014-08-15-163512.png "chat room last text"
  [10]: http://developer.android.com/google/play/billing/billing_integrate.html#Purchase
  [11]: http://developer.android.com/google/play/billing/billing_reference.html#getBuyIntent
  [12]: http://developer.android.com/google/play/billing/billing_reference.html#getBuyIntent
  [13]: http://developer.android.com/google/play/billing/billing_reference.html#getSkuDetails
  [14]: http://developer.android.com/google/play/billing/billing_reference.html#getBuyIntent