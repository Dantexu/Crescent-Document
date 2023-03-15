# 开发指南

### Crescent SDKs

Crescent钱包支持多平台SDK，方便开发者在各种平台上便捷接入Crescent钱包。

| **SDK**     | **状态** | **开发文档**                              |
| ----------- | ------ | ------------------------------------- |
| Android SDK | 已完成    | [开发文档](kai-fa-zhi-nan.md#android-sdk) |
| iOS SDK     | 已完成    | [开发文档](kai-fa-zhi-nan.md#ios-sdk)     |
| Unity SDK   | 开发中    | 开发中                                   |

### Android SDK

#### **安装和使用**

1. 将提供的crescent-sdk.aar放到项目的libs文件夹下。
2. 初始化sdk：

```javascript
CrescentConfigure config = new CrescentConfigure();
config.style  =  “”;  //定制样式
CrescentSdk.getInstance().init(config)；
```

3. 连接钱包

```javascript
CrescentSdk.getInstance().connect(this, new ConnectCallback() {
    @Override
    public void onConnectSuccess(UserInfo info) {
String email = info.email
String address = info.address
    }


	@Override
	    public void onConnectFaill() {
	    }
	});
```

4. 直接发送交易

```javascript
    // TransactionInfo相关类型定义
	public class TransactionInfo {
	    private String from;
	    private String to;
	    private String value;
	    private String data;
	}
 
	//交易前判断是否交易成功
	if (CrescentSdk.getInstance().isConnected()) {
	TransactionInfo info = new TransactionInfo(from, to, value, data);
	CrescentSdk.getInstance().sendTransaction(info, 
	new  TransactionCallback() {
	                 @Override
	                 public void onSendSuccess(TransactioinResult result) {
	String hash = result.hash;
	                 }
	 
	                 @Override
	                 public void onSendFail() {
                      }
	});
	}
```

5. 钱包断开连接

```javascript
CrescentSdk.getInstance().disconnect();
```

### iOS SDK

#### **安装和使用**

1. 将提供的CrescentSDK.framework添加到项目
2. 初始化sdk:

```swift
var config = CrescentConfigure()
     config.style = "" //定制样式
     CrescentSDK.config(configure: config)
```

3. 连接钱包

```swift
CrescentSDK.connect(connectSuccessBlock: { userinfo in
      	 let email = userinfo.email;
         let address = userinfo.address;
     }, connectFailBlock: {
         print("connectFailBlock")
    })
```

4. 直接发送交易

```swift
var tx = TransactionInfo()
    tx.from = ""
    tx.to = ""
    tx.value = ""
    tx.data = "";
    CrescentSDK.sendTransaction(info: tx, sendSuccessBlock: { transactionResult in
    		let hash = transactionResult.hash;
    }, sendFailBlock: {
         print("sendFailBlock")
    })
```

5. 钱包断开连接

```swift
CrescentSDK.disconnect()
```

### ****
