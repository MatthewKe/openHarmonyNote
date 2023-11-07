
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

CMSIS实现调用osMessageQueueNew, osMessageQueuePut, osMessageQueueGet, osMessageQueueDelete等API

```code
MQueueId QUEUE_Create(const char *name, int size, int count)
{
    osMessageQueueAttr_t queueAttr = {name, 0, NULL, 0, NULL, 0};
    return (MQueueId)osMessageQueueNew(count, size, &queueAttr);
}

int QUEUE_Put(MQueueId queueId, const void *element, uint8 pri, int timeout)
{
    uint32_t waitTime = (timeout <= 0) ? 0 : (uint32_t)timeout;
    osStatus_t ret = osMessageQueuePut(queueId, element, pri, waitTime);
    if (ret != osOK) {
        return EC_BUSBUSY;
    }
    return EC_SUCCESS;
}

int QUEUE_Pop(MQueueId queueId, void *element, uint8 *pri, int timeout)
{
    uint32_t waitTime = (timeout <= 0) ? osWaitForever : (uint32_t)timeout;
    osStatus_t evt = osMessageQueueGet(queueId, element, pri, waitTime);
    if (evt != osOK) {
        return EC_BUSBUSY;
    }
    return EC_SUCCESS;
}

int QUEUE_Destroy(MQueueId queueId)
{
    osStatus_t evt = osMessageQueueDelete(queueId);
    if (evt != osOK) {
        return EC_FAILURE;
    }
    return EC_SUCCESS;
}
```

kernel/liteos_m/kal/cmsis/cmsis_liteos2.c

```code
osMessageQueueId_t osMessageQueueNew(uint32_t msg_count, uint32_t msg_size, const osMessageQueueAttr_t *attr)
{
    UINT32 uwQueueID;
    UINT32 uwRet;
    UNUSED(attr);
    osMessageQueueId_t handle;

    if (0 == msg_count || 0 == msg_size || OS_INT_ACTIVE) {
        return (osMessageQueueId_t)NULL;
    }

    uwRet = LOS_QueueCreate((char *)NULL, (UINT16)msg_count, &uwQueueID, 0, (UINT16)msg_size);
    if (uwRet == LOS_OK) {
        handle = (osMessageQueueId_t)(GET_QUEUE_HANDLE(uwQueueID));
    } else {
        handle = (osMessageQueueId_t)NULL;
    }

    return handle;
}


osStatus_t osMessageQueuePut(osMessageQueueId_t mq_id, const void *msg_ptr, uint8_t msg_prio, uint32_t timeout)
{
    UNUSED(msg_prio);
    UINT32 uwRet;
    UINT32 uwBufferSize;
    LosQueueCB *pstQueue = (LosQueueCB *)mq_id;

    if (pstQueue == NULL || msg_ptr == NULL || ((OS_INT_ACTIVE) && (0 != timeout))) {
        return osErrorParameter;
    }
    if (pstQueue->queueSize < sizeof(UINT32)) {
        return osErrorParameter;
    }
    uwBufferSize = (UINT32)(pstQueue->queueSize - sizeof(UINT32));
    uwRet = LOS_QueueWriteCopy((UINT32)pstQueue->queueID, (void *)msg_ptr, uwBufferSize, timeout);
    if (uwRet == LOS_OK) {
        return osOK;
    } else if (uwRet == LOS_ERRNO_QUEUE_INVALID || uwRet == LOS_ERRNO_QUEUE_NOT_CREATE) {
        return osErrorParameter;
    } else if (uwRet == LOS_ERRNO_QUEUE_TIMEOUT) {
        return osErrorTimeout;
    } else {
        return osErrorResource;
    }
}


osStatus_t osMessageQueueGet(osMessageQueueId_t mq_id, void *msg_ptr, uint8_t *msg_prio, uint32_t timeout)
{
    UNUSED(msg_prio);
    UINT32 uwRet;
    UINT32 uwBufferSize;
    LosQueueCB *pstQueue = (LosQueueCB *)mq_id;

    if (pstQueue == NULL || msg_ptr == NULL || ((OS_INT_ACTIVE) && (0 != timeout))) {
        return osErrorParameter;
    }

    uwBufferSize = (UINT32)(pstQueue->queueSize - sizeof(UINT32));
    uwRet = LOS_QueueReadCopy((UINT32)pstQueue->queueID, msg_ptr, &uwBufferSize, timeout);
    if (uwRet == LOS_OK) {
        return osOK;
    } else if (uwRet == LOS_ERRNO_QUEUE_INVALID || uwRet == LOS_ERRNO_QUEUE_NOT_CREATE) {
        return osErrorParameter;
    } else if (uwRet == LOS_ERRNO_QUEUE_TIMEOUT) {
        return osErrorTimeout;
    } else {
        return osErrorResource;
    }
}
```

