# STL基础体系
## STL六大部件
- 容器(Containers)
- 分配器（Allocators）
- 算法(Algorithms)
- 迭代器(Iterator)
- 适配器(Adapters)
- 仿函式(Functors)
![[Pasted image 20240419211022.png]]

程序示例： 
![[Pasted image 20240419211400.png]]

```cpp
bind2nd   // 绑定第二个参数为40
less<int>()  // 仿函数对象，小于
not1      // not 非运算
```

上面的代码含义为，计算vi这个vector中大于等于40的元素的数量。

在标准库中，一般使用前闭后开区间 \[ )，v.begin()指向第一个元素，v.end()执行最后一个元素的**后一个元素**。
## 容器的结构与分类
### 序列式容器（Sequence Containers）
- Array：数组，普通数组的封装
- Vector：可以在最后扩充或删除元素
- Deque：双端队列，两端都可扩充或删除元素
- List：双向链表
- ForwardList：单向链表
### 关联式容器（Associative Containers）
- Set：集合
- Multiset：可重复的集合
- Map：映射，key-value对
- Multimap：key可重复的映射
- Unordered Set/Multiset：无序的集合
- Unordered Map/Multimap：无序的映射
- 
Associative Containers在查找时有很高的效率，标准库中并未规定如何实现这两个容器，但一般会使用红黑树，Unordered Associative Containers一般使用哈希表。

vector.size()  元素个数
vector.capacity() 容量大小
vector的内存增长一般是成倍增长的

list.max_size()  链表可放置的最大元素数（ForwardList和Deque也有)

Stack和Queue底层使用的是Deque，所以它们是特殊的容器，不再底层容器之列，我们也将其称为Adapter(容器适配器)。它们是没有iterator的。

Unordered Set/Multiset中有一个bucket_count()函数，返回哈希表bucket的数量。

Multimap不能使用map\[key] = value这种方式插入数据，必须使用map.insert(pair<key_type, value_type>(key, value))这种方式插入，因为它允许有重复的key，multimap\[key] = value这种方式会引起错误，而map就可以使用这种方式。

在使用容器时指定分配器的示例：
![[Pasted image 20240423222203.png]]


# STL容器源码分析
## OOP和GP（模板编程）
OOP将数据类操作数据的方法放在一起。
GP将数据和方法分开。比如容器和算法可以分开，使用iterator将它们关联起来。

必要基础：
- 操作符重载
  ![[Pasted image 20240424210112.png]]
- 模板编程
   特化（泛化。全特化）。
   偏特化。
   ![[Pasted image 20240424212020.png]]
- 分配器：
    详细可见 [[C++内存管理 笔记]]

## 容器间的关系
 ![[Pasted image 20240425193518.png]]
 上面的图中通过缩减来表示各个容器之间的关系，eg：heap中包含了vector，stack和queue中包含了deque。
## 容器list
GUN 2.9
![[Pasted image 20240425194253.png]]
list中的元素都是node，一个list_node* 的指针。
每一个list_node都有3个变量：指向上一个和下一个节点类型的指针（void*），T类型的数据data。
可以看出，list是一个循环链表，它在末尾加入了一个空节点，用来隔开头尾节点，正好实现前闭后开的迭代器。
iterator是一个模板类，有3个模板参数，它的定义如下：
![[Pasted image 20240425195440.png]]
iterator作为一种特殊的指针，在其中使用了大量的运算符重载，让iterator可以使用指针的功能，比如解引用，++。
前置的++，将它的next指针设置给当前指针（指针后移），然后返回当前的迭代器对象(为什么这里没有调用重载的\*)；
后置的++，现在当前迭代器取出为tmp，然后调用前置++，最后返回tmp
注意`self tmp = *this`并不会调用* 运算符取出当前node的值，而是调用=运算符，执行拷贝构造。所以后置++依然返回的tmp依然是一个迭代器。
（为什么使用=的重载就不使用\* 的重载了）。
注意前置与后置++的返回值是不同的，前置返回self&，后置返或self，所以可以连续使用前置++，而不能连续使用后置++。

