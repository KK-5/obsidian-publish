# Policy设计理念
Policy是设计模式中“策略模式”的一种应用。在程序中，使用组合代替继承是一种更好的方法，假设一个类需要实现特别多的功能，将所有的功能在其中实现会导致大量的编程心智负荷，并且其中一个功能发生改变的时候，不可避免的需要修改整个类的设计。
如果将此类的所有功能拆分成多个部分，每一部分由一个类来实现，然后再将所有的类组合起来，不但使类的设计更加清晰，并且可以根据需要灵活地替换掉其中一部分功能。这就是组合带来的好处。
如果使用多重继承来实现组合，并不利于替换其中的某个功能，修改类的继承方式是一件麻烦的事。在C++中，模板非常合适用来实现组合功能，因为模板在编译期生成代码，可以通过不同的特化实现不同的组合方式。
举一个例子，假设现在有一个类，它的其中一个功能就是通过Create函数创造自身对象（因为各种原因，我们希望可以控制这个类的构建行为，比如管理它的内存分配方式）。可以这样设计：
```cpp
template<typename T>
class CNewCreator
{
public:
    static T* Create() { return new T; }
};

template<typename T>
class CMallocCreator
{
public:
    static T* Create()
    {
        void* pBuff = malloc(sizeof(T));
        if (pBuff)
        {
            return static_cast<T*>(pBuff);
        }
        return nullptr;
    }
};

template<typename T>
class CNotCreator
{
public:
    static T* Create()
    {
        return nullptr;
    }
};

template<typename T, template<typename> class Creator>
class Test: public Creator<T>
{
public:
    void Create()
    {
        value = Creator<T>::Create();
    }
private:
    T* value;
}
```
接下来就可以根据需要选择特化哪种类型的Creator。
```cpp
int main()
{
    Test<AngClass, CNewCreator> newTest;
    Test<AngClass, CMallocCreator> mallocTest;
}
```
newTest和mallocTest都是Test类型，但是它们使用不同的创建方式创建AngClass对象。

