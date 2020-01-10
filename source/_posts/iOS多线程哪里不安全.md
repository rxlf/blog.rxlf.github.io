---
title: iOS多线程哪里不安全
date: 2020-01-10 13:23:05
tags:
---


## 数据在内存中的访问
在内存访问中，所有对内存的访问都是通过地址总线找到地址，数组总线返回数据。我们只有一个地址总线以及数据总线，所以对内存的访问，无论如何都是串行的。  
所以能得出两个结论
1. 内存访问是串行的，并不会导致内存数据的错乱或者crash
2. 如果读写的内存长度小于数据总线的长度，那么读写操作一次完成。比如 bool, int, long 在64位操作系统中的读写都可以看成是原子操作（64位操作系统一次性支持8字节数据的读取）。

## 值类型和引用类型

在 OC 或者 Swift 里面，以及其他语言也是类似的，要区分两个概念，值类型和引用类型。  
值类型是指直接访问内存中的数据，引用类型是指通过指针，间接的访问内存中的数据。  

通过一个例子来简单区分，有这么两个属性: 
```
@property (nonatomic, assign) int a;
@property (nonatomic, strong) NSString *str
```

<!-- more -->

对引用（指针）进行修改:
```
// 赋值后，str 会指向另外一个内存地址
self.str = @"日夕凉风";
```
对指进行修改:
```
self.a = 100;
```
对指针指向的内存区域访问:
```
[self.str rangeOfString:@"风"];
```

两种类型的内存布局大概如下:
![1](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/memoryLaout.png)

### 值类型

```
@property (atomic, assign) int a;

// ThreadA
for (int i = 0; i < 10000; i ++) {
    self.a = self.a + 1;
}

// Thread B
for (int i = 0; i < 10000; i ++) {
    self.a = self.a + 1;
}
```
最后的结果一定是 20000 吗，答案是不一定的，这就是我们常说的 `atomic` 不能保证线程安全的。原因是因为虽然对 `a` 的 `set` 和 `get` 方法进行了原子操作，但是，对于 `self.a = self.a + 1;` 这个语句实际上包含了三步。
1. 取出 `self.a`
2. 执行 `self.a + 1`
3. 将结果赋值给 `self.a`

所以当前线程执行 `self.a + 1` 操作时，其他的线程可能已经执行了若干次了，返回的赋值结果可能会覆盖其他线程的结果，导致结果偏小。  

对于 int 来讲， 64位系统占 4 字节，数据总线一次可以读写 8 个字节，所以对于 int 本身，加不加 `atomic` 都是一样的

### 引用类型

```
@property (atomic, strong) NSString*   stringA;
// Thread A
for (int i = 0; i < 100000; i ++) {
    if (i % 2 == 0) {
        self.stringA = @"a very long string";
    }
    else {
        self.stringA = @"string";
    }
    NSLog(@"Thread A: %@\n", self.stringA);
}

// Thread B
for (int i = 0; i < 100000; i ++) {
    if (self.stringA.length >= 10) {
        NSString* subStr = [self.stringA substringWithRange:NSMakeRange(0, 10)];
    }
    NSLog(@"Thread B: %@\n", self.stringA);
}
```

虽然 `stringA` 是 `atomic` 的 property ，而且在取substring的时候做了length判断，线程B还是很容易crash，因为在前一刻读length的时候 `self.stringA = @"a very long string";`，下一刻取substring的时候线程A已经将`self.stringA = @"string";`，立即出现out of bounds的Exception，crash，多线程不安全。

## atomic
### atomic 作用
1. **生成原子操作的getter和setter。**
设置atomic之后，默认生成的getter和setter方法执行是原子的。也就是说，当我们在线程1执行getter方法的时候（创建调用栈，返回地址，出栈），线程B如果想执行setter方法，必须先等getter方法完成才能执行。举个例子，在32位系统里，如果通过getter返回64位的double，地址总线宽度为32位，从内存当中读取double的时候无法通过原子操作完成，如果不通过atomic加锁，有可能会在读取的中途在其他线程发生setter操作，从而出现异常值。如果出现这种异常值，就发生了多线程不安全。
2. **设置Memory Barrier**
因为存在编译器优化，所以编译器可能在一些场景下先执行line2，再执行line1，因为它认为line1和line2之间并不存在依赖关系，虽然在代码执行的时候，在另一个线程intA和intB存在某种依赖，必须要求line1先于line2执行。
memory barrier能够保证内存操作的顺序，按照我们代码的书写顺序来。

### atomic 局限
通过前面的例子，atomic的作用只是给getter和setter加了个锁，atomic只能保证代码进入getter或者setter函数内部时是安全的，一旦出了getter和setter，多线程安全只能靠程序员自己保障了。所以atomic属性和使用property的多线程安全并没什么直接的联系。另外，atomic由于加锁也会带来一些性能损耗，所以我们在编写iOS代码的时候，一般声明property为nonatomic，在需要做多线程安全的场景，自己去额外加锁做同步。

## 如何做到多线程安全
做到多线程安全，关键还是 `atomic`， 只要做到原子性，小到对变量的访问，大到一段代码的执行，原子性都能保证在代码的执行过程中不被打断，能保证一个线程执行到一半时，不会被另外一个线程抢占。

### 如何实现
加锁，举个例子
```
// Thread A
[_lock lock];
for (int i = 0; i < 100000; i ++) {
    if (i % 2 == 0) {
        self.arr = @[@"1", @"2", @"3"];
    }
    else {
        self.arr = @[@"1"];
    }
    NSLog(@"Thread A: %@\n", self.arr);
}
[_lock unlock];
    
// Thread B
[_lock lock];
if (self.arr.count >= 2) {
    NSString* str = [self.arr objectAtIndex:1];
}
[_lock unlock];
```
### 如何选择哪种锁
借用一张图
![2](https://rxlf-1259783270.cos.ap-chengdu.myqcloud.com/blogImg/lock_performance.jpeg)

可以看到，从上到下依次是性能从高到低，但是, 由于 OSSpinLock 已经不再安全。所以不在使用。  
这些常用的锁，如何选择，这个就看个人喜好，什么用着顺手吧，从横坐标看，都是 us 级别的，所以对于应用层来说，代码的逻辑性更重要。我个人比较喜欢使用 NSLock ，因为简单。。。 

