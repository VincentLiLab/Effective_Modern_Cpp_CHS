- [Chapter 2 auto](#chapter-2-auto)
  - [Item 5 首选 _auto_ 而不是显式类型声明](#item-5-首选-auto-而不是显式类型声明)
    - [需要记住的规则](#需要记住的规则)
  - [Item 6 当 _auto_ 推导出的类型是 _undesired_ 时，使用 _the explicitly typed initializer idiom_](#item-6-当-auto-推导出的类型是-undesired-时使用-the-explicitly-typed-initializer-idiom)
    - [需要记住的规则：](#需要记住的规则-1)

# Chapter 2 auto

在概念上，_auto_ 是非常简单的，但是实际上它要比看起来隐晦的多。使用 _auto_  可以少写一些代码，当然使用 _auto_  
也可以避免手动声明类型所带来的正确性问题和效率问题_。此外，一些 _auto_ 的类型推导结果虽然是完全地遵守规  
定的算法的，但是站在程序员的角度看却是错误的。当处在这种场景时，重要的是知道如何引导 _auto_ 去得到正确  
的答案，因为通常需要避免回退使用手动类型声明。

这个简短的章节涵盖了 _auto_ 的所有。

## Item 5 首选 _auto_ 而不是显式类型声明

啊，多么简单有趣的一句  
```C++
  int x;
``` 
等等，糟糕。我忘记初始化 _x_ 了，所以它的值是不确定的。它也有可能会被初始化为 _0_。这取决于环境。哎。

不要紧。让我们再来看一个简单有趣的例子：声明一个局部变量，并解引用一个 _iterator_ 来初始化这个变量：  
```C++
  template<typename It>       // algorithm to dwim ("do what I mean")
  void dwim(It b, It e)       // for all elements in range from
  { // b to e
    while (b != e) {
    typename std::iterator_traits<It>::value_type
      currValue = *b;
    …
    }
  }
```  
天啊！_typename std::iterator_traits<It>::value_type_ 表示的是 _iterator_ 所指向的值的类型？真的？我真想忘记这有多有  
趣。天啊。等等，我说过这有趣吗？

好吧，再看一个简单有趣的例子，第三次了：声明一个局部变量，它的类型是 _closure_ 的类型。而 _closure_ 的类型  
只有编译器知道，因此不能被写出来。哎，太糟糕了。

糟糕，糟糕，糟糕。_C++_ 编程没有它应该有的愉快体验啊！

好吧，过去就是这样的。但从 _C++11_ 开始，_auto_ 出现了，全部的这些问题都得走开。因为 _auto_ 变量的类型是根  
据它们的 _initializer_ 而推导出的，所以是必须要进行初始化的。这意味着当你在 _modern C++_ 的高速公路上飞驰  
时，可以和未初始化问题说拜拜了：  
```C++
  int x1;                     // potentially uninitialized
  
  auto x2;                    // error! initializer required
  
  auto x3 = 0;                // fine, x's value is well-defined
```  

这条高速公路上也没有那些使用解引用 _iterator_ 来声明局部变量时所遇到的坑：  
```C++
  template<typename It>       // as before
  void dwim(It b, It e)
  {
    while (b != e) {
      auto currValue = *b;
      …
    }
  }
```

另外，因为 _auto_ 使用了类型推导，见 [_Item 2_](./Chapter%201.md#item-2-理解-auto-的类型推导)，它可以表示那些只被编译器所知道的类型：  
```C++
  auto derefUPLess =                              // comparison func.
    [](const std::unique_ptr<Widget>& p1,         // for Widgets
      const std::unique_ptr<Widget>& p2)          // pointed to by
    { return *p1 < *p2; };                        // std::unique_ptrs
```

非常酷。在 _C++14_ 中会更酷，因为 _lambda expressions_ 的形参可以使用 _auto_：  
```C++
  auto derefLess =            // C++14 comparison
    [](const auto& p1,        // function for
      const auto& p2)         // values pointed
    { return *p1 < *p2; };    // to by anything
                              // pointer-like
```

尽管这很酷，但是你大概正在想：我们真的不需要使用 _auto_ 来声明一个持有 _closures_ 的变量，因为我们可以使用  
_std::function_ 对象。是的，我们可以使用 _std::function_ 对象，但是它可能并不是你想的那样。你可能现在正在想“什  
么是 _std::function_ 对象啊？”所以，让我们一起来弄清楚。

_std::function_ 是一个 _C++11_ 标准库的模板，它推广了函数指针的概念。函数指针只可以指向函数，而 _std::function_  
对象可以引用任意可调用的对象，也就是任意可以像函数那样被调用的东西。就像是当你创建一个函数指针时，你  
必须要指明函数的类型也就是函数的 _signature_ 一样，当你创建一个 _std::function_ 对象时，你也必须要指明函数的  
类型。这可以通过 _std::function_ 的模板形参来完成。例如，去声明一个被命名为 _func_ 的 _std::function_ 对象，它可以  
引用任意可调用对象，就像它有这样的 _signature_ 一样：  
```C++
  bool(const std::unique_ptr<Widget>&,  // C++11 signature for
      const std::unique_ptr<Widget>&)   // std::unique_ptr<Widget>
                                        // comparison function
```  
你可以这样写：  
```C++
  std::function<bool(const std::unique_ptr<Widget>&,
                    const std::unique_ptr<Widget>&)> func;
```

因为 _lambda expressions_ 会生成可调用对象，所以 _closures_ 可以被存储到 _std::function_ 对象中。这意味着：我们可  
以在不使用 _auto_ 的情况下，像下面这样来声明 _derefUPLess_ 的 _C++11_ 版本：  
```C++
  std::function<bool(const std::unique_ptr<Widget>&,
                    const std::unique_ptr<Widget>&)>
    derefUPLess = [](const std::unique_ptr<Widget>& p1,
                    const std::unique_ptr<Widget>& p2)
                    { return *p1 < *p2; };
```

重要的是要意识到：即使抛开语法的冗余和需要去重复形参的类型不谈，使用 _std::function_ 和使用 _auto_ 也是不相  
同的。_auto_ 声明的持有 _closure_ 的变量的类型和 _closure_ 的类型是相同的，所使用的内存大小和 _closure_ 所需要的内  
存大小也是相同的。而 _std::function_ 声明的持有 _closure_ 的变量的类型是 _std::function_ 模板的实例，对于任意的所  
给定的 _signature_ 来说，它的大小都是固定的。这个大小对于存储 _closure_ 来说，可能是不够的。在这种场景下，  
_std::function_ 的构造函数就会去分配堆栈内存来存储 _closure_。导致的结果就是 _std::function_ 对象通常会比 _auto_ 所  
声明的对象使用更多的内存。由于限制内联和产生间接调用的实现细节，通过 _std::function_ 对象来调用 _closuer_ 几  
乎肯定要比通过 _auto_ 所声明的对象来调用 _closure_ 要慢。换句话说，_std::function_ 方法通常是比 _auto_ 方法要大和  
慢的，而且可能还会产生内存溢出的异常。再加上，正如你在上面的例子中看到的，写 _auto_ 要比写 _std::function_  
的实例化的类型轻松的多。在持有 _closure_ 的 _auto_ 和 _std::function_ 的比赛中，_auto_ 就是赢家。还有一个类似的讨  
论，那就是要使用 _auto_ 而不是使用 _std::function_ 来持有调用 _std::bind_ 所产生的结果，在  [_Item 34_](./Chapter%201.md#item-2-首选-lambdas-而不是-std::bind) 中，我会尽力说  
服你在任何情况下都使用 _lambdas_ 来代替 _std::bind_。

_auto_ 的优势不止可以避免未初始化的变量和冗长的变量声明，还可以直接持有 _closures_。它还有一个能力，那就是  
可以避免我称作为 **_type shortcuts_** 的问题。下面是一些你可能曾经看过的代码，甚至可能写成这样：  
```C++
  std::vector<int> v;
  …
  unsigned sz = v.size();
```  
_v.size()_ 的官方的返回类型是 _std::vector<int>::size_type_，但是很少有开发者会注意到它。_std::vector<int>::size_type_ 被指定为是  
_unsigned integral_ 类型，所以很多开发者认为使用 _unsigned_ 也足够了，所以就写成了上面那样。这可以有一些有  
趣的后果。例如，在 _32-bit_ _Windows_ 上，_unsigned_ 和 _std::vector<int>::size_type_ 是相同的大小，但是在 _64-bit_ _Windows_  
上，_unsigned_ 是 _32bits_ 而 _std::vector<int>::size_type_ 则是  _64bits_。这意味着在 _32-bit_ _Windows_ 上可以正确工作的代码可  
能在 _64-bit_ _Windows_ 下就表现的是不正确的，然后当你将程序从 _32bits_ 移植到 _64bits_ 时，谁会想花时间在这种问  
题上呢？

使用 _auto_ 可以确保你不需要做这些：  
```C++
  auto sz = v.size();         // sz's type is std::vector<int>::size_type
```

还不确定使用 _auto_ 是明智的？那就再考虑下面的代码：  
```C++
  std::unordered_map<std::string, int> m;
  …
  
  for (const std::pair<std::string, int>& p : m)
  {
    …                         // do something with p
  }
```  
这个代码看起来非常合理，但是存在一个问题，你看出来了吗？

要意识到缺失了什么，需要记住 _std::unordered_map_ 的 _key_ 是 _const_ 的，所以在哈希表也就是 _std::unordered_map_  
中的 _std::pair_ 的类型不是 _std::pair<std::string, int>_ 而是 _std::pair<const std::string, int>_。但是这却不是上面循环中  
的变量 _p_ 的类型。结果就是编译器将会尽力寻找方法来将 _std::pair<const std::string, int>_ 对象也就是哈希表中的对  
象转换为 _std::pair<std::string, int>_ 也就是 _p_ 的类型的对象。这可以通过以下方法来完成：通过拷贝
_m_ 中的每一个  
对象来创建 _p_ 想要绑定的类型的临时对象，然后再将引用 _p_ 绑定到这个临时对象上。在每次循环结束时，所对应  
的临时变量都将会被销毁。如果你写了这个循环的话，你可能会被这个行为所惊讶到，因为几乎可以肯定的是你想  
要只是将引用 _p_ 绑定到 _m_ 中的每个元素上而已。

一些无意识的类型不匹配可以被 **_自动_** 消除:  
```C++
  for (const auto& p : m)
  {
    … // as before
  }
```

这不仅是高效的，而且还是容易输入的。此外，这个代码中的一个非常吸引人的特性是：如果你获取到了 _p_ 的地  
址的话，那么你就是获取到了 _m_ 中的某个元素的地址。在没有使用 _auto_ 的代码中，你将会获取到是临时对象的地  
址，而这个临时对象将会在每次循环结束后被销毁掉。

最后的这两个例子：当应该写 _std::vector<int>::size_type_ 时却写了 _unsigned_，当应该写 _std::pair<const std::string, int>_ 时  
却写了 _std::pair<std::string, int>_ ，描述了显式指明类型是如何导致你既不想要也没想到的隐式转换所发生的，如  
果你使用了 _auto_ 来做为了目标变量的类型的话，那么你就不再需要担心你正在声明的变量的类型和用于初始化它  
的表达式的类型之间的不匹配了。

因此，这些都是首选 _auto_ 而不是显式类型声明的理由。但 _auto_ 不是完美的。每一个 _auto_ 变量的类型都是根据它  
的初始化表达式而推导出的，但有一些初始化表达式的类型却既不是所期待的也不是所符合要求的。关于这种场景  
发生的条件和关于这种场景你应该怎么做是在 [_Item 2_](./Chapter%201.md#item-2-理解-auto-的类型推导) 和 [_Item 6_](./Chapter%202.md#item-6-当-auto-推导出的类型是-undesired-时使用-the-explicitly-typed-initializer-idiom) 中进行讨论的，所以我不会在这里进行解决。相反  
地，我将会转移我的注意力到一个不同的问题上来，那就是在使用 _auto_ 来代替传统类型声明时所产生的源代码可  
读性问题。

首先，深呼吸，放轻松。_auto_ 是一个可选项，而不是必选项。如果你在你的专业判断下认为，通过使用显式类型  
声明，你的代码将会是更清晰的、更易维护的或者能以某种方式变得更好的话，那么你可以继续使用显式类型声  
明。但是请记住 _C++_ 在采用编程语言界普遍称之为类型推断的概念上并没有开创新的领域。其他的静态类型过程  
式语言，比如：_C#_、_D_、_Scala_ 和 _Visual Basic_，都有或多或少相同的特性，更不用说各种静态类型函数式语言了，  
比如：_ML_、_Haskell_、_OCaml_ 和 _F#_ 等。在某种程度上，这得归因于动态类型语言的成功，比如：_Perl_、_Python_ 和  
_Ruby_，它们其中的变量很少会被显式声明类型。软件开发社区对类型推断拥有丰富经验，并已证明这种技术与创  
建和维护大型、工业级代码库之间并无矛盾。

使用 _auto_ 减弱了通过快速看一眼源代码就能知道一个对象的类型的能力，一些开发者会对这种情况感到焦虑。但  
是，_IDE_ 显示对象的类型的能力常常可以缓解这种焦虑，即使是考虑到了 [_Item 4_](./Chapter%201.md#item-4-了解如何查看所推导的类型) 中提到的 _IDE_ 类型显示问题。对  
于一个对象的类型的抽象理解和它的实际类型是一样有用的。例如，通常知道一个对象是 _container_、_counter_ 或智  
能指针就足够了，而无需知道这个对象是什么类型的 _container_、_counter_ 或智能指针。假如再选一个好的变量名  
的话，这样抽象类型的信息应该几乎总是在眼前了。

事实上，显式类型声明通常会在正确性上或效率性上或两者上加大引入细小错误的机会。此外，如果 _auto_ 变量所  
对应的初始化表达式发生了改变的话，那么 _auto_ 变量所对应的类型也会自动地发生改变，这意味着通过使用 _auto_  
一些重构也更方便了。例如，一个函数最初被声明是返回 _int_，但是你后来又决定返回 _long_ 了，那么如果调用这个  
函数的结果是被保存在了 _auto_ 变量的话，那么调用代码会在你下次编译的时候自动进行更新。而如果调用这个函  
数的结果是被保存在了被显式声明为 _int_ 的变量中的话，那么你需要找到所有的调用点来进行修改。

### 需要记住的规则

* _auto_ 变量：必须要进行初始化、通常可以避免类型不匹配所导致的可移植问题或效率问题、可以使重构的过  
程简化还比显式指明类型变量要少进行一些输入。
* _auto_ 类型的变量有在 [_Item 2_](./Chapter%201.md#item-2-理解-auto-的类型推导) 和 [_Item 6_](./Chapter%202.md#item-6-当-auto-推导出的类型是-undesired-时使用-the-explicitly-typed-initializer-idiom) 中所描述的缺陷。

## Item 6 当 _auto_ 推导出的类型是 _undesired_ 时，使用 _the explicitly typed initializer idiom_

 [_Item 5_](./Chapter%202.md#item-5-首选-auto-而不是显式类型声明) 解释了使用 _auto_ 比起使用显式指明类型有着大量的技术优势，但是 _auto_ 的类型声明有时候会在你想去 **_东_**  
 时却去了 **_西_**。例如，我有一个函数，它持有 _Widget_ 并返回 _std::vector<bool>_，其中的每一个 _bool_ 都表示 _Widget_ 是否提  
 供有一种特性：  
 ```C++
  std::vector<bool> features(const Widget& w);
 ```  
更进一步假设 _bit 5_ 指示了 _Widget_ 是否有高优先级。因此我们可以像下面这样写代码：  
```C++
  Widget w;
  …
  
  bool highPriority = features(w)[5];   // is w high priority?
  …
  
  processWidget(w, highPriority);       // process w in accord
                                        // with its priority
```

这个代码没有错误。它可以很好地工作。但是，如果我们使用 _auto_ 来代替 _highPriority_ 的显式类型，做这样一个看似无害的改变的话：  
```C++
  auto highPriority = features(w)[5];   // is w high priority?
```  
那么这个情况就发生了改变。代码仍然可以通过编译，但是它的行为却不再可预期了：  
```C++
  processWidget(w, highPriority);       // undefined behavior!
```
正如注释所指明的，_processWidget_ 现在有了 _undefined behavior_。但是为什么呢？答案可能是令人惊讶的。在使  
用 _auto_ 的代码中，_highPriority_ 的类型不再是 _bool_ 了。尽管 _std::vector<bool>_ 在概念上持有 _bools_，但是 _std::vector<bool>_ 返回  
的却不是它的元素的引用了，_std::vector::operator[]_ 对除了 _bool_ 以外的类型都是返回引用。而对于 _bool_ 返回的却  
是类型 _std::vector<bool>::reference_，这是 _std::vector<bool>_ 中的一个类。 

_std::vector<bool>::reference_ 存在的原因是因为 _std::vector<bool>_ 被指定以压缩的格式来表示它的 _bools_，一个 _bool_ 一个 _bit_。这  
就产生了一个 _std::vector<bool>_ 的 _operator[]_ 所对应的问题，因为 _std::vector<bool>_ 的 _operator[]_ 应该返回 _T&_，但是 _C++_ 禁止  
引用 _bits_。不能够返回 _bool&_，所以 _std::vector<bool>_ 的 _operator[]_ 返回的是一个表现的像是 _bool&_ 的对象。为了可以这  
样，_std::vector<bool>::reference_ 必须在基本上所有可以使用 _bool&_ 的上下文中都可以使用。在 _std::vector<bool>::reference_ 中的  
可以完成这个工作的一个特性是可以隐式转换为 _bool_，是 _bool_ 而不是 _bool&_。去解释整套 _std::vector<bool>::reference_ 所  
使用的用来模拟 "bool&" 行为的的技巧将会偏离主题，所以我简单地认为这个隐式转换仅仅是大马赛克上的一颗  
石头。

记住的这些信息，再看这个代码：  
```C++
  bool highPriority = features(w)[5];   // declare highPriority's
                                        // type explicitly
```  
此处，_features_ 会返回一个 _std::vector<bool>_ 对象，而这个对象会再调用 _operator[]_。此时，这个被调用的 _operator[]_ 会返  
回一个 _std::vector<bool>::reference_ 对象，而这个对象会被隐式转换为 _bool_ 来初始化 _highPriority_。_highPriority_ 最终得到  
了 _features_ 所返回的 _std::vector<bool>_ 中的 _bit 5_ 的值，就像应该的那样。

和使用 _auto_ 声明 _highPriority_ 所发生的进行对比：  
```C++
  auto highPriority = features(w)[5];   // deduce highPriority's
                                        // type
```  

再一次 _features_ 会返回一个 _std::vector<bool>_ 对象，而再一次这个对象会再调用 _operator[]_。此时，再一次这个被调用的  
_operator[]_ 会返回一个 _std::vector<bool>::reference_ 对象，但是在此处就有变化了，因为 _auto_ 会将 highPriority 的类型推导  
为 _std::vector<bool>::reference_。_highPriority_ 完全不会是 _features_ 所返回的   _std::vector<bool>_ 中的 _bit 5_ 的值了。

_highPriority_ 是多少完全依赖于 _std::vector<bool>::reference_ 是如何实现的。一种实现是去包含一个指针，这个指针指向一  
个机器字，而这个机器字中保存着那个 _bit_ 和那个 _bit_ 所对应的偏移量。假设 _std::vector<bool>::reference_ 的实现就是这样  
的话，那么考虑这对 _highPriority_ 的初始化意味着什么呢？

 调用 _features_ 会返回了一个 _std::vector<bool>_ 类型的临时对象。这个临时对象没有名字，但是为了讨论，我会将他称之为  
 _temp_。_temp_ 会调用 _operator[]_ 来返回一个 _std::vector<bool>::reference_ 类型的对象，而这个 _std::vector<bool>::reference_ 类型的  
 对象包含着一个指针，这个指针指向一个机器字，而这个机器字是存放在一个持有那个 _bit_ 和那个 _bit_ 所对应的偏  
 移量的数据结构中的，而这个数据结构是由 _temp_ 所管理的。因为 _highPriority_ 是这个 _std::vector<bool>::reference_ 类型的  
 对象的拷贝，所以它也包含着一个指向 _temp_ 中的机器字的指针和 _bit 5_ 所对应的偏移量。在这个语句结束后，因  
 为 _temp_ 是临时对象会被销毁掉。所以，_highPriority_ 就包含有一个 _dangling_ 指针了，那么在调用 _processWidget_ 中就会有 _undefined behavior_ 了：  
 ```C++  
  processWidget(w, highPriority);       // undefined behavior!
                                        // highPriority contains
                                        // dangling pointer!
 ```  
 _std::vector<bool>::reference_ 是一个 _proxy class_ 的例子，_proxy class_ 存在的目的是模拟和增加一些类型的行为。_proxy class_  
 可用于多种目的。_std::vector<bool>::reference_ 的存在提供了一种错觉，那就是 _std::vector<bool>_ 的 _operator[]_ 返回的是 _bit_ 的引  
 用，又例如，_Standard Library_ 的智能指针类型就是 _proxy class_，智能指针是将资源管理嫁接到原始指针上的，见  
 [_Chapter 4_](./Chapter%204.md#-Chapter-4-智能指针)。_proxy classes_ 的使用历史久远。事实上，设计模式 **_Proxy_** 是软件设计模式中最长久的成员。

一些 _proxy classes_ 是被设计为客户可见的。例如，_std::shared_ptr_ 和 _std::unique_ptr_ 就是。还有一些 _proxy classes_  
是被设计为客户隐藏的，或多或少是的。_std::vector<bool>::reference_ 就是这样 **_invisble_** _proxies_ 的典型例子，_std::bitset_ 和  
_std::bitset::reference_ 也是这种场景。

_proxy class_ 阵营中还有一些 _C++_ 库中的类，它们使用了被称为 **_expression templates_** 的技术。这样的库最初是被  
开发来提高 _mumeric code_ 的效率的。给定一个类 _Matrix_ 和 _Matrix_ 对象 _m1_、_m2_、_m3_ 和 _m4_，例如，有一个表达  
式：  
```C++
  Matrix sum = m1 + m2 + m3 + m4;
```  
如果 _Matrix_ 的 _operator+_ 对象返回的是 _Matrix_ 所对应的  _proxy_ 而不是 _Matrix_ 时，那么这个表达式可以被高效地计  
算。所以，两个 _Matrix_ 对象所对应的 _operator+_ 将会返回一个 _proxy class_ 的对象，比如是 _Sum<Matrix, Matrix>_  
类型的对象而不是 _Matrix_ 类型的对象。正如 _std::vector<bool>::reference_ 和 _bool_ 的场景，这里也有从 _Matrix_ 到 _proxy_ 的  
隐式转换，这将允许使用 _=_ 右侧的表达式所生成的 _proxy_ 类型的对象来初始化 _sum_。这个 _proxy_ 通常会 _encode_ 整  
个初始化表达式，比如  _Sum<Sum<Sum<Matrix, Matrix>, Matrix>, Matrix>_。这绝对是要对客户进行屏蔽的。

一个普遍规则是 **_invisble_** _proxy classes_ 不能和 _auto_ 一起使用。因为这种类型的对象通常不会被设计为比单语句存  
在的还久，所以创建这种类型的变量就是在违反基础库的设计假设的。_std::vector<bool>::reference_ 也是这种场景，我们也  
已经看到了违反这种假设会导致 _undefined behavior_。

因此你要避免下面这样格式的代码：  
```C++
auto someVar = expression of "invisible" proxy class type; 
```  

但是你如何知道是否使用了 _proxy class_ 所对应的对象呢？使用了 _proxy class_ 所对应的对象的软件不太可能宣传它  
们的存在。它们应该是 _invisble_ 的，至少概念是的。一旦你发现它们存在，你真的必须要放弃 _auto_ 和 [_Item 5_](./Chapter%202.md#item-5-首选-auto-而不是显式类型声明) 中所  
描述的那么多优势吗？

让我们先来解决如何去发现它们的问题。尽管 **_invisble_** _proxy classes_ 被设计为是在日常使用中是在程序员的雷达之  
外飞行的，但是使用它们的库还是常常会进行说明的。你越熟悉你使用的库的基础设计决定，你就越可能少被这些  
库中的 _proxy_ 的使用所伤害。

文档不足时，头文件就会补足。源码几乎不可能完全掩盖 _proxy class_ 所对应的对象。因为它们通常是从客户所期  
待的调用中返回的，所以 _function signatures_ 通常会反应它们的存在。例如下面是 _std::vector<bool>::operator[]_ 的 _spec_：  
```C++
  namespace std { // from C++ Standards
  template <class Allocator>
  class vector<bool, Allocator> {
    public:
    …
    class reference { … };
    reference operator[](size_type n);
    …
    };
  }
```  
假设你知道 _std::vector<T>_ 的 _operator[]_ 通常返回的是 _T&_，则在这个场景中的 _operator[]_ 的非常规的返回类型就是一  
个提醒：正在使用 _proxy class_。仔细注意你正在使用的接口通常可以发现 _proxy classes_ 的存在。  

实际上，很多开发者只有当尝试追踪困惑的编译问题或调试错误的单元测试结果时，才发现使用了 _proxy classes_。  
不管是如何发现它们的，只要确定 _auto_ 推导的是 _proxy class_ 的类型而不是 _proxied_ 的类型，解决方案都不需要放  
弃 _auto_。_auto_ 本身不是问题。问题是 _auto_ 没有推导你想让它去推导的类型。解决方案是强迫去进行一个不同类型  
的推导。这种方法我称为 _the explicitly typed initializer idiom_。

_the explicitly typed initializer idiom_ 使用了 _auto_ 来声明变量，但是需要转换初始化表达式为你想要 _auto_ 去推导的  
类型。下面是如何使用 _auto_ 来将 _highPriority_ 强制转换为 _bool_，例如：  
```C++
  auto highPriority = static_cast<bool>(features(w)[5]);
```  
此处，_features(w)[5]_ 像以前一样继续返回一个 _std::vector<bool>::reference_ 类型的对象，但是这个转换将这个表达式的类  
型转换为了 _bool_，然后 _auto_ 推导 _bool_ 来做为 _highPriority_ 的类型。在运行时，将 _std::vector<bool>::operator[]_ 所返回的  
_std::vector<bool>::reference_ 类型的对象转换为了所支持的 _bool_，作为这个转换的一部份，那个仍然有效的指向 _feature_ 所  
返回的 _std::vector<bool>_ 的指针也被解引用了。这可以避免我们在前面遇到的 _undefined behavior_。然后 _index 5_ 被应用  
到那个指针所指向的 _bits_ 上，显现出来的 _bool_ 值就被用于初始化 _highPriority_ 了。

对于 _Matrix_ 的例子， _the explicitly typed initializer idiom_ 将会是下面这样：  
```C++
  auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```  
这种用法并不只限于在这种会产生 _proxy class_ 类型的 _initializers_ 的场景中使用。当你想要故意创建一个和是初始  
化表达式所生成的类型是不同的类型的变量时，这种用法也是有用的。例如，假定你有一个计算容差值的函数：  
```C++
  double calcEpsilon();       // return tolerance value
```  
_calcEpsilon_ 显然返回的是 _double_，但是假定了：你知道对于你的应用来说 _float_ 的精度就是足够的，并且你是在乎   
_floats_ 和 _doubles_ 之间的大小差异的。你可能会声明一个 _float_ 变量来存储 _calcEpsilon_ 的结果，  
```C++
  float ep = calcEpsilon();   // impliclitly convert
                              // double → float
```  
但是这很难表示出“我是故意缩小了这个函数所返回的值的精度。”但是使用 _the explicitly typed initializer idiom_ 的  
声明却可以做到：  
```C++
  auto ep = static_cast<float>(calcEpsilon());
```
当你有一个 _floating-point expression_ ，但却故意想储存到 _integral value_ 中时，也是类似的理念。假定你需要计算  
带有 _random access itertor_ 的 _container_ 中的元素的索引时，比如：_std::vector_、std::deque 或
_std::array_，然后给了  
你一个 _0.0_ 和 _1.0_ 之间的 _double_，这个 _double_ 表明了所要求的元素的位置距离 _container_ 的开头有多远，_0.5_ 表明  
是在 _container_ 的中间。再假定你确定所生成的索引是适合 _int_ 的。如果 _container_ 是 _c_ 而 _double_ 是 _d_ 的话，你可以这样来计算索引，  
 ```C++
  int index = d * c.size();
 ```  
你是故意将右侧的 _double_ 转换为 _int_ 的，但这样的写法却模糊了这个事实。而 _the explicitly typed initializer idiom_  
会让事情变得清晰：  
```C++
  auto index = static_cast<int>(d * c.size());
```

### 需要记住的规则：

* **_invisble_** _proxy_ 类型可以导致 _auto_ 根据初始化表达式推导出 **_错误_** 的类型。
* _the explicitly typed initializer idiom_ 可以强制 _auto_ 去推导你想要的 **_正确_** 的类型。