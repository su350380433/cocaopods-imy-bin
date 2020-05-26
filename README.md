# iOS如何稳定提高10倍编译速度

## 一、概述


经过多年的发展，美柚iOS项目代码已经达到40W行+的规模，所使用的 Pod 库的数量达到了**110+**，App Store 安装包210M+，在这么大的项目规模下（CI机器 MAC配置：3 GHz 8-Core Intel Xeon E5；时间：发布20min+），（开发机器iMac ：Retina 5K, 27-inch, 2017 融合硬盘；时间：build30min+）打包、编译问题逐步成为我们团队一个躲不过的痛，严重影响了我们的研发效率与其他团队之间的协作。

我们一台13年的ci机器同时需要承接七八个项目、多个分支的打包任务，在有多个项目同时打包的情况，显得尤其地力不从心。

在硬件资源有限的情况下，并且在`无侵入`、`无影响现有的业务`的前提下，如何解决这些摆在团队面前的难题，便成了我们迫在眉睫的迫切需求，最近半年多来一直在寻找加快打包速度的方案。



## 二、编译提速探索与尝试

***

### 1、CCache

CCache 是一个编译缓存器，一个能够把编译的中间产物缓存起来的工具

其原理是通过把项目的源文件用`ccache`编译器编译，然后缓存编译生成的信息，从而在下一次编译时，利用这个缓存加快编译的速度，目前支持的语言有：`C`、`C++`、`Objective-C`、`Objective-C++`

下面这张图基本就阐述了CCache的工作原理。

![ccache工作原理](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/ccache工作原理.jpg)

在项目中的实际编译流程![Ccache实际编译流程](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/Ccache实际编译流程.jpg)

Ccache我们经过在工程的一番尝试、确实在某些方面上极大的提升了我们出包的速度。美柚iOS Ci打包从之前的最快`20min+`出包到最快`10min`，确实能够给我们带来比较不错的提升，大大加快了我们项目的出包速度。在我们项目运行了几个月后，对于我们项目的情况，也发现了一些问题，现在总结了以下几点：

优点：

1. 满足我们追求的`无侵入`、`无影响现有的业务`的要求，无入侵、且开发人员无感知。
2. 确实能大幅度地提升编译速度，最快时提高一倍以上的编译时间。
3. 不需要对项目作出大调整，只需部署相关环境和一些脚本支持。
4. 不需要改变开发工具链。
5. 同一个目录下，CCache 的缓存命中率相对稳定。

对我们项目中有存在些问题点：

1. 在未有缓存的情况下，首次打包编译的时间比原来的翻近一倍，原来20+min，首次将近40+min，在资源紧张的情况下，甚至是70min+。
2. 修改一些引用较多的文件（如公共库、底层库改动），容易造成大范围的缓存失效，速度会变得比原来未使用ccache时更慢。
3. 多个项目相同的组件不支持缓存共享，我们有多个分支打包的需求，修改目录名称后，缓存即失效。
4. 我们机器的Ccache最大的缓存上限约18GB，且Debug/Release区别缓存，美柚iOS项目占用5GB+的缓存，多个项目、多个分支很容易超出上限，一台机器不能同时支持多个项目。
5. 对机器硬盘读写要求高，如不是全部固态硬盘，速度影响大。
6. CCache 不支持 Clang Modules，系统框架例如 AVFoundation、CoreLocation等， Xcode 不会再帮你自动引入，会导致编译失败。
7. CCache 不支持 PCH 文件
8. CCache 目前不支持 Swift

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526102046.png)

<center>CCache与原生编译时间曲线图</center>

### 2、静态库二进制方案的探索

***

虽然我们已经在Ci的出包速度已经有提升近一倍的速度了，但是存在的问题也比较明显。

在去年的某次技术周会上，我们的大佬提出了使用二进制编译的自研任务，可以更进一步提高研发效率。得到了大佬的启发后，就一直在实践与探索二进制之路上。

我们的项目使用 CocoaPods 来管理第三方库和私有库的依赖，对大部分项目来说应该是标配了。目前还是纯 Objective-C 的项目，有少量C++，暂没有引入 Swift。

### 3、 调研过的二进制组件方案

