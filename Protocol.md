<br>

# <center > 数据交换协议(草案)</center>

<center>第八组</center>
<center>2024 / 3 / 284</center>

<br>

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

### 注意
设计时并未考虑安全认证问题


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
string jsonPayload = @"
{
    ""account"":114514,
    ""dorm_number"": 1
}
";
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
|/|Get|-|测试服务器是否可用|"Server is running!!!"|
|/update/account|POST|new_account:你想要更新的用户名|更新用户名|{ "is_success":bool, "failed_message":string }|
|/update/schedule|POST|{"segments":[{"start_time":int, "end_time":int}, ...]}|更新计划信息，字符串格式参考 [更新计划信息](#更新计划信息)|{"is_success":bool, "failed_message":string}|
|/update/interval|POST|{"interval":int}|更新interval(秒)|{"is_success":bool, "failed_message":string}|
|/check/account/[q]|GET|q:你需要查询的用户名|查询用户名是否存在|{"is_exist":bool}|
|/[account]/schedule|Get|account:用户名|查询计划信息字符串格式参考 [更新计划信息](#更新计划信息)|[ {"start_time":int, "end_time":int} ...]|
|/all_account|Get||获取所有account|["name1","name2"]|


## 服务端主程序_与_单片机端

考虑到单片机羸弱的性能以及电量消耗，网络连接使用socket

### socket连接说明

socket连接类型为TCP

服务器端运行socketServer来接受socket连接，socket连接由单片机端发起

socket传输的均为使用UTF-8编码的文本信息

其中，为避免粘包，每次发送完信息必须在末尾加上一个EOF标识符`\n`，注意避免使用该字符

文本信息格式均采用JSON

设计模式采用事件驱动模式

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
|volume|[发送的是音量信息](#发送音量信息)|
|login|[登录账户名（用于区分不同大寝）](#登录)|
|request|[向服务端请求信息](#请求信息)|


#### 发送音量信息
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
为区分不同的大寝，每个大寝都设置一个独一无二的账号名。且这应当是建立TCP连接后做的第一件事。
```json
{
    "type":"login",
    "account":"114514",
    "dorm":2//小寝号
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
- interval : 请求获取[间隔时间](#更新间隔时间)

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
|update|[更新信息](#更新信息)|
|mute|[屏蔽警告工作](#屏蔽警告工作)|

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
- interval : 表示更新interval

##### 更新计划信息
```json
{
    "type":"update",
    "content":"schedule",
    "schedule":[//以每周一0点作为起始点，之后以分钟为单位累加
        {//segment 1
            "start_time":1380,//23*60=1380
            "end_time":1890//24*60+7*60 +30=1890表示这是在周二
        },
        {//segment 2
            "start_time":2820,//24*60+23*60=2820
            "end_time":3360//24*60+24*60+8*60=3360//周三
        },
        {//segment 3
            "start_time":...,
            "end_time":...,
        },
        {//...
            "start_time":...,
            "end_time":...
        },
    ],

}
```
时间计划以`周`为单位，以周日末周一初为计时零点，时间不断向上加

    "start_time":"1380",//23*60=1380
    "end_time":"1890"//24*60+7*60 +30=1890表示这是在周二


##### 更新用户名
```json
{
    "type":"update",
    "content":"account",
    "account":"your new account name"
}
```
##### 更新间隔时间

在单片机检测到大吵大闹之后会上报至服务网进行QQ提醒，若仍不安静，将间隔`interval`时间之后进行设备端声音提示

```json
{
    "type":"update",
    "content":"interval",
    "interval":30
}
```

#### 屏蔽警告工作

用于在此时刻起停止警告，停止`time`时间

```json
{
    "type":"mute",
    "time":"hh:mm",
}
```


# 更改

如果你有更好的建议，可以点击下面链接进行更改

<https://github.com/Happy-SCU-Team/guideline/blob/main/Protocol.md>
