# Effective C++

## 第一章 让自己习惯 C++

### 条款 01 ：视 C++ 为一个语言联邦

> C++ 高效编程守则视情况而变化，取决于你使用 C++ 的哪部分

- 面向过程、面向对象、泛型、函数式、模板元编程
- C：预处理器、内置数据类型、数组、指针等
- 面向对象的 C++：构造函数、析构函数、封装、继承、多态
- 模板的C++：
- STL：容器、迭代器、算法、函数对象

### 条款 02 ：尽量以 const、enum、inline 替换 #define

> 对于单纯常量，最好以 const 对象或者 enums 替换 #define
>
> 对于形似函数的宏，最好改用 inline 函数替换 #define

- 以编译器代替预处理器，因为 #define 不被视作语言的一部分

- 常量
  - 常量指针：const int myAge = 19; int * const age = &myAge;  顶层 const
  - 指向常量的指针：const char * name;  底层 const

- 类专属常量

```c++
// 为了将常量的作用域限制于类内，必须让它成为类的一个成员；而为确保此常量至多只有一份实体，必须让它成为一个 static 成员
 class GamePlayer {
     private:
     	static const int NumTurns = 5;	// 常量声明式，而非定义式
     	int scores[NumTurns];			// 使用该常量
 }
// 对于一个类专属静态整数类型，只要不取地址，可以不提供定义式
// 以下为定义式，在类中已经获得初始值，在定义时不可以再设初值
const int GamePalyer::NumTurns;
```

- 无法利用 #define 创建一个类专属常量，因为 #define 不重视作用域
- the enum hack - 一个属于枚举类型的数值可权充 int 被使用

```c++
 class GamePlayer {
     private:
     	enum { NumTurns = 5 };
     	int scores[NumTurns];			
 }
// enum hack 的行为某方面像 #define 而不像 const
// 取一个 const 的地址是合法的，取一个 #define 或者 enum 的地址不合法
```

- template inline 函数可以获得宏函数带来的效率以及一般函数的所有可预料行为和类型安全性

### 条款 03 ：尽可能使用 const

> 将某些东西声明为 const 可以帮助编译器侦测出错误用法。const 可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。
>
> 编译器强制实施 bitwise constness，但你编写程序时应该使用概念上的常量性。
>
> 当 const 和非 const 成员函数有着实质等价的实现时，令非 const 版本调用 const 版本可避免代码重复。

- 顶层 const、底层 const (* 前 const 表示数据不变，* 后 const 表示指针不变)

```c++
char greeting[] = "hello"；
char* p = greeting;			    // non-const pointer, non-const data
const char* p = greeting;       // non-const pointer, const data
char* const p = greeting;       // const pointer, non-const data
const char* const p = greeting; // const pointer, const data
```

- 令函数返回一个常量值，往往可以降低因客户自身错误而造成的意外，而又不至于放弃安全性和高效性。

```c++
const Rational operator* (const Rational &lhs, const Rational &rhs);
Rational a, b, c;
a * b = c			// 如果返回值不为 const，此操作合法但无意义
```

- const 成员函数（两层含义）
  - 函数不会对数据成员做出改动
  - 该 const 成员函数可以用在 const 对象身上

- 两个函数如果只是常量性不同，可以被重载

```c++
class TextBlock {
public:
    const char& operator[](std::size_t pos) const { return text[pos]; }
    char& operator[](std::size_t) { return text[pos]; }
private:
    std::string text;
}
// 第一个函数操作 const 对象，第二个操作非 const 对象
// 第一个的返回值也是 const，但是是否能重载看的是后面的 const
// 函数的返回值都是引用，因为要修改 text 里面的值
void print(const TextBlock& ctb) {
    std::cout << ctb[0];		// 调用 const 版本
}
```