![[Pasted image 20240425202011.png]]
->运算符的重载比较复杂

GUN 4.9
![[Pasted image 20240425202319.png]]
4.9版本相比于2.9版本做了一点改进：
\_List_iterator只使用一个模板参数T,\_list_node中的指针类型由void* 改为了 \_List_node_base\*

![[Pasted image 20240425202734.png]] 

4.9版本的容器设计更加复杂。。。
list的size也从2.9的4字节变成了4.9的8字节。
## Traits
Iterator遵循的原则：
与它相关的5种类型(associated types)：
Iterator的分类 
两个Iterator差值的类型 
所指向的数据类型 
所指向数据的引用的类型 
所指向数据的指针的类型 
![[Pasted image 20240425212134.png]]

算法使用以下方式获取这5种类型：
![[Pasted image 20240425212352.png]]

所谓Traits，就是需要区分出所收到的iterator到底是class形式的iterator还是一个普通的指针，因为如果是普通的指针就不能用上面的方式获取这5种类型了，
![[Pasted image 20240425213228.png]]
一个例子：
![[Pasted image 20240425213607.png]]

使用模板的偏特化，当模板接收的类型是指针时，value_type直接使用T。其余的associated types也是通过这种方式声明。
除了iterator traits置为，还有一些其他的traits，但它们使用的方法都是一样的。
## vector
vector在容量不足时，会扩充到原来容量的两倍（并不是标准规定的），不是在原来的内存上扩充，而是搬到一块更大的内存。
![[Pasted image 20240426213157.png]]
vector中有3个迭代器，start，finish，end_of_storage，它们分别指向vector头部、尾部和内存的尾部。
push_back函数
![[Pasted image 20240426214228.png]]![[Pasted image 20240426214436.png]]
由于vector的生长机制，每次触发扩充容量的操作时，都需要将其中的元素搬到新的地方并销毁原来的元素，如果vector中的元素是对象，会触发大量的拷贝构造和析构操作。
vector::iterator
![[Pasted image 20240426215701.png]]
vector的iterator就是一个T\* 的指针。所以要取iterator相关的5种类型就用到traits了。
同样，4.9版本的vector变得更加复杂：
![[Pasted image 20240426220013.png]]
4.9版本的iterator
![[Pasted image 20240426220353.png]]
不再是一个简单的指针了。它的成员\_M_current才是T* 的指针。
## array和forwardlist
array是数组的包装，在原来数组的基础上加了一些功能，如迭代器，可以让array也使用stl中的算法和功能。
![[Pasted image 20240427162902.png]]
array中只有一个\_M_instance成员，它是一个数组，array的大小在初始化时就决定了，而且不能被改变。它的迭代器就是一个指针。
4.9的array
![[Pasted image 20240427163322.png]]

它的成员变量\_M_elems是一个\_Tp\[\_Nm]类型的数据。
forwardlist时一个单项链表，和list相似。不再赘述。
## deque
双向的队列（vector是单向的）
deque的实现是使用多个buffer实现，在其中维护多个buffer的指针，将他们串连起来，看起来像是一块连续的内存。
![[Pasted image 20240427171035.png]]
他有4个成员变量：
- start迭代器
- finish迭代器
- 二级指针map：保存了deque中的所有buffer的指针。可以理解为vector<T*>
- map_size：map的大小

