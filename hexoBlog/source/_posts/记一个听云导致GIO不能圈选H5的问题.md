---
title: 记一个听云导致GIO不能圈选H5的问题
date: 2020-11-13 11:18:33
tags: hook
---

# 记一个听云导致GIO不能圈选H5的问题

产品侧是一直是使用 GIO 在 APP 内圈选 H5 页面来进行埋点的。然后有一天产品跑过来跟我说，GIO 在圈选 H5 元素时，不能弹出圈选操作的界面了，于是就走上了排查的不归路。

## 联系 GIO 技术客服

联系了 GIO 技术客服，给他们提供了不能圈选 H5 的视频和 SDK 版本号。另外也告知了 GIO，我们最近是把 UIWebView 升级到了 WKWebView。

```
- Growing (2.7.8)
  - GrowingAutoTrackKit (2.7.7):
    - GrowingCoreKit (~> 2.7.7)
  - GrowingCoreKit (2.7.8):
    - Growing (~> 2.7.8)
  - GrowingTouchKit (0.1.5):
    - GrowingAutoTrackKit (>= 2.6.7)
```

不久就收到了 GIO 的回复，他们说最新版本的 2.8.7 是没有问题的，于是我就升级到了 2.8.7 去试下，试完后还是有不能圈选的问题。

然后我就去看 GIO 是怎么做圈选 H5 的，发现他们是 hook 了 didFinishNavigation，在 H5 加载后就会注入一段 JS 脚本。

```
try {
    (function() {
        try {
            var p = document.createElement('script');
            p.src = 'https://assets.giocdn.com/sdk/hybrid/2.0/gio_hybrid.min.js?sdkVer=2.8.7&platform=iOS';
            document.head.appendChild(p);
        } catch(e) {}
    })()
} catch(e) {}

try {
    (function() {
        try {
            var p = document.createElement('script');
            p.src = 'https://assets.giocdn.com/sdk/hybrid/2.0/vds_hybrid_circle_plugin.min.js?sdkVer=2.8.7&platform=iOS';
            document.head.appendChild(p);
        } catch(e) {}
    })()
} catch(e) {}
```


然后我 hook  evaluateJavaScript:completionHandler: 方法，打开圈选，打开一个百度链接。准备去圈选百度按钮，可以发现执行的下面的脚本：

```
2019-12-25 15:58:27.030537+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(362.333328, 170.666656); } catch (e) { }

2019-12-25 15:58:27.047863+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(362.333328, 170.666656); } catch (e) { }

2019-12-25 15:58:27.062563+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(362.666656, 170.666656); } catch (e) { }

2019-12-25 15:58:27.080025+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(362.666656, 170.666656); } catch (e) { }

2019-12-25 15:58:27.095867+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(362.666656, 170.666656); } catch (e) { }

2019-12-25 15:58:27.112504+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(362.666656, 170.666656); } catch (e) { }

2019-12-25 15:58:27.129417+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(363.000000, 170.666656); } catch (e) { }

2019-12-25 15:58:27.196647+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(363.333328, 170.666656); } catch (e) { }

2019-12-25 15:58:27.833718+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(363.666656, 170.666656); } catch (e) { }

2019-12-25 15:58:29.405819+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(364.000000, 170.666656); } catch (e) { }

2019-12-25 15:58:29.420993+0800 App[8816:2927073] ssssss try { _vds_hybrid.hoverOn(364.000000, 170.666656); } catch (e) { }

2019-12-25 15:58:29.446618+0800 App[8816:2927073] ssssss try { _vds_hybrid.cancelHover(); } catch (e) { }

2019-12-25 15:58:29.463312+0800 App[8816:2927073] ssssss try { _vds_hybrid.findElementAtPoint("seq5"); } catch (e) { }

2019-12-25 15:58:29.480739+0800 App[8816:2927073] ssssss try { _vds_hybrid.pollEvents(); } catch (e) { }

2019-12-25 15:58:29.482341+0800 App[8816:2927073] ssssss try { _vds_hybrid.pollEvents(); } catch (e) { }

```

说明他这是通过 Native 拿到的圈选坐标，然后把坐标传给 JS 去定位到 H5 元素的。这段打印也另外说明了，GIO 注入的脚本是成功的， 既然注入是成功的，那为什么没有弹出圈选界面呢？

<!--more-->

## 联系听云技术客服

既然是 GIO 那边自己的 DEMO 是没有问题的，那有可能是我们 APP 的环境比较复杂，我就让 GIO 把他们的 DEMO 发给我，GIO 提供的 DEMO 确实是可以圈选 H5 并且能够弹出圈选界面，那此时就感觉 GIO 可能跟我们第三方的某个 SDK 冲突了，然后导致了 H5 圈选失效了。

