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

我们先将目光转到另外一个更可疑的方法：`hookSubOfController`，具体实现如下：

```
void +[_priv_NBSUIAgent hookSubOfController](void * self, void * _cmd) {
    r14 = self;
    r12 = [_subMetaClassNamesInMainBundle_c("UIViewController") retain];
    var_C0 = r12;
    if ((r12 != 0x0) && ([r12 count] != 0x0)) {
            var_C8 = object_getClass(r14);
            if ([r12 count] != 0x0) {
                    r15 = @selector(nbs_jump_initialize:);
                    rdx = 0x0;
                    do {
                            var_98 = rdx;
                            r12 = [[r12 objectAtIndexedSubscript:rdx, rcx, r8] retain];
                            [r12 release];
                            if ([r12 respondsToSelector:r15, rcx, r8] == 0x0) {
                                    _hookClass_CopyAMetaMethod();
                            }
                            r13 = class_getName(r12);
                            rax = [NSString stringWithFormat:@"nbs_%s_initialize", r13];
                            rax = [rax retain];
                            var_A0 = rax;
                            rax = NSSelectorFromString(rax);
                            var_B0 = rax;
                            rax = objc_retainBlock(__NSConcreteStackBlock);
                            var_A8 = rax;
                            r15 = objc_retainBlock(rax);
                            var_B8 = imp_implementationWithBlock(r15);
                            [r15 release];
                            rax = class_getSuperclass(r12);
                            r15 = objc_retainBlock(__NSConcreteStackBlock);
                            rbx = objc_retainBlock(r15);
                            r13 = imp_implementationWithBlock(rbx);
                            [rbx release];
                            rcx = r13;
                            r8 = var_B8;
                            _nbs_Swizzle_orReplaceWithIMPs(r12, @selector(initialize), var_B0, rcx, r8);
                            rdi = r15;
                            r15 = @selector(nbs_jump_initialize:);
                            [rdi release];
                            [var_A8 release];
                            [var_A0 release];
                            rax = [var_C0 count];
                            r12 = var_C0;
                            rdx = var_98 + 0x1;
                    } while (var_98 + 0x1 < rax);
            }
    }
    [r12 release];
    return;
}
```

从 `_subMetaClassNamesInMainBundle_c` 的命名和传入的 "UIViewController" 参数，基本可以推断这个 C 函数是获取 MainBundle 中所有 `UIViewController` 的子类。而事实上，如果通过 LLDB 在这个函数 Call 完之后的那行汇编代码下断点，会发现返回的确实是 `UIViewController` 子类的数组。下面的 `if` 语句判断 `r12` 寄存器不为 `nil` 并且 `r12` 寄存器的 `count` 不等于0才执行 `if` 里面的逻辑，而 `r12` 寄存器存放的正是 `_subMetaClassNamesInMainBundle_c` 函数的返回值，也就是 `UIViewController` 子类的数组。

下面再来重点看看里面的 `do-while` 循环语句，循环判断的语句为 `var_98 + 0x1 < rax`，`var_98` 在循环开始的位置赋值 `rdx` 寄存器，`rdx` 寄存器在循环外初始化为0，所以 `var_98` 就是计数器，而 `rax` 寄存器则是赋值为 `r12` 寄存器的 `count` 方法，依此得出这个 `do-while` 循环实际就是遍历 `UIViewController` 子类的数组。遍历的行为则是通过 `_nbs_Swizzle_orReplaceWithIMPs` 实现 `initialize` 和 `nbs_jump_initialize:` 的方法交换。

`nbs_jump_initialize` 的代码如下：

```
void +[_priv_NBSUIAgent nbs_jump_initialize:](void * self, void * _cmd, void * arg2) {
    rbx = arg2;
    r15 = self;
    r14 = [NSStringFromSelector(rbx) retain];
    if ((r14 != 0x0) && ([r14 isEqualToString:@""] == 0x0)) {
            [r15 class];
            rax = _nbs_getClassImpOf();
            (rax)(r15, @selector(initialize));
    }
    rax = class_getName(r15);
    r13 = [[NSString stringWithUTF8String:rax] retain];
    rdx = @"_Aspects_";
    if ([r13 hasSuffix:rdx] == 0x0) goto loc_100050137;

loc_10005011e:
    if (*(int8_t *)_is_tiaoshi_kai == 0x0) goto loc_100050218;

loc_10005012e:
    rsi = cfstring__V__A;
    goto loc_100050195;

loc_100050195:
    __NBSDebugLog(0x3, rsi, rdx, rcx, r8, r9, stack[2048]);
    goto loc_100050218;

loc_100050218:
    [r13 release];
    rdi = r14;
    [rdi release];
    return;

loc_100050137:
    rdx = @"RACSelectorSignal";
    if ([r13 hasSuffix:rdx] == 0x0) goto loc_10005016b;

loc_100050152:
    if (*(int8_t *)_is_tiaoshi_kai == 0x0) goto loc_100050218;

loc_100050162:
    rsi = cfstring__V__R;
    goto loc_100050195;

loc_10005016b:
    if (_classSelf_isImpOf(r15, "nbs_vc_flag") == 0x0) goto loc_1000501a3;

loc_10005017e:
    if (*(int8_t *)_is_tiaoshi_kai == 0x0) goto loc_100050218;

loc_10005018e:
    rsi = cfstring____Yh;
    goto loc_100050195;

loc_1000501a3:
    rbx = objc_retainBlock(void ^(void * _block, void * arg1) {
        return;
    });
    rax = imp_implementationWithBlock(rbx);
    class_addMethod(r15, @selector(nbs_vc_flag), rax, "v@:");
    [rbx release];
    [_priv_NBSUIAgent hook_viewDidLoad:r15];
    [_priv_NBSUIAgent hook_viewWillAppear:r15];
    [_priv_NBSUIAgent hook_viewDidAppear:r15];
    goto loc_100050218;
}
```
`nbs_jump_initialize` 的代码有点长，但是从 `loc_1000501a3` 的例程可以观察到主要逻辑会执行 `hook_viewDidLoad`、`hook_viewWillAppear` 和 `hook_viewDidAppear` 三个方法，从而 hook 住 `UIViewController` 子类的这三个方法。

先以 `hook_viewDidLoad:` 方法为例讲解，下面这段代码可能有点晦涩难懂，需要认真分析

```
void +[_priv_NBSUIAgent hook_viewDidLoad:](void * self, void * _cmd, void * arg2) {
    rax = [_priv_NBSUIHookMatrix class];
    var_D8 = _nbs_getInstanceImpOf();
    var_D0 = _nbs_getInstanceImpOf();
    rbx = class_getName(arg2);
    r14 = class_getSuperclass(arg2);
    rax = [NSString stringWithFormat:@"nbs_%s_viewDidLoad", rbx];
    rax = [rax retain];
    var_B8 = rax;
    var_C0 = NSSelectorFromString(rax);
    r12 = objc_retainBlock(__NSConcreteStackBlock);
    var_D0 = imp_implementationWithBlock(r12);
    [r12 release];
    rbx = objc_retainBlock(__NSConcreteStackBlock);
    r14 = imp_implementationWithBlock(rbx);
    [rbx release];
    _nbs_Swizzle_orReplaceWithIMPs(arg2, @selector(viewDidLoad), var_C0, r14, var_D0);
    [var_B8 release];
    return;
}

```

`hook_viewDidLoad:` 方法中的参数 `arg2` 即是要 hook 的 `ViewController` 的类，