- 成员函数为 const 的实际意义
  - bitwise constness：编译器对常量性的定义，成员函数不改变对象的任何一个 bit。（很多成员函数属于 bitwise const，但却不具备逻辑上的常量性，比如一个更改了指针所指物的成员函数）
  - logical constness：可以修改对象的某些 bit，只要具备逻辑意义的常量性。（声明为常成员函数后，编译器不允许改动对象，所以需要 mutable。mutable 释放掉非静态数据成员的 bitwise constness 约束）

- 当 const 和非 const 成员函数有着实质等价的实现时，令非 const 版本调用 const 版本可避免代码重复。

```c++
char& operator[](std::size_t pos) {
    return const_cast<char&>(static_case<const TextBlock&>(*this)[pos]);
}
// 第一次转型：为 *this 加上 const，如果不加调用的还是非 const 函数，导致无穷递归
// 第二次转型：去掉返回值里的 const
```

- 不可以让 const 版本成员函数调用非 const 版本

### 条款04 ：确定对象被使用前已先被初始化

> 为内置类型进行手工初始化，因为C++不保证初始化它们。
>
> 构造函数最好使用成员初始化列表，而不要在构造函数本体内使用赋值操作。初始化列表列出的数据成员，其排列次序应该和它们在类中的声明次序相同。
>
> 为免除跨编译单元的初始化次序问题，请以 local static 对象替换 non-local static 对象

- 永远在使用对象之前先将它初始化。对于内置类型，需要手工初始化；对于类类型，构造函数来初始化。

- 不要混淆赋值与初始化。总是使用成员初始化列表。总是在成员初始化列表中列出所有数据成员。
- 基类早于派生类被初始化，类的成员变量总是以其声明的次序被初始化。





## 第二章 构造、析构、赋值运算

### 条款 05 ：了解 C++ 默默编写并调用哪些函数

> 编译器可以暗自创建默认构造函数、拷贝构造函数、拷贝赋值运算符以及析构函数

- 默认构造函数、析构函数：用来放置藏身幕后的代码，比如调用基类或者非静态成员的构造函数。

- 拷贝构造函数：
  - 内置类型：拷贝每一个 bits
  - 类类型：调用其构造函数
- 拷贝赋值运算符：不合法时不会生成
  - 成员变量为 reference-to-non-const，（不允许 reference 改指向不同对象）
  - 成员变量为 const，（不允许更改 const 成员）
  - 基类将拷贝赋值运算符声明为 private，（派生类生成的拷贝赋值运算符无法处理基类成分）

### 条款 06 ：若不想使用编译器自动生成的函数，就应该明确拒绝

> 为驳回编译器暗自提供的功能，可将相应的成员函数声明为 private 并且不予实现，或者使用类似 Uncopyable 的基类。

- 将拷贝构造函数和拷贝赋值运算符声明为 private，
  - 阻止了编译器创建默认函数，阻止了使用者调用    ---->    编译器错误
  - 但是成员函数和友元函数还是可以调用
- 将拷贝构造函数和拷贝赋值运算符声明为 private，并且不实现它们
  - 在调用时会得到一个连接错误（linkage error）    ---->    连接器错误
- 定义 Uncopyable 基类，并将拷贝构造函数和拷贝赋值运算符声明为 private
  - 将连接期错误提前编译期

- =delete

### 条款 07 ：为多态基类声明虚析构函数

> 带多态性质的基类应该声明一个虚析构函数。如果类带有任何虚函数，它就应该拥有一个虚析构函数。
>
> 类设计的目的如果不是作为基类使用，或不是为了具备多态性，就不应该声明虚析构函数。

- 当派生类对象经由一个基类指针被删除，而该基类带着一个非虚析构函数，其结果未定义，实际执行时通常发生的是对象的派生成分没有被销毁
- 任何类只要带有虚函数都几乎确定应该也有一个虚析构函数
- 如果类不含虚函数，令其析构函数为虚是不合适的，因为会多出虚指针(vpte)和虚表(vtbl)的内存占用
- 企图继承一个标准容器或者任何其他带有非虚析构函数的类都会产生严重的后果
- 纯虚析构函数（抽象类、虚析构）

