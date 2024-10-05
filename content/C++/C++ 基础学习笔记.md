# ·C++ 基础

## 第三章 处理数据

### 整数

short: 至少16位

int: 至少与short一样长

long: 至少32位，且至少与int一样长

long long: 至少64位，且至少与long一样长

### 类型转换

1. 整数提升：计算表达式时，将bool，char，unsigned char，singed char和short转换为int类型，因为Int类型为计算机最自然的类型，处理速度可能最快。
2. 函数参数传递时也会char和short进行整形提升。
3. 强制转换格式（在C++中一般不使用）

```c++
int var;
long l_var = (long)var;  //与C相同
long l_var = long(var);  //C++新增，像函数调用一样使用强制类型转换
```

### 变量在头文件中定义

在头文件中声明变量时，为了防止变量的重复声明，可以使用预编译宏 #ifndef 来防止头文件重复包含引起的错误。但是在头文件中定义了变量时，#ifndef 宏就不行了，因为#ifndef只能保证#include预处理后在同一个文件中不会重定义，如果该头文件被多个cpp文件包含，就会出现在多个文件中定义了相同名称的全局变量。

同样函数也是如此，头文件中只能包含函数的声明，不要包含函数定义。

所以，不要在头文件中直接定义变量，更准确地说是不要让编译器在头文件中做分配内存空间的事。结构体和类可以在头文件中定义，因为他们实际上并未做出分配内存的动作。

以下情况是例外：

1. const 或 static 变量，这两种变量链接性为内部，多个文件不会相互影响。
2. 内联函数，内联函数允许多个定义，只要这些定义是相同的。
3. 类中直接定义的函数，这些函数被视为内联函数。

