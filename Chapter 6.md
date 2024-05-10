# _chapter 6_ _lambda expression_

 _lambda expression_ _lambda_ 是 _C++_ 编程中的游戏变革者。这有点让人惊讶，因为 _lambda_ 并没有给语言带来新的表达能力。所有 _lambda_ 可以完成的事情你都可以手动来完成，只不过是要多些敲代码而已。但是 _lambda_ 可以非常方便的创建函数对象，这对于日常 _C++_ 软件开发的影响是巨大的。当没有 _lambda_ 时，_STL_ 的 _if_ 算法，比如：_std::find_if_、_std::remove_if_、_std::count_if_ 等，通常只能使用最简单的 _predicate_ 来进行调用，但是当有了 _lambda_ 时，就能使用复杂的 _predicate_ 来进行调用了。这种情况也发生在可以自定义 _comparison function_ 的算法中，比如 _std::sort_、_std::nth_element_、_std::lower_bound_ 等。在 _STL_ 之外，_lambda_ 可以快速地来为 _std::unique_ptr_ 和 _std::shared_ptr_ 来创建 _custom deleter_，见 [_Item 18_](Chapter%204.md#item-18-对于-exclusive-ownership-的资源管理使用-stdunique_ptr) 和 [_Item 19_](Chapter%204.md#item-19-对于-shared-ownership-的资源管理使用-stdshared_ptr)，也让线程 _API_ 中的条件变量的 _predicate_ 的 _specification_ 变得简单了，见 [_Item 39_](Chapter%207.md#item-39-对于-one-shot-event-通信考虑-void-future)。在标准库之外，_lambda expression_ 方便了回调函数、接口适配函数和单次调用的特定上下文函数的  _on-the-fly_ _specification_。_lambda_ 确实让 _C++_ 成为了更友好的编程语言。

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

顺便一说，_C++14_ 可以在 _lambda_ 的形参 _specification_ 中使用 _auto_ 了，这意味着上面的代码可以在 _C++14_ 中变得更加简洁了。可以淘汰掉 _ContElemT typedef_，然后 _if_ 条件语句可以被修改为下面这样：  
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

默认 _by-value_ 捕获的另外一个缺点是会让人误认为所对应的 _closure_ 是 _self-contained_ 的，这个 _closure_ 之外的数据的改动是不会影响到这个 _closure_ 本身的。通常来说并不是的，因为 _lambda_ 不仅依赖于可以被捕获的局部变量和形参，还依赖于静态存储期对象。这些对象可以是全局作用域中或 _namespace_ 作用域中所定义的对象，也可以是类中、函数中或者文件中所声明的 _static_ 对象。这些对象都可以在 _lambda_ 中被使用，但不可以被捕捉。然而，默认 _by-value_ 捕获的 _specification_ 可能会给人一种这些对象也是可以被捕捉的印象。考虑对我们之前看到的 _addDivisorFilter_ 函数做个修改：  
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

### 需要记住的规则：

* 默认 _by-reference_ 捕获可能会导致悬空引用。
* 默认 _by-value_ 捕获容易受到悬空指针的影响，特别是 _this_，还会误导性地暗示所对应的 _lambda_ 是 _self-contained_ 的。

## _Item 32_ 使用初始化捕获来将对象移动到 _closure_ 中

## _Item 33_ 在 _auto&&_ 形参上使用 _decltype_ 来进行完美转发

## _Item 34_ 首选 _lambda_ 而不是 _std::bind_