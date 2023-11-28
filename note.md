
# 分布式任务调度子系统（轻量系统）

## 三个模块

//todo 部署视图，文件和模块

//todo 底层信息通信机制（信号量？）



dmsfwk_lite：分布式任务调度实现（支持FA）
safwk_lite：负责提供基础服务运行的空进程（foundation进程实现），用以启动或初始化服务管理器，并使其持续运行
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
    folder dmsfwk_lite
    folder safwk_lite
    folder samgr_lite
    dmsfwk_lite <-- 分布式框架
    safwk_lite <-right- samgr_lite
    samgr_lite <-- 服务管理

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
    Samgr <-down- Provider : 注册服务和服务
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
    title 进程间异步通信
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
    data : void **
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
    data : void **
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

### 从轻量系统的拉起过程看服务注册与发现

```c
//系统初始化
void OHOS_SystemInit(void)
{
    MODULE_INIT(bsp);
    MODULE_INIT(device);
    MODULE_INIT(core);
    SYS_INIT(service); //注册系统服务
    SYS_INIT(feature); //注册系统服务的特性
    MODULE_INIT(run);
    SAMGR_Bootstrap(); //启功samgr并管理系统服务和特性
}
```

SYS_INIT()负责初始化由SYS_SERVICE_INIT，SYS_FEATURE_INIT宏进行修饰的服务和特性



```c
#define LAYER_INITCALL(func, layer, clayer, >priority)            \
    static const InitCall USED_ATTR >__zinitcall_##layer##_##func \
        __attribute__((section(".zinitcall." clayer >#priority ".init"))) = func
#endif

#define LAYER_INITCALL_DEF(func, layer, clayer) \
    LAYER_INITCALL(func, layer, clayer, 2)

#define SYS_SERVICE_INIT(func) LAYER_INITCALL_DEF(func, sys_service, "sys.service")

#define SYS_FEATURE_INIT(func) LAYER_INITCALL_DEF(func, sys_feature, "sys.feature")

```

```c
static void Init(void)
{
    PubSubFeature *feature = &g_broadcastFeature;
    feature->relations.topic = -1;
    feature->relations.callbacks.consumer = NULL;
    UtilsListInit(&feature->relations.callbacks.node);
    UtilsListInit(&feature->relations.node);
    feature->mutex = MUTEX_InitValue();
    SAMGR_GetInstance()->RegisterFeature(BROADCAST_SERVICE, (Feature *)feature);

    PubSubImplement *apiEntry = BCE_CreateInstance((Feature *)feature);
    SAMGR_GetInstance()->RegisterFeatureApi(BROADCAST_SERVICE, PUB_SUB_FEATURE, GET_IUNKNOWN(*apiEntry));
}
SYS_FEATURE_INIT(Init);

```
其实现与Linux的initcall相似
>Linux内核使用了一个称为initcall的机制，该机制允许在内核启动时以特定的顺序执行初始化函数。initcall背后的基本思想是将一系列的初始化函数地址存储在特定的内存区段，然后在内核启动时按顺序执行这些函数。
>这个机制依赖GCC的特性，特别是\_\_attribute\_\_((section(...))),\_\_attribute\_\_((section(...)))将函数地址在链接时放入zInit代码段，而后由其它函数遍历这些段内的函数

#### BOOT_SYS阶段
```plantuml
    !theme plain
    |OHOS_SystemInit|
    :OHOS_SystemInit();
    group 注册系统服务
    :SYS_INIT(service)遍历所有SYS_SERVICE_INIT宏修饰的Init();
    group 注册Bootstrap
    :注册Bootstrap;
    :调用SAMGR_GETInstance()获取单例对象g_samrImpl;
    :g_samrImpl懒加载;
    :调用g_samrImpl的RegisterService进行注册;
    :为BootStrap(继承自Servcie)创建一个ServiceImpl进行管理;
    :将该ServiceImpl VECTOR_Add()到g_samrImpl的services字段中;
    end group
    :注册Broadcast;
    :注册Hiview;
    end group
    group 注册系统服务的特性
    :SYS_INIT(feature)遍历所有SYS_Feature_INIT宏修饰的Init();
    group 注册Broadcast的特性PUB_SUB_FEAFURE
    :调用g_samrImpl的RegisterFeature()注册特性;
    :为PUB_SUB_FEAFURE创建FeatureImpl进行管理;
    :将该FeatureImpl VECTOR_Add()到serviceImpl的features字段中;
    :创建特性接口g_pubSubImplement
    关联g_pubSubImplement与PubSubFeature对象;
    :调用RegisterFeatureApi注册特性的默认接口
    使serviceImpl->defaultApi指向g_pubSubImplement.iUnknown;
    end group
    end group
    group 调用SAMGR_Bootstrap()启动服务和特性
    :samgr创建Vector initServices记录未初始化的服务
    （根据serviceImpl的inited字段是否为SVC_INIT);
    group 调用InitializeAllServices()初始化所有未初始化服务
    :调用AddTaskPool()为服务创建任务池和消息队列;
    :调用InitializeSingleService()向消息队列发送MSG_DIRECT的初始化消息;
    :调用SAMGR_StartTaskPool()启动各服务线程
    各服务循环监听各自消息队列;
    end group
    end group
```

