Ebus是O3DE引擎中的消息传递系统，更加详细的信息可以参考官方文档：[EBus](https://www.docs.o3de.org/docs/user-guide/programming/messaging/)
这里详细研究它的实现机制。

EBus使用模板编程，主要用来分发事件或者接收请求。
EBus代码架构：
...
# EBus中的一些概念
Interface：一个抽象类，其中定义了EBus需要分发或者接收的虚函数。
Traits：用来定义EBus的属性，一般来说如果Interface继承了EBusTraits，它就不用提供了。
Handler：连接到EBus的实例，EBus分发事件或接收请求时，会触发它们的函数。
Address：用来指定确定事件或请求需要分发到哪些Handle，如果使用的话，一般以ID来指定，默认不使用，也就是事件通知到所有Handler。
# Internal

Internal中包含了EBus的底层基础组件。
## StoragePolicies
此模块提供了Address和Handle的保存机制。
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
### HandlerStoragePolicy
用来决定如何保存handler列表，由于一个EBus可能只连接一个handler，也可能连接多个handler，当连接多个Handler时，要决定如何保存它们。假设EBus是使用AddressPolicy的，那么这个类会被提供为上面AddressStoragePolicy的第二个模板参数。
```cpp
template <typename Interface, typename Traits, typename Handler, EBusHandlerPolicy = Traits::HandlerPolicy>
struct HandlerStoragePolicy;
```
与上面的AddressStoragePolicy一样，它依靠模板的特化来实现具体的功能。
4个模板参数：
Interface，Traits：都是从EBus中获取，含义与EBus中的相同。
Handler：handler的数据类型。
EBusHandlerPolicy：只有EBu使用了HandlerPolicy::Multiplexxx（即允许多个Handler连接到EBus）才会使用这个类。

需要排序的HandlerStoragePolicy，第4个模板参数提供EBusHandlerPolicy::Multiple。
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
其中的StorageType是一个list。
需要排序的HandlerStoragePolicy，第4个模板参数提供EBusHandlerPolicy::Multiple。

## EBusContainer

## Handles
Handle在EBus中作为事件的监听者，它可以连接到EBus系统，当事件触发时Handle响应此事件，并且可以在不需要时断开与EBus的连接。
下面是几种不同类型得Handle。
### NonIdHandler
这种Handle用于single address的Bus，也就是说它监听的事件不需要绑定id，所有Handle都可以监听到。
```cpp
template <typename Interface, typename Traits, typename ContainerType>
class NonIdHandler: public Interface
{
...
}
        
```
