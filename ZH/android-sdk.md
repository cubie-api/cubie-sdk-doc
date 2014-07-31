# Cubie SDK for Android

## 開始

### 建立一個新的 Cubie App

到 [Cubie Developer 管理介面][1] 註冊一個開發帳號，成功後登入後點擊 `Create New App` 創建一個新的 應用程式，名稱為 `Demo`

按 `Details...` 修改應用程式資訊，在 Android 的 App Package 欄位輸入 `com.cubie.sdk.demo`

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

在 Eclipse 中選擇 File -> Import.. -> Android -> Existing Android Code Into Workspace

![Eclipse File Import Android][4]

瀏覽到 cubie-openapi 目錄，下方的 Projects 選擇 `cubie-openapi-demo` 和 `cubie-openapi-sdk`

![Import Projects][5]

匯入後打開 `cubie-openapi-demo` 的 `res/values/strings.xml`

### 執行範例程式

到 [Cubie Developer 管理介面][3] 複製中 App Details 的 App Key

假設 App Key 為 `abcdefghijklmnopqrstu`
把 `strings.xml` 的 `cubie_app_key` 和 `cubie_return_url_scheme` 改成
```
    <string name="cubie_app_key">abcdefghijklmnopqrstu</string>
    <string name="cubie_return_url_scheme">cubie-abcdefghijklmnopqrstu</string>
```

執行 cubie-openapi-demo 看否能登入並發送訊息

## 登入

### strings.xml

參考上方 **執行範例程式** 中設定 strings.xml 方法修改 `cubie_app_key` 和 `cubie_return_url_scheme`

### AndroidManifest.xml

假設程式有兩個畫面，一個是起始畫面 LoginActivity，一個是登入後才能使用的 MainActivity

```
<activity android:name="com.example.demo.LoginActivity" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity android:name="com.example.demo.MainActivity" >
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />

        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data
            android:host="@string/cubie_return_url_host"
            android:scheme="@string/cubie_return_url_scheme" />
    </intent-filter>
</activity>
```

最後還要定義一個 SDK 和 Cubie 間溝通的 activity（此 Activity 已包含在 SDK 中，只需在 AndroidManifest.xml 中定義，你的程式裹不需要真的有這個當案）

```
<activity android:name="com.cubie.openapi.sdk.LoginCubieActivity" />
```

### 詢問使用者是否允許存取 Cubie 資訊

在 LoginActivity 中放一個登入按鈕然後在 `OnClickListener` 中呼叫以下片斷就會打開 Cubie 請求使用同意你的應用程式存取他在 Cubie 的資訊

```
Session.getSession().open(this, new SessionCallback()
{
    @Override
    public void onClose()
    {
    }

    @Override
    public void onOpen()
    {
        goToMainActivity();
    }
});
```

在 `LoginActivity` 的 `onResume` 時可以使用 `Session.init(this, sessionCallback)` 直接判斷是否已經登入，如果是的話就無需呈現登入畫面，直接切換到 `MainActivity` 中；同樣地，在 `MainActivity` 的 `onResume` 時也應該呼叫 `Session.init(this, sessionCallback)`，當使用者的 Session 無效時，`onClose` 會被呼叫，此時應把使用者導引回去 `LoginActivity`。

## 取得所用者的個人資訊

使用者必需已經同意讓你的應用程式存取他的個人資訊，而且權限是有效的（權限會有使用期限，會自動更新，但太久沒用會變回無效），可以用 `Session.getSession().isOpen()` 判斷。接下來可參考以下程式片斷：

```
CubieService.requestProfile(session, new CubieServiceHandler<CubieProfile>(CubieProfile.class)
{
    @Override
    public void onException(final IOException e)
    {
        // display error msg
    }

    @Override
    public void onFailure(final CubieServiceError cubieServiceError)
    {
        // display error msg
    }

    @Override
    public void onSuccess(final CubieProfile profile)
    {
        nameView.setText(profile.getNickname());
        Picasso.with(getActivity()).load(profile.getIconUrl()).into(iconView);
    }
});
```

