# Internal
Internal中包含了EBus的底层基础组件。
## StoragePolicies
此模块提供了Address和Handle的保存机制。主要区分需要排序和不需要排序两种类型。
### AddressStoragePolicy
用来决定如何保存Address（ID）和HandlerStorage的映射，假如此Ebus不使用AddressPolicy，那么这个类不会被使用。
默认的定义为：
```cpp
template <typename Traits, typename HandlerStorage, EBusAddressPolicy = Traits::AddressPolicy>
struct AddressStoragePolicy;
```
它需要3个模板参数：
Traits：从EBus的Traits中得到，决定了id类型，内存分配器类型，是否排序，如果排序的话如何比较id。
HandlerStorage：保存的handler的数据类型。
EBusAddressPolicy：只有AddressPolicy的EBus才能实例化这个模板。
它是一个空的结构体，具体的功能由模板的特化来实现。
下面为两个模板特化：

其中的不需要排序的AddressStoragePolicy，第3个模板参数提供的是EBusAddressPolicy::ById
```cpp
// Unordered
template <typename Traits, typename HandlerStorage>
struct AddressStoragePolicy<Traits, HandlerStorage, EBusAddressPolicy::ById>
{
    ...
}
```
其中定义的StorageType为哈希表：
```cpp
struct AddressStoragePolicy<Traits, HandlerStorage, EBusAddressPolicy::ById>
{
    ...
public:
    struct StorageType
        : public AZStd::hash_table<hash_table_traits>
    {
        ...
    }
}
```
这个哈希表的key是Traits::BusIdType，value是HandlerStorage。

需要排序的AddressStoragePolicy，第三个模板参数提供的是EBusAddressPolicy::ByIdAndOrdered
```cpp
// Ordered
template <typename Traits, typename HandlerStorage>
struct AddressStoragePolicy<Traits, HandlerStorage, EBusAddressPolicy::ByIdAndOrdered>
{
    ...
}
```
其中定义的StorageType为红黑树：
```cpp
struct AddressStoragePolicy<Traits, HandlerStorage, EBusAddressPolicy::ById>
{
    ...
public:
    struct StorageType
        : public AZStd::rbtree<rbtree_traits>
    {
        ...
    }
}
```
同样key是Traits::BusIdType，value是HandlerStorage。比较函数的类型从Traits中获取。
当EBus的Address指定为id时，表示此EBus限定为只有连接至id的handler才会接收到事件，所以需要一个容器将id与所有handler关联起来，最合适的就是哈希表和红黑树。
### HandlerStoragePolicy
用来决定如何保存handler列表，由于一个EBus可能只连接一个handler，也可能连接多个handler，当连接多个Handler时，要决定如何保存它们（如果只是一个handle的话直接保存它就行了）。假设EBus是使用AddressPolicy的，那么这个类会被提供为上面AddressStoragePolicy的第二个模板参数。
```cpp
template <typename Interface, typename Traits, typename Handler, EBusHandlerPolicy = Traits::HandlerPolicy>
struct HandlerStoragePolicy;
```
与上面的AddressStoragePolicy一样，它依靠模板的特化来实现具体的功能。
4个模板参数：
Interface，Traits：都是从EBus中获取，含义与EBus中的相同。
Handler：handler的数据类型。
EBusHandlerPolicy：只有EBu使用了HandlerPolicy::Multiplexxx（即允许多个Handler连接到EBus）才会使用这个类。

不需要排序的HandlerStoragePolicy，第4个模板参数提供EBusHandlerPolicy::Multiple。
```cpp
// Unordered
template <typename Interface, typename Traits, typename Handler>
struct HandlerStoragePolicy<Interface, Traits, Handler, EBusHandlerPolicy::Multiple>
{
public:
    struct StorageType
          : public AZStd::intrusive_list<Handler, AZStd::list_base_hook<Handler>>
    {
        using base_type = AZStd::intrusive_list<Handler, AZStd::list_base_hook<Handler>>;
        void insert(Handler& elem)
        {
            base_type::push_front(elem);
        }
    };
};
```
其中的StorageType是一个list。这里的intrusive_list是O3DE中自定义的一种类型，目前就将其视为list，

需要排序的HandlerStoragePolicy，第4个模板参数提供EBusHandlerPolicy::MultipleAndOrdered，
```cpp
// Ordered
template <typename Interface, typename Traits, typename Handler>
struct HandlerStoragePolicy<Interface, Traits, Handler, EBusHandlerPolicy::MultipleAndOrdered>
{
private:
    using Compare = HandlerCompare<Interface, Traits>;
public:
    using StorageType = AZStd::intrusive_multiset<Handler, AZStd::intrusive_multiset_base_hook<Handler>, Compare>;
};
```
此时StorageType是一个multiset，比较函数从Traits中获取。
### HandlerStorageNode
这个模板类用于定义HandlerStoragePolicy中保存的数据节点类型，在HandlerStoragePolicy模板中作为Handler参数提供的类应该是从HandlerStorageNode类派生的。
```cpp
template <typename Handler, EBusHandlerPolicy>
struct HandlerStorageNode
{
};
template <typename Handler>
struct HandlerStorageNode<Handler, EBusHandlerPolicy::Multiple>
: public AZStd::intrusive_list_node<Handler>
{
};
template <typename Handler>
struct HandlerStorageNode<Handler, EBusHandlerPolicy::MultipleAndOrdered>
: public AZStd::intrusive_multiset_node<Handler>
{
};
```
目前看起来这里应该是为了保证节点是intrusive_list_node或者intrusive_multiset_node，以便于适配O3DE中自定义的intrusive容器。
显然只有Multiple Handler才会使用这种Node类型，如果只有一个Handler的话直接保存指针就行了。
## Handles
Handle在EBus中作为事件的监听者，它可以连接到EBus系统，当事件触发时Handle响应此事件，并且可以在不需要时断开与EBus的连接。
### HandlerNode 
HandlerNode定义了Handler的储存类型，它继承自StoragePolicies中的HandlerStorageNode。HandlerNode持有Interface的反向引用，并且拥有与Interface\*相同的行为。

需要Address（ID）的HandlerNode
```cpp
template <typename Interface, typename Traits, typename HandlerHolder, bool /*hasId*/ = Traits::AddressPolicy != AZ::EBusAddressPolicy::Single>
class HandlerNode
: public HandlerStorageNode<HandlerNode<Interface, Traits, HandlerHolder, true>, Traits::HandlerPolicy>
{
    ...
}
```
从代码中可以看出，它继承自HandlerStorageNode，为HandlerStorageNode提供的模板参数为HandlerNode（自身类型）和Traits::HandlerPolicy（会是EBusHandlerPolicy::Multiple或EBusHandlerPolicy::MultipleAndOrdered）。
该类的第3个模板参数是HandlerHolder，表示Handler的持有者，后续介绍。
第4个模板参数表示当前HandlerNode所在的EBus是不是使用了Address的，这里是true，没有使用了Address（Traits::AddressPolicy != AZ::EBusAddressPolicy::Single）。
它的数据成员如下：
```cpp
Interface* m_interface = nullptr;
// This is stored as an intrusive_ptr so that the holder doesn't get destroyed while handlers are still inside it.
AZStd::intrusive_ptr<HandlerHolder> m_holder;
```
保存了Interface的指针和HandlerHolder的指针。
上面说过，HandlerNode应该有着Interface\*一样的行为，所有针对HandlerNode进行一些操作符的重载，
```cpp
typename Traits::BusIdType GetBusId() const
{
    return m_holder->m_busId;
}

operator Interface*() const
{
    return m_interface;
}

Interface* operator->() const
{
    return m_interface;
}

// Allows resetting of the pointer
HandlerNode& operator=(Interface* inst)
{
    m_interface = inst;
    // If assigning to nullptr, do a full reset
    if (inst == nullptr)
    {
        m_holder.reset();
    }
    return *this;
}
```
可以使用\*和->操作符，也可以使用Interface\*为其赋值。