### 条款 08 ：别让异常逃离析构函数

> 析构函数绝对不要吐出异常。如果一个析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下他们或者结束程序
>
> 如果客户需要对某个操作函数运行期间抛出的异常作出反应，那么类应该提供一个普通函数（而非在析构函数中）执行该操作

- 析构函数吐出异常会导致程序过早结束或产生未定义的行为
- 避免异常逃离可以有强迫结束程序和吞下异常两种选择，或者重新设计接口，使用户有机会对可能出现的问题做出反应
- 如果某个操作可能在失败时抛出异常，而又存在某种需要必须处理该异常，那么这个异常必须来自析构函数以外的某个函数

### 条款 09 ：绝不在构造和析构过程中调用虚函数

> 在基类构造和析构期间不要调用虚函数，因为这类调用从不下降至派生类

- 在基类构造期间，虚函数不是虚函数
- 在派生类对象的基类构造期间，对象的类型是基类而不是派生类（虚函数、typeid、dunamic_cast 都会将对象看作基类）
- 一旦派生类析构函数开始执行，对象内的派生成分便呈现未定义值，进入基类析构函数后对象就成为一个基类对象
- 要确保构造函数和析构函数内没有调用虚函数，构造函数和析构函数调用的函数也不能调用虚函数
- 令派生类将必要的构造 信息向上传递至基类构造函数来弥补虚函数无法在基类向下调用的遗憾

### 条款 10 ：令 operator= 返回一个 reference to *this

> 令赋值运算符返回一个 reference to *this

- 实现连锁赋值
- 所有赋值相关运算都要遵守（=、+= ...）

### 条款 11 ：在 operator= 中处理自我赋值

> 确保当对象自我赋值时 operator= 有良好的行为。其中技术包括：比较来源对象和目的对象的地址，精心周到的语句顺序，以及 copy-and-swap
>
> 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确

- 错误：delete 不仅删除了 *this 的 bitmap (数据成员)，同时也删除了 rhs 的 bitmap，返回了一个指针指向的是被删除的对象    ---->    不具备自我赋值安全性，不具备异常安全性

```c++
Widget &Widget::operator= (const Widgte &rhs) {
    delete pb;
    pb = new Widget(rhs.pb);
    return *this;
}
```

- 证同测试 - 比较来源对象和目的对象的地址    ---->    不具备异常安全性(new 或者 pb 的拷贝构造出错)

```c++
// 在开始加上
if (this == &rhs) return *this;
```

- 精心周到的语句顺序

```c++
Widget &Widget::operator= (const Widgte &rhs) {
    Bitmap *pOrig = pb;
    pb = new Widget(rhs.pb);
    delete pOrig;
    return *this;
}
```

- copy-and-swap （条款 29）

```c++
Widget &Widget::operator= (const Widgte &rhs) {
    Widget temp(rhs);
    swap(temp);			// 条款 29
    return *this;
}
// 以下：牺牲可读性，可以提升效率
Widget &Widget::operator= (Widgte rhs) {  // pass by value 构造了副本，
    swap(rhs);		
    return *this;
}
```

### 条款 12 ：复制对象时勿忘其每个成分

> 拷贝函数应该确保复制对象内的所有成员变量以及所有基类成分
>
> 不要尝试以某个拷贝函数实现另一个拷贝函数。应该将共同机能放进第三个函数中，并由两个拷贝函数共同调用

- 自己实现的拷贝构造和拷贝赋值即使执行了局部拷贝，编译器也不会报错
- 未拷贝基类成分时：拷贝构造函数中的基类成分使用使用默认构造函数构造，拷贝赋值运算符中的基类成分使用原来的值
- 具体实现

