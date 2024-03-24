# <center > 数据交换协议</center>

## 总览
整个项目可以解耦为以下部分
- [QQ机器人](#qq机器人服务)
- 服务端主程序（包含后端[RESTful API](#服务端主程序的restful_api) 与 [单片机通信](#服务端主程序_与_单片机端)部分）
- [单片机端程序](#服务端主程序_与_单片机端)
- Web前端，用于设置该设备


数据流向如下

$$
QQ机器人 \leftarrow 服务端主程序 \rightleftarrows 单片机端程序
$$

## QQ机器人服务

考虑到使用简便性，QQ机器人的通信方式采用HTTP POST

 QQ机器人服务(本地调用):使用HTTP POST来调用，内容为JSON格式
  
HTTP地址`http://localhost/alert`

```json
{
    "account":"114514",
    "dorm_number":1
}
```
调用示例 `c#`
```cs
string jsonPayload = @"{""dorm_number"": 1}";
using (HttpClient client = new HttpClient())
{
    HttpResponseMessage response = 
        await client.PostAsync(
            "http://localhost/alert", 
            new StringContent(jsonPayload, Encoding.UTF8, "application/json")
        );
}
```
## 服务端主程序的RESTful_API

注：所有有返回值的API，返回值均采用JSON格式

注：使用POST时，参数均为JSON格式，并且继承于如下JSON
```json
{
    "account":"your account name",
}
```


|地址|方式|参数|说明|返回值|
|:---|:---|:---|:---|:---|
|/update/account|POST|new_account:你想要更新的用户名|更新用户名|{ "is_success":bool, "failed_message":string }|
|/update/schedule|POST|{"day":int, "start_time":string, "end_time":string}|更新计划信息，字符串格式参考 [更新计划信息](#更新计划信息)|{"is_success":bool, "failed_message":string}|
|/check/account/[q]|GET|q:你需要查询的用户名|查询用户名是否存在|{”is_exist:bool}|


## 服务端主程序_与_单片机端

考虑到单片机羸弱的性能以及电量消耗，网络连接使用socket

### socket连接说明
服务器端运行socketServer来接受socket连接，socket连接由单片机端发起

socket传输的均为使用UTF-8编码的文本信息

其中，为避免粘包，每次发送完信息必须在末尾加上一个EOF标识符`\u0004`(或HTML表示法`&#04;`)，因为这是一个不可见字符，通常不会使用，当然也要避免使用该字符

文本信息格式均采用JSON

### 服务端主程序 $\leftarrow$ 单片机端

#### 基础JSON
每个json都应包含以下内容
```json
{
    "type":"..."
}
```
`type`共有以下类型

|type|含义|
|:---:|:---|
|volume|发送的是音量信息|
|login|账户名（用于区分不同大寝）|
|request|向服务端请求信息|


#### 发送音量信息：
```json
{
    "type":"volume",//类型
    "volume_type":"alert",
    "volume":35,//分贝
    "time":1711202028,//时间戳
    "dorm":2//寝室号
}
```
其中`volume_type`共有两种值   
- alert : 代表单片机通过算法确定了某寝室在休息时间吵闹，需要提醒
- info : 代表只是单纯地向服务器发送这一时刻的音量信息

#### 登录
为区分不同的大寝，每个大寝都设置一个独一无二的账号名
```json
{
    "type":"login",
    "account":"114514"
}
```


#### 请求信息
```json
{
    "type":"request",
    "content":"...",
}
```
`content`有以下值
- schedule : 请求获取[计划信息](#更新计划信息) 

### 服务端主程序 $\rightarrow$ 单片机端

#### 基础JSON
每个json都应包含以下内容
```json
{
    "type":"..."
}
```
`type`共有以下类型

|type|含义|
|:---:|:---|
|update|更新信息|

#### 更新信息
```json
{
    "type":"update",
    "content":"..."
}
```
`content`有以下类型
- schedule : 表示更新计划信息
- account : 表示更新用户名

##### 更新计划信息
```json
{
    "type":"update",
    "content":"schedule",
    "schedule":[
        {//周一
            "start_time":"23:00",
            "end_time":"31:30"//24:00+7:30=31:30表示这是在后一天
        },
        {//周二
            "start_time":"...",
            "end_time":""
        },
        {//周三
            "start_time":"",
            "end_time":""
        },
        {//周四
            "start_time":"",
            "end_time":""
        },
        {//周五
            "start_time":"",
            "end_time":""
        },
        {//周六
            "start_time":"",
            "end_time":""
        },
        {//周日
            "start_time":"",
            "end_time":""
        },
    ],

}
```
其中，考虑到人类的习惯，将次日凌晨放入前一天的数据中

如，周一

    "start_time":"23:00",
    "end_time":"31:30"//24:00+7:30=31:30表示这是在后一天

##### 更新用户名
```json
{
    "type":"update",
    "content":"account",
    "account":"your new account name"
}
```
