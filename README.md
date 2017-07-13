<p align="center">

<img src="Images/banner.jpg" alt="Reveal" title="Reveal"/>

</p>

# 揭秘 APM iOS SDK 的实现

## 前言

有关 APM 的技术文章非常多，但大部分文章都只是浅尝辄止，并未对实现细节进行深挖。本文旨在通过剖析 SDK 具体实现细节，揭露知名 APM 厂商的 iOS SDK 背后的秘密。

## 页面渲染时间

页面渲染的监控，比较容易想到的是通过 hook 页面的几个关键的生命周期方法，例如 `viewDidLoad`、`viewDidAppear:` 等，从而计算出页面渲染时间，最终发现慢加载的页面。然而如果真正通过上述思路去着手实现的时候，便会遇到难题。在 APM SDK 中如何才能 hook 应用所有页面的生命周期的方法呢？如果尝试 hook `UIViewController` 的方法又会如何呢？hook `UIViewController` 的方法明显不可行，原因是他只会作用 `UIViewController` 的方法，而应用中大部分的视图控制器都是继承自 `UIViewController` 的，所以这种方法不可行。但是听云 SDK 却能够实现。页面 Hook 的逻辑主要是 `_priv_NBSUIAgent` 类中实现的，下面是 `_priv_NBSUIAgent` 类的定义，其中 `hook_viewDidLoad` 等几个方法便是线索。

```
                    ; @class _priv_NBSUIAgent : NSObject {
                                       ;     +hookUIImage
                                       ;     +hookNSManagedObjectContext
                                       ;     +hookNSJSONSerialization
                                       ;     +hookNSData
                                       ;     +hookNSArray
                                       ;     +hookNSDictionary
                                       ;     +hook_viewDidLoad:
                                       ;     +hook_viewWillAppear:
                                       ;     +hook_viewDidAppear:
                                       ;     +hook_viewWillLayoutSubviews:
                                       ;     +hook_viewDidLayoutSubviews:
                                       ;     +nbs_jump_initialize:
                                       ;     +hookSubOfController
                                       ;     +hookFMDB
                                       ;     +start
                                       ; }
```