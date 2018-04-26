iOS开发中的锁


# iOS 中的八大锁
锁是最常用的同步工具。一段代码段在同一个时间只能允许被有限个线程访问，比如一个线程 A 进入需要保护代码之前添加简单的互斥锁，另一个线程 B 就无法访问，只有等待前一个线程 A 执行完被保护的代码后解锁，B 线程才能访问被保护代码。

## NSLock

```
@protocol NSLocking

- (void)lock;
- (void)unlock;

@end

@interface NSLock : NSObject <NSLocking> {
@private
    void *_priv;
}

- (BOOL)tryLock;
- (BOOL)lockBeforeDate:(NSDate *)limit;

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end

```

NSLock 遵循 NSLocking 协议，lock 方法是加锁，unlock 是解锁，tryLock 是尝试加锁，如果失败的话返回 NO，lockBeforeDate: 是在指定Date之前尝试加锁，如果在指定时间之前都不能加锁，则返回NO。

举个🌰

```
    //主线程中
    NSLock *lock = [[NSLock alloc] init];
    
    //线程1
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lock];
        NSLog(@"线程1");
        sleep(2);
        [lock unlock];
        NSLog(@"线程1解锁成功");
    });

    //线程2
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);//以保证让线程2的代码后执行
        [lock lock];
        NSLog(@"线程2");
        [lock unlock];
    });

2016-08-19 14:23:09.659 ThreadLockControlDemo[1754:129663] 线程1
2016-08-19 14:23:11.663 ThreadLockControlDemo[1754:129663] 线程1解锁成功
2016-08-19 14:23:11.665 ThreadLockControlDemo[1754:129659] 线程2

```

线程 1 中的 lock 锁上了，所以线程 2 中的 lock 加锁失败，阻塞线程 2，但 2 s 后线程 1 中的 lock 解锁，线程 2 就立即加锁成功，执行线程 2 中的后续代码。

查到的资料显示互斥锁会使得线程阻塞，阻塞的过程又分两个阶段，第一阶段是会先空转，可以理解成跑一个 while 循环，不断地去申请加锁，在空转一定时间之后，线程会进入 waiting 状态，此时线程就不占用CPU资源了，等锁可用的时候，这个线程会立即被唤醒。

所以如果将上面线程 1 中的 sleep(2); 改成 sleep(10); 输出的结果会变成

```
2016-08-19 14:25:16.226 ThreadLockControlDemo[1773:131824] 线程1
2016-08-19 14:25:26.231 ThreadLockControlDemo[1773:131831] 线程2
2016-08-19 14:25:26.231 ThreadLockControlDemo[1773:131824] 线程1解锁成功

```

从上面的两个输出结果可以看出，线程 2 lock 的第一秒，是一直在轮询请求加锁的，因为轮询有时间间隔，所以 ”线程 2“ 的输出晚于 ”线程 1 解锁成功“，但线程 2 lock 的第九秒，是当锁可用的时候，立即被唤醒，所以 ”线程 2“ 的输出早于 ”线程 1 解锁成功“。多做了几次试验，发现轮询 1 秒之后，线程会进入 waiting 状态。

```
    //主线程中
    NSLock *lock = [[NSLock alloc] init];
    
    //线程1
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lock];
        NSLog(@"线程1");
        sleep(10);
        [lock unlock];
    });
    
    //线程2
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);//以保证让线程2的代码后执行
        if ([lock tryLock]) {
            NSLog(@"线程2");
            [lock unlock];
        } else {
            NSLog(@"尝试加锁失败");
        }
    });

2016-08-19 11:42:54.433 ThreadLockControlDemo[1256:56857] 线程1
2016-08-19 11:42:55.434 ThreadLockControlDemo[1256:56861] 尝试加锁失败

```

由上面的结果可得知，tryLock 并不会阻塞线程。[lock tryLock] 能加锁返回 YES，不能加锁返回 NO，然后都会执行后续代码。

如果将 [lock tryLock] 替换成

