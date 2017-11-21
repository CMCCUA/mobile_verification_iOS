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
     [TYRZBaseApi customInit:APPID appKey:APPKEY sourceID:SOURCEID];
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
| token       | NSString   | 成功时返回：临时凭证                               | 成功时必填 |
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

![1](image\1.png)

</br>

### 3.1.2. 接口说明

校验用户输入的号码是否本机号码。
应用将手机号码传给统一认证SDK，统一认证SDK向统一认证服务端发起本机号码校验请求，统一认证服务端通过网关或者短信上行获取本机手机号码和第三方应用传输的手机号码进行校验，返回校验结果。</br>

**调用次数说明：**本产品属于收费业务，开发者未签订服务合同前，每天总调用次数有限。

**请求地址：** https://www.cmpassport.com/openapi/rs/tokenValidate

**协议：** HTTPS

**请求方法：** POST+json

**回调地址：**请参考开发者接入流程文档中`配置业务服务端地址`相关操作

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
| phoneNum      | String | 2     | 是                     | 待校验的手机号码的64位sha256值，字母大写。（手机号码+appKey+timestamp）（注：“+”号为合并意思） |      |
| token         | String | 2     | 是                     | 身份标识，字符串形式的token                         |      |
| sign          | String | 2     | 是                     | 签名，HMACSHA256(appId+ msgId+phonNum+timestamp+token+version)，输出64位大写字母 （注：“+”号为合并意思，不包含在被加密的字符串中,appkey为秘钥, 参数名做自然排序（Java是用TreeMap进行的自然排序）） |      |
|               |        |       |                       |                                          |      |

</br>

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
    "version ": "1.0", 
    "appId ": "0008"
  }
}
```



**响应参数**

| 参数           | 层级    | 类型     | 约束   | 说明                                       |      |
| ------------ | ----- | :----- | :--- | :--------------------------------------- | ---- |
| **header**   | **1** |        | 必选   |                                          |      |
| msgId        | 2     | string | 必选   | 对应的请求消息中的msgid                           |      |
| timestamp    | 2     | string | 必选   | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |      |
| appId        | 2     | string | 必选   | 应用ID                                     |      |
| resultCode   | 2     | string | 必选   | 规则参见具体接口返回码说明                            |      |
| **body**     | **1** |        | 必选   |                                          |      |
| resultDesc   | 2     | String | 必选   | 返回结果描述信息：<br/>000:是本机号码<br/>001:非本机号码<br/>102:参数无效<br/>108:无效的手机号<br/>302:签名校验不通过<br/>606:token校验失败<br/>999:系统异常<br/>102315：使用次数为0<br/>其中，000和001状态纳入计费次数 |      |
| message      | 2     | String | 否    | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码      |      |
| expandParams | 2     | String | 否    | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |      |
|              |       |        |      |                                          |      |

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
    "resultCode": "1.0"
  }
}
```

<div STYLE="page-break-after: always;"></div>



# 4. 平台返回码说明

## 4.1. 本机号码校验接口返回码

本返回码表仅针对`本机号码校验接口`使用

| 返回码    | 说明                   |
| ------ | -------------------- |
| 000    | 成功                   |
| 001    | 接口处理失败               |
| 002    | 短信验证码下发失败            |
| 003    | 生成短信验证码失败            |
| 004    | 短信验证码下发频率限制          |
| 026    | 同一用户认证过于频繁           |
| 028    | 与上次登陆地点不一致，两次登陆地点不一样 |
| 101    | 接口名称无效               |
| 102    | 参数无效                 |
| 103    | 短信验证码已过期             |
| 104    | 短信验证码错误              |
| 105    | 密码错误                 |
| 106    | ip地址非法               |
| 107    | 免登录令牌无效              |
| 108    | 手机号码格式错误/无效的手机号      |
| 109    | 邮箱地址格式错误             |
| 110    | 新密码不能与当前密码相同         |
| 111    | 登录认证类型不存在            |
| 112    | 登录认证类型非法             |
| 113    | WAP网关获取信息失败          |
| 200    | 用户已开通通行证，但未开通当前业务    |
| 201    | 用户已开通通行证，且已开通当前业务    |
| 202    | 短信网关异常               |
| 203    | 数据库异常                |
| 204    | 访问频率限制               |
| 205    | 调用app接口失败            |
| 206    | 该用户不存在               |
| 207    | 用户还未创建通行证            |
| 208    | UID不存在或失效            |
| 209    | UID与openid不匹配        |
| 210    | 二级密码输入错误             |
| 211    | 密码加密的哈希类型错误          |
| 212    | 上行短信为空或发送失败          |
| 228    | 解码失败                 |
| 233    | 接收联通取号结果为空           |
| 240    | 接收电信取号结果为空           |
| 302    | 签名校验不通过              |
| 303    | 参数解析错误               |
| 601    | 公钥不存在或过期             |
| 602    | 用户密码解密失败             |
| 603    | 客户端随机数解密失败           |
| 604    | 用户未登录或密钥不存在          |
| 605    | 用户密钥已过期              |
| 606    | 验证Token失败            |
| 607    | 强制重新登录               |
| 608    | clientid不存在          |
| 609    | Android包名或签名值验证错误    |
| 610    | 身份凭证不存在（token）       |
| 611    | imsi验证失败             |
| 612    | imei验证失败             |
| 999    | 未知系统异常               |
| 102315 | 使用次数为0               |

</br>

## 4.2. SDK返回码说明

| 错误编号   | 返回码描述                                    |
| ------ | ---------------------------------------- |
| 102000 | 成功                                       |
| 102101 | 无网络                                      |
| 102102 | 网络异常                                     |
| 102103 | 非移动网络                                    |
| 102109 | 网络错误                                     |
| 102201 | 自动登陆失败，用户选择自定义界面时，需要继续处理手动登陆流程，比如弹出登陆界面。 |
| 102202 | APP签名验证不通过                               |
| 102203 | 接口入参错误                                   |
| 102204 | 正在进行GetToken动作，请稍后                       |
| 102205 | 当前环境不支持指定的登陆方式                           |
| 102206 | 选择用户登陆时，本地不存在指定的用户                       |
| 102207 | 获取的中间件值错误                                |
| 102208 | 参数错误                                     |
| 102209 | 没有sim卡                                   |
| 102210 | 不支持短信发送                                  |
| 10299  | 其他错误                                     |
| 102301 | 用户取消                                     |
| 102302 | 没有进行初始化参数                                |
| 102303 | 用户名为空                                    |
| 102304 | 密码为空                                     |
| 102305 | 验证码获得成功                                  |
| 102306 | 验证码获得失败                                  |
| 102307 | 用户名格式错误                                  |
| 102308 | 用户名格式错误                                  |
| 102309 | 验证码格式错误                                  |
| 102310 | 用户名和验证码格式错误                              |
| 102311 | 密码格式错误                                   |
| 102312 | 用户名和密码格式错误                               |

<div STYLE="page-break-after: always;"></div>

# 附录1 认证方法标识

| authType | authTypeDes        |
| -------- | ------------------ |
| 0        | 其他（超时无法确认认证类型，返回0） |
| 1        | WIFI下网关鉴权          |
| 2        | 网关鉴权               |
| 3        | 短信上行鉴权             |
| 4        | WIFI下网关鉴权复用中间件登录   |
| 5        | 网关鉴权复用中间件登录        |
| 6        | 短信上行鉴权复用中间件登录      |
| 7        | 短信验证码登录            |

