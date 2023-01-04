---
title: 本地环境如何访问需要 ssh 通道的服务（mysql、redis 等）
date: 2022-12-08 16:13:49
tags:
---
# 前言
因为日常工作中，测试服务器需要 ssh 通道才能连接，当测试反馈 bug 的时候，而本地环境缺失数据或者造对应数据无法复现从而导致解决问题低下，这个时候如果本地环境可以连接的测试的数据库，那调试真是如鱼得水。


# 实现步骤
## 下载模板
将下方的代码生成 filename.sh 放置在对应 UNIX 系统中（windows 系统可以安装 wsl， 参照 [wsl 安装](https://blog.csdn.net/weixin_43832080/article/details/127385743)）

```bash
#!/bin/bash

filter="IPv4"

#1. 环境判断，方便切换不同环境的 mysql
if [ $1 = "test" ] ; then
	#本地需要映射的端口
	port=33061 
	#mysql服务IP&端口, 切记前面有一个:, 举例 :10.1.1.1:3306
	portMap=""
	#ssh的端口, 一般情况不用修改
	serverPort=22
	#认证证书文件绝对地址, 通过运维提供的跳板机后台下载的 pem 文件， 举例 /home/public/.ssh/public.pem
	pemPath=""
	#账户信息&服务器IP, 举例 public@40.40.40.1
	username=""
	#密码, ssh密钥文件密码 || 通行短语
	password=""
	#关键字
	key=".pem"
else
	#可以复制填写上述内容，实现 master 环境的连接
fi	


#2. 查询端口是否被占用 (如果被其他程序占用可以切换 port)
# /usr/bin/lsof -i:33061 | grep IPv4
pids=$(/usr/bin/lsof -i:${port} | grep ${filter} | awk '{print $2}') 
if [ "$pids" != "" ]; then	
	echo " ssh alive " $pids
	exit
fi 


#3. 执行 ssh连接
expect << EOF
	set timeout 3
	spawn sudo ssh -i ${pemPath} -fN -L${port}${portMap} -p${serverPort} ${username}
	expect {
		 "*alive*" {send "alive\r"}
		 "*${key}" { send "${password}\r" }	
	}
	expect eof
EOF
```


## 修改模板文件
修改对应模板中变量声明定义，按照对应文档注释进行修改， 下方提供一个修改好的样例


```bash

#1. 环境判断，方便切换不同环境的 mysql
if [ $1 = "test" ] ; then
	port=33061
	filter="IPv4"
	portMap=":10.10.0.1:3306"
	serverPort=22
	pemPath="/home/public/.ssh/public.pem"
	username="public@40.4.15.183"
	password="1234567"
	#关键字
	key=".pem"
else
    #可以复制填写上述内容，实现 master 环境的连接
fi 
```
修改完成后， 保存即可


## 安装 expect 
**Expect** 是UNIX系统中用来实现自动化控制和测试的软件工具，用于解决交互式的处理，例如输入密码等。

相关命令：
```bash
sudo apt-get install expect
```

## 运行 shell 脚本
输入 filename test 的命令
```bash
 ./filename.sh test
```
 若端口被其他应用占用可以使用 sudo kill PID 来进行处理，个人建议更换新的端口号


## 修改代码的DB连接串
需要注意 host 地址改为本主机IP， 端口改为 filename.sh 修改 port 

```php
'db' => [
    'class'     => 'yii\db\Connection',
    'charset' => 'utf8mb4',
    // 测试环境配置
    'dsn'       => 'mysql:host=127.0.0.1;port=33061;dbname=db',
    'username' => 'root',
    'password' => '123456',
],
```



