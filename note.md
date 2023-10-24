
# 分布式任务调度子系统（轻量系统）

## 三个模块

dmsfwsk_lite：
safwk_lite：
samgr_lite：系统服务框架基于面向服务的架构，提供了服务开发、服务的子功能开发、对外接口的开发、以及多服务共进程、进程间服务调用等开发能力

```plantuml
    !theme plain
    component 轻量分布式调度{
        component 分布式框架{
            component 远程启动
            component 远程调用
        }
        component 服务管理{
            component 服务注册
            component 服务发现
            component 服务间通信
        }
    }

```

## SamgrLite与SOA架构

### SOA架构

>SOA，即面向服务的架构（Service-Oriented Architecture），是一种构建计算机软件的设计模式，其核心思想是将功能分解为单独的、定义良好的服务。这些服务通常为独立部署的、独立运行的软件组件，它们可以跨多种系统、应用和网络进行通信和交互。

```plantuml
    !theme plain
    rectangle Samgr
    rectangle Provider
    rectangle Consumer

    Samgr <-down- Consumer : 发现服务
    Samgr <-down- Provider : 注册特性和服务
    Provider <-right- Consumer : 调用服务
```
#### 通信方式：
```plantuml
    !theme plain
    title 进程内同步通信
    rectangle Provider
    rectangle Consumer
    Provider <-right- Consumer : 直接调用服务
```
```plantuml
    !theme plain
    title 进程内异步通信
    rectangle Provider
    rectangle Consumer
    
    rectangle 本地过程调用的消息枢纽
    rectangle 本地过程调用的广播枢纽

    本地过程调用的消息枢纽 <-right- Consumer : 发送请求信息
    Provider <-up- 本地过程调用的消息枢纽 : 接受请求信息
    本地过程调用的广播枢纽 <-left- Provider : 发布事件
    Consumer <-left- 本地过程调用的广播枢纽 : 通知订阅者
```
```plantuml
   !theme plain
    title 进程内异步通信
    rectangle Provider
    rectangle Consumer
    
    rectangle 进程间通信消息接收枢纽
    rectangle 进程间通信消息发送枢纽

    进程间通信消息接收枢纽 <-right- Consumer : 发送请求信息
    Provider <-up- 进程间通信消息接收枢纽 : 接受请求信息
    进程间通信消息发送枢纽 <-left- Provider : 发送响应信息
    Consumer <-left- 进程间通信消息发送枢纽 : 接受响应信息
```
-   Provider：服务的提供者，为系统提供能力（对外接口）。
-   Consumer：服务的消费者，调用服务提供的功能（对外接口）。
-   Samgr：作为中介者，管理Provider提供的能力，同时帮助Consumer发现Provider的能力。


```plantuml
    !theme plain
    hide empty members
    Struct SamgrLite{
    	RegisterService()
    	UnregisterService()
    	RegisterFeatrue()
    	UnregiseterFeature()
    	RegisterDefaultFeatureApi()
    	UnregisterDefaultFeatureApi()
    	RegisterFeatureApi()
    	UnregisterFeatureApi()
    	GetDefaultFeatureApi()
    	GetFeatureApi()
    	AddSystemCapability()
    	HasSystemCapability()
    	GetSystemAvailableCapabilities()
    }
    Struct Vector{
    max : int16
    top int16
    free : int16
    data: void **
    key : VECTOR_Key
    compare : VECTOR_Compare
    }
    Struct SamgrLiteImpl{
    vtbl : SamgrLite
    mutex : MutexId
    status : BootStatus
    services : Vector
    sharedPool : TaskPool[]
    }


    SamgrLiteImpl--> SamgrLite
    SamgrLiteImpl--> Vector


    note left of SamgrLiteImpl::vtbl
    samgr利用SamgrLite实现
    对系统服务和特性的管理
    end note

    note left of SamgrLiteImpl::mutex
    互斥锁其实现与底层平台
    的标准CMSIS、POSIX有关
    end note

```

### SamgrLite

>foundation/distributedschedule/samgr_lite/interfaces/kits/samgr/samgr_lite.h

samgr利用SamgrLite实现对系统服务和特性的管理

### MutexId

```c
//foundation/distributedschedule/samgr_lite/interfaces/kits/samgr/common.h
typedef void *MutexId;
```

互斥锁，用于保护对共享资源的访问
其实现与平台所用接口（CMSIS, POSIX）相关：

```plantuml
    !theme plain
    package adpter{
        file memory_adapter
        file queue_adapter
        file thread_adapter
        file time_adapter
        file build.gn
        package posix
        package cmsis

        posix --|>memory_adapter
        posix --|>queue_adapter
        posix --|>thread_adapter
        posix --|>time_adapter

        cmsis --|>memory_adapter
        cmsis --|>queue_adapter
        cmsis --|>thread_adapter
        cmsis --|>time_adapter

        note top of build.gn
        根据ohos_kernel_type决定构建的依赖
        liteos_m内核则链接cmsis包下的源文件
        liteos_a或linux内核则链接posix包下的源文件
        end note
    }
```