参考： [C++头文件保护符和变量的声明定义_小葱CC的博客-CSDN博客_c++ 头文件保护符](https://blog.csdn.net/wxc237786026/article/details/38377221)

类中直接定义的函数举例：

```c++
// 头文件
class test{
private:
    int a;
    void init();
};

void test::init()   // 错误，init()多次定义
{
    a = 0;
}

// 头文件
class test{
private:
    int a;
    void init()    // 正确，init() 视为内联函数
    {
        a = 0;
    }
};
```



## 第四章 复合类型

### 数组

1.将数组初始化为0

```c++
int array[10] = {};
int array[10] = {0};  //只提供第一个值时，会将其他未提供的值全部初始化为0
```

### 字符串

简单的字符串处理函数：

```c++
char* m_strcpy(char* dest, const char* src)
{
    char* p = dest;
    while(*src != '\0')
    {
        *p++ = *src++;
    }
    *p = *src;
    return dest;
}

int m_strcmp(char* str1, char* str2)
{
    while(*str1 != '\0' && *str2 != '\0' && *str1 == *str2)
    {
        str1++, str2++;
    }
    return *str1 - *str2;
}

size_t m_strlen(const char* str)
{
    size_t res = 0;
    while(*str++ != '\0') {
        res++;
    }
    return res;
}
```

### 结构体

在C++中，使用已定义的结构体不需要struct关键字，直接使用结构体名称就行了。

## 第7章 函数

### 函数与数组(Const的使用)

const 和 指针

const int * p  表示p是一个指向const int数据类型的指针，即p指向的数据不能被改变（*p = xxx 是不允许的）

int * const p  表示p是一个const *类型的指针，指向一个int数据，p的值是不能被改变的（p = &xxx或 p++是不允许的）

将常规变量的地址赋给指向const的指针是允许的，但是将const变量的地址赋给常规指针是不允许的

```c++
int a = 10;
const int * p = &a; //允许

const int PI = 3.14;
int* p = &PI; //不允许，防止通过 *p 修改PI的值
```

但是进入2级关系时，将常规变量的地址赋给指向const的指针也不被允许

```c++
const int ** p2;
int * p1;
const int PI = 3.14;

*p2 = &PI;
p2 = &p1; //不允许，防止使用p1来修改const值；
```

指向const的指针不能赋值给常规指针

```c++
const int PI = 3.14;
const int * p = &PI;
int * p2 = p;  //不允许
```

如果可能的话，将函数的指针参数声明为const，如 func (const int * a) 或 func (const a [])，这样做有两个好处

1. 提醒编程人员func函数不应该修改a指向的数据，否则编译将报错。
2. 使函数可以接受指向const的指针作为实参，否则只能接受常规指针作为实参。

### 函数与二维数组

函数如果接受二维数组作为实参，则对应形参必须数组的列数，如 func(int arr\[][5])，这种声明对实参的行数没有限制，比如

```c++
int a[4][4];
int b[10][4];
...
func(a);
func(b);  //都允许
```

### 函数指针

函数指针的声明

```c++
int (*pfunc)(int a);  // pfunc是一个函数指针，它指向的函数接收一个int型参数，返回一个int型参数

int func(int x);
pfunc = func; //初始化函数指针

int res = pfunc(9); //使用函数指针
int res = (*pfunc)(9); //使用函数指针，二者相同

int ((*pfunc)[3])(int a);  //函数指针的数组，pfunc是一个数组，数组中有三个函数指针

typedef int (*pfunc)(int a) pfunction; //使用typedef简化函数指针
pfunction p1 = func;   //使用pfunction声明函数指针

```

### 函数指针的使用——结构体实现类的功能

OOP中类的功能有三部分：封装、继承、多态

封装：封装的思想是仅将数据和接口暴露给使用者，而忽略内部实现。struct本身就具有数据封装的作用，再使用函数指针可以将方法和数据封装在一起，实现OOP中封装的大部分功能，但是访问权限的控制无法解决，因为struct中暴露出去的接口和数据肯定是public权限，而且实现继承必须知道struct的内存布局。

【base.h】

```c++
#pragma once
// 基类的实现
struct _Base;
typedef struct _Base Base;

// 使用函数指针封装接口，接口的参数都为void*是为了后面实现多态，类中自动包含this指针，但struct中没有，所以要显示传入this指针
typedef unsigned int (*pfGetProperty)(void* This);
typedef void (*pfSetProperty)(void* This, unsigned int);
typedef void (*pfPrint)(void* This);
typedef unsigned int (*pfSum)(void* This);

struct _Base {
    void* This;   //this指针
    unsigned int _property;
    pfGetProperty GetProperty;
    pfSetProperty SetProperty;
    pfPrint Print;
    pfSum Sum;
};

// 提供构造函数，控制struct实现细节
Base* NewBase(unsigned int);
void DeleteBase(Base* base);
```

【base.cpp】

```c++
#include "base.h"
#include <iostream>
// 接口的实现
unsigned int GetPropertyBase(void* This)
{
    return ((Base *)This)->_property;
}

void SetPropertyBase(void* This, unsigned int value)
{
    ((Base *)This)->_property = value;
}

void PrintBase(void* This)
{
    using namespace std;
    cout<< "This is Base class !"<< endl;
}
// 此方法可视为纯虚函数
unsigned int SumBase(void* This)
{
}

// 构造函数，将刚刚实现的接口赋值给函数指针，基类的this指针就是自己
Base* NewBase(unsigned int value)
{
    Base* base = (Base*)malloc(sizeof(Base));
    base->_property = value;
    base->This = base;
    base->GetProperty = GetPropertyBase;
    base->SetProperty = SetPropertyBase;
    base->Print = PrintBase;
    base->Sum = SumBase;
    return base;
}

void DeleteBase(Base* base)
{
    if (base->This == nullptr)
    {
        free(base->This);
    }
    base->This = nullptr;
    if (base == nullptr) {
        return;
    }
    free(base);
    base = nullptr;
}

```

继承：继承主要用到的思想是 “结构在内存中的布局与结构的声明具有一致的顺序”，在子类中保存基类的对象，就可以获得基类的数据和接口，将基类的对象放在子类struct的第一个元素的话，将子类强制转换为基类是完全可以的。

【child.h】

```c++
#pragma once

#include "base.h"
// 因为子类实现不需要暴露，将结构体的实现放到cpp文件中，起到更好的封装作用
struct _Child ;
typedef struct _Child Child;
// 构造函数返回的是一个基类的指针，因为所有的接口都定义在基类中
Base* NewChild(unsigned int, unsigned int, unsigned int);
void DeleteChild(Base* child);
```

【child.cpp】

```c++
#include "child.h"
#include <iostream>
// 基类的对象放到第一个元素，在基类的几个接口中都是将this指针强制转换为基类指针,因为“结构在内存中的布局与结构的声明具有一致的顺序”，这样即使传入的是子类指针也不会错误(x和y将会被丢弃)
struct _Child {
    Base base;
    int x, y;
};
// 子类实现自己的方法
void Print_Child(void* This)
{
    using namespace std;
    cout<< "This is Child class !"<< endl;
}
// 实现基类未实现的方法
unsigned int Sum_Child(void* This)
{
    Child* child = (Child*)This;
    return child->base._property + child->x + child->y;
}

Base* NewChild(unsigned int baseValue, unsigned int x, unsigned int y)
{
    Base* base = NewBase(baseValue);
    Child* child = (Child*)malloc(sizeof(Child));

    child->base = *base;
    child->x = x;
    child->y = y;

    base->Print = Print_Child;
    base->Sum = Sum_Child;   //这里将基类的默认方法替换掉了
    base->This = child;  //子类的构造函数实际返回的是基类指针，但是this指针指向自己，运行时使用this指针作为接口的参数
    return base;
}
```

多态：要在运行时实现多态，主要依靠this指针指向的对象不同

【main.cpp】

```c++
    Base *base = NewBase(6);
    base->Print(base->This);  // 基类提供使用接口的方法，真正传入的参数是this指针
    base->SetProperty(base->This, 8);
    EXPECT_EQ(base->GetProperty(base->This), 8);
    DeleteBase(base);

    Base *child = NewChild(5, 9, 4);
    child->Print(child->This);  // this指针是指向子类的，在子类的构造函数在已经将一些方法替换掉了，所以可以实现多态
    child->SetProperty(child->This, 7); // 调用基类的方法也没问题，this指针会被强制转换为基类
    EXPECT_EQ(child->GetProperty(child->This), 7);
    EXPECT_EQ(child->Sum(child->This), 20);
    DeleteChild(child);
```

## 第8章 函数探幽

### 引用变量

引用时已定义变量的别名，可以和变量交换使用，和被引用的变量指向相同的值和内存单元。

引用与指针作用相似，但是他们也有一些差别

1. 引用变量必须在声明时进行初始化。

2. 一旦一个引用和变量关联起来，就无法再改变它的引用。所以引用的值只能通过初始化来设置，而不能通过赋值来设置。

   ```c++
   int a;
   int &ref = a;
   类似与以下代码
   int* const ref = &a;
   ```

3. 存在多级指针，但是不存在多级引用。

4. 某些运算比如++，sizeof，对引用使用时，相当与对引用指向的变量使用，而对指针使用时，相当于对指针类型的变量使用。

   

引用作为函数参数时参数传递方式是引用传递而不是值传递，也就是说可以在函数类改变参数的值，但是引用传递对参数的要求更加严格

```c++
void func1(int value); 
void func2(int& value);

int a = 10;
func1(a); //值传递，函数类改变a的值是没用的
func2(a); //引用传递，可以在函数类改变a的值

func1(a + 1); //允许使用
func2(a + 1); //编译报错， a + 1是右值
```

但是如果函数的引用参数声明为const，也就是说函数虽然传递了引用，但是并不会改变他们，这时就可以将右值也传递给函数

```c++
void func(const int& value);

int a = 10;
func(a + 1); //允许使用，参数的类型正确，但是并不是左值
short b = 10;
func(b); // 允许使用，参数类型不正确，但是short可以转化为int类型，而且b是左值
```

这是由于右值的传递（传递给函数参数或者引用）需要生成一个临时变量，而引用就是为了防止临时变量的产生，为一个引用加上了const，编译器就会允许临时变量的产生。



由于OOP的继承特性，父类的引用可以指向子类的对象（指针也一样），而不需要强制类型转换。

```c++
Child child;
Base& ref = child; //支持这种操作
Base* ref2 = &child; //支持这种操作
```



引用最大的使用场景就是作为函数的参数和返回值，主要用于以下两个方面：

1. 函数需要引用传递来修改实参的值，这时使用引用就像使用指针一样可以间接寻址来修改实参的值。

2. 函数传递的参数或返回值为结构体或者类，使用引用可以防止临时变量的产生，提高程序的运行速度。但是这种方式也使函数的调用更加严格：

   2.1 函数的形参为引用时，会导致无法传递右值作为函数实参，这时可以加上const（允许临时变量的生成，但是无法再提升效率）    或者使用移动语义来避免。

   2.2 函数返回引用时，需要考虑返回类中私有成员时的安全性，这时可以返回一个const引用，防止调用函数时将函数的返回值结果赋给一个普通引用来修改类中私有成员的值。

   2.3 函数返回引用时，由于返回的引用是一个左值，可能会出现 func() = value 这种形式的使用，虽然没有语法错误，但是让人难以理解，这时也可以返回const引用来禁止这种使用方式。

   2.4 返回引用时，就像返回一个指针一样，需要考虑到所返回的内存的合理性，不能返回函数中临时创建的变量，因为函数调用结束后临时变量就销毁了，最好也不要返回New或者malloc分配出的内存，除非可以保证内存可以正常释放。
   
3. 使用const引用作为参数来代替普通的参数，除了可以防止临时变量生成，还可以将函数用于没有复制构造函数的类型（没有复制构造函数是没法生成临时变量的），虽然大多数类都是可以复制构造的，但是也有一些类会删除复制构造函数。



### 函数重载

函数重载是函数多态的一种，可以定义参数不同的同名函数来应对不同的使用场景，在C++中函数名可以相同，只要参数不同，编译器会将其识别为不同的函数。



```c++
void Print(char *);
void Print(int* a, unsigned int size);

char* str = "hello";
int a[5] = {0};
Print(str); //调用第一个函数
Print(a, 5); //调用第二个函数
```



### 默认参数

函数可以使用默认参数在未提供实参时提供一个默认值，但是函数参数从右到左传递，所以如果一个参数提供了默认参数，那么它右边的参数必须也提供默认参数。

```c++
int func(int x, int y = 10);

// 可以这样使用func
func(2);
func(7, 9);
```



### 函数模板

需要对不同的数据类型使用同一种算法时，可以使用函数模板，比如Swap函数用于交换数据，但是交换的数据类型有多种，可以使用如下函数模板:

```c++
template <typename T>
void Swap(T& a, T& b);
```

如果针对某种类型，提供的算法不同，可以将函数模板显示具体化

```c++
template <> void Swap(Obj& a, Obj& b)
{
    ... //code
}

template <> void Swap<Obj>(Obj& a, Obj& b)
{
    ... //code
}
```

从模板中自动生成函数定义称为隐式实例化，同样也可以显示实例化函数模板

```c++
template void Swap<int>(int&, int&);
```

**Note**: 有的编译器不支持函数模板分离编译，所以函数模板声明与实现分开可能会导致错误，可以用以下方法解决：

1. 将函数模板的实现直接写到头文件中（STL中是这么做的）
2. 在实现或调用模板函数的文件中显示实例化需要的函数模板



### 函数匹配

对于函数重载，模板重载和函数模板重载，编译器在实际调用时需要确定最终调用哪一个函数，函数的匹配步骤为：

1. 创建函数候选列表，包含与被调用函数名称相同的所有模板函数和函数。比如调用了一个Max()函数，候选列表中会包含所有名称为MAX的函数和函数模板。
2. 创建可行的函数列表，包含参数完全匹配并且实参类型与形参类型匹配的函数，可以执行隐式转换，比如实参为float，形参为double则也可以匹配。
3. 确定最佳匹配函数。

确定最佳匹配函数有以下原则，从最佳到最差依次为：

1. 完全匹配，常规函数优于函数模板。
2. 提升转换。（char ,short -> int,  float -> double）。
3. 标准转换。（int -> char, long -> double）。
4. 用户自定义转换。

完全匹配时，允许无关紧要的转换，主要有：

1. Type和Type&之间的转换。
2. Type[]转换为Type*。
3. Type转换为const Type或volatile Type。
4. Type\*转换为const Type\*或volatile Type\*。

当两个模板都匹配时，编译器将使用较具体的函数模板，也就是所做转换比较少的函数，比如：

```c++
template <typename T>
void func(T value);

template <typename T>
void func(T* value);

int value = 6;
func(&value); //将使用第二个模板，因为&value到T*所做的转换更少
```

### decltype关键字

decltype可以表示一个变量或表达式的类型，使用方法为 decltype(expression) var ：

```c++
int x;
decltype(x) y; // y的类型为int

double z = 0.5;
decltype(x + z) y; // y的类型为double
```

decltype的使用有以下原则

第一步：如果expression是一个没有用括号括起的标识符，则var类型与标识符相同。

```c++
int x;
decltype(x) y; // y的类型为int
```

第二步：如果expression是一个函数调用，则var类型与函数返回值相同（并不会真正调用函数）。

```c++
int func(int x);
decltype(func(7)) y; // y的类型为int
```

第三步：如果expression是一个用括号括起的标识符，则则var类型为标识符类型的引用。

```c++
int x;
decltype((x)) y; // y的类型为int&
```

第四步：如果都不满足以上情况，var类型与expression类型相同（结果与第一种情况相同）。

```c++
int x;
double z = 0.5;
decltype(x + z) y; // y的类型为double
```

decltype一般用于函数模板内。因为无法确定函数模板中真正传入的参数类型，在函数模板中需要声明与传入变量类型相关的变量时就可以使用decltype。

```c++
template <typename T1, typename T2>
void func(T1 x, T2, y)
{
    decltype(x + y) sum = x + y;
    ...
}

// 当返回值不确定时，需要使用C++新的函数声明方法
template <typename T1, typename T2>
??? func(T1 x, T2, y)
{
    ...
    return x + y;
}
// 改为
template <typename T1, typename T2>
auto func(T1 x, T2, y) -> decltype(x + y)
{
    ...
    return x + y;
}
```

## 第9章 内存模型和名称空间

### C++内存分布

![img](https://img2020.cnblogs.com/blog/1309603/202106/1309603-20210608154310188-1142140606.png)

### 存储持续性，作用域和链接性

作用域描述了名称在文内内多大范围可见，链接描述了名称如何在文件之间共享。

链接性分为3种：

外部链接性：可在其他文件中访问。

内部链接性：只能在本文件内访问。

无链接性：只能在函数或代码块中访问。

#### 自动存储持续性

函数中定义的变量存储持续性为自动，他们程序执行函数或代码块时被创建，函数或代码块执行完毕后就销毁，储存在内存的Stack区中。

自动变量的作用范围为定义开始到函数结尾，如果在函数中有开辟可一个代码块，则该代码块中声明的变量作用范围仅为代码块内，如果在代码块声明了与之前函数中变量名称一样的变量，则在代码块中，该变量会隐藏之前函数中声明的变量。

```c++
int a = 10;
{
    int a = 7;
    a++;
    cout << a << endl; // a = 8
}
cout << a << endl; // a = 10
```

**Note**:

C++ 11中，auto关键字用于自动类型推断。但是在C语言中，auto关键字用于显式指出变量为自动储存：

```c++
auto float a = 10;
```

因此这种关键字在C语言中几乎用不到。在C++中，使用关键字register来代替C语言的auto关键字功能。在C语言中，register用来建议将变量储存在寄存器中以提高访问速度，现在这种作用在C++11以后已经失去了。

#### 静态持续变量

静态持续变量在整个程序运行期间都存在，储存在C++内存的全局数据区中，如果变量未出世化，储存在DSS段，如果变量已初始化，储存在DATA段。静态持续变量在未出世化时，会被初始化为0，这个过程叫做零初始化或者静态初始化。

```c++
int a = 10;  // 外链接性的静态持续变量，也叫全局变量, 存储在data段。
static int b; // 内链接性的静态持续变量，未初始化所以储存在bss段。static作用为表示该变量为内连接
void func()
{
    static int c = 0; // 无连接性的静态持续变量，初始化为0，储存在bss段。static作用为表示该变量是静态储存的。
    ...
}
```



变量a是外链接性的静态持续变量，它可以在外部的文件中访问。C++声明变量分为定义声明和引用声明，定义声明会为该变量分配储存空间，而引用声明引用已有的变量，不会分配储存空间，因为它引用已有的变量。引用声明使用关键字 extern：

```c++
extern int a;  // 引用其他文件定义的变量a
```

extern也用于显式声明该变量是外链接性的，但是在使用中一般会省略它。

```c++
// 函数外声明
int a = 10;
extern int a = 10; // 二者效果相同
```



变量b是内链接性的静态持续变量，只有本文件可以使用它，使用此种变量可以减少重复定义问题，全局变量的使用应该是很少的，但是如果两个文件中恰好定义了同名的全局变量，程序将会报错。所以如果一个全局变量只是为了在此文件中使用的话，应当把他声明为内链接性的。

变量c是无连接性的静态持续变量，它在程序运行期间一直存在，但是只用在定义该变量的函数和代码块中才能使用它。同时，如果初始化了静态变量，程序只在启动时进行一次初始化，以后再次调用函数该变量也不会再初始化了。

```c++
void func()
{
    static int a = 3;
    a++;
    cout << a << endl;
}

int main (int argc, char **argv)
{
    func();  // 输出4
    func();  // 输出5

    return 0;
}
```



同样，函数也有存储连续性和链接性，C/C++不允许在函数内定义函数，所以函数一般都是静态存储的，即在整个程序运行期间函数一直存在。同样也可以使用static关键字声明函数为内链接的，

```c++
static void func(int a);
```

这样此函数就只能在此文件中使用，否则函数都是外链接的。遵循单一定义原则，每个函数都应该只有一次定义，否则编译会报错，而内联函数没有此限制，它可以定义多次，但是所有定义必须是相同的。



#### 说明符和限定符

存储说明符有以下几个：

- auto （C++11中不再是说明符）
- register（C++11之前用于说明变量存储在寄存器中，之后与原auto相同，用于说明变量是自动存储的）
- static（指出局部变量是静态存储的或者全局变量是内连接的）
- extern （用于引用声明，该变量在其他文件中定义）
- thread_local（变量在线程运行期间存在）
- mutable（指出即使结构体或类对象是const的，其某个成员也是可以修改的）

C-V限定符

- const（指出内存初始化后，程序不能对其做出修改）
- volatile（表明即使程序不对某个内存做出修改，它存储的值也可能发生变化，比如被硬件或者其他程序修改。使用这个声明是为了告诉编译器不要将变量的值优化到寄存器中，因为程序在几条语句中多次使用了一个变量时，为了提高运行速度，会将该变量的值放到寄存器中以便于下次访问）

const除了声明常量外，当他对一个全局变量使用时，也会将该变量的链接性声明为内链接的

```c++
const int PI = 3.14; // PI为内链接的，即只在本文件范围内有效
```

如果需要显式声明外链接性的常量，则需要加上extern关键字。

```c++
extern const int PI = 3.14; // PI为外链接的，可在其他文件中访问
```



#### 语言的链接性

语言的链接性影响函数的使用。C语言中一个名称只对应一个函数，所以编译器可能将一个func(int)函数翻译成\_func，而C++允许函数重载，所以可能将func(int)函数翻译成\_func\_int，这种方法称为C++的语言链接。所以如果想要在C++中使用C预编译后的函数，需要加上 extern "C"。

```c++
extern "C" void func(int); // 编译器会去寻找_func这种C语言编译器翻译风格的函数名称
```

更加通用的用法是

```c++
/* 如果采用了C++,下述代码extern "C"有效，extern "C"{}中代码使用C编译器 */
#ifdef __cplusplus
        /* 如果没有采用C++，顺序预编译 */
    extern "C" {
#endif
 
... /* 采用C编译器编译的C语言代码段 */
 
#ifdef __cplusplus          
    }           /* 结束使用C编译器 */
#endif
```



### 存储方案与动态分配

使用new和delete运算符来控制动态内存的分配，即Heap去内存的分配。类似于C中的malloc()和free()函数。

#### new和delete运算符的使用

```c++
// -----后面加括号初始化值，用于有合适构造函数的类-----------------
int* p = new int (6);
float* pd = new float (99.99);

delete p;
delete pd;

// -----大括号的列表初始化，c++11后使用-------------------------
struct where {double x, double y}
where* wh = new where {3.4, 6.5};
int* pd = new int[4] {5, 6, 7, 8};
// 也可以这样使用
int* p = new int {6};
float* pd = new float {99.99};

delete wh;
delete[] pd;
```

new和delete运算符会调用分配函数和释放函数

```c++
void * operator new(std:: size_t); // new算符的分配函数
void * operator new[](std:: size_t); // new[]算符的分配函数

void operator delete(void *); // delete运算符的释放函数
void operator delete[](void *); // delete[]运算符的释放函数

int* p = new int;
int* p = new(sizeof(int)); // 上一条语句将会被转换成此语句

int* p = new int[4];
int* p = new(sizeof(int) * 4); // 上一条语句将会被转换成此语句
```

new和delete运算符都是可以重载的，也就是说，他们对应的分配和释放函数可以更具需要来定制。

#### 定位new运算符

除了常规的new运算符外，还有可以指定地址的定位new运算符，用来指定程序员需要使用的特定地址。一般用来通过特定地址访问硬件或者在特定位置创建对象。

```c++
char buffer[512];

int * p = new(buffer) int[4];
std::cout << p << " " << (void *)buffer << std::endl; // buffer强转成 void* 类型防止打印字符串
EXPECT_EQ((int)p, (int)buffer);
```

输出结果：

```
[ RUN      ] testCase.NewOperatorTest
0x69f9fc 0x69f9fc
[       OK ] testCase.NewOperatorTest (0 ms)
```

可以看到定位new运算符将buffer的地址分配给了p。

定位new运算符提供了更强大的功能让程序员可以自己决定使用的地址，同时也将内存管理的负担交给了程序员。同时，与普通new运算符不同的是，定位new运算符分配的内存不位于Heap区，不能用delete运算符释放内存，需要程序员自己对分配的内存进行管理。



### 名称空间

名称空间一般用于大型项目中，解决不同厂商或模块之间的命名冲突问题。

#### 使用名称空间

名称空间用以提供一个声明的命名区域。两个不同的名称空间不会出现命名冲突问题，就像两个函数内不会出现命名冲突问题一样。

```c++
namespace NameTest
{
    int a = 10;
}

namespace NameTest2
{
    int a = 19;
}
```

默认情况下名称空间中声明的变量链接性都是外部的，也就是所其他文件可以使用，除非使用static或const关键字限制链接性为内部。

使用名称空间中声明的变量时，一般用作用域解析符:: 

```c++
TEST(testCase, NameSpaceTest)
{
   EXPECT_EQ(NameTest::a, 10);
   EXPECT_EQ(NameTest2::a, 19);
}
```

对于全局变量，省略名称空间的名称来访问。

```c++
void func()
{
    ::value = 10;
    ...
}
```



#### using声明和using编译指令

为了简化名称空间的使用，可以使用using声明和using编译指令，using声明是名称空间中的某个变量可用，using编译指令使整个名称空间可用。

using声明

```c++
namespace First
{
    int x = 10;
    double y = 9.8;
}

TEST(testCase, NameSpaceTest)
{
   using First::x;
   EXPECT_EQ(x, 10); // 可使用x代替First::x，但是此函数内不能再有名为x的变量
}

using First::x; // 这样使用时将x添加到了全局名称空间
```



using编译指令

```c++
namespace First
{
    int x = 10;
    double y = 9.8;
}

TEST(testCase, NameSpaceTest)
{
   using namespace First;
   EXPECT_EQ(x, 10);
   EXPECT_EQ(y, 9.8); // x和y都可以直接使用
}

using namespace First; // 把First中的所有变量添加到了全局名称空间
```



using声明与using编译指令的区别在于，using声明就像声明了一个变量，不允许同一作用域内再出现其他同名变量，而using编译指令类似于使用大量的域解析符，同一作用域内可以再出现其他同名变量，并且这些同名变量会隐藏名称空间中的变量。

```c++
namespace First
{
    int x = 10;
    double y = 9.8;
}

TEST(testCase, NameSpaceTest)
{
   using First::x;
   int x; // 错误
}

TEST(testCase, NameSpaceTest)
{
   using namespace First;
   int x; // 正确，后面的x使用的将是这个x
   x = 10; // 使用局部变量x而不是First::x
}
```

所以，using声明比using编译指令要安全，因为它可以及时提醒程序员是否使用了错误的名称。



#### 名称空间的其他特性

1. 名称空间可以嵌套。

```c++
namespace First
{
    namespace Second
    {
        int flame;
        ...
    }
}

First::Second::flame; //访问flame
```

2. 名称空间中可以使用using声明和using编译指令，using编译指令是可传递的。

```c++
namespace First
{
    int flame;
}
// -----------using声明-----------
namespace Second
{
    using First::flame;
    flame = 10; // OK
}
// -----------using编译指令--------
namespace Second
{
    using namespace First;
    flame = 10; // OK
}
First::flame = 10;
Second::flame = 10; //都可以
// ---------using编译指令传递-------
namespace Third
{
    using namespace Second;
    flame = 10; // OK
}
```

3. 可以为名称空间创建别名，一般用来简化嵌套名称空间的使用。

```c++
namespace SF = Second::First;
using SF::flame; // 简化使用
```



## 第10章 对象和类

### 构造函数

类的构造函数专门用于创建新的对象，构造函数没有返回值，在创建对象时自动调用。

类的声明

```c++
class Stack
{
private:
   int maxCount = 20;
   int *items;
   int top;
public:
   Stack(int maxCountValue = 10);
};
```

实现

```c++
Stack::Stack(int maxCountValue)
{
    maxCount = maxCountValue;
    items = new int[maxCount];
    top = 0;
}
```

使用构造函数，

```c++
TEST(testCase, StackTest)
{
    Stack stack(20);
    Stack stack1 = Stack(20);
    
    Stack *stack2 = new Stack(20);
    delete stack2;
}
```

构造函数是无法通过类对象调用的，比如

```c++
stack.Stack(20); // 错误，因为在构造函数构造对象之前，对象是不存在的，也就无法通过对象调用成员函数，
```

编译器还会为类提供一个默认构造函数，但是只有在未提供显式的构造函数的情况下才会生成，也就是说，假如已经有了显示的构造函数，以下使用方法会报错：

```c++
Stack stack3;  // 错误
```

如果要避免这种错误，可以通过函数重载的方式再提供一个没有参数的构造函数：

```c++
class Stack
{
private:
   int maxCount = 20;
   int *items;
   int top;
public:
   Stack(int maxCountValue);
   Stack();  // 再提供一个无参的构造函数
};
```

```
Stack::Stack()
{
    maxCount = 10;
    items = new int[maxCount];
    top = 0;
}
```

这时就可以这样使用了

```c++
Stack stack3; // 使用默认构造函数时不使用()
```

在编程时，一般都要提供一个无参的默认的构造函数对成员数据进行隐式的初始化，或者提供一个有默认参数的构造函数作为默认构造函数。如果使用的对象数组，则一定要提供一个默认构造函数。因为对象数组初始化时首先使用默认构造函数创建数组元素，

### 析构函数

析构函数在对象销毁时会被调用，和构造函数一样，如果不显式地提供一个析构函数，编译器会提供一个默认的析构函数，但是由于析构函数一般不会显式地调用，所以不用担心出现提供了显式的析构函数后默认析构函数就没法直接调用了。

析构函数一般在以下情况下被调用：

- 静态对象在程序结束销毁时调用
- 局部变量在离开作用域销毁时调用
- 使用delete释放对象时调用

析构函数的使用：

```c++
class Stack
{
private:
   int maxCount = 20;
   int *items;
   int top;
public:
   Stack(int maxCountValue);
   Stack();
   ~Stack(); // 声明析构函数
};
```

```c++
Stack::~Stack()
{
    delete items; // 因为构造函数中使用new申请了内存，所以在析构函数中释放它。如果构造函数中没有动态申请内存，析构函数中可以什么也不做
}
```

析构函数也可以带参数，但是这些参数必须有默认值。

### 列表初始化

在C++11中，允许使用列表的方式对类进行初始化，比如

```c++
    Stack stack4 = {20};
    Stack stack5 {20};
```

这种初始化方式需要提供与构造函数参数相匹配的值。

### const成员函数

如果一个成员函数不会修改当前对象，应该将其声明为const函数，语法为

```c++
unsigned int GetSize() const;  // 在函数后加上const表示这是一个成员函数


// 定义 
unsigned int Stack::GetSize() const
{
    return size;
}
```

将成员函数声明为const相当于限制了传入的this指针为const, 即 const T * this。同时，如果一个对象是const的，那么它将不能调用非const的成员函数，因为这些函数不能保证不会修改当前的对象。

### this指针

每个类的成员函数（包括构造和析构函数）都包含一个this指针，它指向当前的对象。如果想使用当前对象，可以使用*this。一般来说，在成员函数中访问成员变量时都省略this指针：

```c++
unsigned int Stack::GetSize() const
{
    return size;
    // 等价于 this->size;
}
```

### 类作用域和类作用域内的枚举

代码中每个变量都有自己的作用域，如文件，代码块作用域。类也有自己的作用域，在类中定义的变量和函数都是类作用域的，他们在类中可以直接使用，但是在类外只能通过对象来使用，同样，在定义成员函数时，必须使用作用域解析运算符::。构造函数是例外，因为它与类名是相同的。



如果在同一个作用域内，两个传统的枚举中如果定义了相同的枚举名称，则会发生错误：

```c++
enum Enum1{One, Two};
enum Enum2{One}; // 错误，重复定义
```

可以使用类作用域的枚举来解决这个问题，

```c++
enum class Enum1{One, Two};
enum class Enum2{One};  // 也可以将class换为stuct
```

这时由于两个枚举的作用域不同，将不会发生命名冲突，但是访问枚举变量时需要用到域解析运算符，

```c++
Enum1 var = Enum1::One;
```

同时，类作用域内的枚举将不会自动转换为int型数据，

```c++
int var = Enum1::One; // 错误
```

一般来说，虽然类作用域内的枚举无法自动转换为int类型，但是底层的实现都是int的。也可以自己指定枚举的底层类型，

```c++
enum class Enum2 :short{One}; // 指定底层类型为short
```

这种方法也适用于常规的枚举类型。

## 第11章 使用类

### 运算符重载

C++中可以对运算符进行重载，提供更方便的对象操作，比如可以重载加法运算符来自定义两个对象之间的加法操作，重载运算符的格式为：

```c++
operator OP (argument-list)  // OP 为运算符，可以是 + - * / 等
```

重载加法运算符示例, 将两个栈合并到一起，

```
Stack Stack::operator + (const Stack &st)
{
    int* t_items = new int[maxCount + st.maxCount];
    memcpy(t_items, items, sizeof(int) * size);
    memcpy(t_items + size, st.items, sizeof(int) * st.GetSize());
    delete[] items;
    items = t_items;
    top = size + st.GetSize() - 1;
    return *this;
}
```

在重载运算符中，可以直接访问传入参数的私有成员变量，如上面的st.items（可能是因为这个运算符重载也属于st对象的成员函数）。

同时，如果+运算符重载的话，连续相加也是可以支持的

```c++
// 假设重载了+运算符,a b为该类的对象
a + b <==> a.operator+(b)

a + b + c <==> a.operator+(b.operator+(c))
```

运算符重载中，前置自增和后置自增重载方式不同

```c++
// 前置自增没有形参，返回一个左值
T& operator++()
{
    (*this) += 1;
    return *this;
}

// 后置自增有一个int形参，只用于区分，返回一个右值
T operator++(int)
{
    T temp = *this;
    ++(*this);
    return temp;
}
```

看起来后置自增会比前置自增多一个临时变量，然而编译器通常会进行优化，所以就性能来看，二者几乎是相同的。



运算符的重载是有限制的：

1. 重载后的运算符必须有一个操作数是用户定义的类型。这样是防止用户将int之类的基础类型运算符重载，比如将+重载为求两数之差，这样程序就没法运行了。
2. 不能违反原运算符的语法规则，不能改变运算符操作数的数量和优先级。
3. 不能创建新的运算符。
4. 以下运算符不能重载：
   - sizof运算符
   - 成员运算符 .
   - 作用域解析运算符 ::
   - ? :
   - typeid RTTI的运算符
   - 四种类型转换运算符
5. 以下运算符只能通过成员函数重载
   - =
   - () 函数调用运算符
   - []
   - ->

### 友元函数

友元分为三种，友元函数，友元类，友元成员函数。通过设置为友元来让某些方法和类拥有成员函数的访问权限。

运算符的重载是有限制的，比如想要重载一个可以与基础数据类型结合的运算符*，可以使用如下形式来调用：

```c++
a * 0.5;  <==> a.operator*(0.5);
// 但是另一种形式的调用无法支持
0.5 * a;   // 不能是 0.5.operator*(0.5);
```

所以我们要以非成员函数的形式来重载，

```c++
T operator*(double x, const T& a);
```

但是非成员函数是无法访问成员变量的，这时就要使用友元函数，创建友元函数的的方法是将原型放到类声明中并加上关键字friend,

```c++
friend T operator*(double x, const T& a);
```

然后定义

```c++
// 不需要friend关键字
T operator*(double x, const T& a)
{
    T res;
    res = a;
    res.x *= x; // x为成员函数
    return res;
}
```

友元函数的一个应用是重载<<运算符使标准输出可以使用该类型，因为<<运算符必须使ostream对象在前，所以以成员函数的形式来重载是不行的（除非把语句写成 T << cout），使用以下形式来重载，

```c++
void operator<<(ostream& os, const T& a)
{
    os << a.x << a.y << endl;
}
```

如果想要使用连续的标准输出，cout << a << b，可以返回ostream的引用

```c++
ostream& operator<<(ostream& os, const T& a)
{
    os << a.x << a.y << endl;
    return os;
}
```

### 类型转换

#### 隐式类型转换

在类的类型转换中，如果定义了**接受一个参数**的构造函数，如

```c++
Stack(int maxCountValue);
```

那么就可以使用一个int类型来为这个类赋值，也就是将一个int型数据转换成Stack类型，这种过程叫做隐式类型转换。

```c++
Stack st3 = 10;
```

这种类型转换可能会带来问题，如果想要关闭这个特性，可以在构造函数前使用关键字explicit，

```c++
explicit Stack(int maxCountValue);
```

这样就不能进行隐式类型转换了。但是依然可以使用显示类型转换。

```c++
Stack st3 = Stack(10);
Stack st3 = (Stack)10;
```

需要注意，在以下情况会使用隐式类型转换：

- 将Stack对象初始化为int值（上面的例子）。
- 将int值赋给Stack对象。
- 将int值传递给接受Stack类型参数的函数。
- 返回值为Stack类型的函数返回int值。
- 在上面情况下，使用可以转换为int的内置类型时（如 short, char）。

#### 转换函数

与隐式转换相似，转换函数可以实现Stack类型向int类型的转换，转换函数有以下几点要求：

- 必须是类方法

- 不能指定返回类型

- 不能有参数

  ```c++
  Stack::operator int(){
      return maxCount;
  }
  ```

  这样就可以使用这个转换函数，

  ```c++
  int var = Stack(10);
  ```

转换函数和隐式转换一样，可以执行内置的自动类型转换，比如：

```c++
long var = Stack(10);
```

这样会先转换为int，在转换为long类型。使用转换函数会带来很多隐蔽的问题，比如将一个Stack类型的数据提供给了需要int参数的函数。同样可以使用explicit关键字限制转换函数的使用，这样就必须使用显示转换了。

**在类型转换中，只接受一个参数的构造函数和自定义转换函数使得类之间的类型转换变得方便，但是一定要注意使用隐式转换的安全性**。



## 第12章 类和动态内存分配

#### 特殊的成员函数

当一个类声明时，编译器将自动提供以下成员函数：

- 默认构造函数，如果没有自定义。
- 默认析构函数，如果没有自定义。
- 复制构造函数，如果没有自定义。
- 复制运算符，如果没有自定义。
- 地址运算符，如果没有自定义。

其中复制构造函数的定义如下：

```c++
ClassName (const ClassName &)
```

复制构造函数会逐个复制非静态成员的值，称为成员复制或者浅复制。也就是说如果存在一个指针，复制构造函数只会单纯地拷贝指针的值，而不会复制指针所指向的内容。

复制构造函数会在使用一个对象来初始化同类对象是使用，比如：

```c++
// 假设a为T类对象
T var(t);
T var = a;
T var = T(a);
T* pvar = new T(a);
```

事实上，每当程序生成了对象副本时，复制构造函数就会被使用，比如按值传递对象，函数返回对象，或者多个对象相加（如果重载了+运算符的话）也可能使用。

复值运算符的原型如下：

```c++
T & operator=(const T &);
```

复值运算符在将已有对象赋值给新对象时会调用，比如

```c++
// a和b都是T类对象
a = b;
```

注意，初始化时并不一定会调用复制运算符，一定会调用复制构造函数（取决于编译器实现）。同样，赋值运算符默认使用浅复制。同时赋值运算符会返回一个对象的引用，这是为了提高效率，一般返回*this。

#### 在构造函数中使用new时的注意事项

1. 使用new运算符分配的内存需要在析构函数中delete，new对应delete，new[]对应delete[]。
2. 如果有多个构造函数，它们使用的new应该一致（都为new或new[]），因为只有一个析构函数并且要和其中的delete是对应的。
3. 定义一个深复制的复制构造函数来复制指针指向的值。
4. 重载一个深复制的赋值运算符( = )来复制指针指向的值，赋值运算符（=）还需要检查自我赋值情况和释放以前成员指针申请的内存。
5. 如果一个类的成员变量定义了复制构造函数和赋值运算符，那么定义这个类的复制构造函数和赋值运算符时（如果有必要的话），需要显式调用此成员变量的复制构造函数和赋值运算符。

#### 返回对象的说明

当一个函数需要返回类对象时，有几种方式可以选择：

1. 返回指向const对象的引用

   这种返回是效率最高的方式，因为返回引用不会导致临时变量的产生，但这种方式也有一些限制，首先，返回的对象不能是在函数中定义的局部变量，其次，如果返回了传入的参数，该参数应该被声明为const。如果一个类没有公有复制构造函数（比如ostream），那么必须返回对象的引用。

2. 返回非const的对象引用

   这种情况一般在重载赋值运算符和<<运算符时使用，=运算符使用这种方式可以在连续的赋值时提升效率，而<<运算符必须这么做，因为连续输出时需要返回ostream对象。

3. 返回对象

   如果返回的是一个函数内定义的局部变量，就必须使用这种方式，虽然这种方式会导致赋值构造函数被调用，但这是无法避免的。

4. 返回const对象

   这种情况很少使用，主要用来防止以下情况产生：

   ```c++
   (a + b).func() // 这种方式是合法的，因为a + b 会产生临时的对象，那么他就可以调用成员函数，可以返回const对象来防止此类使用方式，但如果func()为const的则依然可以这样使用。
   ```

#### 使用对象指针

对象指针的使用方法与常规指针相同，可以直接声明指针类型，也可以有已有对象初始化指针：

```c++
T* a;
a = &t;  // t为T类对象
```

也可以使用new来动态申请内存：

```c++
T* a = new T;
T* a = new T(value);
```

使用new运算符将会调用类的构造函数。

同样，也可以使用定位new运算符来动态申请内存，同样也不能使用delete来释放定位new运算符申请的内存。所以想要保证定位new运算符创建的对象可以正常销毁，需要显式地调用析构函数，这也是少数需要显式调用析构函数的情形。

```c++
a->~T();
```

#### 成员初始化列表的语法

可以使用以下格式来初始化成员函数，

```c++
T::T(int m, int n) :m_m(m), m_n(n)
{
    ...
}
```

这将使用m和n的值来初始化成员变量m_m和m_n，需要注意的是：

- 这种格式只能适用于构造函数。
- 必须用这种格式来初始化非静态的const变量。
- 必须用这种格式来初始化引用数据成员。

针对第二点，c++11可以使用在声明变量时就初始化的方式来替代。

#### 在类中定义结构体、类和枚举

在类中也可以定义结构体、类和枚举，这些定义的作用范围将被限制在类中，不会与外界产生冲突：

```c++
class T
{
    class Interval
    {
        ...
    };
    struct Interval
    {
        ...
    };
    enum Interval
    {
        ...
    };
    ...
};
```



## 第13章 类继承

### 使用类的继承

假设有这样一个基类，

```c++
class base
{
private:
    int m;
public:
    base();
    base(int m);
    ~base();
    void SetM(const int value);
    ...
};
```

可以通过以下方式继承一个子类，

```c++
class T : public base
{
private:
    int n
    ...
}
```

这种方式称为公有继承，基类的公有成员将成为子类的公有成员，基类的私有成员也成为子类的一部分，但是不能直接访问，只能通过基类的公有方法和保护方法访问。

继承之后，子类需要添加自己的构造函数，也可以根据需要添加自己的数据成员和成员函数。

子类不能直接访问基类的私有成员，只能通过基类的方法间接访问，比如，在子类的构造函数中是不能直接设置基类私有成员的变量值的。也就是说，子类的构造函数必须使用基类的构造函数，c++中使用成员初始化列表来完成这个工作。

```c++
T::T(int _m, int _n):base(_m)
{
    n = _n;
    ...
}
```

所以从概念上说，创建一个子类对象时必须先创建一个基类对象。

同样可以使用这种方式初始化子类的成员变量，

```c++
T::T(int _m, int _n):base(_m), n(_n)
{
    ...
}
```

如果不显示调用基类的构造函数，程序将使用基类默认的构造函数，

```c++
T::T(int _m, int _n): n(_n)  // 调用默认构造函数base()
{
    ...
}
```

所以，除非想要使用默认构造函数，否则应该显式调用基类的构造函数。

同样可以使用复制构造函数来实现基类的创建，

```c++
T::T(const base& _base, int _n): base(_base), n(_n)  // 调用复制构造函数，由于没有定义，将使用base默认的复制构造函数
{
    ...
}
```

子类对象生命流程为：

基类构造函数---->子类构造函数---->...（使用子类）---->子类析构函数---->基类析构函数



通过继承，子类可以使用基类的方法，除非该方法是私有的，这也是使用继承最重要的原因，可以大量提高代码的重用性。除此之外，继承还有一个重要特性，也成为里氏替换原则：在需要提供基类对象的地方，可以提供一个子类对象。

比如，基类的引用和指针可以指向一个子类对象：

```c++
// t是一个子类对象
base* p = &t;
base& ref = t;
base* p = new T();
// 或者直接将子类对象赋值给基类对象
base b = t;
```

同样，在接受一个基类指针，基类引用或基类对象为形参的函数中，可以提供一个子类地址或子类对象。

### 多态公有继承

继承之后，子类拥有基类的方法，同时也可以改写基类的方法，使子类和基类的对象表现出不一样的特征。这种方式称为多态，有两种机制来实现多态：

- 子类中重新定义基类的方法。
- 使用虚函数。

虚函数是使用多态最常用的方法，在基类中声明一个函数为虚函数后，子类将可以改写这个函数的定义，并且在使用指针或者引用的情况下会根据所指向对象的具体类型来决定调用哪个函数（基类的指针或引用可以指向子类对象）。

声明一个虚函数：

```c++
virtual void show() const; // 函数返回值前加上virtual关键字
```

当一个函数声明为虚函数之后，继承的子类中对应的方法也会自动声明为虚函数，使用关键字virtual显式指出也是一种好方法。

在子类中，为了实现基类中的虚函数，有时需要调用基类中的方法，这时需要使用域解析运算符，比如：

```c++
class base
{
private:
    int m;
public:
    base();
    base(int m);
    ~base();
    void SetM(const int value);
    virtual void Show();  // 基类中的虚函数
    ...
};

// 子类需要改写这个虚函数
void Show()
{
    // 需要显示基类的成员变量，但是子类无法直接访问，需要使用基类的Show方法
    Show(); // 错误,调用了子类的Show(),变成递归了
    base::Show(); // 正确使用::运算符调用基类的Show()
}
```

虚函数还有一个重要的使用方法是虚析构函数，如果我们使用了一个基类指针或引用指向了子类的对象，调用普通析构函数的时候只会调用基类的析构函数，这时如果子类的析构函数包含了某些重要处理，比如释放内存，就会出现问题。要解决这种问题，就需要将基类的析构函数定义为虚函数，因为虚函数可以在使用指针或引用时判断真正的对象类型，这样就保证了子类的析构函数得到调用。所以，基类的析构函数一般要声明为虚函数，即使它什么也不做。

### 静态联编和动态联编（虚函数的原理）

在程序调用函数时，需要找到该函数对应的代码块。将函数的调用解释为执行特定的代码块被称为联编（binding）。在编译过程中就能确定所调用的函数，在编译时进行联编称为静态联编，一般的函数就是静态联编的。而有时函数的调用编译时无法确定，需要到运行时才能确定真正调用的函数，在运行时进行联编称为动态联编，虚函数使用的就是动态联编。

虚函数的特性是使用指针或引用调用函数时可以确定真正指向的对象类型，而这个过程就需要动态联编。编译器发现一个类中有虚函数时，会给这个类加入一个隐藏的成员，这个成员是一个指针，指向了一个保存着函数地址的数组（也被称为虚函数表）。虚函数表中保存了该类中所有虚函数对应的地址，如果一个子类重新定义了基类的虚函数，表中将保存新函数的地址并覆盖原函数，如果没有重新定义虚函数，表中保存的就是原来虚函数的地址。

调用虚函数时，程序会根据虚函数表中的记录去寻找对应的函数地址，因为子类的虚函数在重新定义时使用新函数的地址覆盖了原来的地址，所以虚函数可以保证调用的真正的对象中的函数。

使用虚函数也有一些缺点：

1. 每个对象都需要增大，都需要存储一个指向虚函数表的指针。
2. 对每个类都需要创建额外的虚函数表。
3. 虚函数调用时，需要进行额外解析，调用的效率有所下降。

### 使用虚函数的注意事项

1. 构造函数不能是虚函数。子类对象创建时将调用子类的构造函数，而在其中会调用父类的构造函数，因此子类不继承父类的构造函数。
2. 析构函数应该是一个虚函数，除非这个类不用做基类，即使它在析构函数中什么也不做。
3. 友元函数不能是虚函数，因为它不是类的成员函数。
4. 重载虚函数（即修改虚函数的参数类型或个数）会隐藏基类的虚函数，但是修改了返回类型不会，只要保证新函数返回的类型是原函数返回类型的子类。所以如果想要重载虚函数，需要在子类中显式地重新定义该虚函数。

### 访问控制protected

除了private和public两种访问权限外，还有一种访问权限protected，protected权限的类成员对外部来说无法访问，相当于private，但对于继承它的子类来说是可以访问的，相当于public。一般使用protected的成员变量是比较少的，使用protected的成员函数较为常见，它可以设计一些只有派生类可以访问，外部不能访问的内部函数。

### 抽象基类

纯虚函数，使用以下形式来声明一个纯虚函数：

```c++
virtual void func(int a) const = 0;  // 后面加上 = 0
```

纯虚函数一般是没有定义的，包含纯虚函数的类称为抽象类，他无法被具体化为对象，并且要求继承它的子类必须提供该纯虚函数的实现。

当然，纯虚函数也可以有定义，甚至可以将一个非虚函数声明为虚的：

```c++
void func(int a) const = 0; // 一般情况下不会这么做，这样也会使该类称为一个抽象类。
```

使用抽象基类是更规范的方法，它指出编程时应该首先开发一个模型——指出编程所需的类以及他们之间的相互联系。一种思想甚至认为，设计类时应该只将那些不会被继承的类设计成具体的类。

抽象类可以看作一个必须实现的接口，要求继承它的子类必须提供其纯虚函数的实现，这样确保了派生的类至少都支持抽象类指定的功能。

### 继承和动态内存分配

如果基类使用了动态内存分配，重新定义了赋值构造函数和赋值运算符，那么对它的子类有什么影响，这需要分情况来讨论。

#### 子类没有使用动态分配内存

这种情况不需要显式定义析构函数，赋值构造函数和赋值运算符。因为子类的析构函数执行完毕后总是要调用基类的析构函数，所以析构函数不需要重定义。而对于赋值构造函数和赋值运算符来说，子类在进行赋值时，对于基类的数据将会调用基类自定义的赋值构造函数和赋值运算符，因此不需要特殊处理。

#### 子类使用了动态分配内存

如果是这种情况，需要显式定义析构函数，赋值构造函数和赋值运算符。

对于析构函数来说，基类的析构函数会在执行完之后自动调用，因此自定义的析构函数需要负责释放申请的内存。

对于赋值构造函数来说，除了执行深复制外，还通过列表初始化的方式来调用基类的复制构造函数，

```c++
T::T(const T& a): base(a)
{
    value = new X();
    memcpy(value, a.value, sizeof(X));
    ...
}
```

对于复制运算符来说，同样需要显式调用基类的赋值运算符，但是不能直接用=号，

```c++
T& T::operator=(const T& a)
{
    if (this == &a)
    {
        return;
    }
    base::operator=(a);   // 相当于 *this = base, 但是不能这么写， = 会被解析成子类的赋值运算符，形成递归。
    delete value;
    value = new X();
    memcpy(value, a.value, sizeof(X));
}
```

### 类设计回顾摘要

子类的构造函数会自动调用基类的构造函数，如果没有显式调用，就会调用基类默认的构造函数。

赋值构造函数会在将一个对象初始化为同类对象时使用，赋值运算符会在同类对象赋值时使用。

构造函数和析构函数无法被继承。

如果函数将参数声明为const，则无法再将此const参数转换给其他函数，除非那个函数的参数也为const。

赋值运算符无法被继承，因为其特征标因不同的类而异。

子类的友元函数可以通过强制类型转换来将子类的引用或指针转换为基类的引用或指针，这样就可以调用基类的成员函数。



## 第14章 C++中的代码重用

### 包含对象成员的类

在C++中，想要在一个类中使用其他类的方法时，有两种方式，一种是继承其他的类，一种是在此类中包含其他类的对象。使用哪种方法根据具体情况分析，如果是is-a关系，则使用继承，如果是has-a关系，则使用组合（在类中包含其他类的对象）。

在一个类中，成员初始化的顺序与成员被声明的顺序一致，而与构造函数**初始化列表**中的顺序无关，

```c++
class T
{
    int x;
    string s;
};

T(const string& _s, int _x): s(_s), x(_x)  // 即使在列表中初始化的是s在前，实际也是x先被初始化
{
    ...
}
```

大多数情况下不需要考虑这种初始化顺序，单数如果类成员之间存在依赖的话，就要考虑初始化顺序的影响。

### 私有继承

私有继承是另一种实现hsa-a关系的方式，使用私有继承时，基类的公有和保护方法变成了子类的私有方法，这意味着基类的方法不会成为子类对象公共接口的一部分，但是可以在子类的公共接口中使用他们，就像组合的方式一样：获得实现，但不获得接口。

使用私有继承时，子类的构造函数与公有继承相似，通过显式调用基类的构造函数初始化基类的部分，

```c++
class T : private base
{
    ...
};

T(int _x): base(_x) // 假设基类的初始化需要一个int变量
{
    ...
}
```

私有继承不同与公有继承，既然基类的接口变成了子类的私有方法，那么编写子类接口时一般会大量调用基类的方法，调用基类的方法使用基类名称加作用域解析运算符来实现，

```c++
int T::GetX() const
{
    return base::GetX(); // 基类有一个获得x成员值的方法，这种使用方式就像组合一样。
}
```



在组合的实现中，有一个基类的成员变量来保存基类的信息，我们可以获得这个基类的变量，那么在私有继承中如何实现这一点呢。因为使用的是继承，也就是子类对象在一些情况下可以转换成基类对象来使用，所以可以直接使用类型强制转换来实现这一点，

```c++
const base& GetBase() cosnt
{
    return (const base) *this;
}
```

同样，可以使用强制类型转换来调用基类的友元函数。

需要注意的是，**在私有继承中，未进行显式类型转换的派生类引用或指针，无法赋值给基类的引用或指针**。



通常情况下，应该用包含对象的方式来建立组合关系，而不是私有继承，因为组合关系中经常需要包含多个对象，这意味着如果使用继承的话不得不继承多个基类，虽然C++允许这样做，但是这可能会带来很多问题，而使用包含对象的方式更加简单，也更加易于理解。除非遇到以下情况：

1. 新的类需要访问基类的保护对象。
2. 新类需要重新定义基类的虚函数。

然而这种情况应该是很少而且可以避免的。

### 保护继承

使用保护继承时，基类的私有和保护成员将变成子类的保护成员，这意味着在子类中可以直接使用基类的公共接口，而在继承链之外则不能使用（所以子类还是需要自己提供公共接口），

```c++
class T : protected base
{
    ...
};
```

另一个直观的区别就是，使用私有继承时，第三代不能使用基类的公共接口，而使用保护继承时，第三代依然可以使用基类的公共接口。



### 三种继承方式的区别

| 特征           | public                 | protected              | private                |
| -------------- | ---------------------- | ---------------------- | ---------------------- |
| 公有成员       | 子类的公有成员         | 子类的保护成员         | 子类的私有成员         |
| 保护成员       | 子类的保护成员         | 子类的保护成员         | 子类的私有成员         |
| 私有成员       | 只能通过基类的接口访问 | 只能通过基类的接口访问 | 只能通过基类的接口访问 |
| 向上的隐式转换 | 是                     | 只能在派生类中         | 否                     |



### 使用using重新定义访问权限

使用私有继承或者保护继承时，基类的公共接口在无法在外部使用，所以子类需要重新提供外部接口，一般有两种方法：

1. 间接调用，在子类新的公共接口中调用基类的方法。

   ```c++
   int T::GetSize() const
   {
       return base::GetSize();
   }
   ```

2. 使用using声明指出子类可以使用特定的基类接口（其实也是包装了函数的间接调用）。

   ```c++
   class T: private base
   {
       ...
   public:
       using base::GetSize; //只需要提供函数名，不需要返回值，参数和括号
   };
   
   T t;
   t.GetSize();   // 将调用基类的方法
   ```



### 多重继承

多重继承（MI）是C++专有的语言特性，它允许一个类继承多个基类，但是这将带来一些问题：

1. 子类中应该保存多少个基类的对象。
2. 使用的方法出现冲突。

#### 虚基类

虚基类是为了解决第一个问题使用的。在继承中，子类会保存一个基类的对象，但是多继承在继承的类中有相同的基类时，会导致子类中有多个基类对象副本，

```c++
class A
{
    ...
};

class B: public A
{
    ...
}

class C: public A
{
    ...
}

class D: public A, public B   // 每个继承都要声明为 public，否则默认是私有继承
{
    ...
}
```

D类继承的B ,C两类都有相同的基类A，所以D类中会有两个A类副本，这样的话以下代码会出现二义性，

```c++
D d;
A* a = &d; // 无法确定使用哪个A类对象副本

// 使用下面的方式显式指定
A* a = (B*)&d;
A* a = (C*)&d;
```

除了显式指定的方式，也可以使用虚基类来防止子类中存在多个基类对象副本，使用方法为在继承的声明前加上 virtual,

```c++
class D: virtual public A, public virtual B   // public与virtual顺序无关
{
    ...
}
```

使用了虚基类时，类的构造函数也需要变化，

```c++
// 原来的D类构造函数
D::D(const A& a, int x): B(a), C(a), x(x)
{
    ...
}

// 需要显式地调用A的构造函数，因为B与C类的构造函数将不能再传递在A的构造函数
D::D(const A& a, int x): A(a), B(a), C(a), x(x)
{
    ...
}
```

这种方式只能用于虚基类，不能用于非虚基类。

#### 方法的冲突

假如B类和C类都定义了一个Show的方法，那么D类中继承得到的方法有两个，

```c++
D d;
d.Show(); // 存在二义性，无法确定使用的是B类还是C类的方法
```

解决这种冲突有两种方式，一种是调用时使用域解析运算符，

```c++
d.B::Show(); // 显式指出调用B类的Show()方法
```

更好的方法是自定义一个Show()方法，间接调用基类的方法，

```c++
void D::Show() const
{
    C::Show(); // 调用C类的Show()方法
    ...
}
```

虽然这种方式可能会带来一些问题，比如B类和C类的方法中都调用类A类地Show方法，如果在D类的Show方法中同时调用B和C类的方法，将导致A类的Show方法被调用了两次，不过这种可以修改代码的实现方式解决（将B类和C类的Show方法分解成两个子函数），与语言的特性无关。

#### 多重基层的其他注意事项

1. 当虚基类和非虚基类混用时，在子类中会保存一个虚基类的对象副本和多个非虚基类的对象副本。
2. 使用了虚基类后，方法的冲突不一定非要提供类名的限定来解决，具体来说，子类中的名称优先于直接或间接继承的基类中的名称。
3. 方法的二义性与访问权限无关，不论基类中的方法是private还是public，只要存在相同的名称就会存在二义性问题。
4. 多重继承会增加编程的复杂性，谨慎使用。

### 类模板

泛型编程是代码重用的重要部分，之前已经使用过模板函数，让一个函数可以处理多种不同的数据类型，同样，使用类模板可以让一个类处理多种数据类型，STL模板库就是类模板实现的。

类模板的定义如下，

```c++
template <typename Type>
class test
{
    ...
}
```

同样，成员函数可以替换为模板函数

```c++
template <typename Type>
int Sum<Type>(Type a, Type b)
{
    ...
}
```

在类模板的成员函数实现中，一般不将模板成员函数的实现与声明分离（放在不同的文件中），具体原因见 第8章 函数模板 章节。

使用类模板时，必须请求实例化，

```c++
test<int> t; // 创建一个int类型的模板类对象t
```

类模板必须显式提供类型，这与模板函数不同，因为模板函数可以根据参数的类型自动生成需要的函数。



虽然类模板可以示例化成各种类型的模板类，但是如果示例化的类型是指针的话，需要考虑其中的实现是否正确，比如是否为指针分配了内存空间，成员函数的实现是否考虑了指针类型。



#### 非类型的参数

模板可以使用非类型的参数，可以用这种方法来决定内存分配的大小（当然也可以使用动态内存分配的方式），比如一格Array类模板示例，

```c++
template <typename type, int n>  // 非类型的参数n
class Array
{
private:
    type ar[n];
public:
    Array();
    virtual T operator[](unsigned int n);
};

template <typename type, int n>  // 非类型的参数n
T Array<type, n>::operator[](unsigned int i)
{
    if (i < 0 || i < n)
    {
        return -1;
    }
    return ar[i];
}
```

上面代码中的n称为非类型参数或者表达式参数，假设使用下面的实例化方式：

```c++
Array<int, 12> arr;
```

这将会创建一个使用类型为int，最大储存数数12的Array类。

表达式参数有一些限制，只能是整型，枚举，引用或者指针，所以float或者double数据是不能作为表达式参数的。另外，模板的代码不能改变参数的值，也不能使用参数的地址，也就是说模板中不能出现&n和n++这种代码。最后，表达式参数必须为常量表达式。

使用表达式参数的优点式更加方便，不需要进行内存管理，在使用小型的类时执行速度更快。缺点是每个数组都会产生不同的类声明，而使用动态内存的方式相同的数据类型只会产生一种类声明，而且动态内存的方式更灵活，比如可以实现大小可变的数组。



#### 类模板的功能

类模板与普通类一样，可以作为继承的基类，可以作为其他类的成员变量，还可以作为其他模板的类型参数。

比如，可以使用元素为string的vector模板，

```c++
std::vector<std::string> var;
```

或者递归使用模板，

```c++
Array<Array<int, 5>, 10> arr; // arr是一个大小为10的数组，其中每个元素都是大小为5的int数组， 类似arr[10][5]
```

类模板也可以使用多个类型参数，比如

```c++
template <typename T1, typename T2>
class map
{
private:
    T1 first;
    T2 second;
public:
    ...
}

map<int, std::string> map1;
```

也可以为类模板提供一个类型默认值，

```c++
template <typename T1, typename T2 = int>
class map
{
private:
    T1 first;
    T2 second;
public:
    ...
}

map<std::string> map1;  //map <std::string, int>
```

但是函数模板是不能提供类型默认值的。不过类模板和函数模板都可以为非类型参数提供默认值。



#### 模板的具体化

之前的使用方法都是类模板的隐式实例化，比如可以使用对象指针，

```c++
Array<int, 5>* p;
p = new Array<int, 5>; //生成一个Array<int, 5>类对象
```

同样类模板也可以进行显式示例化，使用template class关键字，

```c++
template class Array<int, 10>; // 虽然没有创建对象，但是编译器将生成类声明
```



类模板的显式具体化功能和函数模板一样，可以为特定的数据类型提供定义，

```c++
template <> class Array<char*, 10>
{
    ...
}
```

这样就可以为char*类型提供单独的类定义。



#### 成员模板

类模板也可用作其他类模板的成员，

```c++
template <typename T>
class beta
{
private:
    template <typename V>
    class hold
    {
     public:
        V val;     
    };
    hold<T> q;   // T类型的hold对象
    hold<int> n; // int类型的hold对象
public:
    beta(T t, int _n): q(t), n(_n) {}
    template<typename U>
    U blab(U u) {return q.val / u;}
};

beta<double> be(3.2, 10) // beta<double>类的对象
be.blab(10); // 生成blab<int> 类型的模板 返回一个int值,值为3.2 / 10 = 0

```

也可以将嵌套模板在类外定义，但是一些老的编译器可能不支持这种做法，

```c++
template <typename T>
  template <typename V>
    class beta<T>::hold
    {
     public:
        V val;     
    };

template <typename T>
  template <typename U>
     U beta<T>::blab(U u) 
     {
        return q.val / u;
     }

// 不能使用 template <typename T， typename V>这种形式
```



#### 将模板用作参数

模板类也是一种类，可以用作模板的类型参数，

```c++
template <template <typename T> class Thing> // template <typename T> class Thing即为类型参数
class Crab
{
    Thing<int> s1;
    Thing<double> s2;
    ...
}

// Crab类的类型由Thing来决定，Thing是一个模板类
Crab<Stack> c1; // Stack<int> s1, Stack<double> s2
```



#### 模板类的友元

1. 非模板友元

   友元函数不是模板函数，这时候函数会成为所有实例化的类的友元。

   ```c++
   template <typename T>
   class HasFriend
   {
       public:
       friend void counts(); // counts是所有实例化类的友元
       ...
   }
   ```

   如果友元要用到模板类，那么必须要指明具体化，

   ```c++
   friend void counts(HasFriend<T>&) // 不能是HasFriend&
   {
       ...
   }
   ```

   但是要注意友元并不是模板函数，这意味这友元函数必须定义显式的具体化，

   ```c++
   void counts(HasFriend<int>& var) // 只有HasFriend<int>有此友元
   {
       ...
   }
   ```

2. 约束模板友元函数

   友元函数为模板函数时，如果友元的类型取决于类的类型，那么可以为每个具体的类生成相应的友元函数，

   ```c++
   template <typename T>
   class HasFriend
   {
       public:
       friend void counts<T>();
       friend void report<T>(HasFriend<T>& var); // report可以从参数推断类型，所以可以省略<T>
       ...
   }
   
   template <typename T>
   void counts()
   {
       ...
   }
   
   template <typename T>
   void report(HasFriend<T>& var)
   {
       ...
   }
   ```

   这样int类型的类生成int类型的友元函数，double类型的类生成double类型的友元函数。

3. 非约束模板友元函数

   如果友元函数是模板函数，但是模板的类型与类不相关，那么函数的具体化是类具体化的友元，即友元函数的类型是类的具体化类型，

   ```c++
   template <typename T>
   class HasFriend
   {
       public:
       friend void counts<V>();
       ...
   }
   
   template <typename V>
   void counts()
   {
       ...
   }
   
   HasFriend<int> h;
   h.counts();   //  匹配counts<HasFriend<int>>()
   ```

#### 模板别名

通过模板别名可以更方便地使用模板类，

```c++
// 使用typedef
typedef std::map<int, std::string> mapIS;
mapIS m1;

// 使用using
using mapIS = std::map<int, std::string>;
mapIS m1;

template <typename T>
  using arrd = std::array<T, 12>;
arrd<int> a1; // std::array<int, 12>

// using也可以何typedef一样用于非模板
typedef void (*pFunc)(int, int);
using pFunc =  void (*)(int, int);
```



## 第15章 友元、异常和其他

### 友元

除了友元函数外，也可以将一个类作为另一个类的友元，友元类可以访问原始类的私有和保护成员。除此之外也可以做更多限制，比如只将一个类的某个成员函数作为原始类的友元。

想要让一个类成为另一个类的友元，可以使用如下声明方法，

```c++
class TV
{
private:
    int state;
public:
    friend class Remote; // Remote类为TV类的友元，声明位置在private, public, protected都一样
    ...
};

class Remote
{
public:
    void SetState(TV& tv, int val) {tv.state = val;} // 可以直接访问tv的私有变量
};
```

如果想要让一个类的某个成员函数称为另一个类的友元，则使用如下方法，注意类的排列顺序，要先保证作为友元的类和函数在原始类之前声明，所以Remote声明应该在TV之前，但是Remote类中提到了TV类，所以使用前向声明（forward declaration）先声明TV类，但是并没有定义TV类，另外，在SetState中使用了tv.state，所以要保证SetState的真正定义位于class TV声明之后，

```c++
class TV;  // forward declaration
class Remote
{
public:
    void SetState(TV& tv, int val)； // 要在TV中的友元之前声明
};

class TV
{
private:
    int state;
public:
    friend void Remote::SetState(TV& tv, int val); // Remote类为TV类的友元，声明位置在private, public, protected都一样
    ...
};

// SetState的定义放到class TV之后
void Remote::SetState(TV& tv, int val)
{
    tv.state = val;
}
```



两个类可以相互称为友元，让他们可以都可以访问对方的私有和保护成员，需要注意的是，在一个类中使用了另一个类对象时，要先保证编译器已经知道该类的声明，

```c++
class Remote
{
public:
    friend class TV;
    void SetState(TV& tv, int val)； 
};

class TV
{
private:
    friend class Remote;
    int state;
public:
    ...
};

// SetState的定义与声明分离放到TV之后，而TV中的成员函数定义则不需要这么做，因为Remote声明与TV之前
void Remote::SetState(TV& tv, int val)
{
    tv.state = val;
}
```



一个函数可以成为多个类的友元，让这个函数可以直接访问这些类的私有和保护成员，具体用法与友元函数相同。这种用法主要用于多个类之间的状态同步。



### 嵌套类

可以在类中声明另外一种类（或者结构体）作为嵌套类，主要用于使用新的类作用域避免名称混乱。在包含类中可以创建嵌套类的对象，当且仅当嵌套类位于public部分时，外部才可以使用嵌套类。

```c++
class Queue
{
private:
    class Node    // Node是一个嵌套类，只能在Queue类中使用
    {
     public:
        int value;
        Node* next;
    };
    ...
};
```

嵌套类的访问权限取决于嵌套类声明的位置，就像其他常规的数据类型（结构体和枚举）一样，

- 位于包含类的私有部分，只有包含类可以访问它。

- 位于包含类的保护部分，只有包含类及其子类可以访问它。

- 位于包含类的公有部分，外部也可以使用这个类，需要使用名称解析运算符。

  ```c++
  // 如果Node类位于Queue类的public部分
  Queue::Node node;
  ```

通常，大家会使用公有枚举来提供外部使用的类常数。

嵌套自身类的访问权限与常规类相同，私有部分只能由自身成员函数访问，即使是包含类无法直接访问，保护部分只有自身及其子类可以访问，公有部分对外开放。



### 模板类中的嵌套

嵌套类也可使用与模板类之中，使用方式与常规类相同，

```c++
template<typename Item>
class Queue
{
private:
    class Node    // Node是一个嵌套类，只能在Queue类中使用
    {
     public:
        Item value; // 储存的数据类型与Queue的类型决定
        Node* next;
    };
    ...
};

Queue<double> queue; // 用于存储double类型的数据
```



### RTTI

RTTI（Runtime Type Identification）是运行阶段类型识别的简称。用途在于在程序的运行阶段根据不同的情况动态地识别当前对象或者创建一个新对象，比如一个基类有多个派生对象，在某种情况下必须使用其中一个派生对象来处理某种情况，而且使用方法并不是一个虚函数。RTTI只能用于包含虚函数的类（因为要用到虚函数表）。

解决这种问题主要有三种方式：

- dynamic_cast运算符

  dynamic_cast执行上行转换，即将一个子类的指针转换成一个基类的指针，否则返回空指针。虽然不能回答“指针指向的是哪种对象”的问题，但是可以回答“能否安全地将对象的地址赋值给特定类型的指针”这样的问题，dynamic_cast执行成功的条件为：该指针指向的对象是转换后的类型或者是从转换后的类型派生出的类型。

  ```c++
  class A
  {
      ...
  };
  
  class B :public A
  {
      ...
  };
  
  B* b = new B;
  A* a = dynamic_cast<A*>(b); // 正确，B类是A类派生的子类，转换是安全的。
  
  A* a = new A;
  B* b = dynamic_cast<B*>(a); //不安全的转换将失败，返回空指针。
  
  void Func(A* p)
  {
      if (ptr = dynamic_cast<B*>(p)) // 用来判断指针p指向的类型是否为B类型或者它的子类（如果有的话），如果是A类型的话会返回空指针。
      {
          ...
      }
      ...
  }
  
  ```

  dynamic_cast也可用于引用，但是没有空指针对应的引用值来指示失败，当转换不正确时，将抛出bad_cast异常（typeinfo头文件中定义）。

- typeid和type_info运算符

  typeid运算符返回type_info对象的引用（typeinfo头文件中定义），type_info重载了==和!=运算符，可以进行类型的比较，

  ```c++
  typeid(A) == typeid(*a);  // 判断a指针是否指向A类型对象
  ```

  type_info有一个name()成员返回一个类型相关的字符串，但是并不一定时类的名称，所以以下用法可能是错误的，

  ```c++
  typeid(a).name() == "A";  // 判断a指针是否指向A类型对象
  ```

  使用typeid可能会导致代码出现多个if else语句（比如需要判断一个指针的多个具体类型采取不同的行动），可以考虑多使用dynamic_cast。



### 类型转换运算符

除了C语言的强制类型转换为，C++提供了更为规范的四种类型转换运算符。

- dynamic_cast：用于进行上行转换，要求转换后对象是转换前对象的基类，否则返回空指针。可用此特性来判断当前指针指向的对象是不是某个类或者他的子类（如果有子类的话）。

- const_cast：用于改变一个值的const和volatile属性，可将一个值增加或去掉这两种属性。注意这个转换并不是万能的，比如：

  ```c++
  void Func(const int* p)
  {
      int* q;
      q = const_cast<int*>(p);
      
      *q += 1;
  }
  ```

  使用此方法可以修改p指向的值，但只有在传入的实参是可修改的才会生效（也就是忽略了形参中的const关键字），如果传入的实参本身就是指向const的指针，那么这个修改将无效。

- static_cast：类似与C中的强制类型转换，只有转换双方可以被隐式类型转换时才合法，可用于类中的上行或下行转换，或者基础类型之间的转换。当然这种转换不一定是安全的。

- reinterpret_cast：专门用来执行不相关的转换，比如将一个结构体转换为一个基础类型，或者将一个结构体转换为另一个不相关的结构体。这种转换是危险的，一般使用时需要保证两边的内存布局是一致的。reinterpret_cast也不是全能的，比如不能将一个指针转换成内存占用更小的数据类型，比如char；也不能函数指针和数据指针之间的转换。



## 第16章 string类和标准模板库

### string类

#### 构造字符串

string类常用的构造函数有：

- string(const char* s)：使用传统的C字符串。
- string(size_type n, char c)：创建一个n大小的string对象，其中每个元素都被初始化为字符c。
- string(const string& str)：赋值构造函数。
- string(): 默认构造函数。
- string(const char* s, size_type n)：将string初始化为s的前n个字符，即使超过了s长度。
- string(Iter begin, Iter end): string初始化为[begin, end)之间的字符。begin为指针或者STL中的迭代器。
- string(const string& str, size_type pos, size_type = npos): 将string初始化为str对象pos开始（不包括pos）到结尾的字符，或者从pos开始的n个字符。
- string(string && str)：移动构造函数。
- string(initializer_list\<char> il): 将string对象初始化为il中的字符。也就是说string可以使用列表初始化方法。

#### 使用字符串

string重载了比较运算符，让两个string对象或者string对象啊和char*对象之间可以进行比较，

```c++
std::string str = "aaaa";
const char* s = "bbbb";
if (str > s)
{
    .....
}
```

string也重载了find()方法，可以使用此方法在一个string对象中查找另一个字符串的位置。

find()方法有四种形式：

```c++
size_type find(const str& str, size_type pos = 0) const  // 从字符串pos位置开始查找字符串str
size_type find(const char* s, size_type pos = 0) const   // 从字符串pos位置开始查找字符串s
size_type find(const char* s, size_type pos, szie_type n) const  // 从字符串pos位置开始查找字串s前n个字符组成的字符串
size_type find(char ch, size_type pos = 0) const  // 从字符串pos位置开始查找字符ch
    
// 如果找到返回该字符串或字符首次出现的位置，如果找不到返回返回sting::npos
```

string也提供了find_first_of(), find_last_of()等方法用于查找某个字符集中出现过的字符。

string字符串有自动调整大小的功能，当一个字符被加入字符串末尾时，不能直接将字符串增大，因为相邻的内存可能已经被占用了。因此一般需要重新分配内存，但是分配内存的效率是很低的。所以一般的做法是，开始的时候就分配更大的内存，为字符串的增大提供了空间，如果字符串一直增大超过了内存的大小，程序将重新分配一个两倍的内存。

size()方法返回字符串的大小，不包括结尾的0字符串。

capacity()方法返回当前分配给字符串的内存大小。

reserve(size_type n)方法用于申请分配内存空间的大小（？）。

c_str()方法返回一格C风格的字符串。

string是一个基于char的模板类，

```c++
template <class charT, class traits = char_traits<charT>, class Allocator = allocator<charT> >
    basic_string{...};

// basic_string有4个具体化
typedef basic_string<char> string;
typedef basic_string<wchar_t> wstring;
typedef basic_string<char16_t> u16string;
typedef basic_string<char32_t> u32string;

// 预定义的char_traits<charT>模板具体化描述关于字符类型的特殊情况，比如如何对值进行比较
// Allocator是一个管理内存分配的类,使用new和delete
```

### 智能指针模板类

#### 智能指针的使用

智能指针主要用于内存的管理，因为常规指针使用时可能会因为忘记释放申请的内存而导致内存泄漏，使用智能指针可以避免这些错误，智能指针有3种 auto_ptr(C++11中被废弃)， unique_ptr和shared_ptr。

三种智能指针模板都定义了类似指针的对象，可以将new申请的内存地址直接赋给这种对象，当智能指针过期时，其析构函数将使用delete来释放内存，而不需要手动释放内存。

使用智能指针需要包含头文件memory，然后使用模板语法来实例化所需类型的指针，比如aoto_ptr的构造函数如下，

```c++
template <class X>
class auto_ptr 
{
public: 
    explicit auto_ptr(X* p = 0) throw();
    ...
};
```

可以看出初始化所需的参数为X类型的指针，那么可以使用下面的方式来使用auto_ptr,

```c++
auto_ptr<double> pd(new double);
auto_ptr<string> pd(new string);

double p = new double;
auto_ptr<double> pd(p);
```

注意，智能指针模板是位于std命名空间里的。



需要注意的是，智能指针都有一个explicit的构造函数，这意味在不能使用隐式类型转换来初始化智能指针，

```c++
shared_ptr<double> pd;
double* p = new double;

pd = p;  // 错误，不能执行隐式类型转换
pd = shared_ptr<double>(p); // 正确

shared_ptr<double> pd2 = p; // 错误
shared_ptr<double> pd2(p); // 正确
```

智能指针对象很多地方都与指针相似，所以很多常规指针的操作都可以用于智能指针，比如解引用（*），成员访问（->），也可以将他的值赋给常规指针。

但是由于使用智能指针是使用delete来释放内存，所以一定要注意内存是否位于堆上，也就是说智能指针只能用于new申请的内存，

```c++
string str = "ooooo";
unique_ptr<string> pd(&str); // 错误的用法，会将delete运算符作用于非堆
```



#### 为何不使用auto_ptr

考虑到两个智能指针同时引用同一个对象的场景，

```c++
auto_ptr<string> pd(new string("99900"));
auto_ptr<string> pd2 = pd;
```

如果两个指针同时指向了同一个对象，那么在智能指针自动释放时，会导致同一个对象同时释放两次，要避免这种问题，有三种解决方法

- 重新定义复制构造函数和赋值运算符，进行深复制，这样两个指针就指向了两个不同的对象，不会相互影响。
- 建立所有权（ownership）概念，对于特定的对象只有一个智能指针可以拥有它，当发生赋值操作时会转让所有权，这样只有一个智能指针可以使用delete删除对象。auto_ptr和unique_ptr就是这么使用的，但是unique_ptr会更加严格。
- 使用引用计数（reference counting）来跟踪特定对象，例如：赋值时引用计数加一，指针过期时引用计数减一。当引用计数为0时，才释放对象。shared_ptr就是这么做的。

auto_ptr使用方案二解决上面的问题，它确实转移了所以权，但是无法保证被剥夺所有权的指针对象（称为悬挂指针）不会在后续被使用，而unique_ptr在编译时不允许出现赋值操作，unique_ptr更加严格，但是也更加安全，

```c++
auto_ptr<string> pd(new string("99900"));
auto_ptr<string> pd2 = pd;  // 此时auto_ptr不会报错，但是unique_ptr会报错

*pd = "uuu";  // 如果使用unique_ptr，就不会执行这个引起错误的语句
```

所以c++11废弃了auto_ptr，只使用unique_ptr和shared_ptr。

当然，在某些情况下会需要使用赋值运算符，这时可以使用移动语义来实现，

```c++
unique_ptr<string> pd(new string("99900"));
unique_ptr<string> pd2 = std::move(pd); 
```

unique_ptr并不是禁止所有赋值操作，比如下面的赋值操作就是合法的，

```c++
unique_ptr<string> Func(const char* s)
{
    unique_ptr<string> p(new string(s));
    return p;
}

unique_ptr<string> pd = Func("aaaaaa");  // 合法的操作
```

这是因为Func函数返回时会产生临时对象，该对象接管了p的所有权，然后pd有接管了临时对象的所有权，临时对象在赋值完毕后被销毁，没有机会使用它来访问无效的数据。更通用的说法是：**程序试图将一个unique_ptr赋值给另一个unique_ptr时，如果源unique_ptr是一个临时的右值，则编译器允许这样做，如果源unique_ptr会存在一段时间，则编译器禁止这样做。**

unique_ptr还有另一个好处，它可以用于数组，因为delete与new配对，delete[]与new[]配对，auto_ptr与shared_ptr都不使用delete[]，所以他们不能用于数组，

```c++
unique_ptr<double[]> p(new double(5));
```

### 标准模板库

STL是C++标准的一部分，但是并不是面向对象的编程，而是泛型编程。这两种编程思想本质都是为了提高代码的可重用性。

#### 模板类vector

vector是一个矢量类，包含在头文件vector中，可以将其看作大小可变的数组，支持随机访问，

```c++
vector<int> vec(5);  // 包含5个int型数据的vector

int n = 10;
vector<double> vec(10); // 包含10个double型数据的vector

vector[6] = 0; // vector支持随机访问
```

使用vector可以方便地创建动态分配的数组。除此之外vector也支持很多功能，比如：

```c++
vector<int> vec(10);
vec.size();     // 返回元素数目
vec.swap(vec2); // 交换两个容器的内容
vec.begin();    // 返回指向第一个元素的迭代器
vec.end();      // 返回一个超过容器尾部的迭代器
vec.insert(vec.begin(), vec2.begin(), vec2.end());  // 在vec的第一个元素之后插入vec2的所有元素,[begin(), end())
vec.erase(vec.begin(), vec.begin() + 2);  // 删除第一个和第二个元素， [begin(), begin() + 2)
// 一般来说容器的区间都是左闭右开的，因为end()返回的是最后一个元素之后的迭代器，所以vec.begin()到vec.end()之间包括了容器内的所有元素
```

除了vector独有的函数外，也可以用stl的通用函数对vector进行操作，比如

```c++
void Func(int a)
{
    ...
}
for_each(vec.begin(), vec.end(), Func);  // 为vec的每个元素执行操作，不能修改元素的值

for(int& v: vec)
{
    ...    // 可以使用基于范围的for循环，然后指定引用参数，这样就可以修改元素的值
}

sort(vec.begin(), vec.end()); // 将vec中所有的元素升序排序，需要使用元素的<运算符

bool MoreThan(const int& a, const int& b)
{
    return a > b;
}
sort(vec.begin(), vec.end()，MoreThan);  // 将vec中所有的元素降序排序

```

### 泛型编程

泛型编程思想在于编写独立于数据类型的代码，在C++中，完成此项工作的工具就是模板，模板可以根据不同的类型实例化成不同的定义，可以让同一个算法适配多种的数据类型。

#### 迭代器

迭代器是泛型编程的重要组成部分，模板使得算法独立于储存的数据类型，而迭代器使算法独立于使用的容器类型，所有数据类型的容器都可以使用迭代器来对元素进行操作。

一个迭代器应该满足以下特征：

- 可以对迭代器进行解引用操作，以此来访问迭代器引用的值。即可以定义*it。
- 可以将一个迭代器赋值给另一个迭代器。如果p和q都是迭代器，可以定义p = q。
- 可以对迭代器进行比较。如果p和q都是迭代器，可以定义p == q和p != q。
- 可以使用迭代器来访问容器内的所有函数。即可以定义++it和it++。

指针可以满足迭代器的所有需求，但是并不是所有的迭代器实现都是指针，也有可能是一个对象。但是所有迭代器的操作都是相同的，比如begin()函数返回指向第一个元素的迭代器，end()函数返回指向最后一个元素之后的超尾迭代器，也许有的容器不需要超尾迭代器，但是将所有设计统一就可以使用通用的算法。比如使用了超尾迭代器后，所有容器的遍历都可以使用这种写法，

```c++
for (auto it = context.begin(); it != context.end(); it++)
{
    ...
}
```

当然，实际编程中有更好的方式来执行遍历操作，比如基于范围的for循环，

```c++
for (auto i : context)
{
    ...
}
```

#### 使用迭代器删除元素

使用迭代器删除元素时，需要格外小心，比如平时可能会使用如下写法，

```c++
vector<int >context;
...
for (auto it = context.begin(); it != context.end(); it++)
{
    if (...)
    {
        context.erase(it);
    }
}
```

然而这将导致错误，原因在于使用一个迭代器删除元素时，该迭代器就会失效，所以后面对他进行++操作会出现错误。

通常有两条规则：

1. 对于节点式容器(map, list, set)元素的删除，插入操作会导致指向该元素的迭代器失效，其他元素迭代器不受影响
2. 对于顺序式容器(vector，string，deque)元素的删除、插入操作会导致指向该元素以及后面的元素的迭代器失效

所以如果想要使用迭代器来删除元素，应该使用下面的写法，

```c++
vector<int >context;
...
auto it = context.begin();
while(it != context.end())
{
    if (...)
    {
        it = context.erase(it); // 对于顺序式容器，erase方法返回删除之后指向下一个元素的有效迭代器（节点式容器返回void）
    } else
    {
        it ++;
    }
}


set<int>context;
...
auto it = context.begin();
while(it != context.end())
{
    if (...)
    {
        context.erase(it++); // 对于节点式容器，删除节点后对它进行++操作以指向下一个有效迭代器
        /* 这里要使用后置++，因为前置++传入的就是该迭代器本身，还是没有解决问题，而后置++传入的时临时对象，真正的迭代器已经   ++了,也就是说，使用前置自增的话需要这种写法；
        auto temp = it;
        ++it;
        context.erase(temp);
        */
    } else
    {
        it ++;
    }
}
```

#### 迭代器类型

1. 输入迭代器：输入迭代器用来读取容器中的信息，对它解引用可以取出容器中的值，但是不一定能修改容器中的值。所以，使用输入迭代器的算法不会修改容器中的值，比如find方法。输入迭代器能够访问容器中所有的值，所以支持++运算符，但是并不能保证第二次遍历容器时结果与前一次相同，也不能保证被递增后以前的值仍然可以被解引用。并且，输入迭代器是一个单向迭代器，只能递增，不能递减。
2. 输出迭代器：与输入迭代器相似，可以修改容器中的值，但是并不能读取。用于单通行、只写算法。
3. 正向迭代器：拥有输入与输出迭代器的功能，同时可以保证按相同的顺序遍历容器的值，也可以对前面的迭代器解引用得到正确的值。可以修改容器中的数据，也可以将迭代器声明为const使得只能读取数据。
4. 双向迭代器：拥有正向迭代器的所有功能，同时支持--操作，可用于reverse()这种双通行的算法。
5. 随机访问迭代器：拥有上述迭代器的所有功能，同时可以直接访问容器内的任何数据，除了上述迭代器的功能外，还支持随机访问运算符[],还可以对迭代器进行+或-运算，如 it + n, it -= n，也支持对迭代器进行比较。

各种迭代器的类型并不是确定的，比如vector容器使用的是随机访问迭代器，vector可以直接访问某一个元素的值，而list使用的是双向迭代器，不能进行随机访问操作。

指针满足迭代器的所有要求，所以可以将指针用于迭代器，比如，在排序算法中，可以直接提供指针而非迭代器，

```c++
int arr[10];
...
sort(arr, arr + 10); // 虽然a+10看起来已经超过了数组的范围(数组最大为arr[9]),但是C++支持将超尾概念用于数组，使得STL的算法可用于常规数组。
```

### STL容器

STL具有不同种类的容器，分别有 deque, list, queue, priority_queue, stack, vector, map, multimap, set, multiset, bitset(不讨论), forward_list, unordered_list, unordered_map, unordered_multimap, unordered_set, unordered_multiset。

#### 容器的概念

容器是储存其他对象的对象，被储存的对象必须是同一种类型。储存在容器中的对象归容器所有，这意味着容器过期时，储存在其中的对象也会过期。

并不是所有的对象都能存储在容器中，该对象必须是可复制构造和可赋值的，基本的数据对象满足这些需求，如果一个类有public的复制构造函数和赋值运算符，则它也满足这种要求。最新的c++将这种要求描述为可复制插入和可移动插入的。

容器的基本的特征：

- T::iterator：指向T类型的迭代器，至少是正向迭代器。
- T::value_type：T的类型。
- T u：创建一个名为u的容器。
- T()：创建一个匿名的空容器。
- T u(a)：赋值构造，u == a。
- T u = a：同T u(a)。
- u = a：赋值，u == a。
- (&u)->~T()：对容器中的每个元素应用析构函数。
- u.begin(), u.end()：返回首元素的迭代器和超尾迭代器。
- u.size()：返回容器内元素个数。
- u.swap(a)：交换u和a的内容。
- a == b：a与b长度相同，并且其中每个元素都相同。
- a != b：!(a == b)。
- T u(rv)：移动构造函数，u == rv。
- T u = rv：同上。
- u = rv：移动赋值。
- a.cbegin()：指向第一个元素的const迭代器。
- a.cend()：超尾的const迭代器。

rv指右值，比如函数的返回值。

#### 序列

序列是容器的一种改进，队列共有7种，deque, list, forward_list, queue, priority_queue, vector。除了容器的特征外，序列保证了其中的元素按特定顺序排列，不会再两次遍历之间发生变化，其中的元素是线性排列的，即除了第一个和最后一个元素，每个元素之前和之后都有一个元素。

序列的基本特征：

- T a(n, t)：创建一个名为a，由n个t组成的序列。
- T (n, t)：创建一个由n个t组成的匿名序列。
- T a(i, j)：i, j为迭代器，创建一个名为a的序列，并将其初始化为[i, j)的内容，也和上面一样支持匿名序列。
- a.insert(p, t)：将t插入到p前面。
- a.insert(p, n, t)：将n个t插入到p前面。
- a.insert(p, i, j)：i, j为迭代器，将[i, j)的内容插入到p前面。
- a.erase(p)：删除p指向的内容。
- a.erase(p, q)：p, q为迭代器，[p, q)之间的元素。
- a.clear()：清除所有元素。

同时有的序列还有以下特征：

| 方法            | 返回值 | 含义                   | 支持的序列          |
| --------------- | ------ | ---------------------- | ------------------- |
| a.front()       | T&     | 返回序列的第一个元素   | vector, list, deque |
| a.back()        | T&     | 返回序列的最后一个元素 | vector, list, deque |
| a.push_front(t) | void   | 将t插入到序列最前面    | list, deque         |
| a.push_back(t)  | void   | 将t插入到序列最后面    | vector, list, deque |
| a.pop_front()   | void   | 删除第一个元素         | list, deque         |
| a.pop_back()    | void   | 删除最后一个元素       | vector, list, deque |
| a[i]            | T&     | 同数组                 | vector, deque       |
| a.at(i)         | T&     | 同T&                   | vector, deque       |

其中，a[i]和a.at(i)功能相同，但是a.at(i)会执行边界检查，越界将会引发异常。vector没有在容器最前端插入数据的功能，因为vector在最前端插入数据时需要把容器内的所有数据后移。

#### 每种容器的简要介绍

1. vector

   类似数组，可以动态改变长度，并随着元素的添加和删除而增大和缩小，并且他是可以被反转的。

2. deque

   双端队列，也就是一个前端和后端都可以插入和删除元素的队列，支持随机访问。为了实现在前端插入和删除元素，deque的实现比vector要复杂，尽管都支持随机访问，但是vector的性能更好一些。

3. list

   双向链表，支持双向遍历。list在任一位置插入和删除的时间都是固定的，强调快速的插入与删除，但是不能随机访问。除了容器和系列的特征，list还支持链表的操作：

   - void merge(list<T, Alloc>& x)：将两个链表合并，这两个链表必须时已排序的。
   - void remove(const T& val)：将所有值为val的元素删除。
   - void sort()：排序链表，使用<运算符，所以默认是升序排序。list不能使用非成员的sort函数(algorithm)，因为他需要使用随机访问迭代器，而list是不支持随机访问的。
   - vois splice(iterator pos, list<T, Alloc>& x)：将链表x插入到pos前，与insert不同的是，Insert将x的副本插入到目标地址，而splice将x移动到目标地址。
   - void unique()：将链表种的连续相同元素压缩为一个元素。

4. forward_list

   单向的链表，不可以反转，比list更简单，功能也更少。

5. queue

   队列，限制比双向队列deque多，不可以随机访问，甚至不允许遍历队列，把使用都限制在基本的队列操作上。

   - void empty() const：查询队列是否为空。
   - size_type size() const：返回队列元素的数目。
   - T& front()：返回队首元素的引用。
   - T& back()：返回队尾元素的引用。
   - void push(const T& x)：在队尾插入元素x。
   - void pop()：删除队首的元素。

6. priority_queue

   优先队列，支持队列的操作，不同的是它总是将最大的元素放到队列最前端，底层是使用vector来实现的，可以修改用于确定哪个元素放到队首的比较方式，也就是说可以构造将最小的元素放到队列最前端的priority_queue。

7. stack

   栈，先进后出的数据结构。

8. array

   普通的数组，好处是可以使用一些stl的算法，比如copy()，for_each()。

#### 关联容器及其简要介绍

关联容器将值与建关联在一起，提供了元素的快速访问，底层一般使用树来实现，关联容器可以插入元素，但是不能指定插入的位置。下面是常用的关联容器。

1. set

   集合，可反转，默认是升序排序的，并且其中的值是唯一的。set可以使用接受两个迭代器区间形式的构造函数。和数学中的集合一样，set支持并集、交集和差集这些算法。当然这些算法是stl通用的算法，并非set专用，其他容器也可以使用，但是set天然满足这些算法的先决条件，即容器内的元素的排序好的。

   并集使用set_union()，交集使用set_intersection()，差集使用set_difference()。他们都接受5个参数，前两个迭代器定义第一个集合的区间，后两个迭代器定义第二个集合的区间，最后一个是输出迭代器，指出将结果复制到什么位置。

   ```c++
   // 求a与b的并集并将结果输出到c
   set_union(a,begin(), a.end(), b.begin(), b.end(), insert_iterator<set<int>>(c, c,begin()));
   ```

2. multiset

   类似于set，但是可以存在多个相同的元素。

3. map

   映射，也是默认排序好的容器，将一个键与一个值关联起来。

   ```c++
   map<int, string> map; // 创建一个int--string类型的map
   ```

   STL使用pair将两种值储存到一个对象中，如果key是键的类型，data是值的类型，那么最终pair的类型为pair<const key, data>，

   ```c++
   pair<const int, string> item(1, "first");
   map.insert(item);
   
   cout <<  item.first << item.second << endl; //使用first和second来访问pair中的元素
   ```

4. multimap

   与map类似，但是一个键可以对应多个值。

5. 无序的关联容器

   unordered_set，unordered_multiset，unordered_map，unordered_multimap四种是无序的关联容器，与原来的关联容器相似，但是其中的元素是无序的，有序的关联容器底层使用树来实现，而无序的关联容器底层使用哈希表来实现。无序的关联容器主要应对需要高效的添加、删除元素和查找元素的方法，但是不关注其中元素顺序的场景。

### 函数对象

函数对象也叫函数符，是可以以函数方式与()结合使用的任意对象。包括函数名，函数指针和重载了()运算符的类对象，比如

```c++
class Linear
{
private:
    double val;
public:
    ...
    // 重载()运算符
    double operator()(double x)
    {
        return val / x;
    }
    ...
};

Linear lin(10.0);
double a = lin(5.0); // a = 2.0
```

在for_each函数中，它最后的参数是一个函数符，但是函数的参数类型已经指定了，怎么保证for_each可以适应不同数据类型的容器呢，for_each使用模板来解决这个问题。

```c++
template <class InputIterator, class Function>
Function for_each(InputIterator first, InputIterator last, Function f); 
```

第三个参数f将的类型将会赋给Function类型。

#### 函数符的常用概念

生成器：指不用参数就可以调用的函数符。

一元函数：使用一个参数就可以调用的函数符。

二元函数：使用两个参数就可以调用的函数符。

谓词(predicate)：返回值是bool值的一元函数。

二元谓词(binary predicate)：返回值是bool值的二元函数。

可以使用重载()运算符的类对象减少参数的个数，因为类对象可以使用构造函数来预定义一些参数的值。这种方式的函数符可以用来满足不同类型的函数符。

#### 预定义的函数符

STL中预定义了一些函数符，用来满足需要函数符作为参数的STL函数，比如相加，比较，求平均值等操作。

详细见书 p575 运算符相应的函数符介绍。

#### 自适应函数符和函数适配器

所有预定义的函数符都是自适应的，他们本身就携带了result_type、first_argument_type和second_argument_type成员，比如plus<int\>的返回类型被标识为plus<int\>:result_type。

这些自适应的函数符可以被函数适配对象使用，在一些不同类型的函数符之间做转换，比如，我们想要将vector中的每个成员都扩大2.5倍，但是multiplies是一个二元函数，所以需要使用函数适配器binder1st来将他转换成一元函数，

```c++
multiplies<double> f;
binder1st (f, 2.5) f2;
transform(vecDoub.begin(), vecDoub.end(), out, f2); // out为一个输出迭代器

// 使用bind1st更加简便，可以直接使用匿名对象
transform(vecDoub.begin(), vecDoub.end(), out, bind1st(multiplies<double>(), 2.5));
```

binder2nd与binder1st相似，不过它将常数赋给第二个参数，而不是第一个。

### 算法

STL包含了很多通用的算法，比如sort(), copy(), find(), set_union()等。他们的总体设计是相同的，使用迭代器来标识要处理的的数据区间和结果放置的位置，有的函数还会接受一个函数符，并使用它来处理数据。

算法函数的设计都使用模板来提供泛型，并且使用迭代器来访问其中的数据，所以算法可以使用于不同数据类型和不同的容器，由于指针是一种特殊的迭代器，所以一些算法也可以用于指针。

统一的容器设计使得不同类型的容器之间具有明显的关系。比如可以使用copy将数组复制到vector，或者将vector复制进list。也可以使用==来比较不同的容器，因为==也使用迭代器来遍历容器，所以如果deque和vector的内容是完全相同的，则它们是相等的。

STL将算法分成4组：

- 非修改式序列操作：对区间中的每个元素进行操作，不修改容器的内容。如find()和for_each()。
- 修改式序列操作：对区间中的每个元素进行操作，修改容器的内容。如transform()，random_shuffe和copy()。
- 排序和相关操作：排序sort()和其他操作，比如集合的运算。
- 通用数字运算：比如计算累加，累乘等，vector是最有可能使用这些操作的容器。

前3种算法组定义在algorithm头文件，第4种定义在numeric头文件。

#### 算法的通用特征

对算法进行分类的一个依据是按照结果存放的位置来分类，比如sort()是就地算法，因为它的结果被放在原来的位置上，而copy是复制算法，因为它的结果发送到了另一个位置。transform()可以使用这两种方式来完成任务。

有的算法有两个版本，复制的版本以_copy结尾，比如replace()用于将区间内的所有old_value替换为new_value, replace_copy()将结果发送到一个新的位置，

```c++
template<class ForwardIterator, class T>
void replace(ForwardIterator first, ForwardIterator last, const T& old_value, const T& new_value);

template<class InputIterator, class OutputIterator, class T>
OutputIterator replace_copy(InputIterator first, InputIterator last, OutputIterator result, const T& old_value, const T& new_value);
```

注意二者的返回值不同，这里有一个通用的约定，复制算法返回一个迭代器，指向复制的最后一个值的后一个位置。

还有的函数根据函数符对元素的应用结果来执行操作，这些版本一般以_if结尾，比如replace_if()将复合条件的旧值替换为新值，

```c++
template<class ForwardIterator, class Predicate, class T>
void replace_if(ForwardIterator first, ForwardIterator last, Predicate pred, const T& new_value);
```

Predicate是一个谓词，即返回bool的一元函数。

#### 函数和容器方法

有时可以选择STL算法或者容器自带的STL函数。比如remove(),

```c++
list<int> list;
...
list.remove(4); // 删除所有值为4的元素
remove(list.begin(), list.end(), 4); // 同样删除所有值为4的元素
```

一般来说使用容器自带的STL函数更好，因为它可以使用模板类的内存管理工具，自动调整容器的长度，而remove算法并不会调整list的长度，它将计算的结果放到list的前面，并返回新list的超尾迭代器，删除后的无效元素将会放到list最后面。

尽管容器自带的STL函数更适合，但是STL算法算法更加通用，它不仅可以用于list，还可以用于其他的容器类型甚至数组，也可以混用容器，比如将vector的结果输出带list中。

#### 其他库和容器

在STL之外，C++也提供一些其他的库和容器，比如valarray，虽然它和数组相似，但它并不是STL标准的库，而是为数值计算提供一些更加简便的方法，比如，如果想要将两个数组的和放入第三个数组，

```c++
transform(vec1.begin(), vec1.end(), vec2.begin(), vec3.begin(), plua<int>); // vector的写法
valArr3 = valArr1 + valArr2; // valarray的写法
```

因为valarray重载了+运算符，同理，他也重载了*运算符，可以使用下面的方式扩大valarray中每个元素的值，

```c++
valArr = 2.5 * valArr; // 将valArr中的每个元素扩大2.5倍
```

除此之外valarry还支持很多运算，比如log，sum, max, min等运算，

```c++
valArr = log(valArr);
valArr = valArr.apply(log); // apply返回一个新对象，不修改原对象
```

valarry在数值运算上更加方便，但是它不是STL的库，所以并没有begin()这种方法来获得迭代器，自然也没有办法来使用需要迭代器的STL算法，不过C++11新增了模板函数begin()和end()，让valarray满足使用迭代器的需求，

```c++
sort(begin(valArr), end(valArr)); // 使用这种方式来让valarray支持STL算法
```

valarray还有一些其他特性，比如下面创建一个bool数组，其中vbool设置为numbers[i] > 9的值,

```c++
valarray<bool> vbool = numbers > 9; // numbers为一个数组
```

slice类可用于数组的索引，使用3个参数来初始化，3个参数的含义分别为：起始索引、索引数和跨距，比如slice(1, 4, 3)创建的对象选择4个索引，分别为1, 4, 7, 10，

```c++
valArr[slice(1, 4, 3)] = 10; // 将1, 4, 7, 10这几个元素的值设为10
```

这种slice的方式可以让你用一维数组表示多维数组，比如表示一个4 x 3的数组，可以建立一个12元素的一维数组，然后用slice(0, 3, 1)来表示第一行，slice(3, 3, 1)来表示第二行，用slice(0, 4, 3)来表示第一列的数据。

#### Initializer_list

Initializer_list为C++11新增（在initializer_list头文件中），目的在于使用列表初始化的方式来使用容器，比如

```c++
vector<int> vec {0, 8, 9. 13};
```

可以这样使用是因为容器的构造函数中有接受Initializer_list为参数的构造函数，Initializer_list中也可以做默认的类型转换，比如double转换为int。

在普通的类中，我们也可以使用{}语法来调用构造函数，比如

```c++
class Position
{
private:
    double x;
    double y;
public:
    Position(double _x, double _y): x(_x), y(_y)
    {    
    }
    ...
};

Position p(5.4, 6.3); // 将调用构造函数
```

当使用这种初始化方法是，接受Initializer_list为参数的构造函数优先于普通的构造函数被调用。

同时，Initializer_list也可以在代码中直接使用或者作为函数的参数，它有成员函数begin()和end()，

```c++
initializer_list<int> l {1, 5, 6};

int sum(initializer_list l)
{
    ...
}

sum(l);
sum({1, 2, 3})
```

注意，Initializer_list的迭代器为const，因为不能修改Initializer_list中的值。

## 第18章 C++11新标准

### 新增改进

#### 新类型

C++11新增的类型有：

- long long和unsigned long long：64位（或更宽）的整形。
- char_16t和char_32t：16位和32位的字符。

#### 统一的初始化

所有内置类型和自定义类型都可以使用{}来初始化，使用初始化列表时，可添加等号也可不添加，

```c++
int a = {5};
double b {9.8};
short c[5] {1, 2, 3, 4, 5};
Position p {1.0, 3.4};
```

如果类有将std::initializer_list作为参数的构造函数，那么只有该构造函数可以使用列表初始化。

使用列表初始化的方式不允许缩窄，即禁止将数据赋给无法存储它的变量，除非该值在变量的取值范围内，

```c++
char c = 87845643; // 不报错，但是是未定义行为
char c = {8.9}; // 报错
char c {88}; // 正确，88在char的表示范围
double d = {99}; // 正确，转换位更宽的类型
```

std::initializer_list也是C++11新增，前面已介绍。

#### 声明

1. auto

   以前的auto关键字用于声明该变量是自动储存的，C++中将其用于自动类型推断，这要求进行显式初始化，让编译器可以识别出类型的值，

   ```c++
   auto a = 112; // int
   auto p = &a; // int*
   ```

   auto最常用于简化模板声明，

   ```c++
   vector<int>::iterator it = vec.begin();
   auto it = vec.begin();
   ```

2. decltype

   decltype将变量的类型声明位指定表达式的类型，使用方法为

   ```c++
   decltype(x) y; // y的类型于x相同
   ```

   示例，

   ```c++
   int a;
   double b;
   decltype(a * b) c; // c的类型为double
   decltype(&b) p; // p的类型为double*
   
   // 更复杂的应用,更多信息在第8章
   int j = 3;
   int& k = j;
   const int& n = j;
   decltype(n) i1; // const int &
   decltype((j)) i2; // int &
   decltype(k + 1) i3; // int
   ```

   decltype最常用于模板之中，

   ```c++
   template<typename T, typename U>
   void Fun(T t, U u)
   {
       ...
       decltype(t + u) var = t + u; // 因为无法确定t + u的类型
   }
   ```

3. 返回类型后置

   新的函数声明方法，

   ```c++
   double Func(double, int);
   auto Func(double, int) -> double;
   ```

   虽然看起来更加复杂了，但是可以用于decltype来指定函数返回类型的场景，

   ```c++
   template<typename T, typename U>
   auto Fun(T t, U u) -> decltype(t + u) // 声明函数时t和u还不在作用域内
   {
       ...
       return t + u;
   }
   ```

4. 模板别名：using = 

   创建别名有几种方式，比如传统的typedef:

   ```c++
   typedef vector<map<int, string>> mapVec;
   ```

   现在也可以使用using来创建别名：

   ```c++
   using mapVec = vector<map<int, string>>;
   ```

   using还可用于模板的部分具体化：

   ```c++
   template<typename T>
     using arr12 = std::array<T, 12>;
   
   arr12<int> a1; // a1的类型为std::array<int, 12>
   ```

5. nullptr

   以前在C++中使用0类表示空指针，但是0也可以表示一个整数，为了解决这种冲突，C++11引入了nullptr来表示空指针，他是指针类型，不可以转换成整型。为了兼容 nullptr == 0 为true，但是使用nullptr 更加安全。

#### 智能指针

为了简化new与delete的使用，C++在C++11之前的版本引入了auto_ptr，在C++11之后，C++摒弃了auto+ptr，改为使用新的unique_ptr，shared_ptr和weak_ptr。使用智能指针可以让new分配的内存在不使用时自动delete，所有新增的智能指针都能与STL容器和移动语义协同工作。

#### 作用域内的枚举

枚举的作用域限定为枚举定义的作用域，这意味着如果在同一个作用域内定义了两个枚举，则他们的成员不能同名，为了解决这个问题，C++11新增了使用class和struct的形式定义枚举，

```c++
enum Old {One, Two, Three};
enum class New1 {Four, Five, Six};
enum struct New2 {Four, Fivem Six};

New1::Four;
New2::Four; // 使用作用域解析运算符来区分，这样就可以使用同名的成员了
```

#### 类的修改

1. 显式转换运算符

   在使用类时，对象的隐式转换可能会带来一些隐蔽的问题，所以加入了explicit来禁止单参数构造函数的隐式转换，

   ```c++
   class A
   {
       A(int);
       explicit A(double);
       ...
   }
   
   A a = 7; // 允许，将调用A(int)
   A a1 = 7.8; // 错误
   A a2 = A(9.0); // 正确，显式转换
   ```

   explicit也可用于转换函数，

   ```c++
   class A
   {
       operator int() const;
       explicit operator double() const;
       ...
   }
   
   A a2(9.0); 
   int n = a2; // 允许
   double b = a2; // 错误
   double b2 = double(a2); // 允许
   ```

2. 类成员的初始化

   C++11允许类成员的初始化（以前居然不允许？）,可以使用下面的方式初始化类成员：

   ```c++
   class A
   {
   private:
       int a = 10;
       double b {2.5};
   public:
       ...
   }
   ```

   可以使用=和{}的初始化，但是不能使用()的初始化，上面的初始化类似于：

   ```c++
   A(): a(10), b(2.5) {}
   ```

   如果类的对象在初始化时已经提供了初始化的值，那么这些值会覆盖默认值。

#### 模板和STL的修改

1. 基于范围的for循环

   ```c++
   vector<int> vec {1, 2, 3, 4, 5};
   for(auto x: vec)
   {
       cout << x << endl;
   }
   
   for(auto &x: vec)
   {
       x += 2;
   }
   ```

2. 新增的容器

   新增的容器有：forward_list、unordered_map、unordered_set、unordered_multimap、unordered_multiset以及array。

3. 新增的方法

   新增了cbegin()和cend()方法，返回const的迭代器，将它引用的元素视为const。同时STL还支持了移动构造函数和移动复制运算符。

4. 摒弃export

   export是为了模板的分离编译而引入的，现已废弃。

### 移动语义和右值引用

#### 右值引用

普通的引用被称为左值引用，它可以使标识符关联到一个左值，左值是一个表示数据的表达式，程序可以获得其地址。一般来说，赋值时左值可出现在赋值语句的左边，

```c++
int n;
int *pt = new int;
const int c = 101;

int &ref1 = n;
int &ref2 = *pt;
const int& refc = c;  // 虽然c不能出现在赋值的左边，但是仍然可以使用它的引用
```

C++11新增了右值引用，使用&&表示。右值引用可以关联到右值，即程序无法获得其地址的值，包括字面常量（字符串常量不算），像x + y之类的表达式以及函数的返回值（不是返回引用）。比如：

```c++
int &&ref = 13;
int &&ref2 = x + y;
double &&ref3 = sqrt(4.0);
```

使用右值引用关联一个值之后会使得该值被存储到一个特定的位置，并且也可以获得该位置的地址，也就是说，取址运算符&可作用于ref这类的右值引用。

#### 移动语义

看一次一个STL模板的赋值过程，

```c++
vector<string> vec;
...
vector<string> vec2(vec);
```

vector和string都使用动态内存分配，为了初始化vec2，vector的复制构造函数将使用new给vec2分配内存，其中的每个元素又使用string的复制构造函数来为vec2中的每个元素分配内存，然后再将其中的数据拷入vec2，这里的工作量是很大的。假如在一个使用函数返回值的场景，

```{
vector<string> Func()
{
...
}

vector<string> vec(Func());
```

这样的话Func()在返回时会产生临时对象，先将结果复制到临时对象中，再将临时对象中的数据拷贝到vec，然后销毁临时对象，这样的话就做了大量的无用功。如果可以保留原来的临时对象，让vec与之关联，这样就可以省下大量工作，而移动语义就是做这样的工作的。这类似于文件在计算机中移动，实际文件还留在原来的地方，只是修改了记录。

要实现移动语义，需要让编译器知道什么时候需要复制，什么时候不需要，这里就要使用右值引用了。比如，定义一个构造函数，它接受一个右值引用作为其参数，这样当传入的参数是右值时（比如函数的返回值），就会使用移动语义，避免复制带来的消耗，由于移动构造函数可能会改变实参，所以移动构造函数的参数不应该是const的。

移动语义示例：

```c++
class A
{
private:
    int n;
    char *p;
public:
    A();
    A(int k);
    A(const A& la);
    A(A&& ra);
    virtual ~A();
}

A::A()
{
    n = 0;
    p = nullptr;
}

A::(int K): n(k) {
    p = new char[n];
}

A::A(const A& la) n(la.n)
{
    p = new char[n];
    for(int i = 0; i < n; i++)
    {
        p[i] = la.p[i];
    }
}

A::A(A&& ra): n(ra.n)
{
    p = ra.p;   //移动构造函数中直接将实参的值赋给对象
    ra.p = nullptr; // p和ra.p都指向同一块对象，将ra.p置空，防止重复delete指针
    ra.n = 0;
}  // 因为ra是一个右值，也就是说后面的程序不可能再使用ra,所以我们可以随意对其进行操作，注意，即使ra的实参时 x + y,我们也只用了x + y这个结果，后续就算x或y改变了也不会有影响。

A::~A()
{
    delete [] p;
}

A a(8);
// 注意这是初始化，不是调用赋值运算符
A b = a; // 使用复制构造函数
A c = Func(); // 假设Func返回一个A类的对象，使用移动构造函数
```



要使用移动语义，需要满足两个条件，一个该类有一个自定义的移动构造函数，二是初始化该对象时使用了一个右值。在没有移动构造函数的时候，const引用作为参数在接受右值时会生成临时变量（就像普通的参数那样），函数调用结束后该临时变量就会销毁，没有充分利用临时变量，这正是移动语义要解决的问题。

同样，也可以定义移动赋值运算符（继续使用上面的A类），

```c++
A& A::operator=(const A& la)
{
    if(&la == this)
    {
        return *this;
    }
    delete p[];
    n = la.n;
    p = new char[n];
    memcpy(p, la.p, n * sizeof(char));
    return *this;
}

A& A::operator=(A&& ra)
{
    if(&ra == this)
    {
        return *this;
    }
    delete p[];
    n = ra.n;
    p = ra.p;
    ra.n = 0;
    ra.p = nullptr;
    return *this;
}
```



如果想要在使用左值的情况下强制使用移动语义，可以使用static_cast来将这个左值强制转换成右值，C++提供了更简便的方法，即使用std::move来将左值转换成右值，

```c++
A a(9);
A a1 = std::move(a); // 将使用移动构造函数（当然A类必须要支持），而不是复制构造函数
```

注意，强制使用移动语义会导致原来的变量(a)被清空，需要注意使用的场景。



#### 新的类功能

1. 特殊的成员函数

   C++11新增了两个默认函数，移动构造函数和移动赋值运算符，假如没有提供移动构造函数，而代码又需要使用，那么将会使用编译器提供的移动构造函数，该函数的原型如下，

   ```c++
   A::A(A&& a);
   ```

   同样，移动赋值运算符也是如此,

   ```c++
   A& A::operator=(A&& a);
   ```

   需要注意的是，如果已经提供了析构函数（？）、复制构造函数或者赋值运算符，那么编译器也不会提供默认的移动构造函数和移动赋值运算符，同理，如果已经提供了移动构造函数和移动赋值运算符，那么编译器也不会再提供默认的复制构造函数或者赋值运算符。

   默认的移动构造函数和移动赋值运算符工作原理和复制版本相似，执行逐成员的初始化并复制内置类型，如果有的成员是对象，那么将会执行该对象的构造函数和赋值运算符。

2. 默认的方法和禁用的方法

   C++11可以控制默认函数的创建，假如想要使用一个默认的函数，而这个函数因为某些原因不会自动创建，比如你已经提供了移动构造函数，这样复制构造函数就不会创建了。在这种情况下，可以使用关键字default显式声明这个函数的默认版本，

   ```c++
   class A
   {
   public:
       A(A&& a);
       A(const A& a) = default;
       A& operator=(const A& a) = default; // 为什么这个也不会默认创建
   }
   ```

   另外，也可以使用delete禁止编译器使用特定的方法，比如，要禁止对象的复制，可以禁用复制构造函数和赋值运算符，

   ```c++
   class A
   {
   public:
       A(A&& a);
       A(const A& a) = delete;
       A& operator=(const A& a) = delete;
   }
   ```

   虽然将复制构造函数和赋值运算符放到private中也能达到这个效果，但是使用delete更不容易犯错，也更容易理解。

   default只能用于6个默认的函数，delete可用于任何成员函数，delete还有一种用法是用于禁止特定的转换，

   ```c++
   class A
   {
   public:
       void Func(double var);
       void Func(int) = delete;
   }
   
   A a;
   ...
   a.Func(8); // 错误，但是如果没有将int参数的函数声明为delete，8将会被提升为8.0,进而匹配到接受double参数的函数
   ```

3. 委托构造函数

   如果给一个类提供了多个构造函数，可能需要编写大量重复的代码，因为很多成员的初始化是一样的，这时就可以使用委托构造函数在一个构造函数内调用另一个构造函数，

   ```c++
   class A
   {
   public:
       A(int k, double b): m_k(k), m_b(b) {...};
       A(): A(0, 0.0){...};  // 使用类似成员初始化列表语法的方式来调用上面的构造函数
       A(int k): A(k, 0.0) {...};
   }
   ```

   这种初始化方式称为委托。

4. 继承构造函数

   构造函数通常是不能被继承的，C++11提供了能够继承基类构造函数的机制，之前有一种让命名空间中的函数可用的方法，

   ```c++
   class C1
   {
   public:
       int fn(int var);
       double fn(double var);
       ...
   };
   
   class C2
   {
   public:
       using C1::fn; // 使用fn的所有重载版本都可用
       ...
   };
   
   C2 c2;
   ...
   c2.fn(9); // 使用C1::fn(int)
   ```

   C++11将这种方式使用在构造函数上，让子类也可以继承父类的构造函数（默认构造函数、复制构造函数和移动构造函数除外），

   ```c++
   class A
   {
   int m_k;
   double m_b;
   public:
       A(int k, double b): m_k(k), m_b(b) {...};
       A(int k): A(k, 0.0) {...};
       ...
   };
   
   class B : public A
   {
   short m_j;
   public:
       using A::A;
       B(int k): A(k), m_j(0) {};  // 使用A中的构造函数
       B(double b): A(0, b), m_j(0) {};
       ...
   };
   
   B b(8, 5.0); // B中没有接受（int, double）的构造函数，将会使用继承自A的构造函数。
   ```

   使用这种方式需要注意的是，继承而来的构造函数只会初始化继承而来的基类成员变量，如果还要初始化子类的成员，需要再用列表初始化的方式来实现（B类中的其他构造函数）。

5. 管理虚方法：override和final

   在类的继承体系中，如果父类定义类一个虚方法，子类想要覆盖这个虚方法，但是如果不小心将该函数的特征标写错了，那么将会隐藏而不是覆盖旧版本，

   ```c++
   class A
   {
   public:
       virtual void Func(char);
       ...
   };
   
   class B : public A
   {
   public:
       virtual void Func(char*);
       ...
   };
   
   B b;
   ...
   b.Func('a'); // 失败，因为B中隐藏了Func(char)这个函数
   ```

   为了避免这个问题，可以使用override标识符来指出你要覆盖一个虚函数，如果发现这个方法与基类的方法不匹配，编译将会报错，

   ```c++
   class B : public A
   {
   public:
       virtual void Func(char*) override; // 此时编译将会报错，因为这个Func没有覆盖任何一个基类的虚函数
       ...
   };
   ```

   而final标识符则可以指定某个函数无法被覆盖或某个类无法被继承,

   ```c++
   class A
   {
   public:
       virtual void Func(char) final;  // 子类将无法覆盖Func函数
       ...
   };
   
   class B final  // B类将不能被继承
   {
   public:
       ...
   };
   ```

#### Lambda函数

Lambda函数是一种函数表达式，可以通过更简单的方式定义一个函数符，它是一个匿名函数，

```c++
[&count](int x){count += x};
```

跟普通的函数相似，但是使用[&count]代替了函数名，并且没有返回值类型，这也是它被称为匿名函数的原因。

1. 比较函数指针、函数符合Lambda函数

   假设我们要使用算法来计算一个vector中多少个整数可被13整除，这里使用count_if算法，

   ```c++
   vector<int> numbers(100);
   ...
       
   // -------- 使用函数指针-----------
   bool fmod_13(int x) {return x % 13 == 0;}
   int res = count_if(numbers.begin(), numbers.end(), fmod_13);
   
   // -------- 使用函数符（重载了()运算符的类）-----------
   class F_Mod
   {
   private:
       int dv;
   public:
       F_Mod(int x): dv(x) { }
       bool opertaor()(int x)
       {
           return x % dv == 0;
       }
   }
   
   F_Mod f_mod_13(13);
   int res = count_if(numbers.begin(), numbers.end(), f_mod_13); // 将该对象传入后，count_if将调用f_mod_13的()运算符。使用这种方式的好处是扩展性强，比如后续需要计算能被15整除的数，再定义一个F_Mod对象并初始化为15就行了。
   
   // -------- 使用Lambda函数-----------
   int res = count_if(numbers.begin(), numbers.end(), [](int x)
       {
           return x % 13 == 0;
       });
   // Lambda函数没有名称，返回类型根据返回值推断得到，仅当 Lambda表达式由一条返回语句组成时自动推断才有用，所以一般的写法是使用返回类型的后置语法
   /*
   [](int x) -> double
   {
   double y = 10.0;
   return y - x;
   }
   */
   ```

2. 为何使用Lambda函数

   使用Lambda函数有一些优点，首先，他提供了在函数内定义函数的方法，这样可以使代码更紧凑，如果需要修改代码，涉及的内容就在附近。其次Lambda函数也是可以复用的，可以给一个Lambda函数命名，比如，

   ```c++
   auto mod13 = [](int x)
   {
       return x % 13 == 0;
   }
   
   bool res = mod13(889); // 像函数一样使用Lambda
   ```

   mod13的实现因不同的编译器而异。最后，考虑性能的问题，编译器一般不会内联普通的函数指针，因为可以获取函数的地址本身就意味着该函数是非内联的，而函数符和Lambda函数是允许内联的。

3. Lambda函数的额外功能

   Lambda可以访问作用域内的任何动态变量，可以将要捕获的变量放入其[]内，比如[a]将按值访问变量，[&a]按引用访问变量，[=]按值访问所有变量，[&]按引用访问所有变量。也可以混合使用这些方式，比如[a, &b]将按值访问a，按引用访问b，[=, &a]将按引用访问a，按值访问其他所有动态变量。

一般来说，Lambda经常是一些测试或比较表达式，简洁并且易于理解。

#### 包装器

C++提供了多个包装器，用于给其他编程接口提供更一致或者更合适的接口，包装器Function让我们能以统一的方式处理多种类似于函数的形式。

在使用函数符的时候，存在一个问题，函数可以是函数指针，函数对象或者有名称的Lambda表达式，这样会导致函数模板效率很低，

```c++
template<typename T, typename F>
T use_f(T v, F f)
{
    return f(v);
}
```

这个函数模板使用第二个参数（函数符），来调用第一个参数，第二个参数可能是函数指针，函数对象或者有名称的Lambda表达式，这导致模板根据不同的参数进行了不同的实例化，

```c++
double y = 2.58;
use_f(y, square); // square是一个函数指针
use_f(y, Fp(5.5)); // Fp是一个重载了()的对象
use_f(y, [](double x)
{
    return x * x;
});                // 这里的第二个参数是一个Lambda表达式
```

上面的代码使用了3种不同的函数符，导致use_f实例化了三次。但是真的需要如此吗，这三种函数符很多地方都是相似的，它们都接受一个double类型的参数，返回一个double类型的值，实际上，他们的类型都为double(*)(double)，根本不需要为每一种类型实例化一个模板，而std::function就是用来解决这个问题，std::function的使用方法为：

```c++
std::function<double(double)> fd;
```

这样就可以将double(*)(double)类型的函数符都赋值给它，

```c++
std::function<double(double)> f1 = square;
std::function<double(double)> f1 = Fp(5.5);
std::function<double(double)> f1 = [](double x)
{
    return x * x;
});

use_f(y, f1);
```

使用这种方式后，模板就只会实例化一次。

std::function的使用方式还能更灵活，比如上面的代码，不需要创建3个std::function<double(double)>对象，可以只使用一个临时对象，

```c++
typedef std::function<double(double)> fdd;

use_f(y, fdd(square));
use_f(y, fdd(Fp(5.5))); // 类似强制类型转换，还是function的构造函数
...
```

或者直接将function声明在函数模板之内，

```c++
template<typename T>
T use_f(T v, std::function<T(T)> f) // 不再使用第二个模板参数，将其声明为T(*)(T)的函数符，当然现在适应的函数符范围变小了，比如不能使用接受两个参数的函数了。
{
    return f(v);
}
```

调用函数时就可以这样写，

```c++
use_f<double>(y, square);
use_f<double>(y, Fp(5.5));
use_f<double>(y,[](double x)
{
    return x * x;
});
```

由于传入的函数符本身不是 std::function<double(double)>，所以这里要指定double的实例化，让第二个参数生成std::function<double(double)>类型。

### 可变参数模板

可变参数模板用于创建接受可变数量参数的模板函数和类（类似C中的stdarg.h），要创建可变参数模板，需要理解以下几点，

- 模板参数包
- 函数参数包
- 展开（unpack）参数包
- 递归

C++11提供了一个源运算符...，可以用来声明表示模板参数包的标识符，模板参数包一般是一个类型列表。同时它也可以用来声明表示函数参数包的标识符，函数参数包一般是一个值的列表，它的语法如下：

```c++
template <typename... Args>  // Args是一个模板参数包
void show_list(Args... args) // args是一个函数参数包,这表示args的类型为Args
{
    ...
}
```

Args与常用的模板类型T的不同之处在于，T只能匹配一种类型，而Args可以匹配多种类型，

```c++
show_list(1, 1.2, "ppp");
// Args 匹配了int, double, const char*类型
// args 匹配了1, 1.2, "ppp" 3个值，args与Args一一对应，无论类型还是数量
```

#### 使用递归展开参数包

参数包展开的核心理念是，对第一个参数进行处理，再将剩下的参数传递给函数递归调用，直到列表为空，这里的技巧是将模板头修改为如下：

```c++
template <typename T, typename... Args>  
void show_list(T value, Args... args)
```

这样每次都可以处理第一个参数，直到参数列表为空，

```c++
template <typename T, typename... Args>  
void show_list(T value, Args... args)
{
    std::cout << value << ",";
    show_list(Args... args);
}
```

当args为空时，将调用show_list()，导致处理结束。

当然，如果需要对最后一个参数做特殊的处理，比如想让最后一个参数输出后输出换行符而不是","，那么可以再定义一个模板，让其与其他模板不同，这样到最后一个参数的时候就会匹配到此模板，

```c++
template <typename T>  
void show_list(T value)
{
    std::cout << value << std::endl;
}
```

另一个可以改进的点是将按值传递改为按引用传递，以此提升性能：

```c++
template <typename T>  
void show_list(const T& value)
{
    std::cout << value << std::endl;
}

template <typename T, typename... Args>  
void show_list(const T& value, const Args&... args)
{
    std::cout << value << ",";
    show_list(Args... args);
}

show_list(1, 1.0, "111");
```

