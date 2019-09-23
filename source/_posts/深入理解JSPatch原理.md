---
title: 深入理解JSPatch原理
date: 2019-09-22 18:15:06
tags: 
- js
- iOS
---

苹果已经禁止JSPatch上线，为什么还要写这一篇，一是对自己学习的总结，二是目前大前端非常火，借此了解如何用JS开发原生界面。  
## JavaScirptCore框架
在介绍JSPatch之前，先了解一下基础知识  
苹果官方对JavaScriptCore框架的说明，可以通过这个[链接](https://developer.apple.com/documentation/javascriptcore?language=occ)查看，可以看到，从结构上，JavaScriptCore主要由JSVirtualMachine、JSContext、JSValue三部分组成。  

JSVirtualMachine是为js代码的运行提供一个虚拟机环境，用于执行js代码，JSVirtualMachine是单线程的，如果想多个线程执行，可以建立多个JSVirtualMachine对象，每个SVirtualMachine有自己独立的垃圾回收器（js内存管理方式是垃圾回收机制，具体原理自行google）,一般不需要自己手动创建，系统会默认创建JSVirtualMachine  

JSContext是js运行环境的上下文，和JSPatch交互首先需要有JSContext。  

JSValue是js的值对象，记录js的原始值，并且提供了原生值对象和js对象的转换接口。对应关系如下：

oc类型 | js类型 
:-: | :-: 
nil | undefined |
NSNull | null | 
NSString | string | 
NSNumber | number、boolean | 
NSDictionary | Object | 
NSArray | Array | 
NSDate | Date | 
NSBlock | Function | 

### JS调用OC方法
JS想要调用oc的方法，首先JS是不知道OC中有哪些方法可以调用的，所以需要OC先告诉JS哪些方法可以调用，目前主要有两种方式
+ JavaScripCore中的block
+ JavaScripCore中的JSExport

#### 通过 JavaScripCore 中的 block
JSPatch中就是通过block的方式，举个JSPatch中的例子（主要是懒，不想自己写）
```
    context[@"resourcePath"] = ^(NSString *filePath) {
        return [_scriptRootDir stringByAppendingPathComponent:filePath];
    };
```
然后在js中这样调用就可以了
`resourcePath("filePath")`

#### 通过 JavaScripCore 中的 JSExport
JSExport 可以导出 Objective-C 的属性、实例例⽅方法、类⽅方法和初始化⽅方法到 JS 环境，这样就可 以通过 JS 代码直接调⽤用 Objective-C 。通过 JSExport 不不仅可以导出⾃自定义类的⽅方法、属性，也可以导出已有类的⽅方法、属性。  
在导出过程中，方法名会合并，第二个参数首字母大写，比如
`- (int)addA:(int)a addB:(int)b;`会被转换成`addAAndB(a, b);`  
那如何导出自定义的类已经对象呢？可以通过实现JSExport协议
```
@protocol MYExportClassProtocol<JSExport>
@property (nonatomic, copy) NSString *name;
+ (MYExportClass *)myExportClass;
- (int)addA:(int)a addB:(int)b;
@end
```
然后实现这个协议
```
@interface MYExportClass : NSObject<MYExportClassProtocol>

+ (MYExportClass *)myExportClass;
- (int)addA:(int)a addB:(int)b;

@end
```
然后这样使用
```
// 导出对象
MYExportClass *myObject = [MYExportClass myExportClass]; 
self.context[@"_OC_Object"] = myObject;
// 导出类
self.context[@"_OC_Class"] = [MYExportClass class];
```

##### 导出已有类的方法和属性
```
@protocol UILabelExportProtocol<JSExport>
@property (nullable, nonatomic, copy) NSString *text;
@end
```
已有类可以通过runtime，动态的添加一个协议，比如
```
// 通过 runtime 给 UILabel 添加协议 UILabelExportProtocol class_addProtocol([UILabel class], @protocol(UILabelExportProtocol));
self.context[@"_OC_label"] = [UILabel class];
```

### OC调用JS方法
#### 第一种方式，通过JavaScriptCore调用
首先创建或者获取JSContext
```
// 创建context
_context = [[JSContext alloc] init]; 
// 设置 context 的名字后，调试的时候可以看到对应环境名称
_context.name = @"debug.context";

//获取context，webView主要通过这种方式
JSContext *context = [_webView valueForKeyPath:@"documentView.webView.mainFrame.JSContext"];
```
获取到context后，可以通过`evaluateScript:withSourceURL:`获取js执行结果
```
// 要执⾏行行的 JS 代码，定义⼀一个 add 函数并执⾏行行
NSString *addjs = @"function add(a, b) {return a + b;};add(1,3)";
// sumValue 为执⾏行行后的结果, withSourceURL 只是为了调试，不会影响执行结果
JSValue *sumValue = [self.context evaluateScript:addjs withSourceURL: [NSURL URLWithString:@"add.js"]];
NSLog(@"sum: %@", @([sumValue toInt32])); // 4
```
也可以通过`callWithArguments:`来调用js中的方法
```
NSString *addjs = @"function add(a, b) {return a + b;}";
[self.context evaluateScript:addjs withSourceURL:[NSURL
URLWithString:@"add.js"]];
JSValue *resultValue = [self.context[@"add"] callWithArguments:@[@2, @4]];
NSLog(@"Result: %@", @([resultValue toInt32])); // 6
```

#### 通过WKWebView调用
WKWebView底层其实也是通过jscontext来调用的，可以查看[WKWebView源码](https://opensource.apple.com/source/WebKit2/WebKit2-7605.3.8/UIProcess/API/Cocoa/WKWebView.mm.auto.html)  
在上层，我们只需要这样使用就可以
```
[self.webView evaluateJS:@"function add(a, b) {return a + b;};add(1,3)" completionHandler:^(id _Nullable msg, NSError * _Nullable error) {
   NSLog(@"evaluateJS add: %@, error: %@", msg, error);
}];
```
******

## JSPatch原理解析
前面介绍了JavaScriptCore，其实在JSPatch中还有很多JS的高级用法，本篇不再详细介绍，同时也不需要深入了解，在源码解析的时候用到的地方在解释  
先贴一个JSPatch GitHub链接 ---> [这是链接](https://github.com/bang590/JSPatch)  
下载下来后，我们发现，其实整个JSPatch只有三个文件，JSPatch.js、JPEngine.h、JPEngine.m , JPEngine.h只是JPEngine.m暴露出来的接口，可以忽略，所以我们只需要看 JSPatch.js 和 JPEngine.m 就可以了  

首先JPEngine.m

JSPatch.js  

---->未完待继续补充<-------