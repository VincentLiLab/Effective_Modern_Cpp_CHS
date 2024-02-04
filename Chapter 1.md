- [Chapter 1 类型推导](#chapter-1-类型推导)
  - [Item 1 理解模板的类型推导](#item-1-理解模板的类型推导)
    - [场景 1：_ParamType_ 是指针类型或者引用类型，但不是 _universal reference_](#场景-1paramtype-是指针类型或者引用类型但不是-universal-reference)
    - [场景 2：_ParamType_ 是 _univeral reference_](#场景-2paramtype-是-univeral-reference)
    - [场景 3：_ParamType_ 既不是指针类型也不是引用类型](#场景-3paramtype-既不是指针类型也不是引用类型)
    - [数组实参](#数组实参)
    - [函数实参](#函数实参)
    - [需要记住的规则：](#需要记住的规则)
    - [译者总结](#译者总结)
  - [Item 2 理解 _auto_ 的类型推导](#item-2-理解-auto-的类型推导)

# Chapter 1 类型推导

_C++98_ 只有一组类型推导规则：函数模板所对应的类型推导。_C++11_ 稍微修改了规则，新增了两组规则：_auto_  
和 _decltype_。_C++14_ 扩展了 _auto_ 和 _decltype_ 的使用范围。类型推导的日益广泛的应用让你从明显的或多余的类型  
拼写的专横中获得了解脱。类型推导使得 _C++_ 软件更有适应性了，因为在源码的一个地方上改变类型会自动通过  
类型推导来传播到其他地方。然而类型推导会使代码变的难于理解，因为编译器推导的类型，可能不会像你想的那  
样明显。

只有深刻理解了类型推导是如何工作的，才能在 _modern C++_ 中进行高效编程。有很多类型推导的场景：调用函  
数模板时、大多数 _auto_ 出现的情景中、_decltype_ 表达式中和 _C++14_ 中的 _decltype(auto)_ 中。

这章提供所提供的类型推导的内容是每一个 "C++" 开发者都需要的。这些内容解释了模板的类型推导是如何工作  
的、_auto_ 是如何在模板的类型推导之上构建的，以及 "decltype" 是如何走它自己的路的。本章甚至解释了如何强  
制编译器使类型推导的结果变得可见，这能够确保编译器正在推导你想让它们去推导的类型。

## Item 1 理解模板的类型推导

当复杂系统的用户不知道它是如何工作却对它的工作感到满意时，就可以表明这个系统的设计是非常出色的。按照  
这样说，_C++_ 的模板的类型推导也是非常成功的。成千上万的程序员已经传递实参到模板函数中，并且得到了完  
全正确的结果，对于这些函数所使用的类型是如何被推导的，很多程序员除了模糊的描述外，很难再给出更好的描  
述。

如果你也不能给出更好的描述的话，那么我有好消息和坏消息。好消息是模板的类型推导是 _auto_ 的基础，而 _auto_  
则是 _modern C++_ 中最吸引人的特性之一。如果你对 _C++98_ 的模板的类型推导感到满意，那么你也准备好了对  
_C++11_ 的 _auto_ 的类型推导感到满意。坏消息是当模板的类型推导规则被应用到 "auto" 上时要比被应用到模板上  
时难理解一点。因此，真正地理解模板的类型推导是非常重要的，因为 _auto_ 是在模板的类型推导之上构建的。本  
_Item_ 覆盖了你需要知道的所有。 

如果你愿意忽略一些伪代码，我们可以考虑一个像下面那样的函数模板：  
```C++
  template<typename T>
  void f(ParamType param);
```  
一个调用可以看起来像是这样：  
```C++
  f(expr);                    // call f with some expression
```

在编译期间，编译器使用 _expr_ 去推导两个类型：一个是 _T_ 类型，一个是 _ParamType_ 类型。这两个类型经常是不相  
同的，因为 _ParamType_ 常常会包含有装饰，比如：_const_ 或 _&_。例如，如果模板声明成下面这样：  
```C++
  template<typename T>
  void f(const T& param);     // ParamType is const T&
```  
然后我们这样调用：  
```C++
  int x = 0;
  
  f(x);                       // call f with an int
```  
_T_ 被推导为 _int_，而 _ParamType_ 被推导为 _const int&_。

会很自然地期待 _T_ 和所传递给函数的实参的类型得是一致的，即为：_T_ 是 _expr_ 的类型。上面的例子就是这样的场景：_x_ 是 _int_，_T_ 被推导为了 _int_，但并不总是会这样。_T_ 不仅依赖于 _expr_ 的类型，还依赖于 _ParamType_ 的格式。共有三种场景：  
 
* _ParamType_ 是一个指针类型或者引用类型，但不是 _universal reference_，_universal reference_ 会在 [_Item 24_](./Chapter%205.md#item-24-区分通用引用和右值引用) 中进  
  行描述。现在你只需要知道是 _universal reference_ 是存在的，而且和左值引用或右值引用是不相同的。
* _ParamType_ 是一个 _universal reference_。
* _ParamType_ 既不是指针类型也不是引用类型。

因此我们有三种类型推导的情况来检查。每个都基于我们的模板的通用格式，然后调用它：  
```C++

  template<typename T>
  void f(ParamType param);
  
  f(expr);                    // deduce T and ParamType from expr
```

### 场景 1：_ParamType_ 是指针类型或者引用类型，但不是 _universal reference_

最简单的场景是当 _ParamType_ 是引用类型或者指针类型，但不是 _universal reference_ 时。在这个场景中，类型推导是像下面这样工作的：  
* 如果 _expr_ 的类型是引用，忽略引用部分。
* 通过 _expr_ 的类型和 _ParamType_ 进行模式匹配去确定 _T_。

比如，这是我们的模板，  
```C++
  template<typename T>
  void f(T& param);           // param is a reference
```  
然后我们有这些变量声明：  
```C++
  int x = 27;                 // x is an int
  const int cx = x;           // cx is a const int
  const int& rx = x;          // rx is a reference to x as a const int 
```  
_param_ 对应的推导的类型和各种调用中的 "T" 是像下面的的这样：  
```C++
  f(x);                       // T is int, param's type is int&
  
  f(cx);                      // T is const int,
                              // param's type is const int&
  
  f(rx);                      // T is const int,
                              // param's type is const int&
```

在第二个和第三个调用中，因为 _cx_ 和 _rx_ 是 _const_ 类型的，所以 _T_ 会被推导为 _const int_，因此产生了 _const int&_ 的  
形参类型。这对于调用者是重要的。当传递一个 _const_ 类型的对象到引用类型的形参上时，是希望这个对象是保持  
不变的，即为：希望形参的类型是 _const&_ 的。这也是为什么传递一个 _const_ 类型的对象到一个持有形参的类型为  
_T&_ 的模板上时是安全的：因为对象的 _constness_ 成为了 _T_ 的一部分。

在第三个例子中，尽管 _rx_ 的类型是引用，但是 _T_ 还是被推导为了 _non-reference_。这是因为 _rx_ 的 _reference-ness_ 在  
类型推导期间被忽略了。

这些例子全部展示的都是左值引用形参，但是右值引用形参也是一样的。当然，只有右值实参可以被传递到右值引  
用形参上，但是这个限制和类型推导是无关的。

如果将 _f_ 的形参的类型从 _T&_ 改为了 _const T&_ 的话，事情是会有所改变，但是不会改动很多。_cx_ 和 _rx_ 的 _constness_  
会继续保持，但是因为我们现在假设 _param_ 是 _reference-to-const_ 的了，所以对于 _const_ 不再需要去被推导做为 _T_  
的一部分了：  
```C++
  template<typename T>
  void f(const T& param);     // param is now a ref-to-const
  
  int x = 27;                 // as before
  const int cx = x;           // as before
  const int& rx = x;          // as before
  
  f(x);                       // T is int, param's type is const int&
  
  f(cx);                      // T is int, param's type is const int&
  
  f(rx);                      // T is int, param's type is const int&
```

和之前的一样，_rx_ 的 _reference-ness_ 在类型推导期间是被忽略的。

如果 _param_ 是指针或 _const_ 指针而不是引用的话，本质是还是一样的：
```C++
  template<typename T>
  void f(T* param);           // param is now a pointer

  int x = 27;                 // as before
  const int *px = &x;         // px is a ptr to x as a const int

  f(&x);                      // T is int, param's type is int*

  f(px);                      // T is const int,
                              // param's type is const int*
```

到目前为止，你可能发现你自己在打哈欠和打瞌睡，因为 _C++_ 的类型推导规则对于引用类型的形参和指针类型的  
形参来说是非常自然的。书面表达真的很无聊。每件事都很明显。都是你在类型推导系统中所希望的。

### 场景 2：_ParamType_ 是 _univeral reference_

对于持有 _univeral reference_ 类型的形参的模板来说就没有那么明显了。这样的形参被声明为像右值引用那样，即  
为：在持有一个类型形参 _T_ 的函数模板中，_univeral reference_ 的声明的类型为 _T&&_，但是当左值实参被传入时，  
会有不同的行为。完整的故事会在 [_Item 24_](./Chapter%205.md#item-24-区分通用引用和右值引用) 中陈述，这里只有大纲版本：  
* 如果 _expr_ 是一个左值的话，_T_ 和 _ParamType_ 都会被推导为左值引用。这就更不寻常了。首先，这是唯一一种  
在模板的类型推导中 _T_ 会被推导为引用的情景。其次，尽管 _ParamType_ 是使用右值引用的语法来声明的，但  
是所推导出的类型却是左值引用。
* 如果 _expr_ 是一个右值的话，那么 **普通的** 规则就会被应用，即为：[场景 1](./Chapter%201.md#场景-1-_ParamType_-是一个指针类型或者引用类型-但不能是-_univeral-reference_)。

例如：  
```C++
  template<typename T>
  void f(T&& param);          // param is now a universal reference

  int x = 27;                 // as before
  const int cx = x;           // as before
  const int& rx = x;          // as before

  f(x);                       // x is lvalue, so T is int&,
                              // param's type is also int&

  f(cx);                      // cx is lvalue, so T is const int&,
                              // param's type is also const int&

  f(rx);                      // rx is lvalue, so T is const int&,
                              // param's type is also const int&

  f(27);                      // 27 is rvalue, so T is int,
                              // param's type is therefore int&& 
```
[_Item 24_](./Chapter%205.md#item-24-区分通用引用和右值引用) 解释了为什么这些例子会按照它们做的那样来进行。关键点是 _univeral reference_ 类型的形参所对应的类型  
推导规则和左值引用类型或右值引用类型的形参所对应的类型推导规则是不相同的。尤其当 _univeral reference_ 在  
使用时，类型推导会区分左值实参和右值实参。这对于 _non-univeral reference_ 来说是永远不会发生的。

### 场景 3：_ParamType_ 既不是指针类型也不是引用类型

当 _ParamType_ 既不是指针类型也不是引用类型时，我们按照 _pass-by-value_ 进行处理:  
```C++
  template<typename T>
  void f(T param);            // param is now passed by value
```  
这意味着 _param_ 会被做为传入实参的副本，做为一个完全新的对象。_param_ 将会是新对象的事实推动了根据 _exp_ 推导出 _T_ 的规则：  
* 和以前一样，如果 _expr_ 的类型是引用，忽略引用部分。
* 在忽略了 _expr_ 的 _reference-ness_ 后，如果 _expr_ 是 _const_ 或者 _volatile_ 的话，也进行忽略。_volatile_ 对象是不  
常见的，_volatile_ 通常用于设备驱动实现中。更多的细节看 [_Item 40_](./Chapter%207.md#item-40-并发使用-_std::atomic_-特殊内存使用-_volatile_)。  

因此：  
```C++
  int x = 27;                 // as before
  const int cx = x;           // as before
  const int& rx = x;          // as before

  f(x);                       // T's and param's types are both int
  
  f(cx);                      // T's and param's types are again both int
  
  f(rx);                      // T's and param's types are still both int
```  

注意即使 _cx_ 和 _rx_ 都是 _const_ 的，但是 **_param_** 也不是 **_const_** 的。这是合理的。_param_ 是一个完全和 _cx_ 和 _rx_ 无关的  
对象，它是 _cx_ 或 _rx_ 的副本。_cx_ 和 _rx_ 不能被更改的事实并不能说明 _param_ 是否可以被更改。这也是为什么当推导  
_param_ 的类型时，_expr_ 的 _constness_ 和 _volatileness_ 会被忽略的原因，因为 _expr_ 不能被更改并不意味着它的副本也  
不能被更改。

_const_ 和 _volatile_ 只有对于 _by-value_ 的形参才能被忽略，记住这个是非常重要的。正如我们刚才已经看到的，对于  
_references-to-const_ 或 _pointers-to-const_ 形参，_expr_ 的 _constness_ 则会在类型推导期间一直保留。但是考虑这种  
场景，_expr_ 是指向 _const_ 对象的 _const_ 指针，并且 _expr_ 会被传递给 _by-value 的 _param_：  
```C++
  template<typename T>
  void f(T param);            // param is still passed by value
  
  const char* const ptr =     // ptr is const pointer to const object
  "Fun with pointers";
  
  f(ptr);                     // pass arg of type const char * const
```  

这里的 * 右边的 _const_ 声明了 _ptr_ 是 _const_ 的：_ptr_ 不能再指向一个其他的地址了，也不能指向
_null_ 了。而 * 左边的  
_const_ 说明了 _ptr_ 所指向的字符串是 _const_ 的，因此是不能被更改的。当 _ptr_ 被传递给 _f_ 时，组成指针的位被复制到  
_param_ 中。因此，_ptr_ 本身是 _pass-by-value_ 的。与 _by-value_ 的形参所对应的类型推导规则一样，_ptr_ 的 _constness_  
会被忽略，而所推导出的 _param_ 的类型是 _const char*_ 的，即为：一个指向 _const_ 字符串的可更改指针。_ptr_ 所指  
向的字符串的 _constness_ 在类型推导期间是被保留的，但是 _ptr_ 的 _constness_ 在拷贝 _ptr_ 去创建一个新的指针 _param_  
时是被忽略的。

### 数组实参

上面基本涵盖了主流的模板的类型推导，但是还有一小部分场景值得我们去关注。那就是数组类型和指针类型是不  
同的，尽管有时候它们俩似乎是可以进行交换的。这个错误的主要原因是在很多的上下文中，数组退化成为了一个  
指向首元素的指针。这种退化允许代码像下面的这样去编译：  
```C++
  const char name[] = "J. P. Briggs";             // name's type is
                                                  // const char[13]

  const char * ptrToName = name;                  // array decays to pointer
```
此处，_const char*_ 指针 _ptrToName_ 是被 _const char[13]_ 的 _name_ 所初始化的的。_const char*_ 和 _const char[13]_ 是不  
相同的，但是因为 _array-to-pointer_ 的退化规则，所以这个代码是可以编译的。

但是，如果是一个数组被传递给一个持有 _by-value_ 的形参的模板的话，那么会发生什么呢？  
```C++
  template<typename T>
  void f(T param);            // template with by-value parameter

  f(name);                    // what types are deduced for T and param?
```

我们从观察到的函数形参中不存在数组这个事实开始，是的，这种语法是合法的，  
```C++
  void myFunc(int param[]);
```  
但是数组声明是被当作为指针声明的，这意味着 "myFunc" 可以被等同地声明为下面这样：
```C++
  void myFunc(int* param);    // same function as above
```

数组形参和指针形参的等价性是一片在 _C++_ 基础之上的从 _C_ 根里长出的绿叶，这造成了一种错觉，那就是数组类  
型和指针类型是相同的。

因为数组形参声明是被当作为指针形参声明的，所以按值传递到模板函数的数组是被推导为指针类型的，这就意味  
着在模板 _f_ 的调用中，类型形参 _T_ 是被推导为 _const char*_ 的。  
```C++
  f(name);                    // name is array, but T deduced as const char*
```

但是现在来一个曲线球。尽管函数不能声明形参为真正的数组，但是可以声明形参为数组引用。所以如果我们更改模板 _f_ 为按引用来接收它的实参，  
```C++
  template<typename T>
  void f(T& param);           // template with by-reference parameter
```  
并传递一个数组给它的话，  
```C++
  f(name);                    // pass array to f
```  
_T_ 是实实在在的数组类型，这个类型包含着数组的大小，所以在这个例子中，_T_ 被推导为了 _const char [13]_，
那么 
 _f_ 的形参的类型就是 _const char (&)[13]_ 了，就是数组的引用。是的，这种语法看起来是有毒的，但了解这个将使你  
 在那些关心此事的少数人中获得巨大的加分。

有趣地是，声明数组引用的能力可以创建一个可以推导出数组所包含的元素数量的模板：  
```C++
  // return size of an array as a compile-time constant. (The
  // array parameter has no name, because we care only about
  // the number of elements it contains.)
  template<typename T, std::size_t N>                        // see info
  constexpr std::size_t arraySize(T (&)[N]) noexcept         // below on
  {                                                          // constexpr
      return N;                                              // and
  }                                                          // noexcept
```

正如 [_Item 15_](./Chapter%203.md#item-15-只要有可能-就使用-_constexpr_) 所解释的，声明这个函数为 _constexpr_ 是为了让它的结果在编译期间就有效。这样就使得可以声明一  
个数组，它的大小和另一个数组是相同的，另一个数组的大小是由花括号初始化表达式所计算的：  
```C++
  int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };     // keyVals has
                                                  // 7 elements

  int mappedVals[arraySize(keyVals)];             // so does
                                                  // mappedVals
```

当然，做为一名 _modern C++_ 开发者，你自然应该首选 _std::array_ 而不是内建的数组。  
```C++
  std::array<int, arraySize(keyVals)> mappedVals; // mappedVals'
                                                  // size is 7
```

至于 _arraySize_ 被声明为 _noexcept_，是为了帮助编译生成更好的代码。更多的细节，请查看 [_Item 14_](./Chapter%203.md#item-14-如果函数不会引发异常的话-就声明函数为-_noexcept_)。

### 函数实参

数组不是 _C++_ 中唯一一个可以退化为指针的东西。函数类型可以退化为函数指针，我们已经讨论过的关于数组的  
类型推导的所有事情都可以应用到函数的类型推导上，函数可以退化为函数指针。因此：  
```C++
  void someFunc(int, double);           // someFunc is a function;
                                        // type is void(int, double)

  template<typename T>
  void f1(T param);                     // in f1, param passed by value

  template<typename T>
  void f2(T& param);                    // in f2, param passed by ref

  f1(someFunc);                         // param deduced as ptr-to-func;
                                        // type is void (*)(int, double)

  f2(someFunc);                         // param deduced as ref-to-func;
                                        // type is void (&)(int, double)
```

这在实践中几乎没有不同，但是如果你正在了解数组指针退化的话，你也要了解函数指针退化。

所以就是这样了：模板的类型推导所对应的 _auto-related_ 规则。在开始的时候，我就说过模板的类型推导是非常直  
观的，对于大部分来说是的。当推导 _univeral references_ 所对应的类型时，给予左值的特殊对待把水搅浑了点，而  
数组和函数所对应的 _decay-to-pointer_ 规则把水搅得更浑了点。有时候你想要只是抓住你的编译器，让它告诉你正  
在推导的类型。当发生这样的情况时，转到 [_Item 4_](./Chapter%201.md#item-4-理解如何去可视化推导出的类型)。

### 需要记住的规则：

* 在模板的类型推导期间，引用实参是被当作是非引用的，即为：实参的 _reference-ness_ 是被忽略的。
* 当推导 _universal reference_ 类型的形参的类型时，左值实参是需要特殊对待的。
* 当推导 _by-value_ 的形参的类型时，_const_ 和 _volatile_ 的实参是被当作是 _non-const_ 和 _non-volatile_ 的。 
* 在模板的类型推导期间，数组名或函数名的实参是会退化为指针的，除非它们是被用于初始化引用的。

### 译者总结  
```C++
  template<typename T>
  void f(ParamType param);

  f(expr);                    // call f with some expression
```
* 当 _ParamType_ 是指针类型或者引用类型，但不能是 _universal reference_ 时，是根据实参的类型和 _ParamType_  
的特征来推导 _T_ 的。
* 当 _ParamType_ 既不是指针类型也不是引用类型时，剥离了 _constness_ 和 "volatile-ness" 的实参的类型就是 _T_。

## Item 2 理解 _auto_ 的类型推导