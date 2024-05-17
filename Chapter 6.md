- [_Chapter 6_ _lambda expression_](#chapter-6-lambda-expression)
  - [_Item 31_ 避免默认捕获模式](#item-31-避免默认捕获模式)
    - [需要记住的规则](#需要记住的规则)
  - [_Item 32_ 使用初始化捕获来将对象移动到 _closure_ 中](#item-32-使用初始化捕获来将对象移动到-closure-中)
    - [需要记住的规则](#需要记住的规则-1)
  - [_Item 33_ 在 _auto\&\&_ 形参上使用 _decltype_ 来进行完美转发](#item-33-在-auto-形参上使用-decltype-来进行完美转发)
    - [需要记住的规则](#需要记住的规则-2)
  - [_Item 34_ 首选 _lambda_ 而不是 _std::bind_](#item-34-首选-lambda-而不是-stdbind)
    - [需要记住的规则](#需要记住的规则-3)


# _Chapter 6_ _lambda expression_

 _lambda expression_ _lambda_ 是 _C++_ 编程中的游戏变革者。这有点让人惊讶，因为 _lambda_ 并没有给语言带来新的表达能力。所有 _lambda_ 可以完成的事情你都可以手动来完成，只不过是要多些敲代码而已。但是 _lambda_ 可以非常方便的创建函数对象，这对于日常 _C++_ 软件开发的影响是巨大的。当没有 _lambda_ 时，_STL_ 的 _if_ 算法，比如：_std::find_if_、_std::remove_if_、_std::count_if_ 等，通常只能使用最简单的 _predicate_ 来进行调用，但是当有了 _lambda_ 时，就能使用复杂的 _predicate_ 来进行调用了。这种情况也发生在可以自定义 _comparison function_ 的算法中，比如 _std::sort_、_std::nth_element_、_std::lower_bound_ 等。在 _STL_ 之外，_lambda_ 可以快速地来为 _std::unique_ptr_ 和 _std::shared_ptr_ 来创建 _custom deleter_，见 [_Item 18_](Chapter%204.md#item-18-对于-exclusive-ownership-的资源管理使用-stdunique_ptr) 和 [_Item 19_](Chapter%204.md#item-19-对于-shared-ownership-的资源管理使用-stdshared_ptr)，也让线程 _API_ 中的条件变量的 _predicate_ 的规范变得简单了，见 [_Item 39_](Chapter%207.md#item-39-对于-one-shot-事件的通信考虑-void-future)。在标准库之外，_lambda expression_ 方便了回调函数、接口适配函数和单次调用的特定上下文函数的  _on-the-fly_ 规范。_lambda_ 确实让 _C++_ 成为了更友好的编程语言。

_lambda_ 的相关术语可以令人很困惑。这是一个简短的提醒：

*  _lambda expression_ 只是一个表达式。它是代码的一部分。下面的就是 _lambda_。  
```C++
  std::find_if(container.begin(), container.end(),
                [](int val) { return 0 < val && val < 10; });
```
* _closure_ 是 _lambda_ 所创建的运行期间的对象。根据捕获模式，_closure_ 持有所捕获的数据的引用或者副本。在上面的 _std::find_if_ 的调用中，_closure_ 就是在运行期间所传递给 _std::find_if_ 的第三个实参。
* _closure class_ 是根据 _closure_ 所实例化出的类。每个 _lambda_ 都导致编译器生成了唯一的 _closure class_。_lambda_ 中的语句变为了它所对应的 _closure class_ 的成员函数中的可执行指令。

_lambda_ 通常被用来创建 _closure_，并只将这个 _closure_ 做为函数的实参。上面的 _std::find_if_ 就是这种场景。然而，_closure_ 通常也是可以被拷贝的，所以，通常单独一个 _lambda_ 可以对应有多个 _closure_。例如：在下面的代码中，  
```C++
  {
    int x;                                        // x is local variable
    …

    auto c1 =                                     // c1 is copy of the
    [x](int y) { return x * y > 55; };            // closure produced
                                                  // by the lambda

    auto c2 = c1;                                 // c2 is copy of c1

    auto c3 = c2;                                 // c3 is copy of c2

    …

  }
```  
_c1_、_c2_ 和 _c3_ 都是由 _lambda_ 所生成的 _closure_ 的副本。 

一般情况下，完全可以模糊 _lambda_、_closure_ 和 _closure class_ 之间的界限。在后续的 _Item_中，重要的是区分清楚：什么是在编译期间存在的：_lambda_ 和 _closure class_，什么是在运行期间存在的：_closure_，以及它们三者之间是如何联系的。

## _Item 31_ 避免默认捕获模式

_C++11_ 中有两个默认捕获模式：_by-reference_ 和 _by-value_。默认 _by-reference_ 捕获可能会导致悬空引用。默认 _by-value_ 捕获会让你误认为不会被悬空引用问题所影响了，实际上并不是，也会让你误认为 _closure_ 是 _self-contained_ 的，实际上并不是的。

这是本 _Item_ 的执行概要。如果你是工程师而不是领导的话，那么你想要的是骨头上的肉，所以让我们从默认 _by-reference_ 捕获的危险开始吧。

默认 _by-reference_ 捕获会导致 _closure_ 包含上指向局部变量的引用或者指向那些在定义出 _lambda_ 的作用域中的形参的引用。如果由 _lambda_ 所创建的 _closure_ 的生命周期超过了所对应的局部变量或形参的生命周期的话，那么 _closure_ 中的引用将会悬空。例如：假设我们有一个过滤函数的 _container_，每一个过滤函数都持有 _int_ 且返回 _bool_，这个 _bool_ 指明了所传入的值是否满足了过滤条件：  
```C++
  using FilterContainer =                         // see Item 9 for
      std::vector<std::function<bool(int)>>;      // "using", Item 2
                                                  // for std::function

  FilterContainer filters;                        // filtering funcs
```   
我们可以为 _5_ 的倍数来增加一个过滤条件，像下面这样：  
```C++
  filters.emplace_back(                           // see Item 42 for
    [](int value) { return value % 5 == 0; }      // info on
  );                                              // emplace_back
```  
然而，可能的是我们需要在运行期间计算除法，即为：我们不能硬编码 _5_ 到 _lambda_ 中。所以添加的过滤条件看起来更像是下面这样：  
```C++
  void addDivisorFilter()
  {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                                   // danger!
      [&](int value) { return value % divisor == 0; }       // ref to
    );                                                      // divisor
  }                                                         // will
                                                            // dangle!
```   
这个代码会出错。_lambda_ 引用了局部变量 _divisor_，但是当 _addDivisorFilter_ 返回后，这个局部变量就不存在了。这是在 _filters.emplace_back_ 返回后立即发生的，所以那个添加到 _filters_ 中的函数在被添加后就消忙了。使用这个过滤条件几乎在它被创建的那一刻起就会产生 _undefined behavior_。

现在，如果 _divisor_ 的 _by-reference_ 捕获是显式的话，相同的问题也会是存在的，  
```C++
  filters.emplace_back(
    [&divisor](int value)                         // danger! ref to
    { return value % divisor == 0; }              // divisor will
  );                                              // still dangle!
```  
但是，使用显式捕获时，更容易看的出 _lambda_ 的有效性是依赖于 _divisor_ 的生命周期的。所写出的名字 _divisor_ 提醒我们要确保 _divisor_ 的生命周期至少和 _lambda_ 的 _closure_ 的生命周期要一样长。比起 _[&]_ 所表达的 **_确保没有任何悬空_** 的警告，显式写出名字会更让人印象深刻。

如果你知道某个 _closure_ 会被立即使用，比如：传递给 _STL_ 算法，并且这个 _closure_ 不会被拷贝的话，那么就不会有这个 _closure_ 所持有的引用会比那些在创建出它所对应的 _lambda_ 的环境中的局部变量和形参存在的更久的风险了。你可能会争辩说，没有悬空引用的风险，就不需要去避免默认 _by-reference_ 捕获模式了。例如，我们的过滤 _lambda_ 可以只被用来做为 _C++11_ 的 _std::all_of_ 的一个实参，而这个 _std::all_of_ 是被用来检查某范围内的所有元素是否都满足于同一个条件的：  
```C++
  template<typename C>
  void workWithContainer(const C& container)
  {
    auto calc1 = computeSomeValue1();             // as above
    auto calc2 = computeSomeValue2();             // as above

    auto divisor = computeDivisor(calc1, calc2);  // as above

    using ContElemT = typename C::value_type;     // type of
                                                  // elements in
                                                  // container

    using std::begin;                             // for
    using std::end;                               // genericity;
                                                  // see Item 13

    if (std::all_of(                              // if all values
      begin(container), end(container),           // in container
      [&](const ContElemT& value)                 // are multiples
      { return value % divisor == 0; })           // of divisor...
      ) {

      …                                           // they are...
    } else {
      …                                           // at least one
    }                                             // isn't...
  }
```  
这是没问题的，这是安全的。但是它的安全性有点不稳固。如果 _lambda_ 被发现在其他的上下文中是有用的，比如：做为一个将会被添加到 _filters_ _container_ 中的函数，然后这个 _lambda_ 又会被拷贝粘贴到一个它的 _closure_ 比 _divisor_ 是存在的更久的上下文中的话，那么你就又回到 _dangle-city_ 了，此时在捕获语句中不存在任何可以提醒你去对 _divisor_ 执行生命周期分析的内容了。

从长远来看，将 _lambda_ 所依赖的局部变量和形参显式列出来是更好的软件工程。

对了，_C++14_ 可以在 _lambda_ 的形参规范中使用 _auto_ 了，这意味着上面的代码可以在 _C++14_ 中变得更加简洁了。_ContElemT typedef_ 可以被淘汰掉了，然后 _if_ 条件语句可以被修改为下面这样：  
```C++
  if (std::all_of(begin(container), end(container),
                  [&](const auto& value)                    // C++14
                  { return value % divisor == 0; }))
```

解决我们 _divisor_ 问题的一个方法是使用默认 _by-value_ 捕获模式。也就是说，我们可以像下面这样将 _lambda_ 添加到 _filters_ 中：
```C++
  filters.emplace_back(                                     // now
    [=](int value) { return value % divisor == 0; }         // divisor
  );                                                        // can't
                                                            // dangle
```
这对于这个例子来说足够了，但是，通常来说，默认 _by-value_ 捕获模式并不是你想要的避免悬空的解药。问题在于：如果你按 _by-value_ 的形式捕获了一个指针，并将这个指针拷贝到了 _lambda_ 所生成的 _closure_ 中，但是你并不能阻止 _lambda_ 之外的代码去删除掉这个指针的话，那么也就不能阻止 _lambda_ 之外的代码去让你的拷贝悬空了。

你抗议说“这根本不可能发生”。“在读完 [_Chapter 4_](Chapter%204.md#chapter-4-智能指针) 后，我就非常敬佩智能指针了。只有失败的 _C++98_ 程序员才会使用原始指针和 _delete_。” 这可能是正确的，但和这儿说的不相关，因为你确实可能会使用到原始指针，并且确实可以删除这些原始指针。只不过在现代 _C++_ 编程风格中，很少有原始指针这样的代码。

假设 _Widget_ 可以做的事情之一是添加成员到 _filters_ 的 _container_ 中：  
```C++
  class Widget {
  public:
    …                                   // ctors, etc.
    void addFilter() const;             // add an entry to filters

  private:
    int divisor;                        // used in Widget's filter
  };
```  
_Widget::addFilter_ 可以被声明为下面这样：  
```C++
  void Widget::addFilter() const
  {
    filters.emplace_back(
      [=](int value) { return value % divisor == 0; }
    );
  }
```  
对于喜悦地外行来说，这个代码看起来是安全的。这个 _lambda_ 是依赖于 _divisor_ 的，但是，默认 _by-value_ 捕获模式确保了 _divisor_ 是被拷贝到这个 _lambda_ 所创建的 _closure_ 中的，对吧？

错。完全的错误。严重的错误。致命的错误。

只能捕获到那些在创建出 _lambda_ 的作用域中是可见的 _non-static_ 局部变量和形参。在函数体 _Widget::addFilter_ 中，_divisor_ 不是局部变量，它是 _Widget_ 类的数据成员。所以它是不能被捕获的。如果默认捕获模式是被取消的话，那么这个代码不可以通过编译：  
```C++
  void Widget::addFilter() const
  {
    filters.emplace_back(                                   // error!
        [](int value) { return value % divisor == 0; }      // divisor
    );                                                      // not
  }                                                         // available
```  
此外，如果试图显式捕获 _divisor_ 的话，_by-value_ 或 _by-reference_ 都无所谓，那么这个代码是不可以通过编译的，因为 _divisor_ 不是局部变量或形参：  
```C++
  void Widget::addFilter() const
  {
    filters.emplace_back(
        [divisor](int value)                        // error! no local
        { return value % divisor == 0; }            // divisor to capture
    );
  }
```
所以，如果默认 _by-value_ 捕获语句没有捕获 _divisor_，甚至都没有默认 _by-value_ 捕获语句的话，那么代码是不可以通过编译的，到底为什么呢？

是因为隐式使用了原始指针 _this_。每一个 _non-static_ 成员函数都有一个 _this_ 指针，每当使用类的数据成员时都会使用到 _this_ 指针。例如：在 _Widget_ 的任意成员函数中，编译器会在内部使用 _this->divisor_ 来代替 _divisor_。在 _Widget::addFilter_ 的默认 _by-value_ 捕获版本中：  
```C++
  void Widget::addFilter() const
  {
    filters.emplace_back(
      [=](int value) { return value % divisor == 0; }
    );
  }
```  
被捕获的是 _Widget_ 的 _this_ 指针，而不是 _divisor_。编译器将代码视为下面这样：
```C++
  void Widget::addFilter() const
  {
    auto currentObjectPtr = this;
    
    filters.emplace_back(
      [currentObjectPtr](int value)
      { return value % currentObjectPtr->divisor == 0; }
    );
  }
```  

理解了这个就等于理解了这个 _lambda_ 所生成的 _closure_ 的有效性和 _Widget_ 的生命周期是相关的，因为这个 _closure_ 包含了这个 _Widget_ 所对应的 _this_ 指针的副本。特别是，考虑这样的代码，依据 [_Chapter 4_](Chapter%204.md#chapter-4-智能指针) 的内容，只使用了智能指针:  
```C++
  using FilterContainer =                         // as before
    std::vector<std::function<bool(int)>>;

  FilterContainer filters;                        // as before

  void doSomeWork()
  {
    auto pw =                                     // create Widget; see
      std::make_unique<Widget>();                 // Item 21 for
                                                  // std::make_unique
    
    pw->addFilter();                              // add filter that uses
                                                  // Widget::divisor
    …
  }                                               // destroy Widget; filters
                                                  // now holds dangling pointer!
```  
当 _doSomeWork_ 被调用时，一个依赖于 _std::make_unique_ 所生成的 _Widget_ 的对象的过滤条件就被创建了，即为：这个过滤条件包含了指向 _std::make_unique_ 所生成的 _Widget_ 的对象的指针的副本，也就是所对应的 _Widget_ 的 _this_ 指针。这个过滤条件被添加到了 _filters_ 中，但是，当 _doSomeWork_ 结束时，这个 _Widget_ 就会被管理着它的生命周期的 _std::unique_ptr_ 所销毁掉了，见 [_Item 18_](Chapter%204.md#item-18-对于-exclusive-ownership-的资源管理使用-stdunique_ptr)。从此时开始，_filters_ 就包含了一个带有悬空指针的成员。

这个特定问题可以通过制作一个你想要捕获的数据成员的局部副本并捕获这个副本来解决：  
```C++
  void Widget::addFilter() const
  {
    auto divisorCopy = divisor;                   // copy data member

    filters.emplace_back(
      [divisorCopy](int value)                    // capture the copy
      { return value % divisorCopy == 0; }        // use the copy
    );
  }
```  
老实说，如果要使用这种方法的话，那么默认 _by-value_ 捕获也是可以的，  
```C++
  void Widget::addFilter() const
  {
    auto divisorCopy = divisor;                   // copy data member
    
    filters.emplace_back(
      [=](int value)                              // capture the copy
      { return value % divisorCopy == 0; }        // use the copy
    );
  }
```
但是为什么要冒此风险呢？默认捕获模式会使你在想要捕获 _divisor_ 时，却意外地捕获了 _this_。

在 _C++14_ 中，一个可以捕获数据成员的更好方法是使用 _generalized lambda_ 捕获，见 [_Item 32_](#item-32-使用初始化捕获来将对象移动到-closure-中)：  
```C++
  void Widget::addFilter() const
  {
    filters.emplace_back(                         // C++14:
      [divisor = divisor](int value)              // copy divisor to closure
      { return value % divisor == 0; }            // use the copy
    );
  }
```

对于 _generalized lambda_ 捕获来说，没有默认捕获模式，所以即使在 _C++14_ 中，本 _Item_ 的建议：避免默认捕获模式，仍然成立。

默认 _by-value_ 捕获的另外一个缺点是会让人误认为 _closure_ 是 _self-contained_ 的，这个 _closure_ 之外的数据的改动是不会影响到这个 _closure_ 本身的。并不是的，因为 _lambda_ 不仅依赖于可以被捕获的局部变量和形参，还依赖于静态存储期对象。这些对象可以是全局作用域中或 _namespace_ 作用域中所定义的对象，也可以是类中、函数中或者文件中所声明的 _static_ 对象。这些对象都可以在 _lambda_ 中被使用，但不可以被捕捉。然而，默认 _by-value_ 捕获的规范可能会给人一种这些对象也是可以被捕捉的印象。考虑对我们之前看到的 _addDivisorFilter_ 函数做个修改：  
```C++
  void addDivisorFilter()
  {
    static auto calc1 = computeSomeValue1();      // now static
    static auto calc2 = computeSomeValue2();      // now static
    
    static auto divisor =                         // now static
      computeDivisor(calc1, calc2);
    
    filters.emplace_back(
      [=](int value)                              // captures nothing!
      { return value % divisor == 0; }            // refers to above static
    );
    ++divisor;                                    // modify divisor
}
```

这个代码的马虎读者看到 _[]_ 会认为“是的，这个 _lambda_ 拷贝了它所使用的全部对象，因此是 _self-contained_ 的”，这是可以被原谅的。但其实并不是 _self-contained_ 的。这个 _lambda_ 没有使用任何 _non-static_ 局部对象，所以没有捕获任何东西。这个 _lambda_ 代码使用的是静态变量 _divisor_。在每个 _addDivisorFilter_ 执行结束后，当 _divisor_ 被增加时，任何通过这个函数被添加到 _filters_ 中的 _lambda_ 都会表现出新的行为，而这个新的行为和新的 _divisor_ 值是所相关的。实际来说，这个 _lambda_ 是按 _by-reference_ 的形式捕获的 _divisor_，这与默认 _by-value_ 捕获语句所暗示的含义是矛盾的。如果你一开始远离了默认 _by-value_ 捕获语句的话，那么也就消除了代码会被如此误导的风险了。

### 需要记住的规则

* 默认 _by-reference_ 捕获可能会导致悬空引用。
* 默认 _by-value_ 捕获容易受到悬空指针的影响，特别是 _this_，还会误导性地暗示所对应的 _lambda_ 是 _self-contained_ 的。

## _Item 32_ 使用初始化捕获来将对象移动到 _closure_ 中

有时候，_by-value_ 和 _by-reference_ 捕获都不是你想要的。如果你想要把一个 _move-only_ 对象，比如：_std::unique_ptr_ 或 _std::future_，放入到一个 _closure_ 中的话，那么 _C++11_ 没有提供可以完成的方法。如果你想要把一个拷贝是代价大的而移动是代价小的对象，比如：标准库中的大部分容器，放入到一个 _closure_ 中的话，那么你想要的是移动而不是拷贝。再一次，_C++11_ 没有提供可以完成的方法。

但只是 _C++11_ 没有提供可以完成的方法。在 _C++14_ 中就是不同的故事了。_C++14_ 提供了直接的方法来将对象移动到 _closure_ 中。如果你的编译器是 _C++14-compliant_ 的话，那么高兴地继续往下读吧。如果你仍然使用是 _C++11_ 的编译器的话，那么你也应该高兴地继续读下去，因为在 _C++11_ 中也有可以模拟移动捕获的方法。

在 _C++11_ 刚被接受时，缺少移动捕获被认为是一个缺点。简单的解决方法是在 _C++14_ 中添加移动捕获，但是 _Standardization Committee_ 却选择了一条不同的路。_Standardization Committee_ 引入了一个新的非常灵活的捕获机制，_capture-by-move_ 只是它可以执行的技术之一。这个新能力被称为初始化捕获。初始化捕获几乎可以做 _C++11_ 的捕获可以做的所有事情，甚至更多。你不可以使用初始化捕获完成的一件事是默认捕获模式，但是 [_Item 31_](#item-31-避免默认捕获模式) 解释了无论如何你都应该远离默认捕获模式。对于 _C++11_ 的捕获所覆盖的情况，初始化捕获的语法是有点冗长的，所以在 _C++11_ 的捕获就可以完成工作的场景下，选择使用 _C++11_ 的捕获而不是初始化捕获也是非常合理的。

使用初始化捕获能够让你去指定：  
* _lambda_ 所生成的 _closure class_ 中的数据成员的名字
* 初始化数据成员的表达式。

下面展示了如何可以使用初始化捕获来将 _std::unique_ptr_ 移动到 _closure_ 中：  
```C++
  class Widget {                                  // some useful type
  public:
    …

    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;

  private:
    …
  };

  auto pw = std::make_unique<Widget>();           // create Widget; see
                                                  // Item 21 for info on
                                                  // std::make_unique

  …                                               // configure *pw

  auto func = [pw = std::move(pw)]                // init data mbr
              { return pw->isValidated()          // in closure w/
                      && pw->isArchived(); };     // std::move(pw)
```

上面的代码包含了初始化捕获。_=_ 的左侧是你所指定的 _closure class_ 中的数据成员的名字，而 _=_ 的右侧则是初始化表达式。有趣地是，_=_ 的左侧的作用域和右侧的作用域是不相同的。左侧的作用域是 _closure class_。右侧的作用域和定义出 _lambda_ 的作用域是相同的。在上面的例子中，_=_ 的左侧的名字 _pw_ 指的是 _closure class_ 中的一个数据成员，而 _=_ 的右侧的名字 _pw_ 则指的是在那个 _lambda_ 的上面所声明的那个对象，即为：通过调用 _std::make_unique_ 所初始化的那个变量。所以 _pw = std::move(pw)_ 意味着：在 _closure_ 中创建了一个数据成员 _pw_，并使用将 _std::move_ 应用到局部变量 _pw_ 上所产生的结果来初始化这个所创建的数据成员 _pw_。

像往常一样，_lambda_ 中的代码是在 _closure class_ 的作用域中的，所以这里所使用的 _pw_ 指的是 _closure class_ 的数据成员。 

这个例子中的注释 _configure *pw_ 表明了：在 _Widget_ 被 _std::make_unique_ 创建之后，在 _Widget_ 所对应的 _std::unique_ptr_ 被 _lambda_ 捕获之前，这个 _Widget_ 会以某些方式被更改。如果这些配置不是必要的话，即为：如果 _std::make_unique_ 所创建的 _Widget_ 是处在一个适合直接被 _lambda_ 进行捕获的状态下的话，那么局部变量 _pw_ 不是必须要存在的，因为 _closure class_ 的数据成员可以直接被 _std::make_unique_ 来进行初始化：  
```C++
  auto func = [pw = std::make_unique<Widget>()]             // init data mbr
              { return pw->isValidated()                    // in closure w/
                        && pw->isArchived(); };             // result of call
                                                            // to make_unique
```  
应该清楚的是：_C++14_ 的 **_捕获_** 概念相比于 _C++11_ 有了非常大的泛化，因为在 _C++11_ 中，不能够捕获表达式的结果。因此，初始化捕获的另一个名字为 _generalized lambda_ 捕获。

但是，如果你使用的编译器没有支持 _C++14_ 的初始化捕获的话，那么如何可以在没有支持移动捕获的语言下完成移动捕获呢？

记住：_lambda expression_ 只是生成了一个类，并且创建了这个类的一个对象。不存在你可以使用 _lambda_ 做而不可以手动做的事情。例如：我们刚才看到的 _C++14_ 的代码的例子可以在 _C++11_ 下这样写：  
```C++
  class IsValAndArch {                                      // "is validated
  public:                                                   // and archived"
    using DataType = std::unique_ptr<Widget>;

    explicit IsValAndArch(DataType&& ptr)                   // Item 25 explains
    : pw(std::move(ptr)) {}                                 // use of std::move
    
    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }

  private:
    DataType pw;
  };

  auto func = IsValAndArch(std::make_unique<Widget>());
```   
比起写 _lambda_，这里做了更多的工作，但是并不会改变这样的事实：如果你在 _C++11_ 中想要一个支持对数据成员实施移动初始化的类的话，那么你去花点时间敲代码就完全可以完成的你的期望。

如果你想要坚持使用 _lambda_ 的话，鉴于 _lambda_ 的方便，你大概是会坚持的，那么在 _C++11_ 可以通过下面的方法来模拟移动捕获：  
* 将需要捕获的对象移动到 _std::bind_ 所产生的函数对象中 
* 给 _lambda_ 一个指向所 **_捕获_** 的对象的引用

如果你熟悉 _std::bind_ 的话，那么代码是非常简单的。如果你不熟悉 _std::bind_ 的话，那么代码就需要花点时间去习惯，但是是值得的。

假设你想要的是：创建一个局部 _std::vector_，然后将一组合适的值放到这个 _std::vector_ 中，最后将这个 _std::vector_ 移动到一个 _closure_ 中。在 _C++14_ 中，这是很容易的：  
```C++
  std::vector<double> data;                       // object to be moved
                                                  // into closure

  …                                               // populate data

  auto func = [data = std::move(data)]            // C++14 init capture
    { /* uses of data */ };
```

这个代码的关键部分是：想要移动的对象的类型 _std::vector&lt;double&gt;_、对象的名字 _data_ 和用于初始化捕获的初始化表达式  _std::move(data)_。_C++11_ 所对应的代码是像下面这样，其中有关键的部分：  
```C++
  std::vector<double> data;                       // as above

  …                                               // as above

  auto func =
    std::bind(                                    // C++11 emulation
      [](const std::vector<double>& data)         // of init capture
      { /* uses of data */ },
      std::move(data)
  );
```  
像 _lambda expression_ 一样，_std::bind_ 也会生成函数对象，我将 _std::bind_ 所返回的是函数对象称为 _bind_ 对象。_std::bind_ 的第一个实参是可调用对象。后续的实参表示的是所要传递给这个可调用对象的值。

_bind_ 对象包含了所有传递给 _std::bind_ 的实参的副本。对于每个左值实参，在 _bind_ 对象中的所对应的对象都是被拷贝构造的。对于每个右值实参，在 _bind_ 对象中的所对应的对象都是被移动构造的。在这个例子中，第二个实参是一个右值，是 _std::move_ 的结果，见 [_Item 23_](Chapter%205.md#item-23-理解-stdmove-和-stdforward)，所以 _data_ 是被移动构造至 _bind_ 对象中的，这个移动构造是模拟移动捕获的关键，因为将右值移动至一个 _bind_ 对象中是我们解决无法将右值移动至 _C++11_ 的 _closure_ 中的方法。 

当 _bind_ 对象被 **_调用_** 时，即为：_bind_ 对象的 _function call operator_ 被执行时，_bind_ 对象所存储的实参会被传递给最初所传递给 _std::bind_ 的那个可调用对象了。在这个例子中，这也意味着：当 _bind_ 对象 _func_ 被调用时，在 _func_ 中所移动构造出的 _data_ 的副本会被传递给所传递给 _std::bind_ 的那个 _lambda_。

除了形参 _data_ 被添加到了所对应的 _pseudo-move-captured_ 对象以外，这个 _lambda_ 和我们在 _C++14_ 中所使用的 _lambda_ 是相同的。这个形参是指向 _bind_ 对象中的 _data_ 的副本的左值引用。这个形参不是右值引用，因为尽管被用来初始化 _data_ 的副本的表达式 _std::move(data)_ 是右值，但是这个 _data_ 的副本却是个左值。因此在 _lambda_ 中所使用的 _data_ 就是 _bind_ 对象中的所移动构造出的 _data_ 的副本了。

默认情况下，由 _lambda_ 所创建的 _closure class_ 中的 _operator()_ 成员函数是 _const_ 的。这将会导致 _closure_ 中的所有数据成员在 _lambda_ 中都是 _const_ 的。然而在 _bind_ 对象中的所移动构造出的 _data_ 的副本却不是 _const_ 的，因此，为了避免 _data_ 的副本在 _lambda_ 中被更改，_lambda_ 的形参应该被声明为 _const &_。如果 _lambda_ 被声明为了 _mutable_ 的话，那么由 _lambda_ 所创建的 _closure class_ 中的 _operator()_ 成员函数将不会再是 _const_ 的了 ，所以将 _lambda_ 的形参声明中的 _const_ 省略了也就是很合适的了：  
```C++
  auto func =
    std::bind(                                    // C++11 emulation
      [](std::vector<double>& data) mutable       // of init capture
      { /* uses of data */ },                     // for mutable lambda
      std::move(data)
  );
```
因为 _bind_ 对象存储了所传递给 _std::bind_ 的所有实参的拷贝，所以在我们的例子中的那个 _bind_ 对象也就包含了由 _lambda_ 所生成的 _closure_ 的副本，而这个 _lambda_ 是 _bind_ 对象的第一个实参。因此这个 _closure_ 的生命周期和这个 _bind_ 对象的生命周期是相同的。这是重要的，因为这意味着：只要这个 _closure_ 存在，包含着 _pseudo-move-captured_ 对象的 _bind_ 对象也会存在。

如果这是你第一次接触 _std::bind_ 的话，那么你需要在全面理解前面论述的细节前先去咨询你最爱的 _C++11_ 资料。即使如此，也应该清楚这些基础知识点：  
* 不能够将对象移动构造至 _C++11_ 的 _closure_ 中，但是能够将对象移动构造至 _C++11_ 的 _bind_ 对象中。
* 将对象移动构造至 _bind_ 对象中，然后再将所移动构造出的对象按 _by-reference_ 的形式传递给 _lambda_，这模拟了 _C++11_ 中的移动捕获。
* 因为 _bind_ 对象的生命周期和 _closure_ 的生命周期是一样的，所以可以认为 _bind_ 对象中的对象是在 _closure_ 中的。  

像第二个使用 _std::bind_ 来模拟移动捕获的例子一样，下面就是我们之前看到过的在  _closure_ 中创建 _std::unique_ptr_ 的代码：  
```C++
  auto func = [pw = std::make_unique<Widget>()]             // as before,
              { return pw->isValidated()                    // create pw
                        && pw->isArchived(); };             // in closure 
```  
下面是 _C++11_ 的模拟版本：
```C++
  auto func = std::bind(
              [](const std::unique_ptr<Widget>& pw)
              { return pw->isValidated()
                    && pw->isArchived(); },
                    std::make_unique<Widget>()
              );
```

我正在展示如何使用 _std::bind_ 来解决 _C++11_ 的 _lambda_ 的限制，这是讽刺的，因为在 [_Item 34_](#item-34-首选-lambda-而不是-stdbind) 中，我建议使用 _lambda_ 而不是 _std::bind_。然而，[_Item 34_](#item-34-首选-lambda-而不是-stdbind) 也解释了在 _C++11_ 中存在一些 _std::bind_ 可以被使用的场景，这个就是其中之一。在 _C++14_ 中，像初始化捕获和 _auto_ 形参这样的特性就消除这些场景。

### 需要记住的规则

* 使用 _C++14_ 的初始化捕获可以将对象移动至 _closure_ 中。
* 在 _C++11_ 中，可以通过手写类或者 _std::bind_ 来模拟初始化捕获。 

## _Item 33_ 在 _auto&&_ 形参上使用 _decltype_ 来进行完美转发

_C++14_ 中最令人激动的其中一个特性是 _generic lambda_，也就是可以在 _lambda_ 的形参规范上使用了 _auto_ 了。这个特性的实现是简单的：_lambda_ 的 _closure class_ 中的 _operator()_ 是模板函数。例如：给定一个这样的 _lambda_，  
```C++
auto f = [](auto x){ return func(normalize(x)); };
```
这个 _closure class_ 的 _function call operator_ 看起来像是这样：  
```C++
class SomeCompilerGeneratedClassName {
public:
  template<typename T>                            // see Item 3 for
  auto operator()(T x) const                      // auto return type
  { return func(normalize(x)); }

  …                                               // other closure class
};                                                // functionality  
```  
在这个例子中，_lambda_ 对于它的形参所做的唯一一件事就是将它转发到 _normalize_ 中。如果 _normalize_ 对左值和右值区别对待的话，那么这个 _lambda_ 就是错误的了，因为它总是将左值转递给了 _normalize_，形参 _x_ 肯定是左值，即使所传递给 _lambda_ 的实参是一个右值。

正确的方法是完美转发 _x_ 到 _normalize_ 中。完成这个需要对代码更改两个地方。首先，_x_ 必须变为 _universal reference_，见 [_Item 24_](Chapter%205.md#item-24-区分-universal-reference-和右值引用)，其次，必须通过 _std::forward_ 来将 _x_ 传递给 _normalize_，见 [_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference)。概念上来说，这些都是很小的改变：  
```C++
  auto f = [](auto&& x)
            { return func(normalize(std::forward<???>(x))); };
```

然而，概念和实现之间的问题是应该传递什么类型给 _std::forward_，即为：在上面写 _???_ 的地方应该写什么呢？    

一般来说，当使用完美转发时，你就是在持有类型形参 _T_ 的模板函数中，所以只需要写 _std::forward&lt;T&gt;_ 就可以了。但是在 _generic lambda_ 中，是没有可用的类型形参 _T_ 给到你的。在 _lambda_ 所生成的 _closure class_ 中的模板函数 _operator()_ 中是存在有 _T_ 的，但是不可以在 _lambda_ 中引用这个 _T_，所以没什么用。 

[_Item 28_](Chapter%205.md#item-28-理解引用折叠) 解释了：如果一个左值实参被传递给了一个 _universal reference_ 形参的话，那么这个形参的类型就会变为左值引用。如果一个右值实参被传递给了一个 _universal reference_ 形参的话，那么这个形参的类型就会变为右值引用。这意味着：在我们的 _lambda_ 中，我们可以通过检查形参 _x_ 的类型来确定所传递的实参是左值还是右值。_decltype_ 给了我们一个可以完成的方法。如果传入的是左值的话，那么 _decltype(x)_ 将会产生的是左值引用类型。如果传入的是右值的话，那么 _decltype(x)_ 将会产生的是右值引用类型。

[_Item 28_](Chapter%205.md#item-28-理解引用折叠) 也解释了：当调用 _std::forward_ 时，惯例规定了：类型实参为左值引用以表明返回的是左值，类型实参为 _non-reference_ 以表明返回的是右值。在我们的 _lambda_ 中，如果 _x_ 绑定的是左值的话，_decltype(x)_ 将会产生的是左值引用。这是符合惯例的。然而，如果 _x_ 绑定的是右值的话，那么 _decltype(x)_ 将会产生是右值引用，而不是符合惯例的 _non-reference_。

但是，看一看 [_Item 28_](Chapter%205.md#item-28-理解引用折叠) 中的 _std::forward_ 的 _C++14_ 的简单实现：  
```C++
  template<typename T>                            // in namespace
  T&& forward(remove_reference_t<T>& param)       // std
  {
    return static_cast<T&&>(param);
  }
```

如果客户代码想要完美转发一个 _Widget_ 类型的右值的话，那么通常使用类型 _Widget_，即为：_non-reference_，来实例化 _std::forward_，那么相应地 _std::forward_ 模板会产生下面这样的函数：  
```C++
Widget&& forward(Widget& param)                   // instantiation of
{                                                 // std::forward when
  return static_cast<Widget&&>(param);            // T is Widget
}
```   
但是考虑一下：如果客户代码想要去完美转发 _Widget_ 类型的右值，但是没有遵循惯例将 _T_ 指明为 _non-reference_ 类型，而是将 _T_ 指明为了右值引用的话，会发生什么呢？也就是说考虑一下：如果 _T_ 是被指明为了 _Widget&&_ 的话，会发生什么呢？在执行 _std::forward_ 的实例化和应用了 _std::remove_reference_t_ 后，在引用折叠之前，再一次见 [_Item 28_](Chapter%205.md#item-28-理解引用折叠)，_std::forward_ 看起来就像下面这样：  
```C++
  Widget&& && forward(Widget& param)              // instantiation of
  {                                               // std::forward when
    return static_cast<Widget&& &&>(param);       // T is Widget&&
  }                                               // (before reference-
                                                  // collapsing)
```  
应用引用折叠规则，右值引用的右值引用会变为右值引用，这个实例化会变为下面这样：  
```C++
  Widget&& forward(Widget& param)                 // instantiation of
  {                                               // std::forward when
    return static_cast<Widget&&>(param);          // T is Widget&&
  }                                               // (after reference-
                                                  // collapsing)
```  
如果将这个实例化和将 _T_ 设置为 _Widget_ 来调用 _std::forward_ 时所产生的实例化进行比较的话，那么你会发现它们两个是一样的。这意味着使用右值引用类型实例化 _std::forward_ 所产生的结果和使用 _non-reference_ 类型实例化 _std::forward_ 所产生的结果是一样的。

这是个非常好的消息，因为当一个右值被传递来做为我们的 _lambda_ 的形参 _x_ 的实参时，_decltype(x)_ 产生的是右值引用。我们在上面就已经知道了：当一个左值被传递给到我们的 _lambda_ 时，_decltype(x)_ 所产生的是符合惯例的可以传递给 _std::forward_ 的类型，我们现在也已经知道了：对于右值来说，_decltype(x)_ 所产生的类型是不符合惯例的可以传递给 _std::forward_ 的类型，但是却和符合惯例的类型产生了相同的结果。所以，对于左值和右值来说，传递 _decltype(x)_ 到 _std::forward_ 都给了我们想要的结果。因此，我们的完美转发 _lambda_ 可以写成下面这样：  
```C++
  auto f =
    [](auto&& param)
    {
      return
        func(normalize(std::forward<decltype(param)>(param)));
    };
```
根据这个，稍作修改，就可以得到接收任意数量形参的 _lambda_，因为 _C++14_ 的 _lambda_ 也可以是 _variadic_：    
```C++
  auto f =
    [](auto&&... params)
    {
      return
      func(normalize(std::forward<decltype(params)>(params)...));
    };
```

### 需要记住的规则

* 在 _auto&&_ 形参上使用 _decltype_ 来进行完美转发。

## _Item 34_ 首选 _lambda_ 而不是 _std::bind_

_std::bind_ 是 _C++11_ 对 _C++98_ 的 _std::bind1st_ 和 _std::bind2nd_ 的继承，但不正式地说，从 _2005_ 年开始 _std::bind_ 就已经是标准库的一部分了。就是说当 _Standardization Committee_ 采纳了被称为 _TR1_ 的包含了 _bind_ 的规范的文档的时候，_std::bind_ 就已经是标准库的一部分了。在 _TR1_ 中，_bind_ 是在一个不同的 _namespace_ 中的，所以应该是 _std::tr1::bind_ 而不是 _std::bind_，还有少数接口细节是不同的。这个历史意味着有一些程序员已经有十多年使用 _std::bind_ 的经验了。如果你也是其中一员的话，那么你可能要不情愿放弃这个对于你来说是非常好的工具。这是可以理解的，但是在这个场景中，改变是好的，因为，在 _C++11_ 中，相对于 _std::bind_ 来说，_lambda_ 几乎总是更好的选择。因为，从 _C++14_ 开始，_lambda_ 的场景不再仅是强壮的了，而是完全坚不可摧的了。

本 _Item_ 假设你是熟悉 _std::bind_ 的。如果你是不熟悉的话，那么在开始前你需要对 _std::bind_ 有基本的了解。这个了解在任何场景下都是值得的，因为你永远不知道何时会在你必须读或者维护的代码中遇到 _std::bind_。

就像在 [_Item 32_](#item-32-使用初始化捕获来将对象移动到-closure-中) 中那样，我将 _std::bind_ 返回的函数对象来称为 _bind_ 对象。

首选 _lambda_ 而不是 _std::bind_ 的最重要的原因是 _lambda_ 的可读性更强。例如：假设我们有一个函数来配置声响警报：  
```C++
// typedef for a point in time (see Item 9 for syntax)
using Time = std::chrono::steady_clock::time_point;

// see Item 10 for "enum class"
enum class Sound { Beep, Siren, Whistle };

// typedef for a length of time
using Duration = std::chrono::steady_clock::duration;

// at time t, make sound s for duration d
void setAlarm(Time t, Sound s, Duration d);
```  
进一步假设：在程序中的一些时刻，我们确定我们想要一个在警报，这个警报会在被设置一个小时后发出声响，并且持续 _30_ 秒。但是，警报声音还没有决定。我们可以写一个修改了 _setAlarm_ 的接口的 _lambda_，所以只有一个声音需要去被指定：  
```C++
// setSoundL ("L" for "lambda") is a function object allowing a
// sound to be specified for a 30-sec alarm to go off an hour
// after it's set
auto setSoundL = 
  [](Sound s)
  {
    // make std::chrono components available w/o qualification
    using namespace std::chrono;
    
    setAlarm(steady_clock::now() + hours(1),      // alarm to go off
    s,                                            // in an hour for
    seconds(30));                                 // 30 seconds
};
```
上面的代码中的 _lambda_ 中有 _setAlarm_ 调用。这只是一个普通的函数调用，甚至只有一点 _lambda_ 使用经验的读者都可以看出来，所传递给 _lambda_ 的形参 _s_ 是被传递来做为 _setAlarm_ 的实参的。

我们可以在 _C++14_ 中通过利用秒 _s_、毫秒 _ms_ 和小时 _h_ 等标准后缀来精简代码，这是建立在 _C++11_ 的用户定义的 _literal_ 的支持上的。这些后缀是在 _std::literals_ 的 _namespace_ 中被实现的，所以上面的代码可以被写为下面这样：  
```C++
  auto setSoundL = 
    [](Sound s)
    {
      using namespace std::chrono;
      using namespace std::literals;          // for C++14 suffixes

      setAlarm(steady_clock::now() + 1h,      // C++14, but
      s,                                      // same meaning
      30s);                                   // as above
    };
```  

我们第一次尝试写相应的 _std::bind_ 调用是下面这样的。它有一个错误，稍后再解决，但是正确的代码是更复杂的，这个简单版本带来一些重要的议题：  
```C++
  using namespace std::chrono;                    // as above
  using namespace std::literals;

  using namespace std::placeholders;              // needed for use of "_1"

  auto setSoundB =                                // "B" for "bind"
    std::bind(setAlarm,
    steady_clock::now() + 1h,                     // incorrect! see below
    _1,
    30s);

```  
上面的代码中的 _std::bind_ 中有 _setAlarm_ 调用，这个代码的读者只需要知道调用 _setSoundB_ 会使用在 _std::bind_ 调用中所指定的时间和持续时间来执行 _setAlarm_。对于不熟悉的人来说，占位符 __1_ 本质是一种魔法，但是，甚至是熟悉的人也必须在心中将这个占位符中的数字映射到它在 _std::bind_ 的形参列表上的位置上，这样才能理解在调用 _setSoundB_ 时所传递的第一个实参是做为 _setAlarm_ 的第二个实参的。这个实参的类型不会在调用 _std::bind_ 时被识别，所以，读者必须去查询 _setAlarm_ 的声明去确定什么类型的实参会传递给 _setSoundB_。

但是正如我说的，这个代码不是完全正确的。在 _lambda_ 中，表达式 _steady_clock::now() + 1h_ 是 _setAlarm_ 的实参。当 _setAlarm_ 被调用时，这个实参会被求值。这是合理的：我们想在执行 _setAlarm_ 的一小时后警报发出声响。然而，在 _std::bind_ 调用中，_steady_clock::now() + 1h_ 是被传递来做为 _std::bind_ 的一个实参的，而不是 _setAlarm_ 的实参。这意味着：当 _std::bind_ 被调用时，这个表达式 _steady_clock::now() + 1h_ 就会被求值，这个表达式所生成的结果会被存储到所生成的 _bind_ 对象中。因此，这个警报会在执行 _std::bind_ 的一个小时后发出声响，而不是正确的在执行 _setAlarm_ 的一个小时后发出声响！

修复这个问题需要告诉 _std::bind_ 直到 _setAlarm_ 被调用后再去对这个表达式求值，完成的方法是在第一个 _std::bind_ 中嵌套第二个 _std::bind_ 调用：  
```C++
  auto setSoundB =
    std::bind(setAlarm,
              std::bind(std::plus<>(), steady_clock::now(), 1h),
              _1,
              30s);
```

如果你熟悉 _C++98_ 中的 _std::plus_ 模板的话，那么你可能会对这个代码感到惊讶，没有在 _<>_ 中指定类型，即为：包含的是 _std::plus<>_，而不是 _std::plus&lt;type&gt;_。在 _C++14_ 中，标准操作符模板的模板类型实参通常可以被省略，所以，不需要在这里进行提供。_C++11_ 没有提供这些特性，所以 _C++11_ 中与 _lambda_ 等同的 _std::bind_ 是下面这样的：  
```C++
  using namespace std::chrono; // as above
  using namespace std::placeholders;
    auto setSoundB =
      std::bind(setAlarm,
                std::bind(std::plus<steady_clock::time_point>(),
                            steady_clock::now(),
                            hours(1)),
                _1,
                seconds(30));
```  
如果此时 _lambda_ 看起来还是没有吸引力的话，那么你应该检查你的视力了。

当 _setAlarm_ 被重载时，就会产生一个新问题。假设有一个重载函数持有第四个形参，用来指明声响音量：  
```C++
  enum class Volume { Normal, Loud, LoudPlusPlus };

  void setAlarm(Time t, Sound s, Duration d, Volume v); 
```  

_lambda_ 仍然可以像之前一样继续工作，因为重载决议会选择 _setAlarm_ 的三个实参版本：  
```C++
  auto setSoundL =                                // same as before                
    [](Sound s)
    {
      using namespace std::chrono;

      setAlarm(steady_clock::now() + 1h,          // fine, calls      
                s,                                // 3-arg version
                30s);                             // of setAlarm                      
    };
```
另一方面，现在 _std::bind_ 调用则不可以通过编译：  
```C++
  auto setSoundB =                                // error! which            
    std::bind(setAlarm,                           // setAlarm?
                std::bind(std::plus<>(),
                          steady_clock::now(),
                          1h),
                _1,
                30s);
```  

存在的问题是编译器无法确定应该传递给 _std::bind_ 哪一个 _setAlarm_ 函数。编译器只有一个函数名，但只有函数名是有歧义的。

为了让 _std::bind_ 调用可以编译，_setAlarm_ 必须被转换为合适的函数指针类型：
```C++
  using SetAlarm3ParamType = void(*)(Time t, Sound s, Duration d);

  auto setSoundB =                                          // now
    std::bind(static_cast<SetAlarm3ParamType>(setAlarm),    // okay
              std::bind(std::plus<>(),
                        steady_clock::now(),
                        1h),
              _1,
              30s);
```  
但是这就又带来了另一个 _lambda_ 和 _std::bind_ 的不同之处。在 _setSoundL_ 的 _function call operator_ 中，即为：在 _lambda_ 的 _closure class_ 中的 _function call operator_ 中，调用 _setAlarm_ 是在调用一个普通的且被编译器按照常用方式而内联的函数：  
```C++
setSoundL(Sound::Siren);                // body of setAlarm may
                                        // well be inlined here
```
然而，_std::bind_ 调用传递的是一个指向 _setAlarm_ 的函数指针，这意味着：在 _setSoundB_ 的 _function call operator_ 中，即为：在 _bind_ 对象的 _function call operator_ 中，调用 _setAlarm_ 是在调用一个函数指针。编译器很少可能会通过函数指针来内联函数调用，这意味着：相比于通过 _setSoundL_ 来调用 _setAlarm_ 来说，通过 _setSoundB_ 来调用 _setAlarm_ 很少有可能是完全内联的：  
```C++
  setSoundB(Sound::Siren);              // body of setAlarm is less
                                        // likely to be inlined here
```

因此，使用 _lambda_ 所生成的代码可能是比使用 _std::bind_ 所生成的代码要快的。

_setAlarm_ 的例子只是执行了一个简单的函数调用。如果你想要完成的是更复杂的事情的话，那么天平会更倾斜于 _lambda_。例如：考虑这样的一个 _C++14_ 的 _lambda_，这个 _lambda_ 会检查它的实参是否是在最小值 _lowVal_ 和最大值 _highVal_ 之间的，其中的 _lowVal_ 和 _highVal_ 是局部变量：  
```C++
  auto betweenL =
    [lowVal, highVal]
    (const auto& val)                             // C++14
    { return lowVal <= val && val <= highVal; };
```

_std::bind_ 可以表示相同的事情，但是需要使用晦涩的方式来构造代码：  
```C++
  using namespace std::placeholders;                        // as above
    
  auto betweenB =
    std::bind(std::logical_and<>(),                         // C++14
              std::bind(std::less_equal<>(), lowVal, _1),
              std::bind(std::less_equal<>(), _1, highVal));
```  
在 _C++11_ 中，我们必须要指明我们想要去比较的类型，那么 _std::bind_ 调用看起来会是下面这样：  
```C++
  auto betweenB =                                           // C++11 version
    std::bind(std::logical_and<bool>(),
              std::bind(std::less_equal<int>(), lowVal, _1),
              std::bind(std::less_equal<int>(), _1, highVal));
```
当然，在 _C++11_ 中，_lambda_ 也不能持有 _auto_ 形参，所以也必须要提交一个类型：  
```C++
  auto betweenL =                                           // C++11 version
    [lowVal, highVal]
    (int val)
    { return lowVal <= val && val <= highVal; };
```  
不管怎样，我希望我们可以同意：_lambda_ 版本不仅是简洁的，而且还是容易理解的和维护的。

之前我就说过，对于有 _std::bind_ 使用经验的人来说，_std::bind_ 的占位符，比如：__1_，__2_ 等，本质上就是魔法。但是不只是因为占位符的行为是不透明的。假定我们有一个函数来创建 _Widget_ 的压缩副本：  
```C++
  enum class CompLevel { Low, Normal, High };     // compression
                                                  // level

  Widget compress(const Widget& w,                // make compressed
                  CompLevel lev);                 // copy of w
```                                                    
我们想要创建一个函数对象，它允许我们指明一个指定的 _Widget w_ 的压缩等级。使用 _std::bind_ 将会创建这样的一个对象：  
```C++
  Widget w;

  using namespace std::placeholders;

  auto compressRateB = std::bind(compress, w, _1);   
```

现在，当我们传递 _w_ 到 _std::bind_ 时，_w_ 必须要被保存起来，以供后续的 _compare_ 所使用。_w_ 是被保存在对象 _compressRateB_ 中的，但是应该如何保存呢？_by-value_ 还是 _by-reference_ 呢？这是不同的，因为，如果 _w_ 会在调用 _std::bind_ 和 _compressRateB_ 之间被更改的话，那么按 _by-reference_ 的形式所保存的 _w_ 也会随之改变，而按 _by-value_ 的形式所保存的 _w_ 则不会有任何改变。  

是按 _by-value_ 的形式所保存的，只有一个方法可以知道这个答案，那就是去记住 _std::bind_ 是如何工作的，在 _std::bind_ 调用中没有任何的迹象。而在 _lambda_ 中 _w_ 是 _by-value_ 捕获还是 _by-reference_ 捕获则是显式指明的：  
```C++
  auto compressRateL =                            // w is captured by     
    [w](CompLevel lev)                            // value; lev is
    { return compress(w, lev); };                 // passed by value
```  
同样显式指明的还有形参是如何传递给的 _lambda_ 的。此处，形参 _lev_ 是按 _by-value_ 的形式所传递的，这是非常清晰的。因此：  
```C++
compressRateL(CompLevel::High);                   // arg is passed
                                                  // by value
```   

但是在 _std::bind_ 所生成的对象的调用中，实参是如何传递的呢？

再一次，只有一个方法可以知道这个答案，那就是去记住 _std::bind_ 是如何工作的。答案是所有传递给 _bind_ 对象的实参都是按 _by-reference_ 的形式所传递的，因为这些对象的 _function call operator_ 使用的是完美转发。

相比于 _lambda_，使用了 _std::bind_ 的代码可读性差、表达性差并且可能效率性也差。在 _C++14_ 中，没有 _std::bind_ 的合理使用场景。然而，在 _C++11_ 中，_std::bind_ 却可以在下面两个受限制的环境下使用：  

* 移动捕获。_C++11_ 的 _lambda_ 不支持移动捕获，但是可以通过结合 _lambda_ 和 _std::bind_ 来模拟移动捕获。对于细节，请查阅 [_Item 32_](#item-32-使用初始化捕获来将对象移动到-closure-中)，[_Item 32_](#item-32-使用初始化捕获来将对象移动到-closure-中) 也解释了：_C++14_ 的 _lambda_ 的初始化捕获减少了这种模拟的需求。
* _polymorphic function object_。因为 _bind_ 对象的 _function call operator_ 使用的是完美转发，所以可以接受任意类型的实参，除了 [_Item 30_](Chapter%205.md#item-30-熟悉完美转发失败的场景) 所描述的完美转发的限制。当你想要绑定一个使用了模板化的 _function call operator_ 的对象时，这是非常有用的。比如，给定这样的一个类，  
```C++
  class PolyWidget {
  public:
    template<typename T>
    void operator()(const T& param);
    …
  };
```  
_std::bind_ 可以下面这样来绑定 _PolyWidget_：  
```C++
  PolyWidget pw;

  auto boundPW = std::bind(pw, _1);
```
可以使用三种不同类型的实参来调用 _boundPW_：  
```C++
  boundPW(1930);                        // pass int to
                                        // PolyWidget::operator()
  
  boundPW(nullptr);                     // pass nullptr to
                                        // PolyWidget::operator()

  boundPW("Rosebud");                   // pass string literal to
                                        // PolyWidget::operator()
```

_C++11_ 的 _lambda_ 是无法完成这个的。然而，在 _C++14_ 中，可以通过使用带有 _auto_ 形参的 _lambda_ 来轻松完成：  
```C++
  auto boundPW = [pw](const auto& param)          // C++14   
                  { pw(param); };
```
当然，这些都是边缘场景，而且是暂时的边缘情况，因为支持 _C++14_ 的 _lambda_ 的编译器在日渐普及。

当 _bind_ 在 _2005_ 年被非正式地添加到 _C++_ 时，是对 _C++98_ 的一个巨大提升。然而 _C++11_ 的 _lambda_ 使得 _std::bind_ 几乎过时，而 _C++14_ 的 _lambda_ 则使得 _std::bind_ 没有任何使用场景了。

### 需要记住的规则

* _lambda_ 比起 _std::bind_ 是更易读的、更易表达的并且效率可能也是更高的。
* 只有在 _C++11_ 中，_std::bind_ 对于实现移动捕获或者绑定带有模板化的 _function call operator_ 的对象来说还算是有用的。 