[lock lockBeforeDate:[NSDate dateWithTimeIntervalSinceNow:10]

的话，则会返回 YES，输出 “线程 2“，lockBeforeDate: 方法会在所指定 Date 之前尝试加锁，会阻塞线程，如果在指定时间之前都不能加锁，则返回 NO，指定时间之前能加锁，则返回 YES。

至于 _priv 和 name，我监测了各个阶段他们的值，_priv 一直都是 NULL。也不知道有什么用，name 是用来标识用的，用来输出 error log 时候作为 lock 的名称。比如解锁状态再解锁，会输出一个 error log。

```
*** -[NSLock unlock]: lock (<NSLock: 0x7a4bdeb0> 'lockName') unlocked when not locked

```

如果是三个线程，那么一个线程在加锁的时候，其余请求锁的线程将形成一个等待队列，按先进先出原则，这个结果可以通过修改线程优先级进行测试得出。

## NSConditionLock

```
@interface NSConditionLock : NSObject <NSLocking> {
@private
    void *_priv;
}

- (instancetype)initWithCondition:(NSInteger)condition NS_DESIGNATED_INITIALIZER;

@property (readonly) NSInteger condition;
- (void)lockWhenCondition:(NSInteger)condition;
- (BOOL)tryLock;
- (BOOL)tryLockWhenCondition:(NSInteger)condition;
- (void)unlockWithCondition:(NSInteger)condition;
- (BOOL)lockBeforeDate:(NSDate *)limit;
- (BOOL)lockWhenCondition:(NSInteger)condition beforeDate:(NSDate *)limit;

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end

```

NSConditionLock 和 NSLock 类似，都遵循 NSLocking 协议，方法都类似，只是多了一个 condition 属性，以及每个操作都多了一个关于 condition 属性的方法，例如 tryLock，tryLockWhenCondition:，NSConditionLock 可以称为条件锁，只有 condition 参数与初始化时候的 condition 相等，lock 才能正确进行加锁操作。而 unlockWithCondition: 并不是当 Condition 符合条件时才解锁，而是解锁之后，修改 Condition 的值，这个结论可以从下面的例子中得出。

```
    //主线程中
    NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:0];
    
    //线程1
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lockWhenCondition:1];
        NSLog(@"线程1");
        sleep(2);
        [lock unlock];
    });
    
    //线程2
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);//以保证让线程2的代码后执行
        if ([lock tryLockWhenCondition:0]) {
            NSLog(@"线程2");
            [lock unlockWithCondition:2];
            NSLog(@"线程2解锁成功");
        } else {
            NSLog(@"线程2尝试加锁失败");
        }
    });
    
    //线程3
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(2);//以保证让线程2的代码后执行
        if ([lock tryLockWhenCondition:2]) {
            NSLog(@"线程3");
            [lock unlock];
            NSLog(@"线程3解锁成功");
        } else {
            NSLog(@"线程3尝试加锁失败");
        }
    });
    
    //线程4
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(3);//以保证让线程2的代码后执行
        if ([lock tryLockWhenCondition:2]) {
            NSLog(@"线程4");
            [lock unlockWithCondition:1];    
            NSLog(@"线程4解锁成功");
        } else {
            NSLog(@"线程4尝试加锁失败");
        }
    });
    
2016-08-19 13:51:15.353 ThreadLockControlDemo[1614:110697] 线程2
2016-08-19 13:51:15.354 ThreadLockControlDemo[1614:110697] 线程2解锁成功
2016-08-19 13:51:16.353 ThreadLockControlDemo[1614:110689] 线程3
2016-08-19 13:51:16.353 ThreadLockControlDemo[1614:110689] 线程3解锁成功
2016-08-19 13:51:17.354 ThreadLockControlDemo[1614:110884] 线程4
2016-08-19 13:51:17.355 ThreadLockControlDemo[1614:110884] 线程4解锁成功
2016-08-19 13:51:17.355 ThreadLockControlDemo[1614:110884] 线程1

```

上面代码先输出了 ”线程 2“，因为线程 1 的加锁条件不满足，初始化时候的 condition 参数为 0，而加锁条件是 condition 为 1，所以加锁失败。locakWhenCondition 与 lock 方法类似，加锁失败会阻塞线程，所以线程 1 会被阻塞着，而 tryLockWhenCondition 方法就算条件不满足，也会返回 NO，不会阻塞当前线程。

回到上面的代码，线程 2 执行了 [lock unlockWithCondition:2]; 所以 Condition 被修改成了 2。

而线程 3 的加锁条件是 Condition 为 2， 所以线程 3 才能加锁成功，线程 3 执行了 [lock unlock]; 解锁成功且不改变 Condition 值。

线程 4 的条件也是 2，所以也加锁成功，解锁时将 Condition 改成 1。这个时候线程 1 终于可以加锁成功，解除了阻塞。

从上面可以得出，NSConditionLock 还可以实现任务之间的依赖。

## NSRecursiveLock

```
@interface NSRecursiveLock : NSObject <NSLocking> {
@private
    void *_priv;
}

- (BOOL)tryLock;
- (BOOL)lockBeforeDate:(NSDate *)limit;

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end

```

NSRecursiveLock 是递归锁，他和 NSLock 的区别在于，NSRecursiveLock 可以在一个线程中重复加锁（反正单线程内任务是按顺序执行的，不会出现资源竞争问题），NSRecursiveLock 会记录上锁和解锁的次数，当二者平衡的时候，才会释放锁，其它线程才可以上锁成功。

```
    NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        
        static void (^RecursiveBlock)(int);
        RecursiveBlock = ^(int value) {
            [lock lock];
            if (value > 0) {
                NSLog(@"value:%d", value);
                RecursiveBlock(value - 1);
            }
            [lock unlock];
        };
        RecursiveBlock(2);
    });

2016-08-19 14:43:12.327 ThreadLockControlDemo[1878:145003] value:2
2016-08-19 14:43:12.327 ThreadLockControlDemo[1878:145003] value:1

```

如上面的示例，如果用 NSLock 的话，lock 先锁上了，但未执行解锁的时候，就会进入递归的下一层，而再次请求上锁，阻塞了该线程，线程被阻塞了，自然后面的解锁代码不会执行，而形成了死锁。而 NSRecursiveLock 递归锁就是为了解决这个问题。

## NSCondition

```
@interface NSCondition : NSObject <NSLocking> {
@private
    void *_priv;
}

- (void)wait;
- (BOOL)waitUntilDate:(NSDate *)limit;
- (void)signal;
- (void)broadcast;

@property (nullable, copy) NSString *name NS_AVAILABLE(10_5, 2_0);

@end

```

NSCondition 的对象实际上作为一个锁和一个线程检查器，锁上之后其它线程也能上锁，而之后可以根据条件决定是否继续运行线程，即线程是否要进入 waiting 状态，经测试，NSCondition 并不会像上文的那些锁一样，先轮询，而是直接进入 waiting 状态，当其它线程中的该锁执行 signal 或者 broadcast 方法时，线程被唤醒，继续运行之后的方法。

用法如下：

```
    NSCondition *lock = [[NSCondition alloc] init];
    NSMutableArray *array = [[NSMutableArray alloc] init];
    //线程1
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lock];
        while (!array.count) {
            [lock wait];
        }
        [array removeAllObjects];
        NSLog(@"array removeAllObjects");
        [lock unlock];
    });
    
    //线程2
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);//以保证让线程2的代码后执行
        [lock lock];
        [array addObject:@1];
        NSLog(@"array addObject:@1");
        [lock signal];
        [lock unlock];
    });

```

也就是使用 NSCondition 的模型为：

锁定条件对象。

测试是否可以安全的履行接下来的任务。

如果布尔值是假的，调用条件对象的 wait 或 waitUntilDate: 方法来阻塞线程。 在从这些方法返回，则转到步骤 2 重新测试你的布尔值。 （继续等待信号和重新测试，直到可以安全的履行接下来的任务。waitUntilDate: 方法有个等待时间限制，指定的时间到了，则放回 NO，继续运行接下来的任务）

如果布尔值为真，执行接下来的任务。

当任务完成后，解锁条件对象。

而步骤 3 说的等待的信号，既线程 2 执行 [lock signal] 发送的信号。

其中 signal 和 broadcast 方法的区别在于，signal 只是一个信号量，只能唤醒一个等待的线程，想唤醒多个就得多次调用，而 broadcast 可以唤醒所有在等待的线程。如果没有等待的线程，这两个方法都没有作用。

## [@synchronized](https://link.jianshu.com?t=http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        @synchronized(self) {
            sleep(2);
            NSLog(@"线程1");
        }
        NSLog(@"线程1解锁成功");
    });
    
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);
        @synchronized(self) {
            NSLog(@"线程2");
        }
    });