`CubieService.requestProfile` 會向 Cubie 伺服器要求使用者登入資訊，參數 `CubieServiceHandler<CubieProfile>` 負責處理伺服器回傳所收到的結果。   
`onException` 是當發生網路連線問題時會被呼叫。   
`onFailure` 則是連線成功但未能獲得使用者的資訊，可能是由於存取權限已變成無效。   
`onSuccess` 是真正能拿到使用者的個人資訊時會被呼叫，此時可以把使用者的名字和頭像呈現於畫面中。   

## 取得好友名單

## 傳送訊息

用 `Cubie.createMessageBuilder(context)` 產生一個 `CubieMessageBuilder`，每一則訊息中由以下 4 種元素所組成：
1. 文字：用 `setText(String text)` 去設定純文字內容
2. 圖片：用 `setImage(String url, int width, int height)` 去設定圖片的網址（必需要是能公開下載）以及寬高（單位是 dp ）
3. 應用程式連結：用 `setAppLink(String linkText)` 設定連結文字
4. 應用程式按鈕：用 `setAppButton(String buttonText)` 設定按鈕上的文字

當使用應用程式連結或應用程式按鈕來打開你的應用程式時，可以傳入額外的參數來決定使用者點開後的行為，方式是傳入第二個參數 `ActionParams` 如下：

```
builder.setAppButton("open gift", new ActionParams().withExecuteParam("gift_id=1234"));
```

讀取的方式是在 onCreate 時或 onNewIntent 時（activity 已存在）擷取 intent 的 data

```
private void consumeExecuteParams()
{
    String giftId = Cubie.resolveExecuteParams(getIntent(), "gift_id");
    if (Strings.isBlank(giftId))
    {
        return;
    }
    Toast.makeText(getActivity(), giftId, Toast.LENGTH_SHORT).show();
}
```

另外，假如收到訊息的好友使用的是 Android 而又尚未安裝你的應用程式時，可以指定點開訊息切換到 Google Play 並安裝你的程式後第一次打開時會收到額外參數，方法如下：

```
builder.setAppButton("open gift", new ActionParams().withMarketParam("via=cubie"));
```

讀取此參數的方式是要定義一個處理 `action` 為 `com.android.vending.INSTALL_REFERRER` 的 `BroadcastReceiver`，然後用 `intent.getStringExtra("referrer")` 中的取得，詳情可參考 http://stevemiller.net/ReferrerTest/

## 常見錯誤

#### 發生 CubieAppKeyNotFoundException

請確認 `strings.xml` 有定義 `cubie_app_key`

#### 發生 CubieLoginActivityNotDefinedException

請確認 `AndroidManifest.xml` 有定義

    ```
    <activity android:name="com.cubie.openapi.sdk.LoginCubieActivity" />
    ```
    
#### 發生 CubieNoActivityHandlingReturnUrlDefinedException

請確認 `AndroidManifest.xml` 有定義

    ```
    <activity android:name="com.example.demo.MainActivity" >
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
    
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
    
            <data
                android:host="@string/cubie_return_url_host"
                android:scheme="@string/cubie_return_url_scheme" />
        </intent-filter>
    </activity>
    ```

#### cannot find app by return url

請確認 `strings.xml` 的 `cubie_return_url_scheme` 設定正確

#### invalid app key

請確認 `strings.xml` 的 `cubie_app_key` 設定正確

#### invalid app signature

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

  [1]: http://dev.cubie.com/
  [2]: https://lh3.googleusercontent.com/-qRKykal36OE/U7oWRsdCQRI/AAAAAAAAAZ4/4z2doxKQDFw/s0/Screen%252520Shot%2525202014-07-07%252520at%25252011.34.39%252520AM.png "Screen Shot 2014-07-07 at 11.34.39 AM.png"
  [3]: http://dev.cubie.com/
  [4]: https://lh4.googleusercontent.com/-naK8Y_91StM/U7oT-1dSuSI/AAAAAAAAAZs/Tx2Upz7zKYA/s0/Screen%252520Shot%2525202014-07-07%252520at%25252011.27.08%252520AM.png "Screen Shot 2014-07-07 at 11.27.08 AM.png"
  [5]: https://lh5.googleusercontent.com/-JkhFsizTs3s/U7oaZsGIggI/AAAAAAAAAaE/eH-vLBlUnsI/s0/Screen%252520Shot%2525202014-07-07%252520at%25252011.55.07%252520AM.png "Screen Shot 2014-07-07 at 11.55.07 AM.png"