---
title: Swift Package Manager
date: 2020-01-05 16:22:02
tags:
---
## 什么是Swift Package Manager
> The Swift Package Manager is a tool for managing the distribution of Swift code. It’s integrated with the Swift build system to automate the process of downloading, compiling, and linking dependencies.

简单来说，就是官方的 Cocoapods 或者 Carthage。可以简写为 SPM 或者 SwiftPM

## 为什么要使用

1. 一个跨平台的
![1](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/SwiftPM1.png)
2. 核心库之外代码复用
3. 充分利用 swift 的优势
4. Swift开源项目的一部分(swift3.0就开始包含了SwiftPM)
5. 在 Xcode 中集成

## 基本概念

### Modules
在 Swift 中我们使用模块来管理代码，每个模块指定一个命名空间并强制指定模块外哪些部分的代码是可以被访问控制的。

一个程序可以将它所有代码聚合在一个模块中，也可以将它作为依赖关系导入到其他模块。除了少量系统提供的模块，像 OS X 中的 Darwin 或者 Linux 中的 Glibc 等的大多数依赖需要代码被下载或者内置才能被使用。

当你将编写的解决特定问题的代码独立成一个模块时，这段代码可以在其他情况下被重新利用。例如，一个模块提供了发起网络请求的功能，在一个照片分享的 app 或者 一个天气的 app 里它都是可以使用的。使用模块可以让你的代码建立在其他开发者的代码之上，而不是你自己去重复实现相同的功能。

<!-- more -->

### Packages
一个包由 Swift 源文件和一个清单文件组成。这个清单文件称为 Package.swift，定义包名或者它的内容使用 PackageDescription 模块。  
一个包有一个或者多个目标，每个目标指定一个产品并且可能声明一个或者多个依赖。

### Products
一个目标可能构建一个库或者一个可执行文件作为其产品。库是包含可以被其他 Swift 代码导入的模块。可执行文件是一段可以被操作系统运行的程序。

### Targets
targets 是 package 的基本组成部分，它描述了如何将一组源文件构建成模块或测试用例，一个 target 可以依赖于当前 package 中的其它 target，也可以依赖于声明为依赖关系的其他 package 所导出的 product（产品）。

### Dependencies
目标依赖是指包中代码必须添加的模块。依赖由包资源的绝对或者相对 URL 和一些可以被使用的包的版本要求所组成。包管理器的作用是通过自动为工程下载和编译所有依赖的过程中，减少协调的成本。这是一个递归的过程：依赖能有自己的依赖，其中每一个也可以具有依赖，形成了一个依赖相关图。包管理器下载和编译所需要满足整个依赖相关图的一切。

## 使用 SwfitPM
### 基本命令
![2](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/SwfitPM2.png)

- `swift build` 用于编译 SPM
- `swift run` 用于编译并运行一个可执行文件
- `swift test` 用于运行 package 中的单元测试
- `swift package` 在 package 中进行各种除编译/运行/测试之外的操作，如创建、编辑、更新、重置、修改编译选项/路径等

### 创建一个  executable package
使用命令行创建一个 executable package
```
$ mkdir HelloPackge    // 创建 HelloPackge 文件夹
$ cd HelloPackge       // 进入 HelloPackge 文件夹
$ swift package init --type executable  // 在当前文件下初始化 package 为可执行项目
$ swift run          // 编译并执行
Hello, world!
```

执行完后的目录结构:  
```
HelloPackge
├── Sources
├── README.md
│   └── HelloPackge
│       └── main.swift
└── Package.swift
```
在当前目录下执行 `swift package generate-xcodeproj` 命令生成一个 Xcode 工程，可以更方便的通过 Xcode 编译调试, 生成后如下:
![4](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/SwiftPM4.png)

### 创建一个 library package
1. 使用命令创建
```
$ mkdir MyPackage
$ cd MyPackage
$ swift package init # or swift package init --type library
$ swift build
$ swift test
```
2. 使用Xcode创建
File -> New -> Swift Package
![3](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/SwiftPM3.png)

## 发布一个 SwiftPM
```
$ git init
$ git add .
$ git remote add origin [github-URL]
$ git commit -m "Initial Commit"
$ git tag 1.0.0
$ git push origin master --tags
```

## SwiftPM 内部结构
![内部结构](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/SwiftPM6.png)

一个 package 主要由如下三个部分组成, 它们的概念前面已经介绍过：
- Dependencies
- Targets
- Products

一个常见的 `Package.swift` 配置文件内容大致如下图：
![](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/SwiftPM5.png)

## SwiftPM 的应用
[R.swift](https://github.com/mac-cain13/R.swift)  
R.swift 是一个高效引入资源的 iOS 框架，防止错误的使用字符串导致的崩溃等问题  
R.swift 是用 SwiftPM 生成一个可执行文件，通过添加 run script 脚本，在编译阶段获取项目资源文件，生成对应的代码方便直接调用, 比如图片字体等。


## 参考
[官方文档](https://swift.org/package-manager/)
[Getting to Know Swift Package Manager](https://developer.apple.com/videos/play/wwdc2018/411/)
[Creating Swift Packages](https://developer.apple.com/videos/play/wwdc2019/410)
[WWDC 2018：细说 Swift 包管理工具 (Swift Package Manager)](https://juejin.im/post/5b1f536a5188257d9b79dbcf)
