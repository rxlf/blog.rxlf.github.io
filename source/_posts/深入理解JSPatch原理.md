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

JSVirtualMachine是为js代码的运行提供一个虚拟机环境，用于执行js代码，JSVirtualMachine是单线程的，如果想多个线程执行，可以建立多个JSVirtualMachine对象，每个JSVirtualMachine有自己独立的垃圾回收器（js内存管理方式是垃圾回收机制，具体原理自行google）,一般不需要自己手动创建，系统会默认创建JSVirtualMachine  

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

<!-- more -->

然后在js中这样调用就可以了
`resourcePath("filePath")`

#### 通过 JavaScripCore 中的 JSExport
JSExport 可以导出 Objective-C 的属性、实例方法、类方法到 JS 环境，这样就可 以通过 JS 代码直接调⽤用 Objective-C 。通过 JSExport 不仅可以导出自定义类的方法、属性，也可以导出已有类的方法、属性。  
在导出过程中，方法名会合并，第二个参数首字母大写，比如
`- (int)addA:(int)a addB:(int)b;`会被转换成`addAAddB(a, b);`  
那如何导出自定义的类和对象呢？可以通过实现JSExport协议
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
@property (nonatomic, copy) NSString *name;
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
// 要执⾏的 JS 代码，定义⼀一个 add 函数并执⾏
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
WKWebView底层其实也是通过 JSContext 来调用的，可以查看[WKWebView源码](https://opensource.apple.com/source/WebKit2/WebKit2-7605.3.8/UIProcess/API/Cocoa/WKWebView.mm.auto.html)  
在上层，我们只需要这样使用就可以了
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

### JPEngine
 从Demo中，我们可以看到，在app启动时，会调用`[JPEngine startEngine];`方法，在这个方法中，回去初始化一个单例的js上下文环境`JSContext`。  
 当`JSContext`初始化完成，JSPatch想要执行下发的脚本，就必须要在JSCore中找到对应的方法，同时，添加或者替换的方法、协议等最终是在OC里动态生产或者替换的，需要将OC将转换接口暴露给JS，这些方法就是OC和JS的桥梁，最终通过runtime动态的添加、替换执行。  
 在`startEngine`中，提供了以下方法给JS  
- `_OC_defineClass`: 添加或者替换的方法、属性等的方法
- `_OC_defineProtocol`: 定义添加协议的方法
- `_OC_callI`: 调用实例方法
- `_OC_callC`: 调用类方法
- `_OC_formatJSToOC`: JS转换成OC的方法
- `_OC_formatOCToJS`: OC转成JS的方法
- `_OC_getCustomProps`: 获取自定义属性
- `_OC_setCustomProps`: 增加自定义属性
- `__weak`: weak的实现
- `__strong`: strong的实现
- `_OC_superClsName`: 获取父类名
- `autoConvertOCType`: 自动转换成OC类型
- `convertOCNumberToString`: 自动转换 Number 到 ŒString
- `include`: 引入其他 JS 文件
- `resourcePath`: 资源文件路径
- `dispatch_after`: GCD的延迟执行
- `dispatch_async_main`: 主线程异步执行
- `dispatch_sync_main`: 主线程同步执行
- `dispatch_async_global_queue`: 全局队列异步执行
- `releaseTmpObj`: 释放临时对象
- `_OC_log`: 把 console.log 打印信息转换到 NSLog 打印
- `_OC_catch`: 把 JS 中的 try catch 转换到 OC 中
- `_OC_null`: 定义 nil 对象

这里只需要知道这些方法的作用，具体的实现我们放到后面demo里一步一步说明

### JSPatch.js  
`JSPatch.js`是一个自动执行函数，即不需要别处调用，就会自动调用
```
;(function() {
     
     // 其他省略的JS代码
      
      var _customMethods = {
    __c: function(methodName) {
      var slf = this
      // self 是不是 Boolean实例
      if (slf instanceof Boolean) {
        return function() {
          return false
        }
      }
      if (slf[methodName]) {
        return slf[methodName].bind(slf);
      }

      if (!slf.__obj && !slf.__clsName) {
        throw new Error(slf + '.' + methodName + ' is undefined')
      }
      if (slf.__isSuper && slf.__clsName) {
          slf.__clsName = _OC_superClsName(slf.__obj.__realClsName ? slf.__obj.__realClsName: slf.__clsName);
      }
      var clsName = slf.__clsName
      if (clsName && _ocCls[clsName]) {
        var methodType = slf.__obj ? 'instMethods': 'clsMethods'
        if (_ocCls[clsName][methodType][methodName]) {
          slf.__isSuper = 0;
          return _ocCls[clsName][methodType][methodName].bind(slf)
        }
      }

      return function(){
        var args = Array.prototype.slice.call(arguments)
        return _methodFunc(slf.__obj, slf.__clsName, methodName, args, slf.__isSuper)
      }
    },

    super: function() {
      var slf = this
      if (slf.__obj) {
        slf.__obj.__realClsName = slf.__realClsName;
      }
      return {__obj: slf.__obj, __clsName: slf.__clsName, __isSuper: 1}
    },

    performSelectorInOC: function() {
      var slf = this
      var args = Array.prototype.slice.call(arguments)
      return {__isPerformInOC:1, obj:slf.__obj, clsName:slf.__clsName, sel: args[0], args: args[1], cb: args[2]}
    },

    performSelector: function() {
      var slf = this
      var args = Array.prototype.slice.call(arguments)
      return _methodFunc(slf.__obj, slf.__clsName, args[0], args.splice(1), slf.__isSuper, true)
    }
  }
  for (var method in _customMethods) {
    if (_customMethods.hasOwnProperty(method)) {
      Object.defineProperty(Object.prototype, method, {value: _customMethods[method], configurable:false, enumerable: false})
    }
  }

// 其他省略的JS代码

})()
```
在这个文件里，主要是进行 JS 的解析，然后去调用 OC 的方法，`global`相当于定义全局函数，这样在同一个`JSContext`里都可以使用这些方法，
，省略的方法主要有

1. `global.require`
生成一个全局对象，避免调用时找不到方法报错
2. `global.defineClass = function(declaration, properties, instMethods, clsMethods){}` 
各个参数的意思如下，想要添加或重写某个方法就需要使⽤ defineClass 来申明，通过 JS 来告诉 OC 的 runtime，哪些方法需要修改，哪些需要添属性
```
@param classDeclaration: 字符串，类名/父类名和Protocol 
@param properties: 新增property，字符串数组，可省略 
@param instanceMethods: 对象，要添加或覆盖的实例方法
@param classMethods: 要添加或覆盖的类方法
```
3. `global.defineProtocol = function(declaration, instProtos , clsProtos){}`  
定义协议
4. `global.block = function(args, cb){}`定义Block
5. `global.YES = 1`，`global.NO = 0` 字面意思理解，定义OC 中的YES 和 NO
6. `global.nsnull = _OC_null`
7. `global._formatOCToJS = _formatOCToJS`

上面自动执行JS代码中包含一个`_customMethods`对象和一个 `for` 循环，为什么有这个方法呢，我们先来分析`_customMethods`  
可以看到，`_customMethods`定义了4个function, 分别是`__c`, `super`, `performSelectorInOC`, `performSelector`  
然后，后面通过`for`循环遍历了这个对象，这个 `for` 循环的作用就是遍历_customMethods对象中的属性，将其加入原型链，加入原型链后，所有的 JS 对象都会自动拥有这些方法，对 JS 感兴趣的同学可以先自行学习一下 JS，（ PS: 我自己是在大前端呆了一段时间，看了 JSPatch 后被这些 JS 的高级用法给惊住了，作者的功力是真的深厚，原谅我水平低以前都不知道这些方法）
具体实现通过代码中的注释说明
```
  for (var method in _customMethods) {
    // 判断属性是通过继承还是自己实现的，自己实现的返回true，即上面提到的4个方法
    if (_customMethods.hasOwnProperty(method)) {
      // value 属性值 configurable:false 无法删除该属性  enumerable: 无法在for-in循环中遍历该属性
      Object.defineProperty(Object.prototype, method, {value: _customMethods[method], configurable:false, enumerable: false})
    }
  }
```
这四个方法在方法调用中作用重要，几乎每一个方法调用都是使用到这几个方法，后续分析demo时用到再说明。

## 官方JSPatchDemo分析
前面介绍了`JSPatch`中的主要方法，JSPatch 中的关键点就是把补丁中要新增或替换的⽅法与 OC 中的方法对应起来，在 OC ⽅法执行中，如果执⾏的方法是补丁中的方法，那么就要执⾏补丁中的实现，在执补丁中的⽅方法时需要把参数的值传递给它。JSPatch 中使用 defineClass 与 OC 中的 runtime 交互，当某个补丁下发时，需要告诉 runtime 新增的属性，要添加或替换的类方法、实例方法，类名，父类名，协议，这样 runtime 即可根据这些信息 进⾏修改。  
首先，在`application:didFinishLaunchingWithOptions:`方法中，调用`[JPEngine startEngine];`方法，这个方法会去初始化一个  JSContext , 并且初始化将上面介绍的方法暴露给 JS ，同时 JSContext 会去加载 JSPatch.js 进行上述的初始化操作。接下来，我们看看demo中的列子，来分析方法是如何替换、定义和调用的

### defineClass 的实现
例子中主要通过 JSPatch ，替换了`JPViewController`中的`clickAction:`空方法，实现了`clickAction:`点击跳转到 JSPatch 动态添加的一个 TableView 中，并且点击 TableView 有个 Alert 弹框的功能。  
```
defineClass('JPViewController', {
  handleBtn: function(sender) {
    var tableViewCtrl = JPTableViewController.alloc().init()
    self.navigationController().pushViewController_animated(tableViewCtrl, YES)
  }
})

defineClass('JPTableViewController : UITableViewController <UIAlertViewDelegate>', ['data'], {
  dataSource: function() {
    var data = self.data();
    if (data) return data;
    var data = [];
    for (var i = 0; i < 20; i ++) {
      data.push("cell from js " + i);
    }
    self.setData(data)
    return data;
  },
  numberOfSectionsInTableView: function(tableView) {
    return 1;
  },
  tableView_numberOfRowsInSection: function(tableView, section) {
    return self.dataSource().length;
  },
  tableView_cellForRowAtIndexPath: function(tableView, indexPath) {
    var cell = tableView.dequeueReusableCellWithIdentifier("cell") 
    if (!cell) {
      cell = require('UITableViewCell').alloc().initWithStyle_reuseIdentifier(0, "cell")
    }
    cell.textLabel().setText(self.dataSource()[indexPath.row()])
    return cell
  },
  tableView_heightForRowAtIndexPath: function(tableView, indexPath) {
    return 60
  },
  tableView_didSelectRowAtIndexPath: function(tableView, indexPath) {
     var alertView = require('UIAlertView').alloc().initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles("Alert",self.dataSource()[indexPath.row()], self, "OK",  null);
     alertView.show()
  },
  alertView_willDismissWithButtonIndex: function(alertView, idx) {
    console.log('click btn ' + alertView.buttonTitleAtIndex(idx).toJS())
  }
})

```
这段 JS 代码是如何实现上述功能的呢，首先，在 OC 中，会调用`+ (JSValue *)_evaluateScript:(NSString *)script withSourceURL:(NSURL *)resourceURL` 方法，在经过一系列的异常处理逻辑，添加自动执行函数，正则表达式替换所有的方法调用后，代码就变成了这样
```
;(function(){try{
defineClass('JPViewController', {
  handleBtn: function(sender) {
    var tableViewCtrl = JPTableViewController.__c("alloc")().__c("init")()
    self.__c("navigationController")().__c("pushViewController_animated")(tableViewCtrl, YES)
  }
})

defineClass('JPTableViewController : UITableViewController <UIAlertViewDelegate>', ['data'], {
  dataSource: function() {
    var data = self.__c("data")();
    if (data) return data;
    var data = [];
    for (var i = 0; i < 20; i ++) {
      data.__c("push")("cell from js " + i);
    }
    self.__c("setData")(data)
    return data;
  },
  numberOfSectionsInTableView: function(tableView) {
    return 1;
  },
  tableView_numberOfRowsInSection: function(tableView, section) {
    return self.__c("dataSource")().length;
  },
  tableView_cellForRowAtIndexPath: function(tableView, indexPath) {
    var cell = tableView.__c("dequeueReusableCellWithIdentifier")("cell") 
    if (!cell) {
      cell = require('UITableViewCell').__c("alloc")().__c("initWithStyle_reuseIdentifier")(0, "cell")
    }
    cell.__c("textLabel")().__c("setText")(self.__c("dataSource")()[indexPath.__c("row")()])
    return cell
  },
  tableView_heightForRowAtIndexPath: function(tableView, indexPath) {
    return 60
  },
  tableView_didSelectRowAtIndexPath: function(tableView, indexPath) {
     var alertView = require('UIAlertView').__c("alloc")().__c("initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles")("Alert",self.__c("dataSource")()[indexPath.__c("row")()], self, "OK",  null);
     alertView.__c("show")()
  },
  alertView_willDismissWithButtonIndex: function(alertView, idx) {
    console.__c("log")('click btn ' + alertView.__c("buttonTitleAtIndex")(idx).__c("toJS")())
  }
})
}catch(e){_OC_catch(e.message, e.stack)}})();
```

通过前面，我们知道，所有 JS 方法都已通过原型链增加了`__c`方法，保证所有方法都在 JS `__c` 中转发，调用不出错。  
接下来，我们看 JS 中 `defineClass` 做了些什么  
首先，会去判断是不是传递了属性，因为在方法定义上 properties 是可省略的，并且 properties 是一个数组，所以, 首先判断 properties 是不是数组，如果是，则直接跳过，如果不是，说明省略了 properties ，实际是实例方法，因此，将 properties 赋值给 instMethods ，instMethods 赋值给 clsMethods
```
    // 判断 properties 是属性还是方法
    if (!(properties instanceof Array)) {
      clsMethods = instMethods
      instMethods = properties
      properties = null
    }
```
接下来，判断是否添加了属性，如果添加了属性，会给属性增加 get 方法和 set 方法， 这样，就把属性转换成了方法，通过关联对象，在 OC 中动态添加属性
```
    if (properties) {
      properties.forEach(function(name){
        // 给属性增加get方法
        if (!instMethods[name]) {
          instMethods[name] = _propertiesGetFun(name);
        }
        // 给属性增加set方法
        var nameOfSet = "set"+ name.substr(0,1).toUpperCase() + name.substr(1);
        if (!instMethods[nameOfSet]) {
          instMethods[nameOfSet] = _propertiesSetFun(name);
        }
      });
    }
```
然后，会通过`_formatDefineMethods`去格式化传递给 OC 的方法, 经过这个方法, 所有的方法都会格式化成` [参数个数，自定义函数]`这样形式的数组, 类似于下面这样
``` 
{
   handleBtn: [1, function() {
          try {
            var args = _formatOCToJS(Array.prototype.slice.call(arguments))
            var lastSelf = global.self
            global.self = args[0]
            if (global.self) global.self.__realClsName = realClsName
            args.splice(0,1)
            var ret = originMethod.apply(originMethod, args)
            global.self = lastSelf
            return ret
          } catch(e) {
            _OC_catch(e.message, e.stack)
          }
        }]
}
```
为什么要格式化成这样，我们后面介绍 OC 中 `_OC_defineClass` 再说明。

经过 `_formatDefineMethods`， 下一步，就是到 OC 中动态生成类、方法、属性了, 调用 `var ret = _OC_defineClass(declaration, newInstMethods, newClsMethods)`方法，传入了类的定义，实例方法，类方法。我们看看 OC 中 `defineClass` 做了什么。  

首先，会通过一个sccanner, 去获取 `classDeclaration` 中的 className、superClassName 、 protocolNames  
然后去判断是否存在这个类，如果不存在，则动态生成，代码如下
```
    // 通过String获取类
    Class cls = NSClassFromString(className);
    // 如果没有这个类，动态的生成这个类
    if (!cls) {
        Class superCls = NSClassFromString(superClassName);
        if (!superCls) {
            // 找不到super class，抛出异常
            _exceptionBlock([NSString stringWithFormat:@"can't find the super class %@", superClassName]);
            return @{@"cls": className};
        }
        cls = objc_allocateClassPair(superCls, className.UTF8String, 0);
        objc_registerClassPair(cls);
    }
```
有了类过后，接下来就是动态添加协议以及实例方法，类方法
```
// 给类动态添加协议
    if (protocols.count > 0) {
        for (NSString* protocolName in protocols) {
            Protocol *protocol = objc_getProtocol([trim(protocolName) cStringUsingEncoding:NSUTF8StringEncoding]);
            class_addProtocol (cls, protocol);
        }
    }
    // for循环遍历，首先添加实例方法，再添加类方法
    for (int i = 0; i < 2; i ++) {
        BOOL isInstance = i == 0;
        JSValue *jsMethods = isInstance ? instanceMethods: classMethods;
      
        // 如果是实例方法，取出类对象，如果是类对象，在元类上添加
        Class currCls = isInstance ? cls: objc_getMetaClass(className.UTF8String);
        NSDictionary *methodDict = [jsMethods toDictionary];
        for (NSString *jsMethodName in methodDict.allKeys) {
            JSValue *jsMethodArr = [jsMethods valueForProperty:jsMethodName];
            int numberOfArg = [jsMethodArr[0] toInt32];
            // 将 JS 中下划线 _ 的函数名转换成 OC 中的 SEL
            NSString *selectorName = convertJPSelectorString(jsMethodName);
            
            // 判断 : 数量是否小于了 参数数量，小于，在最后添加一个:, 只要是有参数的方法，必定会走进来
            if ([selectorName componentsSeparatedByString:@":"].count - 1 < numberOfArg) {
                selectorName = [selectorName stringByAppendingString:@":"];
            }
            
            JSValue *jsMethod = jsMethodArr[1];
            if (class_respondsToSelector(currCls, NSSelectorFromString(selectorName))) {
                overrideMethod(currCls, selectorName, jsMethod, !isInstance, NULL);
            } else {
                BOOL overrided = NO;
                for (NSString *protocolName in protocols) {
                    char *types = methodTypesInProtocol(protocolName, selectorName, isInstance, YES);
                    if (!types) types = methodTypesInProtocol(protocolName, selectorName, isInstance, NO);
                    if (types) {
                        overrideMethod(currCls, selectorName, jsMethod, !isInstance, types);
                        free(types);
                        overrided = YES;
                        break;
                    }
                }
                if (!overrided) {
                    if (![[jsMethodName substringToIndex:1] isEqualToString:@"_"]) {
                        NSMutableString *typeDescStr = [@"@@:" mutableCopy];
                        for (int i = 0; i < numberOfArg; i ++) {
                            [typeDescStr appendString:@"@"];
                        }
                        overrideMethod(currCls, selectorName, jsMethod, !isInstance, [typeDescStr cStringUsingEncoding:NSUTF8StringEncoding]);
                    }
                }
            }
        }
    }
```
这里用到了我们之前说的 参数个数 ，所以它的作用主要有两个
1. 修复方法名，在 OC 中， 方法名 SEL 是 `function:param:param:`这样的形式，所以需要根据参数个数，在最后补充`:`
2. 获取方法签名，在runtime 中，添加方法 ` class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp,
const char * _Nullable types)`, 其中除了 type 外, 其他参数都可以取到，如果方法存在，可以通过`method_getTypeEncoding`获取，否则只能通过参数个数自己拼接，在 JSPatch 中，所有的参数都是id类型，所以可以根据参数个数，拼接成 type ，可以参考[签名规则](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

得到签名后，就会进入`overrideMethod`, 顾名思义，就是覆盖原来的方法，我们看看里面的具体实现
```
static void overrideMethod(Class cls, NSString *selectorName, JSValue *function, BOOL isClassMethod, const char *typeDescription)
{
    SEL selector = NSSelectorFromString(selectorName);
    
    // 没有传，说明是本来就有的方法
    if (!typeDescription) {
        Method method = class_getInstanceMethod(cls, selector);
        typeDescription = (char *)method_getTypeEncoding(method);
    }
    
    // 获得原方法IMP
    IMP originalImp = class_respondsToSelector(cls, selector) ? class_getMethodImplementation(cls, selector) : NULL;
    
    // 手动消息转发
    IMP msgForwardIMP = _objc_msgForward;
    
    // 在非 arm64 下，是 special struct 就走 _objc_msgForward_stret，否则走 _objc_msgForward。
    // 什么时候用 _objc_msgForward_stret 可以参考下面的链接
    #if !defined(__arm64__)
        if (typeDescription[0] == '{') {
            //In some cases that returns struct, we should use the '_stret' API:
            //http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html
            //NSMethodSignature knows the detail but has no API to return, we can only get the info from debugDescription.
            NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:typeDescription];
            if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound) {
                msgForwardIMP = (IMP)_objc_msgForward_stret;
            }
        }
    #endif
    
    // 当前消息转发的方法不是 JPForwardInvocation ，就进入下面消息转发
    if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) != (IMP)JPForwardInvocation) {
        // 将消息转发给 JPForwardInvocation，同时添加执行原来消息转发 ORIG 方法
        IMP originalForwardImp = class_replaceMethod(cls, @selector(forwardInvocation:), (IMP)JPForwardInvocation, "v@:@");
        if (originalForwardImp) {
            class_addMethod(cls, @selector(ORIGforwardInvocation:), originalForwardImp, "v@:@");
        }
    }
  
    // 目测应该是修复7.1版本以下的一个BUG ，忽略
    [cls jp_fixMethodSignature];
  
    if (class_respondsToSelector(cls, selector)) {
        // 增加执行原方法，原方法加上 ORIG
        NSString *originalSelectorName = [NSString stringWithFormat:@"ORIG%@", selectorName];
        SEL originalSelector = NSSelectorFromString(originalSelectorName);
        if(!class_respondsToSelector(cls, originalSelector)) {
            class_addMethod(cls, originalSelector, originalImp, typeDescription);
        }
    }
    
    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
    
    // 用来缓存 JSPatch 重写方法的 类
    _initJPOverideMethods(cls);
    // 缓存类的 JS path 执行方法
    _JSOverideMethods[cls][JPSelectorName] = function;
    
    // Replace the original selector at last, preventing threading issus when
    // the selector get called during the execution of `overrideMethod`
    // 最后进行转发
    class_replaceMethod(cls, selector, msgForwardIMP, typeDescription);
}
```
这里面主要是把消息转发到 `JPForwardInvocation` 里，去实现消息转发，同时替换了原来的实现，加上`ORIG`前缀，并且缓存了 JSPatch 重写后的类以及方法，方法名加上了 `JP` 前缀，在转发中，我们就可以通过缓存找到对应的方法，在 JS 中去执行。  
`defineClass`这里就说完了，其中还有动态添加协议，这里就不作说明了

### 方法调用
在 OC 中动态添加类、方法、协议、和属性后，我们如何去调用这些方法，并且在调用过程中，如果是替换的方法或者新增的方法，需要到 JS 中去实现，参数如何传递到 JS 里面去？比如前面的`handleBtn:`方法，需要sender参数，这个参数如何获取，并传递给 JS ？

> 当调⽤⼀个 NSObject 对象不存在的⽅法时，并不会马上抛出异常，⽽是会经过多层转发，层层调用对象的`-esolveInstanceMethod:`, `forwardingTargetForSelector:`, `methodSignatureForSelector:`, `forwardInvocation:` 等方法，其中最后 `forwardInvocation:` 是会有⼀个 `NSInvocation` 对象，通过命令模式，这个 `NSInvocation` 对象保存了这个方法调用的所有信息，包括 Selector 名，参数和返回值类型，最重要的是有所有参数值，可以从这个 `NSInvocation` 对象里拿到调用的所有参数值 - JSPatch

有了，`forwardInvocation`, 我们只需要将方法 IMP 指向 `_objc_msgForward` 或者 `_objc_msgForward_stret`, 这样，当调用方法是，就会走到`forwardInvocation`方法里。同时，在这个方法里，我们能拿到 `NSInvocation`， 也就是能拿到所有的调用参数。  

那么如果想执行被替换前的`forwardInvocation`怎么办，这里再引用作者的一段话
> 我们把 ViewController 的 -forwardInvocation: 方法的实现给替换掉了，如果程序里真有用到这个方法对消息进⾏转发，原来的逻辑怎么办?⾸先我们在替换 -forwardInvocation: 方法前会 新建一个⽅方法 -ORIGforwardInvocation:，保存原来的实现IMP，在新的 -forwardInvocation: 实现⾥里里做了了个判断，如果转发的⽅法是我们想改写的，就⾛我们的逻辑，若不是，就调 - ORIGforwardInvocation: ⾛原来的流程。

即这一段代码
```
JSValue *jsFunc = getJSFunctionInObjectHierachy(slf, JPSelectorName);
if (!jsFunc) {
    JPExecuteORIGForwardInvocation(slf, selector, invocation);
return; }
```

那么实际方法是如何调用的呢，我们看一下关键部分代码，作者将获得的参数保存在一个数组`params`中，如果转发这个消息的不是自己，那么这个数组第一个参数总是一个 `JPBoxing`对象，否则才是自身，这样做的原因是因为在把参数传给 JS 时，需要告诉 JS `self` 对象，以便在 JS 中调用 `self` 不报错。
调用的关键部分作者采用的宏定义来实现，这里想要说一点，作者为了节省代码量，全篇大量使用了宏定义，由于水平问题，阅读起来还是比较困难，只能看懂大概意思，我们看下 JSPatch 是如何通过 `JPForwardInvocation` 将消息转发给 JS 执行。
```
        #define JP_FWD_RET_CALL_JS \
            JSValue *jsval; \
            [_JSMethodForwardCallLock lock];   \
            jsval = [jsFunc callWithArguments:params]; \
            [_JSMethodForwardCallLock unlock]; \
            while (![jsval isNull] && ![jsval isUndefined] && [jsval hasProperty:@"__isPerformInOC"]) { \
                NSArray *args = nil;  \
                JSValue *cb = jsval[@"cb"]; \
                if ([jsval hasProperty:@"sel"]) {   \
                    id callRet = callSelector(![jsval[@"clsName"] isUndefined] ? [jsval[@"clsName"] toString] : nil, [jsval[@"sel"] toString], jsval[@"args"], ![jsval[@"obj"] isUndefined] ? jsval[@"obj"] : nil, NO);  \
                    args = @[[_context[@"_formatOCToJS"] callWithArguments:callRet ? @[callRet] : _formatOCToJSList(@[_nilObj])]];  \
                }   \
                [_JSMethodForwardCallLock lock];    \
                jsval = [cb callWithArguments:args];  \
                [_JSMethodForwardCallLock unlock];  \
            }
```
没错，这是一个宏定义。。其中的关键在于 `jsval = [jsFunc callWithArguments:params];`, 通过这行代码，又调用到了 JS 里，就是前面介绍的通过`_formatDefineMethods`格式化传递给 OC 的`[参数个数，自定义函数]`数组里面的自定义函数, 然后回去调用原方法，如果你点击 button ，那么就会调用到 DefineClass 里面的 `handleBtn:` 方法。

我们以`handleBtn`方法为例，看看是如何初始化一个 `JPTableViewController` 的
```
defineClass('JPViewController', {
  handleBtn: function(sender) {
    var tableViewCtrl = JPTableViewController.__c("alloc")().__c("init")()
    self.__c("navigationController")().__c("pushViewController_animated")(tableViewCtrl, YES)
  }
})
```
首先转发消息给`JPTableViewController`的`__c`方法(`JPTableViewController` 是在 JS 中 `defineClass` 通过require方法设置的全局变量)，然后，消息就转发给了`__c`方法，`__c`中主要是一些异常处理、缓存、父类方法调用的处理。最终，会调用到`_methodFunc`中，
我们看看`_methodFunc`
```
  var _methodFunc = function(instance, clsName, methodName, args, isSuper, isPerformSelector) {
    var selectorName = methodName
    if (!isPerformSelector) {
      methodName = methodName.replace(/__/g, "-")
      selectorName = methodName.replace(/_/g, ":").replace(/-/g, "_")
      var marchArr = selectorName.match(/:/g)
      var numOfArgs = marchArr ? marchArr.length : 0
      if (args.length > numOfArgs) {
        selectorName += ":"
      }
    }
    // 转发给OC执行
    var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
                         _OC_callC(clsName, selectorName, args)
    return _formatOCToJS(ret)
  }
```
对，这里面主要是处理参数，将其转发给 OC，`_OC_callI`和`_OC_callC`最终都会走到`static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)` 方法中。  
这个方法比较长，其中有很多对签名的处理，我们只需要知道，我们通过参数，获取签名，然后生成了`NSInvocation`对象。
最后通过`[invocation invoke];`调用方法，初始化了一个 JPTableViewController，通过 `[invocation getReturnValue:&result];`获得返回值返回给 JS ，也就是 JPTableViewController 的实例对象， JS 拿到这个对象，就可以进行下一步的调用。

### 总结
以上一个完整的补丁调用过程，可以总结为
- 补丁下发
- 补丁格式替换为 `__c`
- 对需要添加或修改的⽅法进⾏处理，传递给 OC 使⽤ runtime 动态的添加或者替换方法，并保存原方法的实现
- 通过消息转发，调用下发的 JS 函数
- JS 调用 `__c`
- `__c`将需要 OC 实现的方法转发给 `_OC_callI` 或者 `_OC_callI`

## 结尾
本篇只介绍了最基本的方法调用，其中还涉及到父类，Block， GCD等相关知识。读者可自己去学习源码  
注释源码已上传至[GitHub](https://github.com/rxlf/RxlfDemo)

## 参考文章
- [JSPatch 实现原理理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-实现原理详解)
- [JS知识](https://www.liaoxuefeng.com/wiki/1022910821149312/1023020745357888)
- [知识小集](https://github.com/awesome-tips/demo)