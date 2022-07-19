## 带指针的类 String

#### 关于交换操作

> *《C++ Primer 中文版 第5版》* 第457页。

如果没有定义自己的 swap 函数，使用标准库的 swap 操作可能是这样：

```cpp
String temp = s1; // copy s1
s1 = s2; // delete s1, copy s2
s2 = temp; // delete s2, copy s1
```

这些多余的操作是没有必要的。因此为了效率有必要定义自己的 swap 函数：

```cpp
inline
void swap(String &ls, String &rs){
    using std::swap;
    // 使用 swap 而不是 std::swap，因为成员有可能也有自己的 swap
    swap(ls.m_data, rs.m_data); // 交换指针而不是 data
}
```

如果定义了 swap，那么可以**在赋值运算符中使用 swap，这样可以自动处理自赋值情况且天然就是异常安全的。**此外，与原来的拷贝赋值相比具有同等的效率，在传递实参时执行了一次拷贝构造，此后没有新的内存消耗和操作。

```cpp
String& String::operator=(String rs){
    swap(*this, rs); // 交换左侧运算对象和局部变量 rs 的内容
    return *this;
} // rs 被销毁，从而 delete 了 rs 中的指针
```



#### 对象移动

> *《C++ Primer 中文版 第5版》* 第470页。

**C++11 可以移动而非拷贝对象。**移动对象是很有必要的，比如对于一个 `vector<String>` ，当分配的内存不能添加新成员时，需要新分配一块更大的内存，并将原来的数据“移动”到新内存中。如果没有所谓的“移动”操作，则只能拷贝过去。拷贝一份新数据又将原来数据 delete，这明显是多余的。





#### 关于 new 和 delete

使用 `new` 和 `delete` 来创建和删除对象，是一个固定的表达式，这无法改变。但表达式被拆分为几个步骤，可以对这些步骤的操作符 `operator new` 、`operator delete` 进行重载：

值得注意的是：在使用 `new` 和 `delete` **表达式**时并不需要提供任何参数，表达式被拆分的第一步调用 `operator new(sizeof(Foo))` 会自动计算需要的内存大小，提供给参数 `size_t`。

![image-20220513195607348](images/image-20220513195607348.png)

> 需要注意的是，上述重载在 `class Foo` 内部，因此只对 `Foo` 类的 `new` 和 `delete` 有效。而如果在类外部，则重载的是全局操作符，将产生全局的影响！危险！！
>
> 如果用户不想使用类内重载的 `new` 或 `delete` ，使用 `::new` 显示地调用全局版本。

#### 关于堆、栈与内存管理

在函数体内声明的变量存储在 stack 中，函数体结束后自动释放其占用的所有内存。
动态分配（new），则使用 heap 中的内存，只有手动 delect 时才释放内存。

```cpp
class Complex {...};
...
/************ stack objects ************/
{
    // 其生命在作用域结束之后就结束了
    Complex c1(1, 2);
} // 函数体结束，内存释放

/********* static local objects *********/
{
    // 其生命在作用域结束之后仍然存在，直到整个程序结束
    static Complex c2(1, 2);
}

/************ global objects ************/
// 作用域是整个程序，其生命在整个程序结束之后才结束
Complex c3(1, 2);
int main() {
    ...
}

/************* heap objects *************/
{
    // 其生命在它被 delete 之后才结束
    Complex* p = new Complex;
    ...
    delete p;
}
```

此外，array new 一定要搭配 array delete

![](./images/arraynew.png)



#### String 类总览

```cpp
class String
{
public:                                 
   String(const char* cstr=0);                     
   String(const String& str); // 拷贝构造
   String& operator=(const String& str); // 拷贝赋值
   ~String(); // 析构函数
   char* get_c_str() const { return m_data; }
private:
   char* m_data; // class with point member
};

#include <cstring>
inline String::String(const char* cstr)
{
   if (cstr) {
      m_data = new char[strlen(cstr)+1];
      strcpy(m_data, cstr);
   }
   else {   
      m_data = new char[1];
      *m_data = '\0';
   }
}

inline String::~String()
{
   delete[] m_data; // new 出来的内存都需要手动 delete
}

inline String::String(const String& str)
{
   m_data = new char[ strlen(str.m_data) + 1 ];
   strcpy(m_data, str.m_data);
}

// 因为考虑到可能连续赋值，因此应该返回
inline String& String::operator=(const String& str)
{
   if (this == &str) // 检测自我赋值
      return *this;

   delete[] m_data;
   m_data = new char[ strlen(str.m_data) + 1 ];
   strcpy(m_data, str.m_data);
   return *this;
}
```



## 补充

#### static

首先看看普通的情况：对于一个 class 的多个对象，他们有各自独立的成员变量，但是**使用同一份成员函数代码**。当不同的对象调用成员函数时，传入 this 指针来标识自己，但内存中只存有一份函数代码，不同对象共享。

![image-20220207142116053](./images/image-20220207142116053.png)

**static 成员变量：**所有对象共享

**static 成员函数：**与非 static 函数一样，不同对象使用同一份函数代码，但是 static 函数没有接收 this 指针的参数，因此不能访问对象的非 static 变量，只能访问 static 成员变量。

```cpp
class Account {
public:
    static double m_rate; // 声明
    static void set_rate(const double& x) { m_rate = x; }
};
double Account::m_rate = 8.0; // 定义（声明只指明了变量类型和名字，定义则为其分配了内存）

int main() {
    Account::set_rate(5.0); // 通过 class name 调用
    
    Account a;
    a.set_rate(7.0); // 通过 object 调用
}
```

**Meyers Singleton** 设计模式：无法自己创建这个 class 的对象，只有使用时才可以获得，并且永远只有一份：

```cpp
class A {
public:
    static A& getInstance();
    setup() {...}
private:
    A(); // 构造函数私有
};

A& A::getInstance() {
    static A a;
    return a;
}
```



#### 模板类 template class

```cpp
template<typename T>
class complex {
public:
    complex (T r = 0, T i = 0) : re(r), im(i) {}
    ...
}

{
    complex<double> c1(2.5, 1.5);
    complex<int> c2(2, 6);
}
```

模板会导致代码的“膨胀”，比如上面的 complex 类，当使用 double 时会将所有的 T 置换为 double 产生一份代码，当使用 int 时又会同样的产生另一份代码。但这种膨胀是必要的。



#### 模板函数 template function

```cpp
template <typename T>
inline
const T& min(const T& a, const T& b) {
    return b < a ? b : a;
}

{
    complex c1(2, 3), c2(3, 3), c3;
    c3 = min(c1, c2);
}
```

调用模板函数时不需要像调用模板类时那样指定 type，编译器会对模板函数进行“参数推导”。

`<algorithm.h>` 库中的各种算法都是模板函数实现。