![[Pasted image 20240427171604.png]]
迭代器有4个变量：
- cur：指针，表示这个迭代器指向的元素
- first：指针，这个迭代器所指向元素所在buffer的头节点
- last：指针，这个迭代器所指向元素所在buffer的尾节点
- node：这个迭代器所指向元素所在buffer在deque中map中的引用，也就是用来确定当前buffer所在的位置（为什么他也是一个二级指针?因为node是可以++的，可以移动到下一个node）
insert函数：
![[Pasted image 20240427172842.png]]
![[Pasted image 20240427173201.png]]
insert_aux这个函数可以判断插入元素时应该将之前的元素往前挪还是将之后的元素往后挪。
deque使用iterator来模拟连续空间，iterator的一些操作符重载，
![[Pasted image 20240427204824.png]]
![[Pasted image 20240427205128.png]]
迭代器的减法做了重载，注意buffer_size()是一个buffer的容量，即一个buffer包含的元素的个数，`node - x.node - 1`计算出了两个迭代器的node之间的距离（它们都指向deque的map中的元素），也就是两个迭代器之间有几个buffer。然后在加上两个迭代器在它们buffer中的偏移，即 `cur - first` 和`x.last - x.cur`。为什么计算末尾元素数量的时候是`cur - first`，而计算起始元素数量的时候是`x.last - x.cur`，可以这样理解，deque是两端生长的容器，在头部加入元素时，该buffer从last开始使用，逐渐靠向first，所以使用last - cur来计算当前元素的偏移，在尾部加入元素时，该buffer从first开始使用，逐渐靠向last，所以使用cur - first来计算当前元素的偏移。注意buffer内永远是first <= cur <= last。

![[Pasted image 20240427211254.png]]
前置的++，先++cur，如果这个buffer用完了，跳到下一个buffer的起点，在set_node中设置使用的buffer后，重新设置first和last指针指向这个buffer的头尾。--操作也是同理。
![[Pasted image 20240427212119.png]]
+操作符中使用了+=操作符。+=操作符中，首先计算offset，offset = n + (cur - first)，cur - first表示这个buffer中cur与first之间的距离，加上表示还要再偏移n个距离，所以offset的含义为：偏移后的位置与当前buffer的first之间的距离。
注意offset可能是负数。
-操作符、-=操作符和\[]操作符都可以使用+=操作符来实现。
4.9版本
![[Pasted image 20240427224659.png]]
注意在新版本中buffer_size这个参数被去掉了，buffer_size固定为512/sizeof(T)或1。
## queue

![[Pasted image 20240427225708.png]]
内部使用一个deque，即成员变量c，它的所有操作都有c来实现。
queue只能从后进，从前出。
## stack
![[Pasted image 20240427225937.png]]
stack同理，它只能从一端进出。

stack和queue都无法遍历，也不提供iterator。
stack也可以选择vector或者list作为底层容器，只要这个容器可以实现尾部的插入和取出元素。
使用list作为底层容器：
`stack<string, list<string>> s`
同理，queue可以选择list作为底层容器，vector就不行了，因为vector没有push_front方法。
## rb_tree了解

![[Pasted image 20240428201337.png]]

它的实现：
![[Pasted image 20240428202348.png]]
模板参数有5个
- Key：元素的key类型
- Value：元素的key - value类型，也就是节点的类型
- keyOfValue：在一个节点中如何获取key
- Compare：key的比较函数
成员：
- node_count：节点数量
- header：树的根节点，它是一个空的节点，iteratot.end()返回它
- key_compare：key的比较函数
使用示例：
![[Pasted image 20240428203149.png]]
key和value都是int，表示这个节点就是key，没有value。
identity:
![[Pasted image 20240428203642.png]]
用来从T中取出T，这里用来从int中取出int，其实就是返回它自己。
less:
![[Pasted image 20240428203910.png]]
返回 x < y。
测试用例：
2.9
![[Pasted image 20240428204315.png]]
4.9
![[Pasted image 20240428204543.png]]
改了一些名称，几本没有变化。
4.9版的红黑树关系图
![[Pasted image 20240428204719.png]]
## set和multiset
![[Pasted image 20240429210716.png]]
重点，无法使用iterator为元素赋值，它们的迭代器是const的。
set:
![[Pasted image 20240429211032.png]]
3个模板参数：
Key：数据类型
Compare：比较函数
Alloc：分配器
注意set中的key_type和value_type都是Key。
rep_type t就是它使用的红黑树。
## map和multimap
![[Pasted image 20240429213135.png]]
与set的区别是map储存的是key-value对，无法改变key的值，但是可以改变value的值。
map:
![[Pasted image 20240429213355.png]]
与set相似，只是data_type变成了T类型，对应底层红黑树的value_type变成了pair<const key_type, value_type>。
map的迭代器不再是const的了，因为要用来改变value的值。
上面的红黑树的比较函数是select1st，它的定义为：
![[Pasted image 20240429215218.png]]
传入一个pair，返回pair.first。（为什么没有比较）
map的operator\[]
![[Pasted image 20240429215949.png]]
multimap是不能用这种方式插入数据的，而map可以。其中使用lower_bound是二分边界搜索，寻找不小于（大于或等于）它的最小元素。
在这里可以看到，当使用\[]操作符没有找到key时，会向map中插入这个key。
## hashtable
stl中使用的哈希表使用链表来解决哈希碰撞问题。
当buckets的数量小于元素数量时，会扩充哈希表容量，按照素数来扩充，比如原来是53，容量会扩充为97，然后重新计算元素的哈希值。
![[Pasted image 20240430203858.png]]

