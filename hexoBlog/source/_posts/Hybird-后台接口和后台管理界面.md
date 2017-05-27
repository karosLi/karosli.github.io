---
title: Hybird-后台接口和后台管理界面
date: 2017-05-27 23:42:20
tags: hybird
categories: hybird
---

# Hybird-后台接口和后台管理界面
## 目录
> * 背景
> * 接口格式
> * 管理界面
> * 后台逻辑

## 背景
由于线上乐刻客户端 `App` 第一次打开平台 `H5` 需要几秒的加载时间，这个体验对用户来说并不友好，为了让用户跳转 `H5` 和跳转到原生一样的用户体验，就需要把 `H5` 相关的离线资源包下发给客户端，客户端就可以使用离线资源来代替实际网络请求，节省用户等待时间和流量消耗。这里就需要后台来负责离线资源包的管理和下发。
<!--more-->

## 接口格式
`offlineResourceInfo` 接口参数:

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

## 管理界面
### 添加升级资源包
![](http://olf3t4omk.bkt.clouddn.com/add_deupgrade_offline_package.png)

资源包需上传到七牛空间 `offlineh5`, 路径为 `http://cdn.xxx.com/upgrade/[模块名]/activity.full_1.0.0.zip`


### 添加降级资源包
![](http://olf3t4omk.bkt.clouddn.com/add_upgrade_offline_package.png)

资源包需上传到七牛空间 `offlineh5`, 路径为 `http://cdn.xxx.com/degrade/[模块名]/activity.full_1.0.0.zip`

## 后台逻辑

#### App 启动
`App` 第一次请求时， `resourceVersionList` 为空，服务器需要返回所有模块最新的全量资源。

#### App 升级逻辑
`App` 后续请求都会带上本地最新的`resourceVersionList`，服务器遍历`resourceVersionList`，并和服务器上配置的所有升级模块最新版本进行比较，

* 如果升级模块版本与 `App` 本地版本相隔一个版本，就下发增量包。
* 如果升级模块版本比 `App` 本地版本相隔多个版本（跨版本），就下发全量包。
* 如果某个模块不要升级资源包，后台接口就不需要返回该模块的信息。
    
#### App 降级逻辑
`App` 后续请求都会带上本地最新的`resourceVersionList`，服务器遍历`version list`，并和服务器上配置的所有降级模块源版本进行比较，

* 如果降级模块源版本与 `App` 本地版本相同，就下发降级包。
* 当降级逻辑和升级逻辑同时满足条件时，只启用降级逻辑。


