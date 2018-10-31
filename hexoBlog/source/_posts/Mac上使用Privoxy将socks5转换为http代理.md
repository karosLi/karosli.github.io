---
title: Mac上使用Privoxy将socks5转换为http代理
date: 2018-03-05 15:32:40
tags: tool
categories: tool
---

# Mac上使用Privoxy将socks5转换为http

`shadowsocks` 挺不错的，但是有些时候需要使用`http`代理来爬墙。这时候可以使用`privoxy`来将 `socks5` 代理转换为 `http` 代理。

### 配置
首先，确保 `shadowsocks` 已经正常起来的，默认的本地`socks5`端口号为 `1080`,可以使用 `netstat` 和 `lsof` 命令查看端口情况。
    
安装 `privoxy`, `mac` 使用 `brew install privoxy` 即可，安装完成后，修改 `privoxy` 配置文件。

如果安装过程报如下错误，先手动创建这个目录 `sudo mkdir -p  /usr/local/sbin`, 再添加修改权限 `chmod 777 /usr/local/sbin`。

```
Error: The `brew link` step did not complete successfully
The formula built, but is not symlinked into /usr/local
Could not symlink sbin/privoxy
/usr/local/sbin is not writable.

```


编辑 `config` 文件：

```
vim /usr/local/etc/privoxy/config
```
修改内容如下：

```
forward-socks5 / 127.0.0.1:1080 .listen-address 127.0.0.1:8118
```
`listen-address` 默认是监听本地`8118`端口，如果端口没有被占用，可以不用修改

  启动 `privoxy`

```
/usr/local/sbin/privoxy /usr/local/etc/privoxy/config
```
可以使用 `ps aux|grep privoxy`和 `lsof -i:8118`来检查是否成功启动

<!--more-->

### git 上使用代理
  正常情况下,可以使用`http`代理了，代理地址`http://127.0.0.1:8118`
示例
例如，可以给 git 设置 http 或 https 代理,设置方式如下：

```
git config --global http.proxy http://127.0.0.1:8118
git config --global https.proxy http://127.0.0.1:8118
```
取消代理配置：

```
git config --global --unset http.proxy
git config --global --unset https.proxy
```
和其他 `git` 的配置一样，不使用命令行而是直接修改相应的 `.gitconfig` 文件也是可以的。当然，如果不想修改 git 配置，而只是想临时使用一下，可以使用 `-c` 参数：

```
git -c https.proxy=http://127.0.0.1:8118 clone --depth=1 https://github.com/xxx/xxx
```

### 全局代理配置
在命令行终端中输入如下命令后，该终端即可翻墙了。

```
export http_proxy='http://localhost:8118'
export https_proxy='http://localhost:8118'
```

他的原理是讲`socks5`代理转化成`http`代理给命令行终端使用。
如果不想用了取消即可。

```
unset http_proxy
unset https_proxy
```

如果关闭终端窗口，功能就会失效，如果需要代理一直生效，则可以把上述两行代码添加到 `~/.bash_profile` 文件最后。(如果你是使用 `zsh` 那么就是 `~/.zshrc`)

```
vim ~/.zshrc

# 添加如下内容
export http_proxy='http://localhost:8118'
export https_proxy='http://localhost:8118'
```

使配置立即生效

```
source  ~/.zshrc
```

还可以在 `~/.zshrc` 里加入开关函数，使用起来更方便。

```
function proxy_off(){
    unset http_proxy
    unset https_proxy
    echo -e "已关闭代理"
}

function proxy_on() {
    export http_proxy="http://127.0.0.1:8118"
    export https_proxy=$http_proxy
    echo -e "已开启代理"
}
```

使用的时候就是简单运行函数。
开启：

```
proxy_on
```

关闭

```
proxy_off
```

