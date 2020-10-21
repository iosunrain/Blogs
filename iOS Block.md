![借图](https://upload-images.jianshu.io/upload_images/1194012-3b6a5c9d5edb1aae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/900/format/webp)

Blocks是C语言的扩充功能，而Apple 在OS X Snow Leopard 和 iOS 4中引入了这个新功能“Blocks”。从那开始，Block就出现在iOS和Mac系统各个API中，并被大家广泛使用。
### 一、什么是Block？
 最简洁的概述就是：Block就是将**函数**及其**执行上下文**封装起来的**对象**
这里我们可以通过`clang -rewrite-objc 文件名.m`命令，讲文件转译成c++的源码，看到block的本质。
oc这样一段代码
```
int(^Block)(int) = ^int(int num)
    {
        return num * multiplier;
    };
```
转化之后
```
 int(*Block)(int) = ((int (*)(int))&__MCBlock__method_block_impl_0((void *)__MCBlock__method_block_func_0, &__MCBlock__method_block_desc_0_DATA, &multiplier));

```
其中__MCBlock__method_block_impl_0是一个结构体，其中传入了三个参数
- (void *)__MCBlock__method_block_func_0（一个函数指针），
- __MCBlock__method_block_desc_0_DATA（block相关描述），
 - &multiplier（block内使用的变量）
```
struct __MCBlock__method_block_impl_0 {
  struct __block_impl impl;
  struct __MCBlock__method_block_desc_0* Desc;
  int *multiplier;
  __MCBlock__method_block_impl_0(void *fp, struct __MCBlock__method_block_desc_0 *desc, int *_multiplier, int flags=0) : multiplier(_multiplier) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
}; 
```
在结构体中`struct __block_impl impl;`对应的
```
struct __block_impl {
  void *isa;//
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```
在这个结构体中包含`void *isa;`表明Block本质上也是个OC对象。
下面通过block的调用可以更清楚的看清block
使用`Block(2)`转译后的代码就是：
```
((int (*)(__block_impl *, int))((__block_impl *)Block)->FuncPtr)((__block_impl *)Block, 2)
```
可以看出其中`(__block_impl *)Block->FuncPtr`是通过**函数指针获取函数执行体** FuncPtr对应的就是最初传入的`(void *)__MCBlock__method_block_func_0`
再看看c++转译后的执行函数
```
static int __MCBlock__method_block_func_0(struct __MCBlock__method_block_impl_0 *__cself, int num) {
  int *multiplier = __cself->multiplier; // bound by copy
        return num * (*multiplier);
    }
```
这个函数方法，和最上面的block的oc代码是不是很像，其中`__cself`是传入的block,` num`就是传入的变量，通过block获取其捕获的multiplier，然后执行计算。到这里，block的工作原理就算结束了。
### 二、block是如何截获外部变量的？
先上代码
```objc
// 全局变量
int global_var = 4;
// 静态全局变量
static int static_global_var = 5;

- (void)method
{
    //基本数据类型局部变量
    int var = 1;
    //对象类型局部变量
    __unsafe_unretained id unsafe_obj = nil;
    __strong id strong_obj = nil;
    //局部静态变量
    static int static_var = 6;
    int(^Block)(int) = ^int(int num)
    {
//        __block
        NSLog(@"%d",var);
        NSLog(@"%@",unsafe_obj);
        NSLog(@"%@",strong_obj);
        NSLog(@"%d",static_var);
        NSLog(@"%d",global_var);
        NSLog(@"%d",static_global_var);
        return num * multiplier;
    };
    multiplier = 4;
    NSLog(@"result is %d", Block(2));
}
```
```swift
struct __MCBlock__method_block_impl_0 {
  struct __block_impl impl;
  struct __MCBlock__method_block_desc_0* Desc;
    // 截获局部变量的值
  int var;
    // 连同所有权修饰符一起截获
  __unsafe_unretained id unsafe_obj;
  __strong id strong_obj;
    // 以指针形式截获局部变量
  int *static_var;
    // 对全局变量、静态全局变量不截获
    
  __MCBlock__method_block_impl_0(void *fp, struct __MCBlock__method_block_desc_0 *desc, int _var, __unsafe_unretained id _unsafe_obj, __strong id _strong_obj, int *_static_var, int flags=0) : var(_var), unsafe_obj(_unsafe_obj), strong_obj(_strong_obj), static_var(_static_var) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
从上面我们可以清晰的看出:
- **基本类型的局部变量**
捕获其值
- **对象类型局部变量**
连同所有权修饰符一起截获
- **静态局部变量**
 以指针形式截获静态局部变量
- **全局变量、静态全局变量**
不捕获
### 三、_ _block是干嘛用的？
通常情况下，需要对被截获局部变量进行**赋值**操作时，需要使用。
赋值不是使用，`[arr addobject:@"123"]`这样只是使用。
###### __block是如何做到的呢
当局部变量使用__block修饰符修饰的时候，就自动转变成了已给对象类型的结构体
`__block int multiplier `这句话就被转译成下面
```
struct  _Block_byref_mutiplier_0{
      void *__isa;
      __Block_byref_multiplier_0 *__forwarding;
      int  __flags;
      int  __size;
      int  multiplier;
}
```
之后再对block修饰的对象使用或者赋值时类似这句`multiplier = 4`就被转译成了
`(multiplier.__forwarding->mutiplier)=4`
当需要获取变量值的时候，就通过forwarding指针找到对应结构体内对应变量。
下面是经典的__forwarding指针的示意图。
当__block只存在于栈上时，__forwarding指针是指向自己的，而当被copy到堆上时，堆上的__forwarding指针指向自己，而栈上的__forwarding指针则指向堆上的block，这样__forwarding指针始终指向的是同一对象。
![借图](https://upload-images.jianshu.io/upload_images/1763170-32cc95576866dd91.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)
### 四、block有几种类型
刚才提到堆栈，就说一下block的几种类型
- _NSConcreteGlobalBlock //全局的静态block 不会访问任何外部变量
- _NSConcreteMallocBlock //保存在堆区的，引用计数为0时会被销毁
- _NSConcreteStackBlock //保存在栈区，出栈后被销毁
待续。。。
### 五、block循环引用
- 自循环引用

![image.png](https://upload-images.jianshu.io/upload_images/6579721-515f13d153d0cffd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
block被当前对象持有，而当_array被block捕获时，连同属性关键字一起捕获，所以block持有当前对象的成员变量，就形成了循环引用。
当block发生自循环引用时使用__weak通过弱引用解决自循环引用
- 大环引用
**下面的代码在MRC下，__block修饰对象不会增加引用计数，所以不会有问题，
但是在ARC下，__block修饰对象会被强引用，就存在循环引用的问题**

```
__block id blockSelf = self;
_block = ^int(int num){
    resault = blockSelf.var
  return resault;
};

```
![image.png](https://upload-images.jianshu.io/upload_images/6579721-f944c68bc325a720.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以应该是
```objc
__block id blockSelf = self;
_block = ^int(int num){
    resault = blockSelf.var
    blockSelf =nil;
  return resault;
};

```
当然这存在一个问题，如果block很久或者一直不被调用，那么这个环就一直存在。

# 六、其他
- 为什么在MRC下__block可以解除循环引用，而ARC下不行？
在MRC下，__block修饰的对象在使用时，引用计数不会增加，ARC下会增加。
- 为什么block内部不能修改局部变量，需要__blcok才行呢
从block捕获变量可以看出，除了静态变量和全局变量，局部变量并不捕获其内存地址

  
欢迎留言。