### BootStatus
>foundation/distributedschedule/samgr_lite/samgr/source/samgr_lite_inner.h

```plantuml
    !theme plain
    [*] --> BOOT_SYS : SYS_INIT()初始化赋予
    BOOT_SYS -down-> BOOT_APP : Bootstrap服务调用 \n INIT_APP_CALL()
    BOOT_APP -down-> BOOT_DYNAMIC : 系统所有应用程序启动完毕
    BOOT_DYNAMIC --> [*]

    BOOT_SYS : 启动SYS_SERVICE_INIT和 \n SYS_FEATURE_INIT宏修饰的服务与特性
    BOOT_APP : 启动所有SYSSEX_SERVICE_INIT,APP_SERVICE_INIT \n SYSSEX_FEATURE_INIT, APP_FEATURE_INIT宏修饰的服务与特性

```

### Vector
>foundation/distributedschedule/samgr_lite/interfaces/kits/samgr/common.h

支持动态扩容的数组，扩容策略为重新申请一块增大4个单元的内存空间

```c
typedef void *(*VECTOR_biKey)(const void *);
typedef int (*VECTOR_Compare)(const void *, const void *);

typedef struct SimpleVector {
    ...
    VECTOR_Key key; //将元素转化为key，用以比较
    VECTOR_Compare compare; //比较两个key的大小
    //在samgr_lite.c中，key为GetServiceName(), compare为strcmp()
} Vector;
```

### SharedPool[]

```plantuml
    !theme plain
    :SAMGR_Bootstrap()为每一个服务创建消息队列和任务池参数;
    :samgr调用服务的GetTaskConfig()获取其任务配置;
    if (TaskConfig->taskFlags == SHARED_TASK) then (yes)
    :service的任务池指向samgr的共用任务池,以节约系统资源;
    detach
    else (no)
    :服务拥有独立消息队列和任务池，记录在serviceImpl中;
    detach
```

```plantuml
    !theme plain
    hide empty members
    struct TaskConfig {
    level : int16
    priority : int16
    stackSize : uint16
    queueSize : uint16
    taskFlags : uint8
    }
    enum TaskType {
    SHARED_TASK
    SINGLE_TASK
    SPECIFIED_TASK
    NO_TASK
    }

    TaskConfig --> TaskType

    note left of TaskType::SHARED_TASK
    基于它们的priority被多个服务共享的，
    例如BootStap服务
    end note
    note right of TaskType::SPECIFIED_TASK
    这是一个由多个服务共享的特定任务。
    与 SHARED_TASK 不同，这不是基于
    优先级进行共享，而是允许多个指定
    的服务共享同一个任务
    end note
```

### Service

```plantuml
    !theme plain
    hide empty members
    Struct Vector{
    max : int16
    top int16
    free : int16
    data: void **
    key : VECTOR_Key
    compare : VECTOR_Compare
    }
    Struct SamgrLiteImpl{
    vtbl : SamgrLite
    mutex : MutexId
    status : BootStatus
    services : Vector
    sharedPool : TaskPool[]
    }
    Struct ServiceImpl{
        service: Service
        defaultApi: IUnknown
        taskPool: TaskPool
        features: Vector
        serviceId: int16
        inited: uint8
        ops: Operations
    }
    Struct Service{
        GetName()   //获取服务名称
        Initialize()    //服务初始化
        MessageHandle() //消息处理函数
        GetTaskConfig() //获取服务的任务运行配置
    }

    SamgrLiteImpl --> Vector
    Vector -right-> ServiceImpl : data[*]
    ServiceImpl -up-> Service

    note right of ServiceImpl::service
    指向具体服务对象
    end note

    note right of ServiceImpl::defaultApi
    通过宏INHERIT_IUNKNOWN继承IUnknown接口；
    用以记录服务或特性的对象引用数量；
    用以实现父类IUnkown指针到具体服务或特性指针的类型转换
    end note

    note right of ServiceImpl::service
    指向具体服务对象
    end note

    note right of ServiceImpl::serviceId
    与g_samgrImpl->services->data的偏移量一致
    end note

    note right of ServiceImpl::inited
    服务状态：SVC_INIT,SVC_IDLE,SVC_BUSY
    处于SVC_INIT状态可以注册特性，处于SVC_IDLE状态可以处理消息事件
    end note

    note right of ServiceImpl::ops
    //todo
    end note

    note right of Service
    所有服务类的父类，每一个具体的服务
    类继承该类并实现其4个生命周期函数
    end note
```
### Servcie类的3个子类具体实现