#### InitializeSingleService启动特性

```plantuml
    !theme plain
    Participant Samgr
    Queue MQueue
    Samgr -> MQueue : 发送type为MSG_DIRECT\n且handle为HandleInitRequest()的request
    MQueue --> Samgr
    MQueue -> Broadcast : 循环监听消息队列，获取request
    activate Broadcast
    Broadcast -> BroadcastFeature : 调用OnInitialize()进行初始化
    activate BroadcastFeature
    BroadcastFeature --> Broadcast : 
    deactivate BroadcastFeature
    Broadcast --> MQueue
    deactivate Broadcast

```

#### BOOT_APP阶段
所有由SYS_SERVICE_INIT()和SYS_FEATURE_INIT()修饰的服务与特性启动完毕,接下来启动由SYSEX_SERVICE_INIT,SYSEX_FEATURE_INIT,APP_SERVICE_INIT,APP_FEATURE_INIT修饰的服务与特性

```plantuml
    !theme plain
    Participant Samgr
    Queue MQueue
    Samgr -> MQueue : 发送BOOT_SYS_COMPLETED的request
    MQueue --> Samgr
    MQueue -> Bootstrap : 循环监听消息队列，获取request
    activate Bootstrap
    Bootstrap -> "由SYSEX\APP_SERVICE\FEATURE_INIT宏\n修饰的服务与特性" : INIT_APP_CALL()
    activate "由SYSEX\APP_SERVICE\FEATURE_INIT宏\n修饰的服务与特性"
    "由SYSEX\APP_SERVICE\FEATURE_INIT宏\n修饰的服务与特性" --> Bootstrap
    Bootstrap --> MQueue

```

#### BOOT_DYNAMIC阶段
所有由SYSEX_SERVICE_INIT,SYSEX_FEATURE_INIT,APP_SERVICE_INIT,APP_FEATURE_INIT修饰的服务与特性启动完毕

### 消息队列

```plantuml
    !theme plain
    hide empty members
    struct TaskPool {
        queueId : MQueueId
        stackSize : uint16
        priority : uint8
        size : uint8
        top : uint8
        ref : int8
        tasks : ThreadId
        SAMGR_CreateFixedTaskPool()
        SAMGR_StartTaskPool()
        TaskEntry()
        ProcResponse()
        ProcDirectRequest()
        ProcRequest()
    }

    struct Exchange{
        id : Identity
        request : Request
        response : Response
        type : short
        handler : Handler
        sharedRef : uint32
    }

    enum ExchangeType {
        MSG_EXIT //终止消息处理任务
        MSG_NON //不需要响应的请求消息，接收者调用服务的MessageHandle或特性的OnMessage
        MSG_CON  //需要响应的请求消息，接收者调用服务的MessageHandle或特性的OnMessage
        MSG_ACK //响应消息
        MSG_SYNC //同步消息
        MSG_DIRECT  //直接请求消息，无需响应，消息接收者调用handler
    }

    struct Identity {
        serviceId : int16
        featureId : int16
        queueId : MQueueId
    }

    struct Request {
        msgId : int16
        len : int16
        data : void *
        msgValue : uint32
    }

    struct Response {
        data : void *
        len : int16
    }

    TaskPool --> Exchange : 消息队列所交换的信息
    Exchange --> Identity 
    Exchange --> ExchangeType
    Exchange --> Request
    Exchange --> Response

    note left of TaskPool::SAMGR_CreateFixedTaskPool
    为每一个服务创建消息队列和任务池
    end note
    note left of TaskPool::SAMGR_StartTaskPool
    创建并运行监控线程,通过以下代码塞一个TaskEntry()，线程启动后运行TaskEntry()
    register ThreadId threadId = (ThreadId)THREAD_Create(TaskEntry, pool->queueId, &attr);
    end note
    note left of TaskPool::TaskEntry
    循环监听消息队列
    调用ProcResponse,ProcDirectRequest,ProcRequest等进行消息处理
    end note
    note left of TaskPool::ProcResponse
    处理MSG_ACK响应消息
    调用Exchange的handler函数处理消息
    end note
    note left of TaskPool::ProcDirectRequest
    处理MSG_DIRECT响应消息
    调用Exchange的handler函数处理消息
    end note
    note left of TaskPool::ProcRequest
    处理MSG_NON/MSG_CON响应消息
    调用服务或特性的MessageHandle()函数处理消息
    end note

    note left of Exchange::Identity
    消息接收者的身份信息
    end note
    note left of Exchange::Request
    请求消息的主题
    end note
    note left of Exchange::Response
    响应消息的主题
    end note
    note left of Exchange::type
    消息的类型
    end note
    note left of Exchange::handler
    请求消息的消息处理函数或响应消息的异步响应回调函数
    end note
    note left of Exchange::sharedRef
    用以计数，指明request或response的data的引用次数
    end note

    note right of Identity::queueId
    消息发送者的消息队列ID，消息接收者向该ID发送响应消息
    end note

    note bottom of Response
    data指向共享内存的空间首地址
    len为数据长度
    end note

    note bottom of Request
    两类数据传输方式：
    一种共享内存（利用data,len字段，data指向共享内存的空间首地址，len为data的长度）
    一种利用msgID,msgValue（例如Bootstrap，Hiview，broadcast的pub_sub_feature）
    end note
```

