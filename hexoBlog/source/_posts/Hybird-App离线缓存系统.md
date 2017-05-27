---
title: Hybird-App离线缓存系统
date: 2017-05-27 23:42:20
tags: hybird
categories: hybird
---

# Hybird-App离线缓存系统
## 目录
> * 背景
> * 接口格式
> * 离线资源包格式
> * 离线资源下发
> * 离线资源缓存


## 背景
由于线上乐刻客户端 `App` 第一次打开平台 `H5` 需要几秒的加载时间，这个体验对用户来说并不友好，为了让用户跳转 `H5` 和跳转到原生一样的用户体验，就需要把 `H5` 相关的离线资源包下发给客户端，客户端就可以使用离线资源来代替实际网络请求，节省用户等待时间和流量消耗。并且随着业务的发展，不同的业务升级进度不一样，就需要 `App` 支持模块化升级。

<!--more-->

## 接口格式
`offlineResourceInfo` 接口请求方法: `POST`

`offlineResourceInfo` 接口请求参数:

Json 形式：

```
{
    //"appVersion": "2.4.0", 可以去掉，因为请求头会包含
    "resourceversionList": [{
    	"name": "m",
    	"version": "1.0.0"
    },{
    	"name": "coach",
    	"version": "1.0.0"
    },{
    	"name": "activity",
    	"version": "1.0.0"
    }]
}
```

Form 表单形式：

```
resourceNames=m,coach,activity&resourceVersions=1.0.0,1.0.0,1.0.0
```


`offlineResourceInfo` 接口返回结构体:

```
{
    "data": {
    	"resourceList": [{
    			"name": "m",
    			"version": "1.0.1",
    			"url": "http://cdn.xxx.com/resource/m/m_update_1.0.0_1.0.1.zip",
    			"md5": "a4d7feecbcae8e2ccba3b5ba90aa8a83",
    			"isfull": false
    		},{
    			"name": "coach",
    			"version": "1.0.1",
    			"url":  "http://cdn.xxx.com/resource/coach/coach_full_1.0.1.zip",
    			"md5": "a4d7feecbcae8e2ccba3b5ba90aa8a83",
    			"isfull": true
    		}
    	]
    }
}
```

参数说明：

```
"name": 模块名
"version": 升级版本
"url": 资源包下载地址
"md5": 资源包 md5
"isfull": 是否是全量升级包
```

## 离线资源包格式
增量和全量升级包拥有同样的结构，包含 `config.json` 文件和资源文件。

`config.json` 格式如下，`version` 记录的是下发的资源版本号，`validate` 记录的是所有文件的路劲和相应的 `md5 hash` 值。

```
{
	"version": "1.0.1",
	"validate": [{
	   "path": "528/app.min.cs",
	   "md5": "md5($cssFileContent)"
	},{
	   "path": "528/app.min.js",
	   "md5": "md5($jsFileContent)"
	}]
}
```

具体的资源与 `config.json` 平级。

```
--[528]
----app.min.css
----app.min.js

```


## 离线资源下发
### 下发时机
`App` 启动时~~设置定时器定时器~~-> `wifi` 环境下 -> 请求服务器接口获取 `offlineResourceinfo` 接口 `response`，根据`resourceList` 的结果来决定是否需要更新，需要更新的模块可以下载 `zip` 文件

### 资源包存放目录
按模块目录存放资源包。其中目录 `moduleszip` 用于存放资源压缩包的路径，目录 `modules` 用于存放解压后的压缩包路径。

所以每次请求 `offlineResourceInfo` 接口的时候，也需要遍历所有模块目录下的 `config.json` 去获取资源版本号。所以第一次请求的话，由于本地目录是空的，对于接口 `offlineResourceInfo` 的参数 `resourceversionList` 也是空的。

```
--[offlineResource]
----[moduleszip]
------[m]
--------[zip]
----------m.update.1.0.0_1.0.1.zip
----------m.full.1.0.1.zip
--------[temp] 解压临时目录
--------[backup] 原有资源备份目录

------[coach]
--------[zip]
----------coach.update.1.0.0_1.0.1.zip
----------coach.full.1.0.1.zip
--------[temp] 解压临时目录
--------[backup] 原有资源备份目录

----[modules]
------[m]
--------config.json
--------其他资源文件

------[coach]
--------config.json
--------其他资源文件

----[modulesflag]
------[m]
--------flag.json

------[coach]
--------flag.json
```

### 资源包解压
#### 解压
当下载完资源包，解压之前需要根据接口返回的 `md5` 值来校验资源包的合法性。

校验子文件过程：需要结合 `config.json` 和资源来校验每个文件的合法性，如果不合法，~~就不添加该资源文件~~ 就不保留整个资源包。

#### 更新资源
增量更新：文件的替换和增加。而且需要合并新老 `config.json`。
全量更新：覆盖模块目录。

模块资源包更新之前，需要先备份之前的模块资源。例如：拷贝目录 `offlineResource/modules/m` 到目录`offlineResource/moduleszip/m/backup` 来进行备份。


#### 容错处理
需要设置标志位，并持久化到 `flag.json`:

```
{
    "doingUpgradeFlag": false, // false, 表示升级过程正常，否则升级过程有错误。
    "doingBackupFlag": false // false, 表示备份过程正常，否则备份过程有错误。
}
```