下面列出研究过的一些主流方案以及最后没有采用的原因，这些方案有各自的局限性，但是也给了我不少启发，思考过程跟最终方案一样有价值。

##### 3.1、Carthage

Carthage可以将一部分不常变的库打包成framework，再引如到主工程，这样可以减少开发过程中的编译时间。Carthage 可以比较方便地调试源码。因为我们目前已经大规模使用 CocoaPods，转用 Carthage 来做包管理需要做大量的转换工作，变动太大，不满足我们的`无侵入`、`无影响现有的业务`，所以不考虑这个方案了。

##### 3.2、[cocoapods-packager](https://links.jianshu.com/go?to=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fgithub.com%2FCocoaPods%2Fcocoapods-packager)

cocoapods-packager 可以将任意的 pod 打包成 Static Library，省去重复编译的时间，一定程度上可以加快编译时间，但是也有自身的缺点：

- 优化不彻底，只能优化第三方和私有 Pod 的编译速度，对于其他改动频繁的业务代码无能为力

- 私有库和第三方库的后续更新很麻烦，当有源码修改后，需要重新打包上传到内部的 Git 仓库

- 过多的二进制文件会拖慢 Git 的操作速度（目前还没部署 Git 的 [LFS](https://links.jianshu.com/go?to=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fgit-lfs.github.com%2F)）

- 难以调试源码,不共享编译缓存

- 打包成 Static Library 过程缓慢，需要通过pod lint，各个组件间又层层嵌套依赖，在现有阶段来说，是难以实现的。

  

##### 3.3、[cocoapods-binary](https://github.com/leavez/cocoapods-binary)

 Cocoapods-Binary（Cocoapods 官方推荐的二进制插件）， 是一个**即时生成二进制包并缓存**，而非像 CocoaPods-Packager 仅仅针对单个私有库的。原理是通过 CocoaPods 提供的 pre_install hook 在 pod install 的 prepare 阶段拦截到当前的 pod install context，进而 fork 出一份独立的 installer 以完成将预编译源码 clone 至 Pod/_Prebuild 目录下，同时也存在几个不足之处：

- 单私有源，无法实现服务端缓存，在没有对应二进制包版本时，pod install 后会额外去做二进制包的生成，一定程度上会影响 pod install的速度。

- 开发者切回源码调试，二进制缓存会一并清空，需求重新编译。

- 多个项目、不同分支的相同组件依旧无法共享

- 只支持framework，对我们项目现状需要比较大的头文件引用方式改动。


#####  3.4、**[cocoapods-bin](https://github.com/tripleCC/cocoapods-bin)** 双私有源

该插件进行二进制化的策略是采用双私有源，即2个源地址，一个静态服务器保存预先打好包的framework，一个是我们现在保存源码的服务地址，在install的时候去选择使用下载那个，是个很不错的项目，深受启发。

优点：

- 源码和二进制文件之间可以来回切换，速度比较快
- 不影响未接入二进制化方案的业务团队
- 无二进制版本时，自动采用源码版本
- 接近原生 CocoaPods 的使用体验 

对于在我们项目中存在的不足之处：

- 不支持指定分支，:podspec =>'', :git 方式的引用，对需要支持多个分支、多个业务线的项目是致命的。
- Archive二进制文件时，只能去spec仓库下载源码，无法根据指定的分支去下载依赖库，导致编译失败、错乱的问题
- 依赖的组件需要推送到spec仓库，很多私有库并没有推送到仓库，且对于频繁改动的私有库，推送到仓库的verify很慢且与我们的开发习惯不符。
- 不支持.a静态文件输出，项目中大量类似 `#import "IMYPulic.h"`需要一个个库去编译替换为`#import `，想想那110多个组件库~
- 只支持一套环境，对于有Debug/Release/Dev开发环境需求的无法满足
- 不支持二进制组件的源码调试
- 不能流畅的支持频繁变动的业务组件，操作会异常繁琐。
- 针对于我们的项目，目前存在较大的障碍，无法使用起来。

### **4、 思考与总结**

经过一个多月来对业界存在的轮子的分析和思考，并在一定的实践后，最后我们决定自己造一个灵活的、可配置的、简便的、无入侵的、双私有源二进制组件辅助插件。

接下来就撸起袖子，努力干吧~，骚年😀😀😀

## 三、双私有源二进制组件简介

***

在受到cocoapod-bin启发后，在借鉴它的部分框架下，我们实现了自己的二进制辅助插件cocoapods-imy-bin，并新增了几个命令和二进制源码调试能力。

### 1、能做什么？

在cocoapods-imy-bin的辅助下，能无侵入式自动化地制作所有符合条件的组件为二进制，且对于频繁的业务组件也能轻松的应用上二进制组件，无需多余操作，一切交给cocoapods-imy-bin自动化运行。

同时对于研发人员，也能提供独立的二进制组件给研发人员使用，解决日常的编译 效率、跑真机效率低下，被墙等各种问题。

我们的口号是：**只要能编译通过，就制作**。

即使独立的组件库编译不通过，整体项目能编译通过也制作。

**一次编译到处使用**，**无入侵式**。

整套环境下来，没有让我们的开发人员改变原来的开发习惯，没有改动业务中相关的代码，基本上做到了使用人员无感知状态。

### 2、Ci打包效果

#### 2.1 单项目 - 编译最快2分钟一次

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526112303.png)