消息队列的底层POSIX,CMSIS两类实现

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

```plantUML
    !theme plain
    hide empty members
    Struct MQueue{
        QUEUE_Create()
        QUEUE_Put()
        QUEUE_Pop()
        QUEUE_Destroy()
    }
```

#### CMSIS实现

QUEUE_Create调用链：
```plantuml
    !theme plain
    QUEUE_Create -> KAL内核抽象层 :  调用osMessageQueueNew(),\n设置其size,count,name等属性（但其他属性UNUSED掉了)
    activate KAL内核抽象层
    KAL内核抽象层 -> los_queue : 调用LOS_QueueCreate()创建消息队列，\n其由LosQueueCB管理
    activate los_queue
    los_queue -> los_memory : 调用LOS_MemAlloc()在指定内存池为\n消息队列分配内存
    activate los_memory
    los_memory --> los_queue : 分配的内存指针
    deactivate los_memory
    los_queue --> KAL内核抽象层 : 分配成功或失败的信息
    deactivate los_queue
    KAL内核抽象层 --> QUEUE_Create : 如果成功返回message queue ID，如果失败返回null
    deactivate KAL内核抽象层 

```

```code
typedef struct 
{
    UINT8       *queueHandle;                    /* 队列指针 */
    UINT8       queueState;                      /* 队列状态 */
    UINT8       queueMemType;                    /* 创建队列时内存分配的方式 */
    UINT16      queueLen;                        /* 队列中消息节点个数，即队列长度 */
    UINT16      queueSize;                       /* 消息节点大小 */
    UINT32      queueID;                         /* 队列ID */
    UINT16      queueHead;                       /* 消息头节点位置（数组下标）*/
    UINT16      queueTail;                       /* 消息尾节点位置（数组下标）*/
    UINT16      readWriteableCnt[OS_QUEUE_N_RW]; /* 数组下标0的元素表示队列中可读消息数，                              
                                                    数组下标1的元素表示队列中可写消息数 */
    LOS_DL_LIST readWriteList[OS_QUEUE_N_RW];    /* 读取或写入消息的任务等待链表， 
                                                    下标0：读取链表，下标1：写入链表 */
    LOS_DL_LIST memList;                         /* CMSIS-RTOS中的MailBox模块使用的内存块链表 */
} LosQueueCB;

```

QUEUE_Put调用链：

