- [_Chapter 5_ 右值引用、移动语义和完美转发](#chapter-5-右值引用移动语义和完美转发)
  - [_Item 23_ 理解 _std::move_ 和 _std::forward_](#item-23-理解-stdmove-和-stdforward)
    - [需要记住的规则](#需要记住的规则)
  - [_Item 24_ 区分 _universal reference_ 和右值引用](#item-24-区分-universal-reference-和右值引用)
    - [需要记住的规则](#需要记住的规则-1)
  - [_Item 25_ _std::move_ 用于右值引用 _std::forward_ 用于 _univeral reference_](#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference)
  - [_Item 26_ 避免重载 _univeral reference_](#item-26-避免重载-univeral-reference)
  - [_Item 27_ 熟悉重载 _univeral reference_ 的替代方法](#item-27-熟悉重载-univeral-reference-的替代方法)
  - [_Item 28_ 理解引用折叠](#item-28-理解引用折叠)
  - [_Item 30_ 熟悉完美转发失败的场景](#item-30-熟悉完美转发失败的场景)


# _Chapter 5_ 右值引用、移动语义和完美转发

当你第一次学习移动语义和完美转发时，它们看起来是非常简单的：  

* 移动语义使得编译器能够使用成本小的移动来代替成本大的拷贝。像 _copy operation_ 能让你控制拷贝对象的含义一样，_move operation_ 也提供了对移动语义的控制。移动语义也支持创建 _move-only_ 类型的对象了，比如：_std::unique_ptr_ 、_std::future_ 和 _std::thread_。
* 完美转发使得我们能够写这样的函数模板：它可以持有任意实参，并且可以将这些实参转发到其他目标函数中，使得目标函数所接收到的实参和所传递给转发函数的实参完全是相同的。

右值引用将移动语义和完美转发这两个完全不同的特性联系在了一起。它是实现移动语义和完美转发的底层语言机制。

你使用这些特性的经验越丰富，你越能意识到你最初的印象仅仅是冰山一角。移动语义、完美转发和右值引用的世界要比它所展现的更加微妙。例如：_std::move_ 不移动任何东西，而完美转发是不完美的。_move operation_ 并不是一定总是比 _copy operation_ 成本低。就算是，也没有你期待的那么低。在移动是有效的上下文中，_move operation_ 也不总是会被调用到。

不管你对这些特性研究的有多深，但是似乎总是不够深。幸运地是，深度是有限的。本章将会带你去了解根本。一旦你了解了根本，_C++11_ 的这部分内容就容易理解了。例如：你将会了解到 _std::move_ 和 _std::forward_ 的使用惯例。你将会对 _type&&_ 的模糊天性感到舒适。你将会理解为什么 _move operation_ 的行为特点的变化会如此丰富。所有的这些都会被理解。在那时，你将会回到起点，因为移动语义、完美转发和右值引用会再一次显得非常简单。但是这次，你不会再模糊了。

在本章中的 _Item_ 中，一定要记住形参一定是左值，就算形参的类型是右值引用，形参也是左值。也就是，给定：  
```C++
    void f(Widget&& w);
```  
形参 _w_ 是一个左值，即使 _w_ 的类型 _rvalue-reference-to-Widget_。如果这个让你感到很惊讶的话，那么请你回看下 [_Introduction.md_](./Introduction.md) 中关于左值和右值的描述。

## _Item 23_ 理解 _std::move_ 和 _std::forward_ 

从 _std::move_ 和 _std::forward_ 不会做哪些事情的角度来了解它们是有用的 _std::move_ 不移动任何东西。_std::forward_ 也不转发任何东西。在运行时，它们两个都不做任何事情。它们两个不生成执行代码。一个字节都不生成。

_std::move_ 和 _std::forward_ 只是执行转换的函数，它们实际上是函数模板。_std::move_ 会无条件地转换它的实参为右值，而 _std::forward_ 只有在满足特定条件时才会转换它的实参为右值。就是这样。这个解释带来了新的问题，但是基本上，这就是完整的故事了。

为了使故事更具体点，这儿有 _C++11_ 的 _std::move_ 的简单实现。这个简单实现不是完全符合标准细节的，但是也非常接近了。  
```C++
  template<typename T>                            // in namespace std
  typename remove_reference<T>::type&&
  move(T&& param)
  {
    using ReturnType =                            // alias declaration;
      typename remove_reference<T>::type&&;       // see Item 9
    
    return static_cast<ReturnType>(param);
  }
```  

我已经为你高亮了这个代码的两部分。第一部分是函数名，因为返回类型的规范非常复杂，所以我不想你在复杂中迷失方向。第二部分是转换，它包含了本函数的本质。正如你看到的，_std::move_ 持有指向一个对象的引用，是一个 _universal reference_，将会在 [_Item 24_](#item-24-区分-universal-reference-和右值引用) 中进行更详细的描述，然后 _std::move_ 会返回这个对象的引用。

这个函数的返回类型的 _&&_ 部分表明了 _std::move_ 返回的是一个右值引用，但是正如 [_Item 28_](#item-28-理解引用折叠) 所解释的，如果类型 _T_ 是左值引用的话，那么 _T&&_ 就会是左值引用了。为了避免发生这样的情况，所以 _type trait_ _std::remove_reference_ 被应用到了 _T_ 上，见 [_Item 9_](Chapter%203.md#item-9-首选-alias-declaration-而不是-typedef)，从而确保了 _&&_ 被应用到的类型不是引用类型。这也就确保了 _std::move_ 返回的是右值引用，这是非常重要的，因为函数所返回的右值引用肯定是右值。所以 _std::move_ 将它的实参转换为了右值，这正是它所做的全部。

顺便说下，_std::move_ 可以在 _C++14_ 中被轻松实现。由于函数返回类型推导，见 [_Item 3_](Chapter%201.md#item-3-理解-decltype) 和标准库的 _alias template_ _std::remove_reference_t_，见 [_Item 9_](Chapter%203.md#item-9-首选-alias-declaration-而不是-typedef)， _std::move_ 可以被写成这样：  
```C++
  template<typename T>                            // C++14; still in
  decltype(auto) move(T&& param)                  // namespace std
  {
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
  }
```  
看起来好多了，不是吗？

因为 _std::move_ 除了会将实参转换为右值外，不会再做其他的事情，所以有一个建议：_std::move_ 的更好的名字可能是像 _rvalue_cast_ 这样的名字。尽管有此建议，但是最后定下来的名字还是 _std::move_，所以，重要的是去记住 _std::move_ 会做什么和不会做什么。_std::move_ 会做转换。_std::move_ 不会做移动。

当然，右值支持移动，所以，应用 _std::move_ 到一个对象上是在告诉编译器这个对象是可以被移动的。这也是取名为 _std::move_ 的原因：为了方便指定可以被移动的对象。

事实上，右值只在通常情况下才支持移动。假定有一个表示注解的类。这个类所对应的构造函数持有包含有注解的 _std::string_ 类型的形参，这个类的构造函数会将这个形参拷贝到数据成员中。 根据 [_Item 41_](Chapter%208.md#item-41-对于移动成本低且总是会被复制的可拷贝形参考虑-pass-by-value) 的解释，你按  _by-value_ 的形式来声明形参：
```C++
  class Annotation {
  public:
    explicit Annotation(std::string text);        // param to be copied,
    …                                             // so per Item 41,
  };                                              // pass by value
```    
但是 _Annotation_ 的构造函数只需要读取 _text_ 的值。不需要进行修改。为了符合只要有可能就应该去使用 _const_ 的优良传统，你修改了你的声明，使得 _text_ 为 _const_：
```C++
  class Annotation {
  public:
    explicit Annotation(const std::string text)
    …
  };
```  
将 _text_ 拷贝至数据成员是有成本的，为了避免这种拷贝的成本，你遵循 [_Item 41_](Chapter%208.md#item-41-对于移动成本低且总是会被复制的可拷贝形参考虑-pass-by-value)
的建议，将 _std::move_ 应用到了 _text_ 上，从而产生了一个右值：  
```C++
  class Annotation {
  public:
    explicit Annotation(const std::string text)
    : value(std::move(text))                      // "move" text into value; this code
    { … }                                         // doesn't do what it seems to!
    
    …
  
  private:
    std::string value;
  };
```   
这个代码可以编译。这个代码可以链接。这个代码可以运行。这个代码也将数据成员 _value_ 的内容设置为了 _text_ 的内容。这个代码和你所设想的完美实现间的唯一的区别是：_text_ 不是被移动到 _value_ 中的，而是被拷贝到 _value_ 中的。是的，是通过 _std::move_ 将 _text_ 转换为了一个右值，但是 _text_ 是被声明为 _const std::string_ 的，所以 _text_ 在转换之前就是一个 _const std::string_ 类型的左值了，相应地，在转换后就是一个 _const std::string_ 类型的右值了，在这个过程中， _constness_ 一直保留。

考虑一下，当编译器必须要确定调用 _std::string_ 的哪个构造函数时，会面临什么情景。有两种可能性：  
```C++
  class string {                        // std::string is actually a
  public:                               // typedef for std::basic_string<char>
    …
    string(const string& rhs);          // copy ctor
    string(string&& rhs);               // move ctor
    …
  };
```  
在 _Annotation_ 的构造函数的成员初始值列表中，_std::move(text)_ 的结果是一个 _const std::string_ 的右值。这个右值不可以被传递到 _std::string_ 的移动构造函数中，因为 _std::string_ 的移动构造函数持有的是 _non-const std::string_ 的右值引用。然而，这个右值是可以被传递到 _std::string_ 的 _copy constructor_ 中的，因为允许 _lvalue-reference-to-const_ 去绑定一个 _const_ 右值。因此 _Annotation_ 的成员初始化执行的会是 _std::string_ 的 _copy constructor_，尽管 _text_ 已经被转换为了一个右值。这样的行为对于维护 _const-correctness_ 是必不可少的。将值移出对象之外通常会修改这个对象，所以，语言不允许将 _const_ 对象传递给那些像移动构造函数一样的可能会修改它们的函数。 

从这个例子中得出两个注意事项。首先，如果想要让一个对象是可以被移动的话，那么就不要将它声明为 _const_。在 _const_ 对象上的移动请求都会被悄悄地转换为拷贝。其次，实际上，_std::move_ 不仅不移动任何东西，甚至都不能保证它转换后的结果是可以被移动的。对于应用 _std::move_ 到一个对象上所产生的结果，你唯一可以确定的是这个结果肯定是一个右值。

_std::forward_ 和 _std::move_ 的故事是相似的，_std::move_ 会无条件地将它的实参转换为一个右值，而 _std::forward_ 只有在满足特定条件时才会将它的实参转换为一个右值。所以 _std::forward_ 是条件转换。为了了解 _std::forward_ 何时会执行转换何时不执行转换，回忆一下通常是如何使用 _std::forward_ 的。最常见的情景就是：持有一个 _universal references_ 形参的函数模板，这个函数模板会将这个形参传递给另一个函数：  
```C++
  void process(const Widget& lvalArg);            // process lvalues
  void process(Widget&& rvalArg);                 // process rvalues

  template<typename T>                            // template that passes
  void logAndProcess(T&& param)                   // param to process
  {
    auto now = // get current time
      std::chrono::system_clock::now();
    
    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
  }
```  
 考虑 _logAndProcess_ 的两个调用，一个是左值，一个是右值：
 ```C++
  Widget w;
  logAndProcess(w);                     // call with lvalue
  logAndProcess(std::move(w));          // call with rvalue
 ```  

在 _logAndProcess_ 中，形参 _param_ 会被传递给函数 _process_。函数 _process_ 对左值和右值进行了重载。当我们使用左值来调用 _logAndProcess_ 时，我们自然期待这个左值可以做为左值被转发给函数  _process_，当我们使用右值来调用 _logAndProcess_ 时，我们自然期待调用的是 _process_ 的右值重载版本。

但是，_param_ 像所有的函数形参一样是一个左值。因此，在 _logAndProcess_ 中调用 _process_ 将会调用的是 _process_ 的左值重载版本。为了避免发生这样的情况，我们需要一个这样的机制：只有当用来初始化 _param_ 的实参，也就是被传递到 _logAndProcess_ 的实参，是一个右值时，才将 _param_ 转换为右值。这正是 _std::forward_ 所做的。这也是为什么 _std::forward_ 是条件转换的原因：只有当 _std::forward_ 的实参是使用右值来初始化时，_std::forward_ 才会将的实参转换为右值。

你可能会好奇 _std::forward_ 是如何可以知道它的实参是被右值所初始化的呢？例如：如上面的代码中，_std::forward_ 是如何知道 _param_ 是被左值还是右值所初始化的呢？简短的回答是信息被编码到了 _logAndProcess_ 的模板形参 _T_ 中。然后，这个模板形参被传递给了 _std::forward_，_std::forward_ 会解码这个信息。在 [_Item 28_](#item-28-理解引用折叠) 中会详细地描述这个过程到底是如何进行的。

鉴于 _std::move_ 和 _std::forward_ 都是转换，而它们之间的唯一差别是：前者总是会转换，而后者仅仅是满足相应条件时才会转换，你可能会问：我们是否可以不使用 _std::move_ 而只使用 _std::forward_ 呢？只从纯技术角度来说，是可以的：_std::forward_ 可以完成全部工作。_std::move_ 不是必须的。当然，这两个都不是必须的，因为我们可以自己写转换，但是，我希望我们都同意那样做是恶心的。

_std::move_ 的吸引力在于使用方便、可以减少错误发生的可能性和拥有更高的清晰度。假定我们想要记录一个类的移动构造函数被调用了多少次。一个会在移动构造期间被增加的 _static counter_ 就是我们所需要的全部了。 假设这个类中只有一个非静态数据成员 _std::string_，下面是可以实现 _move constructor_ 的传统方法，即为：使用 _std::move_：
```C++
  class Widget {
  public:
    Widget(Widget&& rhs)
    : s(std::move(rhs.s))
    { ++moveCtorCalls; }
    
    …

  private:
    static std::size_t moveCtorCalls;
    std::string s;
  };
```  
为了使用 _std::forward_ 来实现相同的行为，代码是下面这样的：
```C++
  class Widget {
  public:
    Widget(Widget&& rhs)                          // unconventional,
    : s(std::forward<std::string>(rhs.s))         // undesirable
    { ++moveCtorCalls; }                          // implementation
    
    …
    
    };
```

注意，首先，_std::move_ 只需要函数实参 _rhs.s_，而 _std::forward_ 需要函数实参 _rhs.s_ 和模板类型实参 _std::string_ 。其次，所传递给 _std::forward_ 的类型应该是 _non-reference_，因为这是用来表明所传递的实参是一个右值的编码惯例，见 [_Item 28_](#item-28-理解引用折叠)。这两者结合起来就意味着： _std::move_ 会比 _std::forward_ 要少输入一些代码，并且 _std::move_ 省去了传递那个编码了所传递的实参是一个右值的类型实参的麻烦。 _std::move_ 消除了可能会传递一个错误类型的可能性，比如：_std::string&_ 会导致数据成员 _s_ 是拷贝构造所得，而不是移动构造所得。

更重要的是，使用 _std::move_ 所要表达的是会无条件地转换为右值，而使用 _std::forward_ 所要表达的则是只会将那些绑定了右值的引用转换为右值。这是两个完全不同的动作。_std::move_ 通常只是设置了移动，而 _std::forward_ 则是以保留某个对象的原来的 _lvalueness_ 或者 _rvalueness_ 的方式来将这个对象转发到其他函数。因为这两个动作是不同的，所以使用两个不同的函数来进行区分是很好的。

### 需要记住的规则

* _std::move_ 无条件地执行右值转换。它不移动任何东西。
* _std::forward_ 只在它的实参是绑定了一个右值时，才会将它的实参转换为右值。
* _std::move_ 和 _std::forward_ 在运行时都不做任何事。


## _Item 24_ 区分 _universal reference_ 和右值引用

据说：真理使人自由，但是在适当的情况下，一个恰到好处的谎言也同样可以让你感到解脱。本 _Item_ 就是这样的谎言。因为我们面对的是软件，所以让我们避开 _谎言_ 这个词，而去说本 _Item_ 包含了一个 _抽象_。
        
为了声明类型 _T_ 的右值引用，你写了 _T&&_。因此这样的假设似乎也是合理的：如果你在代码中看到了 _T&&_ 的话，那么你正在看到的是右值引用。但是，没有那么简单：
```C++
  void f(Widget&& param);               // rvalue reference

  Widget&& var1 = Widget();             // rvalue reference

  auto&& var2 = var1;                   // not rvalue reference

  template<typename T>
  void f(std::vector<T>&& param);       // rvalue reference
  
  template<typename T>
  void f(T&& param);                    // not rvalue reference
```

事实上， _T&&_ 有两个不同的含义。当然，第一种含义是表示它是右值引用。这样的引用表现地完全如你所愿：只可以绑定右值。右值引用的作用就是识别那些可以移动的对象。

_T&&_ 的第二种含义是表示它可以是右值引用或左值引用。这样的引用看起来像是代码中的右值引用，即为：_T&&_ ，但是它又表现地好像是左值引用一样，即为：_T&_ 。这样的双重特性允许 _T&&_ 可以像右值引用那样绑定右值，也可以像左值引用那样绑定左值。此外，_T&&_ 可以绑定 _const_ 对象或 _non-const_ 对象，也可以绑定 _volatile_ 对象或 _non-volatile_ 对象，甚至可以绑定 _const_ _volatile_ 的对象。几乎可以绑定所有东西。这样空前灵活的引用值得有它们所拥有的的名字。我称他们为 _universal references_。

 _universal references_ 在出现在两种场景下。第一种最常见，是函数模板的形参，例如上面代码中的例子：  
```C++
  template<typename T>
  void f(T&& param);          // param is a universal reference
```  
第二种场景是是 _auto_ 声明，包括来自上面简单代码的例子：
```C++
  auto&& var2 = var1;         // var2 is a universal reference
```  
这两种场景的共同之处是都存在类型推导。在函数模板 _f_ 中， _param_ 的类型是被推导出的，而在 _var2_ 的声明中， _var2_ 的类型也是被推导出的。比较此处的例子和下面的例子，会发现下面的例子中是没有类型推导的。如果你在没有类型推导的情况下看到了 _T&&_ 的话，那么你看到的是右值引用：  
```C++
  void f(Widget&& param);               // no type deduction;
                                        // param is an rvalue reference
  
  Widget&& var1 = Widget();             // no type deduction;
                                        // var1 is an rvalue reference
```  
因为 _universal references_ 是引用，所以它们必须要进行初始化。 _universal references_ 所对应的 _initializer_ 决定了这个 _universal references_ 表示的是右值引用还是左值引用。如果 _initializer_ 是个右值的话，那么 _universal references_ 对应的是右值引用。如果 _initializer_ 是个左值的话，那么 _universal references_ 对应的是左值引用。对于函数形参是 _universal references_ 的情况，_initializer_ 是在调用处被提供的： 
```C++
  template<typename T>
  void f(T&& param);                    // param is a universal reference
  
  Widget w;
  f(w);                                 // lvalue passed to f; param's type is
                                        // Widget& (i.e., an lvalue reference)
  
  f(std::move(w));                      // rvalue passed to f; param's type is
                                        // Widget&& (i.e., an rvalue reference)
```

若想成为 _universal references_，类型推导是必须的，但这还不够。引用声明的 _形式_ 也要必须正确，而且这种形式是非常严格的。必须严格是 _T&&_ 才可以。再看一次我们之前看的简单代码的例子：  
```C++
  template<typename T>
  void f(std::vector<T>&& param);       // param is an rvalue reference
```
当 _f_ 被执行时，类型 _T_ 将会被推导。除非调用方显式指明了它，我们不关心这种边缘场景。但是 _param_ 的类型声明的形式不是 _T&&_，而是 _std::vector&lt;T&gt;&&_。这就排除了 _param_ 是 _universal references_ 的可能性。因此，_param_ 是一个右值引用。如果你尝试传递一个左值给 _f_ 的话，编译器将会很乐意去为你确认一些事情：  
```C++
  std::vector<int> v;
  f(v);                       // error! can't bind lvalue to
                              // rvalue reference
```  
即使存在简单的 _const_ _qualifier_ 也足够取消引用成为 _universal references_ 的资格：  
```C++
  template<typename T>
  void f(const T&& param);    // param is an rvalue reference
```  
如果你在模板中看到一个函数的形参的类型是 _T&&_ 的话，那么你可能会认为你可以假设 _T&&_ 是 _universal references_ 了。你不可以。因为在模板中并不能保证存在有类型推导。考虑 _std::vector_ 的 _push_back_ 成员函数：  
```C++
  template<class T, class Allocator = allocator<T>>         // from C++
  class vector {                                            // Standards
  public:
    void push_back(T&& x);
    …
  };
```  
_push_back_ 的形参的类型肯定是 _universal references_ 的正确形式，但是在这种场景下没有类型推导。这是因为 _push_back_ 是做为 _vector_ 的一部分的，如果特定的 _vector_ 实例化不存在，那么 _push_back_ 也不可能存在，这个实例化的类型完全决定了 _push_back_ 的声明。也就是说：  
```C++
  std::vector<Widget> v;
```  
导致了 _std::vector_ 模板将会被实例化为下面这样：  
```c++
  class vector<Widget, allocator<Widget>> {
  public:
    void push_back(Widget&& x);          // rvalue reference
    …
  };
```  

现在，你可以清晰地看到 _push_back_ 没有应用类型推导。 _vector&lt;T&gt;_ 的 _push_back_ 总是声明了 _rvalue-reference-to-T_ 类型的形参，共有两个函数，是函数重载。

相比之下，概念上类似地 _std::vector_ 的 _emplace_back_ 成员函数却应用了类型推导：  
```C++
  template<class T, class Allocator = allocator<T>>         // still from
  class vector {                                            // C++
  public:                                                   // Standards
    template <class... Args>
    void emplace_back(Args&&... args);
    …
  };
```  
此处，类型形参 _Args_ 和 _vector_ 的类型形参 _T_ 是无关的，每当 _emplace_back_ 被调用时，_Args_ 都是会被推导的。好吧，_Args_ 是类型形参包，而不是类型形参，但是此处只是为了讨论，我们可以认为它就是类型形参。

_emplace_back_ 的类型形参被命名为了 _Args_，但它却仍然是 _universal references_，这个事实强化了我之前的论点：_universal references_ 的形式必须是 _T&&_ 。但没有必要非得使用 _T_ 。例如：下面的函数持有 _universal references_，因为形式 _type&&_ 是正确的，而且 _param_ 的类型是会被推导的，再次强调，排除那种调用方会显式指明类型的极端情况：  
```C++
  template<typename MyTemplateType>               // param is a
  void someFunc(MyTemplateType&& param);          // universal reference
```

我之前就提到过：_auto_ 声明的变量也可以是 _universal references_。更精确的说，使用类型 _auto&&_ 所声明的变量是 _universal references_，因为发生了类型推导，并且有正确的形式 _T&&_。_auto_ 所对应的 _universal references_ 没有函数模板的形参所对应的 _universal references_ 那么常见，但是在 _C++11_ 中偶尔还是存在的。但是在 _C++14_ 中就非常多了，因为 _lambda expression_ 可以声明 _auto&&_ 形参了。例如：如果你想写一个 _C++14_ 的 _lambda_ 去记录调用任意函数所花费的时间的话，那么你可以这样写：
```C++
  auto timeFuncInvocation =
  [](auto&& func, auto&&... params) // C++14
  {
    start timer;
    std::forward<decltype(func)>(func)(           // invoke func
    std::forward<decltype(params)>(params)...     // on params
    );
    stop timer and record elapsed time;
  };
```

如果你对于 _lambda_ 中的 _std::forward&lt;decltype(blah blah blah)&gt;_ 的反应是 _What the fuck?!_ 的话，
那么很可能是你还没有读 [_Item 33_](Chapter%206.md#item-33-在-auto-形参上使用-decltype-来进行完美转发)。别担心，本 _Item_ 的重点是 _lambda_ 所声明的 _auto&&_ 形参。_func_ 是可以绑定可调用对象、左值或右值的 _universal references_。_args_ 是可以绑定任意数量的任意类型的对象的 _universal references_ 形参包。由于 _auto_ 所对应的 _universal references_ 的存在， _timeFuncInvocation_ 才可以计时几乎任何函数的执行时间。对于 _几乎任何_ 和 _任何_ 之间的差别，请看 [_Item 30_](#item-30-熟悉完美转发失败的场景)。

牢牢记住本 _Item_，也就是 _universal references_ 的基础，是一个 _谎言_ ，不，是一个 _抽象_。底层真相被称为引用折叠，[_Item 28_](#item-28-理解引用折叠) 会专门来探讨这个主题。但是，真相不会降低这种抽象的实用性。区分右值引用和 _universal references_ 可以帮助更加精确地阅读代码，我看到的 _T&&_ 是只能绑定右值呢？还是可以绑定所有？它还可以避免在你和你同事沟通时产生歧义，我在这里使用的是 _universal references_，而不是右值引用。它能帮助你理解 [_Item 25_](#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference) 和 [_Item 26_](#item-26-避免重载-univeral-reference)，它们依赖于这个区分。所以拥抱 _抽象_ 吧。享受 _抽象_ 吧。严格来说牛顿运动定律是不正确的，而爱因斯坦广义相对论则是正确的，但是前者和后者一样有用，并且更容易应用，同样地，掌握 _universal references_ 的概念比掌握引用折叠的细节要更好。
        
### 需要记住的规则

* 如果函数模板的形参有类型 _T&&_ 且 _T_ 是会被推导的话，或者如果一个对象是使用 _auto&&_ 所声明的话，那么这个形参或者这个对象是 _universal references_。
* 如果类型声明的形式并不是严格为 _type&&_ 的话，或者如果并没有发生类型推导的话，那么 _type&&_ 表示的是右值引用。
* 如果 _universal references_ 是被右值所初始化的话，那么 _universal references_ 对应的是右值引用。如果 _universal references_ 是被左值所初始化的话，那么 _universal references_ 对应的是左值引用。

## _Item 25_ _std::move_ 用于右值引用 _std::forward_ 用于 _univeral reference_

## _Item 26_ 避免重载 _univeral reference_

## _Item 27_ 熟悉重载 _univeral reference_ 的替代方法

## _Item 28_ 理解引用折叠

## _Item 30_ 熟悉完美转发失败的场景