<center>arm64和armv7架构</center>

上图是个由我们打了几千个包的经验得出对单个项目编译时间大致的曲线图。这里假设一台机器只一次只有一次job。Y轴编译时间，X轴某次的编译， 红色线条表示的是原生（未使用Ccache和二进制组件），黄色线表示使用了Ccache，蓝色表示使用了二进制组件。

由图可以看出来在无任何辅助下原生的编译时间曲线（红色）是趋于平缓，在20min上下左右。Ccache和二进制第一次在无任何缓存的情况下，在一定程度上是会比原生的耗时，Ccache主要耗时在边编译边缓存项目的编译产物。二进制主要耗时在编译完成后，对.a编译产物的组装和push到私有源仓库的时间上。

在ccache完全命中、二进制文件完全都存在的情况下，ccache比原生的提高一倍以上， 二进制会比ccache编译时间再提高一倍大概在2分钟。二进制在之后的表现更趋于平稳，而ccache在修改了某个被引用较多的文件时、如底层的公共文件后，命中率就会大大地降低，有时会比不用ccache更耗时，如#4位置。在ci有多个job同时并发在跑的情况下，由于ccache 需要对IO频繁地读写操作，耗时表现可能会更糟糕些，我们经常遇到过等了七十几分钟才出包的情况。

二进制的编译时间相对平稳很多(蓝色曲线)，在我们架构强有力的支撑下，划分出110多个独立组件，每次的打包基本上是就耗在某个组件的编译+archive。如果是某些变更比较频繁的组件，我们还可以考虑对颗粒较大组件配上ccache，做`双层编译缓存`。双层编译缓存原理是Pods组件库无二进制组件采用源码编译时，源码编译同时应用ccache缓存支持，加速源码组件的编译。

同时组件库可以配合Gitlab-Ci的runner的应用，每次已提交代码就触发独立组件的制作二进制，让每次的编译速度都达到最快，蓝色二进制曲线将会更接近直线。Gitlab-Ci具体的使用教程参见后文。

如果存在有独立组件无法编译问题和版本依赖问题，也可以再跑个定时Job，或者其他轮询条件Job，及时提供最新二进制组件。

#### 2.2、多项目情况

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526152757.png)

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526152413.png)

一台机器上多个项目的ccache显得是比较吃力的，且不稳定，超出ccache的缓存最大值就会被清掉。

使用了二进制后，即使是多个项目编译时间都是趋于比较平稳的。这里面的原理估计大家都能想得到为什么。

### 3、开发使用效果 - 10倍以上的提升

在Podfile引入插件后，在pod install/update后，符合条件的情况下，会自动转换为二进制组件。

在我们的开发机器（iMac ：Retina 5K, 27-inch, 2017 融合硬盘；）上，全量代码之前Build需要**30min+**，现在使用全部使用二进制后，最快只需要**2min+**就可以，提高的效率达到10倍以上。

#### 3.1.源码编译

Ps：110+个Pods库中，有20+个稳定Pods库已经被制作为二进制库，并非全部源码编译。

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526112946.png)

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526135031.png)

#### 3.2. 二进制编译

