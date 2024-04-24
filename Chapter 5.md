- [Chapter 5 右值引用、移动语义和完美转发](#chapter-5-右值引用移动语义和完美转发)
  - [Item 23 理解 _std::move_ 和 _std::forward_](#item-23-理解-stdmove-和-stdforward)
  - [Item 24 区分 _universal reference_ 和右值引用](#item-24-区分-universal-reference-和右值引用)


# Chapter 5 右值引用、移动语义和完美转发

当你第一次学习移动语义和完美转发时，它们看起来是非常简单的：  

* 移动语义使得编译器能够使用成本小的移动来代替成本大的拷贝。就像 _copy operation_ 可以让你控制拷贝对象  
的含义一样，_move operation_ 也提供了对移动语义的控制。移动语义也支持创建 _move-only_ 类型的对象了，比  
如：_std::unique_ptr_ 、_std::future_ 和 _std::thread_。

* 完美转发使得我们能够写这样的函数模板：它可以持有任意实参，并且可以将这些实参转发到其他目标函数中，  
使得目标函数所接收到的实参和所传递给转发函数的实参完全是相同的。

右值引用将移动语义和完美转发这两个完全不同的特性联系在了一起。它是实现移动语义和完美转发的底层语言机  
制。

你使用这些特性的经验越丰富，你越能意识到你最初的印象仅仅是冰山一角。移动语义、完美转发和右值引用的世  
界要比它所展现的更加微妙。例如：_std::move_ 不移动任何东西，而完美转发是不完美的。_move operation_ 并不是  
一定总是比 _copy operation_ 成本低。就算是，也没有你期待的那么低。在移动是有效的上下文中，_move operation_  
也不总是会被调用到。

不管你对这些特性研究的有多深，但是似乎总是不够深。幸运地是，深度是有限的。本章将带你去了解根本。一旦  
你了解了根本，_C++11_ 的这部分内容就容易理解了。例如：你将会了解到 _std::move_ 和 _std::forward_ 的使用惯例。  
你将会对 _type&&_ 的模糊天性感到舒适。你将会理解为什么 _move operation_ 的行为特点的变化会如此丰富。所有  
的这些都会被理解。在那时，你将会回到起点，因为移动语义、完美转发和右值引用会再一次显得非常简单。但是  
这次，你不会再模糊了。

在本章中的 _Item_ 中，一定要记住形参一定是左值，就算形参的类型是右值引用，形参也是左值。也就是，给定：  
```C++
    void f(Widget&& w);
```  
形参 _w_ 是一个左值，即使 _w_ 的类型 _rvalue-reference-to-Widget_。如果这个让你感到很惊讶的话，那么请你回看下   
[_Introduction.md_](./Introduction.md) 的左值和右值。

## Item 23 理解 _std::move_ 和 _std::forward_ 

从 _std::move_ 和 _std::forward_ 不会做哪些事情的角度来了解它们是有用的 _std::move_ 不移动任何东西。_std::forward_   
也不转发任何东西。在运行时，它们两个都不做任何事情。它们两个不生成执行代码。一个字节都不生成。

_std::move_ 和 _std::forward_ 只是执行转换的函数，它们实际上是函数模板。_std::move_ 会无条件地转换它的实参为右  
值，而 _std::forward_ 只有在满足特定条件时才会转换它的实参为右值。就是这样。这个解释带来了新的问题，但是  
基本上，这就是完整的故事了。

为了使故事更具体点，这儿有 _C++11_ 的 _std::move_ 的简单实现。这个简单实现不是完全符合标准细节的，但是也  
非常接近了。  
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

我已经为你高亮了这个代码的两部分。第一部分是函数名，因为返回类型的规范非常复杂，所以我不想你在复杂中  
迷失方向。第二部分是转换，它包含了本函数的本质。正如你看到的，_std::move_ 持有指向一个对象的引用，是一  
个 _universal reference_，将会在 [_Item 24_](./Chapter%205.md#item-24-区分-universal-reference-和右值引用) 中进行更详细的描述，然后 _std::move_ 会返回这个对象的引用。

这个函数的返回类型的 _&&_ 部分表明了 _std::move_ 返回了一个右值引用，但是正如 [_Item 28_](./Chapter%205.md#item-28-理解引用折叠) 所解释的，如果类型 _T_   
是左值引用的话，那么 _T&&_ 就会是左值引用了。为了避免发生这样的情况，所以 _type trait_ _std::remove_reference_   
被应用到了 _T_ 上，见 [_Item 9_](./Chapter%203.md#item-9-首选-alias-declaration-而不是-typedef)，从而确保 _&&_ 被应用到的类型不是引用类型。这也就确保了 _std::move_ 返回的是右值  
引用，这是非常重要的，因为函数所返回的右值引用肯定是右值。所以 _std::move_ 将它的实参转换为了右值，这正  
是它所做的全部。

顺便说下，_std::move_ 可以在 _C++14_ 中被轻松实现。由于函数返回类型推导，见 [_Item 3_](./Chapter%201.md#item-3-理解-decltype) 和标准库的 _alias template_  
_std::remove_reference_t_，见 [_Item 9_](./Chapter%203.md#item-9-首选-alias-declaration-而不是-typedef)， _std::move_ 可以被写成这样：  
```C++
  template<typename T>                            // C++14; still in
  decltype(auto) move(T&& param)                  // namespace std
  {
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
  }
```  
看起来好多了，不是吗？

因为 _std::move_ 除了会将实参转换为右值外不会做其他的事情，所以有一个建议：_std::move_ 的更好的名字可能是像  
_rvalue_cast_ 这样的名字。尽管有此建议，但是最后定下来的名字还是 _std::move_，所以，重要的是去记住 _std::move_  
会做什么和不会做什么。_std::move_ 会做转换。_std::move_ 不会做移动。

当然，右值支持移动，所以，应用 _std::move_ 到一个对象上是在告诉编译器这个对象是可以被移动的。这也是取名  
为 _std::move_ 的原因：为了方便指定可以被移动的对象。

事实上，右值只在通常情况下才支持移动。假定有一个表示注解的类。这个类所对应的构造函数持有包含有注解的  
_std::string_ 类型的形参，这个类的构造函数会将这个形参拷贝到数据成员中。 根据 [_Item 41_](./Chapter%208.md#item-41-对于移动成本低且总是会被复制的可拷贝形参-考虑-pass-by-value) 的解释，你按  _by-value_   
的形式来声明形参：
```C++
  class Annotation {
  public:
    explicit Annotation(std::string text);        // param to be copied,
    …                                             // so per Item 41,
  };                                              // pass by value
```    
但是 _Annotation_ 的构造函数只需要读取 _text_ 的值。不需要进行修改。为了符合只要有可能就应该去使用 _const_ 的  
优良传统，你修改了你的声明，使得 _text_ 为 _const_：
```C++
  class Annotation {
  public:
    explicit Annotation(const std::string text)
    …
  };
```  
将 _text_ 拷贝至数据成员是有成本的，为了避免这种拷贝的成本，你遵循 [_Item 41_](./Chapter%208.md#item-41-对于移动成本低且总是会被复制的可拷贝形参-考虑-pass-by-value)
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
这个代码可以编译。这个代码可以链接。这个代码可以运行。这个代码也将数据成员 _value_ 的内容设置为了 _text_ 的  
内容。这个代码和你所设想的完美实现间的唯一的区别是：_text_ 不是被移动到 _value_ 中的，而是被拷贝到 _value_ 中  
的。是的，是通过 _std::move_ 将 _text_ 转换为了一个右值，但是 _text_ 是被声明为 _const std::string_ 的，所以 _text_ 在转  
换之前就是一个 _const std::string_ 类型的左值了，相应地，在转换后就是一个 _const std::string_ 类型的右值了，在这  
个过程中， _constness_ 一直保留。

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
在 _Annotation_ 的构造函数的成员初始值列表中，_std::move(text)_ 的结果是一个 _const std::string_ 的右值。这个右值  
是不可以被传递到 _std::string_ 的移动构造函数中的，因为它的移动构造函数持有的是 _non-const std::string_ 的右值引  
用。然而，这个右值是可以被传递到 _std::string_ 的 _copy constructor_ 中的，因为允许 _lvalue-reference-to-const_ 去绑  
定一个 _const_ 右值。因此，_Annotation_ 的成员初始化执行的将会是 _std::string_ 的 _copy constructor_，尽管 _text_ 已经  
被转换为了一个右值。这样的行为对于维护 _const-correctness_ 是必不可少的。将值移出对象之外通常会修改这个对  
象，所以，语言不允许将 _const_ 对象传递给那些像移动构造函数一样的可能会修改它们的函数。 

从这个例子中得出两个注意事项。首先，如果想让一个对象是可以被移动的话，那么就不要将它声明为 _const_。在   
_const_ 对象上的移动请求都会被悄悄地转换为拷贝。其次，实际上，_std::move_ 不仅不移动任何东西，甚至都不能保  
证它转换后的结果是可以被移动的。对于应用 _std::move_ 到一个对象上所产生的结果，你唯一可以确定的是这个结  
果肯定是一个右值。

_std::forward_ 和 _std::move_ 的故事是相似的，_std::move_ 会无条件地将它的实参转换为一个右值，而 _std::forward_ 只有  
在满足特定的条件下才会将它的实参转换为一个右值。所以 _std::forward_ 是条件转换。为了了解 _std::forward_ 何时  
转换何时不转换，回忆一下通常是如何使用 _std::forward_ 的。最常见的情景就是：持有一个通用引用形参的函数模  
板，这个函数模板会将这个形参传递给另一个函数：  
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

在 _logAndProcess_ 中，形参 _param_ 会被传递到函数 _process_ 中。函数 _process_ 对左值和右值进行了重载。当我们使  
用左值来调用 _logAndProcess_ 时，我们自然期待这个左值可以做为左值被转发给函数  _process_，而当我们使用右值  
来调用 _logAndProcess_ 时，我们自然期待调用的是 _process_ 的右值重载版本。

但是，_param_ 像所有的函数形参一样是一个左值。因此，在 _logAndProcess_ 中调用 _process_ 将会调用的是 _process_ 的左值重载版本。为了避免发生这样的情况，我们需要一个这样的机制：只有当用来初始化 _param_ 的实参，也就是被传递到 _logAndProcess_ 的实参，是一个右值时，才将 _param_ 转换为右值。这正是 _std::forward_ 所做的。这也是为什么 _std::forward_ 是条件转换的原因：只有当 _std::forward_ 的实参是使用右值来初始化时，_std::forward_ 才会将的实参转换为右值。

## Item 24 区分 _universal reference_ 和右值引用