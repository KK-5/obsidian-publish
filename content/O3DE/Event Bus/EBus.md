EBus是O3DE引擎中的事件系统，更加详细的信息可以参考官方文档：[EBus](https://www.docs.o3de.org/docs/user-guide/programming/messaging/)
# EBus中的一些概念
Interface：一个抽象类，其中定义了EBus需要分发或者接收的事件（虚函数）。
Traits：用来定义EBus的属性，一般来说会使用Interface继承EBusTraits来实现Traits功能，无需额外提供。
Handler：连接到EBus的实例，EBus分发事件或接收请求时，会触发它们的响应函数，它继承自Interface的类。
Address：地址，用来确定事件或请求需要分发到哪些Handle，如果使用的话，一般以ID来指定，默认不使用，也就是事件通知到所有连接到此EBus的Handler。

[[Evnt Bus Internal]]、[[Policies]]、[[BusImpl]]中已经基本介绍了EBus的底层机制，这里是EBus的最终实现。
# EBusTraits
定义EBus中必要的数据类型信息，用于配置EBus的行为。其中的几乎所有信息都会提供给[[BusImpl#EBusImplTraits]]，这里就不完整列出它的所有数据成员，
```cpp
    struct EBusTraits
    {
    protected:

        /**
         * Note - the destructor is intentionally not virtual to avoid adding vtable overhead to every EBusTraits derived class.
         */
        ~EBusTraits() = default;

    public:
        /**
         * Allocator used by the EBus.
         * The default setting is Internal EBusEnvironmentAllocator
         * EBus code stores their Context instances in static memory
         * Therfore the configured allocator must last as long as the EBus in a module
         */
        using AllocatorType = AZ::Internal::EBusEnvironmentAllocator;

        /**
         * Defines how many handlers can connect to an address on the EBus
         * and the order in which handlers at each address receive events.
         * For available settings, see AZ::EBusHandlerPolicy.
         * By default, an EBus supports any number of handlers.
         */
        static constexpr EBusHandlerPolicy HandlerPolicy = EBusHandlerPolicy::Multiple;

        /**
         * Defines how many addresses exist on the EBus.
         * For available settings, see AZ::EBusAddressPolicy.
         * By default, an EBus uses a single address.
         */
        static constexpr EBusAddressPolicy AddressPolicy = EBusAddressPolicy::Single;
        
        ...
    }
```
重点是EBusTraits中定义的几个锁，
```cpp
       /**
        * Template Lock Guard class to use during event dispatches.
        * By default it will use a scoped_lock, but IsLocklessDispatch=true will cause it to use a NullLockGuard.
        * The IsLocklessDispatch bool is there to defer evaluation of the LocklessDispatch constant
        * Otherwise the value above in EBusTraits.h is always used and not the value
        * that the derived trait class sets.
        */
        template <typename DispatchMutex, bool IsLocklessDispatch>
        using DispatchLockGuard = AZStd::conditional_t<IsLocklessDispatch, AZ::Internal::NullLockGuard<DispatchMutex>, AZStd::scoped_lock<DispatchMutex>>;
```
DispatchLockGuard用于在发布事件时加锁，默认使用scoped_lock。使用m_contextMutex互斥量。

```cpp
        /**
         * Template Lock Guard class to use during connection / disconnection.
         * By default it will use a unique_lock if the ContextMutex is anything but a NullMutex.
         * This can be overridden to provide a different locking policy with custom EBus MutexType settings.
         * Also, some specialized policies execute handler methods which can cause unnecessary delays holding
         * the context mutex, such as performing blocking waits. These methods must unlock the context mutex before
         * doing so to prevent deadlocks, especially when the wait is for an event in another thread which is trying
         * to connect to the same bus before it can complete.
         */
        template<typename ContextMutex>
        using ConnectLockGuard = AZStd::conditional_t<
            AZStd::is_same_v<ContextMutex, AZ::NullMutex>,
            AZ::Internal::NullLockGuard<ContextMutex>,
            AZStd::unique_lock<ContextMutex>>;
```
ConnectLockGuard，在Handle connect和disconnect时加锁，使用unique_lock。使用m_contextMutex互斥量。

```cpp
        /**
         * Template Lock Guard class to use for EBus bind calls.
         * By default it will use a scoped_lock.
         * This can be overridden to provide a different locking policy with custom EBus MutexType settings.
         */
        template<typename ContextMutex>
        using BindLockGuard = AZStd::scoped_lock<ContextMutex>;
```
调用Bind函数时加锁，使用m_contextMutex互斥量。

```cpp
        /**
         * Template Lock Guard class to use for EBus callstack tracking.
         * By default it will use a unique_lock if the ContextMutex is anything but a NullMutex.
         * This can be overridden to provide a different locking policy with custom EBus MutexType settings.
         */
        template<typename ContextMutex>
        using CallstackTrackerLockGuard = AZStd::conditional_t<
            AZStd::is_same_v<ContextMutex, AZ::NullMutex>,
            AZ::Internal::NullLockGuard<ContextMutex>,
            AZStd::unique_lock<ContextMutex>>;
```
获取Context时加锁，有时获取Context需要将当前线程的CallstackEntry缓存起来，期间需要加锁保证数据安全，使用m_contextMutex互斥量。

# Context
EBus核心部分，顾名思义是EBus中事件运行的上下文环境，里面保存了实现EBus功能所需的所有数据，多线程情况下ContextMutex用来保护共享数据。
Context的基类是ContextBase，
```cpp
        class ContextBase
        {
            template<class Context>
            friend struct AZ::EBusEnvironmentStoragePolicy;
            friend class AZ::EBusEnvironment;

        public:
            ContextBase();
            ContextBase(EBusEnvironment*);

            virtual ~ContextBase() {}

        private:

            int m_ebusEnvironmentTLSIndex;
            EBusEnvironmentGetterType m_ebusEnvironmentGetter;
        };
```
这里涉及到了Environment机制，也就是整个程序中的”环境变量“或全局变量，它决定了如何为Context分配内存和保存数据。在EBus中，决定此保存机制的是StoragePolicy,
```cpp
        /**
         * Specifies where EBus data is stored.
         * This drives how many instances of this EBus exist at runtime.
         * Available storage policies include the following:
         * - (Default) EBusEnvironmentStoragePolicy - %EBus data is stored
         * in the AZ::Environment. With this policy, a single %EBus instance
         * is shared across all modules (DLLs) that attach to the AZ::Environment. It also
         * supports multiple EBus environments.
         * - EBusGlobalStoragePolicy - %EBus data is stored in a global static variable.
         * With this policy, each module (DLL) has its own instance of the %EBus.
         * - EBusThreadLocalStoragePolicy - %EBus data is stored in a thread_local static
         * variable. With this policy, each thread has its own instance of the %EBus.
         *
         * \note Make sure you carefully consider the implication of switching this policy. If your code use EBusEnvironments and your storage policy is not
         * complaint in the best case you will cause contention and unintended communication across environments, separation is a goal of environments. In the worst
         * case when you have listeners, you can receive messages when you environment is NOT active, potentially causing all kinds of havoc especially if you execute
         * environments in parallel.
         */
        template <class Context>
        using StoragePolicy = EBusEnvironmentStoragePolicy<Context>;
```
Context的数据保存机制有三种策略：
- 全局数据，所有dll共享此数据。
- 全局数据，每个dll拥有自己的数据，一般使用头文件中的静态变量实现，在[[Policies#StoragePolicy]]实现。
- threal local数据，每个线程拥有自己的数据，线程间不能共享，用于只支持单线程的EBus，在[[Policies#EBusThreadLocalStoragePolicy]]实现。

```cpp
        class Context : public AZ::Internal::ContextBase
        {
            friend ThisType;
            friend Router;
        public:
            /**
             * The mutex type to use during broadcast/event dispatch.
             * When LocklessDispatch is set on the EBus and a NullMutex is supplied a shared_mutex is used to protect the context otherwise the supplied MutexType is used
             * The reason why a recursive_mutex is used in this situation, is that specifying LocklessDispatch is implies that the EBus will be used across multiple threads
             * @see EBusTraits::LocklessDispatch
             */
            using ContextMutexType = AZStd::conditional_t<BusTraits::LocklessDispatch && AZStd::is_same_v<MutexType, AZ::NullMutex>, AZStd::shared_mutex, MutexType>;

            /**
             * The scoped lock guard to use
             * during broadcast/event dispatch.
             * @see EBusTraits::LocklessDispatch
             */
            using DispatchLockGuard = DispatchLockGuardTemplate<ContextMutexType>;

            /**
            * The scoped lock guard to use during connection / disconnection.  Some specialized policies execute handler methods which
            * can cause unnecessary delays holding the context mutex or in some cases perform blocking waits and
            * must unlock the context mutex before doing so to prevent deadlock when the wait is for
            * an event in another thread which is trying to connect to the same bus before it can complete
            */
            using ConnectLockGuard = ConnectLockGuardTemplate<ContextMutexType>;

            /**
             * The scoped lock guard to use for bind calls.
             */
            using BindLockGuard = BindLockGuardTemplate<ContextMutexType>;

            /**
             * The scoped lock guard to use for callstack tracking.
             */
            using CallstackTrackerLockGuard = CallstackTrackerLockGuardTemplate<ContextMutexType>;

            BusesContainer          m_buses;         ///< The actual bus container, which is a static map for each bus type.
            ContextMutexType        m_contextMutex;  ///< Mutex to control access when modifying the context
            QueuePolicy             m_queue;
            RouterPolicy            m_routing;

            Context();
            Context(EBusEnvironment* environment);
            ~Context() override;

            // Disallow all copying/moving
            Context(const Context&) = delete;
            Context(Context&&) = delete;
            Context& operator=(const Context&) = delete;
            Context& operator=(Context&&) = delete;

        private:
            using CallstackEntryBase = AZ::Internal::CallstackEntryBase<Interface, Traits>;
            using CallstackEntryRoot = AZ::Internal::CallstackEntryRoot<Interface, Traits>;
            using CallstackEntryStorageType = AZ::Internal::EBusCallstackStorage<CallstackEntryBase, !AZStd::is_same_v<ContextMutexType, AZ::NullMutex>>;

            mutable AZStd::unordered_map<AZStd::native_thread_id_type, CallstackEntryRoot, AZStd::hash<AZStd::native_thread_id_type>, AZStd::equal_to<AZStd::native_thread_id_type>, AZ::Internal::EBusEnvironmentAllocator> m_callstackRoots;
            CallstackEntryStorageType s_callstack;    ///< Linked list of other bus calls to this bus on the stack, per thread if MutexType is defined
            AZStd::atomic_uint m_dispatches;   ///< Number of active dispatches in progress

            friend CallstackEntry;
        };
```
首先是ContextMutexType，如果使用锁的话会使用shared_mutex互斥量（但其实后面没有用到shared_lock，这是优化方向？），并且后面定义了EBus所需的几种说，可以说EBus的多线程支持就在这里。

然后是4个重点数据结构：
```cpp
            BusesContainer          m_buses;         ///< The actual bus container, which is a static map for each bus type.
            ContextMutexType        m_contextMutex;  ///< Mutex to control access when modifying the context
            QueuePolicy             m_queue;
            RouterPolicy            m_routing;
```
这些数据类型在[[Evnt Bus Internal]]和[[Policies]]中都介绍过。

最后是两个私有变量，m_callstackRoots是一个unordered_map，保存线程id和CallstackEntryRoot的映射，这样就可以为每一个线程保存一份CallstackEntry列表。
s_callstack记录当前线程的调用栈。
m_dispatches是一个原子uint变量，记录当前正在执行的事件数量。

再看Context的构造和析构函数：
```cpp
    //=========================================================================
    // Context::Context
    //=========================================================================
    template<class Interface, class Traits>
    EBus<Interface, Traits>::Context::Context()
        : m_dispatches(0)
    {
        s_callstack = nullptr;
    }

    //=========================================================================
    // Context::Context
    //=========================================================================
    template<class Interface, class Traits>
    EBus<Interface, Traits>::Context::Context(EBusEnvironment* environment)
        : AZ::Internal::ContextBase(environment)
        , m_dispatches(0)
    {
        s_callstack = nullptr;
    }

    template <class Interface, class Traits>
    EBus<Interface, Traits>::Context::~Context()
    {
        // Clear the callstack in this thread. It is expected that most buses will be lifetime managed
        // by the thread that creates them (almost certainly the main thread). This allows a bus
        // to be re-entrant within the same main thread (useful for unit tests and code reloading).
        s_callstack = nullptr;
    }
```
对m_dispatches和s_callstack进行初始化和析构。

# EBus
## 定义
```cpp
    template<class Interface, class BusTraits = Interface>
    class EBus
        : public BusInternal::EBusImpl<AZ::EBus<Interface, BusTraits>, BusInternal::EBusImplTraits<Interface, BusTraits>, typename BusTraits::BusIdType>
    {
    public:
        class Context;

        /**
         * Contains data about EBusTraits.
         */
        using ImplTraits = BusInternal::EBusImplTraits<Interface, BusTraits>;

        /**
         * Represents an %EBus with certain broadcast, event, and routing functionality.
         */
        using BaseImpl = BusInternal::EBusImpl<AZ::EBus<Interface, BusTraits>, BusInternal::EBusImplTraits<Interface, BusTraits>, typename BusTraits::BusIdType>;

        /**
         * Alias for EBusTraits.
         */
        using Traits = typename ImplTraits::Traits;
    ...
}
```
EBus完全继承于[[BusImpl#EBusImpl]]第一个模板参数为EBus（当前类），各种功能的实现依赖EBus的数据，第二个模板参数为EBusImplTraits，相当于BusTraits，包含EBus内数据类型的定义。
在EBus中通过ImplTraits获取了各个数据类型的定义，所以Traits实际是按照BusTraits -> EBusImplTraits -> EBus传递的。
关于Traits中的数据在EBusImplTraits中有介绍。

## Context的创建和获取
```cpp
        /**
         * Returns the global bus data (if it was created).
         * Depending on the storage policy, there might be one or multiple instances
         * of the bus data.
         * @return A reference to the bus context.
         */
        static Context* GetContext(bool trackCallstack=true);

        /**
         * Returns the global bus data. Creates it if it wasn't already created.
         * Depending on the storage policy, there might be one or multiple instances
         * of the bus data.
         * @return A reference to the bus context.
         */
        static Context& GetOrCreateContext(bool trackCallstack=true);
```
根据指定的StoragePolicy获取或创建一个Context。
具体的实现：
```cpp
    //=========================================================================
    // GetContext
    //=========================================================================
    template<class Interface, class Traits>
    typename EBus<Interface, Traits>::Context* EBus<Interface, Traits>::GetContext(bool trackCallstack)
    {
        Context* context = StoragePolicy::Get();
        if (trackCallstack && context && !context->s_callstack)
        {
            // Cache the callstack root into this thread/dll. Even though s_callstack is thread-local, we need a mutex lock
            // for the modifications to m_callstackRoots, which is NOT thread-local.
            typename Context::CallstackTrackerLockGuard lock(context->m_contextMutex);
            context->s_callstack = &context->m_callstackRoots[AZStd::this_thread::get_id().m_id];
        }
        return context;
    }

    //=========================================================================
    // GetContext
    //=========================================================================
    template<class Interface, class Traits>
    typename EBus<Interface, Traits>::Context& EBus<Interface, Traits>::GetOrCreateContext(bool trackCallstack)
    {
        Context& context = StoragePolicy::GetOrCreate();
        if (trackCallstack && !context.s_callstack)
        {
            // Cache the callstack root into this thread/dll. Even though s_callstack is thread-local, we need a mutex lock
            // for the modifications to m_callstackRoots, which is NOT thread-local.
            typename Context::CallstackTrackerLockGuard lock(context.m_contextMutex);
            context.s_callstack = &context.m_callstackRoots[AZStd::this_thread::get_id().m_id];
        }
        return context;
    }
```
StoragePolicy会使用Get（GetOrCreate）创建出一个Context变量，然后根据trackCallstack参数（默认为true），确定是否追踪EBus的调用栈，也就是将Context的m_callstackRoots中保存的Callstack链表更新到s_callstack中，m_callstackRoots是一个map，如果没有这条记录，\[\]运算符会插入它。这样对s_callstack的操作就只影响当前线程。
## Connect和Disconnect函数
将一个Handle连接（断连）至此EBus，
```cpp
    //=========================================================================
    // Connect
    //=========================================================================
    template<class Interface, class Traits>
    inline void EBus<Interface, Traits>::Connect(HandlerNode& handler, const BusIdType& id)
    {
        Context& context = GetOrCreateContext();
        // scoped lock guard in case of exception / other odd situation
        // Context mutex is separate from the Dispatch lock guard and therefore this is safe to lock this mutex while in the middle of a dispatch
        ConnectLockGuard lock(context.m_contextMutex);
        ConnectInternal(context, handler, lock, id);
    }

    //=========================================================================
    // Disconnect
    //=========================================================================
    template<class Interface, class Traits>
    inline void EBus<Interface, Traits>::Disconnect(HandlerNode& handler)
    {
        // To call Disconnect() from a message while being thread safe, you need to make sure the context.m_contextMutex is AZStd::recursive_mutex. Otherwise, a deadlock will occur.
        if (Context* context = GetContext())
        {
            // scoped lock guard in case of exception / other odd situation
            ConnectLockGuard lock(context->m_contextMutex);
            DisconnectInternal(*context, handler);
        }
    }
```
连接时GetOrCreate Context，加锁，然后ConnectInternal进行连接，断连时同样加锁，调用DisconnectInternal。
注释中提到，Connect加锁是线程安全的（任何时候都需要先连接才能收到消息），而Disconnect加锁需要注意，如果是在一个消息处理期间Disconnect（比如处理一个消息后将此Handler断开），需要保证m_contextMutex是一个recursive_mutex，因为此时ConnectLockGuard正在dispatch中使用。
与Handler的Connect和Disconnect比较，主要流程是相似的，获取contect，加锁，然后调用ConnectInternal或DisconnectInternal，这也说明一个Handler与EBus连接时，可以通过Handler建立，也可以通过EBus建立，关于Handler的Connect函数，在[[Evnt Bus Internal#Handles]]中有介绍。
它们最终都会使用ConnectInternal或DisconnectInternal进行真正的连接和断连，这两个函数实现如下，
```cpp
    //=========================================================================
    // ConnectInternal
    //=========================================================================
    template<class Interface, class Traits>
    inline void EBus<Interface, Traits>::ConnectInternal(Context& context, HandlerNode& handler, ConnectLockGuard& contextLock, const BusIdType& id)
    {
        // To call this while executing a message, you need to make sure this mutex is AZStd::recursive_mutex. Otherwise, a deadlock will occur.
        AZ_Assert(!Traits::LocklessDispatch || !IsInDispatch(&context), "It is not safe to connect during dispatch on a lockless dispatch EBus");

        // Do the actual connection
        context.m_buses.Connect(handler, id);

        BusPtr ptr;
        if constexpr (EBus::HasId)
        {
            ptr = handler.m_holder;
        }
        CallstackEntry entry(&context, &id);
        ConnectionPolicy::Connect(ptr, context, handler, contextLock, id);
    }


    //=========================================================================
    // DisconnectInternal
    //=========================================================================
    template<class Interface, class Traits>
    inline void EBus<Interface, Traits>::DisconnectInternal(Context& context, HandlerNode& handler)
    {
        // To call this while executing a message, you need to make sure this mutex is AZStd::recursive_mutex. Otherwise, a deadlock will occur.
        AZ_Assert(!Traits::LocklessDispatch || !IsInDispatch(&context), "It is not safe to disconnect during dispatch on a lockless dispatch EBus");

        auto callstack = context.s_callstack->m_prev;
        if (callstack)
        {
            callstack->OnRemoveHandler(handler);
        }

        BusPtr ptr;
        if constexpr (EBus::HasId)
        {
            ptr = handler.m_holder;
        }
        ConnectionPolicy::Disconnect(context, handler, ptr);

        CallstackEntry entry(&context, nullptr);

        // Do the actual disconnection
        context.m_buses.Disconnect(handler);

        if (callstack)
        {
            callstack->OnPostRemoveHandler();
        }

        handler = nullptr;
    }
```
两个函数都做了断言，防止在不加锁时多个线程调用此函数出现错误。
对于ConnectInternal，真正的连接操作是context.m_buses.Connect(handler, id)，即使用[[Evnt Bus Internal#EBusContainer]]中的Connect函数，它们会根据不同的EBus类型来决定如何将此Handler保存到EBus中。后面会定义一个CallstackEntry，这是一个局部变量，创建时插入到调用栈s_callstack，销毁时从s_callstack中删除，主要是维护m_dispatches。
DisconnectInternal更加复杂一些，它会调用s_callstack中所有节点的OnRemoveHandler函数，注意这里的OnRemoveHandler是会递归调用的，见[[Evnt Bus Internal#CallstackEntry]]，然后同样使用context.m_buses.Disconnect执行真正的断连，这里同样创建CallstackEntry维护m_dispatches，然后调用s_callstack所有节点的OnPostRemoveHandler函数，最后把handler置为空指针。
这里DisconnectInternal时会调用OnRemoveHandler和OnPostRemoveHandler函数，是为了处理在dispatch时发生断连的情况，[[Evnt Bus Internal#MakeDisconnectFixer]]中介绍过，发生断连时可以通过此机制在OnRemoveHandler和OnPostRemoveHandler函数进行对应的处理，比如跳过这个断连的Handler。
##  关于Context的m_dispatches
从上面的功能中可以看到m_dispatches记录了一次事件触发中被触发的Handler的数量，并且，Handler的连接和断连也被认为是一次触发，也就是说，当一个Handler连接状态改变时，EBus认为这个Handler“响应了一次事件”。因为事件触发和连接状态改变在多线程中都是需要同步的。
## 一些状态查询函数
### GetTotalNumOfEventHandlers
```cpp
    //=========================================================================
    // GetTotalNumOfEventHandlers
    //=========================================================================
    template<class Interface, class Traits>
    size_t  EBus<Interface, Traits>::GetTotalNumOfEventHandlers()
    {
        size_t size = 0;
        BaseImpl::EnumerateHandlers([&size](Interface*)
        {
            ++size;
            return true;
        });
        return size;
    }
```
统计已连接的Handler数量。
### HasHandlers
```cpp
    //=========================================================================
    // HasHandlers
    //=========================================================================
    template<class Interface, class Traits>
    inline bool EBus<Interface, Traits>::HasHandlers()
    {
        bool hasHandlers = false;
        auto findFirstHandler = [&hasHandlers](InterfaceType*)
        {
            hasHandlers = true;
            return false;
        };
        BaseImpl::EnumerateHandlers(findFirstHandler);
        return hasHandlers;
    }

    //=========================================================================
    // HasHandlers
    //=========================================================================
    template<class Interface, class Traits>
    inline bool EBus<Interface, Traits>::HasHandlers(const BusIdType& id)
    {
        return BaseImpl::FindFirstHandler(id) != nullptr;
    }

    //=========================================================================
    // HasHandlers
    //=========================================================================
    template<class Interface, class Traits>
    inline bool EBus<Interface, Traits>::HasHandlers(const BusPtr& ptr)
    {
        return BaseImpl::FindFirstHandler(ptr) != nullptr;
    }
```
查询是否有Handler连接，查询某个id的Handler是否来连接。

### IsInDispatch 和 IsInDispatchThisThread
```cpp
    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::IsInDispatch(Context* context)
    {
        return context != nullptr && context->m_dispatches > 0;
    }

    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::IsInDispatchThisThread(Context* context)
    {
        return context != nullptr && context->s_callstack != nullptr
            && context->s_callstack->m_prev != nullptr;
    }
```
查询是否有事件dispatche，或者此线程上是否有事件dispatche。
### GetCurrentBusId
```cpp
    //=========================================================================
    // GetCurrentBusId
    //=========================================================================
    template<class Interface, class Traits>
    const typename EBus<Interface, Traits>::BusIdType * EBus<Interface, Traits>::GetCurrentBusId()
    {
        Context* context = GetContext();
        if (IsInDispatchThisThread(context))
        {
            return context->s_callstack->m_prev->m_busId;
        }
        return nullptr;
    }
```
查询当前线程是否有Handler在处理事件，如果有，返回它的BusId。
### HasReentrantEBusUseThisThread
```cpp
    //=========================================================================
    // HasReentrantEBusUseThisThread
    //=========================================================================
    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::HasReentrantEBusUseThisThread(const BusIdType* busId)
    {
        Context* context = GetContext();

        if (busId && IsInDispatchThisThread(context))
        {
            bool busIdInCallstack = false;

            // If we're in a dispatch, callstack->m_prev contains the entry for the current bus call. Start the search for the given
            // bus ID and look upwards. If we find the given ID more than once in the callstack, we've got a reentrant call.
            for (auto callstackEntry = context->s_callstack->m_prev; callstackEntry != nullptr; callstackEntry = callstackEntry->m_prev)
            {
                if ((*busId) == (*callstackEntry->m_busId))
                {
                    if (busIdInCallstack)
                    {
                        return true;
                    }

                    busIdInCallstack = true;
                }
            }
        }

        return false;
    }
```
判断在EBus dispatch过程中此busId上连接的Handler是否被触发了多次，这可以用来判断发生死锁的可能性，因为事件的触发（包括连接和断连）是线程安全的，发生此种情况可能是Handler在一次事件处理过程中又触发了此EBus上的事件，这有可能导致递归调用或死锁。
### Router相关函数
```cpp
    //=========================================================================
    // SetRouterProcessingState
    //=========================================================================
    template<class Interface, class Traits>
    void EBus<Interface, Traits>::SetRouterProcessingState(RouterProcessingState state)
    {
        Context* context = GetContext();
        if (IsInDispatch(context))
        {
            context->s_callstack->m_prev->SetRouterProcessingState(state);
        }
    }

    //=========================================================================
    // IsRoutingQueuedEvent
    //=========================================================================
    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::IsRoutingQueuedEvent()
    {
        Context* context = GetContext();
        if (IsInDispatch(context))
        {
            return context->s_callstack->m_prev->IsRoutingQueuedEvent();
        }

        return false;
    }

    //=========================================================================
    // IsRoutingReverseEvent
    //=========================================================================
    template<class Interface, class Traits>
    bool EBus<Interface, Traits>::IsRoutingReverseEvent()
    {
        Context* context = GetContext();
        if (IsInDispatch(context))
        {
            return context->s_callstack->m_prev->IsRoutingReverseEvent();
        }

        return false;
    }
```
设置和查询s_callstack第一个节点Router状态，其中的含义在[[Policies#EBusRouterPolicy]]中有介绍。
# *ForwardEvent功能*
这是EBus中的事件转发功能，可以将一个EBus上的事件发送到另一个EBus。EBus中整个Router模块都是为此功能设计的，但是在引擎中并没有看到此功能的应用，只在单元测试里验证了一下。这个功能的设计是否有问题？EBus的设计初衷应该就是需要将各个EBus上的事件分开不要相互干扰，即使需要相互触发，也应该交由具体的Hanlder实现，EBus提供此机制看起来有点画蛇添足。
这里就简单了解一下这个功能，
## EBusRouterQueueEventForwarder
```cpp
        template <class EBus, class TargetEBus, class BusIdType>
        struct EBusRouterQueueEventForwarder
        {
            static_assert((AZStd::is_same<BusIdType, typename EBus::BusIdType>::value), "Routers may only route between buses with the same id/traits");
            static_assert((AZStd::is_same<BusIdType, typename TargetEBus::BusIdType>::value), "Routers may only route between buses with the same id/traits");

            template<class Event, class... Args>
            static void ForwardEvent(Event event, Args&&... args);

            template <class Event, class... Args>
            static void ForwardEventResult(Event event, Args&&... args);
        };
```
事件转发器，约束EBus和TargetEBus使用的BusIdType是相同的，并包含两个转发函数。
## ForwardEvent
```cpp
        //////////////////////////////////////////////////////////////////////////
        template <class EBus, class TargetEBus, class BusIdType>
        template<class Event, class... Args>
        void EBusRouterQueueEventForwarder<EBus, TargetEBus, BusIdType>::ForwardEvent(Event event, Args&&... args)
        {
            const BusIdType* busId = EBus::GetCurrentBusId();
            if (busId == nullptr)
            {
                // Broadcast
                if (EBus::IsRoutingQueuedEvent())
                {
                    // Queue broadcast
                    if (EBus::IsRoutingReverseEvent())
                    {
                        // Queue broadcast reverse
                        TargetEBus::QueueBroadcastReverse(event, args...);
                    }
                    else
                    {
                        // Queue broadcast forward
                        TargetEBus::QueueBroadcast(event, args...);
                    }
                }
                else
                {
                    // In place broadcast
                    if (EBus::IsRoutingReverseEvent())
                    {
                        // In place broadcast reverse
                        TargetEBus::BroadcastReverse(event, args...);
                    }
                    else
                    {
                        // In place broadcast forward
                        TargetEBus::Broadcast(event, args...);
                    }
                }
            }
            else
            {
                // Event with an ID
                if (EBus::IsRoutingQueuedEvent())
                {
                    // Queue event
                    if (EBus::IsRoutingReverseEvent())
                    {
                        // Queue event reverse
                        TargetEBus::QueueEventReverse(*busId, event, args...);
                    }
                    else
                    {
                        // Queue event forward
                        TargetEBus::QueueEvent(*busId, event, args...);
                    }
                }
                else
                {
                    // In place event
                    if (EBus::IsRoutingReverseEvent())
                    {
                        // In place event reverse
                        TargetEBus::EventReverse(*busId, event, args...);
                    }
                    else
                    {
                        // In place event forward
                        TargetEBus::Event(*busId, event, args...);
                    }
                }
            }
        }
```
获取当前正在dispatch的busId，如果busId为空，执行Broadcast逻辑，如果busId存在，执行Event逻辑，然后根据IsRoutingQueuedEvent配置决定是否立即转发，还是将其加入执行队列，再根据IsRoutingReverseEvent决定是顺序dispatch还是倒序。
IsRoutingQueuedEvent和IsRoutingReverseEvent在上面[[EBus#Router相关函数]]中可以配置。更多关于Router的介绍在[[Policies#EBusQueuePolicy]]。
所谓的转发，就是直接调用TargetEBus::Event或者TargetEBus::QueueEvent执行该事件。
此外它还有各种特化版本，比如BusIdType为NullBusId，这样就只有Broadcast；或者不支持QueueEvent的版本，这里就不一一列举。
## EBusRouterForwarderHelper
```cpp
        template<class EBus, class TargetEBus, bool allowQueueing = EBus::EnableEventQueue>
        struct EBusRouterForwarderHelper
        {
            template<class Event, class... Args>
            static void ForwardEvent(Event event, Args&&... args)
            {
                EBusRouterQueueEventForwarder<EBus, TargetEBus, typename EBus::BusIdType>::ForwardEvent(event, args...);
            }

            template<class Result, class Event, class... Args>
            static void ForwardEventResult(Result&, Event, Args&&...)
            {

            }
        };
```
对EBusRouterQueueEventForwarder进行简单的包装。
## EBusRouter
```cpp
        /**
        * EBus router helper class. Inherit from this class the same way
        * you do with EBus::Handlers, to implement router functionality.
        *
        */
        template<class EBus>
        class EBusRouter
            : public EBus::InterfaceType
        {
            EBusRouterNode<typename EBus::InterfaceType> m_routerNode;
            bool m_isConnected;
        public:
            EBusRouter();
            virtual ~EBusRouter();

            void BusRouterConnect(int order = 0);

            void BusRouterDisconnect();

            bool BusRouterIsConnected() const;

            template<class TargetEBus, class Event, class... Args>
            static void ForwardEvent(Event event, Args&&... args);

            template<class Result, class TargetEBus, class Event, class... Args>
            static void ForwardEventResult(Result& result, Event event, Args&&... args);
        };
```
实现转发功能的类，同时记录了连接状态。
构造函数，
```cpp
        template<class EBus>
        EBusRouter<EBus>::EBusRouter()
            : m_isConnected(false)
        {
            m_routerNode.m_handler = this;
        }
```
m_routerNode初始化为当前对象，关于EBusRouterNode类型在[[Policies#EBusRouterNode]]中有介绍。
BusRouterConnect函数，
```cpp
        //////////////////////////////////////////////////////////////////////////
        template<class EBus>
        void EBusRouter<EBus>::BusRouterConnect(int order)
        {
            if (!m_isConnected)
            {
                m_routerNode.m_order = order;
                auto& context = EBus::GetOrCreateContext();
                // We could support connection/disconnection while routing a message, but it would require a call to a fix
                // function because there is already a stack entry. This is typically not a good pattern because routers are
                // executed often. If time is not important to you, you can always queue the connect/disconnect functions
                // on the TickBus or another safe bus.
                AZ_Assert(context.s_callstack->m_prev == nullptr, "Current we don't allow router connect while in a message on the bus!");
                {
                    AZStd::scoped_lock<decltype(context.m_contextMutex)> lock(context.m_contextMutex);
                    context.m_routing.m_routers.insert(&m_routerNode);
                }
                m_isConnected = true;
            }
        }
```
order指定了排序顺序，主要是将m_routerNode插入到了context.m_routing.m_routers中，它是一个intrusive_multiset。结合上面的m_routerNode初始化为this对象，这就相当于将当前EBusRouter放到了context.m_routing.m_routers中。
ForwardEvent函数：
```cpp
        //////////////////////////////////////////////////////////////////////////
        template<class EBus>
        template<class TargetEBus, class Event, class... Args>
        void EBusRouter<EBus>::ForwardEvent(Event event, Args&&... args)
        {
            EBusRouterForwarderHelper<EBus, TargetEBus>::ForwardEvent(event, args...);
        }
```
调用上面EBusRouterForwarderHelper的ForwardEvent函数。
如果外部的对象向作为此EBus可转发的Router，就来继承这个EBusRouter并定义Interface的响应函数。

