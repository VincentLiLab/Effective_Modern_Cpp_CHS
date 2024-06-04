- [_Chapter 8_ _tweak_](#chapter-8-tweak)
  - [_Item 41_ 对于移动是成本小的且总是会被拷贝的可拷贝的形参考虑 _pass-by-value_](#item-41-对于移动是成本小的且总是会被拷贝的可拷贝的形参考虑-pass-by-value)
    - [需要记住的规则](#需要记住的规则)
  - [_Item 42_ 考虑使用 _emplacement_ 来代替 _insertion_](#item-42-考虑使用-emplacement-来代替-insertion)
    - [需要记住的规则](#需要记住的规则-1)

# _Chapter 8_ _tweak_

对于 _C++_ 中的每一个通用的技术或特性，存在使用它们是合理的环境，也存在使用它们是不合理的环境。描述通用的技术或特性什么时候应该被使用是情理之中的，这通常是非常简单的，但是本 _chapter_ 覆盖了两个例外。一个通用的技术是 _pass-by-value_，另一个通用的特性是 _emplacement_。什么时候应该去使用它们是被很多因素影响的，我可以提供的最好建议是考虑它们的用途。然而，它们是高效 _modern C++_ 编程中的重要成员，后续的 _Item_ 提供了你需要的可以被用来确定使用它们是否对于你的软件是合适的的信息。

## _Item 41_ 对于移动是成本小的且总是会被拷贝的可拷贝的形参考虑 _pass-by-value_

一些函数的形参旨在被拷贝。例如：一个成员函数 _addName_ 可以拷贝它的形参到私有的 _container_ 中。为了效率，这样的函数应该拷贝左值实参，移动右值形参：  
```C++
  class Widget {
  public:
    void addName(const std::string& newName)      // take lvalue;
    { names.push_back(newName); }                 // copy it
    
    void addName(std::string&& newName)           // take rvalue;
    { names.push_back(std::move(newName)); }      // move it; see
      …                                           // Item 25 for use
                                                  // of std::move
  private:
    std::vector<std::string> names;
  };
```  
这可以工作，但是需要写两个本质上是做相同事情的函数。这有点让人恼火：两个函数都需要声明，两个函数都需要实现，两个函数都需要记录，两个函数都需要维护。额。

此外，在这个代码中存在两个函数，如果你关心你的程序的 _footprint_ 的话，那么你需要关心这个事情。在这个场景中，这两个函数可能都是内联的，这可能会消除这两个函数的实体所相关的膨胀问题，但是如果这些函数不是在每个地方都是内联的话，那么你真的就会在你的目标代码中得到这两个函数了。

一个替代的方法是去写一个持有 _universal reference_ 的函数模板 _addName_，见 [_Item 24_](Chapter%205.md#item-24-区分-universal-reference-和右值引用)：  
```C++
  class Widget {
  public:
    template<typename T>                          // take lvalues
    void addName(T&& newName)                     // and rvalues;
    {                                             // copy lvalues,
      names.push_back(std::forward<T>(newName));  // move rvalues;
    }                                             // see Item 25
                                                  // for use of
    …                                             // std::forward

  };
```  
这会减少你必须要处理的源代码，但是使用 _universal reference_ 会导致其他的复杂。做为一个模板，_addName_ 的实现通常必须在头文件中。_addNmae_ 可能会在目标代码中产生一系列函数，它不仅对左值和右值进行不同实例化，还对 _std::string_ 和可以转换为 _std::string_ 的类型进行不同的实例化，见 [_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference)。同时，还存在不可以被传递给 _universal reference_ 的实参的类型，见 [_Item 30_](Chapter%205.md#item-30-熟悉完美转发失败的场景)，如果客户转递了不合适的实参的类型的话，那么编译器的错误信息可以让你感到惶恐，见 [_Item 27_](Chapter%205.md#item-27-熟悉重载-univeral-reference-的替代方法)。

如果存在一个方法可以写像 _addName_ 一样左值被拷贝，右值被移动的函数，而且在源代码和目标代码中只有一个函数需要去处理并且可以避免 _universal reference_ 的复杂的话，那么这该有多棒。碰巧是存在的。你需要去做的是放弃可能是你做为 _C++_ 程序员所学习到第一条规则。这个规则是避免按 _by-value_ 的形式去传递用户定义的对象。对于像 _addName_ 这样的函数中的像 _newName_ 这样的形参，_pass-by-value_ 可能是完全合理的策略。

在我们讨论为什么 _pass-by-value_ 对于 _newName_ 和 _addName_ 是非常合适的前，我们先看下它是如何被实现的：  
```C++
  class Widget {
  public:
    void addName(std::string newName)             // take lvalue or
    { names.push_back(std::move(newName)); }      // rvalue; move it
    
    …
    
  };
```

这个代码的唯一不明显的地方是应用 _std::move_ 到形参 _newName_ 上。通常来说，_std::move_ 被用于右值引用，但是这个场景中，我们知道：（1） _newName_ 是一个完全和用户传入的实参无关的对象，所以改变 _newName_ 不会影响到调用方。（2）这是最后一次使用 _newName_，所以移动 _newName_ 不会对剩下的函数有任何影响。

只存在有一个 _addName_ 的事实解释了我们如何避免了在源代码和目标代码中的代码重复。我们没有使用 _universal reference_，所以这个方法不会导致头文件膨胀、奇怪的失败场景或令人困惑的错误信息。但是这个设计的效率怎么样呢？我们正在使用 _pass-by-value_。这不是成本大的吗？

在 _C++98_ 中，这是一个合理的赌注。不管用户所传入的是什么，形参 _newName_ 都是被拷贝构造出的。然而，在 _C++11_ 中，_addName_ 只会对于左值进行拷贝构造。对于右值，它将会进行移动构造。看下面：
```C++
  Widget w;
  
  …
  
  std::string name("Bart");
  
  w.addName(name);                      // call addName with lvalue
  
  …
  
  w.addName(name + "Jenne");            // call addName with rvalue
                                        // (see below)
```  

在第一个 _addName_ 调用中，当 _name_ 被传入时，形参 _newName_ 是使用左值来初始化的。因此 _newName_ 是被拷贝构造的，就像在 _C++98_ 中一样。在第二个 _addName_ 调用中，形参 _newName_ 是使用 _std::string_ 所对应的 _operator+_ 的调用所生成的 _std::string_ 对象来初始化的，即为：_append_ 操作。这个所生成的对象是右值，因此 _newName_ 是被移动构造的。 

因此，左值是被拷贝的，右值是被移动的，就像我们想的那样，干净漂亮，对吧？

是干净漂亮的，但是有一些警告你需要牢记。如果我们回顾一下我们考虑过的 _addName_ 的三个版本，那么完成这个就更简单了：  
```C++
  class Widget {                                  // Approach 1:
  public:                                         // overload for
    void addName(const std::string& newName)      // lvalues and
    { names.push_back(newName); }                 // rvalues
    
    void addName(std::string&& newName)
    { names.push_back(std::move(newName)); }
    …
  
  private:
    std::vector<std::string> names;
  };
  
  class Widget {                                  // Approach 2:
  public:                                         // use universal
    template<typename T>                          // reference
    void addName(T&& newName)
    { names.push_back(std::forward<T>(newName)); }
    …
  };

  class Widget {                                  // Approach 3:
  public:                                         // pass by value
  void addName(std::string newName)
  { names.push_back(std::move(newName)); }
  …
  };
```  
我把前两个版本称为 **_by-reference approach_**，因为它们都是基于按 _by-reference_ 的形式传递的它们的形参。

下面是两个我们已经考察过的两个调用情景：  
```C++
  Widget w;
  …
  std::string name("Bart");
  
  w.addName(name);                      // pass lvalue
  …
  w.addName(name + "Jenne");            // pass rvalue
```

从拷贝和移动的角度来看，现在考虑为这两个调用情景和我们已经讨论过的三个 _addName_ 实现中的每一个添加 _name_ 到 _Widget_ 的成本。这个成本将会在最大程度上忽略编译器优化掉拷贝和移动的可能性，因为这样的优化是依赖于上下文和编译器的，实际上，这样的优化不会改变这个分析的本质。  
* 重载：不管是左值还是右值被传入，调用方的实参绑定的都是被称为 _newName_ 的引用。从拷贝和移动的角度看，没有成本。在左值重载函数中，_newName_ 是被拷贝到 _Widget::names_ 中的。在右值重载函数中，_newName_ 是被移动到 _Widget::names_ 中的。成本合计：对于左值是一次拷贝，对于右值是一次移动。
* 使用 _universal reference_：就像重载那样，调用方的实参绑定是引用 _newName_。这是一个 _no-cost_ 操作。由于使用了 _std::forward_，左值 _std::string_ 实参是被拷贝到 _Widget::names_ 中的，而右值 _std::string_ 实参是被移动到 _Widget::names_ 中的。_std::string_ 实参所对应的成本合计和重载是一样的：对于左值是一次拷贝，对于右值是一次移动。  
[_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference) 解释了如果调用方传递的是 _std::string_ 之外的类型的实参的话，那么这个实参也会被转发给 _std::string_ 的构造函数，这只会导致 _0_ 个 _std::string_ 的拷贝和移动被执行。因此持有 _universal reference_ 的函数可以是独有高效的。然而，这不会影响本 _Item_ 中的分析，我们通过假设调用方总是传递 _std::string_ 实参来保持事情简单。
* _pass-by-value_：不管是左值还是右值被传入，形参 _newName_ 都必须被构造。如果是左值被传入的话，那么成本是拷贝构造。如果是右值被传入的话，那么成本是移动构造。在这个函数中，_newName_ 是被无条件地移动到 _Widget::names_ 中的。成本合计是：对于左值是一次拷贝加一次移动，对于右值是两次移动。相比于 _by-reference approach_，对于左值和右值都有一次额外的移动。

再看一次本 _Item_ 的标题：
对于移动是成本小的且总是会被拷贝的可拷贝的形参考虑 _pass-by-value_。

![Alt text](image/image11.jpg)

这么写是有原因的。实际上，有 _4_ 个原因：  
* 你应该只考虑使用 _pass-by-value_。是的，它只需要写一个函数。是的，它只会在目标代码中生成一个函数。是的，它避免了 _universal reference_ 所相关的问题。但是它的成本比替代方法的成本要大，正如我们在下面看到的，在一些场景中，存在一些我们还没有讨论过的成本。
* 只对于可拷贝的形参，考虑 _pass-by-value_。不符合这个条件的形参必定包含有 _move-only_ 类型，因为如果这些形参是不可拷贝的，但是函数总是会创建副本的话， 那么必须要通过 _move constructor_ 来创建副本。回忆一下 _pass-by-value_ 相比于重载的优势是：当使用 _pass-by-value_ 时，只有一个函数必须被写出来。但是对于 _move-only_ 类型来说，不需要为左值实参来提供重载函数了，因为拷贝左值必须要调用所对应的 _copy constructor_，_move-only_ 类型所对应的 _copy constructor_ 是被删除的。这意味着只有右值实参需要被支持，在这种场景中，**_重载_** 方案只需要一个函数：持有右值引用的函数。  
考虑一个类包含有 _std::unique_ptr&lt;std::string&gt;_ 类型的数据成员和这个数据成员所对应的 _setter_。因为 _std::unique_ptr_ 是 _move-only_ 类型，所以这个类的 _setter_ 的 **_重载_** 方法是由单一的函数组成的。调用方可以像下面这样使用这个类的 _setter_。此处的 _std::make_unique_ 所返回的右值 _std::unique_ptr&lt;std::string&gt;_，见 [_Item 21_](Chapter%204.md#item-21-首选-stdmake_unique-和-stdmake_shared-而不是直接使用-new)，是按 _by-rvalue-reference_ 的方式传递给 _setPtr_ 的，在 _setPtr_ 中这个右值会被移动到数据成员 _p_ 中。整体成本是一次移动。如果 _setPtr_ 是按 _by-value_ 的形式持有它的形参的话，那么相同的调用将会移动构造形参 _ptr_，_ptr_ 也将会被移动到数据成员 _p_ 中。因此整体成本是两次移动，是 **_重载_** 方法的两倍。  
```C++
  class Widget {
  public:
    …
    void setPtr(std::unique_ptr<std::string>&& ptr)
    { p = std::move(ptr); }

  private:
    std::unique_ptr<std::string> p;
  };
```  
```C++
  Widget w;
  …
  
  w.setPtr(std::make_unique<std::string>("Modern C++"));

```  
```C++
  class Widget {
  public:
    …
    
    void setPtr(std::unique_ptr<std::string> ptr)
    { p = std::move(ptr); }

    …
  };
```
* 只对于那些移动是成本小的的形参，_pass-by-value_ 才是值得考虑的。当移动是成本小的时，额外的一个移动的成本是可以接受的，但是当移动是成本大的时，执行不必要的移动类似于执行不必要的拷贝，避免不必要的拷贝的重要性正是 _C++98_ 中要避免 _pass-by-value_ 的规则的出发点。  
* 你应该只对那些总是会被拷贝的形参考虑 _pass-by-value_。为了搞清楚为什么这是重要的，假定： _addName_ 会在拷贝它的形参到 _names_ _container_ 之前，先去检查新 _name_ 是太短或太长。如果是太短或太长的话，那么所添加的 _name_ 是被忽略的。一个 _pass-by-value_ 实现可以被写成下面这样。这个函数产生了构造和销毁 _newName_ 的成本，即使没有东西被添加到 _names_ 中。这是 _by-reference approach_ 不会被要求付出的成本。   
```C++
  class Widget {
  public:
    void addName(std::string newName)
    {
      if ((newName.length() >= minLen) &&
      (newName.length() <= maxLen))
      {
        names.push_back(std::move(newName));
      }
    }

    …

  private:
    std::vector<std::string> names;
}; 
```

甚至当你正在处理一个对移动是成本小的可拷贝的类型执行了无条件的拷贝的函数时，_pass-by-value_ 可能也有不合适的时候。这是因为函数可以通过下面两种方法来拷贝形参：构造方法，即为：_copy constructor_ 和 _move constructor_，和赋值方法，即为：_copy assignment operator_ 和 _move assignment operator_。_addName_ 使用了构造方法：它的形参 _newName_ 被传递给了 _vector::push_back_，在这个函数中，_newName_ 被拷贝构造到了在 _std::vector_ 最后所创建的新元素中。对于使用构造方法去拷贝它们的形参的函数，我们之前看到的分析是完整的：对于左值实参和右值实参，使用 _pass-by-value_ 产生了一个额外的移动。

当形参是使用赋值方法被拷贝的时，情况是更加复杂的。例如：假定我们有一个表示密码的类。因为密码可以被更改，所以我们提供了一个 _setter_ 函数 _changeTo_。使用 _pass-by-value_ 策略，我们可以像下面这样实现 _PassWord_：  
```C++
  class Password {
  public:
    explicit Password(std::string pwd)            // pass by value
    : text(std::move(pwd)) {}                     // construct text
    
    void changeTo(std::string newPwd)             // pass by value
    { text = std::move(newPwd); }                 // assign text
    
    …
  
  private:
    std::string text;                             // text of password
  };
```

以 _plain text_ 的方式存储密码会让软件安全 _SWAT_ 团队狂怒，但是忽略这个，并考虑这样的代码：  
```C++
  std::string initPwd("Supercalifragilisticexpialidocious");
  
  Password p(initPwd);
```  

此处没有什么惊喜：_p.text_ 是使用给定的密码进行构建的，在构造函数中使用 _pass-by-value_ 产生了一个 _std::string_ 移动构造的成本，如果重载和完美转发被使用了的话，那么这个成本是不需要的。一切都好。

然而，这个程序的用户可能对这个密码不太乐观，因为 _Supercalifragilisticexpialidocious_ 在很多词典中都可以被找到。因此，他或她采取了行动，导致了相当于下面这样的代码被执行：  
```C++
  std::string newPassword = "Beware the Jabberwock";
  
  p.changeTo(newPassword);
```  
新密码是否是好于旧密码是可讨论的，这是用户的问题。

_changeTo_ 使用赋值方法去拷贝形参 _newPwd_ 可能会导致函数的 _pass-by-value_ 成本激增。

因为所传递给 _changeTo_ 的实参是左值 _newPassword_，所以形参 _newPwd_ 是被构造的，调用的是 _std::string_ 的 _copy constructor_。这个构造函数会分配内存去保存新密码。然后 _newPwd_ 会 _move-assigned_ 给 _text_，这会导致已经被 _text_ 占用的内存被释放掉。在 _changeTo_ 中存在两个动态内存管理的动作：一个是新密码所对应的内存分配，一个是旧密码的内存释放。

但是在这个场景中，旧密码 _Supercalifragilisticexpialidocious_ 的长度是长于新密码 _Beware the Jabberwock_ 的长度的，所以不需要分配和释放任何东西。如果重载方法被使用了的话，那么可能这些都不会发生：  
```C++
  class Password {
  public:
    …
    
    void changeTo(const std::string& newPwd)      // the overload
    {                                             // for lvalues
      
      text = newPwd;                              // can reuse text's memory if
                                                  // text.capacity() >= newPwd.size()
    }

    …
  
  private:
    std::string text;                             // as above
  };
```  
在这个情景中，_pass-by-value_ 的成本包含了一个额外的内存分配和释放，这个成本可能比 _std::string_ 的移动的成本高出几个数量级。

有趣地是，如果旧密码的长度是短于新密码的长度的话，那么通常可以在赋值期间避免分配和释放操作，在这个场景中，_pass-by-value_ 和 _pass-by-reference_ 的速度是相同的。因此 _assignment-based_ 形参的拷贝的成本可以依赖于参与赋值的对象的值。这种分析适用于所有使用动态内存分配保存值的形参类型。不是所有的类型都符合这个条件，但是很多都符合，包括：_std::string_ 和 _std::vector_。

只有当左值被传入时，这个潜在的成本增长通常才会发生，因为只有当真正的拷贝，即为：不是移动，被执行时，执行内存分配和释放的需求通常才会发生。对于右值形参，移动几乎总是足够的。

总之，对于使用赋值进行拷贝形参的函数，_pass-by-value_ 的成本依赖于所传递的类型、左值实参与右值实参的比例、以及所对应的类型是否使用了动态内存分配，如果是的话，还有这个类型的 _assignment operator_ 的实现和赋值的目标所相关的内存和赋值的源头所相关的内存至少是一样大的可能性。对于 _std::string_ 来说，成本还依赖于：实现是否使用了 _small string optimization_ _SSO_，见 [_Item 29_](Chapter%205.md#item-29-假设-move-operation-是不存在的成本大的和不可使用的)，如果是的话，那么被赋值的值是否适合所对应的 _SSO buffer_。

所以，正如我说的，当形参是通过赋值方法被拷贝的时，分析 _pass-by-value_ 的成本是复杂的。通常来说，最实用的方法是采取 **_guilty until proven innocent_** 的策略，你需要使用重载或 _universal reference_ 而不是 _pass-by-value_，除非能证实对于你需要的形参类型，_pass-by-value_ 能产生可接受的高效的代码。

现在，对于要尽可能运行要快的软件，_pass-by-value_ 可能不是可行的策略，因为即使移动是成本小的，避免移动也是重要。此外，究竟会发生多少次移动也不总是清晰的。在 _Widget::addName_ 的例子中，_pass-by-value_ 只产生了一个额外的移动，但是假定：_Widget::addName_ 调用了 _Widget::validateName_，而这个 _Widget::validateName_ 也是 _pass-by-value_ 的，_Widget::validateName_ 总是要拷贝它们的形参可能也是有理由的，比如：需要存储它们的形参到数据结构中。再假设 _validateName_ 调用了也是 _pass-by-value_ 的第三方函数...  

你可以看出这样下去会怎么样。当存在这样的函数调用链路时，如果整个链路中的每一个都会使用 _pass-by-value_ 的话，因为所对应的移动的成本是小的，那么整个链路所对应的成本可能也是你不可以接受的。当使用 _pass-by-reference_ 时，整个链路上的调用就不会产生这种累计的成本了。

一个和性能不相关的但是仍然需要保持注意的问题是 _pass-by-value_ 不同于 _pass-by-reference_，它容易遭遇切割问题。这是 _C++98_ 中经常遇到的问题，所以我就不再阐述了，但是如果你有一个函数，它被设计去接受 _base class_ 类型的形参或继承自这个 _base class_ 的 _derived class_ 类型的形参的话，那么你不会想要声明这个类型的 _pass-by-value_ 的形参的，因为你将切割可能被传入的所有 _derived class_ 类型的对象所对应的 _derived-class_ 的特性：  
```C++
  class Widget { … };                             // base class
  
  class SpecialWidget: public Widget { … };       // derived class
  
  void processWidget(Widget w);                   // func for any kind of Widget,
                                                  // including derived types;
  …                                               // suffers from slicing problem
  
  SpecialWidget sw;
  
  …
  
  processWidget(sw);                              // processWidget sees a
                                                  // Widget, not a SpecialWidget!
```

如果你不熟悉切割问题的话，那么搜索引擎和互联网会是你的朋友，上面有很多有用的信息。你会发现切割问题是另一个 _pass-by-value_ 在 _C++98_ 是如此负面的原因。这也是为什么关于 _C++_ 编程你学习到的第一件事是去避免按 _by-value_ 的形式去传递用户定义的对象。

_C++11_ 不会从根本上改变关于 _pass-by-value_ 的智慧。通常来说，_pass-by-value_ 仍然会有你想要去避免的性能损耗，_pass-by-value_ 仍然可能导致切割问题。_C++11_ 中新引入的特性可以区分左值实参和右值实参。实现函数可以对可拷贝的类型的右值利用移动语义，这需要重载或使用 _universal reference_。这两个都有缺点。对于所传递给那些总是拷贝形参且不关心切割问题的函数的移动是成本小的可拷贝的类型的特殊情况，_pass-by-value_ 可以提供一个容易去实现的替代的方法，这个方法的效率接近于 _pass-by-reference_，但是却避免了后者的劣势。

### 需要记住的规则

* 对于移动是成本小的且总是会被拷贝的可拷贝的形参，_pass-by-value_ 和 _pass-by-reference_ 的效率是接近的，_pass-by-value_ 非常容易实现，并且可以生成较少的目标代码。
* 通过赋值方法拷贝形参的成本可能比通过构造方法拷贝形参的成本要大的多。
* _pass-by-value_ 会遭遇切割问题，所以它对于 _base class_ 类型的形参是不合适的。

## _Item 42_ 考虑使用 _emplacement_ 来代替 _insertion_

如果你有一个持有 _std::string_ 的容器，当你通过 _insertion_ 函数，比如：_insert_、_push_front_、_push_back_ 和 _std::forward_list_ 的 _insert_after_ 等，来增加一个新的元素时，你将要传递给函数的元素的类型是 _std::string_，这似乎是符合逻辑的。毕竟，那是 _container_ 中的元素的类型。

尽管逻辑可能是这样，但也不总是这样。考虑下面这样的代码：
```C++
  std::vector<std::string> vs;          // container of std::string
  vs.push_back("xyzzy");                // add string literal
```  
此处，这个 _container_ 持有 _std::string_，但是你手上有的，你实际正在尝试给到 _push_back_ 的是 _string literal_，即为：_""_ 中的一个字符序列。_string literal_ 不是 _std::string_，这意味着你正在传递给 _push_back_ 的实参不是这个 _container_ 所持有的类型。

_std::vector_ 所对应的 _push_back_ 重载了左值和右值，像下面这样：  
```C++
  template <class T,                              // from the C++11
            class Allocator = allocator<T>>       // Standard
  class vector {
  public:
    …
    void push_back(const T& x);                   // insert lvalue
    void push_back(T&& x);                        // insert rvalue
    …
  };
```  
在这个调用中  
```C++
  vs.push_back("xyzzy");
```  
编译器看到实参的类型 _const char[6]_ 和 _push_back_ 所持有的形参的类型 _std::string_ 引用是不匹配的。编译器可以通过根据 _string literal_ 来创建临时 _std::string_ 并将这个临时 _std::string_ 传递给 _push_back_ 来解决这个不匹配。换句话说，编译器对待这个调用就像下面这样：  
```C++
  vs.push_back(std::string("xyzzy"));   // create temp. std::string
                                        // and pass it to push_back
```   
这个代码可以通过编译和运行，所有人都可以高兴地回家了。是除了性能怪以外的所有人，因为性能怪意识到了这个代码没有它应该有的高效。  

为了创建 _std::string_ 的 _container_ 中的新元素，性能怪知道，一个 _std::string_ 的构造函数必须被调用，但是上面的代码不是只调用了一次构造函数。是两次。还调用了一次 _std::string_ 的析构函数。下面是在运行时调用 _push_back_ 会发生的事情：  
*  临时 _std::string_ 对象是根据 _string literal_ 所创建的。这个对象没有名字，我们称它为 _temp_。_temp_ 的构造是第一次 _std::string_ 的构造。因为 _temp_ 是个临时对象，所以它是个右值。
*  _temp_ 是被传递给 _push_back_ 所对应的右值重载函数的，_temp_ 绑定了右值引用形参 _x_。_x_ 的副本是在 _std::vector_ 的内存中被构造的。这个构造是第二次构造，这个构造在 _std::vector_ 中创建了一个新的对象。被用于将 _x_ 拷贝到 _std::vector_ 中的构造函数是移动构造函数，因为做为右值引用的 _x_ 在它被拷贝前被转换为了右值。对于转换右值引用形参为右值的信息，见 [_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference)。
*  在 _push_back_ 返回后，_temp_ 立即被销毁，因此会调用 _std::string_ 的析构函数。
  
性能怪不能提供帮助，但是提醒说：如果存在一个方法可以获取 _string literal_ 并可以将这个 _string literal_ 直接传递给在 _std::vector_ 中所构造出的 _std::string_ 对象的话，那么我们可以避免构造和析构 _temp_。这将是极大高效的，甚至性能怪也可以满意。因为你是 _C++_ 程序员，大概率你也是性能怪。如果你不是的话，那么你仍然可能会认可性能怪的观点。如果你完全对性能不感兴趣的话，那么你不应该出门去 _python_ 的房间吗？我非常高兴告诉你存在一个方法可以在调用 _push_back_ 时实现效率最大化。不是去调用 _push_back_。_push_back_ 是错误的函数。你想要的函数是 _emplace_back_。

_emplace_back_ 做的完全如你所愿：_emplace_back_ 使用所传递给它的形参去直接在 _std::vector_ 中构造 _std::string_。没有涉及到临时变量：  
```C++
  vs.emplace_back("xyzzy");             // construct std::string inside
                                        // vs directly from "xyzzy"
```   
因为 _emplace_back_ 使用了完美转发，所以只要你没有撞上完美转发的限制的话，见 [_Item 30_](Chapter%205.md#item-30-熟悉完美转发失败的场景)，那么你可以传递任意数量任意类型组合的形参给 _emplace_back_，例如：如果你想要通过持有字符和重复数量的 _std::string_ 的构造函数来在 _vs_ 中去创建 _std::string_ 的话，那么你可以这样做：  
```C++
  vs.emplace_back(50, 'x');             // insert std::string consisting
                                        // of 50 'x' characters
```  
_emplace_back_ 对于所有支持 _push_back_ 的标准 _container_ 都是有效的。类似，所有支持 _push_front_ 的标准 _container_ 都支持 _emplace_front_。除了 _std::forward_list_ 和 _std::array_ 以外的所有支持 _insert_ 的标准 _container_ 都支持 _emplace_。_associative containers_ 提供了 _emplace_hint_ 去补充它们的持有着 **_hint_** _iterator_ 的 _insert_ 函数，_std::forward_list_ 拥有 _emplace_after_ 去匹配它们的 _insert_after_。

使得 _emplacement_ 函数可以胜过 _insertion_ 函数的是前者的更加灵活的接口。_insertion_ 函数接受的是将要被插入的对象，而 _emplacement_ 函数接受的是将要被插入的对象所对应的构造函数的实参。这个区别允许 _emplacement_ 函数去避免 _insertion_ 函数必须要的临时对象的创建和销毁。

因为 _container_ 所持有的类型的实参可以被传递给 _emplacement_ 函数，这些实参会导致所对应的函数去执行拷贝构造或者移动构造，甚至当 _insertion_ 函数不需要临时变量时，_emplacement_ 也可以被使用。在那种情况下，_insertion_ 和 _emplacement_ 本质上做的是相同的事情。例如：给定：
```C++
  std::string queenOfDisco("Donna Summer");
```  
下面这两个调用都是有效的，它们两个对 _container_ 有着相同的净效果：  
```C++
  vs.push_back(queenOfDisco);           // copy-construct queenOfDisco
                                        // at end of vs
  
  vs.emplace_back(queenOfDisco);        // ditto
```  

因此 _emplacement_ 函数可以完成 _insertion_ 函数可以完成的所有事。有时候 _emplacement_ 函数完成的更高效，至少理论上，它们应该永远不会低效地完成。所以，为什么不一直使用 _emplacement_ 函数呢？

因为正如俗话所说，在理论上，理论和现实之间没有区别，但是在现实中，理论和现实之间有区别。在当前的标准库实现中，正如所期待的，存在 _emplacement_ 胜过 _insertion_ 的情景。但是糟糕地是，也存在 _insertion_ 运行更快的情景。这样的情景不太容易去描述，因为这依赖于所传递的实参的类型、所使用的 _container_、需要 _insertion_ 和 _emplacement_ 的 _container_ 中的位置、所包含的类型的构造函数的异常安全性以及对于重复值是被禁止的 _container_，即为：_std::set_、_std::map_、_std::unordered_set_ 和 _std::unordered_map_，将要被记录的值是否已经在 _container_ 中了。因此通常的性能调优建议适用于：想要确定 _emplacement_ 或 _insertion_ 哪个运行更快，可以对它们进行测试。

当然，这不是非常令人满意的，所以你会非常高兴学习启发思维，这个启发思维可以帮助你识别 _emplacement_ 函数是最有可能值得的情景的方法。如果下面的这些全部都是满足的话，那么 _emplacement_ 几乎肯定是胜过 _insertion_ 的：
* 所增加的值是被构造而不是赋值到 _container_ 中的。本 _Item_ 开头的例子：增加具有 _xyzzy_ 的 _std::string_ 到 _std::vector_ vs 中，展示了被增加到 _vs_ 的尾部的值，这个尾部是一个还没有数据存在的位置。因此，新值必须被构造到 _std::vector_ 中。如果我们修改这个例子，以让新的 _std::string_ 去到一个已经被某个对象所占用了的位置的话，那么就是不同的故事了。考虑下面的代码。对于这个代码，很少会有实现将所增加的 _std::string_ 构造到 _vs[0]_ 所占用的内存中。相反，都是将所增加的 _std::string_ _move-assign_ 到 _vs[0]_ 所占用的内存中。但是这需要可以移动的对象，这意味着一个临时对象需要被创建来做为移动的源。因为 _emplacement_ 胜过 _insertion_ 的主要优势是临时对象既不会被创建，也不会被销毁，所以当被增加的值是通过赋值被放到 _container_ 时，_emplacement_ 的优势就消失了。  
增加一个值到 _container_ 中是通过构造还是赋值来完成通常是取决于实现者的。但是再一次，启发思维可以帮助。_node-based_ _container_ 几乎总是使用构造去增加新值的，大多数标准 _container_ 是 _node-based_。唯一不是的是 _std::vector_、_std::deque_ 和 _std::string_，_std::array_ 不是，但是它不支持 _insertion_ 和 _emplacement_，所以它和这里不相关。在 _non-node-based_ _container_ 中，你可以依赖 _emplace_back_ 以去使用构造而不是赋值来将新值放到所对应的位置，对于 _std::deque_ 的 _emplace_front_ 也是这样的。
```C++
  std::vector<std::string> vs;          // as before
  
  …                                     // add elements to vs
  
  vs.emplace(vs.begin(), "xyzzy");      // add "xyzzy" to
                                        // beginning of vs
```
* 所传递的实参的类型和所对应的 _container_ 所持有的类型是不相同的。再次，_emplacement_ 胜过 _destruction_ 是因为当所传递的实参的类型不是所对应的 _container_ 所持有的类型时，_emplacement_ 的接口不需要创建和销毁临时对象。当类型 _T_ 的对象被增加到 _container&lt;T&gt;_ 时，没有理由期待 _emplacement_ 是快于 _insertion_ 的，因为没有临时对象需要被创建去满足 _insertion_ 接口。
* _container_ 不太可能因为重复去拒绝新值。这意味着 _container_ 要么允许重复，要么你增加的大部分的值是唯一的。这个很重要，因为为了探测某个值是否已经存在于所对应的 _container_ 中了，_emplacement_ 的实现通常会创建一个具有新值的 _node_，为的是可以将这个值和 _container_ 中已经存在的值做比较。如果被增加的值是不在所对应的 _container_ 中的话，那么这个被增加的值的 _node_ 是会被链入的。然而，如果被增加的值是已经在所对应的 _container_ 中的话，那么这个 _emplacement_ 是会被拒绝的，这个被增加的值的 _node_ 是会被销毁的，这意味着构造和析构的成本是被浪费的。这样的 _node_ 经常为 _emplacement_ 函数而非 _insertion_ 函数创建。

之前在本 _Item_ 就提到过的调用满足上面的所有条件。_emplace_back_ 也是快于相应的 _push_back_ 调用的：
```C++
  vs.emplace_back("xyzzy");             // construct new value at end of
                                        // container; don't pass the type in
                                        // container; don't use container
                                        // rejecting duplicates
  
  vs.emplace_back(50, 'x');             // ditto
```

当决定是否使用 _emplacement_ 函数时，其他的两个问题值得牢记。第一个是关于资源分配的。假定你有一个 _std::shared_ptr&lt;Widget&gt;_ 的容器，  
```C++
  std::list<std::shared_ptr<Widget>> ptrs;
```  
并且你想要增加一个应该通过 _custom deleter_ 进行释放的 _std::shared_ptr_ 的话，见 [_Item 19_](Chapter%204.md#item-19-对于-shared-ownership-的资源管理使用-stdshared_ptr)。[_Item 21_](Chapter%204.md#item-21-首选-stdmake_unique-和-stdmake_shared-而不是直接使用-new) 解释了：只要有可能，你都应该去使用 _std::make_shared_ 去创建 _std::shared_ptr_，但是也有不可以使用 _std::make_shared_ 的情景。这样的一个情景是当你想要去指明 _custom deleter_ 时。在这种场景中，你必须直接使用 _new_ 来获取将会被 _std::shared_ptr_ 所管理的原始指针。

如果 _custom deleter_ 是下面这个函数的话：  
```C++
  void killWidget(Widget* pWidget);
```  
那么使用 _insertion_ 函数的代码可能看下来像是这样：  
```C++
  ptrs.push_back(std::shared_ptr<Widget>(new Widget, killWidget));
```  
也可能看起来像是这样，尽管含义是相同的：  
```C++
  ptrs.push_back({ new Widget, killWidget });
```

不管哪种方式，一个临时的 _std::shared_ptr_ 将会在调用 _push_back_ 之前被构造。_push_back_ 的形参是 _std::shared_ptr_ 的引用，所以形参必须指向 _std::shared_ptr_。

临时 _std::shared_ptr_ 的创建是 _emplace_back_ 要避免的，但是在这个场景中，这个临时对象是值得的，远超过它的成本。考虑下面潜在的事件顺序：  
* 在上面的两个调用中，一个临时的 _std::shared_ptr&lt;Widget&gt;_ 对象是被构造出来以去持有 _new Widget_ 所生成的原始指针的。称这个对象为 _temp_。
* _push_back_ 按 _by-value_ 的形式持有 _temp_。在持有 _temp_ 的副本的 _list node_ 的分配期间，一个 _out-of-memory_ 异常抛出了。
* 当这个异常传播到 _push_back_ 外面时，_temp_ 是被销毁的。
做为唯一的 _std::shared_ptr_，它指向着它所管理的 _Widget_，它会自动释放所管理的 _Widget_，在这个场景中，是通过调用 _killWidget_。

尽管一个异常发生了，但是没有东西泄露：在 _push_back_ 中通过 _new Widget_ 所创建的 _Widget_ 是在被创建以去管理 _temp_ 的 _std::shared_ptr_ 的析构函数中进行释放的。生活是美好的。

现在考虑：如果调用的是 _emplace_back_ 而不是 _push_back_ 的话，那么会发生什么：  
```C++
  ptrs.emplace_back(new Widget, killWidget);
```  
* _new Widget_ 所生成的原始指针是被完美转发给 _emplace_back_ 的，并且在 _emplace_back_ 中将会分配一个 _list node_。这个分配失败，一个 _out-of-memory_ 异常被抛出。
*  当这个异常传播到 _push_back_ 外面时，唯一可以获取堆上的 _Widget_ 的原始指针就丢失了。这个 _Widget_ 和它所拥有的所有资源被泄露了。
  
在这个情景中，生活不是美好的，错误不在于 _std::shared_ptr_。同类型的问题可以在使用 _std::unique_ptr_ 和 _custom deleter_ 时发生。从根本上说，像 _std::shared_ptr_ 和 _std::unique_ptr_ 这样的资源管理类想要发挥作用，前提是像根据 _new_ 所生成的原始指针这样的资源要被立即传递给资源管理的对象的构造函数。像 _std::make_shared_ 和 _std::make_unique_ 这样的函数会自动化这个过程，这个事实是它们如此重要的原因。

在持有资源管理的对象的 _container_，比如：_std::list&lt;std::shared_ptr&lt;Widget&gt;&gt;_，的 _insertion_ 函数的中，这个函数的形参的类型通常确保了在资源获取，比如：使用 _new_，和管理着所对应的资源的那个对象的构造之间无任何东西。在 _emplacement_ 函数中，完美转发会推迟资源管理的对象的创建，直到这些对象可以在所对应的 _container_ 的内存中被构造，这会打开一个窗口，在此期间异常可以导致资源泄露。所有的标准 _container_ 都容易受到这个问题的影响。但是使用资源管理的对象的 _container_ 时，你必须注意确保：如果你选择了 _emplacement_ 函数来代替 _insertion_ 函数的话，那么你不会因为提升代码效率而牺牲异常安全。

坦率地说，你不应该传递像 _new Widget_ 这样的表达式给到 _emplace_back_、_push_back_ 或大多说其他函数，因为 [_Item 21_](Chapter%204.md#item-21-首选-stdmake_unique-和-stdmake_shared-而不是直接使用-new) 所解释的，这会导致发生我们刚才讨论过的那种异常安全问题。关上这扇门需要将 _new Widget_ 所生成的原始指针移交给独立语句中的资源管理的对象，然后将这个资源管理的对象做为右值传递给你最初想要将 _new Widget_ 进行传递的那个函数，[_Item 21_](Chapter%204.md#item-21-首选-stdmake_unique-和-stdmake_shared-而不是直接使用-new) 覆盖了这个技术的更多细节。因此，使用了 _push_back_ 的代码应该被写成下面这样：  
```C++
  std::shared_ptr<Widget> spw(new Widget,         // create Widget and
                              killWidget);        // have spw manage it
  
  ptrs.push_back(std::move(spw));                 // add spw as rvalue
```   
_emplace_back_ 的版本也是类似的：  
```C++
  std::shared_ptr<Widget> spw(new Widget, killWidget);
  ptrs.emplace_back(std::move(spw));
```  
不管哪种方法，都产生了创建和销毁 _spw_ 的成本。鉴于选择 _emplacement_ 而不是 _insertion_ 的动机是它可以避免所对应的 _container_ 所持有的类型的临时对象的成本，概念上 _spw_ 就是这样的临时对象，当你增加资源管理的对象到一个 _container_ 中，并且你遵循了正确的做法，确保了在获取资源和将这个资源移交给资源管理的对象之间无任何东西可以介入时，_emplacement_ 函数是不可能胜过 _insertion_ 函数的。

_emplacement_ 函数第二个值得注意的地方是它们和 _explicit_ 构造函数的交互。为了庆祝 _C++11_ 对 _regular expression_ 的支持，假定你创建了 _regular expression_ 对象的 _container_：  
```C++
  std::vector<std::regex> regexes;
```   

你的同事在为每天访问 _Facebook_ 账户的理想次数而争吵，这分散了你的注意力，你不小心地写下了下面看似无意义的代码：  
```C++
  regexes.emplace_back(nullptr);        // add nullptr to container
                                        // of regexes?
```  

当你写的时候，你没有注意到，你的编译器接受了这个代码并且没有抱怨，所以你最终会浪费大量的调试时间。在某个时间点，你发现你将一个空指针插入到了你的 _regular expression_ 的 _container_ 中。但是如何可能这样呢？指针不是 _regular expression_，如果试着写像下面这样的代码的话，  
```C++
  std::regex r = nullptr;               // error! won't compile
```  
那么编译器会拒绝你的代码。有趣地是，如果你调用的是 _push_back_ 而不是 _emplace_back_ 的话，那么也会拒绝你的代码：  
```C++
  regexes.push_back(nullptr);           // error! won't compile
```  
你正在经历的奇怪行为是因为 _std::regex_ 对象可以根据字符串进行构造。这让下面代码变得合理：  
```C++
  std::regex upperCaseWord("[A-Z]+");
```  
根据字符串创建的 _std::regex_ 可以造成相对较大的运行成本，所以为了最小化这个成本无意产生的可能性，_std::regex_ 的持有 _const char*_  指针的构造函数是 _explicit_。这也是为什么这些代码无法通过编译：  
```C++
  std::regex r = nullptr;               // error! won't compile
  
  regexes.push_back(nullptr);           // error! won't compile
```   
在这些场景中，我们要求从指针到 _std::regex_ 的隐式转换，而 _std::regex_ 的构造函数的 _explicitness_ 会阻止这样的转换。

然而，在 _emplace_back_ 的调用中，我们没有传递 _std::regex_ 对象。相反，我们传递的是 _std::regex_ 对象的构造函数的实参。这不会被认为是隐式转换需求。而是被看成就好像是下面这样的：  
```C++
  std::regex r(nullptr);                // compiles
```   
如果简短的注释 _complies_ 暗示了这是缺乏热情的，这是好的，因为这个代码即使它可以通过编译，也会有 _undefined behavior_。_std::regex_ 的持有 _const char*_  指针的构造函数要求所指向的字符串由一个有效的 _regular expression_ 组成，而空指针不符合这个要求。如果你写了这个代码并且进行了编译的话，那么你可以期待最好的结果是这个代码会在运行时崩溃。如果不够幸运的话，那么你和你的调试器会有一次特别亲密的体验了。

暂时把 _push_back_、_emplace_back_ 和亲密放在一边，注意这些非常相似的初始化语法如何产生的不同的结果：  
```C++
  std::regex r1 = nullptr;              // error! won't compile
  
  std::regex r2(nullptr);               // compiles
```   
按照标准的官方术语，被用于初始化 _r1_ 的语法，使用 _=_ 的语法，被称为是 _copy initialization_。相反，被用于初始化 _r2_ 的语法，使用 _()_ 尽管 _{}_ 也可能被使用，被称为是 _direct initialization_。_copy initialization_ 不允许使用 _explicit_ 构造函数。_direct initialization_ 允许使用 _explicit_ 构造函数。这也是为什么 _r1_ 的那行不可以通过编译，但是 _r2_ 那行可以通过编译。

回到 _push_back_ 和 _emplace_back_ 中，更一般地说，_insertion_ 函数和 _emplacement_ 函数。_emplacement_ 函数使用的是 _direct initialization_，这意味着可以使用 _explicit_ 构造函数。_insertion_ 函数使用的是 _copy initialization_，这意味不可以使用 _explicit_ 构造函数。因此：  
```C++
  regexes.emplace_back(nullptr);        // compiles. Direct init permits
                                        // use of explicit std::regex
                                        // ctor taking a pointer
  
  regexes.push_back(nullptr);           // error! copy init forbids
                                        // use of that ctor
```

这里得到的教训是：当你使用 _emplacement_ 函数时，特别要注意确保要传递正确的实参，因为甚至 _explicit_ 构造函数也会被编译器考虑，因为它们会试着寻找可以按有效的方式理解你的代码的方法。

### 需要记住的规则

* 原则上，_emplacement_ 函数有时应该比 _insertion_ 函数要高效，而且永远不应该低效。
* 原则上，当 （1）被增加的值是被拷贝而不是被赋值到所对应的 _container_ 中时；（2）被传递的实参的类型不同于所对应的 _container_ 所持有的类型时；（3）编译器不会拒绝导致成为重复的被增加的值时；_emplacement_ 函数很可能会更快。
* _emplacement_ 函数可能会执行那些被 _insertion_ 函数所拒绝的类型转换。