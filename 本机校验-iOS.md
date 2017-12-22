# 1. 开发环境配置 

## 1.1. 环境配置及发布

1. 导入统一认证framework

直接将统一认证`TYRZSDK.framework`拖到项目中

![iOS-1](image\iOS-1.png)

2. 在Xcode中找到`TARGETS-->Build Setting-->Linking-->Other Linker Flags`在这选项中需要添加`-ObjC`

![iOS-4](image\iOS-4.jpg)

3. TARGETS-->Build Setting-->搜索框中搜索"BitCode"选项,并且将该选项的属性设置为NO

![iOS-5](image\iOS-5.jpg)

## 1.2. Hello 统一认证 

本节内容主要面向新接入统一认证的开发者，介绍快速集成统一认证的基本服务的方法。

### 1.2.1. 统一认证登录流程

![](image/19.png)

由流程图可知，业务客户端集成SDK后只需要完成2步集成实现登录

    1.	调用登录接口获取token
    2.	携带token请求登录

### 1.2.2. 统一认证登录集成步骤

**第一步：**

在appDelegate.m中的didFinish函数中添加初始化代码。初始化代码只需要执行一次就可以。

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
     [TYRZBaseApi customInit:APPID appKey:APPKEY];
    return YES;
}
```

**第二步：**

在需要用到登录的地方调用登录接口即可，以下是登录示例

```objective-c
- (void)showImplicitLogin {
    self.waitBGV.hidden = NO;
    [self.waitAV startAnimating];
     __weak typeof(self) weakSelf = self;
    [TYRZLogin loginImplicitly:^(id sender) {
        dispatch_async(dispatch_get_main_queue(), ^{
            weakSelf.waitBGV.hidden = YES;
            [weakSelf.waitAV stopAnimating];
            NSString *resultCode = sender[@"resultCode"];
            self.token = sender[@"token"];
            NSMutableDictionary *result = [NSMutableDictionary dictionaryWithDictionary:sender];
            if ([resultCode isEqualToString:SUCCESSCODE]) {
                result[@"result"] = @"获取token成功";
            } else {
                result[@"result"] = @"获取token失败";
            }
            [self showInfo:result];
        });
    }];
}
```

<div STYLE="page-break-after: always;"></div>

#2. SDK方法描述
## 2.1. 获取token

### 2.1.1 方法描述

开发者向统一认证服务器获取用户身份标识`openId`和临时凭证`token`。</br>

**openId：**每个APP每个手机号码对应唯一的openId。</br>

**临时凭证token：**开发者服务端可凭临时凭证token通过3.1本机号码校验接口对本机号码进行验证。

</br>

**原型**

`TYRZLogin -> getTokenWithAppId`

```objective-c
+ (void)getTokenWithAppId:(NSString *)appid appkey:(NSString *)appkey complete:(void (^)(id sender))complete;;
```



### 2.1.2 参数说明

**请求参数**

| 参数       | 类型            | 说明              | 是否必填 |
| -------- | ------------- | --------------- | ---- |
| appid    | NSString      | 开放平台申请得到的appid  | 是    |
| appkey   | NSString      | 开放平台申请得到的appkey | 是    |
| complete | UAFinishBlock | 登录回调            | 是    |


**响应参数**


| 参数          | 类型         | 说明                                       | 是否必填  |
| ----------- | ---------- | ---------------------------------------- | ----- |
| resultCode  | NSUinteger | 返回相应的结果码                                 | 是     |
| token       | NSString   | 成功时返回：临时凭证，token有效期5min，一次有效             | 成功时必填 |
| openid      | NSString   | 成功时返回：用户身份唯一标识                           | 成功时必填 |
| authType    | NSString   | 认证类型：0:其他；</br>1:WiFi下网关鉴权；</br>2:网关鉴权；</br>3:短信上行鉴权；</br>7:短信验证码登录 | 成功时必填 |
| authTypeDes | NSString   | 认证类型描述，对应authType                        | 成功时必填 |
| desc        | NSString   | 调用描述                                     | 否     |



### 2.1.3 示例

**请求示例代码**


```objective-c
/**
 获取token
 */
