- [_Chapter_ 1 类型推导](#chapter-1-类型推导)
  - [_Item 1_ 理解模板的类型推导](#item-1-理解模板的类型推导)
    - [场景 1：_ParamType_ 是指针类型或者引用类型，但不是 _univeral reference_](#场景-1paramtype-是指针类型或者引用类型但不是-univeral-reference)
    - [场景 2：_ParamType_ 是 _univeral reference_](#场景-2paramtype-是-univeral-reference)
    - [场景 3：_ParamType_ 既不是指针类型也不是引用类型](#场景-3paramtype-既不是指针类型也不是引用类型)
    - [数组实参](#数组实参)
    - [函数实参](#函数实参)
    - [需要记住的规则](#需要记住的规则)
    - [译者总结](#译者总结)
  - [_Item 2_ 理解 _auto_ 的类型推导](#item-2-理解-auto-的类型推导)
    - [需要记住的规则](#需要记住的规则-1)
    - [译者总结](#译者总结-1)
  - [_Item 3_ 理解 _decltype_](#item-3-理解-decltype)
    - [需要记住的规则](#需要记住的规则-2)
  - [_Item 4_ 了解如何查看所推导的类型](#item-4-了解如何查看所推导的类型)
    - [_IDE_ 编辑器](#ide-编辑器)
    - [编译器诊断](#编译器诊断)
    - [运行时输出](#运行时输出)
    - [需要记住的规则](#需要记住的规则-3)

# _Chapter_ 1 类型推导

_C++98_ 只有一组类型推导规则：函数模板所对应的类型推导。_C++11_ 稍微修改了规则，新增了两组规则：_auto_ 和 _decltype_。_C++14_ 扩展了 _auto_ 和 _decltype_ 的使用范围。类型推导的日益广泛的应用让你从明显的或多余的类型拼写的专横中获得了解脱。类型推导使得 _C++_ 软件更有适应性了，因为在源码的一个地方上改变类型会自动通过类型推导来传播到其他地方。然而类型推导会使代码变的难于理解，因为编译器推导的类型，可能不会像你想的那样明显。

只有深刻理解了类型推导是如何工作的，才能在 _modern C++_ 中进行高效编程。有很多类型推导的场景：调用函数模板时、大多数 _auto_ 出现的情景中、_decltype_ 表达式中和 _C++14_ 中的 _decltype(auto)_ 中。

这章提供所提供的类型推导的内容是每一个 _C++_ 开发者都需要的。这些内容解释了模板的类型推导是如何工作的、_auto_ 是如何在模板的类型推导之上构建的，以及 _decltype_ 是如何走它自己的路的。本章甚至解释了如何强制编译器使类型推导的结果变得可见，这能够确保编译器正在推导你想让它们去推导的类型。

## _Item 1_ 理解模板的类型推导

当复杂系统的用户不知道它是如何工作却对它的工作感到满意时，就可以表明这个系统的设计是非常出色的。按照这样说，_C++_ 的模板的类型推导也是非常成功的。成千上万的程序员已经传递实参到模板函数中，并且得到了完全正确的结果，对于这些函数所使用的类型是如何被推导的，很多程序员除了模糊的描述外，很难再给出更好的描述。

如果你也不能给出更好的描述的话，那么我有好消息和坏消息。好消息是模板的类型推导是 _auto_ 的基础，而 _auto_ 则是 _modern C++_ 中最吸引人的特性之一。如果你对 _C++98_ 的模板的类型推导感到满意，那么你也准备好了对于 _C++11_ 的 _auto_ 的类型推导感到满意。坏消息是当模板的类型推导规则被应用到 _auto_ 上时比被应用到模板上时要难理解一点。因此，真正地理解模板的类型推导是非常重要的，因为 _auto_ 是在模板的类型推导之上而构建的。本 [_Item 1_](#item-1-理解模板的类型推导) 覆盖了你需要知道的所有。 

如果你愿意忽略一些伪代码，我们可以考虑一个像下面那样的函数模板：  
```C++
  template<typename T>
  void f(ParamType param);
```  
一个调用可以看起来像是这样：  
```C++
  f(expr);                    // call f with some expression
```

在编译期间，编译器使用 _expr_ 去推导两个类型：一个是 _T_ 类型，一个是 _ParamType_ 类型。这两个类型经常是不相同的，因为 _ParamType_ 常常会包含有装饰，比如：_const_ 或 _&_。例如：如果模板声明成下面这样：  
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
* _ParamType_ 是一个指针类型或者引用类型，但不是 _universal reference_，_universal reference_ 会在 [_Item 24_](Chapter%205.md#item-24-区分-universal-reference-和右值引用) 中进行描述。现在你只需要知道是 _universal reference_ 是存在的，而且和左值引用或右值引用是不相同的。
* _ParamType_ 是一个 _universal reference_。
* _ParamType_ 既不是指针类型也不是引用类型。

因此我们有三种类型推导的情况来检查。每个都基于我们的模板的通用格式，然后调用它：  
```C++
  template<typename T>
  void f(ParamType param);
  
  f(expr);                    // deduce T and ParamType from expr
```

### 场景 1：_ParamType_ 是指针类型或者引用类型，但不是 _univeral reference_

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
_param_ 对应的推导的类型和各种调用中的 _T_ 是像下面的的这样：  
```C++
  f(x);                       // T is int, param's type is int&
  
  f(cx);                      // T is const int,
                              // param's type is const int&
  
  f(rx);                      // T is const int,
                              // param's type is const int&
```

在第二个和第三个调用中，因为 _cx_ 和 _rx_ 是 _const_ 类型的，所以 _T_ 会被推导为 _const int_，因此产生了 _const int&_ 的形参类型。这对于调用方是重要的。当传递一个 _const_ 类型的对象到引用类型的形参上时，希望的是这个对象是保持不变的，即为：希望形参的类型是 _const&_ 的。这也是为什么传递一个 _const_ 类型的对象到一个持有形参的类型为 _T&_ 的模板上时是安全的：因为对象的 _constness_ 成为了 _T_ 的一部分。

在第三个例子中，尽管 _rx_ 的类型是引用，但是 _T_ 还是被推导为了 _non-reference_。这是因为 _rx_ 的 _reference-ness_ 在类型推导期间被忽略了。

这些例子全部展示的都是左值引用形参，但是右值引用形参也是一样的。当然，只有右值实参可以被传递到右值引用形参上，但是这个限制和类型推导是无关的。

如果将 _f_ 的形参的类型从 _T&_ 改为了 _const T&_ 的话，事情是会有所改变，但是不会改动很多。_cx_ 和 _rx_ 的 _constness_ 会继续保持，但是因为我们现在假设 _param_ 是 _reference-to-const_ 的了，所以对于 _const_ 不再需要去被推导做为 _T_ 的一部分了：  
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

到目前为止，你可能发现你自己在打哈欠和打瞌睡，因为 _C++_ 的类型推导规则对于引用类型的形参和指针类型的形参来说是非常自然的。书面表达真的很无聊。每件事都很明显。都是你在类型推导系统中所希望的。

### 场景 2：_ParamType_ 是 _univeral reference_

对于持有 _univeral reference_ 类型的形参的模板来说就没有那么明显了。这样的形参被声明为像右值引用那样，即为：在持有一个类型形参 _T_ 的函数模板中，_univeral reference_ 的声明的类型为 _T&&_，但是当左值实参被传入时，会有不同的行为。完整的故事会在 [_Item 24_](Chapter%205.md#item-24-区分-universal-reference-和右值引用) 中陈述，这里只有大纲版本：  
* 如果 _expr_ 是一个左值的话，_T_ 和 _ParamType_ 都会被推导为左值引用。这就更不寻常了。首先，这是唯一一种在模板的类型推导中 _T_ 会被推导为引用的情景。其次，尽管 _ParamType_ 是使用右值引用的语法来声明的，但是所推导出的类型却是左值引用。
* 如果 _expr_ 是一个右值的话，那么 **普通的** 规则就会被应用，即为：[场景 1](#场景-1paramtype-是指针类型或者引用类型但不是-univeral-reference)。

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
[_Item 24_](Chapter%205.md#item-24-区分-universal-reference-和右值引用) 解释了为什么这些例子会按照它们做的那样来进行。关键点是 _univeral reference_ 类型的形参所对应的类型推导规则和左值引用类型或右值引用类型的形参所对应的类型推导规则是不相同的。尤其当 _univeral reference_ 在使用时，类型推导会区分左值实参和右值实参。这对于 _non-univeral reference_ 来说是永远不会发生的。

### 场景 3：_ParamType_ 既不是指针类型也不是引用类型

当 _ParamType_ 既不是指针类型也不是引用类型时，我们按照 _pass-by-value_ 进行处理:  
```C++
  template<typename T>
  void f(T param);            // param is now passed by value
```  
这意味着 _param_ 会被做为传入实参的副本，做为一个完全新的对象。_param_ 将会是新对象的事实推动了根据 _exp_ 推导出 _T_ 的规则：  
* 和以前一样，如果 _expr_ 的类型是引用，忽略引用部分。
* 在忽略了 _expr_ 的 _reference-ness_ 后，如果 _expr_ 是 _const_ 或者 _volatile_ 的话，也进行忽略。_volatile_ 对象是不常见的，_volatile_ 通常用于设备驱动实现中。更多的细节看 [_Item 40_](Chapter%207.md#item-40-并发使用-stdatomic-特殊内存使用-volatile)。  

因此：  
```C++
  int x = 27;                 // as before
  const int cx = x;           // as before
  const int& rx = x;          // as before

  f(x);                       // T's and param's types are both int
  
  f(cx);                      // T's and param's types are again both int
  
  f(rx);                      // T's and param's types are still both int
```  

注意：即使 _cx_ 和 _rx_ 都是 _const_ 的，但是 **_param_** 也不是 **_const_** 的。这是合理的。_param_ 是一个完全和 _cx_ 和 _rx_ 无关的对象，它是 _cx_ 或 _rx_ 的副本。_cx_ 和 _rx_ 不能被更改并不能说明 _param_ 不可以被更改。这也是为什么当推导 _param_ 的类型时，_expr_ 的 _constness_ 和 _volatileness_ 会被忽略的原因，因为 _expr_ 不能被更改并不意味着它的副本也不能被更改。

_const_ 和 _volatile_ 只有对于 _by-value_ 的形参才能被忽略，记住这个是非常重要的。正如我们刚才已经看到的，对于 _references-to-const_ 或者 _pointers-to-const_ 形参，_expr_ 的 _constness_ 则会在类型推导期间一直保留。但是考虑这种场景：_expr_ 是指向 _const_ 对象的 _const_ 指针，并且它会被传递给 _by-value_ 的 _param_：  
```C++
  template<typename T>
  void f(T param);            // param is still passed by value
  
  const char* const ptr =     // ptr is const pointer to const object
  "Fun with pointers";
  
  f(ptr);                     // pass arg of type const char * const
```  

这里的 * 右边的 _const_ 声明了 _ptr_ 是 _const_ 的：_ptr_ 不能再指向一个其他的地址了，也不能指向 _null_ 了。而 * 左边的 _const_ 则说明了 _ptr_ 所指向的字符串是 _const_ 的，因此是不能被更改的。当 _ptr_ 被传递给 _f_ 时，这个指针会按位复制给 _param_ 。因此，_ptr_ 本身是 _pass-by-value_ 的。与 _by-value_ 的形参所对应的类型推导规则一样，_ptr_ 的 _constness_ 会被忽略掉，所推导出的 _param_ 的类型就是 _const char*_ 了，即为：一个指向 _const_ 字符串的可更改指针。_ptr_ 所指向的字符串的 _constness_ 在类型推导期间是被保留的，而 _ptr_ 的 _constness_ 在拷贝 _ptr_ 去创建一个新的指针 _param_ 时是被忽略掉的。

### 数组实参

上面基本涵盖了主流的模板的类型推导，但是还有一小部分场景值得我们去关注。那就是数组类型和指针类型是不同的，尽管有时候它们俩似乎是可以进行交换的。这个错误的主要原因是在很多的上下文中，数组退化成为了一个指向首元素的指针。这种退化允许代码像下面的这样去编译：  
```C++
  const char name[] = "J. P. Briggs";             // name's type is
                                                  // const char[13]

  const char * ptrToName = name;                  // array decays to pointer
```
此处，_const char*_ 指针 _ptrToName_ 是被 _const char[13]_ 的 _name_ 所初始化的的。_const char*_ 和 _const char[13]_ 是不相同的，但是因为存在有 _array-to-pointer_ 的退化规则，所以这个代码是可以编译的。

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
但是数组声明是被当作为指针声明的，这意味着 _myFunc_ 可以被等同地声明为下面这样：
```C++
  void myFunc(int* param);    // same function as above
```

数组形参和指针形参的等价性是一片在 _C++_ 基础之上的从 _C_ 根里长出的绿叶，这造成了一种错觉，那就是数组类型和指针类型是相同的。

因为数组形参声明是被当作为指针形参声明的，所以按 _by-value_ 的形式传递到模板函数的数组是被推导为指针类型的，这就意味着在模板 _f_ 的调用中，类型形参 _T_ 是被推导为 _const char*_ 的。  
```C++
  f(name);                    // name is array, but T deduced as const char*
```

但是现在来一个曲线球。尽管函数不能声明形参为真正的数组，但是可以声明形参为数组引用。所以如果我们更改模板 _f_ 为按 _by-reference_ 的形式来接收它的实参，  
```C++
  template<typename T>
  void f(T& param);           // template with by-reference parameter
```  
并传递一个数组给它的话，  
```C++
  f(name);                    // pass array to f
```  
那么 _T_ 就是数组类型了，这个类型包含着数组的大小，所以在这个例子中，_T_ 就被推导成为了 _const char [13]_，相应地，_f_ 的形参的类型就是 _const char (&)[13]_ 了，就是数组的引用。是的，这种语法看起来是有毒的，但了解这个将使你在那些关心此事的少数人中获得巨大的加分。

有趣地是，声明数组引用的能力可以创建一个可以推导出数组所包含的元素数量的模板：  
```C++
  // return size of an array as a compile-time constant. (The
  // array parameter has no name, because we care only about
  // the number of elements it contains.)
  template<typename T, std::size_t N>                       // see info
  constexpr std::size_t arraySize(T (&)[N]) noexcept        // below on
  {                                                         // constexpr
    return N;                                               // and
  }                                                         // noexcept
```

正如 [_Item 15_](Chapter%203.md#item-15-只要有可能就使用-constexpr) 所解释的，声明这个函数为 _constexpr_ 是为了让它的结果在编译期间就有效。这样就使得可以声明一个数组，它的大小和另一个数组是相同的，另一个数组的大小是由花 _braced initializer_ 所计算的：  
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

至于 _arraySize_ 被声明为 _noexcept_，是为了帮助编译生成更好的代码。更多的细节，请查看 [_Item 14_](Chapter%203.md#item-14-当函数不会抛出异常时声明函数为-noexcept)。

### 函数实参

数组不是 _C++_ 中唯一一个可以退化为指针的东西。函数类型可以退化为函数指针，我们已经讨论过的关于数组的类型推导的所有事情都可以应用到函数的类型推导上，函数可以退化为函数指针。因此：  
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

所以就是这样了：模板的类型推导所对应的 _auto-related_ 规则。在开始的时候，我就说过模板的类型推导是非常直观的，对于大部分来说是的。当推导 _univeral references_ 所对应的类型时，给予左值的特殊对待把水搅浑了点，而数组和函数所对应的 _decay-to-pointer_ 规则把水搅得更浑了点。有时候你想要只是抓住你的编译器，让它告诉你正在推导的类型。当发生这样的情况时，转到 [_Item 4_](#item-4-了解如何查看所推导的类型)。

### 需要记住的规则

* 在模板的类型推导期间，引用实参是被当作是 _non-reference_ 的，即为：实参的 _reference-ness_ 是被忽略的。
* 当推导 _universal reference_ 类型的形参的类型时，左值实参是需要特殊对待的。
* 当推导 _by-value_ 的形参的类型时，_const_ 和 _volatile_ 的实参是被当作是 _non-const_ 和 _non-volatile_ 的。 
* 在模板的类型推导期间，数组名或函数名的实参是会退化为指针的，除非它们是被用于初始化引用的。

### 译者总结  

```C++
  template<typename T>
  void f(ParamType param);

  f(expr);                    // call f with some expression
```
* 当 _ParamType_ 是指针类型或者引用类型，但不能是 _universal reference_ 时，是根据实参的类型和 _ParamType_ 的特征来推导 _T_ 的。
* 当 _ParamType_ 既不是指针类型也不是引用类型时，剥离了 _constness_ 和 _volatile-ness_ 的实参的类型就是 _T_。

## _Item 2_ 理解 _auto_ 的类型推导

如果你已经阅读了 [_Item 1_](#item-1-理解模板的类型推导) 的模板的类型推导的话，那么你已经知道了几乎关于 _auto_ 的类型推导的所有事情，因为除了一个稀奇的例外以外，_auto_ 的类型推导就是模板的类型推导了。但是是如何可以这样的呢？模板的类型推导涉及到了模板函数和形参，但是 _auto_ 却不涉及到它们。

的确是这样的，但是并不重要。模板的类型推导和 _auto_ 的类型推导是直接映射的。是逐字地从一个到另一个的算法交换。

在 [_Item 1_](#item-1-理解模板的类型推导) 中，模板的类型推导使用下面通用的函数模板来解释：  
```C++
  template<typename T>
  void f(ParamType param);
```  
然后这是通用的调用：  
```C++
  f(expr);                    // call f with some expression
```

在 _f_ 的调用中，编译器使用 _expr_ 去推导 _T_ 和 _ParamType_ 的类型。

当使用 _auto_ 来声明一个变量时，_auto_ 扮演的是模板的类型推导中的 _T_ 的角色，而变量的 _type specifier_ 充当的是 _ParamType_ 的作用。展示比描述要简单，所以考虑下面这个例子：  
```C++
  auto x = 27;
```  
> 译者注：变量声明中的 = 右侧的表达式可以当作是模板的类型推导中的实参的角色，这样就一一对应了。

此处，_x_ 的 _type specifier_ 就只有 _auto_ 它自己。另一方面，在下面这个声明中：  
```C++
  const auto cx = x;
```
_type specifier_ 是 _const auto_，然后这里，  
```C++
  const auto& rx = x;
```
_type specifier_ 是 _const auto&_。为了推导上面例子中的 _x_、_cx_ 和 _rx_，编译器的行为就好像每个声明都有一个模板，并且使用相应的 _initializer_ 来调用这个模板：  
```C
  template<typename T>                  // conceptual template for
  void func_for_x(T param);             // deducing x's type

  func_for_x(27);                       // conceptual call: param's
                                        // deduced type is x's type

  template<typename T>                  // conceptual template for
  void func_for_cx(const T param);      // deducing cx's type

  func_for_cx(x);                       // conceptual call: param's
                                        // deduced type is cx's type

  template<typename T>                  // conceptual template for
  void func_for_rx(const T& param);     // deducing rx's type

  func_for_rx(x);                       // conceptual call: param's
                                        // deduced type is rx's type
```

正如我说过的，除了有一个例外以外，我们马上就要讨论它，_auto_ 的类型推导和模板的类型推导都是相同的。

根据通用函数模板中的 _param_ 所对应的 _type specifier_ 也就是 _ParamType_ 的特征，[_Item 1_](#item-1-理解模板的类型推导) 将模板的类型推导分为了三个场景。在使用 _auto_ 的变量声明中，_type specifier_ 代替了 _ParamType_，所以也对应有三种场景：  
* 场景 1：_type specifier_ 是指针类型或引用类型，但不是 _universal reference_。
* 场景 2：_type specifier_ 是 _universal reference_。
* 场景 3：_type specifier_ 既不是指针类型也不是引用类型。
  
我们已经看到了 _场景 1_ 和 _场景 3_ 所对应的例子：  
```C++
  auto x = 27;                // case 3 (x is neither ptr nor reference)

  const auto cx = x;          // case 3 (cx isn't either)
  
  const auto& rx = x;         // case 1 (rx is a non-universal ref.)
```  
_场景 2_ 也如你希望地那样工作：  
```C++
  auto&& uref1 = x;           // x is int and lvalue,
                              // so uref1's type is int&

  auto&& uref2 = cx;          // cx is const int and lvalue,
                              // so uref2's type is const int&

  auto&& uref3 = 27;          // 27 is int and rvalue,
                              // so uref3's type is int&&
```

[_Item 1_](#item-1-理解模板的类型推导) 是以数组名和函数名是如何退化为 _non-reference type specifier_ 为结束的。在 _auto_ 的类型推导中也是这样的：  
```C++
  const char name[] =                   // name's type is const char[13]
    "R. N. Briggs";

  auto arr1 = name;                     // arr1's type is const char*
  
  auto& arr2 = name;                    // arr2's type is
                                        // const char (&)[13]
  
  void someFunc(int, double);           // someFunc is a function;
                                        // type is void(int, double)
  
  auto func1 = someFunc;                // func1's type is
                                        // void (*)(int, double)
  
  auto& func2 = someFunc;               // func2's type is
                                        // void (&)(int, double)
```  
正如你可以看到的，_auto_ 的类型推导像模板的类型推导一样地工作。它们本质上是一个硬币的两面。

除了它们有一点不同。如果你想要使用初始值 _27_ 来声明一个 _int_ 的话，那么 _C++98_ 给你两个语法做选择，我们先从这个观察开始：  
```C++
  int x1 = 27;
  int x2(27);
```  
_C++11_ 通过支持 _uniform initialization_ 增加了下面这样的语法：  
```C++
int x3 = { 27 };
int x4{ 27 };
```

共有 _4_ 种语法，但是只有一个结果：一个有着 _27_ 的 _int_。

正如 [_Item 5_](Chapter%202.md#item-5-首选-auto-而不是显式类型声明) 所解释的，使用 _auto_ 来代替固有的类型来进行变量声明是有优势的，所以在上面的变量声明最好使用 _auto_ 来代替 _int_。直接的文本替换生成了以下代码：  
```C++
  auto x1 = 27;
  auto x2(27);
  auto x3 = { 27 };
  auto x4{ 27 };
```  
这些声明全部都能编译，但是有着不同的含义。首先前两个语句实际上声明了一个初始值是 _27_ 的 _int_ 类型的变量。然而后两个语句却是声明了一个包含有一个值是 _27_ 的元素的 _std::initializer_list<int>_ 类型的变量。  
```C++
  auto x1 = 27;               // type is int, value is 27
  
  auto x2(27);                // ditto
  
  auto x3 = { 27 };           // type is std::initializer_list<int>,
                              // value is { 27 }
  
  auto x4{ 27 };              // ditto
```

这是由于 _auto_ 所对应的一条特殊类型推导规则所导致的。当 _auto-declared_ 的变量的 _initializer_ 是使用 _{}_ 进行包围的时，所推导出的类型是 _std::initializer_list_。如果这个类型不能被推导的话，比如：在 _braced initializer_ 中的值是不同的类型的话，那么这个代码将会被拒绝。  
```C++
   auto x5 = { 1, 2, 3.0 };   // error! can't deduce T for
                              // std::initializer_list<T>  
```

正如注释所指明的，这种场景中的类型推导会失败，重要的是要认识到此处实际上发生了两种类型推导。第一种是来自于 _auto_ 的使用：_x5_ 的类型是必须要被推导出来的。又因为 _x5_ 的 _initializer_ 是在 _{}_ 中的，所以 _x5_ 必须被推导为 _std::initialier_list_。但是 _std::initialier_list_ 是一个模板。实例化是一些类型 _T_ 的 _std::initialier_list&lt;T&gt;_，而这也意味着 _T_ 的类型也是必须要被推导出来的。这些推导会在此处发生的第二种类型推导的范围内失败：模板的类型推导会失败。在这个例子中，推导会失败，因为在  _braced initializer_ 中的值不是只有一种类型。

_braced initializer_ 的处理是唯一一个 _auto_ 的类型推导和模板的类型推导不同的地方。当 _auto-declared_ 变量是使用 _braced initializer_ 来初始化时，所推导出的类型是 _std::initialier_list_。但是如果相应的模板被传递了相同的 _initializer_ 的话，那么类型推导会失败，而且代码会被拒绝：
```C++
  auto x = { 11, 23, 9 };     // x's type is
                              // std::initializer_list<int>

  template<typename T>        // template with parameter
  void f(T param);            // declaration equivalent to
                              // x's declaration

  f({ 11, 23, 9 });           // error! can't deduce type for T
```  
然而，如果你在模板中为一些未知的 _T_ 指定了 _param_ 是 _std::initialier_list&lt;T&gt;_ 的话，模板的类型推导将会推导出 _T_ 是什么：  
```C++
  template<typename T>
  void f(std::initializer_list<T> initList);

  f({ 11, 23, 9 });                               // T deduced as int, and initList's
                                                  // type is std::initializer_list<int>
```  
所以，_auto_ 的类型推导和模板的类型推导只有一个真正的不同，那就是 _auto_ 会 **_假设_**  _braced initializer_ 表示的是 _std::initialier_list_，但是模板的类型推导则不会。

你可能会好奇对于 _braced initializer_ 为什么 _auto_ 的类型推导会有一个特殊的规则，但是模板的类型推导则没有呢？我自己也好奇。我没能找到一个令人信服的解释。但是规则就是规则，这意味着你必须记住：如果你使用 _auto_ 声明变量，并且是使用 _braced initializer_ 进行的初始化的话，那么所推导出的类型将是 _std::initialier_list_。如果你接受了 _uniform initializer_ 的哲学体系的话，也就是使用 _{}_ 来将初始化值括起来的话，那么你要特别注意这个规则。在 _C++11_ 编程中的一个经典错误是当你想要声明其他东西时，却不小心地声明了 _std::initialier_list_ 类型的变量。这个陷阱使得一些开发者只有在必要时才会在 _initializer_ 上加上 _{}_。什么时候才是必要会在 [_Item 7_](Chapter%203.md#item-7-创建对象时区分--和) 中进行讨论。

对于 _C++11_ 来说，这是一个完整的故事，而对于 _C++14_，故事还在继续。_C++14_ 允许 _auto_ 去指明一个函数的返回值类型应该被推导，然后 _C++14_ 的 _lambda_ 可以在形参的声明中使用 _auto_ 。然而，这些 _auto_ 的用法利用的是模板的类型推导，而不是 _auto_ 的类型推导。所以，返回类型为 _auto_ 的函数返回 _braced initializer_ 是不能通过编译的：  
```C++
  auto createInitList()
  {
    return { 1, 2, 3 };       // error: can't deduce type
  }                           // for { 1, 2, 3 }
```
> 译者注：脑残规则，后续不修复的话，更是智障。

当 _auto_ 被用于 _C++14_ 的 _lambda_ 的形参类型规范中时，也是这样的。
```C++
  std::vector<int> v;
  …

  auto resetV =
    [&v](const auto& newValue) { v = newValue; };           // C++14
  
  …
  
  resetV({ 1, 2, 3 });                                      // error! can't deduce type
                                                            // for { 1, 2, 3 }
```  
> 译者注：脑残规则，后续不修复的话，更是智障。

### 需要记住的规则

* _auto_ 的类型推导和模板的类型推导一般情况下都是相同的，只是 _auto_ 的类型推导会假设 _braced initializer_ 表  
示的是 _std::initialier_list_，而模板的类型推导则不会。  
* 在函数的返回值或者 _lambda_ 的形参中，_auto_ 意味着模板的类型推导，而不是 _auto_ 的类型推导。
> 译者注：脑残规则，后续不修复的话，更是智障。

### 译者总结

* _auto_ 扮演的是模板的类型推导中的 _T_ 的角色，而变量的 _type specifier_ 充当的是 _ParamType_ 的作用，变量声明中的 = 右侧的表达式可以当作是模板的类型推导中的实参的角色。  
```C++
  template<typename T>
  void f(T& param);

  
  int x = 27;
  const int cx = x;

  auto &rx = cx;              // auto == T, type specifier == T&, rx == param, cx == argument                        
                              // 'auto &rx = cx' is mapped with 'f(cx)'. T is const int 
                              // and ParamType is 'const int&', so auto is const int
                              // and rx's type is const int&
```

## _Item 3_ 理解 _decltype_

_decltype_ 是一个古怪的产物。给定一个名字或表达式，它会告诉你这个名字或表达式的类型。一般地情况下，它所告诉你的正是你所期待的。然而，偶尔 _decltype_ 所提供的结果会让你抓头，会让你转向参考手册或在线 _Q&A_ 网站去寻找启示。

我们从平常的例子开始，就是从没有惊喜的例子开始。与模板的类型推导和 _auto_ 的类型推导期间所发生的事情不同，见 [_Item 1_](#item-1-理解模板的类型推导)，_decltype_ 一般会返回一所给定的名字或表达式的类型：  
```C++
  const int i = 0;                                // decltype(i) is const int

  bool f(const Widget& w);                        // decltype(w) is const Widget&
                                                  // decltype(f) is bool(const Widget&)
  struct Point {
    int x, y;                                     // decltype(Point::x) is int
  };                                              // decltype(Point::y) is int

  Widget w;                                       // decltype(w) is Widget
  if (f(w)) …                                     // decltype(f(w)) is bool

  template<typename T>                            // simplified version of std::vector
  class vector {
  public:
    …
    T& operator[](std::size_t index);
    …
  };

  vector<int> v;                                  // decltype(v) is vector<int>
  …
  if (v[0] == 0) …                                // decltype(v[0]) is int&
```  
看？没有惊喜。

在 _C++11_ 中，_decltype_ 的主要用法可能是用来声明函数的返回类型依赖于它的形参类型的函数模板。例如：假定我们想要写一个这样的函数，这个函数持有一个支持通过 _[]_ 和索引值来进行索引的容器，并会在返回索引操作结果前进行用户验证。函数的返回类型应该和索引操作所返回的类型是一样的。 

类型 _T_ 的对象所对应的容器的 _operator[]_ 一般返回的是 _T&_。例如：_std::deque_ 就是这种场景，_std::vector_ 的大部分场景也是这样。然而，对于 _std::vector&lt;bool&gt;_ 来说，_operator[]_ 返回的不是 _bool&_。相反地是，返回的是一个全新的对象。这种情景的原因和方式会在 [_Item 6_](Chapter%202.md#item-6-当-auto-推导出的类型是-undesired-时使用-the-explicitly-typed-initializer-idiom) 中进行探讨，现在重要的是容器的 _operator[]_ 所返回的类型是依赖于这个容器本身的。
```C++
  template<typename Container, typename Index>    // works, but
  auto authAndAccess(Container& c, Index i)       // requires
  -> decltype(c[i])                               // refinement
  {
    authenticateUser();
    return c[i];
  }
```

函数名字之前的 _auto_ 是和类型推导无关的。更准确地说，这表明 _C++11_ 的 _trailing return type_ 语法正在被使用，即为：函数的返回类型将会在形参列表 _-&gt;_ 后进行声明。_trailing return type_ 是有优势的，函数的形参可以被用到返回类型的规范中。例如：在 _authAndAccess_ 中，我们使用 _c_ 和 _i_ 来指明返回类型。如果我们按照传统的方式将返回类型放在函数之前的话，那么 _c_ 和 _i_ 将是无效的，因为还没有声明它们。

使用这个声明，_authAndAccess_ 返回传入的容器的 _operator[]_ 所返回的类型，完全符合我们的预期。

_C++11_ 只允许那些有着 _single-statement_ 的 _lambda_ 的返回类型可以被推导，而 _C++14_ 扩展到了全部的 _lambda_ 和函数中，包括那些有着 _multiple-statement_ 的 _lambda_ 和函数。在 _authAndAccess_ 的场景中，这意味着：在 _C++14_ 中，我们可以忽略 _trailing return type_，而只留下前置的 _auto_。使用这种声明的格式，_auto_ 意味着类型推导将会发生。特别是意味着编译器将会根据函数的实现来产生函数的返回类型：  
```C++
  template<typename Container, typename Index>    // C++14;
  auto authAndAccess(Container& c, Index i)       // not quite
  {                                               // correct
    authenticateUser();
    return c[i];                                  // return type deduced from c[i]  
  }
```  
[_Item 2_](#_item-2_-%E7%90%86%E8%A7%A3-_auto_-%E7%9A%84%E7%B1%BB%E5%9E%8B%E6%8E%A8%E5%AF%BC) 解释了：对于使用了 _auto_ 返回类型规范的函数，译器会利用模板的类型推导。在这个场景中是有问题的。正如我们已经讨论过的，对于大多数容器来说，_operator[]_ 返回的是 _T&_，而且 [_Item 1_](#item-1-理解模板的类型推导) 解释了：在模板的类型推导期间，初始化表达式的 _reference-ness_ 是被忽略的。思考这个客户的含义：  
```C++
  std::deque<int> d;
  …
  authAndAccess(d, 5) = 10;   // authenticate user, return d[5],
                              // then assign 10 to it;
                              // this won't compile!
```
此处，_d[5]_ 返回 _int&_，_authAndAccess_ 的 _auto_ 返回类型是会剥离掉引用的，因此产生的返回类型是 _int_。做为函数的返回值的 _int_ 是一个右值，而上面的代码试图分配 _10_ 到一个右值 _int_ 上。这在 _C++_ 中是被禁止的，所以代码不会通过编译。

为了让 _authAndAccess_ 按照我们想的那样去工作，对于它的返回类型，我们需要使用 _decltype_ 的类型推导，即为：_authAndAccess_ 应该返回和表达式 _c[i]_ 的类型是相同的类型。_Ｃ++_ 的维护者预想到了在一些类型是会被推测的场景中会需要使用 _decltype_ 的类型推导规则，所以通过 _decltype(auto)_ 来让这些变成可能。最初看来是矛盾的 _decltype_ 和 _auto_ 实际上却是非常合理的：_auto_ 指明了类型会被推导，而 _decltype_ 则说明了 _decltype_ 的规则应该在推导期间被使用。因此我们可以像下面这样写 _authAndAccess_：  
```C++
  template<typename Container, typename Index>    // C++14; works,
  decltype(auto)                                  // but still
  authAndAccess(Container& c, Index i)            // requires
  {                                               // refinement
    authenticateUser();
    return c[i];
  }
```  
现在 _authAndAccess_ 将会真正地返回 _c[i]_ 所返回的。特别是，对于 _c[i]_ 是返回 _T&_ 的常见场景，_authAndAccess_ 也将会返回 _T&_，而在 _c[i]_ 是返回对象的不常见场景中，_authAndAccess_ 也将会返回一个对象。

_decltype(auto)_ 的用法不限于只用在函数的返回类型上。当你想要应用 _decltype_ 的类型推导规则到初始化表达式时，声明变量也是方便的:  
```C++
  Widget w;

  const Widget& cw = w;
  
  auto myWidget1 = cw;                  // auto type deduction:
                                        // myWidget1's type is Widget

  decltype(auto) myWidget2 = cw;        // decltype type deduction:
                                        // myWidget2's type is
                                        // const Widget&
```

但是两件事正在困扰你，我知道。一件事是我提到的但是还没有描述的 _authAndAccess_ 的改进。现在我们一起来解决它。

再看一下 _C++14_ 版本的 _authAndAccess_ 声明。
```C++
  template<typename Container, typename Index>
  decltype(auto) authAndAccess(Container& c, Index i);
```

容器是通过 _lvalue-reference-to-non-const_ 而传入的，因为返回的是容器的元素的引用，所以允许客户去修改这个容器。但是这也意味着不能传递右值类型的容器到这个函数中。右值不能绑定左值引用，除非是 _lvalue-references-to-const_，这里并不是这种场景。

诚然，传递右值类型的容器到 _authAndAccess_ 是一个边缘场景。做为临时对象的右值容器一般是在  _authAndAccess_ 调用语句结束后所销毁的，这意味着这个容器中的元素的引用在创建它的语句结束后是悬空的，而 _authAndAccess_ 一般也会返回这个引用。尽管如此，传递一个临时对象到 _authAndAccess_ 中仍然是合理的。客户可能只是想要临时容器中的一个元素的副本而已，例如：  
```C++
  std::deque<std::string> makeStringDeque();      // factory function
  
                                                  // make copy of 5th element of deque returned
                                                  // from makeStringDeque
  auto s = authAndAccess(makeStringDeque(), 5);
```  
支持这样的用法意味着我们需要去修改 _authAndAccess_ 的声明以去接受右值和左值。重载将会起作用，一个重载声明左值引用类型的形参，另一个声明右值引用类型的形参，但是我们就有了两个函数需要维护。避免重载的方法是让 _authAndAccess_ 利用可以同时绑定左值和右值的引用类型的形参，[_Item 24_](Chapter%205.md#item-24-区分-universal-reference-和右值引用) 也解释了那正是 _univeral reference_ 的作用。因此 _authAndAccess_ 可以声明为下面这样：  
```C+++
  template<typename Container, typename Index>    // c is now a
  decltype(auto) authAndAccess(Container&& c,     // universal
                              Index i);           // reference
``` 

在这个模板中，我们不知道我们正在操作的容器的类型，这也意味着我们同样地忽略了容器所使用的索引对象的类型。对于未知类型的对象使用 _pass-by-value_ 通常是冒有一些风险的：不必要拷贝的性能损耗、对象切割的行为问题，见 [_Item 40_](Chapter%207.md#item-40-并发使用-stdatomic-特殊内存使用-volatile) 和同事嘲笑的刺痛，但是在容器索引的场景中，遵循索引值所对应的标准库的例子看起来仍然是合理的，比如：_std::string_、_std::vector_ 和 _std::deque_ 的 _operator[]_，所以对此我们仍然 _pass-by-value_。

然而，我们需要去更新模板的实现，使其符合 [_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference) 的警告，将 _std::forward_ 应用到 _univeral reference_ 上：  
```C++
  template<typename Container, typename Index>        // final
  decltype(auto)                                      // C++14
  authAndAccess(Container&& c, Index i)               // version
  {
    authenticateUser();
    return std::forward<Container>(c)[i];
  }
```

这个应该做了我们想要做的所有事了，但是需要 _C++14_ 的编译器。如果你没有的话，你需要使用 _C++11_ 的模板版本。除了你必须要自己指定返回类型以外，它和 _C++14_ 是相同的：  
```C++
  template<typename Container, typename Index>    // final
  auto                                            // C++11
  authAndAccess(Container&& c, Index i)           // version
  -> decltype(std::forward<Container>(c)[i])
  {
    authenticateUser();
    return std::forward<Container>(c)[i];
  }
```

另一个可能困扰你的问题在本 _Item_ 开始的时候就提到了，那就是 _decltype_ **几乎**总是产生你所期待的类型，**很少**有惊喜。说实话，除非你是一个重度库实现者，否则你不太可能遇到这些规则的例外。

为了**全面**理解 _decltype_ 的行为，你必须熟悉少量的特殊场景。这些特殊场景的大部分都很少见，以至于没必要在  书中像这样进行讨论，但是看一个让我们深入了解 _decltype_ 和它的用法的场景。

应用 _decltype_ 到名字上会得到这个名字的声明的类型。名字是左值表达式，但是不会影响 _decltype_ 的行为。然而对于比名字还要复杂的左值表达式，_decltype_ 是确保所报告的类型总是左值引用的。也就是说：如果非名字的左值表达式有类型 _T_ 的话，那么 _decltype_ 所报告的类型是为 _T&_ 的。这很少会有影响，因为大部分左值表达式的类型内部就是包含有左值引用 _qualifier_ 的。例如：返回左值的函数返回的总是左值引用。

然而，这样的行为有一个值得我们注意的影响。在：  
```C++
  int x = 0;
```  
_x_ 是变量的名字，所以 _decltype(x)_ 是 _int_。但是是使用 _parenthese_ 将名字 _x_ 括起来的，也就是 _(x)_，产生的是比名字还要复杂的表达式。做为名字的 _x_ 是左值，而 _C++_ 把 _(x)_ 也是定义为左值的。因此，_decltype((x))_ 是 _int&_。将名字用 _parenthese_ 括起来可以改变 _decltype_ 所报告的类型。

在 _C++_ 中，这不过是稍微稀奇点。但是结合 _C++14_ 对于 _decltype(auto)_ 的支持，这意味着在你写的 _return_ 语句上的一点看起来是微不足道的改变就可以影响函数的所推导出的类型：  
```C++
  decltype(auto) f1()
  {
    int x = 0;
    …
    return x;                 // decltype(x) is int, so f1 returns int
  }

  decltype(auto) f2()
  {
    int x = 0;
    …
    return (x);               // decltype((x)) is int&, so f2 returns int&
  }
```  
注意：不仅 _f2_ 有着和 _f1_ 不同的返回类型，而且还返回了一个指向局部变量的引用。这样的代码让你坐上了未定义行为的快车，一辆你确定不想上车的快车。  
> 译者注：脑残规则，后续不修复的话，更是智障。

最重要的是：当使用 _decltype(auto)_ 时，要非常小心。在类型被推导的表达式中的一些看起来是微不足道的细节就可以影响 _decltype(auto)_ 所报告的类型。为了确保正在被推导的类型是你所期待的类型，使用  [_Item 4_](#item-4-了解如何查看所推导的类型) 所描述的技术。

与此同时，不要忽视大局。_decltype_ 单独使用和结合 _auto_ 一起使用确实偶尔可能会产生类型推导的惊喜，但那不是常见的场景。一般情况下，_decltype_ 是会产生你所期待的类型的。当 _decltype_ 被应用到名字上时，尤其如此，因为在那种场景下，_decltype_ 就是做听起来的那样：报告名字的类型的类型。

### 需要记住的规则

* _decltype_ 几乎总是能在不做修改的情况下就产生变量或表达式的类型。
* 对于非名字的类型 _T_ 的左值表达式，_decltype_ 总是报告类型为 _T&_。
* _C++14_ 支持 _decltype(auto)_，像 _auto_ 一样，_decltype(auto)_ 是根据它的表达式来进行类型推导的，但是使用的是 _decltype_ 的规则。

## _Item 4_ 了解如何查看所推导的类型

查看类型推导结果的工具的选择是取决于软件开发过程的阶段的。我们将会探讨三种需要获取类型推导信息的可能性：当你编辑代码时、当编译代码时和当运行代码时。

### _IDE_ 编辑器

当你将鼠标放在程序实体上时，_IDE_ 的代码编辑器通常会显示程序实体的类型，比如：变量、形参和函数等。例如这样的代码：  
```C ++
  const int theAnswer = 42;
  
  auto x = theAnswer;
  auto y = &theAnswer;
```  
_IDE_ 编辑器可能会显示 x 的推导出的类型是 _int_，_y_ 则是 _const int*_。

为了可以这样，你的代码或多或少得处于可编译状态，因为是运行在 _IDE_ 内部的 _C++_ 编译器或至少是它的前端才能使得 _IDE_ 可以提供这种信息。如果编译器不能充分理解你的代码以去解析它并执行类型推导的话，那么是不能显示它所推导的类型的。

对于像 _int_ 这样的简单的类型，来自于 _IDE_ 的信息通常是好的。然而，正如我们很快要看到的，当涉及到更复杂的类型时，_IDE_ 所显示的信息可能就不是特别有帮助了。

### 编译器诊断

一个可以让编译器显示它所推导的类型的高效方法是按照产生编译问题的方式来使用这个类型。报告出的问题几乎肯定会提到那个导致问题出现的类型。

例如：假如我们想要看到前面例子中的 _x_ 和 _y_ 的推导出的类型。我们首先声明一个我们没有定义的类模板。最好这样做：  
```C++
  template<typename T>        // declaration only for TD;
  class TD;                   // TD == "Type Displayer
```  
尝试实例化这个模板会得到一个错误信息，因为没有模板定义可以实例化。为了看到 _x_ 和 _y_ 的类型，试着使用它们的类型来实例化 _TD_：  
```C++
  TD<decltype(x)> xType;      // elicit errors containing
  TD<decltype(y)> yType;      // x's and y's types
```

我使用 _variableNameType_ 格式的变量名，因为它们倾向于产生一些错误信息，而这些错误能帮助你发现你正在查找的信息。对于上面的代码，我的一个编译器发出的诊断信息部分如下，我已经高亮了我们想要的类型信息：
```C++
  error: aggregate 'TD<int> xType' has incomplete type and
      cannot be defined
  error: aggregate 'TD<const int *> yType' has incomplete type
      and cannot be defined
```  
另一个不同的编译提供了相同的信息，但是使用了不同的格式：
```C++
  error: 'xType' uses undefined class 'TD<int>'
  error: 'yType' uses undefined class 'TD<const int *>'
```

抛开格式差异不谈，当使用这个技术时，所有的我已经测试过的编译器都会发出类型信息有关的错误信息。

### 运行时输出

使用 _printf_ 来显示类型信息的方法直到运行时才可以使用，不代表我建议你使用 _printf_，但是 _printf_ 提供了对格式化输出的全部控制。存在的挑战是创建一个适合显示你所关注的类型的文本表示。你正在想“不要流汗，可以使用 _typeid_ 和 _type_info::name_。”我们继续探究 _x_ 和 _y_ 的推导的类型，你可能认为我们可以写成这样：  
```C++
  std::cout << typeid(x).name() << '\n';          // display types for
  std::cout << typeid(y).name() << '\n';          // x and y
```

这个方法依赖于这样的事实，那就是在像 _x_ 或 _y_ 这样的对象上调用 _typeid_ 会产生 _std::type_info_ 类型的对象，而 _std::type_info_ 有一个成员函数 _name_，它可以生成一个类型的名字的 _C-style_ 字符串，即为：一个 _const char*_。

调用 _std::type_info::name_ 并不能保证所返回的内容都是明显的，但是会尽力提供帮助。帮助的级别各不相同。例如，_GUN_ 编译器和 _Clang_ 编译器所报告的 _x_ 的类型是 _i_，而 _y_ 的类型是 _PKi_。在这些编译器的输出中，_i_ 意味着是 _int_，而 _PK_ 意味着是 _pointer to const_，一旦你学会了这个，这些结果也就合理了，这两个编译器都支持一个工具 _c++filt_，这个工具可以解析这些无逻辑的类型。_Microsoft_ 的编译器有更清晰的输出：_x_ 是 _int_，而 _y_ 是 _int const *_。

因为对于 _x_ 和 _y_ 的类型来说，结果是正确的，所以你可能会倾向于认为类型报告的问题已经解决了，但是先不要草率，考虑一个更复杂的例子：  
```C++
  template<typename T>                  // template function to
  void f(const T& param);               // be called

  std::vector<Widget> createVec();      // factory function
  
  const auto vw = createVec();          // init vw w/factory return
  if (!vw.empty()) {
    f(&vw[0]);                          // call f
    …
  } 
```  
这个代码涉及到了一个用户定义类型 _Widget_、一个 _STL_ 容器 _std::vector_ 和一个 _auto_ 变量 _vw_，这更能代表你可能想要看到你的编译器推导出类型的情景。例如：能知道所推导出的模板类型形参 _T_ 和 _f_ 的函数形参 _param_ 的类型就好了。

在这个问题上使用 _typeid_ 是直观的。仅仅需要添加一些代码到 _f_ 中，就可以显示你想要看到的类型：  
```C++
template<typename T>
void f(const T& param)
{
  using std::cout;
  cout << "T =     " << typeid(T).name() << '\n';     
  
  cout << "T = " << typeid(T).name() << '\n';               // show T

  cout << "param = " << typeid(param).name() << '\n';       // show
  …                                                         // param's
}                                                           // type
```  
_GUN_ 编译器和 _Clang_ 编译器所生成的可执行文件会产生这样的输出：  
```C++
  T = PK6Widget
  param = PK6Widget
```  
我们已经知道了，对于这些编译器来说，_PK_ 意味着 **_pointer-to-const_** ，所以唯一神秘的只有 _6_ 了。它只是后面的类型名 _Widget_ 的字符数量。所以这些编译器告诉我们的是 _T_ 和 _param_ 的类型都是为 _const Widget*_。

_Microsoft_ 的编译器也这样：  
```C++
  T = class Widget const *
  param = class Widget const *
```  

三个独立的编译器都产生了相同信息，应该能认为这些信息是准确的了吧。但是仔细看下。在模板 _f_ 中，_param_ 的类型是 _const T&_。如果是这样的话，_T_ 和 _param_ 有着相同的类型难道不奇怪吗？例如：如果 _T_ 是 _int_ 的话，那么 _param_ 应该是 _const int&_，不应该是相同的。

不幸地是，_std::type_info::name_ 的结果是不可靠的。例如：在这个场景中，三个编译器所报告的 _param_ 的类型都是不正确的。此外，它们是被要求出错的，因为 _std::type_info::name_ 的规范要求这些类型是按照 _by-value_ 的形式传递给模板函数的。参考 [_Item 1_](#item-1-理解模板的类型推导)，如果这些类型是引用的话，那么它们的 _reference-ness_ 是会被忽略的，如果忽略后还有 _const_ 或 _volatile_ 的话，那么它们的 _constness_ 或 _volatileness_ 也是会被忽略的。这也是为什么 _param_ 的类型实际上是 _const Widget * const &_ 但却会被报告为 _const Widget *_ 的原因。它们的 _reference-ness_ 首先会被忽略，然后所生成的指针的 _constness_ 也会被忽略。

同样不幸地是，_IDE_ 所显示的类型信息也是不可靠的，或者至少是不能可靠地被使用的。对于相同的实例，一个我知道的 _IDE_ 编辑器报告的类型是下面这样的，这不是我乱写的：  
```C++
  const
  std::_Simple_types<std::_Wrap_alloc<std::_Vec_base_types<Widget,
  std::allocator<Widget> >::_Alloc>::value_type>::value_type *
```  
这个 _IDE_ 编辑器显示的 _param_ 的类型为： 
```C++
  const std::_Simple_types<...>::value_type *const &
```  
比起 _T_ 的类型来说，这没有那么令人不安，但是其中的 _..._ 是令人困惑的，直到你明白了其中的 _..._ 是 _IDE_ 编辑器在说“我正在忽略 _T_ 的类型那部分呢。”幸运的话，你的开发环境可以更好地处理这样的代码。

如果你更倾向于依赖库而不是幸运的话，那么你会很想知道：_std::type_info::name_ 和 _IDE_ 为什么会失败呢？而通常被写为 _Boost.TypeIndex_ 的 _Boost TypeIndex library_ 又为什么会成功呢？这个库不是标准 _C++_ 的一部分，但 _IED_ 和像 _TD_ 这样的模板也不是标准 _C++_ 的一部分。此外，[boost.com](https://www.boost.org/) 上的 _Boost library_ 是跨平台的、开源的，并且可以在一个许可证下使用，即使是最偏执的公司的法律团队也能接受这个许可证，这意味着使用 _Boost library_ 的代码几乎可以像依赖于标准库的那些代码一样可移植。

下面是我们的函数 _f_ 如何使用 _Boost.TypeIndex_ 来生成准确的类型信息：  
```C++
  #include <boost/type_index.hpp>

  template<typename T>
  void f(const T& param)
  {
    using std::cout;
    using boost::typeindex::type_id_with_cvr;
    
    // show T
    cout << "T = "
      << type_id_with_cvr<T>().pretty_name()
      << '\n';
    
    // show param's type
    cout << "param = "
      << type_id_with_cvr<decltype(param)>().pretty_name()
      << '\n';
    …
}
```

它的工作方式是：函数模板 _boost::typeindex::type_id_with_cvr_ 接受一个类型实参，而这个类型就是我们想要的信息，并且这个函数模板是不会将 _const_、_volatile_ 和 _reference qualifier_ 进行忽略的 ，所以这个函数模板名中包含有 **_with_cvr_**。结果就是 _boost::typeindex::type_id_with_cvr_ 的 _pretty_name_ 成员函数会产生一个包含有类型的人性化表示的 _std::string_。

有了这个 _f_ 的实现，再一次考虑那个当 _typeid_ 被使用时产生了 _param_ 的错误类型信息的调用：  
```C++
  std::vector<Widget> createVec();      // factory function

  const auto vw = createVec();          // init vw w/factory return
  
  if (!vw.empty()) {
    f(&vw[0]);                          // call f
    …
  }
```  
在 _GUN_ 编译器和 _Clang_ 编译器下，_Boost TypeIndex_ 会产生下面这样的输出，它是准确的：  
```C++
  T = Widget const*
  param = Widget const* const&
```  
_Microsoft_ 的编译器所产生的结果本质上也是相同的：  
```C++
  T = class Widget const *
  param = class Widget const * const &
```  

这样的近似一致性是好的，但是更重要的是记住 _IDE_ 编辑器、编译器错误信息和像 _Boost.TypeIndex_ 这样的库都只是工具，可以使用这些工具来帮助你搞清楚编译器正在推导的类型是什么。这些都是有用的，但说到底，还是要理解 _Item 1-3_ 中的类型推导信息，这是无可替代的。

### 需要记住的规则

* 所推导的类型通常可以通过使用 _IDE_ 编辑器、编译器错误信息和 _Boost TypeIndex library_ 来进行查看。
* 因为一些工具的结果可能是无用的和不准确的，所以理解 _C++_ 的类型推导规则仍然是必要的。