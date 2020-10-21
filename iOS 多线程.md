今天我们从大家最关心的GCD和NSOperation共同和不通开始
![image](http://upload-images.jianshu.io/upload_images/6579721-a28ca26ecf0d9c9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们可以看到，NSOperation Queue 作为高级 API，有很多 GCD 没有的功能，如需要支持：控制并发数、取消、添加依赖关系等需要使用 NSOperation Queue。
另外，由于 block 可复用性没有 NSOperation 好，对于独立性强、可复用性高的任务建议使用 NSOperation 实现。
当然，NSOperation 在使用时需要 sub-classing，工作量较大，对于简单的任务使用 GCD 即可。
如果看的有点懵，下面，我们再详细了解。
### GCD
首先四种不同的状态的线程
```
    dispatch_queue_t  serial_queue = dispatch_queue_create("xxx", DISPATCH_QUEUE_SERIAL);
    dispatch_queue_t  concurrent_queue = dispatch_queue_create("xxx", DISPATCH_QUEUE_CONCURRENT);
    dispatch_sync(serial_queue,^{});//同步串行
    dispatch_sync(concurrent_queue,^{});//同步并发
    dispatch_async(serial_queue,^{});//异步串行
    dispatch_async(concurrent_queue,^{});//异步并发
```
**Global Queue & Main Queue**
这是系统为我们准备的2个队列：
Global Queue其实就是系统创建的Concurrent Diapatch Queue
Main Queue 其实就是系统创建的位于主线程的Serial Diapatch Queue
通常情况我们会把这2个队列放在一起使用，也是我们最常用的开异步线程-执行异步任务-回主线程的一种方式：
```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"异步线程");
    dispatch_async(dispatch_get_main_queue(), ^{
        NSLog(@"异步主线程");
    });
});
```
`dispatch_get_global_queue`存在优先级，没错，他一共有4个优先级：
```
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```
**dispatch_suspend & dispatch_resume**
队列**挂起和恢复**，这个没什么好说的，直接上代码：
```
dispatch_queue_t concurrentDiapatchQueue=dispatch_queue_create("com.test.queue", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(concurrentDiapatchQueue, ^{
    for (int i=0; i<100; i++)
    {
        NSLog(@"%i",i);
        if (i==50)
        {
            NSLog(@"-----------------------------------");
            dispatch_suspend(concurrentDiapatchQueue);
            sleep(3);
            dispatch_async(dispatch_get_main_queue(), ^{
                dispatch_resume(concurrentDiapatchQueue);
            });
        }
    }
});
```
我们甚至可以在不同的线程对这个队列进行挂起和恢复，因为GCD是对队列的管理。

- 案例：多图返回后拼接成一个大图的方案
划重点：` dispatch_group_t group = dispatch_group_create();`
` dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        // 当添加到组中的所有任务执行完成之后会调用该Block});`
```
 // 创建一个group
    dispatch_group_t group = dispatch_group_create();
    // for循环遍历各个元素执行操作
    for (NSURL *url in arrayURLs) {
        // 异步组分派到并发队列当中
        dispatch_group_async(group, concurrent_queue, ^{
            //根据url去下载图片
            NSLog(@"url is %@", url);
        });
    }
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        // 当添加到组中的所有任务执行完成之后会调用该Block
        NSLog(@"所有图片已全部下载完成");
    });
```
- 案例：在多读单写情况下使用异步栅栏调用方法
划重点` dispatch_barrier_async(concurrent_queue, ^{});`
```
#import "UserCenter.h"

@interface UserCenter()
{
    // 定义一个并发队列
    dispatch_queue_t concurrent_queue;
    
    // 用户数据中心, 可能多个线程需要数据访问
    NSMutableDictionary *userCenterDic;
}

@end

// 多读单写模型
@implementation UserCenter

- (id)init
{
    self = [super init];
    if (self) {
        // 通过宏定义 DISPATCH_QUEUE_CONCURRENT 创建一个并发队列
        concurrent_queue = dispatch_queue_create("read_write_queue", DISPATCH_QUEUE_CONCURRENT);
        // 创建数据容器
        userCenterDic = [NSMutableDictionary dictionary];
    }
    
    return self;
}

- (id)objectForKey:(NSString *)key
{
    __block id obj;
    // 同步读取指定数据
    dispatch_sync(concurrent_queue, ^{
        obj = [userCenterDic objectForKey:key];
    });
    
    return obj;
}

- (void)setObject:(id)obj forKey:(NSString *)key
{
    // 异步栅栏调用设置数据
    dispatch_barrier_async(concurrent_queue, ^{
        [userCenterDic setObject:obj forKey:key];
    });
}

@end
```



### NSOpretion
**特点**
- 添加任务依赖
- 任务执行状态控制
- 最大并发量
NSOperation 本身是个抽象类，在使用前必须子类化，系统预定义了两个子类：NSInvocationOperation 和 NSBlockOperation。

```
// 创建操作
    // 使用 NSInvocationOperation 创建操作1
    NSInvocationOperation *op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];
    [op1 start];
  // 使用 NSBlockOperation 创建操作2
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    //添加
    [op2 addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"4---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [op2 start];

```
NSInvocationOperation 和 NSBlockOperation本身创建并使用`start`方法是不会开辟线程的，都是在**当前线程**中执行.
##### 想要异步执行就需要结合NSOperationQueue来执行
```
/**
 * 使用 addOperation: 将操作加入到操作队列中
 */
- (void)addOperationToQueue {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];
 // 2.设置最大并发操作数
    queue.maxConcurrentOperationCount = 1; // 串行队列
// queue.maxConcurrentOperationCount = 2; // 并发队列

    // 3.创建操作
    // 使用 NSInvocationOperation 创建操作1
    NSInvocationOperation *op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];

    // 使用 NSBlockOperation 创建操作2
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [op3 addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"4---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.使用 addOperation: 添加所有操作到队列中
    [queue addOperation:op1]; // [op1 start]
    [queue addOperation:op2]; // [op2 start]
    [queue addOperation:op3]; // [op3 start]
}
```
##### 最大并发操作数：`maxConcurrentOperationCount`
-  默认情况下为-1，表示不进行限制，可进行并发执行。
-  为1时，队列为串行队列。只能串行执行。
-  大于1时，队列为并发队列。操作并发执行，当然这个值不应超过系统限制，即使自己设置一个很大的值，系统也会自动调整为 min{自己设定的值，系统设定的默认最大值}。
##### 线程间依赖
NSOperation、NSOperationQueue 最吸引人的地方是它能添加操作之间的依赖关系。通过操作依赖，我们可以很方便的控制操作之间的执行先后顺序。NSOperation 提供了3个接口供我们管理和查看依赖。

- (void)addDependency:(NSOperation *)op; 添加依赖，使当前操作依赖于操作 op 的完成。
- (void)removeDependency:(NSOperation *)op; 移除依赖，取消当前操作对操作 op 的依赖。
@property (readonly, copy) NSArray<NSOperation *> *dependencies; 在当前操作开始执行之前完成执行的所有操作对象数组。

当然，我们经常用到的还是添加依赖操作。现在考虑这样的需求，比如说有 A、B 两个操作，其中 A 执行完操作，B 才能执行操作。
如果使用依赖来处理的话，那么就需要让操作 B 依赖于操作 A。具体代码如下：
```
/**
 * 操作依赖
 * 使用方法：addDependency:
 */
- (void)addDependency {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.添加依赖
    [op2 addDependency:op1]; // 让op2 依赖于 op1，则先执行op1，在执行op2

    // 4.添加操作到队列中
    [queue addOperation:op1];
    [queue addOperation:op2];
}
```
##### 状态控制

##### 自定义NSOperation注意事项
- 如果同步执行，只需要重写main方法，其他由系统控制
- 如果要异步执行，需要重写start方法，所有方法需要自己控制，可以参考AFURLConnectionOperation
- 线程安全（使用NSRecursvieLock）

[NSOpretion详尽总结 ](https://juejin.im/post/5a9e57af6fb9a028df222555)

### 锁
###### @synchronized 
作用是创建一个互斥锁，保证此时没有其它线程对self对象进行修改。这个是objective-c的一个锁定令牌，防止self对象在同一时间内被其它线程访问，起到线程的保护作用。
**通常在多线程环境下创建单例对象时使用**
###### atomic 原子
属性关键字，线程安全的，但安全有限。
**在赋值时可以保证线程安全，使用时并不保证**
###### OSSpinLock 自旋锁
![](https://upload-images.jianshu.io/upload_images/6579721-e9715b5f85e64238.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

循环等待询问，不释放当前资源
用于轻量级数据访问。
###### NSLock
【lock lock】
【lock unlock】
###### NSRecursvieLock
递归锁
###### dispatch_semaphore_t 信号量
信号量的阻塞行为是主动的
信号量的唤醒行为是被动的
```
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_async(group, queue, ^{
            dispatch_semaphore_signal(semaphore);

    });
    dispatch_group_async(group, queue, ^{
            dispatch_semaphore_signal(semaphore);
         
    });
  
    dispatch_group_notify(group, queue, ^{
        
        // 两个请求对应三次信号等待
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)；
        //在这里 进行请求后的方法，回到主线程
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.tableView.mj_header endRefreshing];
        });
       
```
### 总结：
在我们日常开发中，通常使用最多的是GCD，它可以解决我们大多数需求，比如简单的线程同步，子线程的分派，多读单写等。而NSOperation在我们需要方便控制任务状态和依赖关系时，是更好的选择（AF和SD有很多使用）。最轻量的NSThread因为需要管理线程的周期，通常我们需要常驻线程的时候使用。


