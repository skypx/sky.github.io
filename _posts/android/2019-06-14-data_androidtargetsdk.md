---
layout: post
title: 关于minsdk, targetsdk, maxsdk
category: Android
tags: Android SDK 区别
---
* content
{:toc}

>详解targetsdk

## 语法
```java
<uses-sdk android:minSdkVersion="integer"
          android:targetSdkVersion="integer"
          android:maxSdkVersion="integer" />
```
## 属性
### `android:minSdkVersion`

一个用于指定应用运行所需最低 API 级别的整数。 如果系统的 API 级别低于该属性中指定的值，Android 系统将阻止用户安装应用。 您应该始终声明该属性.

注意：如果您不声明该属性，系统将假定默认值为“1”，这表示您的应用兼容所有 Android 版本。 如果您的应用并不兼容所有版本（例如，它使用 API 级别 3 中引入的 API），并且您尚未声明正确的 minSdkVersion，则当应用安装在 API 级别小于 3 的系统上时，应用将在运行时尝试访问不可用的 API 时发生崩溃。 因此，请务必在 minSdkVersion 属性中声明合适的 API 级别。

### `android:targetSdkVersion`

一个用于指定应用的目标 API 级别的整数。如果未设置，其默认值与为 minSdkVersion 指定的值相等。
该属性用于通知系统，您已针对目标版本进行测试，并且系统不应启用任何兼容性行为来保持您的应用与目标版本的向前兼容性。 应用仍可在较低版本上运行（最低版本为 minSdkVersion）。

在 Android 随着每个新版本的推出而进化的过程中，一些行为甚至是外观可能会发生变化。不过，如果平台的 API 级别高于您的应用的 targetSdkVersion 所声明的版本，系统就可以通过启用兼容性行为来确保您的应用继续以您所期望的方式工作。 您可以通过将 targetSdkVersion 指定为与应用所运行平台的 API 级别一致来停用此类兼容性行为。 例如，如果将该值设置为“11”或更高，系统便可在您的应用运行在 Android 3.0 或更高版本的平台上时对其应用新的默认主题 (Holo)，还可在您的应用运行在更大屏幕上时停用屏幕兼容性模式（因为对 API 级别 11 的支持隐含了对更大屏幕的支持）。

系统可根据您为该属性设置的值启用许多兼容性行为。 Build.VERSION_CODES 参考资料中的相应平台版本对其中的几种行为做了说明。

要让您的应用与各 Android 版本保持同步，您应该增加该属性的值，使其与最新 API 级别一致，然后在相应平台版本上对您的应用进行全面测试。

引入的版本：API 级别 4

### `android:maxSdkVersion`

一个指定作为应用设计运行目标的最高 API 级别的整数。
在 Android 1.5、1.6、2.0 及 2.0.1 中，系统会在安装应用以及系统更新后重新验证应用时检查该属性的值。 在任一情况下，如果应用的 maxSdkVersion 属性低于系统本身使用的 API 级别，系统均不允许安装应用。 在系统更新后重新验证这种情况下，这实际上相当于将您的应用从设备中移除。

为说明该属性在系统更新后对您的应用的影响，请看看下面这个示例：

Google Play 上发布了一个在其清单中声明了 maxSdkVersion="5" 的应用。 一位设备运行 Android 1.6（API 级别 4）的用户下载并安装了该应用。 几周后，该用户收到了 Android 2.0（API 级别 5）OTA 系统更新。 更新安装后，系统检查该应用的 maxSdkVersion 并顺利完成了对其的重新验证。 应用仍可照常工作。 不过，一段时间后，设备又收到了一个系统更新，这次是更新到 Android 2.0.1（API 级别 6）。 更新完成后，系统无法再重新验证应用，因为此时系统本身的 API 级别 (6) 已超过该应用支持的最高级别 (5)。 系统会使该应用对用户不可见，这实际上相当于将它从设备上删除。

警告：不建议声明该属性。 首先，没有必要设置该属性，将其作为阻止您的应用部署到 Android 平台新发布版本上的一种手段。 从设计上讲，新版本平台完全向后兼容。 只要您的应用只使用标准 API 并遵循部署最佳实践，应该能够在新版本平台上正常工作。 其次，请注意在某些情况下，声明该属性可能导致您的应用在系统更新至更高 API 级别后被从用户设备中移除。 大多数可能安装您的应用的设备都会定期收到 OTA 系统更新，因此您应该在设置该属性前考虑这些更新对您的应用的影响。

# 总结

## 参考Google:
[Targetsdk](https://developer.android.com/guide/topics/manifest/uses-sdk-element.html?hl=zh-cn)

* minSdkVersion 和 maxSdkVersion 还好理解. 下面我重点说下tagetSdkVersion

下面我举个列子: (以运行时权限为例子)
我有一个Android 6.0 系统的手机. 我写了一个app
```java
<uses-sdk android:minSdkVersion="15"
          android:targetSdkVersion="28"
          android:maxSdkVersion="28" />
```
这个时候毫无疑问我运行app的时候需要适配运行时权限的代码, 那巧了, 我不想写运行时权限的代码怎么办
可以, 你只需要把targetSdkVersion 设置小于23就可以了
为什么要设置小于23就可以了?

* 我们可以参考下链接:
<http://androidxref.com/9.0.0_r3/xref/frameworks/base/core/java/android/content/res/Resources.java#161>

可以看到里面根据了targetsdk的不同而设置了不一样的主题. 其实targetsdk就是为了设置兼容性的.
来确保你的App在新平台上面是否运行一些新特性.

## 窥一斑而知全貌,下面是网上看到的一个总结挺好的:
### targetSdkVersion（24） > 手机的版本（6.0）
对应项目会运行手机版本内一切特性。譬如项目targetSdkVersion是24，那么项目里没做6.0权限管理，调用危险权限相机就会闪退（利大于弊）

### 手机的版本 > targetSdkVersion
项目运行效果为targetSdkVersion 版本。譬如项目targetSdkVersion是22，那么项目里没做6.0（23）权限处理，调用危险权限如相机不会出现闪退和提示，照常运行，项目特性运行到22版本。

### 手机的版本 = targetSdkVersion
如果目标设备的API版本正好等于此数值， 他会告诉Android平台：此程序在此版本已经经过充分测，没有问题。不必为此程序开启兼容性检查判断的工作了。 也就是说，如果targetSdkVersion与目标设备的API版本相同时，运行效率可能会高一些。 但是，这个设置仅仅是一个声明、一个通知，不会有太实质的作用
用较低的 minSdkVersion 来覆盖最大的人群，用最新的 SDK 设置 targetSdkVersion和 compile 来获得最好的外观和行为。强烈推荐targetSdkVersion升到最新版本。

参考: <https://www.jianshu.com/p/26eff76f7883>
