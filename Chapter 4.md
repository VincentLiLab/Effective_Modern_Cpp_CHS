- [Chapter 4 智能指针](#chapter-4-智能指针)
  - [Item 18 对于 _exclusive-ownership_ 的资源管理使用 _std::unique\_ptr_](#item-18-对于-exclusive-ownership-的资源管理使用-stdunique_ptr)
    - [需要记住的规则](#需要记住的规则)

# Chapter 4 智能指针

诗人和作曲家都对爱情情有独钟，有时候也对计数情有独钟。偶尔两者都有。受到 _Elizabeth Barrett Browning_ 的  
**_How do I love thee? Let me count the ways._** 和 _Paul Simon_ 的 **_There must be 50 ways to leave your lover._** 在爱与  
计数方面不同观点的启发，我们可以试着去列举原始指针很难被喜欢的原因：  
* 原始指针的声明不会表明这个原始指针是指向单个对象还是一组对象。
* 原始指针的声明没有透露：当你使用完这个原始指针的时候，你是否应该销毁这个原始指针所指向的内容，即  
为：这个原始指针是否 **_拥有_** 它所指向的内容。
* 如果你确定应该销毁原始指针所指向的内容的话，那么是没有方法告诉你应该如何进行的。应该使用 _delete_  
还是应该使用不同的析构机制呢？比如：应该将原始指针传入到一个专用的析构函数中。
* 如果你发现了 _delete_ 是合适的方法的话，那么也可能并不知道应该使用 _delete_ 还是应该使用 _delete[]_ 呢？如  
果你使用了错误的格式的话，那么结果是未定义的。
* 假设你确定：原始指针拥有它所指向的内容，而且你知道了应该如何销毁它，但还是很难去沿着你代码的每一  
条路径和这些路径所导致的异常，去确定只执行了一次析构。错过一条路径会导致资源泄露，而多执行一次析  
构则会导致 _undefined behavior_。
* 通常没有方法可以表明原始指针是否悬空了，即为：原始指针所指向的内存不再保存它应该指向的对象了。当  
对象被销毁，而原始指针仍然指向这个对象时，悬空指针就产生了。

原始指针是强大的工具，这是可以肯定的，但是数十年的经验已经证明：只要在专注和纪律中有一点小疏忽，这些  
工具可以反过来攻击使用它的人。

智能指针是解决这些问题的一种方法。智能指针是原始指针的包装，它表现的非常像它所包装的那个原始指针，而  
且它还可以避免很多原始指针的陷阱。因此，应该首选智能指针而不是原始指针。智能指针几乎可以做原始指针可  
以做的所有事，并且出错的机会要少的多。

_C++11_ 有 _4_ 种智能指针：_std::auto_ptr_、_std::unique_ptr_ 、_std::shared_ptr_ 和 _std::weak_ptr_。它们是被设计来帮助管  
理动态分配的对象的生命周期的，即为：通过确认这些对象是在合适的时间以合适的方式被销毁来避免资源泄露,  
包括在异常事件中。 

_std::auto_ptr_ 是 _C++98_ 遗留下来的一个被废弃的类型，它是对后来成为标准化 _C++11_ 的 _std::unique_ptr_ 的尝试。  
想要正确的完成这个工作是需要移动语义的，但是 _C++98_ 并没有移动语义。做为一种解决方法，_std::auto_ptr_ 利  
用了它的 _copy operation_ 来完成移动操作。这会产生令人惊讶的代码：拷贝 _std::auto_ptrs_ 会将它设置为空，这也  
会导致令人沮丧的使用限制，比如：不能在 _container_ 中存储 _std::auto_ptr_ 类型的对象。

_std::unique_ptr_ 可以做 _std::auto_ptr_ 可以做的所有事情，而且更多。_std::unique_ptr_ 可以高效率地完成工作，而且是  
在不改变复制对象含义的情况下完成的工作。无论哪个方面，_std::unique_ptr_ 都比 _std::auto_ptr_ 要好。 唯一合理的  
使用 _std::auto_ptr_ 的场景是需要使用 _C++98_ 编译器来编译代码。除非有这种限制，否则应该使用 _std::unique_ptr_   
来代替 _std::auto_ptr_，并且永不再用 _std::auto_ptr_。

智能指针的 _API_ 多种多样。唯一对于所有的智能指针都共有的功能是默认构造。因为这些 _API_ 所对应的全面的参  
考文档非常地多，所以我将专注于讨论那些通常在 _API_ 的概述中所没有的信息，比如，值得使用的场景和运行成  
本分析等等。掌握这些信息会让你从只会使用智能指针变为高效使用智能指针。

## Item 18 对于 _exclusive-ownership_ 的资源管理使用 _std::unique_ptr_

当你接触到智能指针时，_std::unique_ptr_ 通常应该是首选。默认情况下，有理由去假设： _std::unique_ptr_ 和原始指  
针是一样大的，并且对于大多数操作来说，包括解引用，它们执行的都是相同的指令。这意味着即使在内存和周期  
都很紧张的情况下也可以使用 _std::unique_ptr_。如果原始指针对于你来说是足够小和快的话，那么 _std::unique_ptr_  
几乎也可以确定是这样的。

_std::unique_ptr_ 体现了 _exclusive ownership_ 的语义。一个非空的 _std::unique_ptr_ 总是拥有着它所指向的内容。移动   
_std::unique_ptr_ 会将 _ownership_ 从源指针转移到目标指针中，源指针会被设置为空。而拷贝 _std::unique_ptr_ 是不被  
允许的，因为如果你可以拷贝的话，那么你最终将有两个指向相同资源的 _std::unique_ptr_，而且每一个都认为自己  
拥有着这个资源且应该自己去销毁这个资源。因此，_std::unique_ptr_ 是 _move-only_ 类型。一旦发生析构时，非空的  
_std::unique_ptr_ 应该去销毁它的资源。默认情况下，资源销毁是通过在 _std::unique_ptr_ 内应用 _delete_ 到原始指针来  
完成的。

一个对于 _std::unique_ptr_ 的常见用法是做为层级结构中的对象的工厂函数的返回类型。假设我们有一个关于投资类  
型的层级结构，比如：股票、债券、房地产等，使用的 _base class
_ 是 _Investment_。  
```C++
  class Investment { … };
  
  class Stock:
    public Investment { … };
  
  class Bond:
    public Investment { … };
  
  class RealEstate:
    public Investment { … };
```

![Image2](./image/image2.jpg)  

这个层级结构的工厂函数一般会在堆上分配一个对象，并返回一个指向这个对象的指针，当不再需要这个对象时，  
调用方负责去删除这个对象。这是 _std::unique_ptr_ 的完美匹配，因为调用方承担了工厂函数所返回的资源的责任，  
即为：_exclusive ownership_，当 _std::unique_ptr_ 被销毁时，_std::unique_ptr_ 会自动删除它所指向的对象。_Investment_  
的层级结构的工厂函数被声明为下面这样：
```C++
  template<typename... Ts>              // return std::unique_ptr
  std::unique_ptr<Investment>           // to an object created
  makeInvestment(Ts&&... params);       // from the given args
```  

调用方可以像下面这样在单一作用域内使用返回的 _std::unique_ptr_：  
```C++
  {
    …

    auto pInvestment =                  // pInvestment is of type
      makeInvestment( arguments );      // std::unique_ptr<Investment>
    
    …
}
```  
但是也可以在 _ownership-migration_ 的情景下使用它，比如：将工厂函数所返回的 _std::unique_ptr_ 移动到 _container_   
中，随后又将 _container_ 中的这个 _std::unique_ptr_ 移动到某个对象的数据成员中，最后这个对象被销毁。当这种情  
况发生时，这个对象的 _std::unique_ptr_ 数据成员也是会被销毁，_std::unique_ptr_ 的析构函数会导致工厂函数返回的  
资源被销毁。如果 _ownership_ 链条由于异常或者其他非典型的控制流程而中断了，比如：提前结束函数或者中断循  
环，拥有所管理的资源的 _std::unique_ptr_ 最终还是会调用到它的析构函数，这会将正在管理的资源销毁掉。

默认情况下，析构是通过 _delete_ 来进行的，但是也可以在构造期间使用 _custom deleter_ 来对 _std::unique_ptr_ 对象进  
行配置：当到了要销毁资源的时候，这些任意的函数或函数对象，包括 _lambda expression_ 所产生的 _closure_，会被  
执行。如果 _makeInvestment_ 所创建的对象不应该直接被直接被删除，而是应该先写一个 _log_ 的话，那么可以向下  
面这样来实现  _makeInvestment_ ，代码后面跟着解释，所以当你看到一些事情的意图不太明显时，不要担心：  
```C++
  auto delInvmt = [](Investment* pInvestment)               // custom
  {                                                         // deleter
    makeLogEntry(pInvestment);                              // (a lambda
    delete pInvestment;                                     // expression)
  };

  template<typename... Ts>                                  // revised
  std::unique_ptr<Investment, decltype(delInvmt)>           // return type
  makeInvestment(Ts&&... params)
  {
    std::unique_ptr<Investment, decltype(delInvmt)>         // ptr to be
      pInv(nullptr, delInvmt);                              // returned

    if ( /* a Stock object should be created */ )
    {
      pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* a Bond object should be created */ )
    {
      pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* a RealEstate object should be created */ )
    {
      pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
  }
```   
 稍后，我会解释这些是如何工作的，但是我们先来考虑一下：如果你是调用方的话，那么事情会是怎样的。假设你  
 将 _makeInvestment_ 的结果存储到了 _auto_ 变量中，你是可以忽略你所使用的资源是需要在删除期间被特殊对待的  
 事实的。事实上，你该高兴，因为使用 _std::unique_ptr_ 意味着：不需要关心资源应该何时被销毁，更不用去确保在  
 程序中的每条路径上只发生了一次析构。_std::unique_ptr_ 会自动处理这些事情。站在客户的角度，_makeInvestment_   
 的接口是完美的。

一旦你理解了以下内容，这个实现也是非常好的：  

* _delInvmt_ 是 _makeInvestment_ 所返回的对象所对应的 _custom deleter_。所有的 _custom deleter_ 都接收一个指向  
了会被销毁的对象的原始指针，会去做一些要销毁这个对象所必须要做的动作。在这个场景中，这个所对应的  
动作是调  _makeLogEntry_，然后应用 _delete_。使用 _lambda expression_ 去创建 _delInvmt_ 是方便的，而且正如我  
们很快会看到的，使用 _lambda expression_ 比使用常规的函数是要高效的。
* 当一个 _custom deleter_ 被使用时，它的类型是被用来做为 _std::unique_ptr_ 的第二个类型实参的。在这个场景中，  
是 _delInvmt_ 的类型，_makeInvestment_ 所返回的类型必须是 _std::unique_ptr&lt;Investment, decltype(delInvmt)&gt;_。  
更多关于 _decltype_ 的信息见  [_Item 3_](./Chapter%201.md#item-3-理解-decltype)。
* _makeInvestment_ 的基本策略是创建一个空的 _std::unique_ptr_ 并使它指向一个合适类型的对象，然后再返回这个  
_std::unique_ptr_。为了将 _custom deleter_ _delInvmt_ 与 _pInv_ 关联起来，我们将 _delInvmt_ 来做为 _std::unique_ptr_ 的  
构造函数的第二个实参。
* 尝试将从 _new_ 得来的原始指针赋值给 _std::unique_ptr_ 是不能通过编译的，因为这将被视为从原始指针到智能  
指针的隐式转换。这样的隐式转换是有问题的，所以 _C++11_ 的智能指针禁止了这种转换。这也是为什么需要  
使用 _reset_ 来让 _pInv_ 获取到通过 _new_ 所创建的对象的 _ownership_。
* 在每次使用 _new_ 时，我们都使用 _std::forward_ 来完美转发所传递给 _makeInvestment_ 的实参，见 [_Item 25_](./Chapter%205.md#item-25-std::move-用于右值引用-std::forward-用于-univeral-reference)。这  
使得所创建的对象的构造函数可以获取到调用方所提供的全部信息。
* _custom deleter_ 持有一个 _Investment*_ 类型的形参。不管在 _makeInvestment_ 中所创建的对象的类型是什么，即  
为：_Stock_、_Bond_ 或 _RealEstate_，最终在 _lambda expression_ 中都是做为一个 _Investment*_ 对象来删除的。这意  
味着我们可以通过 _base class_ 指针来删除 _derived class_ 对象。为了可以这样，_base class_ _Investment_ 必须有一个 _virtual_ 析构函数：
```C++
  class Investment {
    public:
    …                         // essential
    virtual ~Investment();    // design
    …                         // component!
  };
```

在 _C++14_ 中，函数返回值类型推导的存在，见 [_Item 3_](./Chapter%201.md#item-3-理解-decltype)，意味着 _makeInvestment_ 可以以更简洁和更好封装的形式  
来实现：  
```C++
  template<typename... Ts>
  auto makeInvestment(Ts&&... params)             // C++14
  {
    auto delInvmt = [](Investment* pInvestment)             // this is now
    {                                                       // inside
      makeLogEntry(pInvestment);                            // make-
      delete pInvestment;                                   // Investment
    };

    std::unique_ptr<Investment, decltype(delInvmt)>         // as
      pInv(nullptr, delInvmt);                              // before
    
    if ( … )                                                // as before
    {
      pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( … )                                           // as before
    {
      pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( … )                                           // as before
    {
      pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    
    return pInv;                                            // as before
  }
```  

我之前就提到过，当使用 _default deleter_ 时，即为 _delete_，你有理由假设 _std::unique_ptr_ 和原始指针是一样大的。  
当 _custom deleter_ 参与进来时，可能就不是这样了。当 _deleter_ 是函数指针时，通常会导致 _std::unique_ptr_ 的大小  
由一个字增长为两个字。而对于 _deleter_ 是函数对象的情况，大小的改变依赖于有多少种状态存储在函数对象中。  
无状态的函数对象，比如没有捕获的 _lambda expression_，不会改变大小，这意味着：当 _custom deleter_ 的实现可  
以由函数或无捕获的 _lambda expression_ 来实现时，_lambda expression_ 是更好的选择：
```C++
  auto delInvmt1 = [](Investment* pInvestment)    // custom
  {                                               // deleter
    makeLogEntry(pInvestment);                    // as
    delete pInvestment;                           // stateless
  };                                              // lambda
  
  template<typename... Ts>                        // return type
  std::unique_ptr<Investment, decltype(delInvmt1)>// has size of
  makeInvestment(Ts&&... args);                   // Investment*
  
  void delInvmt2(Investment* pInvestment)         // custom
  {                                               // deleter
    makeLogEntry(pInvestment);                    // as function
    delete pInvestment;
  }
  
  template<typename... Ts>                        // return type has
  std::unique_ptr<Investment,                     // size of Investment*
                  void (*)(Investment*)>          // plus at least size
  makeInvestment(Ts&&... params);                 // of function pointer!
```  

具有大量状态的函数对象 _deleter_ 可以产生显著增大的 _std::unique_ptr_ 对象。如果你发现了 _custom deleter_ 使你的  
_std::unique_ptr_ 变的大到不可接受了的话，那么你大概需要去改变你的设计了。

工厂函数并不是 _std::unique_ptrs_ 唯一常见的使用场景。工厂函数更常见的是做为实现 _Pimpl Idiom_ 的机制。工厂  
函数实现 _Pimpl Idiom_ 的代码并不复杂，只是在一些场景中，没有那么直观，所以，我推荐你去看 [_Item 22_](./Chapter%204.md#item-22-首选-std::make_unique-和-std::make_shared-而不是直接使用-new)，其中  
专门描述了这个话题。

_std::unique_ptr_ 有两种形式：_std::unique_ptr&lt;T&gt;_ 对应于单一对象，而 _std::unique_ptr&lt;T[]&gt;_ 对应于数组对象。总  
之，对于 _std::unique_ptr_ 所指向的实体永远不会有任何歧义。_std::unique_ptr_ 的 _API_ 是被设计成与你正在使用的形  
式相匹配的。例如：对于单一对象的形式是没有 _operator[]_ 的，而对于数组形式是缺少 _operator*_ 和 _operator->_   
的。

对于数组形式的 _std::unique_ptr_，你只需要知道它是存在的就可以了。因为 _std::array_、_std::vector_ 和 _std::string_ 相  
比于原始数组几乎总是更好的数据结构选择。只有一种我可以想到的情况：当你正在使用一个会返回指向堆数组的  
原始指针的 _C-like_ 的 _API_，并且你拥有这个堆数组的所有权时，再去使用 _std::unique_ptr&lt;T[]&gt;_ 才能说的通。

_std::unique_ptr_ 是 _C++11_ 表达 _exclusive ownership_ 的方法，但是它还有一个最吸引人的特性：_std::unique_ptr_ 可以  
被简单高效地转换为 _std::shared_ptr_：
```C++
  std::shared_ptr<Investment> sp =      // converts std::unique_ptr
  makeInvestment( arguments );          // to std::shared_ptr
```  
这个特性使得 _std::unique_ptr_ 非常适合做为工厂函数的返回类型。对于工厂函数所返回的对象来说，它们并不知道  
调用方会使用 _exclusive ownership_ 语义还是 _shared ownership_ 语义，即为：_std::shared_ptr_ 是否会更合适。通过返  
回 _std::unique_ptr_，工厂函数只是提供给了调用方一个高效的智能指针，但并不会阻碍调用方使用更灵活的其他智   
能指针来代替它。更多关于 _std::shared_ptr_ 的信息见  [_Item 19_](./Chapter%204.md#item-19-对于-shared-ownership-的资源管理使用-std::shared_ptr)。

### 需要记住的规则

* _std::unique_ptr_ 是小的、快的和 _move-only_ 的智能指针，用于使用 _exclusive-ownership_ 语义来管理资源。
* 资源析构默认是通过 _delete_ 来进行的，但是也可以指定 _custom deleters_。带状态的 _stateful deleter_ 和函数指  
针做为 _deleter_ 时，会增加 _std::unique_ptr_ 对象的大小。
* 转换 _std::unique_ptr_ 为 _std::shared_ptr_ 是容易的。  