Ps：2个Pods和5个Action Extension使用源码依赖，其他全部是二进制Pods。

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526113633.png)

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526135053.png)

在二进制Build 127秒中，约45秒消耗在copy pods Resource。

全量编译中，13496个Tasks/727个Tasks，1710秒(28.5分钟)/127秒(2分钟)，提升的速度远远超过10倍。

#### 3.3 演示

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526134949.png)

<center>Pods自动切换为二进制组件</center>

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/gifhome_640x360_12s%20(1).gif)

在环境搭建完后，开发人员在Podfile中，加入以下两句，就能享用到自动切换为二进制组件，体验极速编译。

``` ruby
plugin 'cocoapods-imy-bin' 
use_binaries!
```

更具体情况[视频演示](https://github.com/su350380433/iOS_build_share_article/tree/master/video)

### 4、功能点

目前`cocoapods-imy-bin`插件支持的功能如下

1. 无侵入、无影响现有的业务。
2. 不影响未接入二进制化方案的业务团队，提供配置文件。
3. 只要项目能编译通过就制作，即使独立组件编译失败。
4. 支持无二进制版本时，自动采用源码版本。
5. 支持只需项目能编译通过就能制作二进制组件，无需再关心pod lint等。
6. 支持`pod bin local` 命令一键自动化制作、上传、存储项目本地已经存在的二进制组件，可配合ci打包的编译产物使用。
7. 支持指定依赖分支、支持:podspec =>'', :git 方式的引用
8. 支持同时 .a、Framework 静态库产出
9. 支持archive时，根据Podfile自动获取podsepc依赖的库，无需强制去spec仓库拉取。
10. 支持多套隔离环境，如Debug/Release/Dev配置，方便为Debug/Release/Dev各种环境提供专用二进制组件。
11. 支持输出.a二进制组件制作binary.podsepc无需模板。
12. 支持稳定的二进制组件，在上传二进制组件的binary.podsepc跳过pod lint验证，加快速度。
13. 支持`pod bin auto` 命令一键自动化制作、上传、存储单个二进制组件
14. 支持`pod bin auto --all-make` 命令一键自动化制作、上传、存储该项目下`所有组件`的二进制组件
15. 支持 是否使用二进制文件、是否制作二进制文件和二进制/源码调试功能的白名单设置
16. 支持`pod install/update `多线程模式，加快pod过程，**Pod速度提升80%+**。
17. 支持`pod bin install/update` 命令，实现无入侵修改Podfile内容，避免直接修改工程的Podfile文件而导致提交冲突、误提交。
18. 支持`pod bin code`命令，实现二进制库不切换源码库、程序无需重新运行的调试能力

## 四、双私有源二进制组件整体设计方案

***

### 1、制作流程 - 二进制组件

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200507151414.png)

### 2、使用流程 - 二进制组件

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200507151507.png)

### 3、分析

如上图所示

**`server`**是一个自建的服务，专门存在二进制文件

**`二进制repo`**是一个专门存储二进制podspec的私有源仓库

两者的设计很好的将二进制组件产物分别的存储起来，同时我们借助cocoapods-imy-bin插件在pod install/update的时机去自动切换源码/二进制组件，巧妙的避开入侵原有组件的问题。即使我们把**`server`**关闭或者**`二进制repo`**删除，清了缓存后，原有的打包流程和开发流程还是能照旧执行，并不受任何影响。

即插即用，无需关心服务是否存在、二进制组件是否已经制作完成了，方便灵活。

同时github/组件源码/源码repo属于`源码源`，与**`二进制repo`**组成`双私有源`

## 五、制作二进制组件原理

***

只要项目能编译通过就可以制作，不再需要关注组件是否能通过pod lint，不再需要关注组件是否有push到repo，不再需要耗在的pod lint的等待时间上，完全无门槛。

### 1、为什么能这么灵活，怎么做到的

大部分的轮子都要求组件能pod lint通过，但是绝大部分的组件并没有这么规范化，梦想总是美好的，现实总是残酷的。

为了能让绝大部分组件使用，我们绕过了pod lint，优化了部分cocoapods流程，直接去取build后的.a编译产物，结合对应的podspec文件，去组装Headers、Resource。很多轮子都只制作当前的组件库，对于依赖的并不提供支持，在最新代码的前提下，利用所有的.a编译产物制作对应二进制组件可以极大的提高效率。