```c++
PriorityCustomer::PriorityCustomer (const PriorityCustomer &rhs) 
	: Customer(rhs)				// 调用基类拷贝构造函数
    , priority(rhs.priority){
    ...
}
PriorityCustomer &PriorityCustomer::operator= (const PriorityCustomer &rhs) {
    Customer::operator=(rhs);    // 调用基类拷贝赋值运算符
    priority = rhs.priority;
    ...
    return *this;
}
```

- 令拷贝赋值运算符调用拷贝构造函数是不合理的。试图构造一个以及存在的对象
- 令拷贝构造函数调用拷贝赋值运算符同意无意义。对尚未构造好的对象赋值

## 第三章 资源管理

### 条款 13 ：以对象管理资源

> 为防止资源泄露，请使用 RAII 管理对象，它们在构造函数中获得资源并在析构函数中释放资源。
>
> 被经常使用的 RAII 类是 shared_ptr 和 unique_ptr，前者通常是较佳选择，因为其拷贝行为比较直观。若选择 unique_ptr，std::move 会使得被复制一方为空。

- C++ 常使用的资源：动态分配内存、文件描述符、互斥锁、数据库连接、socket
- 关键思想
  - 获得资源后立刻放进管理对象内    ---->    资源取得时机便是初始化时机（RAII）
  - 管理对象运用析构函数确保资源被释放
- 在动态分配而得的数组身上使用 shared_ptr 等智能指针是个馊主意，因为有 vector、string 等容器是为更好的选择

```c++
std::shared_ptr<int> isp(new int[1024]);    // shared_ptr 在内部做 delete 动作，而不是delete[]，编译不会报错，但是会产生未定义行为
```

### 条款 14 ：在资源管理类中小心拷贝行为

> 复制 RAII 对象必须一并复制它所管理的资源，所以资源的拷贝行为决定 RAII 对象的拷贝行为
>
> 普遍而常见的 RAII 类拷贝行为是：禁用拷贝、引用计数

- 禁用拷贝：private成员、Uncopyable 类、=delete
- 引用计数：
- 复制：深拷贝
- 转移底部资源的在所有权：unique_ptr、std::move()

### 条款 15 ：在资源管理类中提供对原始资源的访问

> APIs 往往要求访问原始资源，所以每一个 RAII 类应该提供一个取得其所管理的资源的办法
>
> 对原始资源的访问可能经由显示转换或隐式转换，一般而言显示转换比较安全，但隐式转换对客户比较方便。

- 显示转换
  - shared_ptr 等智能指针的 get 成员函数
  - 重载指针取值操作符（operator->、operator*）
- 隐式转换
  - 重载 operator()

### 条款  16 ：成对使用 new 和 delete 时要采用相同形式

> 如果你在 new 表达式中使用 []，必须在相应的 delete 表达式中也使用 []。如果你在 new 表达式中不使用 []，一定不要在相应的 delete 表达式中使用 []。

### 条款 17 ：以独立语句将 new 创建的对象置入智能指针

> 以独立语句将 new 创建的对象置入智能指针。如果不这样做，一旦异常被抛出，有可能导致难以觉察的内存泄露。

```c++
int priority();
void process(std::shared_ptr<Widget> pw, int priority);

process(new Widget, priority());// 错误，shared_ptr 构造函数是 explicit，无法进行隐式转换
process(std::shared_ptr<Widget>(new Widget), priority());//正确，但是可能资源泄露
// 三个操作：执行 new Widget、调用 priority()、调用 std::shared_ptr 构造函数
// 三个的执行顺序不确定，可能是 1、2、3。如果对 priority() 的调用出错，则 new Widget 
// 返回的指针会泄露
```

- 编译器对语句内的各项操作有重新排列的自由，跨语句各项操作没有

- 在资源被创建和资源被转换为资源管理对象两个时间节点间可能发生异常干扰

```c++
std::shared_ptr<Widget> pw(new Widget);
process(pw, priority());
```

## 第四章 设计与声明

