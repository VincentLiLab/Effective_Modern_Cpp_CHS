- [_Chapter 3_ _Moving to Modern C++_](#chapter-3-moving-to-modern-c)
  - [_Item 7_ 创建对象时区分 _()_ 和 _{}_](#item-7-创建对象时区分--和-)
    - [需要记住的规则](#需要记住的规则)
  - [_Item 8_ 首选 _nullptr_ 而不是 _0_ 和 _NULL_](#item-8-首选-nullptr-而不是-0-和-null)
    - [需要记住的规则](#需要记住的规则-1)
  - [_Item 9_ 首选 _alias declaration_ 而不是 _typedef_](#item-9-首选-alias-declaration-而不是-typedef)
    - [需要记住的规则](#需要记住的规则-2)
  - [_Item 10_ 首选 _scoped enum_ 而不是 _unscoped enum_](#item-10-首选-scoped-enum-而不是-unscoped-enum)
    - [需要记住的规则](#需要记住的规则-3)
  - [_Item 11_ 首选 _deleted function_ 而不是 _private undefined function_](#item-11-首选-deleted-function-而不是-private-undefined-function)
    - [需要记住的规则](#需要记住的规则-4)
  - [_Item 12_ 使用 _override_ 来声明重写函数](#item-12-使用-override-来声明重写函数)
    - [需要记住的规则](#需要记住的规则-5)
  - [_Item 13_ 首选 _const\_iterator_ 而不是 _iterator_](#item-13-首选-const_iterator-而不是-iterator)
    - [需要记住的规则](#需要记住的规则-6)
  - [_Item 14_ 当函数不会抛出异常时声明函数为 _noexcept_](#item-14-当函数不会抛出异常时声明函数为-noexcept)
    - [需要记住的规则](#需要记住的规则-7)
  - [_Item 15_ 只要有可能就使用 _constexpr_](#item-15-只要有可能就使用-constexpr)
    - [需要记住的规则](#需要记住的规则-8)
  - [_Item 16_ 使 _const_ 成员函数成为线程安全的](#item-16-使-const-成员函数成为线程安全的)
    - [需要记住的规则](#需要记住的规则-9)
  - [_Item 17_ 理解特殊成员函数的生成](#item-17-理解特殊成员函数的生成)
    - [需要记住的规则](#需要记住的规则-10)

# _Chapter 3_ _Moving to Modern C++_

_C++11_ 和 _C++14_ 有很多值得夸耀的特性：_auto_、智能指针、移动语义、_lambda_ 和 _concurrency_，每一个都是非常重要的。我专门写了这一章来介绍它们。掌握这些特性是必不可少的，但是需要一系列更小的步骤才能变为一名高效率的 _modern C++_ 的程序员。每一个步骤都回答了在 _C++98_ 转向 _modern C++_ 的过程中所产生的问题。什么时候应该使用 _{}_ 代替 _()_ 来创建对象？为什么 _alias declartion_ 会比 _typedef_ 要更好呢？_constexpr_ 和 _const_ 是如何的不同呢？_const_ 成员函数和线程安全之间有什么关系？还有很多，本章一个接着一个来进行解答。

## _Item 7_ 创建对象时区分 _()_ 和 _{}_

取决于你的观点，_C++11_ 中的对象的初始化语法的选择要么是令人尴尬的富裕，要么是令人困惑的混乱。一般而言，初始化值可以使用 _()_、_=_ 和 _{}_ 来进行指定：
```C++
  int x(0);                   // initializer is in parentheses
  
  int y = 0;                  // initializer follows "="
  
  int z{ 0 };                 // initializer is in braces
```

在很多场景下，_=_ 和 _{}_ 也可以一起使用：  
```C++
  int z = { 0 };              // initializer uses "=" and braces
```

对于本 _Item_ 的剩余部分，我通常会忽略 _equals-sign-plus-braces_ 语法，因为 _C++_ 通常把它和 _braces-only_ 做同样地处理。 

> 实际测试不是这样，使用的编译器是 _c++ (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0_。_auto v{0, 1};_ 这样是无法通过编译的并且 _auto v{0}_ 中的 _v_ 是 _int_ 类型而不是 _std::initializer_list_ 类型。

**_令人困惑的混乱_** 派指出：使用 _=_ 来进行初始化常常会让 _C++_ 菜鸟误认为赋值发生了，实际并不是。对于像 _int_ 这样的内建类型来说只是学术上的不同，但是对于用户定义的类型来说，区分初始化和赋值就非常重要了，因为这会涉及到不同的函数：  
```C++
  Widget w1;                  // call default constructor
  
  Widget w2 = w1;             // not an assignment; calls copy ctor
  
  w1 = w2;                    // an assignment; calls copy operator=
```
即使有着这么多的初始化语法，但是 _C++98_ 仍然无法去表达一个所期待的初始化。例如：不能直接创建持有一组特定值的 _STL_ _container_，比如：_1，2，5_。

为了解决这么多个初始化语法的所带来困惑和这些初始化语法并不能覆盖全部初始化情况的事实，_C++11_ 引入了 _uniform initialization_：一个可以，至少概念上可以，被用在任意地方和表达任意事情的单一的初始化语法。它是基于 _{}_ 的，所以我更喜欢 _braced initialization_ 这个术语。**_uniform initialization_** 是思想，**_braced initialization_** 是语法构成。
            
_braced initialization_ 让你能表达之前不能被表达出的内容，使用 _{}_，非常容易指定一个 _container_ 的初始化内容：  
```C++
  std::vector<int> v{ 1, 3, 5 };        // v's initial content is 1, 3, 5
```

_{}_ 也可以被用于指明非静态数据成员的默认初始化值。这个是新添加到 _C++11_ 的中的功能，_=_ 也可以完成这个功能，但是 _()_ 不可以：  
```C++
  class Widget {
  …
  private:
    int x{ 0 };               // fine, x's default value is 0
    int y = 0;                // also fine
    int z(0);                 // error!
  };
```  

另一方面，_uncopyable_ 对象可以使用 _{}_ 和 _()_ 来进行初始化，但是不能使用 _=_：  
```C++
  std::atomic<int> ai1{ 0 };  // fine

  std::atomic<int> ai2(0);    // fine
  
  std::atomic<int> ai3 = 0;   // error!
```

这也就能理解了为什么 _braced initialization_ 被称为 **_uniform_**。在 _C++_ 的三种指定初始化表达式的方法中，只有 _{}_ 可以被用在所有地方。

_braced initialization_ 的新颖特性是可以禁止内建类型之间的 _implicit narrowing conversion_。如果 _braced initializer_ 中的表达式的值不能保证被正在被初始化的对象的类型所表达的话，那么就不能通过编译：  
```C++
  double x, y, z;
  
  …
  
  int sum1{ x + y + z };      // error! sum of doubles may
                              // not be expressible as int
```

使用 _()_ 和 _=_ 进行初始化不会检查 _narrowing conversion_，因为这会破坏很多的 _legacy_ 代码：  
```C++
  int sum2(x + y + z);        // okay (value of expression
                              // truncated to an int)

  int sum3 = x + y + z;       // ditto
```

_braced initialization_ 的另一个值得关注的特性是它对于 _C++_ 的 _most vexing parse_ 是免疫的。在 _C++_ 中有一个规则：任何可以被解析为声明的事物都必须被解释为声明，这个规则是有副作用的，那就是 _most vexing parse_ 会让开发者在想要调用默认构造函数时却不经意间声明出了一个函数。这个问题的根源在于，如果你想要调用实参构造函数的话，你可以这样做：  
```C++
  Widget w1(10);              // call Widget ctor with argument 10
```  
但是，如果你使用相似的语法来调用 _Widget_ 的无参构造函数的话，你会声明出一个函数而不是对象：
```C++
  Widget w2();                // most vexing parse! declares a function
                              // named w2 that returns a Widget!
```

函数不能使用 _{}_ 来声明参数列表，所以使用 _{}_ 来进行默认构造就不会有这个问题：
```C++
  Widget w3{};                // calls Widget ctor with no args
```

因此，_braced initialization_ 有很多值得一提的地方。它是可以在最广泛的语境下被使用的语法、它可以避免 _implicit narrowing conversion_，并且它对于 _C++_ 的 _most vexing parse_ 是免疫的。三连冠！那为什么本 _Item_ 
不命名为“首选 _braced initialization_ 语法”呢？

_braced initialization_ 的缺点是有时会出现令人惊讶的行为。这些行为源自于 _braced initializer_、_std::initializer_list_ 和 构造函数重载决议之间的异常复杂的关系。它们的相互作用可能会导致代码看起来像是在做一件事，但实际上却是在做另一件事。例如：[_Item 2_](Chapter%201.md#item-2-理解-auto-的类型推导) 解释了：当 _auto_ 声明的变量有 _braced initializer_ 时，所推导出的这个变量的类型应该是 _std::initializer_list_ 的，尽管用其他方式声明带有相同 _initializer_ 的变量可能会产生更直观的类型。结果就是你越喜欢使用 _auto_，你就越不喜欢 _braced initialization_。

在构造函数调用中，只要不涉及到 _std::initializer_list_ 的话，那么 _{}_ 和 _=_ 就有着相同的意义：  
```C++
class Widget {
public:
  Widget(int i, bool b);      // ctors not declaring
  Widget(int i, double d);    // std::initializer_list params
…
};

Widget w1(10, true);          // calls first ctor

Widget w2{10, true};          // also calls first ctor

Widget w3(10, 5.0);           // calls second ctor

Widget w4{10, 5.0};           // also calls second ctor
```
然而，如果一个或多个构造函数声明了 _std::initializer_list_ 类型的形参的话，那么使用了 _braced initialization_ 语法的调用会强烈地优先选择持有 _std::initializer_list_ 的重载函数。是强烈地。如果编译器有方法将使用了 _braced initializer_ 的调用解释为持有 _std::initializer_list_ 的构造函数的话，那么编译器将会采用这种解释。如果上面的 _Widget_ 类增加了持有 _std::initializer_list&lt;long double&gt;_ 的构造函数的话，例如：  
```C++
  class Widget {
  public:
    Widget(int i, bool b);                                  // as before
    Widget(int i, double d);                                // as before
    Widget(std::initializer_list<long double> il);          // added

  …
};
```  

_Widget_ _w2_ 和 _w4_ 将使用新的构造函数来进行构造，虽然 _std::initializer_list_ 的元素的类型 _long double_ 相对于其他的 _non-std::initializer_list_ 构造函数来说是对所有实参的更糟糕的匹配！看：
```C++
  Widget w1(10, true);        // uses parens and, as before,
                              // calls first ctor
  
  Widget w2{10, true};        // uses braces, but now calls
                              // std::initializer_list ctor
                              // (10 and true convert to long double)

  Widget w3(10, 5.0);         // uses parens and, as before,
                              // calls second ctor

  Widget w4{10, 5.0};         // uses braces, but now calls
                              // std::initializer_list ctor
                              // (10 and 5.0 convert to long double)
```

甚至那些常规的拷贝构造和移动构造也可以被 _std::initializer_list_ 构造函数所劫持：
```C++
  class Widget {
  public:
    Widget(int i, bool b);                                  // as before
    Widget(int i, double d);                                // as before
    Widget(std::initializer_list<long double> il);          // as before
    
    operator float() const;                                 // convert
    …                                                       // to float
  };

  Widget w5(w4);                                            // uses parens, calls copy ctor
  
  Widget w6{w4};                                            // uses braces, calls
                                                            // std::initializer_list ctor
                                                            // (w4 converts to float, and float
                                                            // converts to long double)

  Widget w7(std::move(w4));                                 // uses parens, calls move ctor

  Widget w8{std::move(w4)};                                 // uses braces, calls
                                                            // std::initializer_list ctor
                                                            // (for same reason as w6)
```

编译器决定将持有 _std::initializer_list_ 的构造函数和 _braced initializer_ 来进行匹配的决心是非常强烈的，即使这个所匹配的 _std::initializer_list_ 构造函数是不可以被调用的，它也是占上风的。比如：
```C++
  class Widget {
  public:
    Widget(int i, bool b);                        // as before
    Widget(int i, double d);                      // as before

    Widget(std::initializer_list<bool> il);       // element type is
                                                  // now bool
    …                                             // no implicit
  };                                              // conversion funcs

  Widget w{10, 5.0};                              // error! requires narrowing conversions
```  
此处，编译器将会忽略前两个构造函数，尽管对于所有的实参类型来说，其中的第二个构造函数提供了准确的匹配，然后会尝试去调用持有 _std::initializer_list&lt;bool&gt;_ 的构造函数。调用这个构造函数是需要将 _int(10)_ 和 _double(5.0)_ 转换为 _bool_ 的。因为这两个转换都是 _narrowing_ 的，_bool_ 不能准确表示这两个值，_braced initializer_ 是禁止 _narrowing conversion_ 的，所以这个调用是无效的，代码是无法通过编译的。

只有当没有办法将 _braced initializer_ 中的实参的类型转换为 _std::initializer_list_ 中的类型时，编译器才会回退到一般的重载决议中。例如：如果我们使用 _std::initializer_list&lt;std::string&gt;_ 的构造函数来代替 _std::initializer_list&lt;bool&gt;_ 的话，那么 _non-std::initializer_list_ 构造函数将再次成为候选，因为没有办法转换将 _int_ 和 _bool_ 转换为 _std::string_：
```C++
  class Widget {
  public:
    Widget(int i, bool b);                                  // as before
    Widget(int i, double d);                                // as before
                                                            // std::initializer_list element type is now std::string
    Widget(std::initializer_list<std::string> il);
    …                                                       // no implicit
  };                                                        // conversion funcs

  Widget w1(10, true);                                      // uses parens, still calls first ctor
  Widget w2{10, true};                                      // uses braces, now calls first ctor
  Widget w3(10, 5.0);                                       // uses parens, still calls second ctor
  Widget w4{10, 5.0};                                       // uses braces, now calls second ctor
```

这让我们靠近 _braced initializer_ 和构造函数重载的讨论的末尾了，但是仍然有令人感兴趣的边缘场景需要被解决。假定你使用了一个空 _{}_ 去构造一个对象，并且这个对象支持默认构造和 _std::initializer_list_ 构造。空 _{}_ 意味着什么呢？如果是 **_no arguments_** 的话，那么调用的应该是默认构造函数，如果是 **_empty std::initializer_list_** 的话，那么调用的应该是没有元素的 _std::initializer_list_ 构造函数。

答案是默认构造函数。空 _{}_ 意味着没有实参，而不是 _empty std::initializer_list_：  
```C++
  class Widget {
  public:
    Widget();                                     // default ctor

    Widget(std::initializer_list<int> il);        // std::initializer
                                                  // _list ctor

    …                                             // no implicit
  };                                              // conversion funcs

  Widget w1;                                      // calls default ctor

  Widget w2{};                                    // also calls default ctor
  Widget w3();                                    // most vexing parse! declares a function!
  ```

如果你想要调用 _empty std::initializer_list_ 的 _std::initializer_list_ 构造函数的话，那么可以通过空 _{}_ 构造函数实参来完成：通过 _({})_ 或 _{{}}_ 来清楚表明你正在传递的内容：
```C++
  Widget w4({});              // calls std::initializer_list ctor
                              // with empty list
  
  Widget w5{{}};              // ditto
```

此时，_braced initializer_、_std::initializer_list_ 和构造函数重载这些看起来是晦涩难懂的规则在你的脑海中萦绕，你可能正在好奇这些在日常编程中又有多重要呢？比你想的要重要，因为直接受影响的类便是 _std::vector_。_std::vector_ 有 _non-std::initializer_list_ 构造函数，这个构造函数允许你指明 _container_ 的大小和每个元素的初始值，_std::vector_ 也有 _std::initializer_list_ 构造函数，这个构造函数允许你指明 _container_ 的初始值。如果你创建了 _numeric_ 类型 _std::vector_ 的话，比如：_std::vectort&lt;int&gt;_，然后你传递了两个实参到构造函数中，使用 _{}_ 或 _()_ 会有巨大的不同：  
```C++
  std::vector<int> v1(10, 20);          // use non-std::initializer_list
                                        // ctor: create 10-element
                                        // std::vector, all elements have
                                        // value of 20

std::vector<int> v2{10, 20};            // use std::initializer_list ctor:
                                        // create 2-element std::vector,
                                        // element values are 10 and 20
```

但是让我们从 _std::vector_ 中回退一步，也从 _()_、_{}_ 以及构造函数重载决议规则的细节中回退一步。从这个讨论中可以得到两个重点。首先，做为一个类的实现者，你需要了解：如果你的一组重载构造函数中包含了一个或者多个持有 _std::initializer_list_ 的函数的话，那么那些使用了 _braced initialization_ 的客户代码可能就只会看到 _std::initializer_list_ 类型的重载函数了。总之，要精心设计你的构造函数，让重载函数不被用户使用 _()_ 还是 _{}_ 所影响。换句话说，现在要把 _std::vector_ 的设计看成是错误的，应该从中学习并让你的类去避免发生这种错误。

一个可能的影响是：如果你有一个原先并没有 _std::initializer_list_ 构造函数的类，然后你添加了一个的话，那么使用 _braced initialization_ 的客户代码可能会认为：原先调用的是 _non-std::initializer_list_ 构造函数，现在调用的是新添加的构造函数了。当然，这种事情可以发生在任何你添加新的函数到一组重载函数的时候：原先调用的是旧的重载函数，现在开始调用的是新的重载函数了。和 _std::initializer_list_ 构造函数重载不同的是：_std::initializer_list_ 重载函数不只是会和其他的重载函数进行竞争，而是会遮盖其他的重载函数，其他的重载函数很难再被考虑到了。所以只有在深思熟虑后，再去添加这些 _std::initializer_list_ 重载函数。

其次：做为一个类用户，当创建对象时，你必须在使用 _()_ 还是 _{}_ 之间谨慎选择。大多数开发者会选择其中一种来做为其默认，只有当必须要使用另一种时，才会去使用另一种。 _{}_ 的无与伦比的适用性广度、_{}_ 可以禁止 _narrowing conversion_ 以及 _{}_ 对 _C++_ 的 _most vexing parse_ 的免疫都吸引了 _Braces-by-default_ 派。他们认为只有在一些特定场景下，比如：在创建给定大小和每个元素初始值的 _std::vector_ 时，才需要去使用 _()_。而在另一方面，_go-parentheses-go_ 派则将 _()_ 做为其默认。_()_ 和 _C++98_ 的语法一致、_()_ 能避免 _auto-deduced-a-std::initializer_list_ 的问题以及创建对象的调用不会被 _std::initializer_list_ 构造函数所无意地拦截都吸引着 _go-parentheses-go_ 派。他们承认有时候只有 _{}_ 才能完成一些事，比如：当创建一个有着特定值的 _container_ 时。哪个更好是没有共识的，所以我的建议是选择一个，然后一直使用它。

如果你是一个模板实现者的话，那么在创建对象时 _{}_ 和 _()_ 之间的紧张关系会令人更沮丧，因为一般不可能知道哪个应该被使用。例如：假定你从任意数量的实参中创建一个任意类型的对象。可变参数模板让这一切在概念上变得直观：  
```C++
  template<typename T,                  // type of object to create
  typename... Ts>                       // types of arguments to use
  void doSomeWork(Ts&&... params)
  {
    create local T object from params...
    …
  }
```

有两种方法将伪代码转为真实代码，对于 _std::forward_ 的信息见 [_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference)：
```C++
  T localObject(std::forward<Ts>(params)...);     // using parens
  
  T localObject{std::forward<Ts>(params)...};     // using braces
```  
所以，考虑这样的调用代码：  
```C++
  std::vector<int> v;
  …
  doSomeWork<std::vector<int>>(10, 20);
```

如果创建 _localObject_ 时 _doSomeWork_ 使用了 _()_ 的话，那么 _std::vector_ 就是有 _10_ 个元素。如果创建 _localObject_ 时  _doSomeWork_ 使用了 _{}_ 的话，那么 _std::vector_ 就是有 _2_ 个元素。哪一个是正确的？_doSomeWork_ 的作者是不知道的，只有调用方知道。

这是标准库函数 _std::make_unique_ 和 _std::make_shared_ 面对的问题，见 [_Item 21_](Chapter%204.md#item-21-首选-stdmake_unique-和-stdmake_shared-而不是直接使用-new)。这些函数通过内部使用 _()_ 和记录这个决定做为接口的一部分来解决这个问题。

### 需要记住的规则

* _braced initialization_ 是可使用的最广泛的初始化语法，它禁止 _narrowing conversion_ ，并且可以对 _C++_ 的 _most vexing parse_ 所免疫。
* 在构造函数重载决议期间，只要有可能，_braced initializer_ 就会和 _std::initializer_list_ 形参匹配，即使其他的构造函数提供了看起来是更好的匹配。
* 在选择使用 _()_ 和 _{}_ 时可能产生显著差异的一个例子是使用两个实参来创建 _std::vector&lt;numeric type&gt;_ 时。
* 当在模板中创建对象时，在 _()_ 和 _{}_ 之间进行选择是具有挑战性的。

## _Item 8_ 首选 _nullptr_ 而不是 _0_ 和 _NULL_

是这样的：字面上的 _0_ 是 _int_，而不是一个指针。如果 _C++_ 是在只有指针被使用的环境下看到了 _0_ 的话，那么是会勉强将 _0_ 解释为空指针的，但这是一个次选方案。_C++_ 的主要方针是 _0_ 是 _int_，而不是指针。

实际上，对于 _NULL_ 也是这样的。在 _NULL_ 的场景中是存在有一些细节上的不确定性的，这是因为实现允许 _NULL_ 是 _integral_ 类型而不是 _int_ 类型的，比如：_long_。这是不常见的，但是不重要，因为问题不在于 _NULL_ 的实际类型是什么，而是 _0_ 和 _NULL_ 都不是指针类型。

在 _C++98_ 中，一个可能的影响是：重载指针类型和 _integral_ 类型时可能会导致惊喜。传递 _0_ 或 _NULL_ 到这些重载函数时，永远不会调用到指针的重载函数：
```C++
  void f(int);                // three overloads of f
  void f(bool);
  void f(void*);
  
  f(0);                       // calls f(int), not f(void*)
  f(NULL);                    // might not compile, but typically calls
                              // f(int). Never calls f(void*)
```
_f(NULL)_ 的行为的不确定性反映出了在执行 _NULL_ 的类型时所给予的余地。如果 _NULL_ 被定义为是 _0L_ 的话，比如：_0_ 是 _long_，那么这个调用是具有二义性的，因为从 _long_ 到 _int_、从 _long_ 到 _bool_ 以及从 _0L_ 到 _void*_ 的 _conversion_ 都被认为是一样地好的。关于这个调用的有意思的地方是在于这个调用 _表面上的_ 含义和 _实际上的_ 含义是不一致的：使用 _NULL_ 来调用 _f_ 时是空指针，而使用 _integer_ 来调用 _f_ 则不是空指针。这种违反直觉的行为产生了 _C++98_ 程序员要避免重载指针类型和 _integral_ 类型的这样的准则。这个准则在 _C++11_ 中仍然有效，因为尽管有本 _Item_ 的建议，但是有一些开发者仍然会继续 _0_ 和 _NULL_，虽然 _nullptr_ 是更好的选择。

_nullptr_ 的优势是它不是 _integral_ 类型。老实说，它也不是指针类型，但是你可以认为它是所有类型的指针。它的实际类型是 _std::nullptr_t_，在一个完美的循环定义下，_std::nullptr_t_ 是被定义为 _nullptr_ 的类型的。_std::nullptr_t_ 可以隐式地被转换为原始指针类型，表现得就好像是它是所有类型的指针一样。

使用 _nullptr_ 调用重载函数 _f_ 会调用 _void*_ 重载函数，即为：指针重载函数，因为 _nullptr_ 不可以被做为 _integral_ 的：
```C++
  f(nullptr);                 // calls f(void*) overload
```  
因此，使用 _nullptr_ 来代替 _0_ 或 _NULL_ 可以避免重载决议的惊喜，但这不是唯一的优势。它还可以提高代码的清晰性，特别是当涉及到 _auto_ 变量时。例如：假如你在代码中遇到了下面这种情况：  
```C++
  auto result = findRecord( /* arguments */ );
  
  if (result == 0) {
    …
  }
```  
如果你碰巧不知道或者不容易发现 _findRecord_ 返回的是什么的话，那么就不清楚 _result_ 是指针类型还是 _integral_ 类型了。毕竟，这个 _result_ 所比较的 _0_ 可以是指针类型，也可以是 _integral_ 类型。另一方面，如果你看到的是下面这样的代码的话：  
```C++
  auto result = findRecord( /* arguments */ );
  
  if (result == nullptr) {
    …
  }
```  
那么就没有二义性了：_result_ 必须是指针类型。

当使用模板时，_nullptr_ 尤其地闪亮。假定你有一些函数，这些函数只有在合适的 _mutex_ 被锁定时才会被调用。每一个函数都持有不同类型的指针：
```C++
  int f1(std::shared_ptr<Widget> spw);            // call these only when
  double f2(std::unique_ptr<Widget> upw);         // the appropriate
  bool f3(Widget* pw);                            // mutex is locked
```  
想要传递空指针的调用代码看起来像是下面这样：
```C++
  std::mutex f1m, f2m, f3m;             // mutexes for f1, f2, and f3
  
  using MuxGuard =                      // C++11 typedef; see Item 9
  std::lock_guard<std::mutex>;
  …
  
  {
    MuxGuard g(f1m);                    // lock mutex for f1
    auto result = f1(0);                // pass 0 as null ptr to f1
  }                                     // unlock mutex
  
  …
  
  {
    MuxGuard g(f2m);                    // lock mutex for f2
    auto result = f2(NULL);             // pass NULL as null ptr to f2
  }                                     // unlock mutex
  
  …
  
  {
    MuxGuard g(f3m);                    // lock mutex for f3
    auto result = f3(nullptr);          // pass nullptr as null ptr to f3
  }
```  
在这个代码中的前两个调用中没有使用 _nullptr_ 是糟糕的，但是代码是可以工作的。这是重要的。在所调用的代码中的所重复的 _pattern_：锁定 _mutex_、调用函数、解锁 _mutex_，是更糟糕的。这样的复用代码是要去设计模板去避免的事情之一，所以让我们模板化这个 _pattern_：  
```C++
  template<typename FuncType,
            typename MuxType,
            typename PtrType>
  auto lockAndCall(FuncType func,
                    MuxType& mutex,
                    PtrType ptr) -> decltype(func(ptr))
  {
    MuxGuard g(mutex);
    return func(ptr);
  }
```  
如果这个函数的返回类型 _auto … -> decltype(func(ptr)_ 让你挠头的话，那么帮你的头个忙，请转到 [_Item 3_](Chapter%201.md#item-3-理解-decltype)，其中解释了这是发生了什么。在 _C++14_ 中，这个返回的类型可以被简化为 _declare(auto)_：  
```C++
  template<typename FuncType,
            typename MuxType,
            typename PtrType>
  decltype(auto) lockAndCall(FuncType func,       // C++14
                              MuxType& mutex,
                              PtrType ptr)
  {
    MuxGuard g(mutex);
    return func(ptr);
  }
```  

对于给定的 _lockAndCall_ 模板，另一个版本，调用方可以像下面这样写代码：
```C++
  auto result1 = lockAndCall(f1, f1m, 0);         // error!
  
  …
  
  auto result2 = lockAndCall(f2, f2m, NULL);      // error!  
  
  …

  auto result3 = lockAndCall(f3, f3m, nullptr);   // fine
```  
好吧，写是写了，但是正如注释所说明的，三个调用中的两个都会编译失败。第一个调用中的问题是：当将 _0_ 被传递给 _lockAndCall_ 时，会使用模板的类型推导来弄清楚它的类型是什么。_0_ 的类型现在、过去和未来都是 _int_，所以 _int_ 是 _lockAndCall_ 调用的实例化的形参 _ptr_ 的类型。不幸地是，这意味着在 _lockAndCall_ 中的 _func_ 调用中所传递的是 _int_ ，这和 _f1_ 所期待的 _std::shared_ptr&lt;Widget&gt;_ 形参是不兼容的。在 _lockAndCall_ 中所传递的 _0_ 原本想要表示的是空指针，但是实际上所传递却是 _run-of-the-mill_ _int_。尝试将 _int_ 做为 _std::shared_ptr&lt;Widget&gt;_ 来传递到 _f1_ 是类型错误的。使用 _0_ 来调用 _lockAndCall_ 是会失败的，因为在模板中，_int_ 是被传递给 _func_ 函数的，而 _func_ 函数所需要的却是 _std::shared_ptr&lt;Widget&gt;_。

对于涉及到 _NULL_ 的调用的分析本质上也是一样的。当 _NULL_ 被传递给 _lockAndCall_ 时，对于形参 _ptr_ 来说，所推导出的是 _integral_ 类型，而当 _int_ 类型或 _int-like_ 类型的 _ptr_ 被传递给 _f2_ 时，会发生类型错误，这是因为 _f2_ 期待的是 _std::unique_ptr&lt;Widget&gt;_。

相比之下，涉及到 _nullptr_ 的调用就不会有这种问题。当 _nullptr_ 被传递给 _lockAndCall_ 时，_ptr_ 的类型是被推导为 _std::nullptr_t_ 的。当 _ptr_ 被传递给 _f3_ 时，会有从 _std::nullptr_t_ 到  _Widget*_ 的隐式转换，因为 _std::nullptr_t_ 可以隐式转为所有指针类型。

模板的类型推导会推导出 _0_ 和 _NULL_ 所对应的 **_错误_** 的类型，即为：它们的真实类型，而不是表示为空指针的备选方案，当你想要引用一个空指针时，这是你使用 _nullptr_ 来代替 _0_ 和 _NULL_ 的最有说服力的理由。使用 _nullptr_ 不会给模板带来特殊的挑战。再结合 _nullptr_ 不会遭遇 _0_ 和 _NULL_ 容易遭遇的重载决议问题的事实的话，那么结论就无懈可击了。当你想要引用空指针时，使用 _nullptr_，而不是 _0_ 或 _NULL_。

### 需要记住的规则

* 首选 _nullptr_ 而不是 _0_ 和 _NULL_。
* 避免重载 _integral_ 类型和指针类型。

## _Item 9_ 首选 _alias declaration_ 而不是 _typedef_

我相信我们都同意使用 _STL_ 的 _container_ 是一个好注意，我也希望 [_Item 18_](Chapter%204.md#item-18-对于-exclusive-ownership-的资源管理使用-stdunique_ptr) 可以说服你使用 _std::unique_ptr_ 是一个好主意。而且我猜测我们都不喜欢写像 _std::unique_ptr&lt;std::unordered_map&lt;std::string, std::string&gt;&gt;_ 这样的类型多过于一次。仅是想想这样的类型就可能会增加患上 _carpal tunnel syndrome_ 的风险。

避免这样的医学悲剧是简单的。引入了 _typedef_：
```C++
  typedef
    std::unique_ptr<std::unordered_map<std::string, std::string>>
    UPtrMapSS;
```  
但是，_typedef_ 太 _C++98_ 了。_typedef_ 可以在 _C++11_ 中工作，但是 _C++11_ 也提供了 _alias declaration_：
```C++
  using UPtrMapSS =
    std::unique_ptr<std::unordered_map<std::string, std::string>>;
```

鉴于 _typedef_ 和 _alias declaration_ 实际上做的是相同的事情，所以就有理由怀疑是否有足够的技术理由来让我们喜欢其中的一个。

是存在的，但是在我说之前，我想说下：当处理涉及到函数指针的类型时，很多人发现 _alias declaration_ 是容易理解的：
```C++
  // FP is a synonym for a pointer to a function taking an int and
  // a const std::string& and returning nothing
  typedef void (*FP)(int, const std::string&);    // typedef

  // same meaning as above
  using FP = void (*)(int, const std::string&);   // alias
                                                  // declaration
```  
当然，这两种形式都不是那么容易理解的，因为很少有人愿意花时间来处理函数指针类型的同义词，所以这不是选择 _alias declaration_ 而不是 _typedef_ 的最有说服力的理由。


但是，最有说服力的理由是存在的，那就是模板。特别是，_alias declaration_ 是可以被模板化的，我们将模板化的 _alias declaration_ 称之为 _alias template_，而 _typedef_ 是不可以被模板化的。这为 _C++11_ 的程序员提供了一个直接的机制来表达那些在 _C++98_ 中必须要通过嵌套在模板化的 _struct_ 中的 _typedef_ 才能拼凑出来的东西。例如：考虑一个使用了定制 _allocator_ 的 _MyAlloc_ 的 _linked list_ 的同义词的定义。当在使用 _alias template_ 时，这就是小菜一碟：
```C++
  template<typename T>                            // MyAllocList<T>
  using MyAllocList = std::list<T, MyAlloc<T>>;   // is synonym for
                                                  // std::list<T,
                                                  // MyAlloc<T>>

  MyAllocList<Widget> lw;                         // client code
```  
而使用 _typedef_ 时，你几乎必须从头开始创建蛋糕：
```C++
  template<typename T>                            // MyAllocList<T>::type
  struct MyAllocList {                            // is synonym for
    typedef std::list<T, MyAlloc<T>> type;        // std::list<T,
  };                                              // MyAlloc<T>>
  
  MyAllocList<Widget>::type lw;                   // client code
```  
这会更糟。如果你想要在模板类中使用 _typedef_ 来创建一个持有模板形参所指定的类型的对象的 _linked list_ 的话，那么你必须在 _typedef_ 前加上 _typename_：
```C++
  template<typename T>
  class Widget {                                  // Widget<T> contains
  private:                                        // a MyAllocList<T>
    typename MyAllocList<T>::type list;           // as a data member
    …
  };
```  
此处，_MyAllocList&lt;T&gt;::type_ 引用了一个依赖于模板类型形参 _T_ 的类型。因此它是一个 _dependent type_。_C++_ 的众多的 **_可爱的_** 规则的其中一个便是必须在 _dependent type_ 的名字前加上 _typename_。

如果 _MyAllocList_ 是被定义来做为 _alias template_ 的话，那么就不需要 _typename_ 了，繁琐的后缀_::type_ 也不需要了：
```C++
  template<typename T>
  using MyAllocList = std::list<T, MyAlloc<T>>;   // as before
  
  template<typename T>
  class Widget {
  private:
    MyAllocList<T> list;                          // no "typename",
    …                                             // no "::type"
  };
```

对于你来说，使用了 _alias template_ 的 _MyAllocList&lt;T&gt;_ 和使用了嵌套的 _typedef_ 的 _MyAllocList&lt;T&gt;::type_ 看起来可能都是依赖于模板形参 _T_ 的，但是你不是编译器。当编译器处理 _Widget_ 模板和遇到 _MyAllocList&lt;T&gt;_，即为：使用 _alias template_ 时，编译器是知道 _MyAllocList&lt;T&gt;_ 是一个类型的名字的，因为它是一个 _alias template_，所以它必然命名了一个类型。因此 _MyAllocList&lt;T&gt; 是一个 _non-dependent type_，所以 _typename spcifier_ 就既不被需要也不被允许了。

另一方面，当编译器在 _Widget_ 模板中看到 _MyAllocList&lt;T&gt;::type_ 也就是使用了嵌套的 _typedef_ 时，编译器不能确定 _MyAllocList&lt;T&gt;::type_ 命名了一个类型，因为还可能存在编译器所看不到的 _MyAllocList_ 的 _specialization_，在其中 _MyAllocList&lt;T&gt;::type_ 引用的不是一个类型。这听起来是疯狂的，但是对于这个可能性不要怪编译器。人们真能写出这样的代码。  

例如，一些被误导的灵魂可能已经写了像下面这样的代码：  
```C++
  class Wine { … };

  template<>                            // MyAllocList specialization
  class MyAllocList<Wine> {             // for when T is Wine
  private:
    enum class WineType                 // see Item 10 for info on
    { White, Red, Rose };               // "enum class"

    WineType type;                      // in this class, type is
    …                                   // a data member!
  };
```  
正如你看到的，_MyAllocList&lt;Wine&gt;::type_ 不是类型。如果 _Widget_ 是被 _Wine_ 所实例化的话，那么 _Widget_ 模板中的 _MyAllocList&lt;Wine&gt;::type_ 引用的就是数据成员，而不是一个类型。在 _Widget_ 模板中的 _MyAllocList&lt;T&gt;::type_ 是否引用类型是老老实实地依赖于 _T_ 是什么的，这也是为什么编译器坚持要在类型前面加上 _typename_ 来声明它是类型。

如果你做过任意一点点模板元编程 _TMP_ 的话，那么你几乎肯定遇到过这样的需求：获取模板类型形参，然后根据它来创建修改过的类型。例如：给定一个类型 _T_，你可能想要剥离 _T_ 所包含的 _const-qualifier_ 或者 _reference-qualifier_，比如：你可能想要将 _const std::string&_ 转换为 _std::string_。或者想要在类型前添加 _const_ 或将其转换为左值引用，比如：将 _Widget_ 转换为 _const Widget_ 或 _Widget&_。如果你没有接触过一点 _TMP_，那就太糟糕了，因为如果你想要成为一位真正地高效率的 _C++_ 程序员的话，那么你需要至少熟悉 _C++_ 的这部分的基础。你可以去看 [_Item 23_](Chapter%205.md#item-23-理解-stdmove-和-stdforward) 和 [_Item 27_](Chapter%205.md#item-27-熟悉重载-univeral-reference-的替代方法) 中的 _TMP_ 的实战例子，包括我刚才提到的那几种类型转换。

_C++_ 给了你工具可以按照 _type trait_ 的形式来执行那几种类型转换，_type trait_ 是在头文件 _&lt;type_trsits&gt;_ 中的一些模板。在这个头文件中有很多 _type trait_，它们都提供了预期的接口，但不是都执行的是类型转换。给定一个类型 _T_，并且你想要对这个类型 _T_ 应用转换，所导致的类型就是 _std::transformation&lt;T&gt;::type_ 了。例如：  
```C++
  std::remove_const<T>::type            // yields T from const T
  std::remove_reference<T>::type        // yields T from T& and T&&
  std::add_lvalue_reference<T>::type    // yields T& from T
```  
注释只是总结了这些转换做了什么，不要太咬文嚼字。我知道在项目上使用它们之前，你会查询精确的规格说明文档。

总之，此处我的目的不是给你 _type trait_ 的 _tutorial_。而是要注意这些转换的应用都需要在每次使用时写 _::type_。如果你想要在模板中应用这些转换到类型形参上的话，在实际的代码中几乎总是这样使用，那么在每次使用时你也必须都加上 _typename_。这两个语法减速带存在的原因是 _C++11_ 的 _type trait_ 是用通过嵌套在模板化的 _struct_ 中的 _typedef_ 来实现的。是的，是使用我一直试图说服你的比 _alias template_ 要差的类型同义词技术来实现的！

这是有历史原因的，但是我们不谈论它，因为它非常傻，我保证，因为 _Standardization Committee_ 后来意识到了 _alias template_ 是更好的方式，并在 _C++14_ 中为 _C++11_ 的类型转换提供了相应的模板。_alias_ 有通用的形式：对每一个 _C++11_ 的转换 _std::transformation&lt;T&gt;::type_ 都所有对应的被命名为 _std::transformation_t_ 的 _C++14_ 的 _alias template_。例子会说明我的意思： 
```C++
  std::remove_const<T>::type            // C++11: const T → T
  std::remove_const_t<T>                // C++14 equivalent
  
  std::remove_reference<T>::type        // C++11: T&/T&& → T
  std::remove_reference_t<T>            // C++14 equivalent
  
  std::add_lvalue_reference<T>::type    // C++11: T → T&
  std::add_lvalue_reference_t<T>        // C++14 equivalent
```

_C++11_ 所对应的方法在 _C++14_ 中仍然有效，但是我不知道你为什么还要使用它们。即使你不能使用 _C++14_，自己写 _alias template_ 也是非常简单的。只需要 _C++11_ 的语言特性，甚至孩子们也可以模仿这种模式，对吧？如果你有 _C++14_ 标准的电子版的副本的话，仍然是简单的，因为只需要一些复制粘贴。在此处，我帮你开始吧：  
```C++
  template <class T>
  using remove_const_t = typename remove_const<T>::type;
  
  template <class T>
  using remove_reference_t = typename remove_reference<T>::type;
  
  template <class T>
  using add_lvalue_reference_t =
    typename add_lvalue_reference<T>::type;
```  
看到了吧，非常简单。

### 需要记住的规则

* _typedef_ 不支持模板化，而 _alias declaration_ 支持。
* _alias template_ 可以避免 _::type_ 后缀，并且在模板中引用 _typedef_ 时常常需要加上 _typename_ 前缀。
* _C++14_ 为 _C++11_ 的 _type trait_ 转换都提供了所对应的 _alias temmplate_。

## _Item 10_ 首选 _scoped enum_ 而不是 _unscoped enum_

一般而言，在 _{}_ 内声明一个名称会将这个名称的可见性限制在 _{}_ 所定义的作用域内。对于使用 _C++98-style_ 的 _enum_ 所声明的 _enumerator_ 来说却不是这样。这些 _enumerator_ 的名称是属于那个包含着它所对应的 _enum_ 的作用域的。这意味着在这个作用域内不能有相同的名称：
```C++
  enum Color { black, white, red };     // black, white, red are
                                        // in same scope as Color

  auto white = false;                   // error! white already
                                        // declared in this scope
```

这些 _enumerator_ 的名称是被泄露到了那个包含着它所对应的 _enum_ 的作用域中了的事实产生了这种 _enum_ 所对应的官方术语：_unscoped_。_C++11_ 有所对应的 _scoped enum_，不会发生泄露：
```C++
  enum class Color { black, white, red };         // black, white, red
                                                  // are scoped to Color
  
  auto white = false;                             // fine, no other
                                                  // "white" in scope

  Color c = white;                                // error! no enumerator named
                                                  // "white" is in this scope

  Color c = Color::white;                         // fine

  auto c = Color::white;                          // also fine (and in accord
                                                  // with Item 5's advice)
```

因为 _scoped enum_ 是通过 _enum class_ 来声明的，所以有时候称为 _enum class_。 

_scoped enum_ 降低了 _namespace_ 的污染，这个就足以选择 _scoped enum_ 而不是 _unscoped enum_ 了，但是还有第二个有说服力的优势：它们的 _enumerator_ 是更 _strongly typed_。_unscoped enum_ 所对应的 _enumerator_ 是可以被隐式转换为 _integral_ 类型的，并是能够进一步被转换为 _floating-point_ 类型的。因此，像下面这样的扭曲语义完全是有效的：  
```C++
  enum Color { black, white, red };     // unscoped enum

  std::vector<std::size_t>              // func. returning
    primeFactors(std::size_t x);        // prime factors of x
  
  Color c = red;
  …

  if (c < 14.5) {                       // compare Color to double (!)
    auto factors =                      // compute prime factors
      primeFactors(c);                  // of a Color (!)
    …
  }
```

然而，在 _enum_ 之后加一个简单的 _class_ 将 _unscoped enum_ 转换为 _scoped enum_ 后，就是不同的故事了。不可以将 _scoped enum_ 所对应的 _enumerator_ 隐式转换为到其他类型：
```C++
  enum class Color { black, white, red };         // enum is now scoped

  Color c = Color::red;                           // as before, but
  …                                               // with scope qualifier

  if (c < 14.5) {                                 // error! can't compare
                                                  // Color and double
    auto factors =                                // error! can't pass Color to
      primeFactors(c);                            // function expecting std::size_t
    …
  }
```

如果真地想要执行从 _Color_ 到不同的类型的转换，去做你一直做的，去扭曲类型系统来满足你那肆意的欲望：使用 _cast_：
```C++
  if (static_cast<double>(c) < 14.5) {            // odd code, but
                                                  // it's valid

    auto factors =                                // suspect, but
      primeFactors(static_cast<std::size_t>(c));  // it compiles
    …
  }
```

_scoped enum_ 相对于 _unscoped enum_ 似乎还有的第三个优势，因为 _scoped enum_ 是可以进行前置声明的，即为：在没有指明 _enumerator_ 的时候，就可以声明 _scoped enum_：
```C++
  enum Color;                 // error!

  enum class Color;           // fine
```  
这有点误导人。在 _C++11_ 中，_unscoped enum_ 也是可以进行前置声明的，但是需要一点额外的工作。这点额外的工作源于这样的事实：在 _C++_ 中每个 _enum_ 都有一个编译器所决定的 _underlying type_，是 _integral_ 的。对于像下面这样的 _Color_ 的 _unscoped enum_，  
```C++
  enum Color { black, white, red };
```  
编译器可能选择 _char_ 来做为 _underlying type_，因为只需要三个值来表示。然而，一些 _enum_ 有一组更大范围的值，比如：  
```C++
  enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                indeterminate = 0xFFFFFFFF
              };
```  
此处所表示的范围是 _0_ 到 _0xFFFFFFFF_。除了不常见的机器外，这些机器的 _char_ 至少有 _32bit_，编译器都将会选择大于 _char_ 的 _integral_ 类型来表示 _Status_ 值。

为了高效率的使用内存，编译器通常想要为 _enum_ 来选择最小的 _underlying type_，只要能足够表示 _enumerator_ 的值的范围就可以了。在一些场景中，编译器优化的是速度而不是大小，在这些场景中，编译器可能不会选择所允许的最小的 _underlying type_，但是仍然想要能够优化大小。为了可以这样做，_C++98_ 就只提供了 _enum_ 定义，就是要列出全部的 _enumerator_，而 _enum_ 声明是被不允许的。这使得编译器可以在每个 _enum_ 被使用之前来为它们来选择一个 _underlying type_ 了。

但是无法前置声明 _enum_ 也有缺点。最明显的大概就是增加了编译依赖。再一次考虑 _Status_ _enum_：
```C++
  enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                indeterminate = 0xFFFFFFFF
              };
```   
这种类型的 _enum_ 可能会被整个系统所使用，它会被放在一个头文件中，而系统中的每一个部分都包含有这个头文件。如果新的状态值被引入的话：  
```C
 enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                audited = 500,
                indeterminate = 0xFFFFFFFF
              };                   
```  
整个系统都必须被重新编译。即使只有一个单独的子系统中的一个单独的函数使用了这个的新的 _enumerator_ 而已。人们都非常讨厌这种事情。在 _C++11_ 中可以前置声明 _enum_ 来避免了这样的事情。例如：有一个 _scoped enum_ 的完美有效的声明和一个函数，这个函数的形参的类型是这个 _scoped enum_：
```C++
  enum class Status;                    // forward declaration
  
  void continueProcessing(Status s);    // use of fwd-declared enum
```

如果 _Status_ 的定义被更改了的话，那么包含有这个声明的头文件是不需要被重新编译的。此外，如果 _Status_ 被更改了，比如：增加了一个 _audited_ _enumerator_ ，但是 _continueProcessing_ 的行为未被影响的话，比如：它并没有使用到 _audited_，那么 _continueProcessing_ 的实现也是不需要被重新编译的。

但是，如果编译器需要在一个 _enum_ 被使用之前就必须得要知道这个 _enum_ 的大小的话，那么为什么 _C++11_ 可以进行前置声明而 _C++98_ 却不可以呢？答案是简单的：_scoped enum_ 所对应的的 _underlying type_ 总是已知的，而对于 _unscoped enum_ 所对应的的 _underlying type_ 则不是，但是你是可以指定的。

_scoped enum_ 所对应的 _underlying type_ 默认是 _int_：  
```C++
  enum class Status;          // underlying type is int
```  
如果默认类型不适合你的话，那么你可以重写它：
```C++
  enum class Status: std::uint32_t;     // underlying type for
                                        // Status is std::uint32_t
                                        // (from <cstdint>)
```  

不管怎样，编译器是知道 _scoped enum_ 中的 _enumerator_ 的大小的。

为了指明 _unscoped enum_ 所对应的的 _underlying type_，需要和 _scoped enum_ 做同样的事情，那么这样就可以进行前置声明了：
```C++
  enum Color: std::uint8_t;   // fwd decl for unscoped enum;
                              // underlying type is
                              // std::uint8_t
```  

_underlying type_ 的规格也可以放在 _enum_ 的定义中：
```C++
  enum class Status: std::uint32_t { good = 0,
                                      failed = 1,
                                      incomplete = 100,
                                      corrupt = 200,
                                      audited = 500,
                                      indeterminate = 0xFFFFFFFF
                                    };
```

因为考虑到 _scoped enum_ 可以避免 _namespace_ 的污染而且不容易受到无意间的隐式类型转换的影响，所以当你听到 _unscoped enum_ 还有有用的使用情景时，你可能是会感到惊讶的。这个有用的使用情景就是当引用 _C++11_ 中的 _std::tuples_ 的各个域时。例如：假定我们有一个持有名字、邮箱地址和用户在社交网站上的信誉值的 _tuple_：
```C++
  using UserInfo =            // type alias; see Item 9
    std::tuple<std::string,   // name
                std::string,  // email
                std::size_t>; // reputation
```

尽管注释指明了 _tuple_ 的每个域都代表了什么，但是当你在独立源文件中遇到下面这样的代码时，可能也不是非常有帮助的：
```C++
  UserInfo uInfo;                       // object of tuple type
  …
  
  auto val = std::get<1>(uInfo);        // get value of field 1
```  
做为一个程序员，你有很多需要关注的东西。你真需要记住域 _1_ 对应的是用户的邮箱地址吗？我不这样认为。使用 _unscoped enum_ 来将名称和域号关联起来就可以避免去记住域号代表的是什么：
```C++
  enum UserInfoFields { uiName, uiEmail, uiReputation };

  UserInfo uInfo;                                           // as before
  …
  
  auto val = std::get<uiEmail>(uInfo);                      // ah, get value of
                                                            // email field
```

从 _UserInfoFields_ 到 _std::size_t_ 的隐式转换可以让上面的代码正确地工作，_std::size_t_ 是 _std::get_ 所需要的类型。

相对应的使用了 _scoped enum_ 的代码要冗长的多：
```C++
  enum class UserInfoFields { uiName, uiEmail, uiReputation };
  
  UserInfo uInfo;             // as before
  …

  auto val =
    std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>
      (uInfo);
```
可以通过写一个持有 _enumerator_ 并且返回相应的 _std::size_t_ 的值的函数来降低这种冗长，但是会有点棘手。因为 _std::get_ 是一个模板，你提供的值是模板实参，注意是 _&lt;&gt;_ 而不是 _()_，所以这个将 _enumerator_ 转换为 _std::size_t_ 的函数必须在 _编译期间_ 就得要生成它的结果。就正如  [_Item 15_](#item-15-只要有可能就使用-constexpr) 所解释的，这意味着这个函数必须是 _constexpr_ 函数。

事实上，这个函数应该是 _constexpr_ 的函数模板，因为它应该能和各种 _enum_ 一起工作。如果我们做了泛化的话，那么我们也应该泛化它的返回类型。我们应该要返回的是这个 _enum_ 的 _underlying type_ 而不是 _std::size_t_。关于 _type trait_ 见 [_Item 9_](#item-9-首选-alias-declaration-而不是-typedef)。最后，我们声明它为 _noexcept_，见 [_Item 14_](#item-14-当函数不会抛出异常时声明函数为-noexcept)，因为我们知道它永远不会产生异常。我们构建一个函数模板 _toUType_，它持有一个任意的 _enumerator_ 并且按照编译期间常量的形式返回其值：
```C++
  template<typename E>
  constexpr typename std::underlying_type<E>::type
    toUType(E enumerator) noexcept
  {
    return
      static_cast<typename
        std::underlying_type<E>::type>(enumerator);
  }
```

在 _C++14_ 中，使用更简洁的 _std::underlying_type_t_ 来代替 _typename std::underlying_type<E>::type_ 可以使 _toUType_ 更简化，见 [_Item 9_](#item-9-首选-alias-declaration-而不是-typedef)：
```C++
  template<typename E>                                      // C++14
  constexpr std::underlying_type_t<E>
    toUType(E enumerator) noexcept
  {
    return static_cast<std::underlying_type_t<E>>(enumerator);
  }
```

在 _C++14_ 中，更简洁的 _auto_ 返回类型也是有效的，见 [_Item 3_](Chapter%201.md#item-3-理解-decltype)：
```C++
  template<typename E>                                      // C++14
  constexpr auto
    toUType(E enumerator) noexcept
  {
    return static_cast<std::underlying_type_t<E>>(enumerator);
  }
  ```

不管如何写，反正 _toUType_ 允许我们像下面这样访问 _tuple_ 的域：
```C++
  auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

这仍然比使用 _unscoped enum_ 的方式写的要多，但是可以避免 _namespace_ 的污染和避免涉及到 _enumerator_ 时的不经意间的转换。在很多的场景下，你都会同意：为了避免使用一种源自数字通信的艺术状态仅为 _2400-baud_ 调制解调器时期的 _enum_ 的技术的陷阱，多打几个字符是一个合理的代价。

### 需要记住的规则

* _C++98-style_ 的 _enum_ 现在被称为 _unscoped enum_。
* _scoped enum_ 的 _enumerator_ 只在 _enum_ 内可见。使用 _cast_ 才能将 _scoped enum_ 所对应的 _enumerator_ 转换为其他的类型。
* _scoped enum_ 和 _unscoped enum_ 都支持指定 _underlying type_。_scoped enum_ 所对应的 _underlying type_ 默认是 _int_。_unscoped enum_ 所对应的 _underlying type_ 没有默认类型。
* _scoped enum_ 可以前置声明。_unscoped enum_ 只有当指明了 _underlying type_ 才可以进行前置声明。

## _Item 11_ 首选 _deleted function_ 而不是 _private undefined function_

如果你是提供代码给到其他开发者，并且你想要阻止他们调用一个特定的函数的话，那么一般的方法是不声明这个函数。没有函数声明，就没有函数调用。这很简单。但是，有时候 _C++_ 会为你声明一些函数，而如果你想要阻止客户去调用这些函数的话，就不是那么简单了。

只有 **_特殊的成员函数_** 才会出现这个情况，这些函数只有当它们被需要的时候，_C++_ 才会自动地生成。[_Item 17_](#item-17-理解特殊成员函数的生成) 详细地讨论了这些函数，但是现在，我们仅仅关心 _copy constructor_ 和 _copy assignment operator_。本节主要致力于那些已经被 _C++11_ 中的更好的实践所取代的 _C++98_ 中的常见的实践，在 _C++98_ 中如果你想要禁止成员函数的话，那么几乎总是 _copy constructor_ 和 _copy assignment operator_。

_C++98_ 阻止使用这些函数的方法是声明这些函数为 _private_ 并且不要进行定义。例如：_C++_ 标准库的 _iostream_ 的基类是类模板 _basic_ios_。_istream_ 和 _ostream_ 都是继承自可能是直接继承自 _basic_ios_ 的。拷贝 _istream_ 和 _ostream_ 是不想要的，因为真的不知道这些操作应该怎么去做。例如：一个 _istream_ 表示的是输入值的 _stream_，其中有的已经读  取过了，而有的稍后可能会读取。如果要拷贝一个 _istream_ 的话，那么需要拷贝已经读取过的和稍后会读取的全部的值吗？处理这个问题的最简单的方法就是让它们不存在。这正是禁止拷贝 _stream_ 这件事要做的。

为了让 _istream_ 和 _ostream_ 是不可拷贝的，_C++98_ 中的 _basic_ios_ 被指定为下面这样：
```C++
  template <class charT, class traits = char_traits<charT> >
  class basic_ios : public ios_base {
  public:
    …

  private:
    basic_ios(const basic_ios& );                 // not defined
    basic_ios& operator=(const basic_ios&);       // not defined
  };
```

声明这些函数为 _private_ 可以阻止客户调用到他们。而故意不去定义他们则意味着：如果仍然有代码调用这些函数的话，即为：类的成员函数或者友元函数，那么会因为没有函数定义而链接出错。

在 _C++11_ 中，有更好的方法去实现这个：使用 _= delete_ 去标记 _copy constructor_ 和 _copy assignment operator_ 为 _deleted function_。下面是 _C++11 中 _basic_ios_：  
```C++
  template <class charT, class traits = char_traits<charT> >
  class basic_ios : public ios_base {
  public:
    …
    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
  };
```

删除函数和声明函数为 _private_ 的区别似乎更多是时尚问题，而非其他，但其实比你想的要更多。_deleted function_ 不能以任何方式被使用，成员函数和友元函数中的代码尝试去拷贝 _basic_ios_ 时也会编译失败。这是相对于 _C++98_ 行为的提升，在 _C++98_ 中直到链接时才能诊断出错误。

按照惯例，_deleted function_ 是应该被声明为 _public_ 而不是 _private_ 的。这是有原因的。因为当客户尝试去使用 _deleted private function_ 时，一些编译器只会抱怨函数是 _private_ 的，即使函数的可访问性并不真正影响它是否能使用。当修改 _legacy_ 代码使用 _deleted member function_ 去代替 _private-and-not-defined member function_ 时，尤其要注意这点，因为声明为 _public_ 一般会得到更好的错误信息。  

_deleted function_ 的一个重要优势是任何函数都可以被删除，而只有成员函数是可以为 _private_。例如：假定我们有一个持有 _integer_ 并且返回它是否为幸运数字的函数：
```C++
  bool isLucky(int number);
```

_C++_ 的 _C_ 的遗产意味着几乎任何可以被模糊地视为数字类型的数据都可以被隐式转换为 _int_ ，但是一些可以编译的调用却可能是不合理的：
```C++
  if (isLucky('a')) …         // is 'a' a lucky number?

  if (isLucky(true)) …        // is "true"?
  
  if (isLucky(3.5)) …         // should we truncate to 3
                              // before checking for luckiness?
```

如果幸运数字必须是 _integer_ 的话，那么我们想要阻止这样的调用被编译。

一种可以完成的方法是去创建我们想要过滤的类型所对应的 _deleted_ 的重载函数：
```C++
  bool isLucky(int number);             // original function
  
  bool isLucky(char) = delete;          // reject chars
  
  bool isLucky(bool) = delete;          // reject bools
  
  bool isLucky(double) = delete;        // reject doubles and
                                        // floats
```  
_double_ 重载函数的注释说 _double_ 和 _float_ 都是会被拒绝的，这可能会让你惊讶，但是的惊讶很快会消失，因为你想起了：对于 _float_ 转换为 _int_ 还是 _double_ 的选择，_C++_ 首选的会是 _double_。因此使用 _float_ 调用 _isLucky_ 所调用会是 _double_ 重载函数，而不是 _int_ 重载函数。是的，它会尝试去调用。相关的重载函数被删除的事实将会阻止这个调用被编译。

即使 _deleted function_ 不能被使用了，但它仍然会是你程序的一部分。因此，在重载决议期间它们是仍然会被考虑到的。这也是在使用了上面的 _deleted function_ 后不想要的 _isLucky_ 调用会被拒绝的原因：
```C++
  if (isLucky('a')) …         // error! call to deleted function
  
  if (isLucky(true)) …        // error!
  
  if (isLucky(3.5f)) …        // error!
```

另一个 _deleted function_ 可以执行而 _private member function_ 不可以执行的技巧是阻止使用那些应该被删除的模板实例。例如：假如你需要一个使用了内建指针的模板。尽管 [_Chapter 4_](Chapter%204.md#chapter-4-智能指针) 建议你首选智能指针而不是原始指针：
```C++
  template<typename T>
  void processPointer(T* ptr);
```  
在指针的世界里有两个特殊的场景。一个是 _void*_ 指针，因为没有方法解引用它、去增加它或者减少它等。另一个是 _char*_ 指针，因为它们一般代表的是 _C-style_ 字符串的指针，而不是一个单独字符的指针。这两个特殊的场景经常需要被特殊处理，在 _processPointer_ 模板的场景中，我们假设有一个合适的处理是拒绝这两种类型的调用。也就是说不能调用 _void*_ 或 _char*_ 所对应的 _processPointer_。

非常容易实施。删除这两个实例就好了：
```C++
  template<>
  void processPointer<void>(void*) = delete;
  
  template<>
  void processPointer<char>(char*) = delete;  
```

现在，如果调用 _void*_ 或 _char*_  所对应的 _processPointer_ 需要是无效的话，那么调用 _const void*_ 或 _const char*_  所对应的 _processPointer_ 也需要是无效的，所以下面这些实例通常也需要被删除：
```C++
  template<>
  void processPointer<const void>(const void*) = delete;
  
  template<>
  void processPointer<const char>(const char*) = delete;
```

如果你真的想要彻底进行的话 ，那么你也需要删除 _const volatile void*_ 和 _const volatile char*_ 重载函数，还有其他标准字符类型指针的重载函数，比如：_std::wchar_t_、_std::char16_t_ 和 _std::char32_t_。

有趣地是，如果在类中有一个函数模板，并且你想通过 _private_ 声明来 _disable_ 它的一些实例的话，那么是不可以的，这是经典的 _C++98_ 的习惯做法，因为不能给成员函数模板的特化一个和主模板是不同的访问级别。例如：如果 _processPointer_ 是 _Widget_ 的成员函数模板，并且你想要 _disable_ _void*_  指针的调用的话，下面这是 _C++98_ 的方法，尽管它不能通过编译：
```C++
  class Widget {
  public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }

  private:
    template<>                          // error!       
    void processPointer<void>(void*);
 };
```  
这是因为模板特化必须被写在 _namespace_ 作用域内，而不是类作用域内。对于 _deleted function_ 不会发生这种问题，因为它们不需要不同的访问级别。它们可以在类外被删除，因为是在 _namespace_ 作用域内的：  
```C++
class Widget {
public:
 …
 template<typename T>
 void processPointer(T* ptr)
 { … }
 …
};
template<>                                                  // still
void Widget::processPointer<void>(void*) = delete;          // public,
                                                            // but
                                                            // deleted
```

事实上，_C++98_ 声明函数为 _private_ 且不去定义它们的这种做法就是试图去实现 _C++11_ 的 _deleted function_ 的功能的。就像是模仿，_C++98_ 的方法就是不如 _C++11_ 的方法好的。因为 _C++98_ 的方法不能应用于类外的函数，也不是总能够应用于类内的函数，当可以应用时，也是在链接时才能工作。所以还是坚持使用 _deleted function_ 吧。

### 需要记住的规则

* 首选 _deleted function_ 而不是 _private undefined function_。
* 任意函数都可以被删除，包括：非成员函数和模板实例。

## _Item 12_ 使用 _override_ 来声明重写函数

_C++_ 中的面向对象编程的世界是以类、继承以及 _virtual function_ 为中心的。在这个世界中的最基本的思想之一是 _derived class_ 中的 _virtual function_ 的实现会重写它所对应的 _base class_ 中的 _virtual function_ 的实现。意识到了重写 _virtual function_ 有这么容易出错，是令人沮丧的。语言的这部分似乎是按照墨菲定律的思想来设计的，而且不仅是要遵守，还是要致敬墨菲定律的。

因为 _overriding_ 听起来像是 _overloading_ 的，但其实是完全不相关的，所以让我澄清下：重写 _virtual function_ 只是使得可以通过 _base class_ 的接口来执行 _derived class_ 的函数了而已：
```C++
  class Base {
  public:
    virtual void doWork();               // base class virtual function
    …
  };

  class Derived: public Base {
  public:
    virtual void doWork();              // overrides Base::doWork
    …                                   // ("virtual" is optional
  };                                    // here)

  std::unique_ptr<Base> upb =           // create base class pointer
    std::make_unique<Derived>();        // to derived class object;
                                        // see Item 21 for info on
  …                                     // std::make_unique

  upb->doWork();                        // call doWork through base
                                        // class ptr; derived class
                                        // function is invoked
```  
要发生重写，必须要满足一些要求：  
* _base class_ 的函数必须是 _virtual function_。
* _base function_ 和 _derived function_ 的名字必须一致，除了析构函数的场景以外。
* _base function_ 和 _derived function_ 的形参类型必须一致。
* _base function_ 和 _derived function_ 的 _constness_ 必须一致。
* _base function_ 和 _derived function_ 的返回类型和异常规范必须要兼容。

这些限制也是 _C++98_ 的一部分，_C++11_ 又增加了一个：  

* 函数的 _reference qualifier_ 必须一致。成员函数的 _reference qualifier_ 是 _C++11_ 中较少宣传的特性之一，所以如果你没有听过它的话，那么也不要感到太惊讶。它可以限制成员函数只能用于左值或右值。成员函数并不需要是 _virtual function_ 才能使用它们稍后我会详细介绍带有 _reference qualifier_ 的成员函数，但是现在我们只需要了解：如果 _base class_ 所对应的 _virtual function_ 是有 _reference qualifier_ 的话，那么 _derived class_ 所对应的 _virtual function_ 也必须要有完全相同的 _reference qualifier_。如果不是完全相同的话，虽然所声明的函数仍然会在 _derived class_ 中，但是却不会重写 _base class_ 中的任何东西了。
```C++
  class Widget {
  public:
    …
    void doWork() &;          // this version of doWork applies
                              // only when *this is an lvalue
  
  void doWork() &&;           // this version of doWork applies
  };                          // only when *this is an rvalue
  
  …

  Widget makeWidget();        // factory function (returns rvalue)
  
  Widget w;                   // normal object (an lvalue)
  
  …
  
  w.doWork();                 // calls Widget::doWork for lvalues
                              // (i.e., Widget::doWork &)
  
  makeWidget().doWork();      // calls Widget::doWork for rvalues
                              // (i.e., Widget::doWork &&)
```  
全部这些对于重写的要求意味着小的错误可以造成大的不同。包含着重写错误的代码一般都是有效的，只是它的含义不是你所期望的。因此你不能依靠编译器去通知你是否发生了错误。例如：下面的代码完全合法，第一眼看时，还像是合理的，但是并没有包含 _virtual function_ 重写，没有一个 _derived class_ 函数绑定到 _base class_ 函数。你可以发现每一个例子中的问题吗？即为：为什么 _derived class_ 不能重写有着相同名字的 _base class_ 函数呢？  
```C++
  class Base {
  public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
  };

  class Derived: public Base {
  public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
  };
```

需要帮助吗？  
* _mf1_ 在 _Base_ 中有 _const_，但在 _Derived_ 中却没有。
* _mf2_ 在 _Base_ 中持有的是 _int_，但在 _Derived_ 中持有的却是 _unsigned int_。
* _mf3_ 在 _Base_ 中是 _lvalue-qualified_，但是在 _Derived_ 中却是 _rvalue-qualified_。
* _mf4_ 在 _Base_ 中不是 _virtual function_。

你可能会想“嘿，在实践中，这些都会引发编译器警告，所以我不需要担心。”可能是的，也可能不是。我测试过两个编译器，它们不会发出抱怨，这个代码是会被接受的，这还是在全部警告都开启的情况下。其他的编译器提供了这些问题的警告，但也不全。

因为声明 _derived class_ 的函数是重写的是非常重要的，所以 _C++11_ 提供了一个方法，可以显式地让 _derived class_ 的函数重写 _base class_ 的函数：
```C++
  class Derived: public Base {
  public:
  virtual void mf1() override;
  virtual void mf2(unsigned int x) override;
  virtual void mf3() && override;
  virtual void mf4() const override;
  };
```  
当然，这是不可以通过编译的，因为当这样写时，编译器会抱怨所有与重写相关的问题。这完全就是你所想要的，这也是为什么你应该声明所有的重写函数为 _override_。

使用 _override_ 的可以通过编译的代码看起来就像下面这样的，假设要重写 _Base_ 中的所有 _virtual function_：  
```C++
  class Base {
  public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
  };
  class Derived: public Base {
  public:
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override;          // adding "virtual" is OK,
  };                                    // but not necessary
```  
在这个例子中，让事情可以工作的一部分是将 _Base_ 中的 _mf4_ 声明为 _virtual function_。大多数与重写相关的错误都是在 _derived class_ 中，但是也可以出现在 _base class_ 中。

在你的 _derived class_ 上使用 _override_ 的策略不只是可以让编译器在你想要重写却没有重写时提醒你。它还可以在你想要打算改变 _base class_ 中的 _virtual function_ 的 _signatrue_ 时来帮助你评估后果。如果你在 _derived class_ 中的每一个地方都使用了 _override_ 的话，那么你可以通过改变 _base class_ 中的 _signatrue_ 且重新编译系统来看这会造成多大的破坏，即为：看看有多少个 _derived class_ 会因此而编译失败，然后再决定这个 _signatrue_ 是否值得改变。而在没有 _override_ 的情况下，你只能希望能有一个完备的单元测试了，因为正如我们已经看到的，应该重写但没有重写 _base class_ 的函数的 _derived class_ 的 _virtual function_ 不会引发编译器的诊断。

_C++_ 一直就是有关键字的，但是 _C++11_ 又引入了两个 _contextual_ 关键字：_override_ 和 _final_。这两个关键字有一个特点，那就是它们是被保留的，只在固定的上下文中被保留。在 _override_ 场景中，只有当 _override_ 出现在成员函数的末尾时，才会有所保留的含义。这意味着：如果你有已经使用了 _override_ 名称的 _legacy_ 代码的话，你不需要为了 _C++11_ 来改变它：  
```C++
  class Warning {             // potential legacy class from C++98
  public:
    …
    void override();          // legal in both C++98 and C++11
    …                         // (with the same meaning)
  };
```

这就是 _override_ 的全部了，但是关于成员函数的 _reference qualifier_ 还有要说的。在前面我就说过会提供更多关于它的信息，现在开始。

如果我们想要写一个只接收左值实参的函数的话，那么我们声明一个 _non-const_ 左值引用形参：
```C++
  void doSomething(Widget& w);          // accepts only lvalue Widgets
```  
如果我们想要写一个只接收右值实参的函数的话，那么我们声明一个右值引用形参：  
```C++
  void doSomething(Widget&& w);         // accepts only rvalue Widgets
```  
成员函数的 _reference qualifier_ 只是使得可以给调用成员函数的对象做区分了，即为：_*this_。类似于成员函数末尾的 _const_，这个 _const_ 表示调用成员函数的对象，即为 _*this_，必须得是 _const_ 的。

对于 _reference-qualified_ 成员函数的需求是不常见的，但也不是没有。例如：假定 _Widget_ 类有一个 _std::vector_ 的数据成员，而且我们提供了一个访问函数来让客户直接访问这个数据成员：
```C++
  class Widget {
  public:
    using DataType = std::vector<double>;          // see Item 9 for
    …                                             // info on "using"

    DataType& data() { return values; }
    …
    private:
      DataType values;
    };
```  

很难这是最有封装性的设计，但先不考虑封装性，考虑一下下面这样的客户代码会发生什么：  
```C++
  Widget w;
  …
  auto vals1 = w.data();      // copy w.values into vals1
```  
_Widget::data_ 的返回类型是左值引用，准确说是 _std::vector&lt;double&gt;&_，因为左值引用是被定义来为左值的，所以我们是根据一个左值来初始化的 _vals1_。因此 _vals1_ 是调用 _w.values_ 来进行拷贝构造的，正如注释所说的那样。  

现在假定我们有创建 _Widget_ 的工厂函数，  
```C++
  Widget makeWidget();
```  
而且我们想要使用 _makeWidget_ 所返回的 _Widget_ 的 _std::vector_ 来初始化一个变量：  
```C++
  auto vals2 = makeWidget().data();     // copy values inside the
                                        // Widget into vals2
```  
再一次，_Widget::data_ 返回一个左值引用，再一次，左值引用是左值，所以，再一次，我们的新对象 _vals2_ 是根据 _Widget_ 的 _values_ 而拷贝构造出的。但是，这一次这个 _Widget_ 是一个 _makeWidget_ 所返回的临时对象，即为：一个右值，所以拷贝 _std::vector_ 是浪费时间的。此时最好去移动它，但是，因为 _data_ 是一个返回的左值引用，所以 _C++_ 的规则会要求编译器生成拷贝的代码。通过所谓的 _as if rule_，是有一些优化的余地的，但不要指望编译器能找到利用它的方法。  

需要一种方法：当在右值 _Widget_ 上执行 _data_ 时，可以指定所产生的结果也是一个右值。使用 _reference qualifier_ 来重载左值 _Widget_ 和右值 _Widget_ 的 _data_ 就可以完成：  
```C+++
  class Widget {
  public:
    using DataType = std::vector<double>;
    …

    DataType& data() &                            // for lvalue Widgets,
    { return values; }                            // return lvalue

    DataType data() &&                            // for rvalue Widgets,
    { return std::move(values); }                 // return rvalue
    …
  private:
    DataType values;
  };
```  
注意 _data_ 重载函数的不同的返回类型。左值引用重载函数返回的是左值引用，即为：一个左值，而右值引用重载函数返回的是临时对象，即为：一个右值。这意味着客户代码现在按照我们想的那样执行了：  
```C++
  auto vals1 = w.data();                // calls lvalue overload for
                                        // Widget::data, copy-
                                        // constructs vals1

  auto vals2 = makeWidget().data();     // calls rvalue overload for
                                        // Widget::data, move-
                                        // constructs vals2
```  

这确实非常棒，但是不要让这个完美结尾的温暖光芒分散了本 _Item_ 的真正观点。真正的观点是：无论何时，只要你想在 _derived class_ 中声明一个重写了 _base class_ 中的 _virtual function_ 的函数时，就请声明函数为 _override_。

### 需要记住的规则

* 声明重写函数为 _override_。
* 成员函数的 _reference qualifier_ 使得可以不同地处理左值 _*this_ 对象和右值 _*this_ 对象了。

## _Item 13_ 首选 _const_iterator_ 而不是 _iterator_

_const_iterator_ 是 _STL_ 的 _pointer-to-const_ 的等价物。它们都指向不可以被更改的值。只要有可能使用 _const_ 就应该去使用 _const_ 的标准实践规定了：当你需要一个 _iterator_，而不需要更改 _iterator_ 所指向的内容时，你应该使用的是 _const_iterator_。

_C++98_ 和 _C++11_ 都是这样的，但是在 _C++98_ 中，对于 _const_iterator_ 的支持是不够全面的。在 _C++98_ 中，不太容易创建 _const_iterator_，就算你创建了一个，使用它的方法也是有限制的。例如：假定你想要在 _std::vector&lt;int&gt;_ 中搜索 _1983_ 出现的第一个位置，_1983_ 是 _C++_ 取代 _C with Classes_ 来做为编程语言名字的年份，然后在这个位置后面插入值 _1998_，这是第一个 _ISO C++ Standard_ 被采用的年份。如果在这个 _vector_ 中没有 _1983_ 的话，那个插入值应该放在这个 _vector_ 的末尾。在 _C++98_ 中使用 _iterator_ 来完成是非常简单的：  
```C++
  std::vector<int> values;
  
  …
  
  std::vector<int>::iterator it =
    std::find(values.begin(),values.end(), 1983);
  values.insert(it, 1998);
```  
但是在这里 _iterator_ 真的不是一个合适的选择，因为这个代码永远不会更改 _iterator_ 所指向的内容。更改代码去使用 _const_iterator_ 应该是容易的，但是在 _C++98_ 却不容易。这是一个在概念上合理但实际上却是不正确的方法：  
```C++
  typedef std::vector<int>::iterator IterT;                 // type-
  typedef std::vector<int>::const_iterator ConstIterT;      // defs
  
  std::vector<int> values;
  
  …
  
  ConstIterT ci =
    std::find(static_cast<ConstIterT>(values.begin()),      // cast
              static_cast<ConstIterT>(values.end()),        // cast
              1983);
  
  values.insert(static_cast<IterT>(ci), 1998);              // may not
                                                            // compile; see
                                                            // below
```

当然 _typedef_ 不是必须的，但是它是可以使得代码中的 _cast_ 更加容易地书写的。你可能会好奇为什么使用的是 _typedef_，而不是遵循 [_Item 9_](#item-9-首选-alias-declaration-而不是-typedef) 的建议去使用 _alias declaration_ 呢？因为这个例子展示的是 _C++98_ 的代码，这时还没有 _alias declaration_， _alias declaration_ 在 _C++11_ 新添加的特性。

在 _std::find_ 调用中存在有 _cast_，这是因为 _values_ 是一个 _non-const_ _container_，在 _C++98_ 中是没有简单的方法来获取 _non-const_ _container_ 的 _const_iterator_ 的。并不是非得要这样转换，因为可以使用其他的方法来获取 _non-const_ _container_ 的 _const_iterator_，比如：你可以首先使用 _reference-to-const_ 的变量来绑定 _values_，然后再在代码中使用这个变量来代替 _values_，但是不论是哪种方法，从 _non-const_ _container_ 中获取到 _const_iterator_ 的过程都经历了一些曲折。

而一旦你有了 _const_iterator_，事情常常会变得更糟，因为在 _C++98_ 中，插入和删除的位置只能通过 _iterator_ 来指定。_const_iterator_ 是能不被接受的。这也是为什么在上面的代码中我要将 _const_iterator_ 转换为 _iterator_，因为传递 _const_iterator_ 到 _insert_ 是不能通过编译的，而这个 _const_iterator_ 是从 _std::find_ 中非常小心得到的。

老实说，我展示的这个代码是不能通过编译的，因为没有 _const_iterator_ 到 _iterator_ 的可移植转换，使用 _static_cast_ 也不行。甚至使用被称为 _reinterpret_cast_ 的语义大招也是不行的。这不是 _C++98_ 的限制。在 _C++11_ 中也有一样的限制。_const_iterator_ 不能简单地转换为 _iterator_，不管这个转换看起来有多么地理所应当。是有一些可以生成指向 _const_iterator_ 所指向位置的 _iterator_ 的可移植方法的，但是这些方法不是明显的，也不是普遍适用的，所以不值得在本书中进行讨论。此外，我希望我目前的观点是清晰的：在 _C++98_ 中的开发者并不是只要有可能使用 _const_iterator_  就去使用 _const_iterator_ 的，而是只在适用的时候才去使用它，所以在 _C++98_ 中，_const_iterator_ 不是非常地适用。

全部的这些都在 _C++11_ 中发生了改变。现在 _const_iterator_ 非常容易被获取，也非常容易被用了。 _container_ 的成员函数 _cbegin_ 和 _cend_ 都产生的是 _const_iterator_，甚至 _non-const_ _container_ 和使用了 _iterator_ 去识别位置的 _STL_ 成员函数，比如：_insert_ 和 _erase_ 函数，都完全可以使用 _const_iterator_ 了。修改使用了 _iterator_ 的 _C++98_ 代码去使用 _C++11_ 的 _const_iterator_ 就非常简单了：  
```C++
  std::vector<int> values;                        // as before
  
  …
  
  auto it =                                       // use cbegin
  
  std::find(values.cbegin(),values.cend(), 1983); // and cend
  
  values.insert(it, 1998);
```

现在是使用了 _const_iterator_ 的代码了，它是适用的。

只有在一个情景中 _C++11_ 对 _const_iterator_ 的支持还不够充足，那就是当你想要写最通用的库代码时。这些代码会考虑到：一些 _container_ 和 _containers-like_ 的数据结构会提供 _begin_、_end_、_cbegin_、_cend_ 和 _rbegin_ 等来做为非成员函数，而不是成员函数。例如：内建数组就是这样，第三方库也是这样，它的接口只由自由函数所组成。因此，最通用代码会使用非成员函数，而不是假设有成员函数的版本存在。

例如：我们可以将我们正在工作的代码抽象为 _findAndInsert_，就像下面这样：  
```C++
  template<typename C, typename V>
  void findAndInsert(C& container,                // in container, find
                      const V& targetVal,         // first occurrence
                      const V& insertVal)         // of targetVal, then
  {                                               // insert insertVal
    using std::cbegin;                            // there
    using std::cend;

    auto it = std::find(cbegin(container),        // non-member cbegin
                          cend(container),        // non-member cend
                          targetVal);

    container.insert(it, insertVal);
  }
```  
这可以在 _C++14_ 中工作，但糟糕地是，不可以在 _C++11_ 中工作。由于标准化期间的疏忽，_C++11_ 只增加了非成员函数 _begin_ 和 _end_，但没有添加 _cbegin_、_cend_、_rbegin_ 和 _crend_。_C++14_ 纠正了这个疏忽。

如果你正在使用的是 _C++11_，并且你想要写最通用的代码，但正在使用的库没有提供所缺失的非成员函数 _cbegin_ 和友元函数的话，那么你可以很轻松地写出你自己的实现。例如：下面是非成员函数 _cbegin_ 的实现：  
  ```C++
    template <class C>
    auto cbegin(const C& container)->decltype(std::begin(container))
    {
      return std::begin(container);     // see explanation below
    }
  ```  

看到非成员函数 _cbegin_ 没有调用成员函数 _cbegin_，你会感到很惊讶，对吗？我也是的。但是这是符合逻辑的。这个 _cbegin_ 模板接受任意类型 _C_ 的实参，这个 _C_ 表示的是 _container-like_ 的数据结构类型，并通过 _reference-to-const_ 的形参 _container_ 来访问它的实参。如果 _C_ 是一个传统的 _container_ 类型的话，比如：_std::vector&lt;int&gt;_，那么这个 _container_ 是一个 _reference-to-const_ 的，比如：_const std::vector&lt;int&gt;&_。在  _const_ _container_ 上执行 _C++11_ 所提供的非成员函数 _begin_ 会产生一个 _const_iterator_，而模板会返回这个 _const_iterator_。这样做的优势是：对于那些提供有成员函数 _begin_ 但没有提供成员函数 _cbegin_ 的 _container_ 来说，这是可以工作的，其实 _C++11_ 的非成员函数 _begin_ 调用的就是成员函数 _begin_。因此你可以在只支持 _begin_ 的 _container_ 上使用非成员函数 _cbegin_。  

如果 _C_ 是内建数组类型的话，那么这个模板也是可以工作的。在这种情况下，_container_ 就是一个 _const_ 数组的引用了。_C++11_ 为内建的数组提供了一个非成员函数 _begin_ 的特化版本，这个函数会返回一个指向数组首元素的指针。因为 _const_ 数组的元素是 _const_ 的，所以，对于 _const_ 数组来说，非成员函数 _begin_ 所返回的指针也就是一个 _pointer-to-const_，事实上，_pointer-to-const_ 就是数组所对应的 _const_iterator_。为了进一步了解对于内建的数组模板应该如何进行特化，查阅 [_Item 1_](Chapter%201.md#item-1-理解模板的类型推导) 中关于持有数组引用形参的模板的类型推导的结论。

但是回到基本原则来。本 _Item_ 的观点是：鼓励你只要可以使用 _const_iterator_ 就去使用 _const_iterator_。基本的原则是只要有可能就应该去使用 _const_，这在 _C++11_ 之前就是一直有的。但是在 _C++98_ 中，当使用到 _iterator_ 时，这个原则就不适用了。而在 _C++11_ 中，这个原则就很适用了，并且在 _C++14_ 修复了 _C++11_ 留下的未完成工作。

### 需要记住的规则

* 首选 _const_iterator_ 而不是 _iterator_。
* 在最通用的代码中，首选非成员函数 _begin_、_end_ 和 _rbegin_ 等，而不是相应的成员函数版本。

## _Item 14_ 当函数不会抛出异常时声明函数为 _noexcept_

在 _C++98_ 中，_exception specification_ 确实是性情不定的野兽。你必须要总结出一个函数所有的可能会抛出的异常类型，所以如果这个函数的实现被更改了的话，那么它的 _exception specification_ 可能也需要更改。而改变一个函数的  _exception specification_  是可能会破坏客户的代码的，因为调用方可能是依赖于原来的 _exception specification_ 的。编译器一般不会帮助你去维护函数实现、_exception specification_ 和客户代码之间的一致性。所以大多数的开发者最终认为：_C++98_ 的 _exception specification_ 是不值得使用的。

在 _C++11_ 开发期间，达成了这样的一个共识：关于函数的异常抛出行为，真正有意义的信息是函数是否有异常。黑或白，函数要么可能会抛出异常，函数要么保证不会抛出异常。_maybe-or-never_ 二分法构成了 _C++11_ 的 _exception specification_ 的基础，本质上取代了 _C++98_ 的 _exception specification_。_C++98-style_ 的 _exception specification_ 仍然是有效的，但是是被废弃的。在 _C++11_ 中，无条件的 _noexcept_ 是对应于那些保证不会抛出异常的函数的。

函数是否应该被如此声明是接口设计的问题。函数的异常抛出行为对于客户是至关重要的。调用方可以查询函数的 _noexcept_ 状态，这个查询的结果可以影响调用代码的异常安全性和运行效率。本身来说，函数是否是 _noexcept_ 的和成员函数是否是 _const_ 的是同等重要的。当你知道函数不会抛出异常，却没有声明函数为 _noexcept_ 时，就是一个糟糕的接口规范了。

还有一个应用 _noexcept_ 到不会抛出异常的函数的额外激励：这会允许编译器去生成更好的对象代码。想要理解为什么会这样，只需要检查 _C++98_ 和 _C++11_ 在指明函数不会抛出异常的方式上的不同。考虑一个函数 _f_，它对调用方承诺永远不会抛出一个异常。有两种方法表示：  
```C++
  int f(int x) throw();       // no exceptions from f: C++98 style
  
  int f(int x) noexcept;      // no exceptions from f: C++11 style
```  

在运行期间，如果有一个异常离开了 _f_ 的话，那么这就是违反了 _f_ 的 _exception specification_ 了。又由于 _C++98_ 的 _exception specification_，调用堆栈会被展开至 _f_ 的调用方，然后在一些和本 _Item_ 不相关的操作之后，程序执行被终止。而由于 _C++11_ 的 _exception specification_，运行行为会稍微有不同：调用堆栈只是在程序执行被终止之前，可能会被展开。

会展开调用堆栈和可能会展开调用堆栈之间的不同对于代码的生成有着很大的影响。在 _noexcept_ 函数中，如果一个异常将会被传播到到函数之外的话，那么 _optimizer_ 是不需要将运行堆栈保持为展开状态的，_optimizer_ 也不必确保：当一个异常离开函数时，_noexcept_ 函数中的对象必须要按照构造的相反顺序来进行析构。而对于带有 _throw()_ _exception specification_ 的函数则缺乏这样的优化灵活性，就和完全没有异常明细的函数做的一样。这个情景可以这样总结：  
```C++
  RetType function(params) noexcept;    // most optimizable
  
  RetType function(params) throw();     // less optimizable
  
  RetType function(params);             // less optimizable
```

单是这个就有足够理由让你只要知道函数不会产生异常就声明函数为 _noexcept_ 了。

对于某些函数，情况更典型。_move operation_ 是极好的例子。假定你使用了 _std::vector&lt;Widget&gt;_ 的 _C++98_ 代码。偶尔会通过 _push_back_ 来将 _Widget_ 添加到 _std::vector_ 中：  
```C++
  std::vector<Widget> vw;
  
  …
  
  Widget w;
  
  …                           // work with w
  
  vw.push_back(w);            // add w to vw
  
  …
```  
假设这个代码是可以工作的，并且你也没兴趣为了 _C++11_ 去修改这个代码。然而，你想利用以下这个事实的优势：当涉及到 _move-enabled_ 类型时，_C++11_ 的移动语义可以提高 _legacy_ 代码的性能。因此你必须确保 _Widget_ 存在所对应的 _move operation_，要么是你要自己写它的 _move operatin_，要么是你要确保满足 _move operation_ 的自动生成条件，见 [_Item 17_](#item-17-理解特殊成员函数的生成)。

当一个新元素被添加到 _std::vector_ 时，_std::vector_ 可能已经没有这个元素的空间了，即为：_std::vector_ 的大小等于它的容量了。当发生这种情况时，_std::vector_ 会分配一个新的大的内存块去保存它的元素，并将它的元素从旧内存块转移到新内存块上。在 _C++98_ 中，是通过将每一个元素从旧内存拷贝到新内存来完成的，然后再去销毁旧内存上的对象。这种方法给 _push_back_ 提供了强异常安全保证：如果在拷贝期间抛出了一个异常的话，那么 _std::vector_ 的状态是保持不变的。因为直到全部元素都被成功拷贝到新内存后，旧内存上的全部元素才会被销毁。

在 _C++11_ 中，一个自然的优化是：对于 _std::vector_ 的元素，会使用移动来代替拷贝。不幸地是，这样做是冒着违反 _push_back_ 的异常安全保证风险的。如果 _n_ 个元素已经从旧内存移走了，然后在移动第 _n+1_ 
个元素期间抛出了一个异常，那么 _push_back_ 操作是不可以完成的。但是原来的 _std::vector_ 已经被更改了：因为 _n_ 个元素已经被移动走了。不可能恢复原来的状态，因为试图移动每一个元素回到原来的旧内存都可能会产生异常。

这是一个严重的问题，因为 _legacy_ 代码的行为可能是依赖于 _push_back_ 的强异常安全保证的。因此，_C++11_ 的实现不可以悄悄地使用所对应的 _move operation_ 来替换 _push_back_ 中的 _copy operation_，除非确认 _move operation_ 不会抛出异常。不会抛出异常时，_move operation_ 代替 _copy operation_ 是安全的，唯一的附加影响是将会提升性能。

_std::vector::push_back_ 利用了 **_move if you can, but copy if you must_** 的策略的优势，它不是唯一一个在标准库中这样做的函数。其他的利用了 _C++98_ 的强异常安全保证的函数也是使用了相同的方式，比如：_std::vector::reserve_ 和 _std::deque::insert_ 等。如果 _move operation_ 被认为是不会抛出异常异常的话，那么全部的这些函数会使用 _C++11_ 的 _move operation_ 来代替 _C++98_ 的 _copy operation_。但是这些函数如何可以知道 _move operation_ 是否不会产生异常呢？答案是明显的：去检查 _move operation_ 是否被声明为了 _noexcept_。

_swap_ 函数包括了另一种 _noexcept_ 尤其是值得拥有的场景。它是很多 _STL_ 算法实现的关键组成部分，它通常也会在 _copy assignment operator_ 中被使用。_swap_ 的广泛应用使得 _noexcept_ 所提供的优化特别地值得。有趣地是，标准库的 _swap_ 是否是 _noexcept_ 的有时是依赖于用户所定义的 _swap_ 是否是 _noexcept_ 的。例如：_array_ 和 _std::pair_ 所对应的标准库的 _swap_ 的声明是：  
```C++
  template <class T, size_t N>
  void swap(T (&a)[N],                                      // see
            T (&b)[N]) noexcept(noexcept(swap(*a, *b)));    // below
  
  template <class T1, class T2>
  struct pair {
    …
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                                noexcept(swap(second, p.second)));
    …
  };
```  

这些函数是 **_有条件地_** _noexcept_ 的：这些函数是否是 _noexcept_ 的是依赖于 _noexcept clause_ 中的表达式是否是 _noexcept_ 的。例如：给定两个 _Widget_ 类型的 _array_，只有当 _array_ 中的单独的元素的交换是 _noexcept_ 时，即为：_Widget_ 的 _swap_ 是 _noexcept_ 的时，这两个 _Widget_ 所对应的 _array_ 的交换才会是 _noexcept_ 的。因此，_Widget_ 的 _swap_ 的作者决定了 _Widget_ 的 _array_ 的交换是否是 _noexcept_ 的。这依次决定了其他的像 _Widget_ 的 _array_ 的 _array_ 这样的 _swap_ 是否是 _noexcept_ 的。类似地，还有两个包含着 _Widget_ 的 _std::pair_ 对象的交换是否是 _noexcept_ 的是依赖于 _Widget_ 的  _swap_ 是否是 _noexcept_ 的。只有当低级别成分的交换是 _noexcept_ 的时，高级别的数据结构才可以是 _noexcept_ 的，这个事实激励你只要可以就应该去提供 _noexcept_ 的 _swap_ 函数。

目前，我希望你对 _noexcept_ 所提供的优化机是感到兴奋的。但我必须缓和你的热情。因为虽然优化是重要的，但是正确性是更重要的。在本 _Item_ 的开头我就强调了 _noexcept_ 是函数接口的一部分，所以，只有当你愿意长期来维护一个 _noexcept_ 实现时，你才应该声明一个函数为 _noexcept_。如果你在开始时把一个函数声明为了 _noexcept_，但是后续却后悔了的话，那么你的选择是很有限的。你可以从函数的声明中删除 _noexcept_，即为：改变它的接口，但这是冒着破坏客户的代码的风险的。你还可以改变函数的实现，以让一个异常可以 _escape_，但还是保持原始的但现在是错误的 _exception specification_。如果你这样做了，当这个异常尝试离开函数时，你的程序将会被终止。又或者你可以接受现有的实现，而从一开始就抛弃任何激起你想要改变实现的想法。这些选择都没有吸引力。

事实上，大多数函数都是 _exception-neutral_ 的。这些函数本身是不会抛出异常的，但是它们所调用的函数是可能会抛出异常的。当这种情况发生时，_exception-neutral_ 的函数允许它们所调用的函数所抛出的异常通过它们的路到达更上层的调用链路的处理程序。_exception-neutral_ 的函数永远不是 _noexcept_ 的，因为它们还是可能会抛出 **_just passing through_** 的异常的。因此大多数函数很恰当地缺少了 _noexcept_ 名称。

然而，有些函数有着不会抛出任何异常的自然实现，对于一些函数来说，尤其是 _move operation_ 和 _swap_，声明为 _noexcept_ 可以有巨大的回报，如果可能的话，那么是值得按照 _noexcept_ 的方式去实现的。当你能确保一个函数是永远不会抛出异常时，你确实应该声明这个函数为 _noexcept_。

请注意，我说的是有些函数有着自然的 _noexcept_ 的实现。扭曲函数的实现以去让 _noexcept_ 成为可能是不合理的。**_Is putting the cart before the horse. Is not seeing the forest for the trees_.** 选择你最爱的比喻。如果一个简单的函数实现可能会产生异常的话，比如：调用可能会抛出异常的函数，那么你可以付出努力来将这些抛出的异常对调用方进行隐藏，比如：捕获所有的异常并使用状态码或特殊返回值来代替它，这不仅会使得你的函数的实现变得非常复杂，还会通常使得调用点的代码变得复杂。例如：调用方必须要检查状态码或者特殊返回值。这些复杂度的运行时间成本，比如：额外的分支和更大的函数会给指令缓存施加更大的压力，可能会超出你希望通过 _noexcept_ 所实现的加速，而且代码更难以理解和维护了。这将是糟糕的软件工程。

对于一些函数来说，成为 _noexcept_ 的是非常重要的，它们默认就应该是那样。在 _C++98_ 中，允许内存释放函数，即为：_operator delete_ 和 _operator delete[]_，和析构函数去抛出异常被认为是一个糟糕的设计，而在 _C++11_ 中，这种风格规则已经被升级为是一个语言规则了。所有的内存释放函数和析构函数，包括用户定义的和编译器生成的，默认都是隐式 _noexcept_ 的。因此不需要声明它们为 _noexcept_。进行声明也不会伤害任何东西，只是不常见而已。只有当类的数据成员，包括所继承的成员和被包含在其他数据成员中的成员，明确地声明了它们的析构函数可能会抛出异常时，比如：声明为 _noexcept(false)_，才是仅有的析构函数不是隐式 _noexcept_ 的情况。这样的析构函数是不常见的。在标准库中是没有这样的析构函数的，如果标准库所使用的对象的析构函数抛出了一个异常的话，比如：这个对象是在 _container_ 中的，是被传递到算法中的，那么程序的行为是未定义的。

值得注意的是：一些库接口设计者会区分 _wide contract_ 函数和 _narrow contract_ 函数。_wide contract_ 函数没有前置条件。这些函数不管程序状态如何都可以会被调用，没有在调用方所传递的实参上强加限制。_wide contract_ 函数永远不会出现 _undefined behavior_。

没有 _wide contract_ 的函数就是有 _narrow contract_ 的了。对于这样的函数，如果违反了前置条件的话，那么结果将是未定义的。

如果你正在写一个 _wide contract_ 函数，并且你知道它是不会抛出异常的话，那么遵循本 _Item_ 的建议，将它声明为 _noexcept_ 是简单的。对于 _narrow contract_ 函数来说，是狡猾的。比如，假定你正在写一个持有 _std::string_ 形参的函数 _f_，而且假设这个 _f_ 的自然的实现永远不会产生异常。建议应该将 _f_ 声明为 _noexcept_。 

现在假定 _f_ 有一个前置条件：它的 _std::string_ 形参的长度不能超过 _32_ 个字符。如果使用了超过了 _32_ 个字符长度的 _std::string_ 来调用了 _f_ 的话，那么行为将是 _undefined behavior_，因为违反了前置条件会导致 _undefined behavior_。_f_ 没有责任去检查这个前置条件，因为函数可以假设它的前置条件都是被满足的。调用方有责任确保这个假设是有效的。所以，即使有了前置条件，声明 _f_ 为 _noexcept_ 似乎也是合理的：
```C++
  void f(const std::string& s) noexcept;          // precondition:
                                                  // s.length() <= 32
```  

但是，现在假设 _f_ 的实现者选择检查是否违反前置条件了。检查不是必须的，但是也不是被禁止的，而且检查前置条件也可能是有用的。比如：在系统测试期间。调试已经被抛出的异常通常比追踪 _undefined behavior_ 的原因要容易。但是违反前置条件应该如何报告才能使测试工具或客户的错误程序能够探测它呢？一个简单的方法是去抛出一个“违反了前置条件”的异常，但是如果 _f_ 被声明为了 _noexcept_ 的话，那么就不可以这样了，抛出异常将会导致程序被终止。所以，区分 _wide contract_ 和 _narrow contract_ 的库设计者通常只会为 _wide contract_ 函数保留 _noexcept_。

最后，让我详细说明我之前的观察：编译器一般不会提供帮助去识别函数实现和它们的 _exception specification_ 之间的不一致性。考虑这个代码，是完美合法的：  
```C++
  void setup();               // functions defined elsewhere
  void cleanup();

  void doWork() noexcept
  {
    setup();                  // set up work to be done
    
    …                         // do the actual work
    
    cleanup();                // perform cleanup actions
  }
```  
此处，_doWork_ 是被声明为 _noexcept_ 的，尽管 _doWork_ 调用了 _non-noexcept_ 函数 _setup_ 和 _cleanup_ 函数。这似乎是矛盾的，但却是是可以的，那就是当 _setup_ 和 _cleanup_ 函数在文档中说明了它们是永远不会抛出异常的，尽管它们没有被声明为 _noexcept_。有好的理由将这样不会抛出异常的函数不声明为 _noexcept_，例如：它们可能是 _C_ 库的一部分，甚至一些已经从 _C_ 标准库移动到 _std namespace_ 的函数仍然缺少着 _exception specification_，比如：_std::strlen_ 没有被声明为 _noexcept_。又或者它们是决定不使用 _C++98_ 的 _exception specification_，但还没有被修改为使用 _C++11_ 的 _exception specification_ 的 _C++98_ 的库。

因为有着确实的理由使得 _noexcept_ 函数依赖着缺少有 _noexcept_ 保证的代码，所以 _C++_ 允许这样的代码可以通过编译，而且编译器通常不会对这样的代码产生警告。

### 需要记住的规则

* _noexcept_ 是函数接口的一部分，这意味着调用方可能会依赖它。
* _noexcept_ 函数比 _non-noexcept_ 函数是更可优化的。
* _noexcept_ 函数尤其对 _move operation_、_swap_、内存释放函数以及析构函数有价值。
* 大多数函数是 _exception-neutral_ 的，而不是 _noexcept_ 的。

## _Item 15_ 只要有可能就使用 _constexpr_

如果存在有一个 _C++11_ 中的最令人困惑新名词的奖项时，_constexpr_ 大概率会赢得这个奖项。当 _constexpr_ 被应用到对象上时，它就是一个 _const_ 的强化形式，但是当 _constexpr_ 被应用到函数上时，它就有非常不同的含义了。搞清楚这个是值得的。因为当 _constexpr_ 与你想要去表达的内容相符合时，你绝对会想要使用它。

概念上来说，_constexpr_ 不仅表明一个值是常量的，而且还表明这个值在编译期间就是已知的。但是，这个概念只是故事的一部分，因为当 _constexpr_ 被应用到函数上时，事情比这个说明要更加微妙。我先不剧透那个令人惊喜的
结论，现在我只说：你不能假设 _constexpr_ 函数的结果是 _const_ 的，也不能认为 _constexpr_ 函数的结果的值在编译期间就是已知的。大概最有趣地是，这些都是是特性。_constexpr_ 函数不需要产生是 _const_ 的或者在编译期间就是已知的结果。

让我们从 _constexpr_ 对象开始。实际上，它们是 _const_ 的，它们的值在编译期间就是已知的。技术上来说，它们的值是在 _translation_ 期间被确定的，_translation_ 不仅包括编译，还包括链接。然而，除非你写的是 _C++_ 的编译器或链接器，否则不会影响到你，所以你可以无忧无虑地编程，就像是 _constexpr_ 对象的值是在编译期间确定的。

在编译期间就是已知的值是被特殊对待的。例如：它们会被放到只读内存上，特别对于嵌入式系统的开发者来说，这是一个相当重要的特性。更广泛的应用是：那些是常量的和在编译器期间就是已知的 _integral_ 值是可以被用在 _C++_ 是需要 _integral_ 常量表达式的上下文中的。包括指定：_array_ 的大小、_integral_ 模板的实参、_std::array_ 对象的长度、_enumerator_ 值、_alignment specifier_ 等等。如果你想要使用变量来完成这些的话，那么你肯定会想要声明这个变量为 _constexpr_ ，因为编译器会确保这个变量有一个 _compile-time_ 的值：  
```C++
  int sz;                               // non-constexpr variable
  
  …
  
  constexpr auto arraySize1 = sz;       // error! sz's value not
                                        // known at compilation
  
  std::array<int, sz> data1;            // error! same problem
  
  constexpr auto arraySize2 = 10;       // fine, 10 is a
                                        // compile-time constant
  
  std::array<int, arraySize2> data2;    // fine, arraySize2
                                        // is constexpr
```

注意：_const_ 不会提供和 _constexpr_ 相同的保证，因为 _const_ 对象不需要使用在编译期间就是已知的值来进行初始化:  
```C++
  int sz;                               // as before
  
  …
  
  const auto arraySize = sz;            // fine, arraySize is
                                        // const copy of sz

  std::array<int, arraySize> data;      // error! arraySize's value
                                        // not known at compilation
```  

简单来说：所有 _constexpr_ 对象都是 _const_ 的，但所有 _const_ 对象不都是 _constexpr_ 的。如果你想要编译器去保证一个变量的值是可以被用在需要 _compile-time_ 的常量的上下文中的话，那么可以完成这个工作的是 _constexpr_，而不是 _const_。

当涉及到 _constexpr_ 函数时，_constexpr_ 对象的使用情况是会变得更有意思。当使用 _compile-time_ 的常量来调用 _constexpr_ 函数时，函数会产生 _compile-time_ 的常量。当使用 _run-time_ 的值来调用  _constexpr_ 函数的话，函数会产生 _run-time_ 的值。听起来好像不知道它们会做什么，但不是的。应该这样去看待：  
* _constexpr_ 函数可以被用在需要 _compile-time_ 的常量上下文中。在这个上下文中，如果你传递给 _constexpr_ 函数的实参的值是在编译期间就是已知的话，那么这个 _constexpr_ 函数的结果将会在编译期间被计算。如果任意一个实参的值不是在编译期间就是已知的，那么你的代码将会被拒绝。
* 当使用一个或者多个不是在编译期间就是已知的值来调用 _constexpr_ 函数时，这个 _constexpr_ 函数就会变现地像一个普通函数了，这个 _constexpr_ 函数是在运行期间被计算的。这意味着你不需要两个函数来执行相同的操作了，一个对应于 _compile-time_ 的常量，另一个对应于其他的值。_constexpr_ 函数会一起完成。

假定我们需要一个数据结构去保存一个可以以多种方式运行的实验的结果。例如：光照水平在实验过程中可以是高的、低的或者关闭的，同理还有风扇速度和温度等。如果有 _n_ 个与实验相关的环境条件，它们的每一个都有三个可能状态的话，那么就共有 _3^n_ 个组合了。因此，想要存储所有条件的组合的实验结果，就得需要一个有着足够的空间的可以存下 _3^n_ 个值的数据结构。如果每一个结果都是一个 _int_，并且 _n_ 是在编译期间就是已知的或是在编译器期间就是可以被计算出的话，那么 _std::array_ 是一个合理的数据结构的选择。但我们需要一个方法在编译期间就计算出 _3^n_ 的值。_C++_ 的标准库提供了 _std::pow_，它是一个我们需要的数学功能，但是要想完成我们的目标的话，还存在两个问题。首先，_std::pow_ 使用的是 _floating-point_ 类型，但是我们需要的是 _integral_ 结果。其次，它不是一个 _constexpr_ 函数，即为：当使用 _compile-time_ 的值来调试时，不能保证返回 _compile-time_ 的结果，所以我们不能使用 _std::pow_ 函数来指明 _std::array_ 的大小。


幸运地是，我们可以写一个我们所需要的 _pow_。我会立刻来展示如何去做，先让我们看下是如何声明和使用的：  
```C++
  constexpr                                       // pow's a constexpr func
  int pow(int base, int exp) noexcept             // that never throws
  {
    …                                             // impl is below
  }
  
  constexpr auto numConds = 5;                    // # of conditions
  std::array<int, pow(3, numConds)> results;      // results has
                                                  // 3^numConds
                                                  // elements
```

回忆一下：_pow_ 前面的 _constexpr_ 不是说 _pow_ 返回了一个 _const_ 值，而是说如果 _base_ 和 _exp_ 是 _compile-time_ 的常量的话，那么 _pow_ 的结果可以被用来做为一个 _compile-time_ 的常量。如果 _base_ 和或 _exp_ 不是 _compile-time_ 的常量的话，那么 _pow_ 的结果是在运行期间被计算的。这意味着：_pow_ 不仅仅被调用来去做像  _compile-time-compute_ _std::array_ 的大小这样的事，而且可以在运行期间像下面的这样被调用：  
```C++
  auto base = readFromDB("base");       // get these values
  auto exp = readFromDB("exponent");    // at runtime
  
  auto baseToExp = pow(base, exp);      // call pow function
// at runtime
```

因为当使用 _compile-time_ 的值调用 _constexpr_ 函数时，_constexpr_ 函数必须能够返回 _compile-time_ 的结果，所以一 些限制被施加到了这个函数实现中。而且这些限制在 _C++11_ 和 _C++14_ 是不相同的。

在 _C++11_ 中，_constexpr_ 函数不能包含多个 _return_ 语句。听起来限制很多，实际上却不是，因为有两个技巧可以被用来扩展 _constexpr_ 函数的表达性，这是超出你可能想到的。首先，条件操作符 _?:_ 可以被用来代替 _if-else_ 语句，其次，递归可以被用来代替循环。因此 _pow_ 可以被像下面这样实现：  
```C++
  constexpr int pow(int base, int exp) noexcept
  {
    return (exp == 0 ? 1 : base * pow(base, exp - 1));
  }
```  

这是可以工作的，但是很难想象除了硬核函数式程序员以外，还会有其他人会认为它是好的。而在 _C++14_ 中，对 _constexpr_ 函数的限制减少了，所以可以下面这样实现了：
```C++
  constexpr int pow(int base, int exp) noexcept // C++14
  {
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    
    return result;
  }
```

_constexpr_ 函数被限于仅能持有和返回 _literal_ 类型，这样的类型的值是在编译期间就能被确定的。在 _C++11_ 中除了 _void_ 以外的其他全部内建类型都是 _literal_ 类型，用户定义的类型也是 _literal_ 类型，因为构造函数和其他成员函数也可以是 _constexpr_ 的：  
```C++
  class Point {
  public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept
      : x(xVal), y(yVal)
      {}

      constexpr double xValue() const noexcept { return x; }
      constexpr double yValue() const noexcept { return y; }
      
      void setX(double newX) noexcept { x = newX; }
      void setY(double newY) noexcept { y = newY; }
    
    private:
      double x, y;
  };
```  
此处，_Point_ 的构造函数可以被声明为 _constexpr_，因为如果所传入的实参是在编译期间就是已知的话，那么所构造的 _Point_ 的数据成员的值也是在编译期间就是已知的。所以可以是 _constexpr_ 的：  
```C++
  constexpr Point p1(9.4, 27.7);        // fine, "runs" constexpr
                                        // ctor during compilation
  
  constexpr Point p2(28.8, 5.3);        // also fine
```  

类似地，_getter_ 函数 _xValue_ 和 _yValue_ 也可以是 _constexpr_ 的，因为如果这些函数使用的是在编译期间就是已知的值的话，比如：_constexpr_ 的 _Point_ 对象，那么数据成员 _x_ 和 _y_ 的值也就是在编译期间就是已知的了。这样就可以编写有调用了 _Point_ 的 _getter_ 函数的 _constexpr_ 函数了，并使用这些 _constexpr_ 函数的结果来初始化 _constexpr_ 对象：
```C++
  constexpr
  Point midpoint(const Point& p1, const Point& p2) noexcept
  {
    return { (p1.xValue() + p2.xValue()) / 2,               // call constexpr
    (p1.yValue() + p2.yValue()) / 2 };                      // member funcs
  }
  
  constexpr auto mid = midpoint(p1, p2);                    // init constexpr
                                                            // object w/result of
                                                            // constexpr function
```  

这是非常令人兴奋的。这意味着：尽管 _mid_ 的初始化涉及到了调用构造函数、_getter_ 函数和非成员函数，但是，它仍然可以在只读内存中被创建。这意味着：你可以在模板的实参中或者在要指定 _enumerator_ 值的表达式中使用像 _mid.xValue() * 10_ 这样的的表达式了。这意味着：在编译期间所完成的工作和运行期间所完成的工作之间传统上是相当严格的界限开始变得模糊了，一些传统上是在运行期间所完成的计算现在可以迁移到编译期间来进行了。迁移的代码越多，你的程序将会运行的更快，当然编译也会更耗时。

在 _C++11_ 中，有两个限制会阻止 _Point_ 的成员函数 _setX_ 和 _setY_ 被声明为 _constexpr_。首先，这两个函数会更改它们所要操作的对象，而在 _C++11_ 中 _constexpr_ 成员函数是隐式 _const_ 的。其次，这两个函数的返回类型是 _void_ 类型，而在 _C++11_ 中 _void_ 不是 _literal_ 类型。全部的这些限制在 _C++14_ 中都被解除了，在 _C++14_ 中，甚至 _Point_ 的 _setter_ 也可以是 _constexpr_ 的：  
```C++
  class Point {
  public:
    …
    
    constexpr void setX(double newX) noexcept // C++14
    { x = newX; }
    
    constexpr void setY(double newY) noexcept // C++14
    { y = newY; }

    …

  };
```

也可以写下面这样的函数了：  
```C++
  // return reflection of p with respect to the origin (C++14)
  constexpr Point reflection(const Point& p) noexcept
  {
    Point result;                       // create non-const Point

    result.setX(-p.xValue());           // set its x and y values
    result.setY(-p.yValue());
    
    return result;                      // return copy of it
  }
```

客户代码可能看起来像下面这样：  
```C++
  constexpr Point p1(9.4, 27.7);                  // as above
  constexpr Point p2(28.8, 5.3);
  constexpr auto mid = midpoint(p1, p2);

  constexpr auto reflectedMid =                   // reflectedMid's value is
  reflection(mid);                                // (-19.1 -16.5) and known
                                                  // during compilation
```

本 _Item_ 的建议是只要有可能就使用 _constexpr_，现在我希望你已经明白了这是什么：_constexpr_ 对象和函数都可以被使用在比 _non-constexpr_ 对象和函数要更广泛的上下文中。通过只要有可能就使用 _constexpr_，你最大化了你的对象和函数可以被使用的情景的范围。  

_constexpr_ 是对象或者函数接口的一部分。这很重要，需要特别注意。_constexpr_ 强调了“我可以被用在 _C++_ 是需要一个常量表达式的上下文中”。如果你声明了一个对象或函数为 _constexpr_ 的话，那么客户是可以在这样的上下文中使用它的。如果你后续觉得使用 _constexpr_ 是错误的，并且删除了它的话，那么这可能会导致大量的客户代码不能通过编译。增加 _I/O_ 到一个函数中来进行调式或者性能调整，就可能会导致这样的错误发生，因为在 _constexpr_ 函数中一般是不被允许存在 _I/O_ 语句的。在“只要有可能就使用 _constexpr_”中的“只要有可能”表示你愿意长期对某个函数或函数施加这种限制。

### 需要记住的规则

* _constexpr_ 对象是 _const_ 的，需要使用是在编译期间就是已知的值来进行初始化。
* 当调用 _constexpr_ 函数的实参的值是在编译期间就是已知的时，_constexpr_ 函数可以产生 _compile-time_ 的结果。
* _constexpr_ 对象和函数可以比 _non-constexpr_ 对象和函数用在更广泛的上下文中。
* _constexpr_ 是对象或者函数接口的一部分。

## _Item 16_ 使 _const_ 成员函数成为线程安全的

如果你是在数学领域工作的话，那么可以发现拥有一个可以表示多项式的类是方便的。如果在这个类中，有一个可以计算多项式的根的函数的话，那么会是很有用的，即为：多项式等于 _0_ 的值。这样的函数不会更改多项式，所以声明这个函数为 _const_ 是自然的：
```C++
  class Polynomial {
  public:
    using RootsType =         // data structure holding values
      std::vector<double>;    // where polynomial evals to zero
    …                         // (see Item 9 for info on "using")
    
    RootsType roots() const;
    
    …

  };
```

计算多项式的根可以是成本大的，所以如果不是必须去的话，我们是不想去做的。如果我们必须去做的的话，我们肯定不想多做一次。因此，如果必须做的话，我们将会缓存多项式的根，并实现 _roots_ 来返回所缓存的值，下面是基本方法：
```C++
  class Polynomial {
  public:
    using RootsType = std::vector<double>;
  
    RootsType roots() const
    {
      if (!rootsAreValid) {                       // if cache not valid
        
        …                                         // compute roots,
                                                  // store them in rootVals
      rootsAreValid = true;
    }

    return rootVals;
  }
  
  private:
    mutable bool rootsAreValid{ false };          // see Item 7 for info
    mutable RootsType rootVals{};                 // on initializers
  };
```

概念上来说，_roots_ 不会改变它所操作的 _Polynomial_ 对象，但是做为它缓存活动的一部分，它可能会需要去更改 _rootVals_ 和 _rootsAreValid_。这是 _mutable_ 的经典用法，也是为什么 _mutable_ 是这些数据成员的声明的一部分。

想象现在有两个线程同时调用一个 _Polynomial_ 对象的 _roots_：
```C++
  Polynomial p;
  
  …
  /*----- Thread 1 ----- */   /*------- Thread 2 ------- */
  auto rootsOfP = p.roots();  auto valsGivingZero = p.roots();
```  

这个客户代码完全是合理的。因为 _roots_ 是 _const_ 成员函数，所以这意味着它代表的是一个读操作。在没有同步的情况下，多线程执行一个读操作是安全的。至少应该是这样。在这个场景中，却不是的。因为在 _roots_ 的内部，这些线程中的一个或者全部都可能会尝试去更改数据成员 _rootsAreValid_ 和 _rootVals_。这意味着：这个代码中的不同线程可能会在没有同步的情况下读写相同的内存，这就是数据竞争的定义。这个代码有着 _undefined behavior_。

这个问题是 _roots_ 被声明为了 _const_，但却不是线程安全的。_const_ 声明在 _C++11_ 和 _C++98_ 中都是正确的，因为获取多项式的根不会改变多项式的值，所以需要修正的是线程安全的缺失。

解决这个问题最简单的方法是利用 _mutex_，这是常见的：
```C++
  class Polynomial {
  public:
    using RootsType = std::vector<double>;
    
    RootsType roots() const
    {
      std::lock_guard<std::mutex> g(m);           // lock mutex

      if (!rootsAreValid) {                       // if cache not valid
      
        …                                         // compute/store roots
      
        rootsAreValid = true;
      }

    return rootVals;
  }                                               // unlock mutex
  
  private:
    mutable std::mutex m;
    mutable bool rootsAreValid{ false };
    mutable RootsType rootVals{};
  };
```  
_std::mutex m_ 被声明为了 _mutable_，因为 _m_ 的加锁和解锁是 _non-const_ 成员函数，但却是在 _const_ 成员函数 _roots_ 中，如果 _m_ 没有被声明为 _mutable_ 的话，那么 _m_ 将会被认为 _const_ 对象。

值得注意的是：因为 _std::mutex_ 是一个 _move-only_ 类型，即为:只可以被移动，不可以被拷贝的类型。增加了 _m_ 到 _Polynomial_ 的附加影响是 _Polynomial_ 丢失了可以被拷贝的能力。但是仍然是被移动的。

在一些情景下，_mutex_ 太重了。例如：如果你正在做的是计算某个成员函数被调用了多少次的话，那么 _std::atomic_ _counter_，即为：其他的线程保证看到的是这个 _counter_ 的操作是不可分割地发生的，见 [_Item 40_](Chapter%207.md#item-40-并发使用-stdatomic-特殊内存使用-volatile)，通常来说是成本小的，是否确实是成本小的，依赖于你正在运行的硬件和你的标准库中的 _mutex_ 的实现的。此处展示如何可以利  
用 _std::atomic_ 来计数调用次数：  
```C++
  class Point { // 2D point
  public:
    …
    
    double distanceFromOrigin() const noexcept              // see Item 14
    {                                                       // for noexcept
      ++callCount;                                          // atomic increment
      
      return std::sqrt((x * x) + (y * y));
    }
  
  private:
    mutable std::atomic<unsigned> callCount{ 0 };
    double x, y;
  };
```  
因为像 _std::mutex_ 一样，_std::atomic_ 也是 _move-only_ 类型，所以 _Point_ 中的 _callCount_ 的存在是意味着 _Point_ 也是 _move-only_ 的。

因为 _std::atomic_ 变量的操作通常比 _mutex_ 的获取和释放是要成本小的，所以你可能会过度倾向于去使用 _std::atomic_。比如，在一个缓存了 _expensive-to-compute int_ 的类中，你可能会尝试使用 _std::atomic_ 变量来代替 _mutex_：  
```C++
  class Widget {
  public:
    …
    int magicValue() const
    {
      if (cacheValid) return cachedValue;
      else {
        auto val1 = expensiveComputation1();
        auto val2 = expensiveComputation2();
        cachedValue = val1 + val2;                // uh oh, part 1
        cacheValid = true;                        // uh oh, part 2
        return cachedValue;
      }
    }

  private:
    mutable std::atomic<bool> cacheValid{ false };
    mutable std::atomic<int> cachedValue;
  };
```

这是可以工作的，但是有时候工作要困难些。考虑：
* 第一个线程调用了 _Widget::magicValue_，看到了 _cacheValid_ 为 _false_，执行两个成本大的计算后将它们的和分配给了 _cachedValue_。
* 在那时，第二个线程调用了 _Widget::magicValue_，也看到了 _cacheValid_ 为 _false_，因此也执行了和第一个线程已经完成的是相同的成本大的计算。实际上，“第二个线程”可能是其他多个线程。

这样的行为和缓存的目标是矛盾的。颠倒 _cachedValue_ 和 _cacheValid_ 的赋值顺序可以消除这个问题，但是结果甚至更糟：  
```C++
  class Widget {
  public:
    …
    
    int magicValue() const
    {
      if (cacheValid) return cachedValue;
      else {
        auto val1 = expensiveComputation1();
        auto val2 = expensiveComputation2();
        cacheValid = true; // uh oh, part 1
        return cachedValue = val1 + val2; // uh oh, part 2
      }
    }

    …

  };
```  
想象 _cacheValid_ 是 _false_，那么：
* 一个线程调用了 _Widget::magicValue_，并执行到了 _cacheValid_ 被设置为 _true_ 的那一点。
* 此时，第二个线程调用了 _Widget::magicValue_，然后会检查 _cacheValid_。看到的它为 _true_ 后，这个线程会返回 _cachedValue_，尽管第一个线程还没有对它的赋值。因此，所返回的值是错误的。

这里有一个注意事项。对于单个要求同步的变量或内存区域来说，使用 _std::atomic_ 是适当的，但是一旦你有两个或多个要求做为整体来进行操作的变量或内存区域的话，那么你应该去使用 _mutex_。对于 _Widget::magicValue_，看起来像是这样：  
```C++
  class Widget {
  public:
    …

    int magicValue() const
    {
      std::lock_guard<std::mutex> guard(m);       // lock m
      
      if (cacheValid) return cachedValue;
      else {
        auto val1 = expensiveComputation1();
        auto val2 = expensiveComputation2();
        cachedValue = val1 + val2;
        cacheValid = true;
        return cachedValue;
      }
    }                                             // unlock m
    …

  private:
    mutable std::mutex m;
    mutable int cachedValue;                      // no longer atomic
    mutable bool cacheValid{ false };             // no longer atomic
  };
```

目前，本 _Item_ 是以这个假设为依据的：多个线程可能会同步地执行同一个对象的同一个 _const_ 成员函数。如果你正在写的 _const_ 成员函数不是这种场景，可以保证永远不会有多个的线程执行同一个对象的同一个 _const_ 成员函数的话，那么函数的线程安全性是不重要的。例如：专门被设计成用于单线程的类的成员函数是线程安全的。在那个场景中，你可以避免 _mutex_ 和 _std::atomic_ 所代码相关的成本，也可以避免将它们的类呈现为 _move-only_。然而这些 _threading-free_ 的情景越来越不常见了，而且会变得越来越少。可以肯定的是 _const_ 成员函数都会运行在并发执行的条件下，这也是为什么你应该确保 _const_ 成员函数是线程安全的。

### 需要记住的规则

* 使 _const_ 成员函数成为线程安全的，除非你确定这些函数永远不会被用在并发的上下文中。
* 可能使用 _std::atomic_ 变量比使用 _mutex_ 要有更好的性能，但是只适用于操作单一变量和内存区域的情况。

## _Item 17_ 理解特殊成员函数的生成

按照 _C++_ 官方的说法，特殊成员函数是 _C++_ 主动生成的。_C++98_ 有四个这样的函数：_default constructor_、析构函数、_copy constructor_ 和 _copy assignment operator_。当然，这里是有注解的。只有当这些函数在被需要的时候，它们才会被生成，即为：一些代码使用了这些函数，但是在类中没有显式地声明这些函数。只有当在类中没有声明任何的构造函数时，_default constructor_ 才会被生成。在类中指定构造函数所需要的实参就可以阻止编译器去创建 _default constructor_。所生成的特殊成员函数是隐式 _public_ 和 _inline_ 的，而且这些函数都是 _nonvirtual_ 的，除了继承自有着 _virtual_ 析构函数的 _base class_ 的 _derived class_ 中的析构函数以外。在这种情况下，编译器生成 _derived class_ 的析构函数也是 _virtual_ 的。

不过这些都是你已经知道的了。是的，是的，老掉牙的故事：_Mesopotamia_、_the Shang dynasty_、_FORTRAN_ 以及 _C++98_。但是时代变了，_C++_ 的特殊成员函数的生成规则也随着一起变了。知道新的规则是非常重要的，对于高效的 _C++_ 编程来说，很少有事情能像知道编译器何时会悄悄地向类中添加成员函数一样重要。

从 _C++11_ 开始，特殊成员函数俱乐部新增了两个成员：_move constructor_ 和 _move assignment operator_。它们的 _signature_ 是：  
```C++
  class Widget {
  public:
    …
    Widget(Widget&& rhs);               // move constructor
    
    Widget& operator=(Widget&& rhs);    // move assignment operator
    …
  };
```  

这两个函数的生成规则和行为规则类似于它们的拷贝部分。只有当 _move operation_ 被需要时，_move operation_ 才会被生成，如果是被生成的话，那么 _move operation_ 会对类的非静态数据成员执行 **_memberwise move_**。这意味着 _move constructor_ 会根据它的形参 _rhs_ 的相应的成员来 _move-construct_ 所对应类的每一个非静态数据成员，同样 _move assignment operator_ 会根据它的形参 _rhs_ 的相应的成员来 _move-assign_ 所对应类的每一个非静态数据成员。如果存在有 _base class_ 的话，那么 _move constructor_ 也会 _move-construct_ 它所对应的 _base class_ 部分，同样  _move assignment operator_ 也会 _move-assign_ 它的 _base class_。

现在，当我提及 _move construct_ 或 _move-assign_ 数据成员和 _base class_ 的 _move operation_ 时，这并不能保证移动是一定会发生的。实际上，**_memberwise move_** 更像是 _memberwise move_ 请求，因为那些不支持移动的类型，即为：没有提供 _move operation_ 的特殊支持的类型，比如：大多数 _C++98_ 的 _legacy_ 类，都是通过 _copy operation_ 来进行 **_移动_** 的。_memberwise move_ 的关键是：首先将 _std::move_ 应用到所要移动的对象上，然后所产生的结果会在函数重载决议期间被使用，最后会决定执行的是移动还是拷贝。[_Item 23_](Chapter%205.md#item-23-理解-stdmove-和-stdforward) 会详细介绍这个过程。对于本 _Item_，只需要记住：_memberwise move_ 包含了那些支持了 _move operation_ 的数据成员和 _base class_ 所对应的 _move operation_ 和那些不支持 _move operation_ 的数据成员和 _base class_ 所对应的 _copy operation_。

就像 _move operation_ 场景一样，如果你自己声明了 _move operation_，那么编译器就不会生成它。但是，可以生成 _move operation_ 的准确条件和 _copy operation_ 的是稍有不同的。

两个 _copy operation_ 是独立的：声明其中一个并不会阻止编译器去生成另一个。所以，如果你声明了 _copy constructor_，但是没有声明另一个 _copy assignment operator_，而且又写了需要 _copy assignment operator_ 的代码的话，那么编译器是会为你生成 _copy assignment operator_ 的。类似地也是这样，如果你声明了 _copy assignment operator_，但没有声明 _copy constructor_，而且你的代码需要 _copy constructor_ 的话，编译器是会为你生成 _copy constructor_ 的。在 _C++98_ 是这样的，在 _C++11_ 中仍然是这样的。

两个 _move operation_ 不是独立的。如果你声明了其中一个的话，那么这会阻止编译器去生成另一个。根本原因是：如果你为你的类声明了 _move constructor_ 的话，那么你就是在表明你所声明的 _move constructor_ 的实现和编译器所生成的默认的 _memberwise move_ 是不相同的。如果 _memberwise move construction_ 是有问题的话，那么大概率 _memberwise move assignment_ 也是有问题的。所以声明 _move constructor_ 会阻止 _move assignment operator_ 被生成，相应地声明 _move assignment operator_ 也会阻止 _move constructor_ 被生成。

此外，编译器不会为任何显式声明了 _copy operation_ 的类来生成 _move operation_。合理解释是：如果一个类声明了 _copy operation_ 的话，包括 _copy constructor_ 和 _copy assignment operator_，那么就表明了拷贝一个对象的普通方法 _memberwise copy_ 对于这个类来说是不合适的，编译器也会认为：如果 _memberwise copy_ 对于 _copy operation_ 是不合适的话，那么 _memberwise move_ 对于 _move operation_ 也是不合适的。

反之亦然。在一个类中声明了 _move operation_，包括 _move constructor_ 和 _move assignment operator_，这将会导致编译器删除这个类所对应的 _copy operation_，见 [_Item 11_](#item-11-首选-deleted-function-而不是-private-undefined-function)。毕竟，对于移动一个对象来说，如果 _memberwise move_ 是不合适的方法的话，那么对于拷贝一个对象来说，也没有理由去期待 _memberwise copy_ 是合适的方法。听起来这是可能会破坏 _C++98_ 的代码的，因为 _copy operation_ 起作用的条件，在 _C++11_ 中比在 _C++98_ 中更严格了，但是不会的。_C++98_ 的代码不会有 _move operation_，因为在 _C++98_ 中没有像 **_移动_** 对象这样的事情。_legacy_ 的类可以有用户声明的 _move operation_ 的唯一的方法是：如果要为 _C++11_ 添加 _move operation_ 的话，那么被更改了的要去利用移动语义优势的这些类必须要遵守 _C++11_ 的特殊成员函数生成的规则。

大概你已经听过了被称为是 _Rule of Three_ 的准则了。这个准则规定了：如果你声明了任意一个 _copy constructor_、_copy assignment operator_ 和析构函数的话，那么你应该把这三个都声明出来。这个准则的形成源自于这样一个观察：几乎总是因为需要执行某种类型的资源管理操作，我们才需要接管 _copy operation_ 的含义，这也几乎总是暗示了：（1）在一个 _copy operation_ 中所完成的资源管理，大概率也需要在另一个 _copy operation_ 中完成。（2）类的析构函数也需要来参与资源管理，通常是释放资源。典型的需要被管理的资源是内存，这也是为什么所有那些管理 资源的标准库的类，比如：执行动态内存管理的 _STL_ 容器，都声明了这三个函数：_copy operation_ 和析构函数。

_Rule of Three_ 的一个结果是：用户所声明的析构函数表明了简单的  _memberwise copy_ 对于类的 _copy operation_ 来说可能是不合适的。相应地，_Rule of Three_ 也建议：如果一个类声明了析构函数的话，那么 _copy operation_ 也可能不应该被自动生成，因为所生成的 _copy operation_ 是不会做正确的事情的。在 _C++98_ 中，这种思路的重要性没有被完全认识到，所以在 _C++98_ 中，用户声明的析构函数的存在不会影响到编译器生成 _copy operation_。在 _C++11_ 也是这样的，但仅仅是因为对生成 _copy operation_ 的条件做限制会破坏很多 _legacy_ 代码。

然而，_Rule of Three_ 背后的理由仍然是有效的。再结合声明 _copy operation_ 会阻止隐式生成 _move operation_ 的事实，_Rule of Three_ 就推动出了这样的事实：_C++_ 是不会为那些析构函数是用户所声明的类来生成 _move operation_ 的。  

所以，只有当下面这三个条件都满足时，编译器才会在需要时为类生成 _move operation_：
* 没有 _copy operation_ 在类中被声明。
* 没有 _move operation_ 在类中被声明。
* 没有析构函数在类中被声明。
  
在未来的某个时间点，类似的规则也可能会被扩展到 _copy operation_，因为 _C++11_ 也反对为声明有 _copy operation_ 或析构函数的类来自动生成 _copy operation_。这意味着：如果你的代码依赖了那些声明有析构函数或 _copy operation_ 其中之一的类所生成的 _copy operation_ 的话，那么你应该考虑去更新你的代码，以去减小这种依赖。如果编译器所生成的函数的行为是正确的话，即为：对于类的非静态数据成员来说，如果 _memberwise copy_ 是你所想要的话，那么工作就简单了，因为 _C++11_ 的 _=default_ 可以让你显式地说明：  
```C++
  class Widget {
  public:
    …
    ~Widget();                                    // user-declared dtor
    
    …                                             // default copy ctor
    Widget(const Widget&) = default;              // behavior is OK
    
    Widget&                                       // default copy assign
      operator=(const Widget&) = default;         // behavior is OK
    …
  };
```

这个方法对于 _polymorphic base class_ 是有用的，即为：_polymorphic base class_ 是定义有接口的类，并且可以通过这些接口来操作所对应的 _derived class_ 对象。_polymorphic base class_ 一般都有 _virtual_ 析构函数，因为如果析构函数不是 _virtual_ 的话，那么一些操作，比如：_delete_ 指向 _derived class_ 对象的 _base class_ 指针或引用，是会产生未定义的或错误的结果的。除了一个类是继承了一个已经是 _virtual_ 的析构函数以外，只有一种方法可以使这个类的析构函数是 _virtual_ 的了，那就是去显式地声明这个析构函数为 _virtual_ 的。一般情况下，默认实现是正确的，所以 _=default_ 是可以表示这个的正确方法。然而，用户所声明的析构函数是会阻止 _move operation_ 的生成的，所以，如果 _movability_ 是被支持的话，那么 _=default_ 是可以让编译器再次来生成 _move operation_ 的。同样地，声明 _move operation_ 是会阻止 _copy operation_ 的生成的，所以，如果 _copyability_ 也是被支持的话，那么可以再使用一次 _=default_ 来完成工作：  
```C++
  class Base {
  public:
    virtual ~Base() = default;                    // make dtor virtual
    
    Base(Base&&) = default;                       // support moving
    Base& operator=(Base&&) = default;
    
    Base(const Base&) = default;                  // support copying
    Base& operator=(const Base&) = default;

  …
};
```

事实上，即使你有一个这样的类：编译器也会主动为其生成 _copy operation_ 和 _move operation_ 并且所生成的这些函数的行为是如你所愿的，你仍然需要选择这样一个方针：亲自声明这些函数，并使用 _=default_ 来做为这些函数的定义。是多做了点工作，但是这可以让你的目的更加清晰，这可以帮助你规避一些非常不易察觉的 _bug_。例如：假如你有一个表示了字符串表的类，即为：允许通过一个 _integer ID_ 来进行快速查找字符串值的数据结构：  
```C++
  class StringTable {
  public:
    StringTable() {}
    …                         // functions for insertion, erasure, lookup,
                              // etc., but no copy/move/dtor functionality
  private:
    std::map<int, std::string> values;
};
```  
假设这个类没有声明 _copy operation_、_move operation_ 和析构函数，当这些函数被需要时，编译器是会自动生成它们的。这是非常方便的。

但是，假设过段时间后，决定要记录这些对象的的默认构造和析构了。增加这些功能是简单的：  
```C++
  class StringTable {
  public:
    StringTable()
    { makeLogEntry("Creating StringTable object"); }        // added

    ~StringTable()                                          // also
    { makeLogEntry("Destroying StringTable object"); }      // added
    …                                                       // other funcs as before
  
  private:
    std::map<int, std::string> values;                      // as before
  };
```  
这看起来像是合理的，但是声明析构函数会潜在地带来一个重要的副作用：声明析构函数会阻止 _move operation_ 的生成。然而，类所对应的 _copy operation_ 是不受影响的。因此，这个代码是可以被编译和运行的，并且是可以通过功能测试的。包括测试它的移动功能，因为尽管这个类是不支持移动的，但是移动它的请求是仍然可以被编译和运行的。就像在本 _Item_ 之前提到过的，这些请求会导致拷贝被执行。这意味着 **_移动_** _StringTable_ 对象的代码实际上是在进行拷贝此 _StringTable_ 对象，即为：拷贝 _underlying std::map&lt;int, std::string&gt;_ 对象。拷贝一个 _std::map&lt;int, std::string&gt;_ 比移动一个 _std::map&lt;int, std::string&gt;_ 是要慢几个数量级的。因此增加了一个析构函数到类中的这个简单动作可能已经引入了一个重要的性能问题。如果有使用 _=default_ 来显式定义 _copy operation_ 和 _move operation_ 的话，那么这个错误就不会发生了。

是的，我一直在絮叨 _C++11_ 的 _copy operation_ 和 _move operation_ 的操作规则，你已经受够了，你可能会好奇何时我才会将注意力转移到其他两个特殊成员函数上：_default constructor_ 和析构函数。就是现在，但是其实也就一句话，因为几乎没有事情为了这两个函数而改变：在 _C++11_ 中的规则和在 _C++98_ 中的规则几乎是相同。

因此，_C++11_ 的操作特殊成员函数的规则是：
* _default constructor_：和 _C++98_ 的规则相同。只有当类中没有包含用户声明的构造函数时，它才会被生成。
* 析构函数：基本和 _C++98_ 的规则相同；唯一的不同点是默认情况下析构函数是 _noexcept_ 的，见 [_Item 14_](#item-14-当函数不会抛出异常时声明函数为-noexcept)。就像在 _C++98_ 中一样，只有当 _base class_ 的析构函数是  _virtual_ 的时，它才是 _virtual_ 的。
* _copy constructor_：和 _C++98_ 有着相同的运行行为：对非静态数据成员执行 _memberwise copy construction_。只有当类中没有用户声明的 _copy constructor_ 时，它才会被生成。如果类声明了 _move operation_ 的话，那么它会被删除。在有用户声明的
_copy assignment operator_ 或析构函数的类中来生成它是被废弃的。
* _move constructor_ 和 _move assignment operator_：这两个函数都对静态数据成员，执行 _memberwise move_。只有当类中没有包含用户定义的 _copy operations_、_move operation_ 和析构函数时，它才会被生成。

注意：成员函数模板并不会阻止编译器生成特殊成员函数。这意味着：如果 _Widget_ 看起来像下面这样的话：
```C++
  class Widget {
    …
    template<typename T>                // construct Widget
    Widget(const T& rhs);               // from anything
    
    template<typename T>                // assign Widget
    Widget& operator=(const T& rhs);    // from anything
    …
  };
```  
那么编译器仍然会生成 _Widget_ 的 _copy operation_ 和 _move operation_，假设通常的操作这些函数生成的条件是满足的，尽管这些模板可以被实例化去产生出 _copy constructor_ 和 _copy assignment operator_ 的 _signature_，就是当 _T_ 是 _Widget_ 时的场景。很可能，这对你来说只是一个几乎不值得关注的边缘场景，但我提到它是有原因的。[_Item 26_](Chapter%205.md#item-26-避免重载-univeral-reference) 描述了它可以具有重要的后果。

### 需要记住的规则

* 特殊成员函数是编译器能自己生成的函数：_default constructor_、析构函数、_copy operation_ 和 _move operation_。
* _move operation_ 只有当类中没有显式声明的 _move operation_、_copy operation_ 和析构函数时才会被生成。
* _copy constructor_ 只有当类中没有显式声明的 _copy constructor_ 时才会被生成，如果有声明 _move operation_ 的话，那么 _copy constructor_ 会被删除。_copy assignment operator_ 只有当类中没有显式声明的 _copy assignment operator_ 时才会被生成，如果有声明 _move operation_ 的话，那么 _copy assignment operator_ 会被删除。在有用户显式声明的析构函数的类中来生成 _copy operation_ 是被废弃的。
* 成员函数模板永远不会阻止特殊成员函数的生成。