同时制作二进制组件时podspec中dependency依赖组件，不再强制要求去repo拉取，而是优先从podfile拿取依赖值。

### 2、静态文件.a的制作

制作静态文件分为两种，一种是cocoapods-imy-bin自己制作，一种是利用已经编译完成的编译产物来组装制作二进制组件。

#### 2.1、通过插件自身的xcodebuild 制作

可以直接使用插件的 `pod bin auto`命令，在插件初始化配置完成后，目录下只要有包含podspec文件，会自动化执行build、组装二进制组件、制作二进制podspec、上传二进制文件、上传二进制podspec到私有源仓库。

``` shell
pod bin auto --all-make
```

带上—all-make参数会把当前组件所依赖的组件都自动化制作成二进制组件。

带上```—all-make```参数会把当前组件所依赖的组件都自动化制作成二进制组件。

``` shell
//构建真机静态库文件
xcodebuild GCC_PREPROCESSOR_DEFINITIONS='$(inherited)' ARCHS='arm64 armv7' OTHER_CFLAGS='-fembed-bitcode -Qunused-arguments' CONFIGURATION_BUILD_DIR=build clean build -configuration Release -target '目标工程Target' -project ./'目标工程' 2>&1
```

``` shell
//构建模拟器静态库文件
xxcodebuild GCC_PREPROCESSOR_DEFINITIONS='$(inherited)'  -sdk iphoneos CONFIGURATION_BUILD_DIR=build-simulator clean build -configuration Release Release -target '目标工程Target' -project ./'目标工程' 2>&1
```

ci打包只需真机模式，开发需要模拟器模式+真机模式。在构建完对应的架构，用lipo对架构.a/.framework进行合并操作

```shell
lipo -create 'x86.a' 'arm64.a' 'armv7.a' -output '目标静态库'.a  
```

#### 2.2、结合第三方编译产物

通过Jenkins打包，打包过程会生成中间编译.a文件，再通过cocoapods-imy-bin 的`pod bin local`去组装每个二进制组件中的.a/headers/resource,再自动化制作对应的二进制podspec文件，上传对应环境的repo和存储服务。

在我们只有一台iOS打包机，且没有提交代码后还需等几分钟（等GitLab-ci制作完成二进制文件完成）才能打包的习惯，这是一种比较好的选择。

提交代码后等GitLab-ci制作完成二进制文件完成，再打整体app的包，这种方式严重的改变了我们当前的开发习惯，且并在提交代码到出包的过程中没有加快整体流程的速度。

我们同时也用了Gitlab-Ci触发二进制组件的制作、Jenkins定时构建二进制组件。

#### 2.3、ccache+二进制双层编译缓存

在Jenkins打包上，对未制作二进制的组件，同时也应用了ccache。双编译缓存的机制，在一定的情况下可以很大地加速整体速度。

### 3、Headers

Headers .h文件在pod install/update过程中，其实已经就组装整理好了，在Pods下Headers/Public/xxx ，这些头文件，我们只需要拷贝就行了，无需再去根据对应的podspec的source去组装。

### 4、Resource

Resource是通过对应podspec的resource字段去对应的组件库中搜索资源，再组合起来。这里需要特别注意查看下有些中文、带空格路径的文件。

### 6、开发机器，本地命令

开发机器只需安装了cocopods-imy-bin插件后，可以通过本地命令 `pod bin auto` or `pod bin local`自动制作二进制组件，自动上传，无需其他配置。

## 六、二进制PodSpec 

***

#### 1、自动化制作PodSpec 

二进制组件现在是一个新的组件库，我们需要为其配置一个podspec配置索引文件。

cocopods-imy-bin制作二进制podspec不需要模板，会自动去提取源码podspec的version去创建，在修改`source`、`source_files` 、`vendored_libraries`、`public_header_files`这几个字段后，其他的都读取原有字段。对于一个静态组件，其他的修饰字段并不是很重要。