2016-08-19 16:42:21.752 ThreadLockControlDemo[2220:208291] 线程1
2016-08-19 16:42:21.752 ThreadLockControlDemo[2220:208291] 线程1解锁成功
2016-08-19 16:42:21.752 ThreadLockControlDemo[2220:208278] 线程2

```

@synchronized(object) 指令使用的 object 为该锁的唯一标识，只有当标识相同时，才满足互斥，所以如果线程 2 中的 @synchronized(self) 改为@synchronized(self.view)，则线程2就不会被阻塞，@synchronized 指令实现锁的优点就是我们不需要在代码中显式的创建锁对象，便可以实现锁的机制，但作为一种预防措施，@synchronized 块会隐式的添加一个异常处理例程来保护代码，该处理例程会在异常抛出的时候自动的释放互斥锁。@synchronized 还有一个好处就是不用担心忘记解锁了。

如果在 @sychronized(object){} 内部 object 被释放或被设为 nil，从我做的测试的结果来看，的确没有问题，但如果 object 一开始就是 nil，则失去了锁的功能。不过虽然 nil 不行，但 @synchronized([NSNull null]) 是完全可以的。^ ^.

## dispatch_semaphore

```
dispatch_semaphore_create(long value);

dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout);

dispatch_semaphore_signal(dispatch_semaphore_t dsema);