代码：
![[Pasted image 20240430204100.png]]
它有6个模板参数：
- Value：node的类型
- Key：key的类型
- HashFcn：哈希函数
- ExtractKey：从node中如何获取key
- EqualKey：如何判断key是否相等
- Alloc：分配器
同样有6个成员函数，前3个hash, equals, get_key成员很明显，主要看后3个：
- node：哈希表的节点，有一个node类型的指针next和一个Value类型的值val
- buckets：用来“装”节点的容器
- num_elements：元素（节点）个数
它有一个函数bucket_count返回bucket的数量。
hashtable的迭代器有一个node\* cur，表示此迭代器指向的节点，还有一个hashtable\* ht指向hashtable本身，可以让迭代器在链表中搜索后再返回到哈希表。
hash表使用示例：
![[Pasted image 20240430210235.png]]
其中的hash函数有如下定义：
![[Pasted image 20240430211006.png]]
![[Pasted image 20240430211420.png]]
这些就是stl提供的哈希函数。hash<const char*>就是上面的\_\_stl\_hash\_string这个哈希函数。
获得hashcode后，就要计算它应该落在哪个buckets，
![[Pasted image 20240430213018.png]]
以上面的find函数为例，它通过key寻找bucket编号时调用bkt_num_key(key)函数，这个函数又调用bkt_num_key(key,  size)函数，最终进行了一个hash(key) % n的计算，n就是哈希表中bucket的个数，所以最终是通过取模运算来确定buckets的编号，比如一个key的hashcode计算出来是107，而哈希表中有100个buckets，那么这个节点会落在107 % 100 = 7号bucket。
## unordered容器
unordered容器底层使用hashtable来实现，对应之前红黑树实现的版本。
![[Pasted image 20240502160833.png]]
# 算法
标准库中算法基本都是此形式：
![[Pasted image 20240502162008.png]]
它不关心具体的Container，它只关心Iterator，所以Iterator要可以“回答”算法所需要的信息。
## 五种迭代器
![[Pasted image 20240502162802.png]]
它们的关系如题：
![[Pasted image 20240502162846.png]]
（forward没有output功能吗？）
以distance函数为例来说明不同类型的迭代器对算法的影响：
![[Pasted image 20240502193244.png]]
distance用来计算两个迭代器之间的距离（最下面的函数），可以看到它调用了内置的\_\_distance函数，\_\_distance根据category的类型来选择不同的算法，如果是random_access的迭代器，直接使用迭代器相减来计算距离，如果是input迭代器，就一步一步往后计算二者的距离，因为input迭代器是没有减法运算的。
为什么这里只写了两种类型迭代器的偏特化版本？注意调用\_\_distance函数时，传入的第三个参数是一个对象category()，所以forward和bidirectional作为input的子类，会匹配到input迭代器。
copy函数：
![[Pasted image 20240502195410.png]]
copy函数接收3个参数，源区间的两个迭代器，目标区间的迭代器，但是他根据数据的类型和迭代器的类型做了很多优化，比如对const char\*类型的数据直接调用memmove()来进行拷贝。如果是T\*指针，又根据T类型的拷贝构造函数是否重要（比如执行了深拷贝）来决定是一个一个复制还是直接memmove。
\_\_unique_copy函数:
![[Pasted image 20240502200816.png]]
与copy函数相似，但是它只copy区间中不同的数据，根据copy目标迭代器类型不同，forward迭代器的实现比较常规，重点是如果迭代器是output类型的时，由于output迭代器无法读取数据，所以它会先获取元素的value_type，再使用这个value_type创建一个临时变量value来作为copy数据的中转站。