```ruby
#二进制podspec.json
{
  "name": "YYModel",
  "version": "1.0.4.1",
  "source": {
    "http": "http://ci.meiyou.im:10240/frameworks/YYModel/1.0.4.1/zip",
    "type": "zip"
  },
  "source_files": "bin_YYModel_1.0.4.1/Headers/*",
  "public_header_files": "bin_YYModel_1.0.4.1/Headers/*.h",
  "vendored_libraries": "bin_YYModel_1.0.4.1/*.a"
}

```

这里有个细节` bin_YYModel_1.0.4.1`，制作完二进制组件后，会把目录改成对应的组件库+版本号，用于识别对应的版本。

## 七、壳工程分离

***

壳工程顾名思义就是将原来的project中的代码全部拆出去，得到一个空壳，仅仅保留一些工程配置选项和依赖库管理文件。

因为自动集成涉及版本号自增，需要机器修改工程配置类文件。如果在创建二进制的过程中有新业务PR合入，会造成commit树分叉大概率产生冲突导致集成失败。抽出壳工程之后，我们的壳只关心配置选项修改（很少），与依赖版本号的变化。业务代码的正常PR流程转移到了各自的业务组件git中，以此来杜绝人工与机器的冲突。

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200526145959.png)

<center>新旧工程示意图</center>

### ![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200507151034.jpg)

<center>壳工程分离</center>

**壳工程分离的意义主要有如下几点：**

- 为自动集成铺路，避免业务PR与机器冲突。
- 提升效率，后续Pods往Pods移动代码比proj往Pods移动代码更快。
- 制作二进制组件、加快编译速度、提高研发效率

## 八、多套完全隔离环境

***

#### 1、什么是多套完全隔离环境

目前我们提供了三套二进制组件的环境，Dev、Debug_iPhoneos和Release_iPhoneos，分别对应开发人员使用的环境dev、Ci打包使用的Debug、Release，三套环境完全隔离，分别对应不同的私有源仓库、二进制文件存储服务，且互相不干扰。默认是Dev环境。

1. Dev 下是 Deubg 设置编译的 x86_64 armv7 arm64。	
   1. 二进制文件服务器：http://xxx:10240/frameworks/
   2. 二进制私有源仓库：https://xxx/binary_spec_dev
2. Debug_iPhoneos下是 Deubg 设置编译的  armv7 arm64。
   1. 二进制文件服务器：http://xxx:9192/frameworks/
   2. 二进制私有源仓库：https://xxx/binary_spec_debug_iPhoneos
3. Release_iPhoneos下是 Release 设置编译的  armv7 arm64。
   1. 二进制文件服务器：http://ci.meiyou.im:20480/frameworks/
   2. 二进制私有源仓库：https://xxx/binary_spec_release_iPhoneos

#### 2、为什么要多套

我们日常ci打包也分Debug和Release。

Debug通常称为调试版本，通过一系列编译选项的配合，编译的结果通常包含调试信息，而且不做任何优化，以为开发人员提供强大的应用程序调试能力。

而Release通常称为发布版本，是为用户使用的，一般客户不允许在发布版本上进行调试。所以不保存调试信息，同时，它往往进行了各种优化，以期达到代码最小和速度最优。为用户的使用提供便利。

Dev是专门提供给研发人员使用，x86_64是针对x86架构的64位处理器，iPhone5s及以上，致力于提供开发人员build的效率。

## 九、配置与开发使用

***

### 1、使用二进制组件配置

#### 1.1、本地配置文件 - Podfile_local

本地组件配置文件 Podfile_local，目前已支持Podfile下的大部分功能，可以把一些本地配置的语句放到Podfile_local。

场景: 

1. 不希望把本地采用的源码/二进制配置、本地库传到远程仓库。
2. 避免直接修改Podfile文件，引起更新代码时冲突、或者误提交。
3. Pod开启多线程，加快Pod速度

##### 用法：

在与Podfile同级目录下，新增一个`Podfile_local`文件


```ruby
#target 'Seeyou' do 不同的项目注意修改下Seeyou的值
#:path => '../IMYYQHome',根据实际情况自行修改，与之前在podfile写法一致

#是否启用二进制插件，想开启把下面两行注释去掉
# plugin 'cocoapods-imy-bin'
# use_binaries! 

#需要替换Podfile里面的组件才写到这里
#在这里面的所写的组件库依赖，默认切换为【源码】依赖
target 'Seeyou' do
  #本地库引用
	#pod 'IMYYQHome', :path => '../IMYYQHome'

  #覆盖、自定义组件
  #pod 'IMYVendor', :podspec => 'http://覆盖、自定义/'
end
```