* 正在升级标志位 (总标志位)
* 正在备份标志位 (是正在升级标志位的一个子集)

正在升级的过程包括， `md5` 校验资源包，`md5` 校验每个资源文件，备份和更新过程。

如果整个升级过程中发生普通错误，恢复所有标志位，然后结束升级流程。

如果整个升级过程中发生崩溃或者被杀掉进程，App 再次启动后。此时 `正在升级标志位` 没有复位 (升级失败)，有以下几种情况需要做容错处理:

1. 如果 `正在备份标志位` 也没有复位 (备份失败)，此时并不会影响目标模块资源，直接恢复所有标志位。
2. 如果 `正在备份标志位` 已经复位 (备份成功)，先清空目标模块资源，然后做回滚操作：
    * 回滚成功，直接复位所有标志位。
    * 回滚失败，先清空目标模块资源：
        1. 正常失败，恢复所有标志位。
        2. App 崩溃或被杀掉进程，Nothing To Do.


#### 容错副作用
当 `正在升级标志位` 没有复位时。此时 `App webview` 请求不走缓存系统。
否则 `App webview` 可以继续使用缓存系统。

#### 升级流程图

```flow
st=>start: 开始
e=>end: 升级结束
success=>operation: 升级成功
清空modulezip/[模块]
appStart=>operation: App 启动
appKill=>condition: 是否App
崩溃或被
杀掉进程
appKill1=>condition: 是否App
崩溃或被
杀掉进程
offlineRequest=>condition: 是否offline
ResourceInfo
接口返回数据
downloadZip=>operation: 下载模块Zip资源包
rollbackModule=>condition: 回滚模块
clearModule=>operation: 清空模块资源
clearModule1=>operation: 清空模块资源
setUpgradingTrue=>operation: 正在升级Flag=Yes
正在备份Flag=Yes
setUpgradingFalse=>operation: 正在升级Flag=No
setBackupingTrue=>operation: 正在备份Flag=Yes
setBackupingFalse=>operation: 正在备份Flag=No
upgradingIsTrue=>condition: 正在升级
Flag==Yes
backupingIsTrue=>condition: 正在备份
Flag==Yes
setUpgradingFalse1=>operation: 正在升级Flag=No
正在备份Flag=No
setUpgradingFalse2=>operation: 正在升级Flag=No
正在备份Flag=No

validateZipMd5Cond=>condition: 校验模块Zip
的Md5
unZipCond=>condition: 清除temp目录
并解压模块Zip
到temp目录
validateFilesMd5Cond=>condition: 校验模块
子资源的Md5
backupTargetModuleCond=>condition: 清空备份
并备份目标
模块资源
upgradeTargetModuleCond=>condition: 升级目标
模块资源


st->appStart->upgradingIsTrue
upgradingIsTrue(no)->offlineRequest
upgradingIsTrue(yes)->backupingIsTrue
backupingIsTrue(no)->clearModule->rollbackModule
backupingIsTrue(yes)->setUpgradingFalse1->offlineRequest
rollbackModule(no)->clearModule1->appKill(right)
rollbackModule(yes,right)->setUpgradingFalse1->offlineRequest
offlineRequest(no)->e
offlineRequest(yes)->downloadZip->setUpgradingTrue->validateZipMd5Cond
validateZipMd5Cond(no)->appKill(right)
validateZipMd5Cond(yes)->unZipCond
unZipCond(no)->appKill(right)
unZipCond(yes)->validateFilesMd5Cond
validateFilesMd5Cond(no)->appKill(right)
validateFilesMd5Cond(yes)->backupTargetModuleCond
backupTargetModuleCond(no)->appKill(right)
backupTargetModuleCond(yes)->setBackupingFalse->upgradeTargetModuleCond
upgradeTargetModuleCond(no)->appKill1
upgradeTargetModuleCond(yes)->setUpgradingFalse->success->e
appKill(no)->setUpgradingFalse2->e
appKill(yes,right)->appStart(right)
appKill1(no)->e
appKill1(yes,right)->appStart(right)

```


## 离线资源缓存
### 使用缓存时机
只针对 `webview` 的以 `xxx.com` 为主域名的请求进行拦截，然后根据请求链接，找到具体文件缓存。

找具体文件缓存的方式：
> * 遍历所有模块下的 `config.json` 文件，看能否找到具体的资源文件。这样效率会比较慢，但是更适合现有的场景。
> * ~~根据链接的一级路径找到对应模块下的 `config.json` 文件，看能否找到具体的资源文件。这样效率会比较高，但是目前链接的一级路径并不规范。~~
> * 直接根据请求 `URL` 的 `PATH` 去本地查找是否存在具体的资源文件。`PATH` 的一级路径代表模块名，剩余部分代表资源路径。

App 这里会使用第二种方式去找缓存文件，这样的话就需要前端小伙伴规范链接路径。

~~拿到缓存文件之后，需要再次校验缓存文件的合法性，合法则使用缓存，不合法就需要下面的容错处理。~~

### 使用缓存容错处理
~~如果找到的缓存文件已经损坏或者不存在（解压过程被中断，杀掉进程或者 `crash`），此时需要继续走网络，并且把网络结果进行 `md5` 校验，如果合法，需要把该结果保存到缓存系统，如果不合法，不做处理。~~


