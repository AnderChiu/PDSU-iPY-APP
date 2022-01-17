> **起因**：家中某宝经常熬夜只为打卡，我觉得为了打卡而去熬的夜实在不值得，遂用所学实现自动报备。现在疫情还在跌宕起伏，本文只为那些长期不变更位置的同学提供参考，如果个人情况有变动请及时手动报备！！！【本文仅提出一种技术方案，不承担滥用造成的后果】
> @[toc]

## 原理简介

因为平院改用了打卡的软件，这次是提供APP打卡，而我们知道要实现自动化一般是对网页进行操作，主要有两种方式，一种是通过模拟人点击网页按钮进行操作，另一种则是使用如Post请求进行打卡。对于APP我们使用第二种。

因此我们首先要找到打卡时APP向服务器发送的post请求，这里需要`抓包`，抓包不会的同学自己去查一下吧，安卓可以用`小黄鸟`，ios用`Stream`。提前打开软件开始监听，打开APP进入报备界面，先自己正常打一次卡，打卡结束去抓包软件找到post请求的记录，如下：

```
>> 本文件内容为 https://yqfk.pdsu.edu.cn/smart-boot/api/healthReport/saveHealthReport 的请求抓包详情，供您分析和定位问题。 

1. 请求内容 Request:

POST /smart-boot/api/healthReport/saveHealthReport HTTP/1.1
Host: yqfk.pdsu.edu.cn
Accept: */*
X-Device-Info: APP
X-Terminal-Info: APP
Accept-Language: zh-CN,zh-Hans;q=0.9
Accept-Encoding: gzip, deflate, br
Content-Type: application/json;charset=UTF-8
Origin: https://yqfk.pdsu.edu.cn
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 15_0_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 Html5Plus/1.0 (Immersed/44) uni-app SuperApp-10919
Referer: https://yqfk.pdsu.edu.cn/serv-h5/
X-Id-Token: eyJhbGciOiJSUzUxMiJ9.eyJBVFR... ...  
Content-Length: 1370
Connection: keep-alive

{"needUpdate":1,"isAgree":true,"createTime":"2022-01-13 00:02:01","age":"20",... ... ,"remark":null}

2. 响应内容 Response:

HTTP/1.1 200 
Server: nginx
Date: Wed, 12 Jan 2022 16:03:19 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
Access-control-Allow-Origin: https://yqfk.pdsu.edu.cn
Access-Control-Allow-Methods: GET,POST,OPTIONS,PUT,DELETE
Access-Control-Allow-Credentials: true
Set-Cookie: rememberMe=deleteMe; Path=/smart-boot; Max-Age=0; Expires=Tue, 11-Jan-2022 16:03:19 GMT; SameSite=lax
Vary: accept-encoding,origin,access-control-request-headers,access-control-request-method,accept-encoding
Content-Encoding: gzip

{"success":true,"message":"操作成功！","code":200,"result":"上报成功","timestamp":1642003399636}

```

分析抓包数据可知打卡就是向服务器发了一次Post请求，思路比较清晰，鉴别身份主要是通过`X-Id-Token`，我发现他这个有效期还挺长，所以抓一次到现在还能用，至于能用多久等失效了再改代码，后面如果经常失效就需要再加一段获取token的代码来动态获得token。我们使用Python向服务器发请求就完事了。

## 代码实现

```python
#!/usr/bin/env python3
import requests
import json
from time import strftime, localtime
import traceback
import logging  # 用于日志控制

token = '32178a9axxxxxxxx'

# 日志基础配置
logger = logging.getLogger()
logger.setLevel(logging.INFO)
fh = logging.FileHandler('./log.txt', mode='a', encoding='utf-8')
fh.setFormatter(logging.Formatter("%(message)s"))
logger.addHandler(fh)
ch = logging.StreamHandler()
ch.setFormatter(logging.Formatter("[%(asctime)s]:%(levelname)s:%(message)s"))
logger.addHandler(ch)


# 返回要推送的通知内容
def readFile_html(filepath):
    content = ''
    with open(filepath, "r", encoding='utf-8') as f:
        for line in f.readlines():
            content += line + '<br>'
    return content


def sendPushplus(token):
    try:
        # 发送内容
        data = {
            "token": token,
            "title": "健康打卡",
            "content": readFile_html('./log.txt')
        }
        url = 'http://www.pushplus.plus/send'
        headers = {'Content-Type': 'application/json'}
        body = json.dumps(data).encode(encoding='utf-8')
        resp = requests.post(url, data=body, headers=headers)
        print(resp)
    except Exception as e:
        print('push+通知推送异常，原因为: ' + str(e))
        logger.info('push+通知推送异常，原因为: ' + str(e))
        print(traceback.format_exc())


# 忽略 requests 请求认证警告
requests.packages.urllib3.disable_warnings()
# 接口地址
url = "https://yqfk.pdsu.edu.cn/smart-boot/api/healthReport/saveHealthReport"
# 抓包签到请求头
headersValue = {
    'Host': 'yqfk.pdsu.edu.cn',
    'Accept': '*/*',
    'X-Device-Info': 'APP',
    'X-Terminal-Info': 'APP',
    'Accept-Language': 'zh-CN,zh-Hans;q=0.9',
    'Accept-Encoding': 'gzip, deflate, br',
    'Content-Type': 'application/json;charset=UTF-8',
    'Origin': 'https://yqfk.pdsu.edu.cn',
    'User-Agent': 'Mozilla/5.0 (iPhone; CPU iPhone OS 15_0_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 Html5Plus/1.0 (Immersed/44) uni-app SuperApp-10919',
    'Referer': 'https://yqfk.pdsu.edu.cn/serv-h5/',
    'X-Id-Token': 'eyJhbGciOiJSUzUxMiJ9....',
    'Content-Length': '1370',
    'Connection': 'keep-alive'
}

now = strftime("%Y-%m-%d %H:%M:%S", localtime())

# 抓包请求 json
jsonValue = {"needUpdate": 1,
             "isAgree": 'true',
             "createTime": now,
             "age": "20",
             "phone": "181xxxx",
             "address": "河南省xxx",
             ... ...
             "lastVaccineDate": '',
             "notVaccineType": '',
             "notVaccineOtherResult": '',
             "remark": ''}


# 打卡方法
def doSign(url, jsonValue, headersValue):
    logging.info("开始执行打卡程序...")
    try:
        r = requests.post(url, json=jsonValue, headers=headersValue, verify=False)
        global results
        results = json.loads(r.text)
    except Exception as e:
        logging.info("程序异常：" + str(e))
    logging.info(strftime("%Y-%m-%d %H:%M:%S", localtime()))
    return results


open('./log.txt', mode='w', encoding='utf-8')
x = doSign(url, jsonValue, headersValue)
logging.info("程序执行结果：" + str(x))
sendPushplus(token)

```

我的代码里加了`PushPlus`的推送，每天的打卡情况会推到这个公众号，大家也可以关注一下修改为自己的token，如果不想加就删除相关的代码。需要改的地方一是推送token，二是两个抓包获取的json数据。

## 部署教程

### 方法一：部署至Linux服务器

先在linux找个合适的文件夹，创建虚拟环境（可选，参考我[另一篇博客](https://blog.csdn.net/Magic_Zsir/article/details/122548170)），如果使用虚拟环境则先激活，然后使用pip install 安装所需包，创建一个python文件将代码粘贴进去，注意把相关信息改为自己的，然后使用crontab创建定时任务就行了。

### 方法二：腾讯云函数

有空再写

### 方法三：Github Action

同上


> 哪里不懂可以评论区留言