```

dispatch_semaphore 是 GCD 用来同步的一种方式，与他相关的只有三个函数，一个是创建信号量，一个是等待信号，一个是发送信号。

```
    dispatch_semaphore_t signal = dispatch_semaphore_create(1);
    dispatch_time_t overTime = dispatch_time(DISPATCH_TIME_NOW, 3 * NSEC_PER_SEC);
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        dispatch_semaphore_wait(signal, overTime);
        sleep(2);
        NSLog(@"线程1");
        dispatch_semaphore_signal(signal);
    });
    

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);
        dispatch_semaphore_wait(signal, overTime);
        NSLog(@"线程2");
        dispatch_semaphore_signal(signal);
    });
    ```
dispatch_semaphore 和 NSCondition 类似，都是一种基于信号的同步方式，但 NSCondition 信号只能发送，不能保存（如果没有线程在等待，则发送的信号会失效）。而 dispatch_semaphore 能保存发送的信号。dispatch_semaphore 的核心是 dispatch_semaphore_t 类型的信号量。

dispatch_semaphore_create(1) 方法可以创建一个 dispatch_semaphore_t 类型的信号量，设定信号量的初始值为 1。注意，这里的传入的参数必须大于或等于 0，否则 dispatch_semaphore_create 会返回 NULL。

dispatch_semaphore_wait(signal, overTime); 方法会判断 signal 的信号值是否大于 0。大于 0 不会阻塞线程，消耗掉一个信号，执行后续任务。如果信号值为 0，该线程会和 NSCondition 一样直接进入 waiting 状态，等待其他线程发送信号唤醒线程去执行后续任务，或者当 overTime  时限到了，也会执行后续任务。

dispatch_semaphore_signal(signal); 发送信号，如果没有等待的线程接受信号，则使 signal 信号值加一（做到对信号的保存）。

从上面的实例代码可以看到，一个 dispatch_semaphore_wait(signal, overTime); 方法会去对应一个 dispatch_semaphore_signal(signal); 看起来像 NSLock 的 lock 和 unlock，其实可以这样理解，区别只在于有信号量这个参数，lock unlock 只能同一时间，一个线程访问被保护的临界区，而如果 dispatch_semaphore 的信号量初始值为 x ，则可以有 x 个线程同时访问被保护的临界区。

##OSSpinLock

​```objc
typedef int32_t OSSpinLock;