# EBusAddressPolicy
用来定义EBus是否使用Address，并且确定Address是否可以排序。
```cpp
    enum class EBusAddressPolicy
    {
        /**
         * (Default) The EBus has a single address.
         */
        Single,

        /**
         * The EBus has multiple addresses; the order in which addresses
         * receive events is undefined.
         * Events that are addressed to an ID are received by handlers
         * that are connected to that ID.
         * Events that are broadcast without an ID are received by
         * handlers at all addresses.
         */
        ById,

        /**
         * The EBus has multiple addresses; the order in which addresses
         * receive events is defined.
         * Events that are addressed to an ID are received by handlers
         * that are connected to that ID.
         * Events that are broadcast without an ID are received by
         * handlers at all addresses.
         * The order in which addresses receive events is defined by
         * AZ::EBusTraits::BusIdOrderCompare.
         */
        ByIdAndOrdered,
    };
```
# EBusHandlerPolicy
决定可以连接到EBus的Handler数量。
```cpp
    /**
     * Defines how many handlers can connect to an address on the EBus
     * and the order in which handlers at each address receive events.
     */
    enum class EBusHandlerPolicy
    {
        /**
         * The EBus supports one handler for each address.
         */
        Single,

        /**
         * (Default) Allows any number of handlers for each address;
         * handlers at an address receive events in the order
         * in which the handlers are connected to the EBus.
         */
        Multiple,

        /**
         * Allows any number of handlers for each address; the order
         * in which each address receives an event is defined
         * by AZ::EBusTraits::BusHandlerOrderCompare.
         */
        MultipleAndOrdered,
    };
```
# EBusConnectionPolicy
提供用户使用的connection policies，可以使用其自定义EBus连接时触发的动作。
```cpp
    template <class Bus>
    struct EBusConnectionPolicy
    {
        // 一些必要的数据，如BusPtr BusIdType
        ...
        
         /**
         * Connects a handler to an EBus address.
         * @param ptr[out] A pointer that will be bound to the EBus address that
         * the handler will be connected to.
         * @param context Global data for the EBus.
         * @param handler The handler to connect to the EBus address.
         * @param id The ID of the EBus address that the handler will be connected to.
         */
        static void Connect(BusPtr& ptr, Context& context, HandlerNode& handler, ConnectLockGuard& contextLock, const BusIdType& id = 0);

        /**
         * Disconnects a handler from an EBus address.
         * @param context Global data for the EBus.
         * @param handler The handler to disconnect from the EBus address.
         * @param ptr A pointer to a specific address on the EBus.
         */
        static void Disconnect(Context& context, HandlerNode& handler, BusPtr& ptr);
    };

    ////////////////////////////////////////////////////////////
    // Implementations
    ////////////////////////////////////////////////////////////
    template <class Bus>
    void EBusConnectionPolicy<Bus>::Connect(BusPtr&, Context&, HandlerNode&, ConnectLockGuard&, const BusIdType&)
    {
    }

    template <class Bus>
    void EBusConnectionPolicy<Bus>::Disconnect(Context&, HandlerNode&, BusPtr&)
    {
    }
```
用户可以自定义EBusConnectionPolicy\<Bus\>::Connect和Disconnect，默认它们什么都不做。
# StoragePolicy
## EBusGlobalStoragePolicy
定义EBus的数据储存的位置，默认每一个dll储存它们自己的EBus实例数据。这里使用了静态局部变量。
```cpp
    /**
     * A choice of AZ::EBusTraits::StoragePolicy that specifies
     * that EBus data is stored in a global static variable.
     * With this policy, each module (DLL) has its own instance of the EBus.
     * @tparam Context A class that contains EBus data.
     */
    template <class Context>
    struct EBusGlobalStoragePolicy
    {
        /**
         * Returns the EBus data.
         * @return A pointer to the EBus data.
         */
        static Context* Get()
        {
            // Because the context in this policy lives in static memory space, and
            // doesn't need to be allocated, there is no reason to defer creation.
            return &GetOrCreate();
        }

        /**
         * Returns the EBus data.
         * @return A reference to the EBus data.
         */
        static Context& GetOrCreate()
        {
            static Context s_context;
            return s_context;
        }
    };
```
## EBusThreadLocalStoragePolicy
定义EBus的数据储存的位置，每个线程储存它自己的EBus实例数据。
```cpp
    /**
     * A choice of AZ::EBusTraits::StoragePolicy that specifies
     * that EBus data is stored in a thread_local static variable.
     * With this policy, each thread has its own instance of the EBus.
     * @tparam Context A class that contains EBus data.
     */
    template <class Context>
    struct EBusThreadLocalStoragePolicy
    {
        /**
         * Returns the EBus data.
         * @return A pointer to the EBus data.
         */
        static Context* Get()
        {
            // Because the context in this policy lives in static memory space, and
            // doesn't need to be allocated, there is no reason to defer creation.
            return &GetOrCreate();
        }

        /**
         * Returns the EBus data.
         * @return A reference to the EBus data.
         */
        static Context& GetOrCreate()
        {
            thread_local static Context s_context;
            return s_context;
        }
    };
```
# EBusQueuePolicy
用来顺序执行EBus指定的function。
默认没有任何实现，
```cpp
    template <bool IsEnabled, class Bus, class MutexType>
    struct EBusQueuePolicy
    {
        typedef AZ::Internal::NullBusMessageCall BusMessageCall;
        void Execute() {};
        void Clear() {};
        void SetActive(bool /*isActive*/) {};
        bool IsActive() { return false; }
        size_t Count() const { return 0; }
    };
```
当IsEnabled为true时，特化版本才有实现，
```cpp
    template <class Bus, class MutexType>
    struct EBusQueuePolicy<true, Bus, MutexType>
    {
        typedef AZStd::function<void()> BusMessageCall; // 可以执行的函数
        
        typedef AZStd::deque<BusMessageCall, typename Bus::AllocatorType> DequeType;
        typedef AZStd::queue<BusMessageCall, DequeType > MessageQueueType;

        EBusQueuePolicy() = default;

        bool                        m_isActive = Bus::Traits::EventQueueingActiveByDefault;
        MessageQueueType            m_messages;
        MutexType                   m_messagesMutex;        ///< Used to control access to the m_messages. Make sure you never interlock with the EBus mutex. Otherwise, a deadlock can occur.
        // 顺序执行m_messages中的所有函数
        void Execute()
        {
            AZ_Warning("System", m_isActive, "You are calling execute queued functions on a bus which has not activated its function queuing! Call YourBus::AllowFunctionQueuing(true)!");

            MessageQueueType localMessages;

            // Swap the current list of queue functions with a local instance
            {
                AZStd::scoped_lock lock(m_messagesMutex);
                AZStd::swap(localMessages, m_messages);
            }

            // Execute the queue functions safely now that are owned by the function
            while (!localMessages.empty())
            {
                const BusMessageCall& localMessage = localMessages.front();
                localMessage();
                localMessages.pop();
            }
        }

        void Clear()
        {
            AZStd::lock_guard<MutexType> lock(m_messagesMutex);
            m_messages = {};
        }

        void SetActive(bool isActive)
        {
            AZStd::lock_guard<MutexType> lock(m_messagesMutex);
            m_isActive = isActive;
            if (!m_isActive)
            {
                m_messages = {};
            }
        };

        bool IsActive()
        {
            return m_isActive;
        }

        size_t Count()
        {
            AZStd::lock_guard<MutexType> lock(m_messagesMutex);
            return m_messages.size();
        }
    };
```
非常简单的功能，管理一个队列的function，并且可以控制是否active，Executes顺序执行所有function，并且此队列保证是线程安全的。
# Router
Router功能用来控制在EBus分发事件的行为，在EBusContainer中就使用过它，见 [[Evnt Bus Internal]]  关于EBUS_DO_ROUTING 章节。
## EBusRouterNode
封装一个Interface，使其可以成为intrusive_multiset_node，也就是这个对象可以作为一个节点放到intrusive_multiset中。
```cpp
    template <class Interface>
    struct EBusRouterNode
        : public AZStd::intrusive_multiset_node<EBusRouterNode<Interface>>
    {
        Interface*  m_handler = nullptr;
        int m_order = 0;

        EBusRouterNode& operator=(Interface* handler);

        Interface* operator->() const;

        operator Interface*() const;

        bool operator<(const EBusRouterNode& rhs) const;
    };

    template <class Interface>
    inline EBusRouterNode<Interface>& EBusRouterNode<Interface>::operator=(Interface* handler)
    {
        m_handler = handler;
        return *this;
    }

    //////////////////////////////////////////////////////////////////////////
    template <class Interface>
    inline Interface* EBusRouterNode<Interface>::operator->() const
    {
        return m_handler;
    }

    //////////////////////////////////////////////////////////////////////////
    template <class Interface>
    inline EBusRouterNode<Interface>::operator Interface*() const
    {
        return m_handler;
    }

    //////////////////////////////////////////////////////////////////////////
    template <class Interface>
    bool EBusRouterNode<Interface>::operator<(const EBusRouterNode& rhs) const
    {
        return m_order < rhs.m_order;
    }
```
在EBus中，Interface表示可以收到事件通知的对象，也就是Handler，所以EBusRouterNode的功能与Handler相似，都是用来接收事件通知并触发某个函数的。
## EBusRouterPolicy
包含多个EBusRouterNode，可以执行这些EBusRouterNode的某个函数。
```cpp
    template <class Bus>
    struct EBusRouterPolicy
    {
        using RouterNode = EBusRouterNode<typename Bus::InterfaceType>;
        typedef AZStd::intrusive_multiset<RouterNode, AZStd::intrusive_multiset_base_hook<RouterNode>> Container;

        // We have to share this with a general processing event if we want to support stopping messages in progress.
        enum class EventProcessingState : int
        {
            ContinueProcess, ///< Continue with the event process as usual (default).
            SkipListeners, ///< Skip all listeners but notify all other routers.
            SkipListenersAndRouters, ///< Skip everybody. Nobody should receive the event after this.
        };

        template <class Function, class... InputArgs>
        inline bool RouteEvent(const typename Bus::BusIdType* busIdPtr, bool isQueued, bool isReverse, Function&& func, InputArgs&&... args);

        Container m_routers;
    };
```
这里定义了EventProcessingState，有3个，分别是：
- ContinueProcess：默认设置，继续处理事件。
- SkipListeners：跳过所有监听者（Handler）但是通知Routers.
- SkipListenersAndRouters：跳过所有人，也就是此事件在此次处理后就结束。
在EBus中对Router功能有这样一段解释：
```cpp
        /**
         * Controls the flow of EBus events.
         * Enables an event to be forwarded, and possibly stopped, before reaching
         * the normal event handlers.
         * Use cases for routing include tracing, debugging, and versioning an %EBus.
         * The default `EBusRouterPolicy` forwards the event to each connected
         * `EBusRouterNode` before sending the event to the normal handlers. Each
         * node can stop the event or let it continue.
         */
```
使用Router可以控制EBus事件的传播流，比如在Handler收到一个事件前，将此事件转发或停止，用来追踪或Debug事件。默认情况下，EBus将事件发送至Handler前，会先将此事件发送至EBusRouterNode，而EBusRouterNode可以控制事件是否停止。
所以上面的3个EventProcessingState就是用来控制Router的设置。
在EBusContainer的实现中，进行Broadcas或Event触发事件时，都会先执行 EBUS_DO_ROUTING，它根据RouteEvent函数的返回结果来决定是否继续执行事件分发任务，所以Router功能的核心就在于它的RouteEvent函数，下面来看它的实现：
```cpp
    template <class Bus>
    template <class Function, class... InputArgs>
    inline bool EBusRouterPolicy<Bus>::RouteEvent(const typename Bus::BusIdType* busIdPtr, bool isQueued, bool isReverse, Function&& func, InputArgs&&... args)
    {
        auto rtLast = m_routers.end();
        // 这里使用m_routers的begin迭代器来创建一个RouterCallstackEntry对象
        typename Bus::RouterCallstackEntry rtCurrent(m_routers.begin(), busIdPtr, isQueued, isReverse);
        // 遍历m_routers中的所有EBusRouterNode，并触发它们的func函数
        while (rtCurrent.m_iterator != rtLast)
        {
            AZStd::invoke(func, (*rtCurrent.m_iterator++), args...);
            // SkipListenersAndRouters，此事件在发送给当前EBusRouterNode后就结束，这里立即返回true
            if (rtCurrent.m_processingState == EventProcessingState::SkipListenersAndRouters)
            {
                return true;
            }
        }
        // ContinueProcess，所有EBusRouterNode收到事件后，还要将事件发送给Handler
        if (rtCurrent.m_processingState != EventProcessingState::ContinueProcess)
        {
            return true;
        }
        // SkipListeners 所有EBusRouterNode收到事件后，不再将事件发送给Handler
        return false;
    }
```
这里的实现有Bug？在EBUS_DO_ROUTING中是RouteEvent返回true事件就停止，这里的ContinueProcess和SkipListeners的处理写反了。
在此函数中，使用到了EBus中的对象Bus::RouterCallstackEntry，它的定义如下：
```cpp
        struct RouterCallstackEntry
            : public CallstackEntry
        {
            typedef typename RouterPolicy::Container::iterator Iterator;

            RouterCallstackEntry(Iterator it, const BusIdType* busId, bool isQueued, bool isReverse);

            ~RouterCallstackEntry() override = default;

            void SetRouterProcessingState(RouterProcessingState state) override;

            bool IsRoutingQueuedEvent() const override;

            bool IsRoutingReverseEvent() const override;

            Iterator m_iterator;
            RouterProcessingState m_processingState;
            bool m_isQueued;
            bool m_isReverse;
        };

    //=========================================================================
    template<class Interface, class Traits>
    EBus<Interface, Traits>::RouterCallstackEntry::RouterCallstackEntry(Iterator it, const BusIdType* busId, bool isQueued, bool isReverse)
        : CallstackEntry(EBus::GetContext(), busId)
        , m_iterator(it)
        , m_processingState(RouterPolicy::EventProcessingState::ContinueProcess)
        , m_isQueued(isQueued)
        , m_isReverse(isReverse)
    {
    }

    //=========================================================================
    template<class Interface, class Traits>
    void EBus<Interface, Traits>::RouterCallstackEntry::SetRouterProcessingState(RouterProcessingState state)
    {
        m_processingState = state;
    }

    //=========================================================================
    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::RouterCallstackEntry::IsRoutingQueuedEvent() const
    {
        return m_isQueued;
    }

    //=========================================================================
    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::RouterCallstackEntry::IsRoutingReverseEvent() const
    {
        return m_isReverse;
    }
```
首先，它是一个CallstackEntry，这种对象的特点是当创建对象时它自动加入EBus的m_context->s_callstack中，它是一个链表，并且每个线程有一个自己的s_callstack。当一个Handler断开时连接，它的OnRemoveHandler和OnPostRemoveHandler函数会被调用（这里没有使用这个特性）。而当一个CallstackEntry析构时，它自动从s_callstack删除。
借用CallstackEntry自动加入s_callstack和从其中删除的特性，在RouteEvent函数调用期间，EBus可以很方便地询问Router的一些状态，比如上面的IsRoutingQueuedEvent和IsRoutingReverseEvent。
# EBusEventProcessingPolicy
最后的EBusEventProcessingPolicy用来实现EBus的函数调用操作，在EBusContainer中，Event和Broadcast函数就是使用它来调用handler的函数的，
EBusEventProcessingPolicy具体的实现如下：
```cpp
    struct EBusEventProcessingPolicy
    {
        template<class Results, class Function, class Interface, class... InputArgs>
        static void CallResult(Results& results, Function&& func, Interface&& iface, InputArgs&&... args)
        {
            results = AZStd::invoke(AZStd::forward<Function>(func), AZStd::forward<Interface>(iface), AZStd::forward<InputArgs>(args)...);
        }

        template<class Function, class Interface, class... InputArgs>
        static void Call(Function&& func, Interface&& iface, InputArgs&&... args)
        {
            AZStd::invoke(AZStd::forward<Function>(func), AZStd::forward<Interface>(iface), AZStd::forward<InputArgs>(args)...);
        }
    };
```
有CallResult和Call两个版本，其中都是使用std::invoke来实现的（C++17加入的新特性）。
# 总结
按照程序数据与行为分离的实现思想，EBusContainer保存了EBus所使用的数据，而Policy模块则提供了EBus各种行为的实现，包括EBus本身Address和Handler的控制，Router功能和EventProcessing调用可执行函数，