```ruby
以前的 pod update --no-repo-update 命令加个前缀 `bin` 变成
```

```shell
pod bin update --no-repo-update 
```

​		or

```shell
pod bin install
```

支持 pod install/update 命令参数

并将其加入 .gitignore ，再也不用担心误提交或者冲突了，Podfile_local 中的配置选项优先级比 Podfile 高，支持和 Podfile 相同的配置语句，同时支持pre_install or post_install。

如果想使用二进制组件，加快编译效果的话

```ruby
#取消注释这两句代码，具体的命令解释查看后文
plugin 'cocoapods-imy-bin'
use_binaries!
```

如果您不习惯Podfile_local的使用方式，可以把命令写在Podfile里面，pod时不需要加bin，依旧是 pod update/install。

#### 2、制作二进制配置文件-BinArchive.json

有些组件在不需要被制作成二进制组件，可以通过在Podspec同级目录下，添加**BinArchive.json**文件。

```ruby
{
  #制作白名单
    "archive-white-pod-list" : [
        "Seeyou",
        "IMYFoundation",
        "IMYPublic"
    ],
  #忽略源码存储在git上的组件被制作为二进制组件
    "ignore-git-list": [
        "git@gitlab.meiyou.com:Github-iOS"
    ],
  #忽略源码存储在http上的组件被制作为二进制组件
    "ignore-http-list": [
        "https://gitlab.meiyou.com/Github-iOS"
    ]
}
```

## 十、GitLab-Ci 配置
***

### 1、是什么？

GitLab CI 是GitLab内置的进行持续集成的工具，只需要在仓库根目录下创建.gitlab-ci.yml 文件，并配置GitLab Runner；每次提交的时候，gitlab将自动识别到.gitlab-ci.yml文件，并且使用Gitlab Runner执行该脚本。可以暂时理解为 Jenkins微服务。

目前我们考虑用Gitlab-Ci来为各个组件独立配置'打包功能'，做二进制组件或者其他一些自动化任务。

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200507142434.png)

### 2、能带来给我们什么

1. 每次提交代码可以独立检查OC-Lint 或者文件监控等功能
2. 及时为每个独立组件提供最新的二进制文件
3. 加快且稳定每次Ci打包速度
4. 独立提供自动化测试和监控功能。

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/further1.png)

<center>可实现的出包速度曲线</center>

### 5、对项目的思考

我们的底层库是相对稳定且规范，目前存在部分上层组件，由于所依赖的底层组件分支变动，导致制作出来的二进制组件不一定是全部最新版本的，所以是到目前为止还没有强力推广的最大因素。

如果能维护好各自组件下的版本依赖是最好的，或者有更好的解决方案？

目前的实现方案是：

- 目前在CI_3的CI机器上部署了定时构建任务，dev开发和Debug_iPhoneos环境在工作日9点到19点区间，分别每1小时、每半小时构建一次，会生成需要被制作的二进制组件，且不生成ipa/dsym/archive等编译产物。
- 如果Gitlab-ci 部署完成后，可以编写脚本在提交代码时来触发这两个相关Jenkins Job执行构建任务，以此来避免有些业务组件未能独立编译通过和所依赖的代码不是最新的问题。


## 十一、二进制不切换源码库、程序无需重新运行的调试能力

***

在看了[美团 iOS 工程 zsource 命令背后的那些事儿](https://tech.meituan.com/2019/08/08/the-things-behind-the-ios-project-zsource-command.html)文章后，身受启发，但是美团没有开源相关的项目，就自己琢磨着怎么实现跟他们一样的功能。经过一番倒腾，终于也有了我们自己的zsource。

### 1、效果展示

使用二进制，虽然会给工程带来构建速度的提升，但是会带来一个新的问题：在调试工程时，那些使用二进制的组件，无法像源码调试那样看到足够丰富的调试信息。例如，如果程序在二进制组件的代码中崩溃，我们只能看到该组件的堆栈信息和一些不明所以的汇编代码：

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/20200508140906.png)