bool    OSSpinLockTry( volatile OSSpinLock *__lock );

void    OSSpinLockLock( volatile OSSpinLock *__lock );

void    OSSpinLockUnlock( volatile OSSpinLock *__lock );

```

OSSpinLock 是一种自旋锁，也只有加锁，解锁，尝试加锁三个方法。和 NSLock 不同的是 NSLock 请求加锁失败的话，会先轮询，但一秒过后便会使线程进入 waiting 状态，等待唤醒。而 OSSpinLock 会一直轮询，等待时会消耗大量 CPU 资源，不适用于较长时间的任务。

```
    __block OSSpinLock theLock = OS_SPINLOCK_INIT;
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        OSSpinLockLock(&theLock);
        NSLog(@"线程1");
        sleep(10);
        OSSpinLockUnlock(&theLock);
        NSLog(@"线程1解锁成功");
    });
    
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);
        OSSpinLockLock(&theLock);
        NSLog(@"线程2");
        OSSpinLockUnlock(&theLock);
    });

2016-08-19 20:25:13.526 ThreadLockControlDemo[2856:316247] 线程1
2016-08-19 20:25:23.528 ThreadLockControlDemo[2856:316247] 线程1解锁成功
2016-08-19 20:25:23.529 ThreadLockControlDemo[2856:316260] 线程2

```

拿上面的输出结果和上文 NSLock 的输出结果做对比，会发现 sleep(10) 的情况，OSSpinLock 中的“线程 2”并没有和”线程 1解锁成功“在一个时间输出，而 NSLock 这里是同一时间输出，而是有一点时间间隔，所以 OSSpinLock 一直在做着轮询，而不是像 NSLock 一样先轮询，再 waiting 等唤醒。

## pthread_mutex

```
int pthread_mutex_init(pthread_mutex_t * __restrict, const pthread_mutexattr_t * __restrict);

int pthread_mutex_lock(pthread_mutex_t *);

int pthread_mutex_trylock(pthread_mutex_t *);

int pthread_mutex_unlock(pthread_mutex_t *);

int pthread_mutex_destroy(pthread_mutex_t *);

int pthread_mutex_setprioceiling(pthread_mutex_t * __restrict, int,
  int * __restrict);

int pthread_mutex_getprioceiling(const pthread_mutex_t * __restrict,
  int * __restrict);

```

pthread pthread_mutex 是 C 语言下多线程加互斥锁的方式，那来段 C 风格的示例代码，需要 #import <pthread.h>

```
static pthread_mutex_t theLock;

- (void)example5 {
    pthread_mutex_init(&theLock, NULL);
    
    pthread_t thread;
    pthread_create(&thread, NULL, threadMethord1, NULL);
    
    pthread_t thread2;
    pthread_create(&thread2, NULL, threadMethord2, NULL);
}

void *threadMethord1() {
    pthread_mutex_lock(&theLock);
    printf("线程1\n");
    sleep(2);
    pthread_mutex_unlock(&theLock);
    printf("线程1解锁成功\n");
    return 0;
}

void *threadMethord2() {
    sleep(1);
    pthread_mutex_lock(&theLock);
    printf("线程2\n");
    pthread_mutex_unlock(&theLock);
    return 0;
}

线程1
线程1解锁成功
线程2

```

```
int pthread_mutex_init(pthread_mutex_t * __restrict, const pthread_mutexattr_t * __restrict);