不需要Address（ID）的HandlerNode
```cpp
template <typename Interface, typename Traits, typename HandlerHolder>
class HandlerNode<Interface, Traits, HandlerHolder, false>
: public HandlerStorageNode<HandlerNode<Interface, Traits, HandlerHolder, false>, Traits::HandlerPolicy>
{
    ...
}
```
第4个模板参数提供了false，既然没有使用Address，那么自然就不需要保存HandlerHolder的指针，所以它的数据成员只有一个：
```cpp
Interface* m_interface = nullptr;
```
其余的操作符重载与需要Address（ID）的HandlerNode相似。
### NonIdHandler
所有的Handler都继承自Interface，不同种类的Handler应用在不同的EBus上。
这种Handle用于single address的EBus，也就是说它监听的事件不需要绑定Address（ID），所有Handle都可以监听到。
```cpp
template <typename Interface, typename Traits, typename ContainerType>
class NonIdHandler: public Interface
{
    ...
}       
```
第3个模板参数ContainerType是一个EBusContainer类型，是保存EBus数据的类。
数据成员：
```cpp
// Must be a member and not a base type so that Interface may be an incomplete type.
typename ContainerType::HandlerNode m_node;
```
只有一个HandlerNode，尽管它是从ContainerType中取出的类型，但是它的定义其实就是HandlerNode类。
省略掉常用的构造、析构、赋值成员函数，Handler中主要的函数有3个，
```cpp
    void BusConnect();
    void BusDisconnect();
    bool BusIsConnected() const
    {
        return static_cast<Interface*>(m_node) != nullptr;
    }
```
这3个函数控制了Handler的连接状态，这里BusIsConnected取决于m_node是否为空。
> [!NOTE] Note
> 这里直接将m_node转换为了Interface\*，使用的是上面HandlerNode重载的Interface\*操作符。