算法中会对迭代器类型有限制，比如sort要求迭代器是random_access的，而find函数要求迭代器为input即可。这种限制并不是语法上的限制，只在参数名称中做了提示，所以传入不满足要求的迭代器并不会使编译报错，只是在运行时会报错。
![[Pasted image 20240502202118.png]]
## 一些常见的算法
### accumulate
用于计算区间内所有元素的值加上init的值。
![[Pasted image 20240502203159.png]]
它有两个版本，一个是简单的累加，一个先执行自定义的binary_op操作再累加，
使用示例：
![[Pasted image 20240502203353.png]]
上面的例子中，第一个例子直接调用了accumulate，所以执行累加，100 + 10 + 20 + 30 = 160，minus函数执行减法，匹配第二个带binary_op参数的版本，所以计算结果为 100 - 10 - 20 - 30 = 40。
### for_each
对容器内的每个函数执行f操作。
![[Pasted image 20240502204155.png]]
### replace_xxx
replace：将区间中的所有旧值替换为新值。
![[Pasted image 20240502204608.png]]
replace：将区间中的满足某个条件的值替换为新值。
![[Pasted image 20240502204849.png]]
replace_copy：将区间中的所有旧值替换为新值，并将所有元素copy到一个新区间。
![[Pasted image 20240502205248.png]]
### count__xx
count：查找区间中与value相等的元素个数。
![[Pasted image 20240502205509.png]]
count_if：查找区间中满足某个条件的元素个数。
![[Pasted image 20240502205624.png]]
以下容器自带count函数：
![[Pasted image 20240502205704.png]]
### find_xx
find：寻找等于value的第一个元素。
![[Pasted image 20240502210002.png]]
find_if：寻找满足某个条件的第一个元素。
![[Pasted image 20240502210046.png]]
以下容器自带find函数:
![[Pasted image 20240502210129.png]]
### binary_search
对已排序的区间进行二分查找，判断区间中是否有value。
![[Pasted image 20240502211303.png]]

lower_bound返回区间中大于等于value的第一个元素的迭代器。
![[Pasted image 20240502211635.png]]

# 仿函数
一个重载了()操作符的类。
stl中的几种仿函数：
![[Pasted image 20240502213145.png]]
仿函数主要用于算法的使用，很多算法中的参数必须是一个函数，所以即使是加减这样简单的操作也需要将他封装成函数。
使用sort算法时使用仿函数：
![[Pasted image 20240502213930.png]]
上面的stl内置仿函数继承了binary_function，它的定义如下：
![[Pasted image 20240502214937.png]]
这是一种stl规范。主要是需要获得binary_function中typedef定义的3个参数，如果不做这种继承，可能会导致获取不到3个参数而报错。如果一个functor继承了binary_function或unary_function，那么它们就是可适配的（adaptable）。

# 适配器
Adapter用于改造原有的接口或者类，它并不是一个单独的模块，而是根据改造对象的不同分为 Itreator Adapter,  Container Adapter和Functor Adapter（主要）。
Adapter都不使用继承方式来使用原有接口，而是使用组合的方式。
Container Adapter最常用的就是stack和queue，它们底层使用deque来实现它们的功能。

