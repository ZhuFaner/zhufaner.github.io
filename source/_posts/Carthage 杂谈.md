---
title: Carthage 杂谈
date: 2018-11-20 18:47:30
tags: 
- iOS 
- framework 
- carthage
---

我将从 `Xcode` 工程项目的结构出发，一步步解释 `CocoaPods` 的工作原理，最后教大家怎样让自己的项目支持 `Carthage`。


## Xcode 工程目录探究
首先应当明确：
> Xcode 工程其实是一个巨大的配置文件

相信大家都有这样的经历，就是在多人协同开发的时候，总是会在 `xcodeproj` 文件当中出现莫名其妙的冲突导致项目打不开，这就是因为可能有人无意中修改了项目配置导致的。
![](/assets/conflict.png)
新建一个项目，可以看到目录下有一个 `xcodeproj` 文件，其他都是代码，图片，xib等资源文件。

![](/assets/project.png)
如果细心观察可以发现，`xcodeproj` 其实不是一个文件，而是一个目录，右键点击“显示包内容”，发现里面还有这些内容：

![](/assets/project_inner.png)

* `xcuserdata`：通常来讲这个文件是可以写到 `gitignore` 里的，因为每个用户都会有这个独立的文件来存放用户状态、目录折叠状态、最后打开的文件等等。如果是第一次打开项目，`xcuserdata`是不会存在的，`Xcode` 会自己创建。

* `project.xcworkspace`： 一般情况下用不到，有意思的是，如果创建名为 x.xcworkspace 的工作区，然后创建名为 y.xcodeproj 的项目，同时将项目添加到x.xcworkspace，然后会发现在 y.xcodeproj 目录下并没有创建 project.xcworkspace。

* `project.pbxproj`：这就是整个 `Xcode` 项目的核心了，几乎所有能在 `Xcode` 里修改的东西都包含在内，包括文件路径、文件引用关系、编译配置，等等。而 `Xcode` 所做的，就是给这些配置项赋予了图形界面。

![看这个图或许更容易理解](/assets/tree.png)

## 几个容易混淆的概念
###  
`project`、`target`、`scheme`、`workspace` 这几个概念是很多开发者都容易搞混的，包括我在很长一段时间里都是一头雾水，下面来简单说一说我的理解。

* #### Project 
`project`是一个 `Xcode` 工程的基本单位，可以独立存在，也可以包含在一个 `workspace`下。

* #### Target
顾名思义，`target` 就是一个项目工程的“目标”，目标可以有一个，也可以有多个，可以认为一个 `target`对应一个新的 `product`。
以喵神的大作 `KingFisher` 为例，就是用同一套代码生成了针对不同设备的 `framework`：

![](/assets/kingfisher.png)

* #### Scheme
`scheme` 定义了在编译`project`时的一些配置，可以保存在 `project` 中，也可以保存在 `workspace`中，如果保存在 `project` 中，那么任意包含了这个 `project` 的 `workspace` 都可以使用。如果保存在 `workspace` 中，那么只有这个 `workspace` 可以使用，选择了 `scheme` 就意味着选择了运行目标。
* #### Workspace
`workspace` 可以理解为为了让若干个 `project` 实现依赖关系的 Xcode 结构，也是 `CocoaPods` 的得以实现的前提。打开一个 `workspace` 文件的源码，可以看出这只是对其中若干个 `project` 的关系描述：

![](/assets/workspace.png)

总结一下，这几个概念的关系可以理解为 `workspace` -> `project` -> `target`，而 `scheme`可以同时配置 `workspace` 和 `project`，这样是不是就清楚很多啦~~

## CocoaPods做了什么？
> 实质是用脚本操作了Xcode项目

当一个新项目在终端里运行"pod install"的一瞬间，可以看一看发生了什么：
### 1、修改xcode工程文件
创建了一个名为 pod 的 `project`和一个`workspace`，并把原有的项目和 pod 纳入了 `workspace`的管理范畴。

![](/assets/pod_install.png)

### 2、将pod生成的framework添加到主工程的依赖中；

![](/assets/add_dependency.png)

### 3、添加run script校验脚本；
这个脚本的作用在于校验 `Podfile.lock` 与 `Manifest.lock` 的一致性，我们都曾经遇见过这样的错误：`沙盒文件与 Podfile.lock 文件不同步 (The sandbox is not in sync with the Podfile.lock)`，这是因为 `Manifest.lock` 文件和 `Podfile.lock` 文件不一致所引起。

由于 `Pods` 所在的目录并不总在版本控制之下，这样可以保证开发者运行 app 之前都能更新他们的 pods，否则 app 可能会 crash，或者在一些不太明显的地方编译失败。

