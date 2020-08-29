# 接口文档

* aes加密:
	* 算法: `aes/ecb/128bit/pkcspadding7`
	* 测试密钥: `12345678abcdefgh`，联调测试使用
	* 正式密钥待定

* 参数签名
	* 算法: md5("data" + <加密字符串> + "ts" + <时间戳> + <签名密钥>) 转小写
	* 测试签名密钥: xiaoxiong
	* 正式密钥待定

## 【GET】登录接口

* Url
	* `http://<ip>/login?data=<加密字符串>&ts=<时间戳>&sign=<参数签名>`
	* 参数说明
		* data: aes加密的json字符串
		* ts: 毫秒级时间戳
		* sign: 参数签名

* 加密字符串data的明文json
```
{
	"did": "<string16位: 设备标识>",
	"account": "<string4~16位: 账号>",
	"md5passwd": "<string32位:md5密码>",
	"version": "<string5位: 客户端版本号, 比如12.02>",
	"ts": <int64: 毫秒级时间戳>,
	"noncestr": "<string16位: 随机字符串,服务器会原样返回,客户端收到response需要比对>"
}
```

* 响应
	* response body `aes/ecb/128bit/pkcspadding7`加密
```
登录成功时 response body明文
{
	"errcode": <int32: 200,表示成功>,
	"uid": <int32: 用户uid>,
	"noncestr": "<string16位: 客户端传的noncestr>",
	"token": "<string32位: 当前登录有效的token,发心跳时使用>"
}

登录失败时 response body明文
{
	"errcode": <int32: 错误码>
}
```

* 接口处理
1. 验签sign,不通过返回status 404
	1. 检查ts，与当前时间差不能超过2小时
	2. 计算参数签名，比对sign
	3. 检查缓存sign值，2小时内不能重复
	4. sign记入缓存，ttl=2小时
2. data解密,解密失败返回403
	1. aes解密data
	2. 比较ts，和接口相同，且与当前时间差不能超过2小时
3. 登录校验
	1. 检查账号密码，取得uid
	2. 更新did对应的uid
	3. 更新uid下的did对应的token


* 客户端处理
1. 密码计算MD5，取设备did，随机生成16位noncestr，组装参数json
2. 把json `aes/ecb/128bit/pkcspadding7`加密为data
3. 计算sign签名，调接口
4. 接口返回后对response进行aes解密
5. 如果errcode=200，比较noncestr是否和自己生成的一样，都通过说明登录成功

## 【GET】心跳接口

* Url
	* `http://<ip>/heart?data=<加密字符串>&ts=<时间戳>&sign=<参数签名>`
	* 参数说明
		* data: aes加密的json字符串
		* ts: 毫秒级时间戳
		* sign: 参数签名

* 加密字符串data的明文json
```
{
	"did": "<string16位: 设备标识>",
	"uid": <int32: 用户uid,登录接口返回的uid>,
	"token": "<string16位: 登录token>",
	"version": "<string5位: 客户端版本号, 比如12.02>",
	"ts": <int64: 毫秒级时间戳>,
	"noncestr": "<随机字符串,服务器会原样返回,客户端收到response需要比对>"
}
```

* 响应
	* response body `aes/ecb/128bit/pkcspadding7`加密

```
接口校验心跳成功时 response body明文
{
	"errcode": <int32: 200表示成功>,
	"noncestr": "<string16位: 客户端发送的noncestr>"
}

接口校验心跳失败时 response body明文
{
	"errcode": <int32: 非200>
}
```

* 接口处理
1. 验签sign,不通过返回status 404
	1. 检查ts，与当前时间差不能超过2小时
	2. 计算参数签名，比对sign
	3. 检查缓存sign值，2小时内不能重复
	4. sign记入缓存，ttl=2小时
2. data解密,解密失败返回403
	1. aes解密data
	2. 比较ts，和接口相同，且与当前时间差不能超过2小时
3. 取did对应的uid，和接口uid比较，不一样则心跳校验失败
4. 取uid下的did对应的token，与接口传的token比较，一样说明心跳校验成功
5. 记录did的最近活跃时间

* 客户端处理
1. 如果用户已登陆，则每分钟发送一次心跳
2. 如果心跳接口返回失败，则强制退出登录(网络波动问题可以多重试几次)


## 计划任务

* `clean_did.php` 定时清理2天不活跃的did及其token

## 服务器脚本同步用户账号密码

* `account_sync.php` 把用户账号密码写入缓存