然后我就在我们的 WebViewController 组件里去逐一排查，看看哪个是可能由于 GIO 冲突的第三方 SDK，排查一圈后，发现听云 SDK 是比较可疑的，于是就联系听云技术客服，询问他们是否有对 WebView 做什么事情，听云客服告知会有一个 WebView 采集模块，然后这个模块可以通过后台来决定是否开启。当我们关掉 WebView 采集模块后，发现是 WebViewController 组件可以正常圈选 H5 了。

{% asset_img "tingyunend.jpg" %}


为了进一步证明是听云 SDK 导致的 GIO 不能圈选 H5 的，于是我就把听云 SDK 集成到 GIO 提供的 DEMO 里去试一下，结果却是可以成功圈选，这就有点匪夷所思了。

这里就会产生冲突了：

> * 我们 APP 里，GIO + 听云，会导致 GIO 不能圈选

> * DEMO APP 里，GIO + 听云，不会影响 GIO 圈选功能


## 缩小范围到 initialize 方法

既然这里有一个冲突点，那就是需要看看我们 APP 里的 WebViewController 和 DEMO 里的 WebViewController 有什么不一样。

经过一轮对比，发现了几个现象：

> * 我们 APP 里，GIO + 听云，并且 WebViewController 实现了 initialize 方法，GIO 不能圈选 H5

> * 我们 APP 里，GIO + 听云，并且 WebViewController 不实现 initialize 方法，GIO 能圈选 H5

> * DEMO APP 里，GIO + 听云，并且 WebViewController 实现了 initialize 方法，GIO 不能圈选 H5

> * DEMO APP 里，GIO + 听云，并且 WebViewController 不实现 initialize 方法，GIO 能圈选 H5


那这里就很明确了，原因就是只要实现了 initialize 方法，就会导致 GIO 不能圈选。


## 成立专案组

那为什么 initialize 方法会导致 GIO 不能圈选呢？通过调试发现听云会 hook initialize 方法。

{% asset_img "hookinitialize.jpg" %}

于是就联系 GIO 和 听云技术客服，跟他们说这个需要三方一起来看这个问题，于是就成立了 GIO 不能圈选 H5 的专案组。

再经过一番讨论和分析，已经确定了根本原因了：

> GIO 在每个页面初始化的时候动态创建一个子类继承当前页面(是通过 KVO 来做这种子类化的)，并给动态创建的子类添加 viewWillAppear 方法，而听云这个时候也会在动态生成的子类中添加  viewWillAppear 方法，由于听云添加此方法的时机是先与 GIO 的，所以会导致 GIO 动态添加方法失败，从而导致 GIO 不能执行添加的 viewWillAppear 方法里的自定义逻辑，从而导致圈选失败。

两张图来解释他们的 hook 时序和 hook 后的状态，由于我们 APP 引入了 Aspect，所以这里也把 Aspect hook 的逻辑化进来了，序号（1~8）代表执行顺序。

{% asset_img "hook时序.jpg" %}
{% asset_img "hook后的状态.jpg" %}


> 听云这个时候是不应该针对动态生成的子类做处理的，需要一个过滤处理。而 GIO 在动态添加方法失败时，需要有一个兜底方案，目前是没有这么一个兜底方案的，所以这个问题是 GIO 和 听云共同造成的问题，并且是需要他们共同来解决的。


听云大概的逻辑如下：

```
+ (void)nbs_jump_initialize:(SEL)sel {
    NSString *clsName = [NSString stringWithUTF8String:class_getName(self)];
    NSString *className = NSStringFromClass(self.class);
    
    // 当 self.class (子类化都会重写 self.class 的返回值) 与 对象的 isa 不是一个类时，那说明该类是一个动态创建的子类。
    // 所以听云可以添加的过滤逻辑。
    if (![className isEqualToString:clsName]) {
        return;
    }
    
    // hook 声明周期方法
    [NBSUIAgent hook_viewDidLoad:self];
    [NBSUIAgent hook_viewWillAppear:self];
    [NBSUIAgent hook_viewDidAppear:self];
}
```

## 解决方案

遇到这类问题，解决方案要两手抓。

* 临时解决：针对 GIO 和听云 hook 的 initialize 方法不去重写，换一个地方去做业务处理，这样就避免了听云给动态创建子类添加方法了。

* 长期解决方案：听云计划升级 SDK，来兼容处理有 GIO SDK 的场景，而 GIO 已经计划发版本去升级来兼容存在听云 SDK 的场景。