```plantuml
!theme plain
participant "Client" as Client
participant "QUEUE_Put" as QueuePut
participant "osMessageQueuePut" as OsMsgQueuePut
participant "LOS_QueueWriteCopy" as LOSQueueWriteCopy
participant "OsQueueOperate" as OsQueueOperate

Client -> QueuePut : 设置队列ID, 元素, 优先级, 允许的超时时间\n源代码中所有该调用皆为DONT_WAIT
activate QueuePut

QueuePut -> OsMsgQueuePut : 设置队列ID, 元素, 优先级, 计算后的超时
activate OsMsgQueuePut

OsMsgQueuePut -> LOSQueueWriteCopy : 设置队列ID, 元素, 缓冲区大小, 超时时间
activate LOSQueueWriteCopy

LOSQueueWriteCopy -> OsQueueOperate : 设置队列ID, 操作类型, 缓冲区地址, 缓冲区大小, 超时时间\n操作类型为写队列尾部且为非指针操作，这意味着会调用memcpy_s复制整个内存
activate OsQueueOperate

OsQueueOperate --> LOSQueueWriteCopy : 返回状态
deactivate OsQueueOperate

LOSQueueWriteCopy --> OsMsgQueuePut : 返回状态
deactivate LOSQueueWriteCopy

alt 如果返回LOS_OK
    OsMsgQueuePut --> QueuePut : 返回osOK
    deactivate OsMsgQueuePut
    QueuePut --> Client : 返回EC_SUCCESS
else 如果发生错误
    OsMsgQueuePut --> QueuePut : 返回错误状态
    deactivate OsMsgQueuePut
    QueuePut --> Client : 返回EC_BUSBUSY
end

deactivate QueuePut

```



#### POSIX实现

生产者-消费者模型

>生产者消费者模式是一种常见的并发设计模式，用于处理在多线程环境下不同进程或线程间的数据共享和同步。在这个模式中，生产者负责生成数据，消费者负责处理数据。关键点在于生产者和消费者的处理速度可能不同，因此需要一种机制来平衡它们之间的速率差异。
>
>主要组件
>生产者（Producer）：负责生成数据，并将其放入缓冲区。生产者可以是一个或多个线程。
>
>消费者（Consumer）：从缓冲区中取出数据并处理它。消费者也可以是一个或多个线程。
>
>缓冲区（Buffer）：临时存储生产者生成的数据，等待消费者处理。缓冲区可以是有限的（有最大容量限制）或无限的。
>
>关键特点
>同步机制：由于生产者和消费者通常以不同的速率运行，因此需要同步机制来协调它们的工作。这通常通过锁（如互斥锁）和信号（如条件变量）实现。
>
>阻塞和唤醒：当缓冲区为空时，消费者线程可能会被阻塞，直到有新数据加入缓冲区。类似地，当缓冲区满时，生产者线程可能会被阻塞，直到消费者取走一些数据。

```plantUML
!theme plain
hide empty members

struct LockFreeBlockQueue {
    wMutex : pthread_mutex_t
    rMutex : pthread_mutex_t
    cond : pthread_cond_t
    queue : LockFreeQueue
}
struct LockFreeQueue {
    write: uint32
    read: uint32
    itemSize: uint32
    totalSize: uint32
    buffer: uint8
}

LockFreeBlockQueue -> LockFreeQueue

note left of LockFreeBlockQueue::wMutex
POSIX 线程（pthreads）库的互斥锁
end note

note left of LockFreeBlockQueue::cond
POSIX 线程（pthreads）库的条件变量，
支持Wait，Signal，Broadcast操作
end note

```

```plantuml

!theme plain

start

:接收参数queueId, element, pri, timeout;
if (参数检查) then (失败)
  :返回 EC_INVALID;
  stop
else (成功)
endif

partition "QUEUE_Put" {
  :锁定 wMutex;
  :执行 LFQUE_Push;
  :解锁 wMutex;
  :锁定 rMutex;
  :发出条件变量信号;
  :解锁 rMutex;
  :返回操作结果;
}
stop

```

```plantuml

!theme plain

start

partition "QUEUE_Pop" {
  :尝试 LFQUE_Pop;
  if (操作成功?) then (是)
    :返回 EC_SUCCESS;
    stop
  else (否)
  endif

  :锁定 rMutex;
  while (LFQUE_Pop 不成功?) is (是)
    :等待条件变量;
  endwhile
  :解锁 rMutex;
  :返回 EC_SUCCESS;
}

stop

```

