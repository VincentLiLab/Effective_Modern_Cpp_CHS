- [Chapter 3 _Moving to Modern C++_](#chapter-3-moving-to-modern-c)
  - [Item 7 创建对象时区分 _()_ 和 _{}_](#item-7-创建对象时区分--和-)
    - [需要记住的规则](#需要记住的规则)
  - [Item 8 首选 _nullptr_ 而不是 _0_ 和 _NULL_](#item-8-首选-nullptr-而不是-0-和-null)
    - [需要记住的规则](#需要记住的规则-1)

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

* _braced initialization_ 是最广泛的可使用的初始化语法，它可以禁止 _narrowing conversions_ 并且对 _C++_ 的  
_most vexing parse_ 所免疫。

* 在构造函数重载决议期间，如果可能，_braced initializer_ 会和 _std::initializer_list_ 形参匹配，即使其他的构造函  
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

使用 _nullptr_ 调用重载函数 _f_ 会调用 _void*_ 重载函数，即为：指针重载函数，因为 _nullptr_ 不可以被做为 _integral_：
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
给模板带来特殊的挑战。结合 _nullptr_ 不会遭遇 _0_ 和 _NULL_ 容易遭遇的重载决议问题的事实，结论就无懈可击了。当你想要引用空指针时，使用 _nullptr_，而不是 _0_ 或 _NULL_。

### 需要记住的规则

* 首选 _nullptr_ 而不是 _0_ 和 _NULL_。

* 避免重载 _integral_ 类型和指针类型