## Functor Adapter
### binder2nd
binder2nd绑定一个函数的第二个参数，比如functor less\<int>(x, y)，用来比较x和y谁更小，
binder2nd(less\<int>(), 40)，就是一个用来比较x和40谁更小的functor。他把less\<int>的第二个参数绑定为了40。
binder2nd代码：
![[Pasted image 20240504223150.png]]
binder2nd继承了unary_function，说明他也是一个adaptable的functor。
它有两个成员：
1. op  记录需要绑定的functor
2. value  记录绑定的第二个参数
并且它重载了()操作符，当调用binder2nd(x)时，`return op(x, value)` 就成功调用了functor并返回了绑定完第二个参数的functor的执行结果。
然而，需要注意的是，binder2nd是一个类模板，使用它需要先指定模板参数，所以按照上面的示例，使用binder2nd(less\<int>(), x)应该时是这样使用的binder2nd\<less\<int>>(less\<int>(), x)，看起来就很麻烦，有没有一种方法自动在使用binder2nd就获取到op的类型呢？标准库中提供了这种更简便的方法：
![[Pasted image 20240504224602.png]]
这个辅助函数可以直接根据参数的类型来创建binder2nd\<Operation>对象（模板函数直接获取到传入参数类型的特性，模板类必须指定类型才能实例化），所以一般都会直接使用bind2nd函数来绑定functor的第二个参数。

> [!NOTE] 关于typename的使用
> 在上面的很多例子中，出现过 `typedef typename xxx:xx xxx_type` 这种写法，typedef就是用来为类型取别名的，为什么还需要typename这个关键字呢，这个跟具体的编译器实现有关，typename告诉编译器 xxx:xx 一定是一个类型，比如上面的 Operation:: second_argument_type一定是一个类型，不会是变量或者函数，防止编译器报错。

在最新的版本中，binder2nd已经被取代了，最新使用bind来实现这个功能：
![[Pasted image 20240504231900.png]]
 `#include <functional>` 后，就可以使用std::bind了。
 std::bind可以绑定以下的函数式：
 - functions 一般函数
 - function objects 函数对象
 - member functions 成员函数，\_1必须是object地址。
 - data members 成员变量，\_1必须是object地址。
### not1
not1与binder2nd相似，用于适配一个functor的取否。
![[Pasted image 20240505123653.png]]
![[Pasted image 20240505123708.png]]
## Itreator Adapter
迭代器适配器用来改造迭代器的功能。
### 反向迭代器reverse_iterator
顾名思义，reverse_iterator从后向前遍历容器，与普通迭代器相反。
![[Pasted image 20240502210950.png]]
反向迭代器的++算法是从后向前移动。
![[Pasted image 20240502211113.png]]
一个需要注意的地方是，普通的end()迭代器指向最后一个元素的后一个元素（其实是空），反向的rend()迭代器指向第一个元素的前一个元素（也是空）。
reverse_inerator代码：
![[Pasted image 20240505130752.png]]
注意operator\*操作，它取值是对普通迭代器的前一个元素取值。
### 插入迭代器inserter
在了解inserter之前，先看普通的copy算法：
![[Pasted image 20240505131649.png]]
它将一个区间内的元素复制到result之后的区间，但是并没有对result之后的区间的合法性做检测。
而inserter可以在copy时执行插入的操作，也就是说，当你copy到result之后的区间时，不是简单的++然后赋值，而是在result之后插入了新的节点。
inserter代码：
![[Pasted image 20240505134229.png]]
辅助函数inserter从Container x和它的迭代器i中构造一个inserter_inerator。
inserter_inerator有两个数据，container是底层的容器，iter是传入的迭代器。
重点在于operater=的重载，在copy时本来是简单的赋值操作，这里将它改造成向此处插入新的节点。
### ostream_iterator
用于输出的迭代器，先看它的使用示例：
![[Pasted image 20240505135804.png]]
out_it就是一个ostream_iterator，它初始化时使用了std::cout和“,”，当元素进入它时，会自动使用std::cout输出它，并且每个元素使用","来隔开。
ostream_iterator的实现：
![[Pasted image 20240505140222.png]]
他有两个成员：
- out_stream是一个basic_ostream引用，上面的例子用std::cout来初始化它
- delim就是分隔符
ostream_iterator也是重载了operatot=，当它被赋值时，std::cout就会输出该元素。
注意它的operator++和operator++(int)操作符，它们都直接返回当前对象的引用，也就是说stream_iterator本质上没有执行后移操作，比起一个迭代器，它更像是一个类。
### istream_iterator
与ostream_iterator相对，istream_iterator执行输入操作。
使用示例：
![[Pasted image 20240505141849.png]]
在这里，istream_iterator iit从缓冲区中取出用户输入的值来给value1和value2，直到取到endl符号。
istream_iterator代码：
![[Pasted image 20240505143643.png]]
成员：
- in_stream 输入流，用std::stream初始化
- value 输入值
当对象创建时，自动执行了++操作。
operator++操作中，如果std::cin读取到了值（这里会阻塞直到读取到值），将其放入value中。（为什么要把in_stream置为0）
另一个使用的示例：
![[Pasted image 20240505145120.png]]
这里使用std::cin一直读取数据，并且把它插入到容器c中，直到读取到endl为止。
# STL其他
## 一个万用的Hash Function
如何写一个Hash Function
假设要为class Customer写一个哈希函数，
第一种方法是直接写一个函数，传入Customer对象，返回它的哈希值，然后在使用unordered容器的时候将该函数作为模板参数传递。
![[Pasted image 20240505150955.png]]
第二种方法是使用Functor形式重载operator()。
![[Pasted image 20240505151535.png]]