//todo 三个服务的具体实现的区别与其优缺点

//todo C语言宏实现继承,IUnkown的QueryInterface操作

```plantuml
    !theme plain
    hide empty members
    Struct Bootstrap{
        identity : Identity
        flag : unit8
    }
    Struct BroadcastService{
    }
    Struct HiviewService{
        identity : Identity
    }
    Struct HiviewInterface{
        output()
    }
    Struct Service{
        GetName()   //获取服务名称
        Initialize()    //服务初始化
        MessageHandle() //消息处理函数
        GetTaskConfig() //获取服务的任务运行配置
    }
    Struct IUnknown{
        QueryInterface()
        AddRef()
        Release()
    }
    
    Bootstrap -up-|> Service
    BroadcastService -up-|> Service
    HiviewService -up-|> Service
    HiviewService -down-|> HiviewInterface
    HiviewInterface -right-|> IUnknown

    note bottom of Bootstrap
    调用INIT_APP_CALL()，以启动App服务与特性
    (被SYSEX\APP_SERVICE\FEATURE_INIT宏修饰)
    end note

    note bottom of BroadcastService
    该服务的功能由其PubSubFeature提供
    end note

    note "DFX的子系统，提升软件质量设计的工具集" as n1
    n1 .. HiviewService

```

>`IUnknown` 是 COM (Component Object Model) 技术中的一个核心接口。COM 是 Microsoft 开发的一个组件化编程模型，广泛用于 Windows 操作系统中。`IUnknown` 接口为所有 COM 对象提供了基本的对象生命周期管理和接口查询机制。
>
>`IUnknown` 提供了三个核心方法：
>1. **AddRef()**: 增加对象的引用计数。
>2. **Release()**: 减少对象的引用计数，并在引用计数达到零时删除对象。
>3. **QueryInterface()**: 允许客户端查询对象是否支持特定的接口，并获取该接口的指针。
任何想要遵循 COM 规则并能够在 COM 环境中使用的对象都必须实现 `IUnknown` 接口。此外，COM 对象的生命周期管理是通过引用计数完成的，即当一个客户端开始使用一个对象时，它会调用 `AddRef`，当不再使用它时会调用 `Release`。当对象的引用计数达到零时，对象会自动销毁。
>
>`QueryInterface` 方法允许对象在运行时查询它是否支持特定的接口。这是一种动态类型检查机制，允许安全地在对象上调用特定的方法。


####BroadcastService

```plantuml
    !theme plain
    hide empty members
    Struct SamgrLiteImpl{
        vtbl : SamgrLite
        mutex : MutexId
        status : BootStatus
        services : Vector
        sharedPool : TaskPool[]
    }
    Struct Service{
        GetName()
        Initialize()
        MessageHandle()
        GetTaskConfig()
    }
    Struct BroadcastService{
    }
    Struct IUnknown{
        QueryInterface()
        AddRef()
        Release()
    }
    Struct ServiceImpl{
        service: Service
        defaultApi: IUnknown
        taskPool: TaskPool
        features: Vector
        serviceId: int16
        inited: uint8
        ops: Operations
    }
    Struct PubSubImplement{
        addTopic()
        subscribe()
        modifyConsumer()
        unsubscribe()
        publish()
        feature : PubSubFeature
    }
    Struct PubSubInterface{
        subscriber : Subscriber
        provider : Provider
    }
    Struct PubSubFeature{
        GetRelation()
        mutex : MutexId
        relations : Relation
        identity : Identity
    }
    Struct FeatureImpl{
        feature : Feature
        iUnknown : IUnknown
    }
    Struct Feature{
        GetName()
        OnInitialize()
        OnStop() //停止对外提供特性
        OnMessage() //消息处理
    }
    Struct Relation {
        node : UTILS_DL_LIST
        callbacks : ConsumerNode
        topic : Topic 
    }

    SamgrLiteImpl --> ServiceImpl : services->data[x]
    ServiceImpl -left-> BroadcastService 
    ServiceImpl -right-> PubSubImplement : features->data[x]
    BroadcastService --|> Service
    PubSubImplement --|> PubSubInterface
    PubSubImplement --|> FeatureImpl
    PubSubInterface --|> IUnknown
    PubSubImplement -right-> PubSubFeature
    PubSubFeature --|> Feature
    PubSubFeature -right-> Relation

    note top of PubSubImplement
    提供对外接口
    end note

    note top of PubSubFeature
    广播的底层实现
    维护一个双向链表Relation
    end note

    note top of Relation
    双向链表
    维护主题和订阅者
    end note

```

### 从系统的拉起过程看服务注册与发现