### 条款 18 ：让接口容易被正确使用，不易被错误使用

> 促进正确使用的方法：接口的一致性、与内置类型的行为兼容
>
> 阻止误用的方法：建立新类型、限制类型上的操作、束缚对象值、消除客户的资源管理责任
>
> shared_ptr 支持定制删除器。这可以防范 DLL 问题，可被用来自动解除互斥锁。

- 导入新类型。如为日期的构造函数传入年月日，用户可能输入顺序出错或者输入无效数字，这时分别新建年月日类作为外覆类型来区分天数、月份和年份。

```c++
Date d(30, 3, 1995);		// error
Date d(Day(30), Month(3), Year(1995));
```

- 确定正确的类型后，就需要限制其值在合理范围内。如年月日的合法值，比较安全的解法是预先定义所有有效值(这里应该是对于有限制来说)。

```c++
class Month {
public:
	static Month Jan() { return Month(1); }		// 这些是函数而非对象
    static Month Feb() { return Month(2); }
    ...
private:
    explicit Month(int m);						// 阻止外部调用
}
Date d(Day(30), Month::Mar(), Year(1995));
```

- 限制类型上的操作。比如加上 const，以 const 修饰 operator* 的返回类型可阻止用户因用户自定义类型而犯错。

```c++
if (a*b = c) ... //	 愿意是比较
```

- 消除资源管理的负担。任何接口如果要求客户必须记得做某些事情，就是有着不正确使用的倾向，因为用户可能会忘记做。

```c++
Investment *createInvestment();
std::shared_ptr<Investment> createInvestment();   //直接返回智能指针，而不是等待用户将返回值存入智能指针
```

- shared_ptr 会使用每个指针专属的删除器，可以消除跨 DLL 问题(cross DLL problem)。对象在动态链接程序库中被 new 创建，在另一个 DLL 中被 delete 销毁。

### 条款 19 ：设计 class 犹如设计 type

> 很复杂，需要慢慢体会。P84

### 条款 20 ：宁以 pass by reference to const 替换 pass by value

> 尽量以 pass by reference to const 替换 pass by value。前者通常比较高效，并且可以避免切割问题。
>
> 以上规则并不适用于内置类型，以及 STL 的迭代器和函数对象。对它们而言，pass by value 比较适当

- 当一个派生类对象以值传递的方式并被视作一个基类对象时，会产生切割问题，只保留了基类的部分。
- 对象小不意味着拷贝构造不复杂，因为可能涉及深拷贝。

### 条款 21 ：必须返回对象时，别妄想返回其 reference

> 绝不要返回指针或引用指向一个 local stack 对象，或者返回引用指向一个 heap-allocated 对象，或者返回指针或引用指向一个 local static 对象而有可能同时需要多个这样的对象。

- 返回 local stack 对象。对象在函数退出前被销毁。
- 返回 heap-allocated 对象。new 出来的对象谁来销毁，何时销毁。
- 返回 local static 对象。多个地方同时需要同一个静态对象，导致不合理的行为。

```c++
const Rational& operator*(const Rational &lhs, const Rational &rhs) {
    static Rational result;
    ...
    return result;
}
if ((a * b) == (c * d)) ...    // 永远为true，同一个静态对象相比较
```

- 必须返回一个新对象时就返回一个新对象。

### 条款 22 ：将数据成员声明为 private

> 切记将数据成员声明为 private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供类作者以充分的实现弹性。
>
> protected 并不比 public 更具封装性。

- 一致性。统一使用函数访问数据成员。
- 使用函数可以对数据成员有更精确的控制。只读，只写，读写，禁止访问。
- 确保类的约束条件，保留了日后变更的权力。
- 封装。可以换用不同的实现而外部没有感知，为所有可能的实现提供弹性。
- public 意味着不封装，不封装意味着不可改变，因为改变影响巨大。
- 某些东西的封装性与**当其内容改变时可能造成的代码破坏量**成反比。public 和 protected 数据成员被改变，都会有不可预知的大量代码受到破坏，所以其封装性相当。

