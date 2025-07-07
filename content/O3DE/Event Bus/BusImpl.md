这个模块是EBus的具体实现，在这里将EBus需要各个模块的功能组合起来，实现一些完整的EBus功能。
# 一些预定义的数据结构
## NullMutex
```cpp
    /**
     * A dummy mutex that performs no locking.
     * EBuses that do not support multithreading use this mutex
     * as their EBusTraits::MutexType.
     */
    struct NullMutex
    {
        void lock() {}
        bool try_lock() { return true; }
        void unlock() {}
    };
```
无锁的EBus使用的Mutex。
## NullBusId
```cpp
    /**
     * Indicates that EBusTraits::BusIdType is not set.
     * EBuses with multiple addresses must set the EBusTraits::BusIdType.
     */
    struct NullBusId
    {
        NullBusId() {};
        NullBusId(int) {};
    };

    /// @cond EXCLUDE_DOCS
    inline bool operator==(const NullBusId&, const NullBusId&) { return true; }
    inline bool operator!=(const NullBusId&, const NullBusId&) { return false; }
    /// @endcond
    
    /**
     * Indicates that EBusTraits::BusIdOrderCompare is not set.
     * EBuses with ordered address IDs must specify a function for
     * EBusTraits::BusIdOrderCompare.
     */
    struct NullBusIdCompare;
```
不使用Address功能的EBus使用的BusId和它的比较函数。
## NullLockGuard
```cpp
    namespace Internal
    {
        // Lock guard used when there is a NullMutex on a bus, or during dispatch
        // on a bus which supports LocklessDispatch.
        template <class Lock>
        struct NullLockGuard
        {
            explicit NullLockGuard(Lock&) {}
            NullLockGuard(Lock&, AZStd::adopt_lock_t) {}

            void lock() {}
            bool try_lock() { return true; }
            void unlock() {}
        };
    }
```
不使用Mutex时，没有任何功能的LockGuard。
# EBusImplTraits
包含了EBusTraits的内部的各种数据结构。
```cpp
        template <class Interface, class BusTraits>
        struct EBusImplTraits
        {
            /**
             * Properties that you use to configure an EBus.
             * For more information, see EBusTraits.
             */
            using Traits = BusTraits;

            /**
             * Allocator used by the EBus.
             * The default setting is AZStd::allocator, which uses AZ::SystemAllocator.
             */
            using AllocatorType = typename Traits::AllocatorType;

            /**
             * The class that defines the interface of the EBus.
             */
            using InterfaceType = Interface;

            /**
             * The events defined by the EBus interface.
             */
            using Events = Interface;

            /**
             * The type of ID that is used to address the EBus.
             * Used only when the address policy is AZ::EBusAddressPolicy::ById
             * or AZ::EBusAddressPolicy::ByIdAndOrdered.
             * The type must support `AZStd::hash<ID>` and
             * `bool operator==(const ID&, const ID&)`.
             */
            using BusIdType = typename Traits::BusIdType;

            /**
             * Sorting function for EBus address IDs.
             * Used only when the AddressPolicy is AZ::EBusAddressPolicy::ByIdAndOrdered.
             * If an event is dispatched without an ID, this function determines
             * the order in which each address receives the event.
             *
             * The following example shows a sorting function that meets these requirements.
             * @code{.cpp}
             * using BusIdOrderCompare = AZStd::less<BusIdType>; // Lesser IDs first.
             * @endcode
             */
            using BusIdOrderCompare = typename Traits::BusIdOrderCompare;

            /**
             * Locking primitive that is used when connecting handlers to the EBus or executing events.
             * By default, all access is assumed to be single threaded and no locking occurs.
             * For multithreaded access, specify a mutex of the following type.
             * - For simple multithreaded cases, use AZStd::mutex.
             * - For multithreaded cases where an event handler sends a new event on the same bus
             *   or connects/disconnects while handling an event on the same bus, use AZStd::recursive_mutex.
             */
            using MutexType = typename Traits::MutexType;

            /**
             * Contains all of the addresses on the EBus.
             */
            using BusesContainer = AZ::Internal::EBusContainer<Interface, Traits>;

            /**
             * Locking primitive that is used when executing events in the event queue.
             */
            using EventQueueMutexType = AZStd::conditional_t<AZStd::is_same<typename Traits::EventQueueMutexType, NullMutex>::value, // if EventQueueMutexType==NullMutex use MutexType otherwise EventQueueMutexType
                MutexType, typename Traits::EventQueueMutexType>;

            /**
             * Pointer to an address on the bus.
             */
            using BusPtr = typename BusesContainer::BusPtr;

            /**
             * Pointer to a handler node.
             */
            using HandlerNode = typename BusesContainer::HandlerNode;

            /**
             * Specifies whether the EBus supports an event queue.
             * You can use the event queue to execute events at a later time.
             * To execute the queued events, you must call
             * `<BusName>::ExecuteQueuedEvents()`.
             * By default, the event queue is disabled.
             */
            static constexpr bool EnableEventQueue = Traits::EnableEventQueue;
            static constexpr bool EventQueueingActiveByDefault = Traits::EventQueueingActiveByDefault;
            static constexpr bool EnableQueuedReferences = Traits::EnableQueuedReferences;

            /**
             * True if the EBus supports more than one address. Otherwise, false.
             */
            static constexpr bool HasId = Traits::AddressPolicy != EBusAddressPolicy::Single;

            /**
             * The following Lock Guard classes are exposed so that it's possible to override them with custom lock/unlock functionality
             * when using custom types for the EBus MutexType.
             */

            /**
            * Template Lock Guard class that wraps around the Mutex
            * The EBus uses for Dispatching Events.
            * This is not the EBus Context Mutex if LocklessDispatch is true
            */
            template <typename DispatchMutex>
            using DispatchLockGuard = typename Traits::template DispatchLockGuard<DispatchMutex, Traits::LocklessDispatch>;

            /**
             * Template Lock Guard class that protects connection / disconnection. 
             */
            template<typename ContextMutex>
            using ConnectLockGuard = typename Traits::template ConnectLockGuard<ContextMutex>;

            /**
             * Template Lock Guard class that protects bind calls. 
             */
            template<typename ContextMutex>
            using BindLockGuard = typename Traits::template BindLockGuard<ContextMutex>;

            /**
             * Template Lock Guard class that protects callstack tracking.
             */
            template<typename ContextMutex>
            using CallstackTrackerLockGuard = typename Traits::template CallstackTrackerLockGuard<ContextMutex>;
        };
```
traits中定义了EBus中的各种数据类型，它们大都从模板参数BusTraits中获得。
# EBusEventer
```cpp
        /**
         * Dispatches events to handlers that are connected to a specific address on an EBus.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusEventer
        {
            /**
             * The type of ID that is used to address the EBus.
             * Used only when the address policy is AZ::EBusAddressPolicy::ById
             * or AZ::EBusAddressPolicy::ByIdAndOrdered.
             * The type must support `AZStd::hash<ID>` and
             * `bool operator==(const ID&, const ID&)`.
             */
            using BusIdType = typename Traits::BusIdType;

            /**
             * Pointer to an address on the bus.
             */
            using BusPtr = typename Traits::BusPtr;

            /**
             * An event handler that can be attached to multiple addresses.
             */
            using MultiHandler = typename Traits::BusesContainer::MultiHandler;

            /**
             * Acquires a pointer to an EBus address.
             * @param[out] ptr A pointer that will point to the specified address
             * on the EBus. An address lookup can be avoided by calling Event() with
             * this pointer rather than by passing an ID, but that is only recommended
             * for performance-critical code.
             * @param id The ID of the EBus address that the pointer will be bound to.
             */
            static void Bind(BusPtr& ptr, const BusIdType& id);
        };
```
用于分发事件到连接至某个特定地址的EBus的handlers。但是这里只有一个Bind函数，具体的定义如下：
```cpp
        template <class Bus, class Traits>
        inline void EBusEventer<Bus, Traits>::Bind(BusPtr& ptr, const BusIdType& id)
        {
            auto& context = Bus::GetOrCreateContext();
            typename Traits::template BindLockGuard<decltype(context.m_contextMutex)> lock(context.m_contextMutex);
            context.m_buses.Bind(ptr, id);
        }
```
context.m_buses是一个EBusContainer，所以这里使用的是EBusContainer的Bind函数，它的功能是将这个id绑定到一个BusPtr，可以理解为给EBus赋予一个id。
这个struct实际的功能感觉和描述并不相符，它里面没有任何分发事件的行为，只有一个bind函数。
# EBusEventEnumerator
用于寻找连接在某个EBus上的handler。
```cpp
        /**
         * Provides functionality that requires enumerating over handlers that are connected
         * to an EBus. It can enumerate over all handlers or just the handlers that are connected
         * to a specific address on an EBus.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusEventEnumerator
        {
            /**
             * The type of ID that is used to address the EBus.
             * Used only when the address policy is AZ::EBusAddressPolicy::ById
             * or AZ::EBusAddressPolicy::ByIdAndOrdered.
             * The type must support `AZStd::hash<ID>` and
             * `bool operator==(const ID&, const ID&)`.
             */
            using BusIdType = typename Traits::BusIdType;

            /**
             * Pointer to an address on the bus.
             */
            using BusPtr = typename Traits::BusPtr;

            /**
             * Finds the first handler that is connected to a specific address on the EBus.
             * This function is only for special cases where you know that a particular
             * component's handler is guaranteed to exist.
             * Even if the returned pointer is valid (not null), it might point to a handler
             * that was deleted. Prefer dispatching events using EBusEventer.
             * @param id        Address ID.
             * @return          A pointer to the first handler on the EBus, even if the handler
             *                  was deleted.
             */
            static typename Traits::InterfaceType* FindFirstHandler(const BusIdType& id);

            /**
             * Finds the first handler at a cached address on the EBus.
             * This function is only for special cases where you know that a particular
             * component's handler is guaranteed to exist.
             * Even if the returned pointer is valid (not null), it might point to a handler
             * that was deleted. Prefer dispatching events using EBusEventer.
             * @param ptr       Cached address ID.
             * @return          A pointer to the first handler on the specified EBus address,
             *                  even if the handler was deleted.
             */
            static typename Traits::InterfaceType* FindFirstHandler(const BusPtr& ptr);

            /**
             * Returns the total number of event handlers that are connected to a specific
             * address on the EBus.
             * @param id    Address ID.
             * @return      The total number of handlers that are connected to the EBus address.
             */
            static size_t GetNumOfEventHandlers(const BusIdType& id);
        };

```
同样通过Traits来获取类型信息，其中有3个静态函数，两个FindFirstHandler功能是相同的，通过BusId/BusPtr来找到第一个连接在它们上面的Handler。GetNumOfEventHandlers计算连接到此EBus上的Handler上数量（它没有BusPtr版本，只有BusId版本）。
具体实现：
```cpp
        template <class Bus, class Traits>
        typename Traits::InterfaceType * EBusEventEnumerator<Bus, Traits>::FindFirstHandler(const BusIdType& id)
        {
            typename Traits::InterfaceType* result = nullptr;
            Bus::EnumerateHandlersId(id, [&result](typename Traits::InterfaceType* handler)
            {
                result = handler;
                return false;
            });
            return result;
        }
```
这里调用了EnumerateHandlersId函数，在BusContainer中出现过，它的功能是在该id的EBus上执行一个函数，就像for_each一样，这里的函数就是将它的hander赋值给result，并且第一次就返回false，实现拿到第一个handler就停止的目的。
GetNumOfEventHandlers实现：
```cpp
        template <class Bus, class Traits>
        size_t EBusEventEnumerator<Bus, Traits>::GetNumOfEventHandlers(const BusIdType& id)
        {
            size_t size = 0;
            Bus::EnumerateHandlersId(id, [&size](typename Bus::InterfaceType*)
            {
                ++size;
                return true;
            });
            return size;
        }
```
同样使用了EnumerateHandlersId函数，对所有handler进行计数。
# EBusBroadcaster
用于分发事件至所有handler，与EBusEventer向对，EBusEventer只分发至某个EBus上的handler。
```cpp
        /**
         * Dispatches an event to all handlers that are connected to an EBus.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusBroadcaster
        {
            /**
             * An event handler that can be attached to only one address at a time.
             */
            using Handler = typename Traits::BusesContainer::Handler;
        };
```
这里没有任何实现，看起来EBusBroadcaster和EBusEventer都不具备它们应有的功能，对应的功能位于EBusContainer中，这是未实现的设计？
# EBusNullQueue
不能实现排序功能的EBus使用的数据结构。
```cpp
        /**
         * Data type that is used when an EBus doesn't support queuing.
         */
        struct EBusNullQueue
        {
        };
```
里面什么都没有。
# EBusBroadcastQueue
EBusBroadcaster的升级版，按顺序广播一个事件至所有handler。
```cpp

        /**
         * EBus functionality related to the queuing of events and functions.
         * This is specifically for queuing events and functions that will
         * be broadcast to all handlers on the EBus.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusBroadcastQueue
        {
            /**
             * Executes queued events and functions.
             * Execution will occur on the thread that calls this function.
             * @see QueueBroadcast(), EBusEventQueue::QueueEvent(), QueueFunction(), ClearQueuedEvents()
             */
            static void ExecuteQueuedEvents()
            {
                if (auto* context = Bus::GetContext())
                {
                    context->m_queue.Execute();
                }
            }

            /**
             * Clears the queue without calling events or functions.
             * Use in situations where memory must be freed immediately, such as shutdown.
             * Use with care. Cleared queued events will never be executed, and those events might have been expected.
             */
            static void ClearQueuedEvents()
            {
                if (auto* context = Bus::GetContext(false))
                {
                    context->m_queue.Clear();
                }
            }

            static size_t QueuedEventCount()
            {
                if (auto* context = Bus::GetContext(false))
                {
                    return context->m_queue.Count();
                }
                return 0;
            }

            /**
             * Sets whether function queuing is allowed.
             * This does not affect event queuing.
             * Function queuing is allowed by default when EBusTraits::EnableEventQueue
             * is true. It is never allowed when EBusTraits::EnableEventQueue is false.
             * @see QueueFunction, EBusTraits::EnableEventQueue
             * @param isAllowed Set to true to allow function queuing. Otherwise, set to false.
             */
            static void AllowFunctionQueuing(bool isAllowed) { Bus::GetOrCreateContext().m_queue.SetActive(isAllowed); }

            /**
             * Returns whether function queuing is allowed.
             * @return True if function queuing is allowed. Otherwise, false.
             * @see QueueFunction, AllowFunctionQueuing
             */
            static bool IsFunctionQueuing()
            {
                auto* context = Bus::GetContext();
                return context ? context->m_queue.IsActive() : Traits::EventQueueingActiveByDefault;
            }

            /**
            * Helper to Queue a Broadcast only when function queueing is available
            * @param func          Function pointer of the event to dispatch.
            * @param args          Function arguments that are passed to each handler.
            */
            template <class Function, class ... InputArgs>
            static void TryQueueBroadcast(Function&& func, InputArgs&& ... args);

            /**
             * Enqueues an asynchronous event to dispatch to all handlers.
             * The event is not executed until ExecuteQueuedEvents() is called.
             * @param func          Function pointer of the event to dispatch.
             * @param args          Function arguments that are passed to each handler.
             */
            template <class Function, class ... InputArgs>
            static void QueueBroadcast(Function&& func, InputArgs&& ... args);

            /**
            * Helper to Queue a Reverse Broadcast only when function queueing is available
            * @param func          Function pointer of the event to dispatch.
            * @param args          Function arguments that are passed to each handler.
            */
            template <class Function, class ... InputArgs>
            static void TryQueueBroadcastReverse(Function&& func, InputArgs&& ... args);

            /**
             * Enqueues an asynchronous event to dispatch to all handlers in reverse order.
             * The event is not executed until ExecuteQueuedEvents() is called.
             * @param func          Function pointer of the event to dispatch.
             * @param args          Function arguments that are passed to each handler.
             */
            template <class Function, class ... InputArgs>
            static void QueueBroadcastReverse(Function&& func, InputArgs&& ... args);

            /**
             * Enqueues an arbitrary callable function to be executed asynchronously.
             * The function is not executed until ExecuteQueuedEvents() is called.
             * The function might be unrelated to this EBus or any handlers.
             * Examples of callable functions are static functions, lambdas, and
             * bound-member functions.
             *
             * One use case is to determine when a batch of queued events has finished.
             * When the function is executed, we know that all events that were queued
             * before the function have finished executing.
             *
             * @param func Callable function.
             * @param args Arguments for `func`.
             */
            template <class Function, class ... InputArgs>
            static void QueueFunction(Function&& func, InputArgs&& ... args);
        };
```
这个结构体的功能就是对[[Policies#EBusQueuePolicy]]的应用，它作为一个数据保存在EBus中，其中有一个队列message，里面保存了很多函数，