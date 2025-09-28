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
这个结构体的功能就是对[[Policies#EBusQueuePolicy]]的应用，它作为一个数据保存在EBus中，其中有一个队列message，其中保存了很多函数，Execute时顺序执行这些函数。
EBusBroadcastQueue执行所有异步事件，其中调用的m_queue.Execute()在[[Policies#EBusQueuePolicy]]中介绍过，就是顺序执行队列中的所有事件，它是线程安全的。

EBusBroadcastQueue中的许多函数功能是相似的，比如QueueBroadcast是将一个事件按顺序异步执行，QueueBroadcastReverse是将一个事件按反序异步执行，所以这里只详细分析它们的其中一个版本。这些函数加入的事件会在EBusBroadcastQueue被调用时执行。
所有的函数都依赖于核心函数QueueFunction，在注释中有解释，它的作用是将一个可调用的函数对象加入队列，在EBusBroadcastQueue执行它们，这个可调用的函数对象并不一定与当前EBus或者Handle关联，它可能是一个静态函数，lambda函数或者成员函数。
```cpp
        template <class Bus, class Traits>
        template <class Function, class ... InputArgs>
        inline void EBusBroadcastQueue<Bus, Traits>::QueueFunction(Function&& func, InputArgs&& ... args)
        {
            // 确认当前EBus支持Queue event功能
            static_assert((AZStd::is_same<typename Bus::QueuePolicy::BusMessageCall, typename AZ::Internal::NullBusMessageCall>::value == false),
                "This EBus doesn't support queued events! Check 'EnableEventQueue'");

            auto& context = Bus::GetOrCreateContext(false);
            if (context.m_queue.IsActive())
            {
                AZStd::scoped_lock<decltype(context.m_queue.m_messagesMutex)> messageLock(context.m_queue.m_messagesMutex);
                // 这里将传入的func和args做了包装
                context.m_queue.m_messages.push(typename Bus::QueuePolicy::BusMessageCall(
                    [func = AZStd::forward<Function>(func), args...]() mutable
                {
                    AZStd::invoke(AZStd::forward<Function>(func), AZStd::forward<InputArgs>(args)...);
                },
                    typename Traits::AllocatorType()));
            }
            else
            {
                AZ_Warning("EBus", false, "Unable to queue function onto EBus.  This may be due to a previous call to AllowFunctionQueuing(false)."
                    "  Hint: This is often disabled during shutdown of a ComponentApplication");
            }
        }
```
这个函数的核心是将一个lambda和一个分配器加入队列，lambda函数的类型是Bus::QueuePolicy::BusMessageCall，在[[Policies#EBusQueuePolicy]]有它的定义：
```cpp
typedef AZStd::function<void()> BusMessageCall;
```
就是一个不接受任何参数，也不返回任何值的函数，这里构造的lambda函数也是这个形式。然后lambda函数捕获了传入的func和args，并在其中调用它们（使用std::invoke）。经过这样的包装后，不论什么形式的函数都可以传进来。第二个参数是typename Traits::AllocatorType()，一个内存分配器，因为EBusQueuePolicy中的队列支持自定义内存分配器，这里使用的是EBus中的内存分配器。
有了这个函数，就可以实现其他的加入队列的功能了，比如:
### EBusBroadcastQueue
```cpp
        template <class Bus, class Traits>
        template <class Function, class ... InputArgs>
        inline void EBusBroadcastQueue<Bus, Traits>::QueueBroadcast(Function&& func, InputArgs&& ... args)
        {
            Internal::QueueFunctionArgumentValidator<Function, Traits::EnableQueuedReferences>::Validate();
            using Broadcaster = void(*)(Function&&, InputArgs&&...);
            Bus::QueueFunction(static_cast<Broadcaster>(&Bus::Broadcast), AZStd::forward<Function>(func), AZStd::forward<InputArgs>(args)...);
        }
```
这个函数可以在EBus上广播Function事件，它第一步先验证参数，使用的是QueueFunctionArgumentValidator，看下它的实现。
### QueueFunctionArgumentValidator
```cpp
            template <class Function, bool AllowQueuedReferences>
            struct QueueFunctionArgumentValidator
            {
                static void Validate() {}
            };

            template <class Function>
            struct ArgumentValidatorHelper
            {
                constexpr static void Validate()
                {
                    ValidateHelper(AZStd::make_index_sequence<AZStd::function_traits<Function>::num_args>());
                }

                template<typename T>
                using is_non_const_lvalue_reference = AZStd::integral_constant<bool, AZStd::is_lvalue_reference<T>::value && !AZStd::is_const<AZStd::remove_reference_t<T>>::value>;

                template <size_t... ArgIndices>
                constexpr static void ValidateHelper(AZStd::index_sequence<ArgIndices...>)
                {
                    static_assert(!AZStd::disjunction_v<is_non_const_lvalue_reference<AZStd::function_traits_get_arg_t<Function, ArgIndices>>...>,
                        "It is not safe to queue a function call with non-const lvalue ref arguments");
                }
            };

            template <class Function>
            struct QueueFunctionArgumentValidator<Function, false>
            {
                using Validator = ArgumentValidatorHelper<Function>;
                constexpr static void Validate()
                {
                    Validator::Validate();
                }
            };
```
QueueFunctionArgumentValidator默认是没有实现的，只有一个空函数Validate。
当QueueFunctionArgumentValidator指定第二个模板参数为false时，表示函数不允许传递引用作为参数，此时它才有实现。
ValidateHelper用来验证这个函数的参数是否都为常量左值引用，如果不是则编译报错，其中使用了is_non_const_lvalue_reference来判断参数类型。
这个结构体使用了大量的模板traits，它们的实现暂时不深入研究，现在确定的是这个QueueFunctionArgumentValidator功能，它用来验证一个函数传入的参数，如果将第二个模板参数AllowQueuedReferences设置为false，它会保证传入此函数的参数必须为常量左值引用（直接传值应该也能匹配？），否则编译就会出错。

再回头看EBusBroadcastQueue的实现，使用QueueFunctionArgumentValidator完成验证后，后面就很简单了，代码片段：
```cpp
using Broadcaster = void(*)(Function&&, InputArgs&&...);
Bus::QueueFunction(static_cast<Broadcaster>(&Bus::Broadcast), AZStd::forward<Function>(func), AZStd::forward<InputArgs>(args)...);
```
它调用QueueFunction并传入了三个参数，通过QueueFunction的定义可以知道，QueueFunction接收多个参数，并将第一个参数作为可调用的函数，后面的参数作为这个函数的参数。所以这里的第一个参数是Broadcaster，它也是一个接收多个参数的函数，并且没有返回值，它由Bus::Broadcast转换而来，Bus::Broadcast定义在[[Evnt Bus Internal#EBusContainer]]中，在Dispatcher里面，其格式与这里定义的Broadcaster相同。
所以这里的调用逻辑是
```cpp
QueueFunction(Broadcaster, func, args...)
//      |
//      |
//     \|/
Broadcaster(func, args...)
//      |
//      |
//     \|/
std::invoke(func, args...)
```
这样就成功调用了func函数并传入args。
### TryQueueBroadcast
```cpp
        template <class Bus, class Traits>
        template <class Function, class ... InputArgs>
        inline void EBusBroadcastQueue<Bus, Traits>::TryQueueBroadcast(Function&& func, InputArgs&& ... args)
        {
            if (EBusEventQueue<Bus, Traits>::IsFunctionQueuing())
            {
                EBusEventQueue<Bus, Traits>::QueueBroadcast(AZStd::forward<Function>(func), AZStd::forward<InputArgs>(args)...);
            }
        }
```
在QueueBroadcast外做了一次验证，保证此EBus开启了FunctionQueuing功能。
### QueueBroadcastReverse
```cpp
        template <class Bus, class Traits>
        template <class Function, class ... InputArgs>
        inline void EBusBroadcastQueue<Bus, Traits>::QueueBroadcastReverse(Function&& func, InputArgs&& ... args)
        {
            Internal::QueueFunctionArgumentValidator<Function, Traits::EnableQueuedReferences>::Validate();
            using Broadcaster = void(*)(Function&&, InputArgs&&...);
            Bus::QueueFunction(static_cast<Broadcaster>(&Bus::BroadcastReverse), AZStd::forward<Function>(func), AZStd::forward<InputArgs>(args)...);
        }
```
QueueBroadcast反序版本，将Bus::Broadcast换成了Bus::BroadcastReverse，它也是在[[Evnt Bus Internal#EBusContainer]]中定义的。
# EBusEventQueue
```cpp
        /**
         * Enqueues asynchronous events to dispatch to handlers that are connected to
         * a specific address on an EBus.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusEventQueue
            : public EBusBroadcastQueue<Bus, Traits>
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
             * Helper to queue an event by BusIdType only when function queueing is enabled
             * @param id            Address ID. Handlers that are connected to this ID will receive the event.
             * @param func          Function pointer of the event to dispatch.
             * @param args          Function arguments that are passed to each handler.
             */
            template <class Function, class ... InputArgs>
            static void TryQueueEvent(const BusIdType& id, Function&& func, InputArgs&& ... args);

            /**
             * Enqueues an asynchronous event to dispatch to handlers at a specific address.
             * The event is not executed until ExecuteQueuedEvents() is called.
             * @param id            Address ID. Handlers that are connected to this ID will receive the event.
             * @param func          Function pointer of the event to dispatch.
             * @param args          Function arguments that are passed to each handler.
             */
            template <class Function, class ... InputArgs>
            static void QueueEvent(const BusIdType& id, Function&& func, InputArgs&& ... args);

            /**
             * Helper to queue an event by BusPtr only when function queueing is enabled
             * @param ptr           Cached address ID. Handlers that are connected to this ID will receive the event.
             * @param func          Function pointer of the event to dispatch.
             * @param args          Function arguments that are passed to each handler.
             */
            template <class Function, class ... InputArgs>
            static void TryQueueEvent(const BusPtr& ptr, Function&& func, InputArgs&& ... args);

            /**
             * Enqueues an asynchronous event to dispatch to handlers at a cached address.
             * The event is not executed until ExecuteQueuedEvents() is called.
             * @param ptr           Cached address ID. Handlers that are connected to this ID will receive the event.
             * @param func          Function pointer of the event to dispatch.
             * @param args          Function arguments that are passed to each handler.
             */
            template <class Function, class ... InputArgs>
            static void QueueEvent(const BusPtr& ptr, Function&& func, InputArgs&& ... args);

            ...
};
```
继承了[[#EBusBroadcastQueue]]的特殊版本，之前是广播，这里是将事件发送给指定id上的Handler，通过指定BusId或BusPtr进行定位r，其中提供的函数与EBusBroadcastQueue相似，这里就看一下QueueEvent的实现。
```cpp
        template <class Bus, class Traits>
        template <class Function, class ... InputArgs>
        inline void EBusEventQueue<Bus, Traits>::QueueEvent(const BusIdType& id, Function&& func, InputArgs&& ... args)
        {
            Internal::QueueFunctionArgumentValidator<Function, Traits::EnableQueuedReferences>::Validate();
            using Eventer = void(*)(const BusIdType&, Function&&, InputArgs&&...);
            Bus::QueueFunction(static_cast<Eventer>(&Bus::Event), id, AZStd::forward<Function>(func), AZStd::forward<InputArgs>(args)...);
        }
```
和EBusBroadcastQueue几乎是一样的，区别就是传递了Bus::Event，它在[[Evnt Bus Internal#EBusContainer]]中定义，需要一个参数id或BusPtr来指定可以接收到事件的Handler。
# EBusBroadcastEnumerator
```cpp
        /**
         * Provides functionality that requires enumerating over all handlers that are
         * connected to an EBus.
         * To enumerate over handlers that are connected to a specific address
         * on the EBus, use a function from EBusEventEnumerator.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusBroadcastEnumerator
        {
            /**
             * Finds the first handler that is connected to the EBus.
             * This function is only for special cases where you know that a particular
             * component's handler is guaranteed to exist.
             * Even if the returned pointer is valid (not null), it might point to a handler
             * that was deleted. Prefer dispatching events using EBusEventer.
             * @return          A pointer to the first handler on the EBus, even if the handler
             *                  was deleted.
             */
            static typename Traits::InterfaceType* FindFirstHandler();
        };
```
与[[#EBusEventEnumerator]]对应的广播版本，其中只有一个函数FindFirstHandler，用来寻找第一个连接到此EBus的Handler。
```cpp
        template <class Bus, class Traits>
        typename Traits::InterfaceType * EBusBroadcastEnumerator<Bus, Traits>::FindFirstHandler()
        {
            typename Traits::InterfaceType* result = nullptr;
            Bus::EnumerateHandlers([&result](typename Traits::InterfaceType* handler)
            {
                result = handler;
                return false;
            });
            return result;
        }
```
同样使用EnumerateHandlers执行一个lambda来寻找handler，实现方式与EBusEventEnumerator的相同。
# EventDispatcher
```cpp
        // This alias is required because you're not allowed to inherit from a nested type.
        template <typename Bus, typename Traits>
        using EventDispatcher = typename Traits::BusesContainer::template Dispatcher<Bus>;
```
使用的就是[[Evnt Bus Internal#EBusContainer]]中的Dispatcher，其中有Event和Broadcast等函数，实现了EBus的事件分发功能。
# EBusImpl
上面介绍了许多结构体，真正的EBusImpl就是通过组合这些结构体的功能来实现的。
```cpp
        /**
         * Base class that provides eventing, queueing, and enumeration functionality
         * for EBuses that dispatch events to handlers. Supports accessing handlers
         * that are connected to specific addresses.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         * @tparam BusIdType The type of ID that is used to address the EBus.
         */
        template <class Bus, class Traits, class BusIdType>
        struct EBusImpl
            : public EventDispatcher<Bus, Traits>
            , public EBusBroadcaster<Bus, Traits>
            , public EBusEventer<Bus, Traits>
            , public EBusEventEnumerator<Bus, Traits>
            , public AZStd::conditional_t<Traits::EnableEventQueue, EBusEventQueue<Bus, Traits>, EBusNullQueue>
        {
        };
```
基础的EBusImpl，结合了EventDispatcher，EBusBroadcaster，EBusEventer，EBusEventEnumerator功能，再根据Traits::EnableEventQueue是否为true选择继承EBusEventQueue或EBusNullQueue。
这里继承了EventDispatcher，所以上面才可以以Bus::Broadcast的形式调用EventDispatcher（EBusContainer中的Dispatcher）的Broadcast函数。
EBusImpl还有一个特化版本：
```cpp
        /**
         * Base class that provides eventing, queueing, and enumeration functionality
         * for EBuses that dispatch events to all of their handlers.
         * For a base class that can access handlers at specific addresses, use EBusImpl.
         * @tparam Bus       The EBus type.
         * @tparam Traits    A class that inherits from EBusTraits and configures the EBus.
         *                   This parameter may be left unspecified if the `Interface` class
         *                   inherits from EBusTraits.
         */
        template <class Bus, class Traits>
        struct EBusImpl<Bus, Traits, NullBusId>
            : public EventDispatcher<Bus, Traits>
            , public EBusBroadcaster<Bus, Traits>
            , public EBusBroadcastEnumerator<Bus, Traits>
            , public AZStd::conditional_t<Traits::EnableEventQueue, EBusBroadcastQueue<Bus, Traits>, EBusNullQueue>
        {
            using EBusBroadcastEnumerator<Bus, Traits>::FindFirstHandler;

            static typename Traits::InterfaceType* FindFirstHandler(const NullBusId&)
            {
                // Invoke the EBusBroadcastEnumerator FindFirstHandler function
                // Since this EBus doesn't use a BusId, the argument isn't needed
                return FindFirstHandler();
            }

            static typename Traits::InterfaceType* FindFirstHandler(const typename Traits::BusPtr&)
            {
                // Invoke the EBusBroadcastEnumerator FindFirstHandler function
                // Since this EBus doesn't use a BusId, the argument isn't needed
                return FindFirstHandler();
            }
        };
```
用来实现无Address的EBus版本，它的第三个模板参数为NullBusId。
同时，这里定义了新的FindFirstHandler，由于是没有Address的EBus，所以显式指定FindFirstHandler使用EBusBroadcastEnumerator中的函数，从代码中也可以看出，传入的参数NullBusId和Traits::BusPtr是没有使用的。
# 总结
EBusImpl是EBus的核心实现部分，它包含以下部件
- EBusEventer和EBusBroadcaster，用于分发事件，但是没有具体实现，推测是后续的更新方向。
- EBusEventEnumerator和EBusBroadcastEnumerator，用户枚举EBus上的Handler，目前的用法只有寻找EBus上的第一个Handler和统计Handler的数量。
- EBusNullQueue，EBusBroadcastQueue和EBusEventQueue，用于将一个事件（函数）加入到执行队列中，后面可以在合适的时机顺序执行。这样EBus有了异步执行的功能。
- EventDispatcher，是BusContainer的Dispatcher的别名，其中实现了EBus最重要的Broadcast和Event函数，推测是一个需要废弃的部件，它的功能应该移动到EBusEventer和EBusBroadcaster中。（也有可能是发现Broadcast和Event函数离开BusContainer实现不了？）


