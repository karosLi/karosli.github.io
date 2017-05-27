---
title: Hybird-离线资源生成工具
date: 2017-05-27 23:42:20
tags: hybird
categories: hybird
---

# Hybird-离线资源生成工具
## 目录
> * 背景
> * 离线资源生成工具
> * 前端协助

## 背景
由于线上乐刻客户端 `App` 第一次打开平台 `H5` 需要几秒的加载时间，这个体验对用户来说并不友好，为了让用户跳转 `H5` 和跳转到原生一样的用户体验，就需要把 `H5` 相关的离线资源包下发给客户端，客户端就可以使用离线资源来代替实际网络请求，节省用户等待时间和流量消耗。为了满足以上需求，就需要制作打包脚本和工具，来满足正常的运维。

<!--more-->
## 离线资源生成工具
离线资源的生成，我们提供了一个工具可以打包出增量和全量升级包。原理是根据 `git diff` 去比较两次 `commit`，然后只关注 `offlineResource` (与 `dist` 目录平级，发布包需要把 `dist` 目录内容拷贝到 `offlineResource`) 目录下的两次提交的文件差别，从而打出增量包。全量包就是整个 `offlineResource` 目录。

`offlineh5` 安装方法：

```
npm install -g offlineh5
```

使用方式：

```
offlineh5 -o package -r  http://github.com/xxx.git -f e24b8f0bb9a85c93c6965a906c1ea0448342821a -u gitusername -p gitpassword -z activity
```

参数说明：
> -o 资源包输出路径
> -r 仓库地址
> -u git 用户名
> -p git 用户密码
> -f 从哪个 commit 导出增量包
> -z 打出来的资源包前缀

打出来的离线资源包需要放到七牛 cdn 存储：

```
http://oq78hrbgk.bkl.clouddn.com/upgrade/activity/activity.full_0.1.1.zip
```

## 前端协助
### 遇到的问题
之前前端打包只把 `html`, `js`, `css` 导出到 `offlineResource` 目录下，没有图片，因为图片都放在 `cdn` 上，本地就没有任何的原始图片，这样导致三个问题：

1. `node` 脚本打出来的离线资源包并不包含图片。
2. 即使找到了原始图片，并不能保证原始图片的本地路径和cdn上的是一致的。
3. 线上现有 `cdn`一级路径比较混乱。

线上现有路径。
> http://cdn.leoao.com/le-activity/528/528-01.png
> http://cdn.leoao.com/activity/528/app.min.css
> http://cdn.leoao.com/activity/528/app.min.js


### 前端调整

1. 使用 `qtool` 脚本获取 `cdn` 上的所有图片，存放到本地作为原始图片，根据模块规范原始图片的路径。比如 `le-activity` 和 `activity` 需要统一成 `activity`。
2. 前端打包不仅输出 `html`, `js`, `css`，同时每次打包需要把原始图片拷贝到 `dist` 目录下。同时发布流程需要把 `dist` 目录内容拷贝到 `offlineResource`目录下。
3. 根据 `offlineResource` 目录，使用 `qtool` 脚本使用该目录下的所有资源路径作为 `cdn key`，然后把所有资源上传到 `cdn` 上。以后前端在打包之前开发的时候，完全可以使用本地的路劲作为相对路径提前配置路径，而不用考虑 `cdn` 的上传路径问题。

调整后，`offlineh5` 打包脚本可以根据 `offlineResource` 目录下的不同的 `commit`，`diff` 出两个版本之间差别，从而打出增量包和全量包。


#### 使用 qtool
`qtool` 安装方法：

```
npm install -g qtool
```

上传资源:

```
qtool upload  -f uploadfolder -a RSxpQIxNIS2vo0vuQR3HX701ddS9fdlUnQ5jV8ul -s xCLWczC5V5kyy7H85MNKNYcXT4wx9k5OzT7YDVFk -b mybucket -k activity -h olf3t4olk.bkt.clouddn.com
```

下载资源:

```
qtool download  -f downloadfolder -a RSxpQIxNIS2vo0vuQR3HX701ddS9fdlUnQ5jV8ul -s xCLWczC5V5kyy7H85MNKNYcXT4wx9k5OzT7YDVFk -b mybucket -k activity -h olf3t4olk.bkt.clouddn.com
```


参数说明:

```
-f, --folder <string> 
    上传和下载目录
    
-k, --keypreffix <string> 
    上传的时候，前缀会插入到 key 的前面。
    下载的时候，前缀会被用于过滤七牛的cdn url。
    
-a, --accessKey <string>
    access Key 七牛官网获取
    
-s, --secretKey <string> 
    Secret Key 七牛官网获取
      
-b, --bucket <string>
    上传和下载对象空间
    
-h, --hostUrl <string>
    七牛 host url，比如：http://cdn.xxx.com    
    
```