## 进程间通信
```plantuml
!theme plain
hide empty members

class RemoteRegister {
    - mtx : MutexId
    - clients : Vector
    - endpoint : Endpoint
}

class Endpoint {
    - name : char
    - routers : Vector
    - boss : ThreadId
    - deadId : uint32
    - running : int
    - registerEP : RegisterEndpoint
    - bucket : TokenBucket
    - context : IpcContext
    - identity : SvcIdentity
}

class SvcIdentity {
    - handle : uint32_t
    - token : uint32_t
    - cookie : uint32_t
    - ipcContext : IpcContext
}

class Router {
    - policyNum : uint32
    - saName : SaName
    - identity : Identity
    - proxy : IServerProxy
    - policy : PolicyTrans
}

class IpcContext{
    - fd : int //文件描述符
    - mmapSize : size_t //Memory Map Size
    - mmapAddr : void* //Memory Map Address
}

RemoteRegister --* Endpoint
Endpoint --*  SvcIdentity
Endpoint --*  IpcContext
Endpoint --*  Router : routers.data[x]

note left of SvcIdentity
本Endpooint的身份信息
end note
note left of Endpoint
每个进程对应一个Endpoint，
用以维护boss线程，其用以IPC
end note
note right of Router
某个进程对外提供服务或特性的服务单元
end note
note right of RemoteRegister
某个进程维护一个g_remoteRegister全局变量
end note

note right of Endpoint::name
客户端Endpoint的名字为ipc client,samgr Endpoint的名字为samgr
end note

note right of Endpoint::boss 
Endpoint用于进程间通信的线程的句柄
end note

note right of Endpoint::RegisterEndpoint
函数指针指向Enpoint向服务管理者注册自己的注册函数，
客户端Endpoint为RegisterRemoteEndpoint(),
samgr Endpoint为RegisterSamgrEndpoint()
end note




```
### IPC概要

```plantuml
!theme plain
hide empty members

class Endpoint {
}

class SamgrEP extends Endpoint {
}

class IpcClientA extends Endpoint {
}

class IpcClientB extends Endpoint {
}

note left of SamgrEP
全局唯一
由SamgrServer g_server维护
end note
```


```plantuml
!theme plain
hide empty members

class SamgrEP  {
}

class IpcClientA {
}

class IpcClientB {
}

IpcClientA -down-> SamgrEP : 注册boss的Tid\n注册Router列表
IpcClientB -down-> SamgrEP : 查询IpcClientA的handle，Router的token\n而后进行IPC

```

### Endpoint注册流程
```plantuml
!theme plain
    :服务或特性初始化时其消息队列会收到
    MSG_DIRECT的消息，该消息由HandleInitRequest()处理;
    :调用DEFAULT_Initialize(serviceImpl)初始化各服务和特性;
    :调用服务的生命周期函数Initialize()、OnInitialize()进行初始化;
    :调用SAMGR_RegisterServiceApi()进行服务接口注册;
    :InitializeRegistry()初始化RemoteRegister;
    group 初始化RemoteRegister
    :调用MUTEX_InitValue()创建互斥量;
    :调用VECTOR_Make()创建空Vector;
    :调用SAMGR_CreateEndpoint()创建名为"ipc client"的Endpoint;
    group 初始化Endpoint
    :调用OpenLiteIpc()初始化context该函数，
    包含打开设备驱动文件(如/dev/binder）,创建内存映射两步;
    :初始化其它成员变量，boss、identity的成员变量置为Null或INVALID_INDEX，
    它们将在后续步骤中进行真正的初始化;
    end group
    end group

    :调用SAMGR_AddRouter()注册Router;
    group 注册Router




```

>### 打开设备驱动文件
>设备驱动文件：在类 Unix 系统（如 Linux）中，设备驱动文件通常位于 /dev 目录下。它们是特殊的文件，提供了用户空间程序与硬件设备或虚拟设备交互的接口。通过这些文件，程序可以读取或写入设备，或者执行特定的控制操作。
>
>打开过程：打开设备驱动文件通常涉及到调用如 open 的系统调用。这个调用需要设备文件的路径和所需的访问模式（如只读、只写或读写）。成功的 open 调用返回一个文件描述符（fd），这是一个整数值，用于后续的读写或控制操作。
>
>用途：这个过程通常用于与特定的硬件设备通信，比如读取传感器数据，控制图形显示设备，或者与网络接口交互。在 IPC (进程间通信) 的上下文中，这可能涉及到打开一个代表某种通信机制（如管道、消息队列）的特殊文件。
>
>### 创建内存映射
>内存映射（Memory Mapping）：内存映射是一种内存管理技术，它允许程序将一部分磁盘文件映射到其地址空间。通过这种方式，文件内容可以像访问普通内存一样直接访问，这样可以提高文件操作的效率，并且支持进程间共享内存。
>
>创建过程：在类 Unix 系统中，这通常通过 mmap 系统调用实现。mmap 调用需要文件描述符、映射区域的大小、期望的保护级别（如是否可读写）、映射类型（共享或私有）等参数。成功调用 mmap 后，它返回一个指向映射区域开始位置的指针。
>
>用途：内存映射常用于高效文件访问，以及在多个进程之间共享内存。在 IPC 上下文中，内存映射可以用来创建一个共享的内存区域，让多个进程可以读写同一块内存，从而实现数据共享和通信。

### IpcClient与samgr EP的IPC

### IpcClient之间的IPC