kernel/liteos_m/kernel/src/los_queue.c
```code


/**************************************************************************
 Function    : OsQueueInit
 Description : queue initial
 Input       : None
 Output      : None
 Return      : LOS_OK on success or error code on failure
**************************************************************************/
LITE_OS_SEC_TEXT_INIT UINT32 OsQueueInit(VOID)
{
    LosQueueCB *queueNode = NULL;
    UINT16 index;

    if (LOSCFG_BASE_IPC_QUEUE_LIMIT == 0) {
        return LOS_ERRNO_QUEUE_MAXNUM_ZERO;
    }

    g_allQueue = (LosQueueCB *)LOS_MemAlloc(m_aucSysMem0, LOSCFG_BASE_IPC_QUEUE_LIMIT * sizeof(LosQueueCB));
    if (g_allQueue == NULL) {
        return LOS_ERRNO_QUEUE_NO_MEMORY;
    }

    (VOID)memset_s(g_allQueue, LOSCFG_BASE_IPC_QUEUE_LIMIT * sizeof(LosQueueCB),
                   0, LOSCFG_BASE_IPC_QUEUE_LIMIT * sizeof(LosQueueCB));

    LOS_ListInit(&g_freeQueueList);
    for (index = 0; index < LOSCFG_BASE_IPC_QUEUE_LIMIT; index++) {
        queueNode = ((LosQueueCB *)g_allQueue) + index;
        queueNode->queueID = index;
        LOS_ListTailInsert(&g_freeQueueList, &queueNode->readWriteList[OS_QUEUE_WRITE]);
    }

    return LOS_OK;
}

/*****************************************************************************
 Function    : LOS_QueueCreate
 Description : Create a queue
 Input       : queueName  --- Queue name, less than 4 characters
             : len        --- Queue length
             : flags      --- Queue type, FIFO or PRIO
             : maxMsgSize --- Maximum message size in byte
 Output      : queueID    --- Queue ID
 Return      : LOS_OK on success or error code on failure
 *****************************************************************************/
LITE_OS_SEC_TEXT_INIT UINT32 LOS_QueueCreate(CHAR *queueName,
                                             UINT16 len,
                                             UINT32 *queueID,
                                             UINT32 flags,
                                             UINT16 maxMsgSize)
{
    LosQueueCB *queueCB = NULL;
    UINT32 intSave;
    LOS_DL_LIST *unusedQueue = NULL;
    UINT8 *queue = NULL;
    UINT16 msgSize;

    (VOID)queueName;
    (VOID)flags;

    if (queueID == NULL) {
        return LOS_ERRNO_QUEUE_CREAT_PTR_NULL;
    }

    if (maxMsgSize > (OS_NULL_SHORT - sizeof(UINT32))) {
        return LOS_ERRNO_QUEUE_SIZE_TOO_BIG;
    }

    if ((len == 0) || (maxMsgSize == 0)) {
        return LOS_ERRNO_QUEUE_PARA_ISZERO;
    }
    msgSize = maxMsgSize + sizeof(UINT32);

    /* Memory allocation is time-consuming, to shorten the time of disable interrupt,
       move the memory allocation to here. */
    queue = (UINT8 *)LOS_MemAlloc(m_aucSysMem0, len * msgSize);
    if (queue == NULL) {
        return LOS_ERRNO_QUEUE_CREATE_NO_MEMORY;
    }

    intSave = LOS_IntLock();
    if (LOS_ListEmpty(&g_freeQueueList)) {
        LOS_IntRestore(intSave);
        (VOID)LOS_MemFree(m_aucSysMem0, queue);
        return LOS_ERRNO_QUEUE_CB_UNAVAILABLE;
    }

    unusedQueue = LOS_DL_LIST_FIRST(&(g_freeQueueList));
    LOS_ListDelete(unusedQueue);
    queueCB = (GET_QUEUE_LIST(unusedQueue));
    queueCB->queueLen = len;
    queueCB->queueSize = msgSize;
    queueCB->queue = queue;
    queueCB->queueState = OS_QUEUE_INUSED;
    queueCB->readWriteableCnt[OS_QUEUE_READ] = 0;
    queueCB->readWriteableCnt[OS_QUEUE_WRITE] = len;
    queueCB->queueHead = 0;
    queueCB->queueTail = 0;
    LOS_ListInit(&queueCB->readWriteList[OS_QUEUE_READ]);
    LOS_ListInit(&queueCB->readWriteList[OS_QUEUE_WRITE]);
    LOS_ListInit(&queueCB->memList);
    LOS_IntRestore(intSave);

    *queueID = queueCB->queueID;

    OsHookCall(LOS_HOOK_TYPE_QUEUE_CREATE, queueCB);

    return LOS_OK;
}

```


#### POSIX实现

```code
struct LockFreeBlockQueue {
    pthread_mutex_t wMutex;
    pthread_mutex_t rMutex;
    pthread_cond_t cond;
    LockFreeQueue *queue;
};
struct LockFreeQueue {
    uint32 write;
    uint32 read;
    uint32 itemSize;
    uint32 totalSize;
    uint8 buffer[0];
};
```