假设Customer中有3个成员fname, lname, no，最容易想到的是将3个成员使用stl中的哈希函数生成哈希值，再将他们组合起来，比如相加，
![[Pasted image 20240505151939.png]]
这样虽然可以运作，但是哈希值重复概率大，不是一个好的设计。
hash_val是stl（version > 2013）自带的一个哈希函数，可以让我们更方便地生成哈希函数。
可以这样使用hash_val：
![[Pasted image 20240505152353.png]]
hash_val使用模板的可变参数方法，可以传入任意个参数，
![[Pasted image 20240505152523.png]]
在这里又调用了一个hash_val，但是它多了一个参数seed，
![[Pasted image 20240505152729.png]]
注意seed是引用传递，它就是最终生成的哈希值。在这个函数中，又调用了hash_conbine函数和hash_val函数，hash_val函数就是它自己，hash_combine就是真正生成哈希值的函数，
![[Pasted image 20240505153846.png]]
当hash_val调用到只剩最后一个参数时，匹配到下面这个hash_val，
![[Pasted image 20240505154012.png]]
也是调用了hash_combine。
hash_combine中的哈希算法有一个数字0x9e3779b9，它的来源：
![[Pasted image 20240505154445.png]]

除了上面两种哈希函数的使用方式，还可以使用偏特化的形式来使用哈希函数，注意到unordered容器是这样定义的：
![[Pasted image 20240505155308.png]]
这里以unordered_set为例，它的第二个模板参数是哈希函数hash\<T\>，我们当然可以通过偏特化来定义hash\<T\>，比如：
![[Pasted image 20240505155530.png]]
这样就为MyString类定义了自己的哈希函数。
## tuple
tuple一般称为元组，可以指定任意类型的变量到一个结构里面，有点像结构体。
tuple使用示例：
![[Pasted image 20240506175826.png]]
![[Pasted image 20240506180424.png]]
上面演示了如何使用tuple。
从上面的例子中可以看出一些tuple的特性：
使用make_tuple可以构造tuple对象。
get<1>(tupleObj)可以取tuple的第一个数据，并且可以为它赋值。
tuple对象之间可以比较大小。
tuple对象之间可以直接赋值。
tuple_size\<TupleType\>::value可以知道tuple有几个成员，tuple_element\<1, TupleType\>::type可以知道第一的成员数据的类型。
tuple的实现：
![[Pasted image 20240506181856.png]]
tuple的模板参数有两个，Head和Tail...，依然是使用可变模板参数。
值得注意的是，tuple类继承了它自己，但是只继承tuple\<Tail...\>，也就是说，如果原来有3个模板参数，该tuple类自己保留第一个（变量m_head），然后继承有2个模板参数的tuple类，依次类推，直到没有任何模板参数的tuple类。

