- [Chapter 2 auto](#chapter-2-auto)
  - [Item 5 首选 _auto_ 而不是显式类型声明](#item-5-首选-auto-而不是显式类型声明)
    - [需要记住的规则](#需要记住的规则)

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
发生的条件和关于这种场景你应该怎么做是在 [_Item 2_](./Chapter%201.md#item-2-理解-auto-的类型推导) 和 [_Item 6_](./Chapter%201.md#item-6) 中进行讨论的，所以我不会在这里进行解决。相反  
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
能指针就足够了，而无需知道这个对象是什么类型的 _container_、_counter_ 或智能指针。假如再选
一个好的变量名  
的话，这样抽象类型的信息应该几乎总是在眼前了。

事实上，显式类型声明通常会在正确性上或效率性上或两者上加大引入细小错误的机会。此外，如果 _auto_ 变量所  
对应的初始化表达式发生了改变的话，那么 _auto_ 变量所对应的类型也会自动地发生改变，这意味着通过使用 _auto_  
一些重构也更方便了。例如，一个函数最初被声明是返回 _int_，但是你后来又决定返回 _long_ 了，那么如果调用这个  
函数的结果是被保存在了 _auto_ 变量的话，那么调用代码会在你下次编译的时候自动进行更新。而如果调用这个函  
数的结果是被保存在了被显式声明为 _int_ 的变量中的话，那么你需要找到所有的调用点来进行修改。

### 需要记住的规则

* _auto_ 变量：必须要进行初始化、通常可以避免类型不匹配所导致的可移植问题或效率问题、可以使重构的过  
程简化还比显式指明类型变量要少进行一些输入。
* _auto_ 类型的变量有在 [_Item 2_](./Chapter%201.md#item-2-理解-auto-的类型推导) 和 [_Item 6_](./Chapter%201.md#item-6) 中所描述的缺陷。