```

首先是第一个方法，这是初始化一个锁，__restrict 为互斥锁的类型，传 NULL 为默认类型，一共有 4 类型。

```
PTHREAD_MUTEX_NORMAL 缺省类型，也就是普通锁。当一个线程加锁以后，其余请求锁的线程将形成一个等待队列，并在解锁后先进先出原则获得锁。

PTHREAD_MUTEX_ERRORCHECK 检错锁，如果同一个线程请求同一个锁，则返回 EDEADLK，否则与普通锁类型动作相同。这样就保证当不允许多次加锁时不会出现嵌套情况下的死锁。

PTHREAD_MUTEX_RECURSIVE 递归锁，允许同一个线程对同一个锁成功获得多次，并通过多次 unlock 解锁。

PTHREAD_MUTEX_DEFAULT 适应锁，动作最简单的锁类型，仅等待解锁后重新竞争，没有等待队列。

```

通过 pthread_mutexattr_t 来设置锁的类型，如下面代码就设置锁为递归锁。实现和 NSRecursiveLock 类似的效果。如下面的示例代码：

```
- (void)example5 {
    pthread_mutex_init(&theLock, NULL);
    
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    pthread_mutex_init(&theLock, &attr);
    pthread_mutexattr_destroy(&attr);
    
    pthread_t thread;
    pthread_create(&thread, NULL, threadMethord, 5);

}

void *threadMethord(int value) {
    pthread_mutex_lock(&theLock);
    
    if (value > 0) {
        printf("Value:%i\n", value);
        sleep(1);
        threadMethord(value - 1);
    }
    pthread_mutex_unlock(&theLock);
    return 0;
}

Value:5
Value:4
Value:3
Value:2
Value:1

```

回到 pthread_mutex，锁初始化完毕，就要上锁解锁了

```
pthread_mutex_lock(&theLock);
pthread_mutex_unlock(&theLock);

```

和 NSLock 的 lock unlock 用法一致，但还注意到有一个 pthread_mutex_trylock 方法，pthread_mutex_trylock 和 tryLock 的区别在于，tryLock 返回的是 YES 和 NO，pthread_mutex_trylock 加锁成功返回的是 0，失败返回的是错误提示码。

pthread_mutex_destroy 为释放锁资源。

至于 pthread_mutex_setprioceiling 和 pthread_mutex_getprioceiling，懵逼脸，这两个用来做什么不太理解。

## 性能

在 [ibireme 的不再安全的 OSSpinLock ](https://link.jianshu.com?t=http://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)一文中，有贴出这些锁的性能对比，如下图：

![img](http://upload-images.jianshu.io/upload_images/1400498-2bfca138992bb7d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

来源：ibireme

当然只是加锁立马解锁的时间消耗，并没有计算竞争时候的时间消耗。可以看出 OSSpinLock 性能最高，但它已经不再安全，如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，由于它会处于轮询的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock。

图中的 pthread_mutex(recursive) 指的是 pthread_mutex 设置为递归锁的情况。

从图中可以知道 @synchronized 的效率最低，不过它的确用起来最方便，所以如果没什么性能瓶颈的话，使用它也不错。

## 总结：

虽然这些锁看起来很复杂，但最终都是加锁，等待，解锁。一下子懂了八锁，有点小激动。

## 参考文章

[iOS中保证线程安全的几种方式与性能对比](https://www.jianshu.com/p/938d68ed832c)
[不再安全的 OSSpinLock](https://link.jianshu.com?t=http://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)
[关于 @synchronized，这儿比你想知道的还要多](https://link.jianshu.com?t=http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

[深入理解iOS开发中的锁](https://bestswifter.com/ios-lock/)
[iOS中保证线程安全的几种方式与性能对比](http://www.jianshu.com/p/938d68ed832c)
[iOS 常见知识点（三）：Lock](http://www.jianshu.com/p/ddbe44064ca4)

[](http://www.tuicool.com/articles/rQNZruA)


