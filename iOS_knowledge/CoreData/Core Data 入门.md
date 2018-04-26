# Core Data 入门

##  认识几个重点类
### NSManagedObjectContext
被管理的数据上下文
直接操作实际内容（持久层）
作用：插入数据、查询数据、删除数据
### NSManagedObjectModel
被管理的数据模型，数据库所有表格或数据结构，包含各实体的定义信息，
作用：添加实体的属性，建立属性间的关系
### NSPersistentStoreCoordinator
持久化存储助理，相当于数据库的连接器，作用：设置数据存储的名字、位置、存储方式和存储时机
### NSManagedObject
被管理的数据记录，相当于数据库中的表格记录
### NDFetchRequest
获取数据的请求，相当于查询语句
### NSEntityDescription
实体结构，相当于表格结构
### .xcdatamodel文件
工程中存放数据的表，可以用数据模型编辑器编辑，编译后为.momd或.mom文件


## Sqlite & Core Data分析
[iOS 端数据库解决方案分析](http://ios.jobbole.com/90523/)

> CoreData默认使用的是sqlite的多线程模式，这种模式下不能跨线程共享数据库的连接，虽然不清楚CoreData的内部实现细节，总体使用下来感觉一个NSManagedObjectContext对应一个数据库连接，同时再维护一套自己的object graph，object graph并不是多线程安全的，object graph当中的object 不能跨线程直接共享，NSManagedObjectContext也不能跨线程使用。所以使用CoreData建立多线程模型的时候有如下规则：
> 
 * 不同的线程要建立自己的NSManagedObjectContext，维护各自的object graph。
 * NSManagedObject不能跨线程传递使用，只要通过传递NSManagedObjectID，再通过ID去从各自的Context中获取Object。

> 不同的context之间并不是自动同步数据的，在write context写入的数据并不能直接在main context中读取到。我们需要自己建立同步机制，一般有两种方式。

###  监听context的写通知

```
//主线程监听write context的写操作
[[NSNotificationCenter defaultCenter] addObserver:self.observer
                                                 selector:@selector(mocDidSave:)
         name:NSManagedObjectContextDidSaveNotification
                                                   object:self];
                                                   
                                                   /merge 来自 write context中的数据变化
NSError *error = nil;
        [[self managedObjectContextForMainThreadWithError:&error] mergeChangesFromContextDidSaveNotification:saveNotification];
```

### 共享context

为了避免多个context之间的merge操作，可以在多个context之间建立paret child关系，使用这种方式一般会建立一个公共的background context，其他所有的main context和background context都是它的child。这种方式确实可以避免merge的问题，但我感觉本质上是把所有的读和写操作都串行化了，虽然最后读写行为都是在子线程发生，但并发的性能反而不如方式一好。