### 条款 23 ：宁以非成员、非友元替换成员函数

> 宁以非成员、非友元替换成员函数。这样做可以增加封装性、包裹弹性和机能扩充性。

- 封装性。考虑数据成员的封装性，越多函数可以访问它，封装性越低。非成员非友元函数比成员函数或友元函数有更大的封装性。注意一个类的非成员函数可以是另一个类的成员函数。

- 包裹弹性。与某个类相关的大量便利函数（非成员非友元）一般写于与类相同的命名空间下，可以按功能分在不同头文件里，编程时只引入需要的头文件。
- 机能扩充性。用户可以在同样的命名空间下编写遍历函数，这种性质类无法提供，因为用户不能扩展类。用户可以继承类，但是派生类不能访问父类的私有数据成员。

### 条款 24 ：若所有参数皆需类型转换，请为此采用非成员函数

> 如果你需要为某个函数的所有参数（包括隐含参数 this）进行类型转换，那么这个函数必须是个非成员函数

- 令类支持隐式转换非常糟糕，但是数值类除外
- 该条款常用于 **四则运算符重载** 时。
- 成员函数的反面是非成员函数，不是友元函数。一个与类相关的函数如果不作为成员函数时，不一定要作为友元函数，非成员非友元函数也可以。

### 条款 25 ：考虑写出一个不抛异常的 swap 函数

> 当 std::swap 对你的类型效率不高时，提供一个 swap 成员函数，并确定这个函数不抛出异常。
>
> 如果你提供一个成员 swap 函数，也应该提供一个非成员 swap 函数来调用前者。对于类（而不是类模板），请特化 std::swap
>
> 调用 swap 时应针对 std::swap 使用 using 定义式，然后调用 swap 并且不带任何命名空间资格修饰
>
> 为用户自定义类型进行 std 模板全特化是好的，但千万不要尝试在 std 内加入某些对 std 而言全新的东西

- swap 是异常安全性编程的核心，以及用来处理自我赋值的常见机制。

```c++
// std::swap 的典型实现
namespace std {
    template<typename T>
    void swap (T& a, T& b) {
        T temp(a);
        a = b;
        b = temp;
    }
}
```

- 对于 pImpl 手法来说，默认实现效率不高

```c++
// 令 Widget 声明一个 swap 成员函数，然后将 std::swap 特化
namespace liucxi {
    class Widget {
    public:
    	void swap(Widget& other) {
            using std::swap;
            swap(pImpl, other.pImpl);
        }    
    }
    
    void swap (Widget& a, Widget& b) {
        a.swap(b);
    }
}
namespace std {
    template<>
    void swap<Widget> (Widget& a, Widget& b) {
        a.swap(b);
    }
}

// 如果是一个类模板，非成员的 swap 函数就不能在 std 空间了
namespace liucxi {
    template<typename T>
    class Widget { ... }
    
    template<typename T>
    void swap (Widget<T>& a, Widget<T>& b) {
        a.swap(b);
    }
}
```

- swap 成员函数、swap 非成员函数、全特化的 std::swap（类有而类模板没有）、默认的 std::swap

## 第九章 杂项讨论

### 条款 53 ：不要轻易忽视编译器的警告

> 严肃对待编译器发出的警告信息。
>
> 不要过度依赖编译器的报警能力，因为不同编译器对待事情的态度并不相同。

### 条款 54 ：让自己熟悉标准程序库（98，TR1，11，20）

> C++98 ：STL(容器、迭代器、算法、函数、适配器)、iostream、exception、C89标准库
>
> TR1 ：智能指针(shared_ptr、weak_ptr、unique_prtr)、一般化函数指针 function、bind、hash 容器、Type Traits

### 条款 55 ：让自己熟悉 Boost