BusConnect与BusDisconnect的定义如下：
```cpp
        // NonIdHandler
        template <typename Interface, typename Traits, typename ContainerType>
        void NonIdHandler<Interface, Traits, ContainerType>::BusConnect()
        {
            // context可以理解为Ebus系统调用Handler时的运行环境
            typename BusType::Context& context = BusType::GetOrCreateContext();
            // 保证连接的线程安全
            typename BusType::Context::ConnectLockGuard contextLock(context.m_contextMutex);
            if (!BusIsConnected())
            {
                // 这里的id是没有作用的
                typename Traits::BusIdType id;
                // m_node初始化为this指针
                m_node = this;
                // 调用内部的ConnectInternal
                BusType::ConnectInternal(context, m_node, contextLock, id);
            }
        }

        template <typename Interface, typename Traits, typename ContainerType>
        void NonIdHandler<Interface, Traits, ContainerType>::BusDisconnect()
        {
            if (typename BusType::Context* context = BusType::GetContext())
            {
                typename BusType::Context::ConnectLockGuard contextLock(context->m_contextMutex);
                if (BusIsConnected())
                {
                    // 调用内部的DisconnectInternal
                    BusType::DisconnectInternal(*context, m_node);
                }
            }
        }
```
### IdHandler
这种Handler用于指定了Address（ID）的EBus，当EBus触发事件时，只有连接至指定ID的Handler才可以监听得到。
它的模板类定义与NonIdHandler一样。
数据成员：
```cpp
using IdType = typename Traits::BusIdType;
typename ContainerType::HandlerNode m_node;
```
也只有一个HandlerNode，但是这个HandlerNode与NonIdHandler中的不同，他是带有ID的，也就是上面定义的带有一个HandlerHolder的HandlerNode。
同样它也有connect相关的函数，
```cpp
            void BusConnect(const IdType& id);
            void BusDisconnect(const IdType& id);
            void BusDisconnect();

            bool BusIsConnectedId(const IdType& id) const
            {
                return BusIsConnected() && m_node.GetBusId() == id;
            }
            
            bool BusIsConnected() const
            {
                return m_node.m_holder != nullptr;
            }
```
判断是否Connected的函数有两个，BusIsConnected()取决于m_node.m_holder是否为空，BusIsConnectedId(const IdType& id)在BusIsConnected()的基础上要求m_node的BusId等于传入的id。
其余connect函数的实现如下：
```cpp
        // IdHandler
        template <typename Interface, typename Traits, typename ContainerType>
        void IdHandler<Interface, Traits, ContainerType>::BusConnect(const IdType& id)
        {
            typename BusType::Context& context = BusType::GetOrCreateContext();
            typename BusType::Context::ConnectLockGuard contextLock(context.m_contextMutex);
            // 判断Handler是否已经连接到了其他BusId，如果是，需要将其断开
            if (BusIsConnected())
            {
                // Connecting on the BusId that is already connected is a no-op
                if (m_node.GetBusId() == id)
                {
                    return;
                }
                AZ_Assert(false, "Connecting to a different id on this bus without disconnecting first! Please ensure you call BusDisconnect before calling BusConnect again, or if multiple connections are desired you must use a MultiHandler instead.");
                BusType::DisconnectInternal(context, m_node);
            }
            // 同样的this赋值给m_node
            m_node = this;
            BusType::ConnectInternal(context, m_node, contextLock, id);
        }
        
        template <typename Interface, typename Traits, typename ContainerType>
        void IdHandler<Interface, Traits, ContainerType>::BusDisconnect(const IdType& id)
        {
            if (typename BusType::Context* context = BusType::GetContext())
            {
                typename BusType::Context::ConnectLockGuard contextLock(context->m_contextMutex);
                // 如果handler确实连接到了此BusId，将其断开
                if (BusIsConnectedId(id))
                {
                    BusType::DisconnectInternal(*context, m_node);
                }
            }
        }
        
        template <typename Interface, typename Traits, typename ContainerType>
        void IdHandler<Interface, Traits, ContainerType>::BusDisconnect()
        {
            if (typename BusType::Context* context = BusType::GetContext())
            {
                typename BusType::Context::ConnectLockGuard contextLock(context->m_contextMutex);
                // 不管handler连接到了哪个BusId，都将其断开
                if (BusIsConnected())
                {
                    BusType::DisconnectInternal(*context, m_node);
                }
            }
        }
```
与NonIdHandler相比，IdHandler需要考虑到BusId是否与当前Handler所连接的id相同，该Handler所连接的id保存在m_node的HandlerHolder中。
### MultiHandler
这种Handler用于使用了Address（ID）并可以将一个Handler连接至多个Address（ID）的EBus。这里需要将Handler所连接的Address（ID）保存起来。
模板类的定义与上面两种Hanlder是相似的，但它的数据成员是一个unordered_map，
```cpp
using IdType = typename Traits::BusIdType;
using HandlerNode = typename ContainerType::HandlerNode;
AZStd::unordered_map<IdType, HandlerNode*, AZStd::hash<IdType>, AZStd::equal_to<IdType>, typename Traits::AllocatorType> m_handlerNodes;
```
这个unordered_map同时指定了hash函数，比较函数，内存分配器。
下面是它的connect相关函数：
```cpp
            void BusConnect(const IdType& id);
            void BusDisconnect(const IdType& id);
            void BusDisconnect();

            bool BusIsConnectedId(const IdType& id) const
            {
                return m_handlerNodes.end() != m_handlerNodes.find(id);
            }

            bool BusIsConnected() const
            {
                return !m_handlerNodes.empty();
            }
```
BusIsConnected判断条件是map是否为空，BusIsConnectedId判断条件是在map中是否保存了该id。
其他相关的函数定义如下：
```cpp
        // MultiHandler
        template <typename Interface, typename Traits, typename ContainerType>
        void MultiHandler<Interface, Traits, ContainerType>::BusConnect(const IdType& id)
        {
            typename BusType::Context& context = BusType::GetOrCreateContext();
            typename BusType::Context::ConnectLockGuard contextLock(context.m_contextMutex);
            // 如果在map中找不到此id，就插入一个
            if (m_handlerNodes.find(id) == m_handlerNodes.end())
            {
                // 分配HandlerNode的内存
                void* handlerNodeAddr = m_handlerNodes.get_allocator().allocate(sizeof(HandlerNode), AZStd::alignment_of<HandlerNode>::value);
                // 使用this指针初始化HandlerNode
                auto handlerNode = new(handlerNodeAddr) HandlerNode(this);
                m_handlerNodes.emplace(id, AZStd::move(handlerNode));
                BusType::ConnectInternal(context, *handlerNode, contextLock, id);
            }
        }
        template <typename Interface, typename Traits, typename ContainerType>
        void MultiHandler<Interface, Traits, ContainerType>::BusDisconnect(const IdType& id)
        {
            if (typename BusType::Context* context = BusType::GetContext())
            {
                typename BusType::Context::ConnectLockGuard contextLock(context->m_contextMutex);
                auto nodeIt = m_handlerNodes.find(id);
                if (nodeIt != m_handlerNodes.end())
                {
                    HandlerNode* handlerNode = nodeIt->second;
                    BusType::DisconnectInternal(*context, *handlerNode);
                    m_handlerNodes.erase(nodeIt);
                    // 由于上面使用了定位new，这里需要显示调用析构函数
                    handlerNode->~HandlerNode();
                    m_handlerNodes.get_allocator().deallocate(handlerNode, sizeof(HandlerNode), alignof(HandlerNode));
                }
            }
        }
        template <typename Interface, typename Traits, typename ContainerType>
        void MultiHandler<Interface, Traits, ContainerType>::BusDisconnect()
        {
            decltype(m_handlerNodes) handlerNodesToDisconnect;
            if (typename BusType::Context* context = BusType::GetContext())
            {
                typename BusType::Context::ConnectLockGuard contextLock(context->m_contextMutex);
                handlerNodesToDisconnect = AZStd::move(m_handlerNodes);
                // 断开所有连接，删除所有节点
                for (const auto& nodePair : handlerNodesToDisconnect)
                {
                    BusType::DisconnectInternal(*context, *nodePair.second);

                    nodePair.second->~HandlerNode();
                    handlerNodesToDisconnect.get_allocator().deallocate(nodePair.second, sizeof(HandlerNode), AZStd::alignment_of<HandlerNode>::value);
                }
            }
        }
```
### 总结
这里定义了两种HandlerNode，一种是带有Id的，一种是不带的，HandlerNode决定了Handler储存时的数据类型，它们都继承自HandlerStorageNode。
定义了3种Handler，分布是不带Id的Handler，带Id的Handler，可以连接至多个Id的Handler（MultiHandler）。前两种Handler将自己的m_node成员赋值为this指针，而MultiHandler比较特殊，它使用一个unordered_map来保存id和HandlerNode（这个HandlerNode也是this指针），并且需要负责它们的生命周期管理。Handler的connect相关函数主要是负责数据成员的管理，然后调用ConnectInternal或DisconnectInternal。
对于NoIdHandler来说，连接就是m_node初始化为this指针，对于IdHandler，需要增加对id的判断，对于MultiHandler，连接时需要自己分配map的节点，将id与HandlerNode关联起来。
## CallstackEntry
CallstackEntry是EBus中一个有用的机制，它可以控制EBus在进行事件发布或者Handler连接时的一些行为。
### CallstackEntryBase
先看CallstackEntry的基础定义：
```cpp
        // Common implementation between single and multithreaded versions
        template <typename Interface, typename Traits>
        struct CallstackEntryBase
        {
            using BusType = EBus<Interface, Traits>;
            // 构造函数需要提供BusId，对于没有BusId的EBus而言，他是一个空Id
            CallstackEntryBase(const typename Traits::BusIdType* id)
                : m_busId(id)
            { }

            virtual ~CallstackEntryBase() = default;
            // CallstackEntry提供的几种行为，它们在不同的时机被调用
            // remove handler时调用，递归调用m_prev的OnRemoveHandler
            virtual void OnRemoveHandler(Interface* handler)
            {
                if (m_prev)
                {
                    m_prev->OnRemoveHandler(handler);
                }
            }
            // hanlder 删除完成时调用
            virtual void OnPostRemoveHandler()
            {
                if (m_prev)
                {
                    m_prev->OnPostRemoveHandler();
                }
            }

            virtual void SetRouterProcessingState(typename BusType::RouterProcessingState /*state*/) { }

            virtual bool IsRoutingQueuedEvent() const { return false; }

            virtual bool IsRoutingReverseEvent() const { return false; }

            const typename Traits::BusIdType* m_busId;
            CallstackEntryBase<Interface, Traits>* m_prev = nullptr;
        };
```
从上面的代码中可以看出，CallstackEntryBase是作为一种链表的节点来设计的，m_prev指向它的前一个节点，m_busId保存了EBus的Id（如果有的话）。OnRemoveHandler和OnPostRemoveHandler两个函数在handler断开连接时调用，并且会一次调用到这条CallstackEntryBase链表上的所有节点。后面3个Route相关的函数用来设置和获取EBus的Route状态。
### CallstackEntry
```cpp
        // Single threaded callstack entry
        template <typename Interface, typename Traits>
        struct CallstackEntry
            : public CallstackEntryBase<Interface, Traits>
        {
            using BusType = EBus<Interface, Traits>;
            using BusContextPtr = typename BusType::Context*;
            // 构造函数接收一个context和一个busId
            CallstackEntry(BusContextPtr context, const typename Traits::BusIdType* busId)
                : CallstackEntryBase<Interface, Traits>(busId)
                , m_threadId(AZStd::this_thread::get_id().m_id)
            {
                EBUS_ASSERT(context, "Internal error: context deleted while execution still in progress.");
                m_context = context;
                // 将此CallstackEntry插到了m_context->s_callstack的链表头部
                this->m_prev = m_context->s_callstack->m_prev;

                // We don't use the AZ_Assert macro here because it places the assert call (unlikely) before the
                // actual work to do (likely) which results in inlining shenanigans and icache misses.
                if (!this->m_prev || static_cast<CallstackEntry*>(this->m_prev)->m_threadId == m_threadId)
                {
                    m_context->s_callstack->m_prev = this;

                    m_context->m_dispatches++;
                }
                else
                {
                    AZ::Debug::Trace::Instance().Assert(__FILE__, __LINE__, AZ_FUNCTION_SIGNATURE,
                        "Bus %s has multiple threads in its callstack records. Configure MutexType on the bus, or don't send to it from multiple threads", BusType::GetName());
                }
            }
            
            ...

            ~CallstackEntry() override
            {
                m_context->m_dispatches--;
                // 将当前的CallstackEntry从链表中删除
                m_context->s_callstack->m_prev = this->m_prev;
            }

            BusContextPtr m_context = nullptr;
            AZStd::native_thread_id_type m_threadId;
        };
```
CallstackEntry是CallstackEntryBase的子类，它专用于单线程，其中保存了当前线程的threadId。
CallstackEntry的初始化需要Ebus的context，当一个CallstackEntry对象创建时，它将自己插入到context中的s_callstack链表，并且保证这个链表上的CallstackEntry都在同一线程中。context的m_dispatches记录了s_callstack链表的节点数量。
从上面的设计中也可以看出，s_callstack就像一个栈一样，插入节点只能从头部插入，删除节点也只能从头部删除，这可能也是将它称为stack的原因。
### CallstackEntryRoot
CallstackEntryRoot同样是CallstackEntryBase的子类，但它与CallstackEntry不同，顾名思义，CallstackEntryRoot作为CallstackEntry的根节点，因为在多线程的情况下，每个线程都有一条CallstackEntry链表，每个链表的头节点都是一个CallstackEntryRoot。
由于CallstackEntryRoot是作为一种“哨兵”节点存在的，所以它不应该被像CallstackEntryBase一样使用。
```cpp
        template <typename Interface, typename Traits>
        struct CallstackEntryRoot
            : public CallstackEntryBase<Interface, Traits>
        {
            using BusType = EBus<Interface, Traits>;

            CallstackEntryRoot()
                : CallstackEntryBase<Interface, Traits>(nullptr)
            {}

            void OnRemoveHandler(Interface*) override { AZ_Assert(false, "Callstack root should never attempt to handle the removal of a bus handler"); }
            void OnPostRemoveHandler() override { AZ_Assert(false, "Callstack root should never attempt to handle the removal of a bus handler"); }
            void SetRouterProcessingState(typename BusType::RouterProcessingState) override { AZ_Assert(false, "Callstack root should never attempt to alter router processing state"); }
            bool IsRoutingQueuedEvent() const override { return false; }
            bool IsRoutingReverseEvent() const override { return false; }
        };

```
可以看到它的所有成员函数基本都处于禁用状态。
### EBusCallstackStorage
用来保存CallstackEntryBase的数据类型，分为单线程和多线程两种。
```cpp
        template <class C, bool UseTLS /*= false*/>
        struct EBusCallstackStorage
        {
            C* m_entry = nullptr;
            ...
        }
```
其中的两个模板参数：
C：保存的数据类型，目前只会被指定为CallstackEntryBase。
UseTLS：是否为多线程。
可以看到，在单线程情况下，就是保存C类型的指针。同时它进行了一些操作符重载，使这个类的行为看起来和指针一样。
相应的，在多线程的情况下，
```cpp
        template <class C>
        struct EBusCallstackStorage<C, true>
        {
            static AZ_THREAD_LOCAL C* s_entry;
            ...
        }
```
保存的指针变成了静态和THREAD_LOCAL的（为什么要是静态的）。
### 总结
CallstackEntry可以看作Ebus的附加机制，使用它可以在某些时机将一个CallstackEntry加入到Ebus的context中，这些CallstackEntry的函数将会在handler断开连接时或者设置route功能时被调用，达到控制EBus行为的效果。
## EBusContainer
EBusContainer可以理解成保存EBus数据的容器，主要是EBus中的Hanlder，并且提供对Handler的回调函数查找和调用。
对于不同类型的EBus，EBusContainer也不一样，最通用的EBusContainer为
```cpp
        // Default impl, used when there are multiple addresses and multiple handlers
        template <typename Interface, typename Traits, EBusAddressPolicy addressPolicy = Traits::AddressPolicy, EBusHandlerPolicy handlerPolicy = Traits::HandlerPolicy>
        struct EBusContainer
        {
            ...
        }
```
它用于multiple addresses and multiple handlers的Ebus，即一个Id和连接多个Handler，一个Handler也可以连接多个Id。
这也是最复杂的EBusContainer，其他类型的EBusContainer都是从这个特化来的，先从简单的EBusContainer开始分析。
### single address, single handler
这是最简单的EBusContainer，它用于只有一个address（不需要Id），只有一个handler的EBus，这种EBus一般用来实现系统中的单例模式：只有一个Handler连接到这个EBus，任何地方向这个Handler发送请求只有这个Handler响应，这个Handler就是一个单例。
它的实现：
```cpp
        // Specialization for single address, single handler
        template <typename Interface, typename Traits>
        struct EBusContainer<Interface, Traits, EBusAddressPolicy::Single, EBusHandlerPolicy::Single>
        {
            // ContainerType为自己的类型，Handller中可以用他来访问自己内部的一些数据结构类型
            using ContainerType = EBusContainer;
            // Id是没用的
            using IdType = typename Traits::BusIdType;
            // CallstackEntry
            using CallstackEntry = AZ::Internal::CallstackEntry<Interface, Traits>;
            // HandlerNode类型会在Handler中用来定义m_node
            // Handlers can just be stored as raw pointer
            // Needs to be public for ConnectionPolicy
            using HandlerNode = Interface*;
            // No need for AddressStorage, there's only 1
            // No need for HandlerStorage, there's only 1
            
            // 使用NonIdHandler，第三个模板参数传入自己的类型
            using Handler = NonIdHandler<Interface, Traits, ContainerType>;
            struct BusPtr { };
            ...
            // Handler只有一个
            HandlerNode m_handler = nullptr;
        }

```
同样，它也有Connect和Disconnect函数，主要用于记录Handler的连接状态，
```cpp
            void Connect(HandlerNode& handler, const IdType&)
            {
                AZ_Assert(!m_handler, "Bus already connected to!");
                m_handler = handler;
            }

            void Disconnect(HandlerNode& handler)
            {
                if (m_handler == handler)
                {
                    m_handler = nullptr;
                }
            }
```
当一个Handler连接到EBus时，就是将这个handler赋值给m_handler。
在EBusContainer中还有一个非常重要的结构体--Dispatcher，它定义了此EBus是如何分发事件的，后续EBus可能将在此基础上进行一些扩充。
```cpp
            // EBus will extend this class to gain the Event*/Broadcast* functions
            template <typename Bus>
            struct Dispatcher
            {
                ...
            }
```
Dispatcher中有5个静态函数：
- Broadcast：广播事件
- BroadcastResult：广播事件并获得结果
- BroadcastReverse：倒序广播事件
- BroadcastResultReverse：倒序广播事件并获得结果
- EnumerateHandlers：所有Handler调用一个回调函数
很明显，对于single address, single handler的EBus来说，Broadcast和BroadcastReverse没有任何区别，因为只有一个Handler，无所谓正序还是倒序，BroadcastResult和BroadcastResultReverse也一样。
Broadcast实现如下：
```cpp
                // Broadcast family
                template <typename Function, typename... ArgsT>
                static void Broadcast(Function&& func, ArgsT&&... args)
                {
                    if (auto* context = Bus::GetContext())
                    {
                        // DispatchLockGuard锁
                        typename Bus::Context::DispatchLockGuard lock(context->m_contextMutex);
                        // 使用Routing可以使某个或者某些Handler不会接收到事件，有可能在这里就导致函数返回
                        EBUS_DO_ROUTING(*context, nullptr, false, false);

                        // m_buses是一个EBusContainer，所以这里的handler就是EBusContainer中的成员m_handler
                        auto handler = context->m_buses.m_handler;
                        if (handler)
                        {
                            // 这里向context的s_callstack插入了这个entry对象
                            // 由于它是一个局部变量，此次调用结束后它就析构了，也就从s_callstack中移出了
                            CallstackEntry entry(context, nullptr);
                            // EventProcessingPolicy调用了handler中的func，相当于触发了事件的回调函数
                            Traits::EventProcessingPolicy::Call(AZStd::forward<Function>(func), handler, AZStd::forward<ArgsT>(args)...);
                        }
                    }
                }
```
Broadcast本质上就是调用了EBusContainer中handler的某个函数，也就是触发了某个事件。
BroadcastResult与Broadcast基本是相同的，只是调用func的时候有一些差异：
```cpp
Traits::EventProcessingPolicy::CallResult(results, AZStd::forward<Function>(func), handler, AZStd::forward<ArgsT>(args)...);
```
Call变成了CallResult，它们都是EventProcessingPolicy模块实现的，后续详细分析它们的实现，这里就看作是函数调用。
EnumerateHandlers函数实现如下：
```cpp
                // Enumerate family
                template <class Callback>
                static void EnumerateHandlers(Callback&& callback)
                {
                    if (auto* context = Bus::GetContext())
                    {
                        typename Bus::Context::DispatchLockGuard lock(context->m_contextMutex);

                        auto handler = context->m_buses.m_handler;
                        if (handler)
                        {
                            CallstackEntry entry(context, nullptr);
                            Traits::EventProcessingPolicy::Call(callback, handler);
                        }
                    }
                }
```
就是让这个handler调用callback，但是它不再有Routing功能。
#### 关于EBUS_DO_ROUTING
在上面的Broadcast函数中，出现了EBUS_DO_ROUTING这个宏，它是一个控制EBus分发事件行为的机制，它的定义为：
```cpp
// Executes router handling in a generic way
#define EBUS_DO_ROUTING(contextParam, id, isQueued, isReverse) \
    do {                                                                                        \
        auto& local_context = (contextParam);                                                   \
        if (local_context.m_routing.m_routers.size()) {                                         \
            if (local_context.m_routing.RouteEvent(id, isQueued, isReverse, func, args...)) {   \
                return;                                                                         \
            }                                                                                   \
        }                                                                                       \
    } while(false)
```
这个宏有4个参数：
contextParam：就是EBus的context。
id：EBus的address，上面没有使用address的EBus提供空指针。
isQueued：事件分发是否排队。
isReverse：是否为反向分发。
它的含义很清晰，主要看EBus的context中的m_routing变量，如果m_routing的m_routers不为空，m_routing就执行RouteEvent方法，如果这个方法返回true，函数就会返回。注意这里的函数指的是Broadcast函数，EBUS_DO_ROUTING只是一个宏，返回语句作用于外面的函数。
这里不需要知道m_routing是什么，也不需要知道RouteEvent中做了什么事，只需要了解在Broadcast函数中，有一个m_routing来控制它，使它提前返回，后面再详细研究m_routing的机制。
### single address, multi handler
这种类型的EBus只有一个address（不需要Id）,并且可以有多个Handler连接到它。可以使用这种EBus来实现观察者模式。
实现：
```cpp
        // Specialization for single address, multi handler
        template <typename Interface, typename Traits, EBusHandlerPolicy handlerPolicy>
        struct EBusContainer<Interface, Traits, EBusAddressPolicy::Single, handlerPolicy>
        {
        public:
            using ContainerType = EBusContainer;
            using IdType = typename Traits::BusIdType;

            using CallstackEntry = AZ::Internal::CallstackEntry<Interface, Traits>;

            // This struct will hold the handlers per address
            // 由于不需要address，HandlerHolder是一个空结构体，只用于模板实例化
            struct HandlerHolder;
             
            // 使用了多个Handler时，HandlerNode需要保存在链表或者multiset中
            // 这里不需要address，所以HandlerNode中不会保存id
            // This struct will hold each handler
            using HandlerNode = AZ::Internal::HandlerNode<Interface, Traits, HandlerHolder>;
            // Defines how handlers are stored per address (will be some sort of list)
            // 决定是使用list还是multiset来保存handler
            using HandlerStorage = HandlerStoragePolicy<Interface, Traits, HandlerNode>;
            // No need for AddressStorage, there's only 1

            struct BusPtr { };
            // Handler是NonIdHandler类型
            using Handler = NonIdHandler<Interface, Traits, ContainerType>;

            EBusContainer() = default;
            ...
            // m_handlers是一个链表或者multiset
            typename HandlerStorage::StorageType m_handlers;
        }
```
与上一个只有一个handler的EBusContainer相比，这个EBusContainer需要一个保存多个handler的机制，所以HandlerStorage会被实例化。
在上一个EBusContainer中，当一个handler连接到EBus时，只需要将该handler赋值给m_handler，而在这个EBusContainer中，当一个handler连接到EBus时，该handler会被插入到m_handlers中。
```cpp
            void Connect(HandlerNode& handler, const IdType&)
            {
                // Don't need to check for duplicates here, because BusConnect would have caught it already
                m_handlers.insert(handler);
            }

            void Disconnect(HandlerNode& handler)
            {
                // Don't need to check that handler is already connected here, because BusDisconnect would have caught it already
                m_handlers.erase(handler);
            }
```
同样，EBusContainer中定义了Dispatcher，由于Dispatcher中各个函数的功能都是相似的，后面就只研究Broadcast函数。
```cpp
                template <typename Function, typename... ArgsT>
                static void Broadcast(Function&& func, ArgsT&&... args)
                {
                    if (auto* context = Bus::GetContext())
                    {
                        typename Bus::Context::DispatchLockGuard lock(context->m_contextMutex);
                        EBUS_DO_ROUTING(*context, nullptr, false, false);

                        auto& handlers = context->m_buses.m_handlers;
                        auto handlerIt = handlers.begin();
                        auto handlersEnd = handlers.end();

                        auto fixer = MakeDisconnectFixer<Bus>(context, nullptr,
                            [&handlerIt, &handlersEnd](Interface* handler)
                            {
                                if (handlerIt != handlersEnd && handlerIt->m_interface == handler)
                                {
                                    ++handlerIt;
                                }
                            },
                            [&handlers, &handlersEnd]()
                            {
                                handlersEnd = handlers.end();
                            }
                        );
                        // 调用m_handlers中的所有handler触发func函数
                        while (handlerIt != handlersEnd)
                        {
                            // @func and @args cannot be forwarded here as rvalue arguments need to bind to const lvalue arguments
                            // due to potential of multiple handlers of this EBus container invoking the function multiple times
                            auto itr = handlerIt++;
                            Traits::EventProcessingPolicy::Call(func, *itr, args...);
                        }
                    }
                }
```
Broadcast函数的逻辑比较清晰，就是遍历所有handler，依次调用func函数。唯一难以理解的部分在fixer变量，它是一个MakeDisconnectFixer类型的对象。
#### MakeDisconnectFixer
MakeDisconnectFixer的定义如下：
```cpp
// Handles updating iterators when disconnecting mid-dispatch
            template <typename Bus, typename PreHandler, typename PostHandler>
            // MidDispatchDisconnectFixer是一个CallstackEntry
            class MidDispatchDisconnectFixer
                : public AZ::Internal::CallstackEntry<typename Bus::InterfaceType, typename Bus::Traits>
            {
                using Base = AZ::Internal::CallstackEntry<typename Bus::InterfaceType, typename Bus::Traits>;

            public:
                template<typename PreRemoveHandler, typename PostRemoveHandler>
                MidDispatchDisconnectFixer(typename Bus::Context* context, const typename Bus::BusIdType* busId, PreRemoveHandler&& pre, PostRemoveHandler&& post)
                    : Base(context, busId)
                    , m_onPreDisconnect(AZStd::forward<PreRemoveHandler>(pre))
                    , m_onPostDisconnect(AZStd::forward<PostRemoveHandler>(post))
                {
                    static_assert(AZStd::is_invocable<PreHandler, typename Bus::InterfaceType*>::value, "Pre handler requires accepting a parameter of Bus::InterfaceType pointer");
                    static_assert(AZStd::is_invocable<PostHandler>::value, "Post handler does not accept any parameters");
                }

                void OnRemoveHandler(typename Bus::InterfaceType* handler) override
                {
                    m_onPreDisconnect(handler);
                    Base::OnRemoveHandler(handler);
                }

                void OnPostRemoveHandler() override
                {
                    m_onPostDisconnect();
                    Base::OnPostRemoveHandler();
                }

            private:
                PreHandler m_onPreDisconnect;
                PostHandler m_onPostDisconnect;
            };

            // Helper for creating a MidDispatchDisconnectFixer, with support for deducing handler types (required for lambdas)
            template <typename Bus, typename PreHandler, typename PostHandler>
            auto MakeDisconnectFixer(typename Bus::Context* context, const typename Bus::BusIdType* busId, PreHandler&& remove, PostHandler&& post)
                -> MidDispatchDisconnectFixer<Bus, PreHandler, PostHandler>
            {
                return MidDispatchDisconnectFixer<Bus, PreHandler, PostHandler>(context, busId, AZStd::forward<PreHandler>(remove), AZStd::forward<PostHandler>(post));
            }
```
MidDispatchDisconnectFixer类的构造需要四个参数，
context：EBus的context。
busId：EBus的address（如果有的话）。
pre：类型是PreRemoveHandler，是一个回调函数。
post：类型是PostRemoveHandler，也是一个回调函数。
从上面对[[Evnt Bus Internal#CallstackEntry]]的分析中可以知道，所有的CallstackEntry都是插入到context的s_callstack中的对象，它们包含了一些函数，Ebus可能在适当时时候调用这些函数，比如这里实现的OnRemoveHandler和OnPostRemoveHandler函数，EBus会在一个handler断开时和断开后调用它们。
那么MidDispatchDisconnectFixer的设计含义就很明显了，它是一个CallstackEntry，拥有
m_onPreDisconnect和m_onPostDisconnect两个回调函数，并且希望Ebus在一个handler断开时和断开后调用它们。MakeDisconnectFixer是用来快速创建MidDispatchDisconnectFixer对象的辅助函数。
那么m_onPreDisconnect和m_onPostDisconnect具体是什么呢，看MidDispatchDisconnectFixer的创建代码：
```cpp
                        auto fixer = MakeDisconnectFixer<Bus>(context, nullptr,
                            [&handlerIt, &handlersEnd](Interface* handler)
                            {
                                if (handlerIt != handlersEnd && handlerIt->m_interface == handler)
                                {
                                    ++handlerIt;
                                }
                            },
                            [&handlers, &handlersEnd]()
                            {
                                handlersEnd = handlers.end();
                            }
                        );
```
它们两个是lambda表达式，并且捕获了Broadcast函数中的局部变量handlerIt，handlersEnd，handlers。注意这里是捕获引用，也就是说在Broadcast函数遍历所有handler时，handlerIt是不断变化的，这两个函数对它们的修改也会影响到Broadcast函数。
对于m_onPreDisconnect，它在一个handler断开连接时调用，如果连接的handler正好是当前的handlerIt指向的handler，那么handlerIt后移，相当于Broadcast函数跳过了这个handler。
对于m_onPostDisconnect，它在一个handler断开连接后调用，将handlersEnd置为handlers.end()，这是为了保证handlersEnd始终是有效的，因为handlers是一个list或者multiset，在一个handler断开连接后该节点会被删除，这有可能导致handlersEnd失效。
CallstackEntry是根据线程来区分的，所以MidDispatchDisconnectFixer只会作用于同一线程。
### multi address, single handler
这种类型的EBus拥有Address(Id)，并且每一个Address只能连接一个handler，但是一个handler可以同时连接至多个address。
```cpp
        template <typename Interface, typename Traits, EBusAddressPolicy addressPolicy>
        struct EBusContainer<Interface, Traits, addressPolicy, EBusHandlerPolicy::Single>
        {
        public:
            using ContainerType = EBusContainer;
            using IdType = typename Traits::BusIdType;

            using CallstackEntry = AZ::Internal::CallstackEntry<Interface, Traits>;

            // This struct will hold the handler per address
            // 由于使用了address，所以这个HandlerHolder需要定义
            struct HandlerHolder;
            // This struct will hold each handler
            // 根据上面的实例化定义，HandlerNode将会保存自己的HandlerHolder
            using HandlerNode = AZ::Internal::HandlerNode<Interface, Traits, HandlerHolder>;
            // Defines how handler holders are stored (will be some sort of map-like structure from id -> handler holder)
            // AddressStorage将会是一个哈希表或者红黑树，每一个节点是一个address-HandlerHolder的键值对
            using AddressStorage = AddressStoragePolicy<Traits, HandlerHolder>;
            // No need for HandlerStorage, there's only 1 so it will always just be a HandlerNode*
            
            // 此时要用到两种handler，只连接一个address的Handler，可以连接至多个address的MultiHandler
            using Handler = IdHandler<Interface, Traits, ContainerType>;
            using MultiHandler = AZ::Internal::MultiHandler<Interface, Traits, ContainerType>;
            // BusPtr将是一个HandlerHolder的指针
            using BusPtr = AZStd::intrusive_ptr<HandlerHolder>;

            EBusContainer() = default;
            ...
            typename AddressStorage::StorageType m_addresses;
}
```
#### HandlerHolder
由于这个EBusContainer是有Address的，所以HandlerHolder需要有定义。
```cpp            
            struct HandlerHolder
            {
                // 保存m_busContainer
                ContainerType& m_busContainer;
                // address
                IdType m_busId;
                // 保存它持有的handler
                HandlerNode* m_handler = nullptr;
                // Cache of the interface to save an indirection to m_handler
                // 和m_handler是同一个
                Interface* m_interface = nullptr;
                // 引用计数
                AZStd::atomic_uint m_refCount{ 0 };
                ...
            }

```
构造函数：
```cpp
                HandlerHolder(ContainerType& storage, const IdType& id)
                    : m_busContainer(storage)
                    , m_busId(id)
                { }
```
HandlerHolder只允许移动构造，拷贝构造，赋值操作符，移动赋值操作符都是禁止的，具体的定义省略。
几个成员函数：
```cpp
                bool HasHandlers() const
                {
                    return m_interface != nullptr;
                }

                void add_ref()
                {
                    m_refCount.fetch_add(1);
                }
                void release()
                {
                    // Must check against 1 because fetch_sub returns the value before decrementing
                    if (m_refCount.fetch_sub(1) == 1)
                    {
                        m_busContainer.m_addresses.erase(m_busId);
                    }
                }
```
这就是HandlerHolder的所有功能，可以判断是否持有handler，增加引用计数和减少引用计数，当减少引用计数为0时，自动将address从EBusContainer中删掉。
HandlerHolder还有一个相关的函数：
```cpp
            HandlerHolder& FindOrCreateHandlerHolder(const IdType& id)
            {
                auto busIt = m_addresses.find(id);
                if (busIt == m_addresses.end())
                {
                    busIt = m_addresses.emplace(*this, id);
                }

                return *busIt;
            }
```
EBusContainer使用id来创建或查找一个HandlerHolder，并返回其引用。
#### EBusContainer
了解了HandlerHolder的定义之后，回到EBusContainer，先看它的connect相关函数。
```cpp
            void Connect(HandlerNode& handler, const IdType& id)
            {
                EBUS_ASSERT(!handler.m_holder,
                    "Internal error: Connect() may not be called by a handler that is already connected."
                    "BusConnect() should already have handled this case.");

                HandlerHolder& holder = FindOrCreateHandlerHolder(id);
                holder.m_handler = &handler;
                holder.m_interface = handler.m_interface;
                handler.m_holder = &holder;
            }

            void Disconnect(HandlerNode& handler)
            {
                EBUS_ASSERT(handler.m_holder, "Internal error: disconnecting handler that is incompletely connected");

                handler.m_holder->m_handler = nullptr;
                handler.m_holder->m_interface = nullptr;

                // Must reset handler after removing it from the list, otherwise m_holder could have been destroyed already (and handlerList would be invalid)
                handler.m_holder.reset();
            }
```
当handler连接至某个id时，EBusContainer从m_addresses中寻找和创建HandlerHolder，并将此handler赋值给HandlerHolder的m_handler，同样handler也将此HandlerHolder保存在自己的m_holder中。
当handler断开连接时，他将自己保存的HandlerHolder引用中的信息清空，HandlerHolder并不会立即从EBusContainer删除，他会reset减少自己的引用计数，当引用减为0时，它才会从EBusContainer中删除（见HandlerHolder的release函数）。

拥有Address的EBusContainer会多一个Bind函数：
```cpp
            void Bind(BusPtr& busPtr, const IdType& id)
            {
                busPtr = &FindOrCreateHandlerHolder(id);
            }
```
将一个id绑定到一个busPtr。其实就是创建一个没有handler的HandlerHolder。

接下来依然是Dispatcher，与上面两个EBusContainer不同的是，这次的Dispatcher多个Event相关函数，它可以指定事件发布到哪些Address的handler。
```cpp
            template <typename Bus>
            struct Dispatcher
            {
                // Event family
                template <typename Function, typename... ArgsT>
                static void Event(const IdType& id, Function&& func, ArgsT&&... args)
                {
                    if (auto* context = Bus::GetContext())
                    {
                        typename Bus::Context::DispatchLockGuard lock(context->m_contextMutex);
                        EBUS_DO_ROUTING(*context, &id, false, false);

                        auto& addresses = context->m_buses.m_addresses;
                        auto addressIt = addresses.find(id);
                        if (addressIt != addresses.end() && addressIt->m_interface)
                        {
                            CallstackEntry entry(context, &addressIt->m_busId);
                            Traits::EventProcessingPolicy::Call(AZStd::forward<Function>(func), addressIt->m_interface, AZStd::forward<ArgsT>(args)...);
                        }
                    }
                }
                ...
            }
```
Event函数需要指定id，然后EBusContainer会从addresses找到到该id对应的HandlerHolder，然后触发它的func函数。

Broadcast函数：
```cpp
                // Broadcast family
                template <typename Function, typename... ArgsT>
                static void Broadcast(Function&& func, ArgsT&&... args)
                {
                    if (auto* context = Bus::GetContext())
                    {
                        typename Bus::Context::DispatchLockGuard lock(context->m_contextMutex);
                        EBUS_DO_ROUTING(*context, nullptr, false, false);

                        auto& addresses = context->m_buses.m_addresses;
                        auto addressIt = addresses.begin();
                        auto addressesEnd = addresses.end();

                        auto fixer = MakeDisconnectFixer<Bus>(context, nullptr,
                            [&addressIt, &addressesEnd](Interface* handler)
                            {
                                if (addressIt != addressesEnd && addressIt->m_handler->m_interface == handler)
                                {
                                    ++addressIt;
                                }
                            },
                            [&addresses, &addressesEnd]()
                            {
                                addressesEnd = addresses.end();
                            }
                        );

                        while (addressIt != addressesEnd)
                        {
                            fixer.m_busId = &addressIt->m_busId;
                            if (Interface* inst = (addressIt++)->m_interface)
                            {
                                // @func and @args cannot be forwarded here as rvalue arguments need to bind to const lvalue arguments
                                // due to potential of multiple addresses of this EBus container invoking the function multiple times
                                Traits::EventProcessingPolicy::Call(func, inst, args...);
                            }
                        }
                    }
                }
```
Broadcast遍历所有HandlerHolder，并触发它的handler函数，由于一个handler可以连接到多个Address，所以有的handler可能被触发多次（MultiHandler）。
有一点值得注意的是，fixer的m_busId会在每一次遍历address的时候赋值，回顾MidDispatchDisconnectFixer的设计，busId一直是CallstackEntry的参数，不过之前没有Address，所以它一直为空，这里有了Address，在每一次遍历时，将当前的Address赋值给它，这样可以在合适的时候读取这个信息，知道当前在触发哪个Address的事件。
### multiple addresses and multiple handlers
这是最复杂的一种EBusContainer，它用于有Address，并且一个Address可以连接多个handler，同样，一个handler也可以连接到多个Address。可以说，这种EBus完全覆盖了之前3种EBus的功能，相对的，它的性能也最差。
EBusContainer定义：
```cpp
        // Default impl, used when there are multiple addresses and multiple handlers
        template <typename Interface, typename Traits, EBusAddressPolicy addressPolicy = Traits::AddressPolicy, EBusHandlerPolicy handlerPolicy = Traits::HandlerPolicy>
        struct EBusContainer
        {
        public:
            using ContainerType = EBusContainer;
            using IdType = typename Traits::BusIdType;

            using CallstackEntry = AZ::Internal::CallstackEntry<Interface, Traits>;

            // This struct will hold the handlers per address
            struct HandlerHolder;
            // This struct will hold each handler
            using HandlerNode = AZ::Internal::HandlerNode<Interface, Traits, HandlerHolder>;
            // Defines how handler holders are stored (will be some sort of map-like structure from id -> handler holder)
            using AddressStorage = AddressStoragePolicy<Traits, HandlerHolder>;
            // Defines how handlers are stored per address (will be some sort of list)
            using HandlerStorage = HandlerStoragePolicy<Interface, Traits, HandlerNode>;

            using Handler = IdHandler<Interface, Traits, ContainerType>;
            using MultiHandler = AZ::Internal::MultiHandler<Interface, Traits, ContainerType>;
            using BusPtr = AZStd::intrusive_ptr<HandlerHolder>;
            ...
            typename AddressStorage::StorageType m_addresses;
        }
```
可以看到基本和multiple address, single handler类型的EBusContainer定义是一样的，多了HandlerStorage的定义，因为现在的handler有多个，需要使用一个list或者multiset来保存，而上一个EBusContainer只需要保存HandlerNode\*(interface\*)就可以了。
#### HandlerHolder
由于需要保存多个handler，HandlerHolder的定义会有所不同，
```cpp
            struct HandlerHolder
            {
                ContainerType& m_busContainer;
                IdType m_busId;
                typename HandlerStorage::StorageType m_handlers;
                AZStd::atomic_uint m_refCount{ 0 };
                ...
            }
```
每一个HandlerHolder保存了m_handlers，其余的定义与上一个HandlerHolder相似。
connect相关函数：
```cpp
            void Connect(HandlerNode& handler, const IdType& id)
            {
                EBUS_ASSERT(!handler.m_holder,
                    "Internal error: Connect() may not be called by a handler that is already connected."
                    "BusConnect() should already have handled this case.");

                HandlerHolder& holder = FindOrCreateHandlerHolder(id);
                holder.m_handlers.insert(handler);
                handler.m_holder = &holder;
            }

            void Disconnect(HandlerNode& handler)
            {
                EBUS_ASSERT(handler.m_holder, "Internal error: disconnecting handler that is incompletely connected");

                handler.m_holder->m_handlers.erase(handler);

                // Must reset handler after removing it from the list, otherwise m_holder could have been destroyed already (and handlerList would be invalid)
                handler.m_holder.reset();
            }
```
当一个handler连接到EBus时，会将此handler保存到m_handlers中。
#### EBusContainer
再来看Dispatcher的定义，了解此类型的EBus是如何工作的，
Event函数：
```cpp
template <typename Function, typename... ArgsT>
                static void Event(const IdType& id, Function&& func, ArgsT&&... args)
                {
                    if (auto* context = Bus::GetContext())
                    {
                        typename Bus::Context::DispatchLockGuard lock(context->m_contextMutex);
                        EBUS_DO_ROUTING(*context, &id, false, false);
                        // 根据id取出addressIt
                        auto& addresses = context->m_buses.m_addresses;
                        auto addressIt = addresses.find(id);
                        if (addressIt != addresses.end())
                        {
                            HandlerHolder& holder = *addressIt;
                            holder.add_ref();

                            auto& handlers = holder.m_handlers;
                            auto handlerIt = handlers.begin();
                            auto handlersEnd = handlers.end();

                            auto fixer = MakeDisconnectFixer<Bus>(context, &id,
                                [&handlerIt, &handlersEnd](Interface* handler)
                                {
                                     if (handlerIt != handlersEnd && handlerIt->m_interface == handler)
                                    {
                                        ++handlerIt;
                                    }
                                },
                                [&handlers, &handlersEnd]()
                                {
                                    handlersEnd = handlers.end();
                                }
                            );

                            while (handlerIt != handlersEnd)
                            {
                                auto itr = handlerIt++;
                                Traits::EventProcessingPolicy::Call(func, *itr, args...);
                            }

                            holder.release();
                        }
                    }
                }
```
在一个HandlerHolder的m_handlers遍历期间，会调用holder.add_ref()，这会让HandlerHolder的引用计数增加，保证它不会被释放。（因为遍历期间迭代器是可能失效的？）
同样Broadcast则是遍历所有m_addresses，再依次遍历它的m_handlers，这里就不再赘述。
### 总结
EBusContainer就是EBus的容器，它用于保存EBus使用的数据，并提供了核心的dispatch方法，其中涉及了Internal中的所有模块。
AddressStoragePolicy用于保存Address（Id），它可能是一个红黑树或一个哈希表，由Address是否需要排序决定。每一个节点id为Address，value为一个handler或handler的列表。
HandlerStoragePolicy用于保存Handler，它可能是一个multi_set或者一个list，只有使用了多个handler的EBus才会用到它，只有一个handler的话只保存handler指针即可。
Handler是事件的监听者，它继承了EBus的interface，并且分为3种类型，没有Address（Id）的handler，有一个id的handler，有多个id的handler（使用unordered_map保存）。
HandlerNode是用来保存Handler的类，它由一个interface\*和一个HandlerHolder组成，当handler有id时，才会使用HandlerHolder。每一种Handler都保存了一个或多个HandlerNode，HandlerNode拥有和interface\*同样的行为（通过重载运算符），当handler连接时，它会将this指针赋值给自己的
HandlerNode，比较特殊的是，有多个id的handler会生成多个HandlerNode。HandlerStoragePolicy保存的每一个节点都是HandlerNode。
Handler与HandlerNode的对应关系是：

| Hander       | HanderNode                                     | 解释                                     |
| ------------ | ---------------------------------------------- | -------------------------------------- |
| NonIdHandler | 不包含HandlerHolder                               | 单纯的保存Interface即可                       |
| IdHandler    | 包含HandlerHolder                                | 保存Interface和HandlerHolder（包含Id）        |
| MultiHandler | 多个包含HandlerHolder的HanderNode，使用unordered_map保存 | 保存Interface和它连接到的所有HandlerHolder（包含Id） |
CallstackEntry是一个控制dispatch行为的机制，每一个线程有自己的CallstackEntry链表，CallstackEntry定义了一些函数，EBus会在某些时候触发它们，比如handler断开连接时。一个使用的示例就是在handler断开连接时调整正在dispatch的行为，跳过已经断开连接的handler。
下面时各种EBusContainer的总结：
- single address, single handler：只需要保存一个HandlerNode即可。
```cpp
using HandlerNode = Interface*;
HandlerNode m_handler = nullptr;
```
- single address, multi handler：需要使用HandlerStoragePolicy保存多个handler。
```cpp
using HandlerNode = AZ::Internal::HandlerNode<Interface, Traits, HandlerHolder>;
using HandlerStorage = HandlerStoragePolicy<Interface, Traits, HandlerNode>;
typename HandlerStorage::StorageType m_handlers;
```
- multi address, single handler：需要使用AddressStoragePolicy保存Address。
```cpp
using HandlerNode = AZ::Internal::HandlerNode<Interface, Traits, HandlerHolder>;
using AddressStorage = AddressStoragePolicy<Traits, HandlerHolder>;
typename AddressStorage::StorageType m_addresses;
```
- multi address, multi handler：需要同时使用AddressStoragePolicy和HandlerStoragePolicy。
```cpp
using HandlerNode = AZ::Internal::HandlerNode<Interface, Traits, HandlerHolder>;
using AddressStorage = AddressStoragePolicy<Traits, HandlerHolder>;
using HandlerStorage = HandlerStoragePolicy<Interface, Traits, HandlerNode>;
typename AddressStorage::StorageType m_addresses;
```