<center>程序断点在二进制组件的代码中时的样子</center>

和业界大多的组件化方案类似，美柚App 的组件化方案也提供了将一个组件从二进制切换到源码的机制。美柚工程的开发者能够使用一系列配置和命令来切换组件的源码和二进制状态，但每次切换都需要重新执行 `pod install`。这种方式在组件化的初期是没有什么问题的。但随着美柚 App 的组件数量不断增长，即便是只切换一个组件的状态，单次 `pod install` 的时间也增长到了分钟级。而且这种方式每切换一次就必须重新编译运行一次 App，在追查一些偶现崩溃问题时，开发体验非常不友好，也不利于崩溃问题的快速定位分析。

为了解决以上提到的这些问题，我们利用 CocoaPods 的插件机制，为 CocoaPods 的 `pod` 命令增加了 `bin code` 子命令，开发者可以在使用二进制构建工程的同时，非常快速地将一个组件调出源码进行调试，具体的使用效果可以看一下如下的屏幕录制：

![](https://raw.githubusercontent.com/su350380433/PicGo_slj/master/img/gifhome_640x494_9s%20(1).gif)

### 2、如何使用

在功能根目录下，输入命令:

``` shell
pod bin code IMYFoundation
```

`IMYFoundation`为需要源码调试的组件库名称。成功之后像平时一样单步调试，控制台打印变量。让我们同时拥有使用二进制的便利和源码调试的能力。

## 十二、遇到的问题

***

### 1、跨组件宏定义

预编译阶段处理的宏定义，在组件进行二进制化后会失效，特别是某些依赖 DEBUG 宏的调试工具，在二进制化之后就不可见了。

个人建议尽量不要跨模块使用宏定义，特别是可以用常量或函数代替的宏。比如有组件 A、B ，B 依赖 A，它们包含如下代码：

```objective-c
// A
#define TDF_THEME_BACKGROUNDCOLOR [[UIColor whiteColor] colorWithAlphaComponent:0.7]

// B
// .m 使用了 TDF_THEME_BACKGROUNDCOLOR
```

假设 A 和 B 都已二进制化，假设后续我们修改了 A ：

```objective-c
// A
#define TDF_THEME_BACKGROUNDCOLOR [[UIColor whiteColor] colorWithAlphaComponent:0.4]
```

由于 B 中的 TDF_THEME_BACKGROUNDCOLOR 宏已经在二进制化打包预编译时被替换为 `[[UIColor whiteColor] colorWithAlphaComponent:0.7]` ，所以 B 并不会感知到此次 A 的变更，这时我们就不得不重新打包组件 B 以同步 A 的变更，即使 B 并未做任何更改，当存在较多使用 TDF_THEME_BACKGROUNDCOLOR 宏的组件时，就容易遗漏同步某些组件。 

## 十三、未来计划

***

- 使cocoapods-imy-bin成为一个工具平台
- 服务内集成代码检查，资源检查等功能
- 服务内集成初步自动化测试、性能监控等功能
- 完善优化GitLab-Ci 各个组件脚本，最大限度降低编译速度曲线幅度
- 实现资源共享，抹去copy pods Resource近34%的耗时，进一步降低编译时间到一分钟内。

## 十四、总结与感谢

***

在自学ruby后写的cocoapods-imy-bin插件，在经过刻骨铭心的努力下终于完成第一阶段的设想，其中涉及到的细节实现非常之多，还有一些设想待去实现与完善。总体来说对研发效率提升是非常可观的。

感谢大家的阅读以及对美柚技术的持续关注，同时也感谢各位大佬的信任与支持，能让这套系统顺利地落地实现。

## 十五、参考文献

***

[用ccache让Xcode运行、打包飞起来](https://www.jianshu.com/p/b61f182f75d2)

[基于 CocoaPods 的组件二进制化实践](https://triplecc.github.io/2019/01/21/%E5%9F%BA%E4%BA%8ECocoaPods%E7%9A%84%E7%BB%84%E4%BB%B6%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%8C%96%E5%AE%9E%E8%B7%B5/)

[美团 iOS 工程 zsource 命令背后的那些事儿](https://tech.meituan.com/2019/08/08/the-things-behind-the-ios-project-zsource-command.html)



