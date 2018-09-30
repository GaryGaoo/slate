---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - java
  - python
  - javascript

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction

目前关于API Key申请和修改，请在“个人中心 - API接口”页面进行相关操作。

常见问题请参考[FAQ](https://support.xdaex.com/hc/zh-cn/search?utf8=✓&query=api)

**通过API可以快速实现以下功能：**

* 获取市场最新行情

* 获取买卖深度信息

* 查询资产总额和可用余额

* 查询自己当前尚未成交的挂单

* 批量撤单

**操作前请您务必阅读[XDAEX程序化交易接入说明](XDAEX程序化交易接入说明)。**

### API说明文档<br>
* [API说明文档](API说明文档)<br>


### XDAEX程序化交易接入说明<br>
* [XDAEX程序化交易接入说明](XDAEX程序化交易接入说明)<br>
* [API协议信(PDF)](https://github.com/XDAEX/java-xdaex/raw/master/API%20%E5%8D%8F%E8%AE%AE%E4%BF%A1.pdf)<br>
* [API申请信息表(PDF)](https://github.com/XDAEX/java-xdaex/raw/master/API%20%E7%94%B3%E8%AF%B7%E4%BF%A1%E6%81%AF%E8%A1%A8.pdf)<br>


# API说明文档

> To authorize, use this code:

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
```

```shell
# With shell, you can just pass the correct header with each request
curl "api_endpoint_here"
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
```

> ## 1. 快速入门

XDAEX 为 API 用户提供了两种接口模式：
* REST 模式：采用“请求/应答”方式，完成创建订单、撤销订单、查询合约、查询资产、查询订单、查询成交、查询行情信息等操作。
* WebSocket 模式：采用“消息订阅/推送”方式，主动推送订单处理结果、行情等消息。

为方便用户快速理解，以下列出了使用本 API 编写交易程序的常规步骤：（各个调用的详细内容可以查看后续对应的章节）

### 1.1. 交易前准备
在开始交易前，建议调用以下 REST 接口，查询服务器时钟、交易所合约、用户资产余额等信息，以确保用户交易程序的正常工作：
* 获取服务器时钟：调用 /marketData/time 接口。系统时间采用 UTC ，香港的时间与 UTC 的时差为 +8 ，也就是 UTC+8 。
* 查询交易所有效合约对信息：调用 /marketData/instrument 接口。
* 查询用户资产余额：调用 /account 接口。
* 接入 WebSocket 公有流、私有流（如果不使用 WebSocket 接收消息，则忽略本条）。

### 1.2. 交易
* 调用 REST 接口 /order/create 创建订单（以下简称：下单、报单），用户可以在订单中自行指定 orderLocalId 。orderLocalId 是用户自定义订单编号，建议保证其增长性及唯一性）。
* 获得订单处理结果：
    + 如果使用 WebSocket 接收消息，则会接受到通过 WebSocket 私有流推送的消息：
        - order_return 消息：表示下单成功，其中含有 XDAEX 为该订单分配的系统订单编号 orderId 。
        - order_insert_rsp 消息：表示下单失败。
    + 如果不使用 WebSocket 接收消息，则调用 REST 接口 /order/getOrderByLocalId 使用 orderLocalId 查询订单信息，查询结果中会含有 XDAEX 为该订单分配的系统订单编号 orderId 。
* 若需撤销订单（以下简称：撤单），可采取两种方式：
    + 根据用户自定义订单编号 orderLocalId 调用 /order/cancelByLocalId 接口撤销订单。
    + 根据 XDAEX 为该订单分配的系统订单编号 orderId 调用 /order/cancel 接口撤销订单。

注意：关于 API 接入管理、流量控制和测试环境等内容请查阅后面的有关章节。在本文档的最后，有多种常用编程语言的“示例代码”供您参考。

## 2. API 接口规范
REST 接口和 WebSocket 接口的使用规范如下：
### 2.1. REST 接口规范

#### 2.1.1. HTTP 请求 URL
REST 接口请求 URL 形如：https://xdaex.com/APITrade/v1/account
URL 中主要包括以下组成部分：
* Protocal: HTTPS
* Domain: xdaex.com （如访问测试系统，则需替换成指定的域名）
* Mode: APITrade
* Version: v1
* Path: 不同功能要访问不同的Path，例如：查询用户资产余额对应的是/account

#### 2.1.2. HTTP请求消息头
* REST 接口 HTTP 请求消息头（Header）中必须包含以下内容：

属性 |描述|示例       
------------- |-----------|--------------------------------------------
API-KEY|API 接入身份标识， 详见后面的“API-KEY管理”|－－－
API-TIMESTAMP|请求发起时的时间戳（UTC）|"1420674445201"
API-SIGNATURE|用于服务端对请求内容进行校验的签名信息（先摘要再加签）|－－－
Content-Type|HTTP 标准 MIME 实体类型|"application/json"

* API-SIGNATURE 签名步骤
	+ 拼装原文：data = timestamp + HttpMethod + version + path + jsonBody
	    - body 是 json 格式
	    - HTTP Method 包含 GET/POST , 全部为大写格式，如果是 GET 请求，则  jsonBody 为空
	    - 请求路径：/version + /path， 如：/v1/account
        - GET 请求范例：data = "1234567891234GET/v1/account"
        - POST 请求范例：data = "1234567891234POST/v1/order/cancelByLocalId"{"orderLocalId":"1234567891234"}
	+ 用 SHA256 算法对原文取摘要，并转化为小写16进制字符串： hashedData = SHA256(data)
        - 范例：
            1. 输入：
                - data = "1234567891234GET/v1/account"
            2. 输出：
                - hashedData = "1d2ecca0e82ff141043eb689686f134c2d7136ebf134fe8bae0140105dd902db"
	+ 用私钥对摘要签名（算法：ECC , 曲线：标准曲线 secp256k1 ）: sign(hashedData)
        - 范例：
            1. 输入：
                - hashedData = "1d2ecca0e82ff141043eb689686f134c2d7136ebf134fe8bae0140105dd902db"
                - privateKey = "Bh9tSghsfhjdFHKJAfajk78fa980kaFDda78sa/fda="
            2. 输出：
                - sighedData = {"r":"FDS/fhdkjaf89YUIfda89fahkjfdaTYUF32IUfda=", "s":"NFMFDsfdsgkj678678431/fahkjaip68fahjKSDh=","v":28}

### 2.2. WebSocket 接口规范

#### 2.2.1. WebSocket 请求 URL
WebSocket 接口请求 URL 形如: wss://xdaex.com/APITradeWS/v1/messages
URL 中主要包括以下组成部分：
* Protocal: WSS
* Domain: xdaex.com （如访问测试系统，则需替换成指定的域名）
* Mode: APITradeWS
* Version: v1
* Path: /messages

#### 2.2.2. WebSocket 消息流
API 接入用户可通过订阅方式获取由服务端推送的私有和公有消息流，提交私有流订阅请求时需要通过 API-KEY 进行认证。
* 私有消息流
    + 使用 PRIVATE-KEY 对字串<font color="#dd0000" face = "黑体">"WSS/APITradeWS/v1/messages"</font>进行签名，得到 signature 。签名方法参照 REST 接口相关示例，但<font color="#dd0000" face = "黑体">不需要对原文取摘要（直接对原文签名即可）</font>
    + 订阅请求，例如：
        ``` JSON
        { "type":"subscribe", "channel":{ "user":[apikey, signature]}}
        ```
    + 取消订阅请求，例如：
        ``` JSON
        { "type":"unsubscribe ", "channel":{ "user":[apikey, signature]}}
        ```

* 公有消息流
    + 订阅请求，例如：
        ``` JSON
        {"type":"subscribe", "channel":{"ticker":["ETH-BTC"]}}
        ```
    + 取消订阅请求，例如：
        ```JSON
        {"type":"unsubscribe", "channel":{"ticker":["ETH-BTC"]}}
        ```

### 2.3. 请求应答编码对照表

#### 2.3.1. 成功应答
对于请求成功的各类应答，HTTP 协议状态返回码均为 200 ，同时包含响应的具体 Response 信息，具体可详见各类业务接口。

#### 2.3.2. 失败应答
对于请求失败的响应目前通过 respCode 和 respMsg 进行标记，用于说明失败的原因。
目前常见的失败应答按接口模式划分，可分为 2 类：

* REST 失败应答

respCode |             respMsg              |      备注        
-------- |------------------------- |----------- 
1000|Failed to validate signature                      |签名验证失败                
1001|Request timeout (30s)                             |已过期的请求（超过30秒）           
1002|API key not found                                 |不存在的API-KEY        
1003|Locked account                                    |用户账号被锁定            
1004|Too many requests                                 |接口访问频次超限         
1005|Internal service error                            |内部服务错误
1006|API key blacklisted                               |已禁用的API-KEY
2001|Order price decimal precision error               |下单金额的小数部分位数超限 
2002|Total volume decimal precision error              |下单数量的小数部分位数超限 
2003|Maximum orderLocalId field length: 20             |请求参数中orderLocalId长度超限（20字节）
2005|Maximum bulkcancel numbers: 10                    |批量撤单数量超限（10条）
2018|Order digit length exceeded                       |下单数量长度超限
2019|OrderLocalId or date must not be empty            |用户自定义订单编号或日期不能为空

* WebSocket 失败应答

respCode |             respMsg              |      备注        
-------- |------------------------- |----------- 
2004|OrderId not exist                                    |订单号不存在                 
2006|InstrumentId not exists                              |合约Id不存在     
2007|Operation not supported by instrument status         |当前合约状态禁止此项操作
2008|Client not found                                     |客户找不到
2009|Order field error                                    |订单字段错误
2010|Account not found                                    |找不到账号
2011|Insufficient funds                                   |资产不足
2012|Invalid volume                                       |不合法的数量
2013|Invalid order type                                   |不支持的订单类型
2014|Order already fully filled                           |订单已经全部成交
2015|Order already canceled                               |订单已经撤销
2016|OrderId must not be empty                            |订单号不能为空
2017|Order direction (sell or buy) error                  |订单方向（Sell or Buy）错误

若 API 用户对于运用中出现的失败应答仍有疑异的请联系客服人员，或可发送邮件至  support@xdaex.com。

## 3. REST接口说明
### 3.1. 账户资产（account）查询
查询用户的资产信息。
* Method: GET
* Version: v1
* Path: /account
* Response:
    ``` JSON
    [//example
        {
        "id": "string",             //账户id
        "asset": "string",          //数字资产类型
        "balance": "string",        //数字资产总额
        "available": "string"       //数字资产可用余额
        },    
        ……
    ]
    ```

### 3.2. 订单（order）相关
#### 3.2.1. 创建订单
1. 目前仅支持限价单（orderType：limit）。一旦下单成功，相应占用资产在订单未成交期间将被冻结。

2. orderLocalId 为用户自定义订单编号，orderLocalId 的最大长度为20， <font color="#dd0000" face = "黑体">为了用户后续订单管理的便利，强烈建议用户要保证其增长性及唯一性，例如：String(时间戳 * 10000 + 本地自增序列) </font>。

3. 下单接口仅返回 XDAEX 是否成功接收了用户下单请求，订单请求结果具体响应信息会经由 WebSocket 私有流推送。
* Method: POST
* Version: v1
* Path: /order/create
* Request:
    ``` JSON
    {//example
        "instrumentId": "ETH-BTC",      //合约编号
        "direction": "buy",             //买卖方向
        "price": "1.34",                //订单价格（订单价格不能为空且需大于0）
        "totalVolume": "1.34",          //订单总量（订单数量不能为空且需大于0）
        "orderLocalId": "1234567"       //用户自定义订单编号，建议确保其增长性及唯一性
    }
    ```
* Response:
    ``` JSON
    {//example
        "timestamp": "1420674445201",   //时间戳
        "orderLocalId": "1234567",      //用户自定义订单编号
    }
    ```

#### 3.2.2. 撤销订单
下单成功但未完全成交的订单可以被撤销，提供两种撤单的方式:根据订单本地编号（orderLocalId）撤单和根据系统订单编号（orderId）撤单。

1. 根据订单本地编号（orderLocalId）撤单： orderLocalId 是由用户在下单时自定义的。若由于用户自身的问题，导致上报的 orderLocalId 重复，则会存在多个  orderLocalId 相同的订单，<font color="#dd0000" face = "黑体">此时仅撤销其中的1笔订单</font>。
* Method: POST
* Version: v1
* Path: /order/cancelBylocalId
* Request:
    ``` JSON
    {//example
        "orderLocalId": "string"          //用户自定义订单编号
    }
    ```
* Response:
    ``` JSON
    //example
    {
        "orderLocalId": "string",    //用户自定义订单编号
        "message": "string"          //状态, “received” or “failed”。“received”仅表示交易所收到了撤单请求，不代表已经完成了撤单操作。
    }
    ```

2. 根据系统订单编号（orderId）撤单： XDAEX 交易所保证了 orderId 的唯一性，<font color="#dd0000" face = "黑体">因此只会撤销该单笔订单的未成交部分</font>。
* Method: POST
* Version: v1
* Path: /order/cancel
* Request:
    ``` JSON
    {//example
        "orderId": "string"           //系统订单编号
    }
    ```
* Response:
    ``` JSON
    {//example
        "orderId": "string",         //系统订单编号
        "message": "string"          //状态, “received” or “failed”。“received”仅表示交易所收到了撤单请求，不代表已经完成了撤单操作。
    }
    ```

#### 3.2.3. 批量撤销未成交订单
此功能每次最多撤销<font color="#dd0000" face = "黑体">10</font>笔未完全成交订单。
* Method: POST
* Version: v1
* Path: /order/bulkCancel
* Request:
    ``` JSON
    {//example
        "orderIds": [ "string",……]  //系统订单编号组，最多可包含10个订单编号
    }
    ```
* Response:
    ``` JSON
    [//example
        "string",…… //交易所收到了撤单请求对应的系统订单编号
    ]
    ```

#### 3.2.4. 查询订单
与撤销订单类似，提供两种查单的方式：根据订单本地编号（orderLocalId）查询和根据系统订单编号（orderId）查询。

1. 根据订单本地编号（orderLocalId）查询订单：仅返回<font color="#dd0000" face = "黑体">未成交和部分成交</font>的订单。orderLocalId 是由用户在下单时自定义的。若由于用户自身的问题，导致上报的 orderLocalId 重复，则会存在多个 orderLocalId 相同的订单，<font color="#dd0000" face = "黑体">此时仅返回其中的1笔订单</font>。
* Method: POST
* Version: v1
* Path: /order/getOrderByLocalId
* Request:
    ``` JSON
    {//example
        "orderLocalId":"String",        //用户自定义订单编号
        "date":"YYYYMMdd"               //订单日期（按照UTC的时区）
    }
    ```
* Response:
    ``` JSON
    {//example
        "orderId": "string",            //系统订单编号
        "orderLocalId": "string",       //用户自定义订单编号
        "totalVolume": "1",             //订单总量
        "instrumentId": "ETH-BTC",      //合约编号
        "direction": "string",          //买卖方向
        "remainVolume ": "0.2",         //剩余未成交量
        "tradedVolume": "0",            //已成交量
        "orderStatus": "0",             //订单状态
        "price": "string",              //订单价格
        "timestamp": "1420674445201"    //下单时间
    }
    ```

2. 根据系统订单编号（orderId）查询订单：如果不指定 orderId ，最多返回<font color="#dd0000" face = "黑体">100</font>条数据，返回数据按照下单时间排序<b>（从晚到早）</b>。
* Method: POST
* Version: v1
* Path: /order/getOrder
* Request:
    ``` JSON
    {//example
        "startTime": "1420674445201",  //订单起始时间（非必填）
        "endTime": "1420674445201",    //订单结束时间（非必填）
        "status": "string",            //订单状态（非必填）｛包括active（未成交和部分成交）、closed（部分成交后转撤单和全部成交）两种状态;default:active+closed｝
        "instrumentId": "string",      //合约编号（非必填）
        "orderId": "string"            //系统订单编号（非必填）
    }
    ```
* Response:
    ``` JSON
    {//example
        "orderId": "string",            //系统订单编号
        "orderLocalId": "string",       //用户自定义订单编号
        "totalVolume": "1",             //订单总量
        "instrumentId": "ETH-BTC",      //合约编号
        "direction": "string",          //买卖方向
        "remainVolume ": "0.2",         //剩余未成交量
        "tradedVolume": "0",            //已成交量
        "orderStatus": "string",        //订单状态｛0-全部成交, 1-部分成交还在队列中, 2-部分成交不在队列中, 3-未成交还在队列中, 4-未成交不在队列中, 5-撤单, 6-部成部撤｝
        "price": "string",              //订单价格
        "timestamp": "1420674445201"    //下单时间
    }
    ```

### 3.3. 成交（trade）相关
#### 3.3.1. 成交查询
查询成交信息，最多返回 100 条数据，返回数据按照成交时间排序<b>（从晚到早）</b>。
* Method: POST
* Version: v1
* Path: /trade/getTrade
* Request:
    ``` JSON
    {//example
        "startTime": "string",      //订单起始时间（非必填）
        "endTime": "string",        //订单结束时间（非必填）
        "instrumentId": "string"    //合约编号（非必填）
    }
    ```
* Response:
    ``` JSON
    [//example
        {
            "orderId": "string",            //订单编号
            "tradeId": "1234567",           //成交编号
            "instrumentId": "ETH-BTC",      //合约编号
            "direction": "string",          //买卖方向｛"buy","sell"｝
            "volume": "0",                  //成交数量
            "fee": "1.1",                   //手续费
            "price": "1.1",                 //成交价格
            "timestamp": "1420674445201"    //成交时间
        }
    ]
    ```

### 3.4. 市场（marketData）相关
#### 3.4.1. 合约查询
查询 instrumentId 对应的合约信息，若 instrumentId 为空，则查询全部有效合约的信息。
* Method: GET
* Version: v1
* Path: /marketData/instrument 或
/marketData/instrument?instrumentId=ETH-BTC&instrumentId=CYB-BTC（Optional）
* Response:
    ``` JSON
    [//example
        {
            "instrumentId": "ETH-BTC",     //合约编号
            "tradingAsset": "BTC",         //基础资产
            "tradingTarget": "ETH",        //目标资产
            "volumePrecision": "3",        //订单数量精度
            "pricePrecision": "8"          //订单价格精度
        }
    ]
    ```

#### 3.4.2. level2行情查询
查询指定合约的 level2 行情信息。
* Method: GET
* Version: v1
* Path: /marketData/level2?instrumentId=ETH-BTC
必须指定合约ID参数。
* Response:
    ``` JSON
    {//example
        "instrumentId ": "ETH-BTC",          //合约编号
        "buy": [["6500.11", "0.45054140"]],  //买队列,目前50个档位
        "sell": [["6500.15", "0.57753524"]]，//卖队列,目前50个档位
        "timestamp": "1420674445201"         //时间戳
    }
    ```

#### 3.4.3. 业务时钟查询
查询服务端业务时钟，目前服务端采用 UTC 时区时钟。
* Method: GET
* Version: v1
* Path: /marketData/time
* Response:
    ``` JSON
    {//example
        "timestamp": "1420674445201"       //时间戳
    }
    ```

## 4. WebSocket 接口说明
HTML5 定义了 WebSocket 协议，允许服务端主动向客户端推送数据，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

### 4.1. 心跳机制
心跳消息由 WebSocket 订阅方主动发起 "ping" 消息，每<b>15</b>秒发送一次,服务端回复 "pong" 消息。

### 4.2. 私有消息流
#### 4.2.1. 私有消息订阅管理
私有消息订阅管理时需要通过 API-KEY 进行认证。
* 订阅/取消订阅的消息格式如下：
    ``` JSON
    { "type":"subscribe", "channel":{ "user":[apikey, signature]}}

    { "type":"unsubscribe ", "channel":{ "user":[apikey, signature]}}
    ```

#### 4.2.2. 接收私有消息推送
当订阅私有消息之后，将会收到以下几类消息推送：
##### 4.2.2.1. 订单已经进入订单簿(order_return)
订单已经进入订单簿后推送。
* Messages:
    ```JSON
    {//example
        "type": "order_return",         //消息类型
        "timestamp": "1420674445201",   //时间戳
        "participantId": "0000000001",  //市场参与者编号
        "instrumentId ": "ETH-BTC",     //合约编号
        "clientId": "1234567",          //客户编号
        "orderLocalId": "1234567",      //用户自定义订单编号
        "orderId": "1234567",           //系统订单编号
        "totalVolume ": "1.34",         //订单总量
        "price": "502.1",               //订单价格
        "direction": "buy",             //买卖方向
        "orderType": "limit"            //订单类型
    }
    ```
##### 4.2.2.2. 下单失败(order_insert_rsp)
下单失败后推送。
* Messages:
    ```JSON
    {//example
        "type": "order_insert_rsp",     //消息类型
        "orderLocalId": "12321",        //用户自定义订单编号
        "respCode": "2000",             //异常应答码，详见前面的“请求应答编码对照表”
        "respMsg": "fdsafdsa"           //异常应答消息
    }
    ```

##### 4.2.2.3. 订单已经从订单簿中撤销(order_cancel_rsp)
订单已经从订单簿中撤销后推送。
* Messages:
    ```JSON
    {//example
        "type": " order_cancel_rsp",    //消息类型
        "timestamp": "1420674445201",   //时间戳
        "instrumentId ": "ETH-BTC",     //合约编号
        "price": "200.2",               //订单价格
        "orderId": "1234567",           //系统订单编号
        "totalVolume": "5.23512"，      //订单总量
        "orderLocalId": "1234567"       //用户自定义订单编号
    }
    ```

##### 4.2.2.4. 从订单簿中撤销订单失败(order_cancel_failed)
从订单簿中撤销订单失败后推送。
* Messages:
    ```JSON
    {//example
        "type": "order_cancel_failed",   //消息类型
        "orderId": "12354534",           //系统订单编号
        "respCode": "2000",              //异常应答码，详见前面的“请求应答编码对照表”
        "respMsg": "fdasfdsa"            //异常应答消息
    }
    ```

##### 4.2.2.5. 订单成交
产生订单成交后推送。
* Messages:
    ```JSON
    {//example
        "type": "trade_return",         //消息类型
        "orderId": "1234567",           //系统订单编号
        "tradeId": "10",                //成交编号
        "timestamp": "1420674445201",   //时间戳
        "instrumentId ": "ETH-BTC",     //合约编号
        "volume": "5.23512",            //本次成交的数量
        "price": "400.23",              //成交价格
        "direction": "buy",             //买卖方向
        "fee ": "string"                //手续费
        "orderLocalId": "12321",        //用户自定义订单编号
    }
    ```

### 4.3. 公共消息流
可订阅的公共消息流目前主要包括：Ticker（每次成交信息）、Level2（深度行情信息）。

#### 4.3.1. Ticker（每次成交信息）
市场中每次成交发生后推送。

* 订阅/取消订阅的消息格式如下：
    ```JSON
    {"type": "subscribe", "channel":{ "ticker":["ETH-BTC", "CYB-BTC"]}}

    {"type": "unsubscribe", "channel":{ "ticker":["ETH-BTC", "CYB-BTC"]}}
    ```

当订阅ticker主题消息之后，将会推送以下消息类型：
* Messages:
    ```JSON
    {//example
        "type": "trade_detail",         //消息类型
        "timestamp": "1420674445201",   //成交时间
        "tradeId": "12345678",          //成交编号
        "instrumentId ": "ETH-BTC",     //合约编号
        "price ": "string",             //成交价格
        "pricingBy":"buy",              //成交价来源
        "volume ": "0.03000000"         //成交数量
    }

    {//example
        "type": "trade_detail",         //消息类型
        "timestamp": "1420674445201",   //成交时间
        "tradeId": "12345678",          //成交编号
        "instrumentId ": "CYB-BTC",     //合约编号
        "price ": "string",             //成交价格
        "pricingBy":"buy",              //成交价来源
        "volume ": "0.03000000"         //成交数量
    }
    ```
#### 4.3.2. Level2（深度行情信息）
按照多主题（N档行情不同深度），推送订单簿中的价位信息。订阅端连接时，首次先推送所选合约的全量N档行情快照，之后就仅推送增量的订单簿变动信息。相关主题定义请见下表：

主题编码 |             价位档数              |      推送频率        
-------- |------------------------- |----------- 
level2|50                      |每秒2次     
level2@5|5                     |每秒2次     
level2@10|10                   |每秒2次     
level2@20|20                   |每秒2次     

* 订阅/取消订阅的消息格式如下：
    ```JSON
    { "type":"subscribe", "channel":{ "level2@5":["ETH-BTC"]}}

    { "type":"unsubscribe", "channel":{ "level2@5":["ETH-BTC"]}}
    ```

当订阅level2主题消息后，将会推送以下消息类型：
* Messages:
    ```JSON
    {//example
        "type": "level2(or level2@5 or level2@10 or level2@20)_snapshot", //消息类型
        "instrumentId ": "ETH-BTC",         //合约编号
        "buy": [["5500.21", "0.47545240"]], //买队列，默认50个档位（可选5/10/20个档位）
        "sell": [["5500.25", "0.56544624"]] //卖队列，默认50个档位（可选5/10/20个档位）
    }

    {
        "type": "level2(or level2@5 or level2@10 or level2@20)_update",  //消息类型
        "instrumentId": "ETH-BTC",          //合约编号
        "buy": [["5500.21", "0.47545240"]], //买队列，默认50个档位（可选5/10/20个档位）
        "sell": [["5500.25", "0"]]          //卖队列，默认50个档位（可选5/10/20个档位）
    }
    ```

## 5. 其他
### 5.1. API-KEY管理
API-KEY的申请及更换流程：
* API-KEY需在交易所网页的“用户中心--API接口”中设置，用户将拥有该API-KEY对应的私钥privateKey, privateKey是私钥，在用户本地浏览器中生成，不会被传送到服务器，请在本地妥善保管和备份。<b>该私钥与您的账户安全密切相关，请勿泄露！</b>
* 若用户privateKey丢失或被盗用，可在交易所网页的“用户中心--API接口”中删除该API-KEY，并重新生成新的API-KEY。

### 5.2. 流量控制
对于API用户提交的请求，交易后端采取流量控制，当超过流控阈值时，将返回"1004 Too many requests"报错信息。
* 所有查询类请求计入同一流量指标
* 下单及撤单请求计入同一流量指标
	+ 1个下单或撤单（create/cancel/cancelByLocalId）计入流量指标的数量是1
	+ 1个批量撤单（bulkCancel）中若包括了m个订单，则计入流量指标的数量是m

用户可发送邮件至support@xdaex.com，了解具体的流量控制信息。

### 5.3. 黑名单
若由于安全原因，某API-KEY被管理员加入黑名单，将返回"1006 API key blacklisted"报错信息。若API-KEY处于黑名单中，用户可发送邮件至support@xdaex.com，申请从黑名单中移除。

### 5.4. 测试环境
为了确保用户资产安全，建议用户在测试环境中完成必要的系统测试后，再接入生产系统进行交易。通过api-test.xdaex.com可以访问测试环境，用户可发送邮件至support@xdaex.com，申请使用测试环境。

### 5.5 示例代码
#### 5.5.1. REST接口示例代码
目前 REST 接口提供 Java 及 Python 两种语言示例代码，相关下载路径如下：
* Java
    +  Java SDK 下载链接为：https://github.com/XDAEX/java-xdaex/blob/master/xdaex-trading-sdk-1.0.jar
    +  Java 代码示例链接为： https://github.com/XDAEX/java-xdaex/wiki/REST%E6%8E%A5%E5%8F%A3%E7%A4%BA%E4%BE%8B%E6%96%87%E6%A1%A3%EF%BC%88Java%EF%BC%89
* Python
    +  Python 代码示例链接为： https://github.com/XDAEX/java-xdaex/wiki/REST%E6%8E%A5%E5%8F%A3%E7%A4%BA%E4%BE%8B%E6%96%87%E6%A1%A3%EF%BC%88Python%EF%BC%89
    
#### 5.5.2. WebSocket订阅示例代码
WebSocket 消息流包含公有流和私有流，目前提供了 javascript 语言的两种消息流订阅示例。
* 私有流订阅 javascript 代码示例链接为：https://github.com/XDAEX/java-xdaex/wiki/WebSocket接口示例文档（Javascript）  
* 公有流订阅 javascript 代码示例链接为：https://github.com/XDAEX/java-xdaex/wiki/WebSocket接口示例文档（Javascript） 
`Authorization: meowmeowmeow`

<aside class="notice">
You must replace <code>meowmeowmeow</code> with your personal API key.
</aside>

# Kittens

## Get All Kittens

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens"
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let kittens = api.kittens.get();
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2"
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.get(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve

## Delete a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.delete(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.delete(2)
```

```shell
curl "http://example.com/api/kittens/2"
  -X DELETE
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.delete(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted" : ":("
}
```

This endpoint deletes a specific kitten.

### HTTP Request

`DELETE http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to delete

