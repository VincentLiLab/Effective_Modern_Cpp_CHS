- [Chapter 3 _Moving to Modern C++_](#chapter-3-moving-to-modern-c)
  - [Item 7 创建对象时区分 _()_ 和 _{}_](#item-7-创建对象时区分--和-)
    - [需要记住的规则](#需要记住的规则)
  - [Item 8 首选 _nullptr_ 而不是 _0_ 和 _NULL_](#item-8-首选-nullptr-而不是-0-和-null)
    - [需要记住的规则](#需要记住的规则-1)
  - [Item 9 首选 _alias declarations_ 而不是 _typedefs_](#item-9-首选-alias-declarations-而不是-typedefs)
    - [需要记住的规则](#需要记住的规则-2)
  - [Item 10 首选 _scoped enums_ 而不是 _unscoped enums_](#item-10-首选-scoped-enums-而不是-unscoped-enums)
    - [需要记住的规则](#需要记住的规则-3)
  - [Item 10 首选 _deleted functions_ 而不是 _private undefined functions_](#item-10-首选-deleted-functions-而不是-private-undefined-functions)

# Chapter 3 _Moving to Modern C++_

_C++11_ 和 _C++14_ 有很多值得夸耀的特性：_auto_、智能指针、移动语义、_lambdas_ 和 _concurrency_，每一个都是非  
常重要的。我专门写了这一章来介绍它们。掌握这些特性是必不可少的，但是需要一系列更小的步骤才能变为一名  
高效率的 _modern C++_ 的编程者。每一个步骤都回答了在 _C++98_ 转向 _modern C++_ 的过程中所产生的问题。什么  
时候应该使用 _{}_ 代替 _()_ 来创建对象？为什么 _alias declartions_ 要比 _typedefs_ 更好呢？_constexpr_ 和 _const_ 是如何的  
不同呢？_const_ 成员函数和线程安全之间有什么关系？还有很多，本章一个接着一个来进行解答。

## Item 7 创建对象时区分 _()_ 和 _{}_

取决于你的观点，_C++11_ 中的对象的初始化语法的选择要么是令人尴尬的富裕，要么是令人困惑的混乱。一般而  
言，初始化值可以使用 _()_、_=_ 和 _{}_ 来进行指定：
```C++
  int x(0);                   // initializer is in parentheses
  
  int y = 0;                  // initializer follows "="
  
  int z{ 0 };                 // initializer is in braces
```

在很多场景下，_=_ 和 _{}_ 也可以一起使用：  
```C++
  int z = { 0 };              // initializer uses "=" and braces
```

对于本 _Item_ 的剩余部分，我通常会忽略 _equals-sign-plus-braces_ 语法，因为 _C++_ 通常把它和 _braces-only_ 做同样  
地处理。 

> 实际测试不是这样，使用的编译器是 _c++ (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0_。_auto v{0, 1};_ 这样是无法通过  
> 编译的并且 _auto v{0}_ 中的 _v_ 是 _int_ 类型而不是 _std::initializer_lists_ 类型。

**_令人困惑的混乱_** 派指出：使用 _=_ 来进行初始化常常会让 _C++_ 菜鸟误认为 _assignment_ 发生了，实际并不是。对于  
像 _int_ 这样的内建类型来说只是学术上的不同，但是对于用户定义的类型来说，区分 _initialization_ 和 _assignment_  
就非常重要了，因为这会涉及到不同的函数：  
```C++
  Widget w1;                  // call default constructor
  
  Widget w2 = w1;             // not an assignment; calls copy ctor
  
  w1 = w2;                    // an assignment; calls copy operator=
```
即使有着这么多的初始化语法，但是 _C++98_ 仍然没有办法去表达一个所期待的初始化。例如：不能直接创建持有  
一组特定值的 _STL_ _container_，比如：_1，2，5_。

为了解决这么多个初始化语法的所带来困惑和这些初始化语法并不能覆盖全部初始化情况的事实，_C++11_ 引入了  
_uniform initialization_：一个可以，至少概念上可以，被用在任意地方和表达任意事情的单一的初始化语法。它是  
基于 _{}_ 的，所以我更喜欢 _braced initialization_ 这个术语。**_uniform initialization_** 是思想，**_braced initialization_** 是语法构成。
            
_braced initialization_ 让你能表达之前不能被表达出的内容，使用 _{}_，非常容易指定一个 _container_ 的初始化内容：  
```C++
  std::vector<int> v{ 1, 3, 5 };        // v's initial content is 1, 3, 5
```

_{}_ 也可以被用于指明非静态数据成员的默认初始化值。这个是新添加到 _C++11_ 的中的功能，_=_ 也可以完成这个功  
能，但是 _()_ 不可以：  
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

这也就能理解了为什么 _braced initialization_ 被称为 **_uniform_**。在 _C++_ 的三种指定初始化表达式的方法中，只有 _{}_   
可以被用在所有地方。

_braced initialization_ 的新颖特性是可以禁止内建类型之间的 _implicit narrowing conversions_。如果 _braced initializer_  
中的表达式的值不能保证被正在被初始化的对象的类型所表达的话，那么就不能通过编译：  
```C++
  double x, y, z;
  
  …
  
  int sum1{ x + y + z };      // error! sum of doubles may
                              // not be expressible as int
```

使用 _()_ 和 _=_ 进行初始化不会检查 _narrowing conversions_，因为这会破坏很多的 _legacy_ 代码：  
```C++
  int sum2(x + y + z);        // okay (value of expression
                              // truncated to an int)

  int sum3 = x + y + z;       // ditto
```

_braced initialization_ 的另一个值得关注的特性是它对于 _C++_ 的 _most vexing parse_ 是免疫的。在 _C++_ 中有一个规  
则：任何可以被解析为声明的事物都必须被解释为声明，这个规则是有副作用的，那就是 _most vexing parse_ 会让  
开发者在想要调用默认构造函数时却不经意间声明出了一个函数。这个问题的根源在于，如果你想要调用实参构造  
函数的话，你可以这样做：  
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

因此，对于 _braced initialization_ 就有很多值得一提的地方了。它是可以在最广泛的语境下被使用的语法、它可以  
避免 _implicit narrowing conversions_，并且它对于 _C++_ 的 _most vexing parse_ 是免疫的。三连冠！那为什么本 _Item_  
不命名为“首选 _braced initialization_ 语法”呢？

_braced initialization_ 的缺点是有时会出现令人惊讶的行为。这些行为源自于 _braced initializers_、_std::initializer_lists_  
和构造函数重载决议之间的异常复杂的关系。它们的相互作用可能会导致代码看起来像是在做一件事，但实际上却  
是在做另一件事。例如：[_Item 2_](./Chapter%201.md#item-2-理解-auto-的类型推导) 解释了：当 _auto_ 声明的变量有 _braced initializer_ 时，所推导出的这个变量的类型  
是 _std::initializer_list_ 的，尽管用其他方式声明带有相同 _initializer_ 的变量可能会产生更直观的类型。结果就是你越  
喜欢使用 _auto_，你就越不喜欢 _braced initialization_。

在构造函数调用中，只要不涉及到 _std::initializer_lists_ 的话，那么 _{}_ 和 _=_ 就有着相同的意义：  
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
然而，如果一个或多个构造函数声明了类型 _std::initializer_lists_ 的形参的话，那么使用了 _braced initialization_ 语法  
的调用会强烈地优选持有 _std::initializer_lists_ 的重载函数。是强烈地。如果编译器有方法将使用了 _braced initializer_  
的调用解释为持有 _std::initializer_lists_ 的构造函数的话，那么编译器将会采用这种解释。如果上面的 _Widget_ 类增加  
了持有 _std::initializer_list&lt;long double&gt;_ 的构造函数的话，例如：  
```C++
  class Widget {
  public:
    Widget(int i, bool b);                                  // as before
    Widget(int i, double d);                                // as before
    Widget(std::initializer_list<long double> il);          // added

  …
};
```  

_Widgets_ _w2_ 和 _w4_ 将使用新的构造函数来进行构造，虽然 _std::initializer_list_ 的元素的类型 _long double_ 相对于其他  
的 _non-std::initializer_list_ 构造函数来说是对所有实参的更糟糕的匹配！看：
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

编译器对持有 _std::initializer_list_ 的构造函数和 _braced initializers_ 进行匹配的决定是非常强烈的，即使最匹配的  
_std::initializer_list_ 构造器不可以被调用，它也是占上风的。比如：
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
此处，编译器将会忽略前两个构造函数，尽管其中的第二个构造函数提供了对全部实参类型的准确匹配，然后尝试  
去调用持有 _std::initializer_list&lt;bool&gt;_ 的构造函数。调用这个构造函数需要将 _int(10)_ 和 _double(5.0)_ 转换为 _bools_。  
这两个转换都是 _narrowing_ 的，_bool_ 不能准确地表示这两个值，_braced initializers_ 是禁止 _narrowing conversions_  
的，所以这个调用是无效的，然后代码无法通过编译。

只有当没有办法将 _braced initializer_ 中的实参的类型转换为 _std::initializer_list_ 中的类型时，编译器才会回退到一般  
的重载决议中。例如：如果我们使用 _std::initializer_list&lt;std::string&gt;_ 的构造函数来代替 _std::initializer_list&lt;bool&gt;_ 的  
话，那么 _non-std::initializer_list_ 构造函数将再次成为候选，因为没有办法转换将 _ints_ 和 _bools_ 转换为 _std::strings_：
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

这让我们靠近 _braced initializer_ 和构造函数重载的讨论的末尾了，但是仍然有令人感兴趣的边缘场景需要被解决。  
假定你使用了一个空 _{}_ 去构造一个对象，并且这个对象支持默认构造和 _std::initializer_list_ 构造。空 _{}_ 意味着什么  
呢？如果是 **_no arguments_** 的话，那么调用的应该是默认构造函数，如果是 **_empty std::initializer_list_** 的话，那么  
调用的应该是没有元素的 _std::initializer_list_ 构造函数。

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

如果你想要调用 _empty std::initializer_list_ 的 _std::initializer_list_ 构造函数的话，那么可以通过空 _{}_ 构造函数实参来完  
成：通过 _({})_ 或 _{{}}_ 来清楚表明你正在传递的内容：
```C++
  Widget w4({});              // calls std::initializer_list ctor
                              // with empty list
  
  Widget w5{{}};              // ditto
```

此时，_braced initializers_、_std::initializer_list_ 和构造函数重载这些看起来是晦涩难懂的规则在你的脑海中萦绕，你  
可能正在好奇这些在日常编程中又有多重要。比你想的要重要，因为直接受影响的类便是 _std::vector_。_std::vector_  
有 _non-std::initializer_list_ 构造函数，这个构造函数允许你指明 _container_ 的大小和每个元素的初始值，_std::vector_  
也有 _std::initializer_list_ 构造函数，这个构造函数允许你指明 _container_ 的初始值。如果你创建了 _numeric_ 类型的  
_std::vector_ 的话，比如：_std::vectort&lt;int&gt;_，然后你传递了两个实参到构造函数中，使用 _{}_ 或 _()_ 会有巨大的不同：  
```C++
  std::vector<int> v1(10, 20);          // use non-std::initializer_list
                                        // ctor: create 10-element
                                        // std::vector, all elements have
                                        // value of 20

std::vector<int> v2{10, 20};            // use std::initializer_list ctor:
                                        // create 2-element std::vector,
                                        // element values are 10 and 20
```

但是让我们从 _std::vector_ 中回退一步，也从 _()_、_{}_ 以及构造函数重载决议规则的细节中回退一步。从这个讨论中可  
以得到两个重点。首先，做为一个类实现者，你需要了解：如果你的一组重载构造函数中包含了一个或者多个持有  
_std::initializer_list_ 的函数的话，那么使用了 _braced initialization_ 的客户代码可能就只会看到 _std::initializer_list_ 类型  
的重载函数了。总之，要精心设计你的构造函数，让重载函数不被用户使用 _()_ 还是 _{}_ 所影响。换句话说，现在要  
把 _std::vector_ 的设计看成是错误的，应该从中学习并让你的类去避免发生这种错误。

一个可能的影响是：如果你有一个原先并没有 _std::initializer_list_ 构造函数的类，然后你添加了一个的话，那么使用  
_braced initialization_ 的客户代码可能会认为：原先调用的是 _non-std::initializer_list_ 构造函数，现在调用的是新添加  
的构造函数了。当然，这种事情可以发生在任何你添加新的函数到一组重载函数的时候：原先调用的是旧的重载函  
数，现在开始调用的是新的重载函数了。和 _std::initializer_list_ 构造函数重载不同的是：_std::initializer_list_ 重载函数  
不只是会和其他的重载函数进行竞争，而是会遮盖其他的重载函数，其他的重载函数很难再被考虑了。所以只有在  
深思熟虑后，再去添加这些 _std::initializer_list_ 重载函数。

其次：做为一个类用户，当创建对象时，你必须在使用 _()_ 还是使用 _{}_ 之间做谨慎选择。大多数的开发者会选择其  
中一种来做为其默认，只有当必须要使用另一种时，才会去使用另一种。 _{}_ 的无与伦比的适用性广度、_{}_ 可以禁  
止 _narrowing conversions_ 以及 _{}_ 对 _C++_ 的 _most vexing parse_ 的免疫都吸引了 _Braces-by-default_ 派。他们认为只  
有在一些场景下，比如：在创建给定大小和每个元素的初始值的 _std::vector_ 时，才需要去使用 _()_。而在另一方面， 
_go-parentheses-go_ 派则将 _()_ 做为其默认。_()_ 和 _C++98_ 的语法一致、_()_ 可以避免 _auto-deduced-a-std::initializer_list_  
的问题以及创建对象的调用不会被 _std::initializer_list_ 构造函数所无意地拦截都吸引着 _go-parentheses-go_ 派。他们  
承认有时候只有 _{}_ 才能完成一些事，比如：当创建一个有着特定值的 _container_ 时。哪个更好是没有共识的，所以  
我的建议是选择一个，然后一直使用它。

如果你是一个模板实现者的话，那么在创建对象时 _{}_ 和 _()_ 之间的紧张关系会令人更沮丧，因为一般不可能知道哪  
个应该被使用。例如：假定你从任意数量的实参中创建一个任意类型的对象。可变参数模板让这一切在概念上变得  
直观：  
```C++
  template<typename T,                  // type of object to create
  typename... Ts>                       // types of arguments to use
  void doSomeWork(Ts&&... params)
  {
    create local T object from params...
    …
  }
```

有两种方法将伪代码转为真实代码，对于 _std::forward_ 的信息见 [_Item 25_](./Chapter%205.md#item-25-std::move-用于右值引用-std::forward-用于-univeral-reference)：
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

如果创建 _localObject_ 时 _doSomeWork_ 使用了 _()_ 的话，那么 _std::vector_ 就是有 _10_ 个元素。如果创建 _localObject_ 时   
_doSomeWork_ 使用了 _{}_ 的话，那么 _std::vector_ 就是有 _2_ 个元素。哪一个是正确的？_doSomeWork_ 的作者是不知道  
的，只有调用者知道。

这是标准库函数 _std::make_unique_ 和 _std::make_shared_ 面对的问题，见 [_Item 21_](./Chapter%204.md#item-21-首选-std::make-unique-和-std::make-shared-而不是直接使用-new)。这些函数通过内部使用 _()_ 和记录  
这个决定做为接口的一部分来解决这个问题。

### 需要记住的规则

* braced initialization_ 是最广泛的可使用的初始化语法，它可以禁止 _narrowing conversions_ 并且对对 _C++_ 的  
_most vexing parse_ 所免疫。  
* 在构造函数重载决议期间，如果可能，_braced initializer_ 会和 _std::initializer_list_ 形参匹配，即
使其他的构造函  
数提供了看起来是更好的匹配。
* 在选择使用 _()_ 和 _{}_ 时可能产生显著差异的一个例子是使用两个实参来创建 _std::vector&lt;numeric type&gt;_ 时。
* 当在模板中创建对象时，在 _()_ 和 _{}_ 之间进行选择是具有挑战性的。

## Item 8 首选 _nullptr_ 而不是 _0_ 和 _NULL_

是这样的：字面上的 _0_ 是 _int_，而不是一个指针。如果 _C++_ 是在只有指针被使用的环境下看到了 _0_ 的话，那么是
会勉强将 _0_ 解释为空指针的，但这是一个次选方案。_C++_ 的主要方针是 _0_ 是 _int_，而不是指针。

实际上，对于 _NULL_ 也是这样的。在 _NULL_ 的场景中是存在有一些细节上的不确定性的，这是因为实现允许 _NULL_  
是 _integral_ 类型而不是 _int_ 类型的，比如：_long_。这是不常见的，但是不重要，因为问题不在于 _NULL_ 的实际类型  
是什么，而是 _0_ 和 _NULL_ 都不是指针类型。

在 _C++98_ 中，一个可能的影响是：重载指针类型和 _integral_ 类型时可能会导致惊喜。传递 _0_ 或 _NULL_ 到这些重载  
函数时，永远不会调用到指针的重载函数：
```C++
  void f(int);                // three overloads of f
  void f(bool);
  void f(void*);
  
  f(0);                       // calls f(int), not f(void*)
  f(NULL);                    // might not compile, but typically calls
                              // f(int). Never calls f(void*)
```
_f(NULL)_ 的行为的不确定性反映出了在执行 _NULL_ 的类型时所给予的余地。如果 _NULL_ 被定义为是 _0L_ 的话，比如： 
_0_ 是 _long_，那么这个调用是具有二义性的，因为从 _long_ 到 _int_、从 _long_ 到 _bool_ 以及从 _0L_ 到 _void*_ 的 _conversion_  
都被认为是一样地好的。关于这个调用的有意思的地方是在于这个调用 _表面上的_ 含义和 _实际上的_ 含义是不一致  
的：使用 _NULL_ 来调用 _f_ 时是空指针，而使用 _integer_ 来调用 _f_ 则不是空指针。这种违反直觉的行为产生了 _C++98_  
程序员要避免重载指针类型和 _integral_ 类型的这样的准则。这个准则在 _C++11_ 中仍然有效，因为尽管有本 _Item_ 的  
建议，但是有一些开发者仍然会继续 _0_ 和 _NULL_，虽然 _nullptr_ 是更好的选择。

_nullptr_ 的优势是它不是 _integral_ 类型。老实说，它也不是指针类型，但是你可以认为它是所有类型的指针。它的  
实际类型是 _std::nullptr_t_，在一个完美的循环定义下，_std::nullptr_t_ 是被定义为 _nullptr_ 的类型的。_std::nullptr_t_ 可  
以隐式地被转换为原始指针类型，表现得就好像是它是所有类型的指针一样。

使用 _nullptr_ 调用重载函数 _f_ 会调用 _void*_ 重载函数，即为：指针重载函数，因为 _nullptr_ 不可以被做为 _integral_ 的：
```C++
  f(nullptr);                 // calls f(void*) overload
```  
因此，使用 _nullptr_ 来代替 _0_ 或 _NULL_ 可以避免重载决议的惊喜，但这不是唯一的优势。它还可以提高代码的清晰  
性，特别是当涉及到 _auto_ 变量时。例如：假如你在代码中遇到了下面这种情况：  
```C++
  auto result = findRecord( /* arguments */ );
  
  if (result == 0) {
    …
  }
```  
如果你碰巧不知道或者不容易发现 _findRecord_ 返回的是什么的话，那么就不清楚 _result_ 是指针类型还是 _integral_ 类  
型了。毕竟，这个 _result_ 所比较的 _0_ 可以是指针类型，也可以是 _integral_ 类型。另一方面，如果你看到的是下面这  
样的代码的话：  
```C++
  auto result = findRecord( /* arguments */ );
  
  if (result == nullptr) {
    …
  }
```  
那么就没有二义性了：_result_ 必须是指针类型。

当使用模板时，_nullptr_ 尤其地闪亮。假定你有一些函数，这些函数只有在合适的 _mutex_ 被锁定时才会被调用。每  
一个函数都持有不同类型的指针：
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
在这个代码中的前两个调用中没有使用 _nullptr_ 是糟糕的，但是代码是可以工作的。这是重要的。在所调用的代码  
中的所重复的 _pattern_：锁定 _mutex_、调用函数、解锁 _mutex_，是更糟糕的。这样的复用代码是要去设计模板去避
免的事情之一，所以让我们模板化这个 _pattern_：  
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
如果这个函数的返回类型 _auto … -> decltype(func(ptr)_ 让你挠头的话，那么帮你的头个忙，请转到 [_Item 3_](./Chapter%201.md#item-3-理解-decltype)，其中解  
释了这是发生了什么。在 _C++14_ 中，这个返回的类型可以被简化为 _declare(auto)_：  
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

对于给定的 _lockAndCall_ 模板，另一个版本，调用者可以像下面这样写代码：
```C++
  auto result1 = lockAndCall(f1, f1m, 0);         // error!
  
  …
  
  auto result2 = lockAndCall(f2, f2m, NULL);      // error!  
  
  …

  auto result3 = lockAndCall(f3, f3m, nullptr);   // fine
```  
好吧，可以这样写，但是正如注释所说明的，三个调用中的两个都会编译失败。第一个调用中的问题是：当将 _0_  
被传递给 _lockAndCall_ 时，会使用模板的类型推导来弄清楚它的类型是什么。_0_ 的类型现在、过去和未来都是 _int_，  
所以 _int_ 是 _lockAndCall_ 调用的实例化的形参 _ptr_ 的类型。不幸地是，这意味着在 _lockAndCall_ 中的 _func_ 调用中所  
传递的是 _int_ ，这和 _f1_ 所期待的 _std::shared_ptr&lt;Widget&gt;_ 形参是不兼容的。在 _lockAndCall_ 调用中的所传递的 _0_  
原本想要表示的是空指针，但是实际上所传递却是 _run-of-the-mill_ _int_。尝试将 _int_ 做为 _std::shared_ptr&lt;Widget&gt;_  
来传递到 _f1_ 是类型错误的。使用 _0_ 来调用 _lockAndCall_ 是会失败的，因为在模板中，_int_ 是被传递给 _func_ 函数的， 
而 _func_ 函数所需要的却是 _std::shared_ptr&lt;Widget&gt;_。

对于涉及到 _NULL_ 的调用的分析本质上也是一样的。当 _NULL_ 被传递给 _lockAndCall_ 时，对于形参 _ptr_ 来说，所推  
导出的是 _integral_ 类型，而当 _int_ 类型或 _int-like_ 类型的 _ptr_ 被传递给 _f2_ 时，会发生类型错误，这是因为 _f2_ 期待的  
是 _std::unique_ptr&lt;Widget&gt;_。

相比之下，涉及到 _nullptr_ 的调用就不会有这种问题。当 _nullptr_ 被传递给 _lockAndCall_ 时，_ptr_ 的类型是被推导为  
_std::nullptr_t_ 的。当 _ptr_ 被传递给 _f3_ 时，会有从 _std::nullptr_t_ 到  _Widget*_ 的隐式转换，因为 _std::nullptr_t_ 可以隐式  
转为所有指针类型。

模板的类型推导会推导出 _0_ 和 _NULL_ 所对应的 **_错误_** 的类型，即为：它们的真实类型，而不是表示为空指针的备选  
方案，当你想要引用一个空指针时，这是你使用 _nullptr_ 来代替 _0_ 和 _NULL_ 的最有说服力的理由。使用 _nullptr_ 不会  
给模板带来特殊的挑战。再结合 _nullptr_ 不会遭遇 _0_ 和 _NULL_ 容易遭遇的重载决议问题的事实的话，那么结论就无  
懈可击了。当你想要引用空指针时，使用 _nullptr_，而不是 _0_ 或 _NULL_。

### 需要记住的规则

* 首选 _nullptr_ 而不是 _0_ 和 _NULL_。
* 避免重载 _integral_ 类型和指针类型。

## Item 9 首选 _alias declarations_ 而不是 _typedefs_

我相信我们都同意使用 _STL_ 的 _containers_ 是一个好注意，我也希望 [_Item 18_](./Chapter%204.md#item-18-对于-exclusive-ownership-的资源管理使用-std::unique_ptr) 可以说服你使用 _std::unique_ptr_ 是一个  
好主意。而且我认为我们都不喜欢写像 _std::unique_ptr&lt;std::unordered_map&lt;std::string, std::string&gt;&gt;_ 这样的代码多  
过于一次。仅是想想这个就会增加 _carpal tunnel syndrome_ 的风险。

避免这样的医学悲剧是简单的。引入了 _typedef_：
```C++
  typedef
    std::unique_ptr<std::unordered_map<std::string, std::string>>
    UPtrMapSS;
```  
但是，_typedefs_ 太 _C++98_ 了。_typedefs_ 可以在 _C++11_ 中工作，但是 _C++11_ 也提供了 _alias declarations_：
```C++
  using UPtrMapSS =
    std::unique_ptr<std::unordered_map<std::string, std::string>>;
```

鉴于 _typedef_ 和 _alias declaration_ 实际上做的是相同的事情，所以就有理由怀疑是否有足够的技术理由来让我们喜  
欢其中的一个。

是存在的，但是在我说之前，我想说下：当处理涉及到函数指针的类型时，很多人发现 _alias declaration_ 是容易理  
解的：
```C++
  // FP is a synonym for a pointer to a function taking an int and
  // a const std::string& and returning nothing
  typedef void (*FP)(int, const std::string&);    // typedef

  // same meaning as above
  using FP = void (*)(int, const std::string&);   // alias
                                                  // declaration
```  
当然，这两种形式都不是那么容易理解的，因为很少有人愿意花时间来处理函数指针类型的同义词，所以这不是选  
择 _alias declarations_ 而不是 _typedefs_ 的最有说服力的理由。


但是，最有说服力的理由是存在的，那就是模板。特别是 _alias declarations_ 是可以被模板化的，我们将模板化的  
_alias declarations_ 称之为 _alias templates_，而 _typedefs_ 是不可以被模板化的。这为 _C++11_ 的程序员提供了一个直  
接的机制来表达那些在 _C++98_ 中必须通过嵌套在模板化的 _structs_ 中的 _typedefs_ 才能拼凑出来的东西。例如：考  
虑一个使用了定制 _allocator_ _MyAlloc_ 的 _linked list_ 的同义词的定义。使用 _alias template_ 时，这就是小菜一碟：
```C
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
这会更糟。如果你想要在模板类中使用 _typedef_ 来创建一个持有模板形参所指定的类型的对象的 _linked list_ 的话，  
那么你必须在 _typedef_ 前加上 _typename_：
```C++
  template<typename T>
  class Widget {                                  // Widget<T> contains
  private:                                        // a MyAllocList<T>
    typename MyAllocList<T>::type list;           // as a data member
    …
  };
```  
此处，_MyAllocList&lt;T&gt;::type_ 引用了一个依赖于模板类型形参 _T_ 的类型。因此它是一个 _dependent type_。_C++_ 的众  
多的 **_可爱的_** 规则的其中一个便是必须在 _dependent types_ 的名字前加上 _typename_。

如果 _MyAllocList_ 是被定义来做为 _alias template_ 的话，那么就不需要 _typename_ 了，繁琐的后缀_::type_ 也不需要  
了：
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

对于你来说，_MyAllocList&lt;T&gt;_ 它使用了 _alias template_ 看起来可能和 _MyAllocList&lt;T&gt;::type_ 它使用了嵌套的 _typedef_  
一样都是依赖于模板形参 _T_ 的，但是你不是编译器。当编译器处理 _Widget_ 模板和遇到 _MyAllocList&lt;T&gt;_ 也就是遇  
到 _alias template_ 时，编译器知道 _MyAllocList&lt;T&gt;_ 是一个类型的名字，因为它是一个 _alias template_，所以它必然  
命名了一个类型。因此 _MyAllocList&lt;T&gt; 是一个 _non-dependent type_，所以 _typename spcifier_ 就既不被需要也不  
被允许了。

另一方面，当编译器在 _Widget_ 模板中看到 _MyAllocList&lt;T&gt;::type_ 也就是使用了嵌套
的 _typedef_ 时，编译器不能确  
定 _MyAllocList&lt;T&gt;::type_ 命名了一个类型，因为还可能存在编译器所看不到的 _MyAllocList_ 的 _specialization_，在其  
中 _MyAllocList&lt;T&gt;::type_ 引用的不是一个类型。这听起来是疯狂的，但是对于这个可能性不要怪编译器。人们真能  
写出这样的代码。  

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
正如你看到的，_MyAllocList&lt;Wine&gt;::type_ 没有引用类型。如果 _Widget_ 是 _Wine_ 所实例化的话，那么 _Widget_ 模板  
中的 _MyAllocList&lt;Wine&gt;::type_ 引用的就是数据成员，而不是一个类型。在 _Widget_ 模板中的 _MyAllocList&lt;T&gt;::type_  
是否引用类型是老老实实地依赖于 _T_ 是什么的，这也是为什么编译器坚持要在类型前面加上 _typename_ 来声明它是  
类型。

如果你做过任意模板元编程 _TMP_ 的话，那么你几乎肯定遇到过这样的需求：获取模板类型形参，然后根据它来创  
建修改过的类型。例如：给定一个类型 _T_，你可能想要剥离 _T_ 所包含的 _const-ualifiers_ 或 _reference-qualifiers_，比  
如：你可能想要将 _const std::string&_ 转换为 _std::string_。或者想要在类型前添加 _const_ 或将其转换为左值引用，比  
如：将 _Widget_ 转换为 _const Widget_ 或 _Widget&_。如果你没有接触过一点 _TMP_，那就太糟糕了，因为如果你想要  
成为一位真正地高效率的 _C++_ 编程者的话，那么你需要至少熟悉 _C++_ 的这部分的基础。你可以去看 [_Item 23_](./Chapter%205.md#item-21-理解-std::move-和-std::forward) 和  
[_Item 25_](./Chapter%205.md#item-25-熟悉重载-univeral-references-的替代方法) 中的 _TMP_ 的实战例子，包括我刚才提到的那几种类型转换。

_C++_ 给了你工具可以按照 _type traits_ 的形式来执行那几种类型转换，它们是在头文件 _&lt;type_trsits&gt;_ 中的一些模  
板。在这个头文件中有很多 _type traits_，它们都提供了预期的接口，但不是都执行的是类型转换。给定一个类型  
_T_，你想要对它应用转换，其结果类型就是 _std::transformation&lt;T&gt;::type_ 了。例如：  
```C++
  std::remove_const<T>::type            // yields T from const T
  std::remove_reference<T>::type        // yields T from T& and T&&
  std::add_lvalue_reference<T>::type    // yields T& from T
```  
注释只是总结了这些转换做了什么，不要太咬文嚼字。我知道在项目上使用它们之前，你会查询精确的规格说明文  
档。

总之，此处我的目的不是给你 _type traits_ 的 _tutorial_。而是要注意这些转换的应用都需要在每次使用时写 _::type_。如  
果你想要在模板中应用这些转换到类型形参上的话，在实际的代码中几乎总是这样使用，
那么在每次使用时你也  
必须都加上 _typename_。这两个语法减速带存在的原因是 _C++11_ 的 _type traits_ 是用通过嵌套在模板化的 _structs_ 中  
的 _typedefs_ 来实现的。是的，是使用我一直试图说服你的比 _alias templates_ 要差的类型同义词技术来实现的！

这是有历史原因的，但是我们不谈论这个历史原因，因为非常傻，我保证，因为 _Standardization Committee_ 后来  
意识到了 _alias templates_ 是更好的方式，并在 _C++14_ 中为 _C++11_ 的类型转换提供了相应的模板。_aliases_ 有通用  
的形式：对每一个 _C++11_ 的转换 _std::transformation<T>::type_ 都有所对应的被命名为 _std::transformation_t_ 的 _C++14_  
的 _alias template_。例子会说明我的意思： 
```C++
  std::remove_const<T>::type            // C++11: const T → T
  std::remove_const_t<T>                // C++14 equivalent
  
  std::remove_reference<T>::type        // C++11: T&/T&& → T
  std::remove_reference_t<T>            // C++14 equivalent
  
  std::add_lvalue_reference<T>::type    // C++11: T → T&
  std::add_lvalue_reference_t<T>        // C++14 equivalent
```

_C++11_ 所对应的方法在 _C++14_ 中仍然有效，但是我不知道你为什么还要使用它们。即使你不能使用 _C++14_，自  
己写 _alias template_ 也是非常简单的。只需要 _C++11_ 的语言特性，甚至孩子们也可以模仿这种模式，对吧？如果  
你有 _C++14_ 标准的电子版副本的话，仍然是简单的，因为只需要一些复制粘贴。在此处，我帮你开始吧：  
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

* _typedef_ 不支持模板化，而 _alias declarations_ 支持。
* _alias templates_ 可以避免 _::type_ 后缀，并且在模板中引用 _typedefs_ 时常常需要加上 _typename_ 前缀。
* _C++14_ 为 _C++11_ 的 _type traits_ 转换都提供了所对应的 _alias temmplates_。

## Item 10 首选 _scoped enums_ 而不是 _unscoped enums_

一般而言，在 _{}_ 内声明一个名称会将这个名称的可见性限制在 _{}_ 所定义的作用域内。对于使用了 _C++98-style_ 的  
_enums_ 所声明的 _enumerators_ 来说却不是这样。这些 _enumerators_ 的名称是属于那个包含着它所对应的 _enum_ 的  
作用域的。这意味着在这个作用域内不能有相同的名称：
```C++
  enum Color { black, white, red };     // black, white, red are
                                        // in same scope as Color

  auto white = false;                   // error! white already
                                        // declared in this scope
```

这些 _enumerators_ 的名称是被泄露到了那个包含着它所对应的 _enum_ 的作用域的事实产生了这种 _enum_ 所对应的  
官方术语：_unscoped_。_C++11_ 有所对应的 _scoped enums_，不会发生泄露：
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

因为 _scoped enums_ 是通过 _enum class_ 来声明的，所以有时候称为 _enum classes_。 

_scoped enums_ 降低了 _namespace_ 的污染，这个就足以选择 _scoped enums_ 而不是 _unscoped enums_ 了，但是还有  
第二个有说服力的优势：它们的 _enumerators_ 是更 _strongly typed_。_unscoped enums_ 所对应的 _enumerators_ 是可以  
被隐式转换为 _integral_ 类型的，并是能够进一步被转换为 _floating-point_ 类型的。因此，像下面这样的扭曲语义完  
全是有效的：  
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

然而，在 _enum_ 之后加一个简单的 _class_ 将 _unscoped enums_ 转换为 _scoped enums_ 后，就是不同的故事了。不可以
将 _scoped enums_ 所对应的 _enumerators_ 隐式转换为到其他类型：
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

如果真地想要执行从 _Color_ 到不同的类型的转换，去做你一直做的，去扭曲类型系统来满足你那肆意的欲望：使用  
_cast_：
```C++
  if (static_cast<double>(c) < 14.5) {            // odd code, but
                                                  // it's valid

    auto factors =                                // suspect, but
      primeFactors(static_cast<std::size_t>(c));  // it compiles
    …
  }
```

_scoped enums_ 相对于 _unscoped enums_ 似乎还有的第三个优势，因为 _scoped enums_ 是可以进行前置声明的，即  
为：在没有指明 _enumerators_ 的时候，就可以声明 _scoped enums_：
```C++
  enum Color;                 // error!

  enum class Color;           // fine
```  
这有点误导人。在 _C++11_ 中，_unscoped enums_ 也是可以进行前置声明的，但是需要一点额外的工作。这点额外  
的工作源于这样的事实：在 _C++_ 中每个 _enum_ 都有一个编译器所决定的 _underlying type_，是 _integral_ 的。对于像  
_Color_ 的 _unscoped enums_，  
```C++
  enum Color { black, white, red };
```  
编译器可能选择 _char_ 来做为 _underlying type_，因为只需要三个值来表示。然而，一些 _enums_ 有一组更大范围的  
值，比如：  
```C++
  enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                indeterminate = 0xFFFFFFFF
              };
```  
此处所表示的范围是 _0_ 到 _0xFFFFFFFF_。除了不常见的机器外，这些机器的 _char_ 至少有 _32bit_，编译器都将会选择  
大于 _char_ 的 _integral_ 类型来表示 _Status_ 值。

为了高效率的使用内存，编译器通常想要为 _enum_ 来选择最小的 _underlying type_，只要能足够表示 
_enumerator_ 的  
值的范围就可以了。在一些场景中，编译器优化的是速度而不是大小，在这些场景中，编译器可能不会选择所允许  
的最小的 _underlying type_，但是仍然想要能够优化大小。为了可以这样做，_C++98_ 就只提供了 _enum_ 定义，就是  
要列出全部的 _enumerators_，而 _enum_ 声明是被不允许的。这使得编译器可以在每个 _enum_ 被使用之前来为它们来  
选择一个 _underlying type_ 了。

但是无法前置声明 _enum_ 也有缺点。最明显的大概就是增加了编译依赖。再一次考虑 _Status_ _enum_：
```C++
  enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                indeterminate = 0xFFFFFFFF
              };
```   
这种类型的 _enum_ 可能会被整个系统所使用，它会被放在一个头文件中，而系统中的每一个部分都包含有这个头  
文件。如果新的状态值被引入的话：  
```C
 enum Status { good = 0,
                failed = 1,
                incomplete = 100,
                corrupt = 200,
                audited = 500,
                indeterminate = 0xFFFFFFFF
              };                   
```  
整个系统都必须被重新编译。即使只有一个单独的子系统中的一个单独的函数使用了这个的新的 _enumerator_ 而  
已。人们都很讨厌这种事情。在 _C++11_ 中可以前置声明 _enum_ 来避免了这样的事情。例如：有一个 _scoped enum_  
的完美有效的声明和一个函数，这个函数的形参的类型是这个 _scoped enum_：
```C++
  enum class Status;                    // forward declaration
  
  void continueProcessing(Status s);    // use of fwd-declared enum
```

如果 _Status_ 的定义被更改了的话，那么包含有这个声明的头文件是不需要被重新编译的。此外，如果 _Status_
被更  
改了，比如：增加了 _enumerator_ _audited_，但是 _continueProcessing_ 的行为未被影响的话，比如：它并没有使用到  
_audited_，那么 _continueProcessing_ 的实现也是不需要被重新编译的。

但是，如果编译器需要在一个 _enum_ 被使用之前就必须得要知道这个 _enum_ 的大小的话，那么为什么 _C++11_ 可以  
进行前置声明而 _C++98_ 却不可以呢？答案是简单的：_scoped enum_ 所对应的的 _underlying type_ 总是已知的，而对  
于 _unscoped enum_ 所对应的的 _underlying type_ 则不是，但是你是可以指定的。

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

不管怎样，编译器是知道 _scoped enum_ 中的 _enumerators_ 的大小的。

为了指明 _unscoped enum_ 所对应的的 _underlying type_，需要和 _scoped enum_ 做同样的事情，那么这样就可以进行  
前置声明了：
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

因为考虑到 _scoped enum_ 可以避免 _namespace_ 的污染而且不容易受到无意间的隐式类型转换的影响，所以当你听  
到 _unscoped enum_ 还有有用的使用情景时，你可能是会感到惊讶的。这个有用的使用情景就是当引用 _C++11_ 中的  
_std::tuples_ 的各个域时。例如：假定我们有一个持有名字、邮箱地址和用户在社交网站上的信誉值的 _tuple_：
```C++
  using UserInfo =            // type alias; see Item 9
    std::tuple<std::string,   // name
                std::string,  // email
                std::size_t>; // reputation
```

尽管注释指明了 _tuple_ 的每个域都代表了什么，但是当你在独立源文件中遇到下面这样的代码时，可能也不是非常  
有帮助的：
```C++
  UserInfo uInfo;                       // object of tuple type
  …
  
  auto val = std::get<1>(uInfo);        // get value of field 1
```  
做为一个编程者，你有很多需要关注的东西。你真需要记住域 _1_ 对应的是用户的邮箱地址吗？我不这样认为。使  
用 _unscoped enum_ 来将名称和域号关联起来就可以避免去记住域号代表的是什么：
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
可以通过写一个持有 _enumerator_ 并且返回相应的 _std::size_t_ 的值的函数来降低这种冗长，但是会有点棘手。因为  
_std::get_ 是一个模板，你提供的值是模板实参，注意是 _&lt;&gt;_ 而不是 _()_，所以这个将 _enumerator_ 转换为 _std::size_t_ 的  
函数必须在 _编译期间_ 就得要生成它的结果。正如  [_Item 15_](./Chapter%203.md#item-15-只要有可能就使用-constexpr) 所解释的，这意味着这个函数必须是 _constexpr_ 函数。

事实上，这个函数应该是 _constexpr_ 的函数模板，因为它应该能和各种 _enum_ 一起工作。如果我们做了这个泛化的  
话，那么我们也应该泛化它的返回类型。我们应该要返回的是这个 _enum_ 的 _underlying type_ 而不是 _std::size_t_。关  
于 _type traits_ 见 [_Item 9_](./Chapter%203.md#item-9-首选-alias-declarations-而不是-typedefs)。最后，我们声明它为 _noexcept_，见 [_Item 14_](./Chapter%203.md#item-14-如果发出异常的话就声明函数为-noexcept)，因为我们知道它永远不会产生异常。我们构  
建一个函数模板 _toUType_，它持有一个任意的 _enumerator_ 并且按照编译期间常量的形式返回其值：
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

在 _C++14_ 中，使用更简洁的 _std::underlying_type_t_ 来代替 _typename std::underlying_type<E>::type_ 可以使 _toUType_ 更  
简化，见 [_Item 9_](./Chapter%203.md#item-9-首选-alias-declarations-而不是-typedefs)：
```C++
  template<typename E>                                      // C++14
  constexpr std::underlying_type_t<E>
    toUType(E enumerator) noexcept
  {
    return static_cast<std::underlying_type_t<E>>(enumerator);
  }
```

在 _C++14_ 中，更简洁的 _auto_ 返回类型也是有效的，见 [_Item 3_](./Chapter%201.md#item-3-理解-decltype)：
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

这仍然比使用 _unscoped enum_ 的方式写的要多，但是可以避免 _namespace_ 的污染和避免涉及到 _enumerators_ 时的  
不经意间的转换。在很多的场景下，你都会同意：为了避免使用一种源自数字通信的艺术状态仅为 _2400-baud_ 调  
制解调器时期的 _enum_ 的技术的陷阱，多打几个字符是一个合理的代价。

### 需要记住的规则

* _C++98-style_ 的 _enums_ 现在被称为 _unscoped enums_。
* _scoped enums_ 的 _enumerators_ 只在 _enum_ 内可见。使用 _cast_ 才能将 _scoped enums_ 所对应的 _enumerators_ 转  
换为其他的类型。
* _scoped enums_ 和 _unscoped enums_ 都支持指定 _underlying type_。_scoped enum_ 所对应的 _underlying type_ 默认  
是 _int_。_unscoped enum_ 所对应的 _underlying type_ 没有默认类型。
* _scoped enums_ 可以前置声明。_unscoped enums_ 只有当指明了 _underlying type_ 才可以进行前置声明。

## Item 10 首选 _deleted functions_ 而不是 _private undefined functions_

如果你是提供代码给到其他开发者，并且你想要阻止他们调用一个特定的函数的话，那么你一般的方法是不声明这  
个函数。没有函数声明，就没有函数调用。这很简单。但是，有时候 _C++_ 会为你声明一些函数，而如果你想要阻  
止客户调用这些函数的话，就不是那么简单了。

只有对 **_特殊的成员函数_** 才会出现这个情况，这些函数只有当它们被需要的时候，_C++_ 才会自动地生成。
[_Item 17_](./Chapter%203.md#item-17-理解特殊成员函数的生成)  
详细地讨论了这些函数，但是现在，我们仅仅关心 _copy constructor_ 和 _copy assignment operator_。本节主要致力  
于已经被 _C++11_ 中的更好的实践所取代的 _C++98_ 中的常见的实践，在 _C++98_ 中如果你想要禁止成员函数的话，  
那么几乎总是 _copy constructor_ 和 _copy assignment operator_。

_C++98_ 阻止使用这些函数的方法是声明这些函数为 _private_ 且不进行实现。例如：_C++_ 标准库的 _iostreams_ 的基类  
是类模板 _basic_ios_。_istream_ 和 _ostream_ 都是继承自可能是直接继承自 _basic_ios_ 的。拷贝 _istream_ 和 _ostream_ 是不  
想要的，因为真的不知道这些操作应该怎么做。例如：一个 _istream_ 表示的是输入值的 _stream_，其中有的已经读  
取过了，而有的稍后可能会读取。如果要拷贝一个 _istreams_ 的话，那么需要拷贝已经读取过的和稍后会读取的全  
部的值吗？处理这个问题的最简单的方法就是让它们不存在。这正是禁止拷贝 _streams_ 这件事要做的。

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

声明这些函数为 _private_ 可以阻止客户调用到他们。而故意不去定义他们则意味着：如果仍然有代码调用这些函数  
的话，即为：类的成员函数或者友元函数，那么会因为没有函数定义而链接出错。

在 _C++11_ 中，有更好的方法去实现这个：使用 _= delete_ 去标记 _copy constructor_ 和 _copy assignment operator_ 为  
_deleted functions_。下面是 _C++11 中 _basic_ios_：  
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

删除函数和声明函数为 _private_ 的区别似乎更多是时尚问题，而非其他，但其实比你想的要更多。_deleted functions_  
不能以任何方式被使用，成员函数和友元函数中的代码尝试去拷贝 _basic_ios_ 时也会编译失败。这是相对于 _C++98_   
行为的提升，在 _C++98_ 中直到链接时才能诊断出错误。

按照惯例，_deleted functions_ 是应该被声明为 _public_ 而不是 _private_ 的。这是有原因的。因为当客户尝试去使用一  
个 _deleted private function_ 时，一些编译器只会抱怨函数是 _private_ 的，即使函数的可访问性并不真正影响它是否  
可以使用。当修改 _legacy_ 代码使用 _deleted member functions_ 去代替 _private-and-not-defined member functions_  
时，尤其要注意这点，因为声明为 _public_ 一般会得到更好的错误信息。  

_deleted functions_ 的一个重要优势是任何函数都可以被删除，而只有成员函数是可以为 _private_。例如：假定我们有  
一个持有 _integer_ 并且返回它是否为幸运数字的函数：
```C++
  bool isLucky(int number);
```

_C++_ 的 _C_ 的遗产意味着几乎任何可以被模糊地视为数字类型的数据都可以被隐式转换为 _int_ ，但是一些可以编译的调用可能是不合理的：
```C++
  if (isLucky('a')) …         // is 'a' a lucky number?

  if (isLucky(true)) …        // is "true"?
  
  if (isLucky(3.5)) …         // should we truncate to 3
                              // before checking for luckiness?
```

如果幸运数字必须是 _integers_ 的话，那么我们想要阻止这样的调用被编译。

一种可以完成的方法是去创建我们想要过滤的类型所对应的 _deleted_ 的重载函数：
```C++
  bool isLucky(int number);             // original function
  
  bool isLucky(char) = delete;          // reject chars
  
  bool isLucky(bool) = delete;          // reject bools
  
  bool isLucky(double) = delete;        // reject doubles and
                                        // floats
```  
_double_ 重载函数的注释说 _doubles_ 和 _floats_ 都是会被拒绝的，这可能会让你惊讶，但是的惊讶很快会消失，因为你  
想起了：对于 _float_ 转换为 _int_ 还是 _double_ 的选择，_C++_ 首选的会是 _double_。因此使用 _float_ 调用 _isLucky_ 调用会  
是 _double_ 重载函数，而不是 _int_ 重载函数。是的，它会尝试去调用。相关的重载函数被删除的事实将会阻止这个  
调用被编译。

即使 _deleted functions_ 不能被使用，但仍然是你程序的一部分。因此，在重载决议期间它们仍然会被考虑到的。这  
也是在使用了上面的 _deleted functions_ 后不想要的 _isLucky_ 调用会被拒绝的原因：
```C++
  if (isLucky('a')) …         // error! call to deleted function
  
  if (isLucky(true)) …        // error!
  
  if (isLucky(3.5f)) …        // error!
```