![](/assets/script.png)

综上所述，`cocoapods` 所做的就是把所有的依赖库都用一个 `Pods_CarthageDemo.framework` 的壳子包裹起来，直接让主工程依赖于这个库，把这个“壳子”库作为中间层，达到方便管理的效果。

有的同学在使用 `CocoaPods` 以后转向使用 `Carthage`时，不知道如何移除pod对项目的侵入，其实解决方法很简单，就只要按照上面的三个步骤挨个移除相关内容即可。

### 小结
`CocoaPods` 的方案意味着主工程在每次编译时，都会把所有的依赖库深度遍历，从依赖关系树的最底层向上依次编译出 `framework`，直到所有的依赖库编译完成以后，再执行主工程的编译工作。这样对计算机的性能压力无疑是巨大的，更别说 `Xcode` 自己还不争气。。


## 让你的项目支持Carthage
对于 `Carthage` 如何使用现在已经有非常多的教程了，本文就不再赘述。

`Carthage` 是广大iOS开发者除了 `CocoaPods` 以外的另一款管理依赖的利器，对项目本身做到了零侵入，虽然使用上比 `Pod` 要繁琐一些，但是跟项目的耦合度大大降低。更重要的是，`Carthage` 不用像 `Pod` 一样每次 `Rebuild` 都把所有的依赖库都编译一遍，而是直接使用打包出来的 `Framework`，对于机器性能较弱的同学可以说是大大的福音了。

得益于去中心化的思想，`Carthage`并不像 `Pod`一样，没有总项目的列表，这能够减少维护工作并且避免任何中心化带来的问题（如中央服务器宕机、Github 对天朝的不友好）。不过，这样也有一些缺点，就是项目的发现将更困难，用户将依赖于 Github 的趋势页面或者类似的代码库来寻找项目。

现在市面上绝大多数的iOS开源项目都支持了 `Cocoapods` ，对 `Carthage` 的支持还比较少，但是让项目支持 `Carthage` 其实非常简单，比 `CocoaPods` 简单多了，下面新建一个空项目为例：
### 1.创建target
创建一个framework类型的target：

![](/assets/carthage_create.png)

### 2.把源代码添加到target的编译目录下

![](/assets/create_file.png)

### 3.把scheme中shared勾选上

![](/assets/scheme_shared.png)

这个选项之前一直没明白意义何在，后来查阅了相关资料如下：
> This is normally done if you want to build your project elsewhere and you don't want to rely on auto-generated schemes or you might have modified the auto-generated schemes (e.g. when building on a CI server).

大意就是如果想要在其他地方构建项目且不想依赖自动生成的方案，就要执行这个操作，我猜想`Carthage` 是利用 `xcodebuild`工具来执行打包操作，所以要求这个选项必须被探测到。

至此，准备工作已经完成，运行`Carthage build --no-skip-current`，完成后会开到在目录下会多出一个 Carthage 文件夹，打包出来的 Framework 就在里面。

![](/assets/framework_done.png)

## 结语
不管是 `Carthage` 还是 `Cocoapods`，都是为了让开发的效率更高，减少工作量。我们平时使用大多时候都是知其然不知其所以然，但是如果能够更加深入的了解其内部的机制，相信可以使自己的开发工作如虎添翼。

## 参考
[https://developer.apple.com/library/archive/featuredarticles/XcodeConcepts/Concept-Targets.html](https://developer.apple.com/library/archive/featuredarticles/XcodeConcepts/Concept-Targets.html)
[https://stackoverflow.com/questions/13952491/is-it-safe-to-ignore-xcuserdata-with-git-if-using-launch-arguments](https://stackoverflow.com/questions/13952491/is-it-safe-to-ignore-xcuserdata-with-git-if-using-launch-arguments)
[https://stackoverflow.com/questions/35054788/carthage-no-shared-framework-schemes-for-ios-platform-for-my-own-framework](https://stackoverflow.com/questions/35054788/carthage-no-shared-framework-schemes-for-ios-platform-for-my-own-framework)
[《Xcode中的 workspace, project, target, scheme》](https://www.jianshu.com/p/1308a199f168)
[《Carthage 使用 / 如何给自己的项目添加 Carthage 支持》](http://blog.csdn.net/andanlan/article/details/78113468)
[《Xcode - Xcodeproject详解》](https://www.cnblogs.com/gongyuhonglou/p/5570864.html)
[《深入理解 CocoaPods》](https://objccn.io/issue-6-4/)

> ***怕什么真理无穷 进一寸有一寸的欢喜。 ***
> <div align="right">------胡适</div>