- (void)showImplicitLogin {
    self.waitBGV.hidden = NO;
    [self.waitAV startAnimating];
    [TYRZLogin getTokenWithAppId:APPID appkey:APPKEY complete:^(id sender) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.waitBGV.hidden = YES;
            [self.waitAV startAnimating];
            NSString *resultCode = sender[@"resultCode"];
            NSMutableDictionary *result = [NSMutableDictionary dictionaryWithDictionary:sender];
            if ([resultCode isEqualToString:SUCCESSCODE]) {
                result[@"result"] = @"获取token成功";
            }else{
                result[@"result"]= @"获取token失败";
            }
            [self showInfo:result];
        });
    }];
}
```


**响应示例代码**

```
{
    authType = 2;
    authTypeDes = "网关鉴权";
    resultCode = 102000;
    token = STsid00000015087457254472qa7Mh1AAZH1U0xwvoMnKu5XxipjWXWE;
}
```

<div STYLE="page-break-after: always;"></div>

# 3. 平台接口说明

## 3.1. 本机号码校验接口

### 3.1.1. 业务流程

SDK在获取token过程中，用户手机必须在打开数据网络情况下才能获取成功，纯wifi环境下会自动跳转到SDK的短信验证码页面（如果有配置）或者返回错误码。**注：本业务目前仅支持中国移动号码，建议开发者在调用SDK的获取token方法前，判断当前用户手机运营商**

![1](image\1.1.png)

</br>

### 3.1.2. 接口说明

校验用户输入的号码是否本机号码。
应用将手机号码传给统一认证SDK，统一认证SDK向统一认证服务端发起本机号码校验请求，统一认证服务端通过网关或者短信上行获取本机手机号码和第三方应用传输的手机号码进行校验，返回校验结果。</br>

**调用次数说明：**本产品属于收费业务，开发者未签订服务合同前，每天总调用次数有限，详情可咨询商务。

**请求地址：** https://www.cmpassport.com/openapi/rs/tokenValidate

**协议：** HTTPS

**请求方法：** POST+json

**回调地址：**请参考开发者接入流程文档

</br>

### 3.1.3.  参数说明

**请求参数**

| 参数            | 类型     | 层级    | 约束                    | 说明                                       |      |
| ------------- | ------ | ----- | --------------------- | ---------------------------------------- | ---- |
| **header**    |        | **1** | 必选                    |                                          |      |
| version       | string | 2     | 必选                    | 版本号,初始版本号1.0,有升级后续调整                     |      |
| msgId         | string | 2     | 必选                    | 使用UUID标识请求的唯一性                           |      |
| timestamp     | string | 2     | 必选                    | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |      |
| appId         | string | 2     | 必选                    | 应用ID                                     |      |
| **body**      |        | **1** | 必选                    |                                          |      |
| openType      | String | 2     | 否，requestertype字段为0时是 | 运营商类型：</br>1:移动;</br>2:联通;</br>3:电信;</br>0:未知 |      |
| requesterType | String | 2     | 是                     | 请求方类型：</br>0:APP；</br>1:WAP              |      |
| message       | String | 2     | 否                     | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码      |      |
| expandParams  | String | 2     | 否                     | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |      |
| phoneNum      | String | 2     | 是                     | 待校验的手机号码的64位sha256值，字母大写。（手机号码 + appKey + timestamp）（注：“+”号为合并意思） |      |
| token         | String | 2     | 是                     | 身份标识，字符串形式的token                         |      |
| sign          | String | 2     | 是                     | 签名，HMACSHA256( appId +     msgId + phonNum + timestamp + token + version)，输出64位大写字母 （注：“+”号为合并意思，不包含在被加密的字符串中,appkey为秘钥, 参数名做自然排序（Java是用TreeMap进行的自然排序）） |      |
|               |        |       |                       |                                          |      |

**响应参数**

| 参数           | 层级    | 类型     | 约束   | 说明                                       |      |
| ------------ | ----- | :----- | :--- | :--------------------------------------- | ---- |
| **header**   | **1** |        | 必选   |                                          |      |
| msgId        | 2     | string | 必选   | 对应的请求消息中的msgid                           |      |
| timestamp    | 2     | string | 必选   | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |      |
| appId        | 2     | string | 必选   | 应用ID                                     |      |
| resultCode   | 2     | string | 必选   | 规则参见4.1平台返回码                             |      |
| **body**     | **1** |        | 必选   |                                          |      |
| resultDesc   | 2     | String | 必选   | 描述参见4.1平台返回码                             |      |
| message      | 2     | String | 否    | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码      |      |
| expandParams | 2     | String | 否    | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |      |
|              |       |        |      |                                          |      |

</br>

### 3.1.4. 示例

**请求示例**

```
{
  "body": {
    "openType": "1", 
    "requesterType": "1", 
    "message ": "", 
	"expandParams": "",
    "phoneNum": "4526285940b6fa7fef49e1dcb04ee944f41a8745444015daf2771bfb7ad7c800", 
    "token": "", 
    "sign": ""
    }, 
  "header": {
    "msgId ": "61237890345", 
    "timestamp ": "20160628180001165", 
    "version ": "2.0", 
    "appId ": "0008"
  }
}
```



**响应示例**

```
{
  "body": {
    "resultDesc ": "", 
    "message": "", 
    "expandParams ": ""
    }, 
   "header": {
    "msgId": "61237890345", 
    "timestamp": "20160628180001165", 
    "resultCode": "000"
  }
}
```

<div STYLE="page-break-after: always;"></div>



# 4. 平台返回码说明

## 4.1. 本机号码校验接口返回码

本返回码表仅针对`本机号码校验接口`使用



| 返回码    | 说明              |
| ------ | --------------- |
| 000    | 是本机号码（纳入计费次数）   |
| 001    | 非本机号码（纳入计费次数）   |
| 002    | 取号失败            |
| 003    | 调用内部token校验接口失败 |
| 004    | 加密手机号码错误        |
| 102    | 参数无效            |
| 124    | 白名单校验失败         |
| 302    | sign校验失败        |
| 303    | 参数解析错误          |
| 606    | 验证Token失败       |
| 999    | 系统异常            |
| 102315 | 次数已用完           |

## 4.2. SDK返回码说明

| 错误编号   | 返回码描述     |
| ------ | --------- |
| 102000 | 成功        |
| 102101 | 无网络       |
| 102102 | 网络异常      |
| 102103 | 非移动网络     |
| 102109 | 网络错误      |
| 102207 | 获取的中间件值错误 |
| 102208 | 参数错误      |
| 102209 | 没有sim卡    |
| 102210 | 不支持短信发送   |
| 102301 | 用户取消      |
| 102302 | 没有进行初始化参数 |

<div STYLE="page-break-after: always;"></div>