一个示例：
![[Pasted image 20240506182207.png]]
它的继承关系为
![[Pasted image 20240506214638.png]]
当继承到最后没有参数的tuple类时，有一个偏特化的版本：
![[Pasted image 20240506215131.png]]

tuple中有两个函数，head()和tail()，
head()就返回成员m_head的值。
tail()返回父类的引用，其实是返回\*this再强制转换为tuple\<Tail...\>类型。子类强制转换为父类是安全的。
## type traits
![[Pasted image 20240507214453.png]]
在最上面的泛化版本中，type traits “回答”了几个问题，默认构造函数是否不重要，拷贝构造函数是否不重要，赋值运算是否不重要...默认都是false，也就是说，默认这些都是重要的。
下面特化了一个int和double版本，它们认为自己的默认构造函数这些是不重要的，这样也许在某些算法中就可以通过type traits获取这些信息，好进行一些优化操作。
从上面的例子中，就可以看出type traits的设计思想，使用模板的偏特化，将一个类型的一些信息保存起来，在将来使用这个类型时可以通过统一的方式获取它的信息， type_traits\<T\>::value。
在C++最新的版本中，type traits中的信息更多：
![[Pasted image 20240507220128.png]]
![[Pasted image 20240507220213.png]]
使用示例：
![[Pasted image 20240507220506.png]]
![[Pasted image 20240507220648.png]]
这里就通过type traits获取到了我们自定义的Foo类的各种信息。
那么type traits是如何实现获取一个类的信息的？
以is_void为例：
![[Pasted image 20240507223114.png]]
它收到模板参数\_Tp后，第一件事是使用remove_cv去掉const和volatile属性。
![[Pasted image 20240507223449.png]]
在这里它又调用了remove_const和remove_volatile两个模板。
remove_const：
![[Pasted image 20240507223652.png]]
他有两个版本，泛化版本直接将_Tp定义为type，const的特化版本拿掉const关键字，也将_Tp定义为type。
同理，remove_volatile:
![[Pasted image 20240507224048.png]]
现在原来的_Tp的参数已经去掉const和volatile了，再使用__is\_void_helper来确认它是否为void，\_\_is\_void_helper定义如下：
![[Pasted image 20240507224334.png]]
同样有泛化和特化两个版本，普通版本匹配到false_type，void类型的版本匹配true_type，这样就可以根据传入的类型来知道它是否为void了。

再看is_integral，用来判断一个类型是否为整数。
![[Pasted image 20240507224800.png]]

![[Pasted image 20240507225030.png]]
![[Pasted image 20240507225049.png]]
可以看到，bool，char，long long都匹配true模板，像未列出的double， float就匹配false模板。

更复杂的is_class，用来判断一个类型是否为一个class
![[Pasted image 20240507225433.png]]
其中__is_class没有源代码，也许是编译器内定义的宏，用来判断一个类型是否为class。
同理struct，union也可以通过这种方式知晓。

is_move_assignable判断一个类型是否可以移动赋值。
![[Pasted image 20240507225911.png]]
![[Pasted image 20240507230010.png]]
同样可能使用了编译器内定义的宏。
## cout
cout是一个ostream类，
![[Pasted image 20240509215236.png]]
它重载了<<运算符，所以它可以输出所有类型的数据。
## moveable对容器插入速度的影响
如果容器内的元素支持move操作，
在vector中，插入元素时move操作可以带来很大的性能提升，因为vector的扩容操作会进行大量的拷贝。
在其他容器中，插入元素时move操作并不会带来很大的性能提升。
但是如果是copy元素，move操作对所有容器都会带来很大的性能提升。 