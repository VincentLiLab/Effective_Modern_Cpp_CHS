- [_Chapter 7_ 并发 _API_](#chapter-7-并发-api)
  - [_Item 35_ 首选 _task-based_ 编程而不是 _thread-based_ 编程](#item-35-首选-task-based-编程而不是-thread-based-编程)
    - [需要记住的规则](#需要记住的规则)
  - [_Item 36_ 如果异步是必须要的话，那么指明 _std::launch::async_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)
    - [需要记住的规则](#需要记住的规则-1)
  - [_Item 37_ 使 _std::thread_ 在所有路径上都为 _unjoinable_](#item-37-使-stdthread-在所有路径上都为-unjoinable)
    - [需要记住的规则](#需要记住的规则-2)
  - [_Item 38_ 注意各种各样的线程 _handle_ 的析构函数](#item-38-注意各种各样的线程-handle-的析构函数)
    - [需要记住的规则](#需要记住的规则-3)
  - [_Item 39_ 对于 _one-shot_ 事件通信考虑 _void future_](#item-39-对于-one-shot-事件通信考虑-void-future)
    - [需要记住的规则](#需要记住的规则-4)
  - [_Item 40_ 并发使用 _std::atomic_ 特殊内存使用 _volatile_](#item-40-并发使用-stdatomic-特殊内存使用-volatile)
    - [需要记住的规则](#需要记住的规则-5)


# _Chapter 7_ 并发 _API_

_C++11_ 的一个重大胜利是将并发合并到了语言和库中。熟悉其他线程 _API_，比如：_pthread_ 或 _Windows thread_，的程序员有时候会对 _C++_ 所提供的相对简陋的特性集感到惊讶，这是因为 _C++_ 对于并发的大部分支持都是按照限制 _compiler-writer_ 的形式存在的。所产生的语言保证意味着：在 _C++_ 历史上，程序员可以首次写具有跨平台标准行为的多线程程序了。这就建立了坚实的基础，很多富有表现力的库都可以此基础上进行构建了，标准库的并发组件：_task_、_future_、_thread_、_mutex_、_condition variable_ 和 _atomic_ 等等，都只是开始，用于开发并发 _C++_ 软件的工具集肯定会越来越丰富。

在后续的 _Item_ 中，请记住对于 _future_ 标准库有两套模板：_std::future_ 和 _std::shared_future_。因为在很多场景下，它们的区别是不重要的，所以我只会说 _future_，这意味着是说的是它们两个。

## _Item 35_ 首选 _task-based_ 编程而不是 _thread-based_ 编程

如果你想要异步运行函数 _doAsyncWork_ 的话，那么你有两个基本的选择。你可以创建 _std::thread_，并在它上面运行 _doAsyncWork_，因此利用了 _thread-based_ 的方法是：  
```C++
  int doAsyncWork();
  
  std::thread t(doAsyncWork);
```  
或者你可以传递 _doAsyncWork_ 给 _std::async_，这是一种被称为 _task-based_ 的策略：  
```C++
  auto fut = std::async(doAsyncWork);   // "fut" for "future"
```  
在这个调用中，所传递给 _std::async_ 的函数对象被认为是一个 _task_。

_task-based_ 方法通常是优于 _thread-basd_ 方法的，我们已经看到过的那些代码已经描述展示了一些原因。此处，_doAsyncWork_ 会生成一个返回值，我们有理由假设调用 _doAsyncWork_ 的代码是对这个返回值感兴趣的。当使用 _thread-based_ 方法时，没有简单的方法获取到这个返回值。当使用 _task-based_ 方法时，就简单了，因为 _std::async_ 所返回的 _future_ 提供了 _get_ 函数。如果 _doAsyncWork_ 会抛出一个异常的话，那么 _get_ 函数就更重要了，因为 _get_ 函数也可以访问这个异常。当使用 _thread-based_ 方法时，如果 _doAsyncWork_ 会抛出异常的话，那么程序会挂掉，这是通过调用 _std::terminate_ 来完成的。

_thread-based_ 编程和 _task-based_ 编程之间更根本的区别是 _task-based_ 表现出了更高级别的抽象。这让你可以从线程管理的细节中解脱出来，说到这里，我需要总结并发 _C++_ 软件中的 **_线程_** 的三个含义：  
* 硬件线程是实际执行计算的线程。现代的机器架构在每个 _CPU_ 上都一个或多个硬件线程。
* 软件线程，也被称为 _OS_ 线程或系统线程，是操作系统在所有进程间进行管理的并在硬件线程上调度执行的线程。通常，创建的软件线程可以比硬件线程多，因为，当一个软件线程被阻塞时，比如：等待 _I/O_ 或等待 _mutex_ 或 _condition variable_，可以通过执行其他非阻塞的线程来被提高吞吐率。
* _std::thread_ 是 _C++_ 进程中的对象，用来做为底层软件线程的 _handle_。一些 _std::thread_ 对象可以表现为 **_null_** _handle_，即为：没有所对应的软件线程，因为它们可能处在 _default-constructed_ 状态下，此时没有可以执行的函数、因为它们可能已经被移动了，此时所被移动到的 _std::thread_ 做为了底层软件线程的 _handle_、因为它们可能已经被 _join_ 了，此时所运行的函数已经结束了以及它们可能已经被 _detach_ 了，此时它们和它们所对应的底层软件线程之间的连接已经被分离了。

软件线程是一种有限资源。如果你尝试创建的软件线程多于系统可以提供的软件线程的话，_std::system_error_ 异常会被抛出。即使你要运行的函数不可以抛出异常，也是会这样的。例如：即使 _doAsyncWork_ 是 _noexcept_ 的，  
```C++
  int doAsyncWork() noexcept;           // see Item 14 for noexcept
```  
下面这个语句也可能会导致一个异常：  
```C++
  std::thread t(doAsyncWork);           // throws if no more
                                        // threads are available
```

友好设计的软件必须以某种方式来处理这种可能性，但是如何呢？一种方法是在当前线程中运行 _doAsyncWork_，但是这可能会导致不平衡的负载，而且如果当前线程是 _GUI_ 线程的话，那么会产生响应性问题。另一个选项是先等待一些现存的软件线程结束，然后再去尝试创建新的 _std::thread_，但那些现存的线程有可能正在等待这个 _doAsyncWork_ 所要执行的动作，比如：产生一个结果或者通知一个 _condition variable_。

即使你没有耗尽线程，你也可以遇到 _oversubscription_ 的问题。也就是当 _ready-to-run_ 的软件线程，比如：非阻塞的软件线程，多于硬件线程时。当这些发生时，通常是做为 _OS_ 一部分的线程调度器会对硬件上的软件线程进行 _time-slice_。当一个线程的 _time-slice_ 结束，另一个线程开始时，上下文切换会被执行。这些上下文切换增加了系统线程管理的整体负载，当调度某个软件线程的硬件线程所在的 _CPU_ 和这个软件线程在上一个 _time-slice_ 期间所对应的硬件线程所在的 _CPU_ 是不相同时，这些负载会尤其大。在这个场景下，（1）那个软件线程所对应的 _CPU_ 缓存通常是 _cold_，即为：这个 _CPU_ 缓存只包含少量对它是有用的数据和指令；（2）在这个 _CPU_ 上所运行的 **_新_** 软件线程会 **_污染_** **_旧_** 软件线程所对应的 _CPU_ 缓存，这个 **_旧_** 软件线程已经在这个 _CPU_ 上运行过了，而且可能还会再一次运行在这个 _CPU_ 上。 

避免 _oversubscription_ 是困难的，因为软件线程与硬件线程的最佳比例是依赖于软件线程变成可运行状态的频繁程度的，这是可以动态改变的，比如：当程序从 _IO_ 密集型成为计算密集型时。软件线程与硬件线程的最佳比例还依赖于上下文切换的成本和软件线程使用 _CPU_ 缓存的效率。此外，硬件线程的数量和 _CPU_ 缓存的细节，比如：_CPU_ 缓存的大小和相对速度，都依赖于硬件架构，所以，即使你调整了你的应用程序，避免了平台上的 _oversubscription_，但如果此时硬件仍然是繁忙的话，那么仍然不能保证你的方案在其他类型的平台是好的。

如果你可以把这些问题推给别人的话，那么你的生活就轻松了，使用 _std::async_ 正是做的这些：  
```C++
  auto fut = std::async(doAsyncWork);             // onus of thread mgmt is
                                                  // on implementer of
                                                  // the Standard Library
```

这个调用将线程管理的责任转换给了 _C++_ 标准库的实现者。例如：接收到 _out-of-thread_ 异常的可能性显著降低了，因为这个调用可能永远不会产生 _out-of-thread_ 异常。你可能好奇，为什么呢？如果我要求的软件线程多于系统可以提供的软件线程的话，那么这为什么会和是通过创建 _std::thread_ 来完成的还是通过创建 _std::async_ 来完成的有关系呢？是有关系的，因为当 _std::async_ 按照上面这样的形式被调用时，即为：使用 _default launch policy_ 时，见 [_Item 36_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)，它并不会保证一定会创建出一个软件线程。使用了 _default launch policy_ 的 _std::async_ 宁愿允许调度器将所指定的函数，在这个例子中是 _doAsyncWork_，安排到那个需要  _doAsyncWork_ 的结果的线程上，即为调用 _fut.get_ 或 _fut.wait_ 的线程上，如果系统是 _oversubscription_ 了或者是线程耗尽了的话，那么合理的调度器会利用这个自由。

在需要它的结果的线程上来运行它，如果你利用了这个技巧的话，那么我提醒你这可能会导致负载均衡的问题，而且这个问题不会轻易地消失，只是 _std::async_ 和运行调度器来面对这些问题了，不再是你了。然而，当谈到负载均衡时，关于在硬件上发生的事情，运行调度器可能比你了解的更全面，因为运行调度器管理着的是所有进程上的线程，而不是只有你的代码里所运行着的线程。

当使用 _std::async_ 时，_GUI_ 线程的响应性问题仍然存在，因为调度器没有方法知道你的哪个线程有严格的响应要求。在这种场景下，你需要将 _std::launch::async launch policy_ 给到 _std::async_。这将会确保你想要运行的函数真正地执行在一个不同的线程上，见 [_Item 36_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)。

_state-of-the-art_ 线程调度器利用 _system-wide_ 线程池从而避免了 _oversubscription_，它通过 _work-stealing_ 算法提升了硬件核心之间的负载均衡。_C++_ 标准不需要使用线程池或 _work-stealing_，老实说，_C++11_ 的并发规范的一些技术特色使得利用这个技术变得很困难，比我们想象的要困难。然而，一些开发商在它们的标准库实现中利用了这个技术，我们有理由期待在这方面会继续取得进展。如果你在你的并发开发中采用了 _task-based_ 方法的话，那么当这个技术变得更加普遍的时候，你就会自动获得了这个技术的益处了。另一方面，如果你是直接使用了 _std::thread_ 的话，你就要自己承担处理线程耗尽、_oversubscription_ 和负载均衡的责任了，更不用说你的这些问题的解决方法如何与运行在同一台机器上的其他进程中的程序中所实现的方案所协调。

相比于 _thread-based_ 编程，_task-based_ 设计避免了手动线程管理，并且它提供了天然的方法去检查异步执行的函数的结果，即为：返回值或异常。然而，也存在直接使用 _thread-based_ 编程是合适的场景。合适的场景包括：
* 你需要访问底层线程实现的 _API_。_C++_ 并发 _API_ 通常使用 _low-level_ 平台的特定的 _API_ 而实现，这些 _API_ 通常是 _pthread_ 和 _Windows_ 的 _Thread_。这些 _API_ 目前所提供的比 _C++_ 所提供的要丰富，例如：_C++_ 没有提供线程优先级和 _affinity_ 的概念。为了提供对底层线程实现的 _API_ 的访问，_std::thread_ 对象通常提供了 _native_handle_ 成员函数。_std::async_ 所返回的 _std::future_ 没有这个功能。
* 你需要并且能够去优化你的应用程序的线程使用率。这是可能会存在的场景，例如：如果你正在开发具有已知的 _execution profile_ 的服务器软件，并且这个 _execution profile_ 将做为唯一重要的进程来被部署到一台具有固定硬件特性的机器上的话。
* 你需要实现 _C++_ 的并发 _API_ 之外的线程技术，比如：在 _C++_ 实现中没有提供线程池的平台上的线程池。

然而，这些都是不常见的场景。大部分情况下，你需要选择 _task-based_ 设计而不是使用线程编程。

### 需要记住的规则

* _std::thread_ 的 _API_ 没有提供直接的方法来获取异步运行的函数的返回值，如果这些函数抛出了异常的话，程序是被终止的。
* _thread-based_ 编程调用需要手动管理线程耗尽、线程 _oversubscription_、线程负载均衡和到新平台的线程移植。
* _task-based_ 编程可以通过使用 _default launch policy_ 的 _std::async_ 来为你处理这些问题。  

## _Item 36_ 如果异步是必须要的话，那么指明 _std::launch::async_

当你调用 _std::async_ 去执行一个函数或其他可调用对象时，你通常期待的是异步执行这个函数。但是这不是你要求 _std::async_ 必须去做的事情。你现在要求的是按照 _std::async_ 的 _launch policy_ 来运行函数。共有两种标准策略，它们由 _std::launch_ _scoped enum_ 中的 _enumerator_ 所代表，_scoped enum_ 的相关信息，见 [_Item 10_](Chapter%203.md#item-10-首选-scoped-enum-而不是-unscoped-enum)。假设所传递给 _std::async_ 的函数是 _f_，

* _std::launch::async launch policy_ 意味着 _f_ 必须异步运行，即为：在一个不同的线程上。
* _The std::launch::deferred launch policy_ 意味着：只有当 _std::async_ 所返回的 _future_ 的 _get_ 或 _wait_ 被调用时，_f_ 才可以运行。也就是说 _f_ 的执行会被推迟到所对应的 _get_ 或 _wait_ 被执行时。当所对应的 _get_ 或 _wait_ 被调用时，_f_ 将会同步执行，即为：调用方将会被阻塞，直到 _f_ 结束运行。如果 _get_ 和 _wait_ 都没被调用时，_f_ 将永远不会运行。

大概上面说的会让你感到惊讶，_std::async_ 的 _default launch policy_，就是没有显式指定时所会使用的那个 _launch policy_，不是上面的那两个。而是上面的那两个的 **_或_**。下面的这两个调用有完全相同的含义：  
```C++
  auto fut1 = std::async(f);                      // run f using
                                                  // default launch
                                                  // policy
  
  auto fut2 = std::async(std::launch::async |     // run f either
                          std::launch::deferred,  // async or
                          f);                     // deferred
```   
_default policy_ 允许 _f_ 可以异步或同步运行。正如 [_Item 35_](#item-35-首选-task-based-编程而不是-thread-based-编程) 所指出的，这个灵活性允许 _std::async_ 和标准库的线程管理组件去承担线程构造和析构、避免 _oversubscription_ 和负载均衡的责任。这是 _std::async_ 的并发编程可以如此方便的原因。

但是，使用了 _default launch policy_ 的 _std::async_ 存在一些令人感兴趣的影响。假设执行下面这个语句的线程是 _t_，  
```C++
  auto fut = std::async(f);             // run f using default launch policy


```  
* 不可以预知 _f_ 是否会与 _t_ 并发运行，因为 _f_ 可能会被调度以被推迟运行。
* 不可以预知 _f_ 是否运行在一个和调用 _fut.get_ 或 _fut.wait_ 的线程是不相同的线程上。如果调用 _fut.get_ 或 _fut.wait_ 的线程是 _t_ 的话，那么不可以预知 _f_ 是否运行在一个和线程 _t_ 是不相同的线程上。
* 不可以预知 _f_ 是否运行，因为不能保证 _fut.get_ 或 _fut.wait_ 会在程序每一个路径上被调用到。
 
_default policy_ 的调度灵活性常常会频繁地和 _thread_local_ 变量搭配使用，因为这意味着：如果 _f_ 读或写这样的 _thread-local
storage_ _TLS_ 的话，那么不可以预知哪个线程的变量会被访问到：  
```C++
  auto fut = std::async(f);             // TLS for f possibly for
                                        // independent thread, but
                                        // possibly for thread
                                        // invoking get or wait on fut
```  
这样的调度灵活性还会影响使用了超时的 _wait-based_ 的循环，因为在被推迟的 _task_ 上所调用的 _wait-for_ 或 _wait-until_ 产生了值 _std::launch::deferred_，见 [_Item 35_](#item-35-首选-task-based-编程而不是-thread-based-编程)。这意味着下面的循环，它看起来最终应该会终止，但实际上却是会永远运行：  
```C++
  using namespace std::literals;        // for C++14 duration
                                        // suffixes; see Item 34

  void f()                              // f sleeps for 1 second,
  {                                     // then returns
    std::this_thread::sleep_for(1s);
  }
  

  auto fut = std::async(f);             // run f asynchronously
                                        // (conceptually)

  while (fut.wait_for(100ms) !=         // loop until f has
  std::future_status::ready)            // finished running...
  {                                     // which may never happen!
    …
  }

```  
如果 _f_ 和调用 _std::async_ 的线程是并发运行的话，即为：如果 _f_ 所选择的是 _std::launch::async_ 的话，那么此处是没有问题的，假设 _f_ 会最终结束，但是如果 _f_ 是被推迟了的话，那么 _fut.wait_for_ 将总是返回 _std::future_status::deferred_。永远不会等于 _std::future_status::ready_，所以这个循环将永远不会终止。 

这种类型的 _bug_ 在开发和单元测试期间很容易被忽略，因为这个 _bug_ 只会在负载很重时才会出现。这些负载是将机器推向 _oversubscription_ 或线程耗尽的条件，也是 _task_ 最有可能被推迟的条件。毕竟，如果硬件不是被 _oversubscription_ 或线程耗尽所威胁的话，运行系统没有理由不调度 _task_ 去并发执行。

修复的方法很简单：检查 _std::async_ 所对应的 _future_ 以去查看所对应的 _task_ 是否是被推迟的。如果是的话，那么就避免进入 _timeout-based_ 循环。不幸地是，没有直接的 _future_ 方法可以查询它多所对应的 _task_ 是否是被推迟的。反而，你必须去调用一个 _timeout-based_ 函数，比如 _wait-for_ 函数。在这个场景下，你真的不想去等待任何事情，你只是想要查看返回值是否是 _std::future_status::deferred_，所以在必须要绕圈子时，请压制你的轻微怀疑，去使用 _0_ 超时来调用 _wait_for_：  
```C++
  auto fut = std::async(f);                       // as above

  if (fut.wait_for(0s) ==                         // if task is
      std::future_status::deferred)               // deferred...
  {
                                                  // ...use wait or get on fut
    …                                             // to call f synchronously

  } else {                                        // task isn't deferred
    while (fut.wait_for(100ms) !=                 // infinite loop not
            std::future_status::ready) {          // possible (assuming
                                                  // f finishes)

    …                                             // task is neither deferred nor ready,
                                                  // so do concurrent work until it's ready
  }
    …                                             // fut is ready
  }
```

这么多的注意事项的结果是:只要下面的条件被满足了，那么对于一个 _task_ 来说，使用 _default launch policy_ 的 _std::async_ 就是好的：   
* 这个 _task_ 不需要和调用 _get_ 或 _wait_ 的线程并发运行。
* 读或写哪个线程的 _thread-local_ 变量并无影响。
* 要么保证 _get_ 或 _wait_ 将会在 _std::async_ 所返回的 _future_ 上被调用，要么可以接受这个 _task_ 可能永远不会被执行。
* 使用了 _wait_for_ 或 _wait_until_ 的代码考虑了推迟状态的可能性。
> 这不是他妈的傻逼设计吗？_C++_ _Standardization Committee_ 傻逼！傻逼！傻逼！

如果没有满足这些条件的任意一个的话，你可能想要的是确保 _std::async_ 去真正地异步执行所对应的 _task_。可以完成这个的方法是：当在调用 _std::async_ 时，传递 _std::launch::async_ 来做为它的第一个实参：  
```C++
  auto fut = std::async(std::launch::async, f);   // launch f
                                                  // asynchronously
```

事实上，如果有一个函数，它表现地像是 _std::async_，但是却会自动使用 _std::launch::async_ 来做为 _launch policy_ 的话，那么它是非常方便的工具，非常容易写。下面是 _C++11_ 的版本：  
```C++
  template<typename F, typename... Ts>
  inline
  std::future<typename std::result_of<F(Ts...)>::type>
  reallyAsync(F&& f, Ts&&... params)              // return future
  {                                               // for asynchronous
    return std::async(std::launch::async,         // call to f(params...)
                      std::forward<F>(f),
                      std::forward<Ts>(params)...);
  }
```

这个函数接收一个可调用对象 _f_ 和 _0_ 个或多个形参 _param_，然后完美转发它们给 _std::async_，见 [_Item 25_](Chapter%205.md#item-25-stdmove-用于右值引用-stdforward-用于-univeral-reference)，并且会转递 _std::launch::async_ 来做为 _std::async_ 的 _launch policy_。就像 _std::async_，对于使用 _param_ 调用 _f_ 的结果，_std::reallyAsync_ 返回的是 _std::future_。确定这个结果的类型是简单的，因为 _type trait_ _std::result_of_ 可以帮你做到，_type trait_ 的相关信息见 [_Item 9_](Chapter%203.md#item-9-首选-alias-declaration-而不是-typedef)。

像 _std::async_ 那样使用 _reallyAsync_：  
```C++
  auto fut = reallyAsync(f);            // run f asynchronously;
                                        // throw if std::async
                                        // would throw
```  
在 _C++14_ 中，可以推导 _reallyAsync_ 的返回类型的能力简化了函数的声明：  
```C++
  template<typename F, typename... Ts>
  inline
  auto                                  // C++14
  reallyAsync(F&& f, Ts&&... params)
  {
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Ts>(params)...);
  }
```

这个版本非常清晰的说明了 _reallyAsync_ 除了使用 _std::launch::async launch policy_ 调用 _std::async_ 外不会做其他的事情。

### 需要记住的规则

* _std::async_ 的 _default launch policy_ 允许异步 _task_ 运行，也允许同步 _task_ 运行。
* 上面的这个灵活性导致了访问 _thread_local_ 的不确定性、暗示了所对应的 _task_ 可能永远不会被执行到以及影响了 _time-based_ _wait_ 调用所对应的程序逻辑。
* 如果异步是必须要的话，那么指明 _std::launch::async_。

## _Item 37_ 使 _std::thread_ 在所有路径上都为 _unjoinable_

每个 _std::thread_ 都有两种状态：_joinable_ 或 _unjoinable_。_joinable_ _std::thread_ 对应的是正在运行或可以运行的底层异步执行的线程。例如：被阻塞的或正在等待被调度的底层线程所对应的 _std::thread_ 就是 _joinable_ _std::thread_。已经运行至结束的底层线程所对应的 _std::thread_ 对象也是 _joinable_ _std::thread_。

_unjoinable_ _std::thread_ 的含义如你所愿：不是 _joinable_ _std::thread_ 的 _std::thread_。_unjoinable_ _std::thread_ 对象包括：
* _default-constructed_ _std::thread_。这些 _std::thread_ 没有可以执行的函数，因此没有所对应的底层执行的线程。
* 已经被移动走的 _std::thread_ 对象。移动走的结果是某个 _std::thread_ 过去所对应的底层执行的线程现在对应上一个不同的 _std::thread_ 了。
* 已经被 _join_ 的 _std::thread_。在 _join_ 后，_std::thread_ 不再对应于那个已经结束运行的底层执行的线程了。
* 已经被 _detach_ 的 _std::thread_。_detach_ 会断开某个 _std::thread_ 对象和这个 _std::thread_ 对象所对应的底层执行的线程之间的联系。 

_std::thread_ 的 _joinability_ 如此重要的一个原因是：如果一个 _joinable_ _std::thread_ 的析构函数被调用了的话，那么程序的执行将会被终止。例如：假定我们有一个函数 _doWork_，它持有一个过滤函数 _filter_ 形参和最大值 _maxVal_ 形参。_doWork_ 会检查以确定它的必要计算条件都是满足的，然后会对 _0_ 和 _maxVal_ 之间的通过过滤的值执行计算。如果所对应的过滤操作和确定 _doWork_ 的条件是否是满足的操作都是耗时操作的话，那么有理由并发去完成这两个操作。

我们偏好利用 _task-based_ 设计来完成，见 [_Item 35_](#item-35-首选-task-based-编程而不是-thread-based-编程)，但是让我们假设我想要设置那个完成过滤的线程的优先级。[_Item 35_](#item-35-首选-task-based-编程而不是-thread-based-编程) 解释了需要使用线程的 _native handle_，这就只能通过 _std::thread_ _API_ 来完成了，而 _task-based_ _API_，即为：_future_，不可以完成。

我们可以像下面这样写代码：  
```C++
  constexpr auto tenMillion = 10000000;           // see Item 15
                                                  // for constexpr
  
  bool doWork(std::function<bool(int)> filter,    // returns whether
  int maxVal = tenMillion)                        // computation was
  {                                               // performed; see
                                                  // Item 2 for
                                                  // std::function

    std::vector<int> goodVals;                    // values that
                                                  // satisfy filter

    std::thread t([&filter, maxVal, &goodVals]    // populate
                  {                               // goodVals
                    for (auto i = 0; i <= maxVal; ++i)
                    { if (filter(i)) goodVals.push_back(i); }
                  });

    auto nh = t.native_handle();                  // use t's native
    …                                             // handle to set
                                                  // t's priority
                                                  
    if (conditionsAreSatisfied()) {
      t.join();                                   // let t finish
      performComputation(goodVals);
      return true;                                // computation was
    }                                             // performed

      return false;                               // computation was
  }                                               // not performed
```  
在我解释为什么这个代码是有问题的之前，我先提醒下 _tenMillion_ 的初始化值可以通过利用 _C++14_ 的 _'_ 做为数字分隔符来变得更可读:  
```C++
  constexpr auto tenMillion = 10'000'000;         // C++14
```  
我也再提醒下：在 _t_ 已经开始运行后再去设置 _t_ 的优先级有点像谚语说的马都跑了才去关上谷仓的门。一个好的设计是在暂停的状态下去开始，这样就可以在执行任意计算前先调整它的优先级，但是我不想你因那个代码而分心。如果你因没有这个代码而更加分心了的话，请转到 [_Item 39_](#item-39-对于-one-shot-事件通信考虑-void-future)，因为其中展示了启动暂停的线程。

你可能好奇为什么 _std::thread_ 的析构函数要这样。这是因为另外两个明显的选项是更糟糕的。这两个选项是：  
* 隐式 _join_。在这个场景下，_std::thread_ 的析构函数会等待它的底层异步运行的线程完成。这听起来是合理的，但是这可能会导致非常难以追踪的性能异常。例如：如果 _conditionsAreSatisfied()_ 已经返回了 _false_ 而  _doWork_ 还在等待所有的值在应用到它的 _filter_ 的话，那么就不合理了。
* 隐式 _detach_。在这个场景下， _detach_ 会断开某个 _std::thread_ 对象和这个 _std::thread_ 对象所对应的底层执行的线程之间的联系。底层线程会继续运行。这个方法听起来和 _join_ 方法一样合理，但是它可以导致的调试问题是更糟糕的。例如：在 _doWork_ 中，_goodVals_ 是按 _by-reference_ 的方式所捕捉的局部变量。这个 _goodVals_ 是在 _lambda_ 中通过调用 _push_back_ 被更改的。假定在这个 _lambda_ 异步运行时 _conditionsAreSatisfied()_ 返回了 _false_。那么在这个场景下，_doWork_ 将会结束，它的局部变量，包括 _goodVals_，都将会被销毁。_doWork_ 的 _stack frame_ 会被弹出，_doWork_ 的线程将从 _doWork_ 的调用点继续执行。这个调用点后的语句在某个时间点会调用其他的函数调用，至少会有一个函数可能最终会使用到那些曾经是被 _doWork_ 的 _stack frame_ 所占用的一部分或全部内存。让我们把这个函数称为 _f_。在 _f_ 运行时，_doWork_ 所创建的 _lambda_ 也将继续异步运行。这个 _lambda_ 可能会在过去是存放 _goodVals_ 而现在是在 _f_ 的 _stack frame_ 中了的栈内存上调用 _push_back_。这个调用会更改过去是存放 _goodVals_ 的内存，这意味着：从 _f_ 的角度看，它的 _stack frame_ 中的内存的内容可能会被莫名其妙的改变。想象下调式这个的乐趣。

_Standardization Committee_ 认为销毁 _joinable_ _std::thread_ 是非常可怕的，所以就禁止了隐式 _join_ 和隐式 _detach_ 方法，而是指定析构 _joinable_ _std::thread_ 会导致程序终止。

_Standardization Committee_ 把责任推给了你，你需要去确保：如果你使用了某个 _std::thread_ 对象的话，那么你要使这个 _std::thread_ 在定义这个 _std::thread_ 的作用域之外的所有路径上都为 _unjoinable_。但是覆盖所有路径是复杂的。它包括有作用域的结束和通过 _return_、_continue_、_break_、_goto_ 或异常的跳转。可以有非常多的路径。

任何时候如果你想要在一块区域外的所有路径上执行一些动作的话，那么通用的方法都是将这个动作放到局部变量的析构函数中去。这些对象被称为是 _RAII_ 对象，它们所来自的类被称为 _RAII_ 类型。_RAII_ 是 **_Resource Acquisition Is Initialization_**。_RAII_ 在标准库中非常常见。这些例子包括：_STL_ _container_，每个 _container_ 的析构函数都会销毁它所包含的内容并释放相关内存、标准智能指针，[_Item 18_](Chapter%204.md#item-18-对于-exclusive-ownership-的资源管理使用-stdunique_ptr) [_Item 19_](Chapter%204.md#item-19-对于-shared-ownership-的资源管理使用-stdshared_ptr) [_Item 20_](Chapter%204.md#item-20-对于可能会悬空的-stdshared_ptr-like-指针使用-stdweak_ptr) 解释了 _std::unique_ 的析构函数会在它所对应的对象上调用它的 _deleter_，_std::shared_ptr_ 和 _std::weak_ptr_ 的析构函数会减少引用计数、_std::fstream_ 对象，它们的析构函数会关闭它们所对应的文件。但是没有 _std::thread_ 对象所对应的标准 _RAII_ 类，可能是因为 _Standardization Committee_ 在拒绝将 _join_ 和 _detach_ 做为默认选项后，不知道这样的类应该做什么。

幸运地是，自己写一个也不困难。例如：当 _ThreadRAII_ 对象，_std::thread_ 所对应的 _RAII_ 对象，被销毁时，下面的类允许调用方去指定 _join_ 还是 _detach_ 应该被调用：  
```C++
  class ThreadRAII {
  public:
    enum class DtorAction { join, detach };       // see Item 10 for
                                                  // enum class info
    
    ThreadRAII(std::thread&& t, DtorAction a)     // in dtor, take
    : action(a), t(std::move(t)) {}               // action a on t
    
    ~ThreadRAII()
    {                                             // see below for
      if (t.joinable()) {                         // joinability test
        if (action == DtorAction::join) {
          t.join();
        } else {
          t.detach();
        }

      }
    }

    std::thread& get() { return t; } // see below
    
    private:
      DtorAction action;
      std::thread t;
    };
```  
我希望这个代码是能够自解释的，但是下面的观点是有用的：  
* 构造函数只接受 _std::thread_ 右值，因为我们想要将所传入的 _std::thread_ 移动到 _ThreadRAII_ 对象中。回忆下 _std::thread_ 对象是不可以拷贝的。
* 构造函数中的参数顺序对调用方来说是符合直觉的，先指定 _std::thread_，后指定析构函数的动作，这比反过来要更合理，但是成员初始化列表是被设计为匹配数据成员的声明顺序的。这个顺序将 _std::thread_ 对象放在了后面。在这个类中，这个顺序没有影响，但是通常来说，一个数据成员的初始化是可以依赖于另一个数据成员的，因为 _std::thread_ 对象可能会在它们被初始化后立即开始运行一个函数，所以将 _std::thread_ 对象声明在类的最后是一个好习惯。这保证了：在 _std::thread_ 对象被构造出时，所有处在它们之前的数据成员都已经被初始化过了，因此这些数据也可以被那些 _std::thread_ 数据成员所对应的异步运行的线程所访问了。
* _ThreadRAII_ 提供了可以访问底层 _std::thread_ 对象的 _get_ 函数。这和那个可以访问底层原始指针的标准智能指针类所提供的 _get_ 函数是相似的。提供 _get_ 避免了 _ThreadRAII_ 去重复 _std::thread_ 的全部接口，这也意味着 _ThreadRAII_ 对象可以被用在需要 _std::thread_ 对象的上下文中。
* 在 _ThreadRAII_ 的析构函数中调用 _std::thread_ 对象 _t_ 的成员函数之前，先去确保 _t_ 是 _joinable_。这是有必要的，因为在 _unjoinable_ 线程上调用 _join_ 或 _detach_ 会产生 _undefined behavior_。客户可以构造一个 _std::thread_、可以根据 _std::thread_ 来创建 _ThreadRAII_ 对象、可以使用 _get_ 来访问 _t_ 以及可以移动 _t_ 或在 _t_ 调用 _join_ 或 _detach_，这些动作都可以让 _t_ 成为 _unjoinable_。

如果你担忧下面的代码中会有竞争的话，因为在 _t.joinable()_ 的执行和 _join_ 或 _detach_ 的调用之间，另一个线程可以让 _t_ 成为 _unjoinable_，那么你的直觉是值得赞扬的，但是你的恐惧是没有理由的。_std::thread_ 对象只可以通过成员函数调用，比如：_join_、_detach_ 或移动函数，才可以将状态从 _joinable_ 改变为 _unjoinable_。当某个 _ThreadRAII_ 对象的析构函数被调用时，是没有其他的线程会去调用这个 _ThreadRAII_ 对象的成员函数的。如果是同时调用的话，那么必然是存在竞争的，但是不会发生在析构函数中，只会发生在尝试在同一时刻调用同一个对象的两个成员函数的客户代码中，一个析构函数，一个其他函数。通常来说，同时调用同一个对象上的成员函数只有当这些成员是 _const_ 时才是安全的，见 [_Item 16_](Chapter%203.md#item-16-使-const-成员函数成为线程安全的)。  
```C++
  if (t.joinable()) {
    
    if (action == DtorAction::join) {
      t.join();
    } else {
      t.detach();
    }
  }
```

在我们的 _doWork_ 上利用 _ThreadRAII_ 会看起来像是下面这样：  
```C++
  bool doWork(std::function<bool(int)> filter,    // as before
              int maxVal = tenMillion)
  {
    std::vector<int> goodVals;                    // as before
    
    ThreadRAII t( // use RAII object
      std::thread([&filter, maxVal, &goodVals]
                  {
                  for (auto i = 0; i <= maxVal; ++i)
                  { if (filter(i)) goodVals.push_back(i); }
                  }),
                  ThreadRAII::DtorAction::join    // RAII action
    );

    auto nh = t.get().native_handle();
    …

    if (conditionsAreSatisfied()) {
      t.get().join();
      performComputation(goodVals);
      return true;
    }

    return false;
  }
```  
在这个场景中，我们已经选择了在 _ThreadRAII_ 的析构函数中对异步运行的线程中执行 _join_ 了，正如我们看到过的，调用 _detach_ 可能导致一些完全是噩梦般的调试。我们也看到过了调用 _join_ 可能会导致性能异常，老实说，也可能会导致不轻松的调试，但是在 _detach_ 带给我们的 _undefined behavior_、使用原始 _std::thread_ 所产生的程序终止和性能异常之间，性能异常似乎是其中最好的。

哎，[_Item 39_](#item-39-对于-one-shot-事件通信考虑-void-future) 描述了使用 _ThreadRAII_ 对 _std::thread_ 来执行 _join_ 有时候不仅可以导致性能异常，还会导致程序挂起。这种类型的问题的 **_合适的_** 解决方案是去和异步运行的 _lambda_ 通信，通知这个 _lambda_ 不再需要它的工作，它应该尽早结束了，但是 _C++11_ 不支持中断线程。中断线程可以被手动实现，但这超出了本书的范围。

[_Item 17_](Chapter%203.md#item-17-理解特殊成员函数的生成) 解释了：因为 _ThreadRAII_ 声明了析构函数，所以不会有编译器生成的 _move operation_ 了，但是 _ThreadRAII_ 对象没有理由不可以被移动。如果编译器可以生成这些函数的话，那么这些函数是会做正确的事情的，所以显式要求创建这些函数是合适的：  
```C++
  class ThreadRAII {
  public:
    enum class DtorAction { join, detach };                 // as before
    
    ThreadRAII(std::thread&& t, DtorAction a)               // as before
    : action(a), t(std::move(t)) {}
    
    ~ThreadRAII()
    {
      …                                                     // as before
    }

    ThreadRAII(ThreadRAII&&) = default;                     // support
    ThreadRAII& operator=(ThreadRAII&&) = default;          // moving
    
    std::thread& get() { return t; }                        // as before
    
  private:                                                  // as before
    DtorAction action;
    std::thread t;
  };
```

### 需要记住的规则

* 使 _std::thread_ 在所有路径上都为 _unjoinable_。
* _join-on-destruction_ 可以导致很难去调试的性能异常。
* _detach-on-destruction_ 可以导致很难去调试的 _undefined behavior_。
* 在数据成员的列表最后再去声明 _std::thread_ 对象。

## _Item 38_ 注意各种各样的线程 _handle_ 的析构函数

[_Item 37_](#item-37-使-stdthread-在所有路径上都为-unjoinable) 解释了一个 _joinable_ _std::thread_ 对应有一个底层系统执行的线程。一个 _non-deferred task_ 所对应的 _future_，见 [_Item 36_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)，也对应有一个底层系统执行的线程。因此，_std::thread_ 对象和 _future_ 对象都可以被认为是系统线程的 _handle_。

从这个角度看，_std::thread_ 和 _future_ 在它们的析构函数中有着不同的行为，这是有趣的。正如 [_Item 37_](#item-37-使-stdthread-在所有路径上都为-unjoinable) 所说的，析构 _joinable_ _std::thread_ 会终止你的程序，因为其他两个明显的替代方法：隐式 _join_ 和隐式 _detach_，被认为是更糟糕的选择。_future_ 的析构函数有时候表现地像是执行了隐式 _join_，有时候表现地像是执行了隐式 _detach_，有时候表现地既不像是执行了隐式 _join_ 也不像是执行了隐式 _detach_。它永远不会导致程序终止。这个线程 _handle_ 行为的杂烩值得详细的解释。

_future_ 是一个通信通道的端点，_callee_ 通过这个通道将结果发送给 _caller_，我们从这个观察开始。通常是异步运行的 _callee_ 通过 _std::promise_ 对象来将它的计算的结果写到通信通道中，_caller_ 使用 _future_ 来读取这个结果。你可以认为是下面这样的，其中的虚线箭头展示了从 _callee_ 到 _caller_ 的信息流：  

![Alt text](image/image8.jpg)

但是 _callee_ 的结果应该被存储在哪里呢？在 _caller_ 在所对应的 _future_ 上调用 _get_ 前，_callee_ 就可能已经结束了，所以这个结果不可以被存储在 _callee_ 的 _std::promise_ 中。因为 _std::promise_ 是 _callee_ 的局部对象，所以当 _callee_ 结束时，这个 _std::promise_ 会被销毁。

这个结果也不可以被存储到 _caller_ 的 _future_ 中，因为 _std::future_ 可以被用来创建 _std::shared_future_，这会将 _std::future_ 所对应的 _callee_ 的结果的 _ownership_ 转移给 _std::shared_future_，而 _std::shared_future_ 可以在原始的 _std::future_ 被销毁后被拷贝很多次。鉴于不是所有的结果类型都可以被拷贝，即为：_move-only_ 类型，这个结果最后一个指向它的 _future_ 存在，它就必须要存在。一个 _callee_ 可以对应有的多个 _future_，其中的哪一个 _future_ 应该包含这个结果呢？ 

因为 _callee_ 所相关的对象和 _caller_ 所相关的对象都不适合来存储 _callee_ 的结果，所以这个结果是被存在它们两个之外的其他位置的。这个位置被称为 _shared state_。_shared state_ 通常用 _head-based_ 对象来表示，但是 _shared state_ 的类型、接口和实现都没有被标准所指定。标准库的作者可以按照它们喜欢的方式来实现 _shared state_。

我们可以预想 _callee_、_caller_ 和 _shared state_ 之间的关系为下面这样，其中的虚线箭头再一次代表了信息流：  

![Alt text](image/image9.jpg)

_share state_ 的存在是重要的，因为 _future_ 的析构函数的行为，本 _Item_ 的观点，是由所对应的 _future_ 所相关的 _shared state_ 所决定的。特别地，  
* 指向着通过 _async_ 所创建的 _non-deferred task_ 所对应的 _shared state_ 的最后一个 _future_ 的析构函数会被阻塞，直到所对应的 _task_ 结束。本质上，这个 _future_ 的析构函数会在那个运行异步执行的 _task_ 的线程上执行隐式 _join_。
* 剩下的其他 _future_ 的析构函数都只会销毁所对应的 _future_ 对象。对于异步运行的线程，这相当于在底层线程上执行了隐式 _detach_。对于最后一个 _future_ 所对应的 _deferred task_，这意味着这个 _deferred task_ 永远不会运行。

这些规则听起来很复杂，其实并没有那么复杂。我们真正要处理的是一个简单的 **_正常_** 行为和一个少见的例外。正常行为是 _future_ 的析构函数会销毁 _future_ 对象。它不会 _join_ 任何东西，它不会 _detach_ 任何东西，它不会运行任何东西。它只是会销毁 _future_ 的数据成员。好吧，实际上，它还会做一件事，它会减小它所对应的 _shared state_ 中的引用计数，而这个 _shared state_ 是被指向着这个 _shared state_ 的所有 _future_ 和这个 _shared state_ 所对应的 _callee_ 的 _std::promise_ 所操作的。这个引用计数使得库可以知道这个 _shared state_ 何时可以被销毁。关于引用计数的通用信息，见 [_Item 19_](Chapter%204.md#item-19-对于-shared-ownership-的资源管理使用-stdshared_ptr)。

只有当满足了下面这些时，某个 _future_ 才会发生正常行为所对应的例外：  
* 这个 _future_ 指向通过调用 _std::async_ 所创建的 _shared state_。
* _task_ 的 _launch policy_ 是 _std::launch::async_，见 [_Item 36_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)，这可以是运行系统所选择的，也可以是使用 _std::async_ 来指定的。
* 这个 _future_ 是最后一个指向所对应的 _shared state_ 的 _future_。对于 _std::future_ 来说，总是这样的。对于 _std::shared_future_ 来说，如果其他的 _std::shared_future_ 指向了与被销毁的 _future_ 是相同的 _shared state_ 的话，那么被销毁的 _future_ 
遵循正常行为，即为：它只会销毁它的数据成员。

只有当这些条件被满足时，_future_ 的析构函数才会表现为特殊行为，这个特殊行为是会被阻塞，直到异步运行的 _task_ 结束。实际来说，这相当于对正在运行的 _std::async_ 所创建的线程执行隐式 _join_。

_future_ 的析构函数的正常行为所对应的例外被概括为了“_std::async_ 所创建的那些 _future_ 会在它们的析构函数中被阻塞”。粗略点说，这是正确的，但是有时候你需要精确点。现在你已经知道了这个真理的全部荣耀和神奇了。

你的好奇可能成为不同的形式了。可能是“我好奇为什么通过 _async_ 所创建的 _non-deferred task_ 所对应的 _shared state_ 存在有特殊规则”。这是合理的疑问。据我所知，_Standardization Committee_ 想要避免隐式 _detach_ 所相关的问题，见 [_Item 37_](#item-37-使-stdthread-在所有路径上都为-unjoinable)，但又不想要采纳像强制程序终止这样的激进策略，就像它们对 _joinable std::threads_ 所做的那样，再一次见，[_Item 37_](#item-37-使-stdthread-在所有路径上都为-unjoinable)，所以它们进行了妥协，使用了隐式 _join_。这个决定并非没有争论，在 _C++14_ 中有过关于放弃这个行为的讨论。最后，最后没有发生，所以 _future_ 的析构函数的行为在 _C++11_ 和 _C++14_ 中是相同的。

_future_ 的 _API_ 没有提供方法来确定某个 _future_ 是否指向了那个调用 _std::async_ 所产生的 _shared state_，所以对于任意的 _future_ 对象，不能知道它是否会在它的析构函数中被阻塞，以去等待异步运行的 _task_ 结束。这有一些有趣的暗示：  
```C++
  // this container might block in its dtor, because one or more
  // contained futures could refer to a shared state for a non-
  // deferred task launched via std::async
  std::vector<std::future<void>> futs;  // see Item 39 for info
                                        // on std::future<void>

  class Widget {                        // Widget objects might
  public:                               // block in their dtors
    
    ...
  
  private:
    std::shared_future<double> fut;
  };
```

当然，如果你有方法知道所给定的 _future_ 没有满足触发析构函数的特殊行为的条件的话，比如：由于程序逻辑，那么你可以确定这个 _future_ 是不会在它的析构函数中阻塞的。例如：只有调用 _std::async_ 所产生的 _shared state_ 才能满足产生特殊行为的条件，但是有其他的方法来创建 _shared state_。其中之一是使用 _std::packaged_task_。_std::packaged_task_ 对象通过封装一个函数或其他调用对象来为异步执行来准备一个函数或其他可调用的对象，这样就把封装的结果放到了所对应的 _shared state_ 中。指向这个 _shared state_ 的 _future_ 可以通过 _std::packaged_task_ 的 _get_future_ 函数来获取：  
```C++
  int calcValue();                      // func to run
  
  std::packaged_task<int()>             // wrap calcValue so it
  pt(calcValue);                        // can run asynchronously
  
  auto fut = pt.get_future();           // get future for pt
```  
此时，我们知道了 _future_ _fut_ 不会指向调用 _std::async_ 所创建的 _shared state_，所以它的析构函数将变现为正常行为。

一旦 _std::packaged_task_ _pt_ 被创建，_pt_ 可以被运行在一个线程上，_pt_ 也可以通过调用 _std::async_ 来被运行，但是如果你想要使用 _std::async_ 来运行一个 _task_ 的话，那么就没有理由去创建 _std::packaged_task_ 了，因为在它调度这个 _task_ 去执行前，_std::async_ 可以做 _std::packaged_task_ 可以做的所有事情。

因为 _std::packaged_task_ 是不可拷贝的，所以当 _pt_ 被传递 _std::thread_ 的构造函数时，它必须被转换为右值
，这是通过 _std::move_ 来完成的，见 [_Item 23_](Chapter%205.md#item-23-理解-stdmove-和-stdforward)：  
```C++
  std::thread t(std::move(pt));         // run pt on t
```  
这个例子对 _future_ 的析构函数的正常行为提供了一些见解，但是更容易看出如果这个语句被放在了一个块中的话：  
```C++
  {                                     // begin block

    std::packaged_task<int()>
      pt(calcValue);

    auto fut = pt.get_future();
    std::thread t(std::move(pt));
    
    …                                   // see below
  
  }                                     // end block
```

这个代码最有趣的部分是 _..._，它在 _std::thread_ 对象 _t_ 的创建之后，在块的结束之前。让这个代码变得有趣的是在 _..._ 中 _t_ 可以发生什么。下面是三种基本的可能性：
* _t_ 没有变化。在这个场景中，_t_ 在这个块结束时将会是 _joinable_。这会导致程序被终止，见 [_Item 37_](#item-37-使-stdthread-在所有路径上都为-unjoinable)。
* _t_ 执行了 _join_。在这个场景下，_fut_ 不需要在它的析构函数中被阻塞，因为 _join_ 已经出现在了代码中。
* _t_ 执行了 _detach_。在这个场景下，_fut_ 不需要在它的析构函数中 _detach_，因为代码已经做了这件事。

换句话说，当你有一个通过 _std::packaged_task_ 所产生的 _shared state_ 所对应的 _future_ 时，通常不需要采纳特殊的析构策略，这是因为在终止、_join_ 和 _detach_ 之间的决定通常是在操作 _std::thread_ 的代码中做出的，而 _std::packaged_task_ 通常运行在这个 _std::thread_ 线程上。

### 需要记住的规则

* _future_ 的析构函数通常只会销毁 _future_ 的数据成员。
* 指向着通过 _async_ 所创建的 _non-deferred task_ 所对应的 _shared state_ 的最后一个 _future_ 的析构函数会被阻塞，直到这个 _task_ 结束。

## _Item 39_ 对于 _one-shot_ 事件通信考虑 _void future_ 

有时候一个 _task_ 需要告诉另一个异步运行的 _task_ 发生了一个特定的事件，因为另一个 _task_ 直到这个事件发生后才可以开始运行。这个条件有可能是：一个数据结构已经被初始化了、一个阶段的计算已经被完成了或者一个重要的传感器的值已经被探测到了。当是这种场景时，最好的线程间通信方式是什么呢？

一个明显的方法是使用 _condition variable_ _condvar_。如果我们把探测某个条件的 _task_ 称为 _detecting task_，把会对这个条件做出反应的 _task_ 称为 _reacting task_ 的话，那么策略是简单的：_reacting task_ 会等待一个 _condition variable_，当某个事件发生时，_detecting task_ 则会通知这个 _condvar_。给定  
```C++
  std::condition_variable cv;           // condvar for event
  
  std::mutex m;                         // mutex for use with cv
```  
_detecting task_ 中的代码是非常简单的：  
```C++
  …                                     // detect event
  
  cv.notify_one();                      // tell reacting task
```  
如果存在多个需要被通知的 _reacting task_ 的话，那么使用 _notify_all_ 来代替 _notify_one_ 是合适的，但现在我们假设只有一个 _reacting task_。  

_reacting task_ 的代码是稍微有点复杂的，这是因为在 _condvar_ 上调用 _wait_ 前，必须先要通过 _std::unique_lock_ 对象来锁定 _mutex_。因为在等待 _condition variable_ 前先锁定 _mutex_ 是线程库的常见操作。通过 _std::unique_lock_ 对象来锁定 _mutex_ 的需求只是 _C++11_ _API_ 的一部分。下面是概念的方法：  
```C++
  …                                     // prepare to react

  {                                     // open critical section

    std::unique_lock<std::mutex> lk(m); // lock mutex

    cv.wait(lk);                        // wait for notify;
                                        // this isn't correct!
    
    …                                   // react to event
                                        // (m is locked)
  
  }                                     // close crit. section;
                                        // unlock m via lk's dtor
  
  …                                     // continue reacting
                                        // (m now unlocked)
```

这个方法所存在的第一个问题有时候被称为 _code smell_：即使这个代码可以工作，有些事情似乎也不是非常正确。在这个场景中，这个 _odor_ 来自于使用 _mutex_ 的需求。_mutex_ 是被用来控制对共享数据的访问的，但是 _detecting task_ 和 _reacting task_ 可能完全不需要这样的保护。例如：_detecting task_ 可能负责初始化一个全局的数据结构，然后将这个数据结构移交给 _reacting task_ 来使用。如果 _detecting task_ 永远不会在初始化这个数据结构后去访问这个数据结构，且 _reacting task_ 永远不会在 _detecting task_ 准备好之前去访问这个数据结构的话，那么这两个 _task_ 可以通过程序逻辑来互不干涉。不需要使用 _mutex_。_condvar_ 方法需要一个 _mutex_ 的事实就留下了可疑设计的不安气息。

即使你回头看下，有两个其他的问题你应该肯定会注意到：  
* 如果 _detecting task_ 在 _reacting task_ 等待之前就已经通知了 _condvar_ 的话，那么 _reacting task_ 将会挂起。按照通知 _condvar_ 去唤醒另一个 _task_ 的顺序，其他的 _task_ 必须先等待这个 _condvar_。如果 _detecting task_ 碰巧是在 _reacting task_ 执行 _wait_ 之前先执行了通知的话，那么 _reacting task_ 将丢失这个通知，它将会永远等待。
* _wait_ 语句没有考虑到 _spurious wakeup_。在很多语言的线程 _API_ 中，不只是在 _C++_ 线程 _API_ 中，等待某个 _condition variable_ 的代码可能会在这个 _condition variable_ 没有被通知的情况下就被唤醒了。这样的唤醒被称为 _spurious wakeup_。合适的代码可以通过确认那个被等待的条件确实真正地产生了，并将这个确认操作做为唤醒后的首个动作来处理这种 _spurious wakeup_ 问题。_C++_ _condvar_ _API_ 非常容易解决这种 _spurious wakeup_ 问题，因为它允许将被用来测试那个被等待的条件的 _lambda_ 或其他函数对象传递给 _wait_。也就是说，在 _reacting task_ 中的 _wait_ 调用看起来像是下面这样。想要利用这个能力需要 _reacting task_ 能够确定它正在等待的条件是否发生了。但是在我们已经考虑到的情景中，_reacting task_ 正在等待的条件是发生了一个 _detecting task_ 负责识别的事件。_reacting task_ 可能没有方法来确定它正在等待的事件是否已经发生了。这也是为什么要等待 _condition variable_。  
```C++
  cv.wait(lk,
    []{ return whether the event has occurred; });
``` 
  
存在很多适合使用 _condvar_ 处理问题的情况，但是这个似乎不是其中之一。

对于很多开发者来说，在他们背包中的下一个技巧是 _shared boolean flag_。这个 _flag_ 最初是 _false_。当 _detecting task_ 识别到它正在查找的事件时，它会设置这个 _flag_：  
```C++
  std::atomic<bool> flag(false);        // shared flag; see
                                        // Item 40 for std::atomic

  …                                     // detect event

  flag = true;                          // tell reacting task
```  
对于 _reacting task_ 来说，_reacting task_ 只会轮询这个 _flag_。当它看到这个 _flag_ 被设置了时，它就知道它正在等待的事件已经发生了。
```C++
  …                                     // prepare to react
  
  while (!flag);                        // wait for event  
  
  …                                     // react to event
```  
这个方法不受 _condvar-based_ 设计的缺点的影响。不需要 _mutex_，如果 _detecting task_ 在 _reacting task_ 开始轮询前就先设置了 _flag_ 的话，那么是没有问题的，没有类似的 _spurious wakeup_ 问题。棒，棒，棒。

不好的地方是会在 _reacting task_ 中花费成本进行轮询。在 _reacting task_ 等待 _flag_ 被设置期间，肯定是会被阻塞的，但它却仍然在运行。因此这个 _reacting task_ 占用了其他的 _task_ 可以去使用的硬件线程，_reacting task_ 在每次开始或者结束它所对应的 _time slice_ 时，都会带来上下文切换的成本，_reacting task_ 还可以只让一个 _CPU_ 保持运行，让其他的 _CPU_ 关闭去节省能耗。真正被阻塞的 _task_ 不会做这些事情。这正是 _condvar-based_ 方法的优势，因为在 _wait_ 调用中的 _task_ 是真正被阻塞的。

非常常见的是将 _condvar_ 和 _flag-based_ 设计结合起来。某个 _flag_ 表示感兴趣的事件是否已经发生了，但是访问这个 _flag_ 是由 _mutex_ 所同步的。因为 _mutex_ 可以避免并发访问这个 _flag_，正如 [_Item 40_](#item-40-并发使用-stdatomic-特殊内存使用-volatile) 所解释的，不需要将这个 _flag_ 设置为 _std::atomic_，一个简单的 _bool_ 就可以了。_detecting task_ 看起来像是下面这样：  
```C++
  std::condition_variable cv;                     // as before
  std::mutex m;
  bool flag(false);                               // not std::atomic
  
  …                                               // detect event
  
  {
    std::lock_guard<std::mutex> g(m);             // lock m via g's ctor
    
    flag = true;                                  // tell reacting task
                                                  // (part 1)
  
  }                                               // unlock m via g's dtor
  
  cv.notify_one();                                // tell reacting task
                                                  // (part 2)

```  
下面是 _reacting task_：  
```C++
  …                                               // prepare to react
  
  {                                               // as before
    std::unique_lock<std::mutex> lk(m);           // as before
    
    cv.wait(lk, [] { return flag; });             // use lambda to avoid
                                                  // spurious wakeups
    
    …                                             // react to event
                                                  // (m is locked)
  }
  
  …                                               // continue reacting
                                                  // (m now unlocked)
```

这个方法避免了我们已经讨论过的问题。不管 _reacting task_ 等待是否是在 _detecting task_ 通知之前，这个方法都可以工作，这个方法就算出现了 _spurious wakeup_ 也可以工作，不需要轮询。但一种 _odor_ 仍然存在，因为 _detecting task_ 仍然在按照非常奇怪的方式与 _reacting task_ 通信。通知 _condition variable_ 是告诉 _reacting task_ 它正在等待的事件已经发生了，但是 _reacting task_ 还必须要去检查这个 _flag_ 才能确定。设置了这个 _flag_ 是告诉了 _reacting task_ 那个事件是真正地发生了，但是 _detecting task_ 仍然要通知 _condition variable_，为的是 _reacting task_ 将被唤醒以去检查那个 _flag_。这个方法可以工作，但似乎不是非常干净。

一个替代方法是通过在 _reacting task_ 上对被 _detecting task_ 设置的 _future_ 执行 _wait_ 来避免 _condition variable_、_mutex_ 和 _flag_。这似乎像是一个奇怪的想法。毕竟 [_Item 38_](#item-38-注意各种各样的线程-handle-的析构函数) 解释了：_future_ 代表了一个从 _callee_ 到 _caller_ 的通信通道的接收端点，通常是异步的，在 _detecting_ 和 _reacting_ 之间没有 _callee-caller_ 的关系。然而，[_Item 38_](#item-38-注意各种各样的线程-handle-的析构函数) 也说了通信通道的发送端点是 _std::promise_ 而接收端点是 _future_， 这可以被用于多种通信，而不只是 _callee-caller_ 通信。这个通信可以被用于你需要将信息从程序中的一点发送到另一点的情景。在这个场景下，我们将使用这个通信来将信息从 _detecting task_ 发送至 _reacting task_，这个我们所传递的信息就是已经发生的感兴趣的事件。

这个设计是简单的。_detecting task_ 有 _std::promise_ 对象，即为：通信通道的写端点，_reacting task_ 有所对应的 _future_。当 _detecting task_ 看到它正在寻找的事件已经发生时，它会设置 _std::promise_，即为：写入通信通道。与此同时，_reacting task_ 会等待它的 _future_。这个等待会阻塞 _reacting task_，直到 _std::promise_ 被设置了。

现在，_std::promise_ 和 _future_，即为 _std::future_ 和 _std::shared_future_，是需要类型形参的模板。这个形参表示通过通信通道来发送的数据的类型。然而，在我们的场景中，没有数据需要传递。_reacting task_ 唯一感兴趣的事情是它的 _future_ 已经被设置了。对于 _std::promise_ 和 _future_ 模板，我们需要的是表示没有数据在通信通道上进行传递的的类型。这个类型正是 _void_。因此，_detecting task_ 会使用 _std::promise&lt;void&gt;_，_reacting task_ 会使用 _std::future&lt;void&gt;_ 或 _std::shared_ptr&lt;void&gt;_。当感兴趣的事件发生时，_detecting task_ 会设置它的 _std::promise&lt;void&gt;_，而 _reacting task_ 则会等待它的 _future_。尽管 _reacting task_ 不会从 _detecting task_ 接收任何数据，但通信通道仍然允许 _reacting task_ 知道什么时候 _detecting task_ 通过在 _std::promise_ 上调用 _set_value_ 写入了它的 _void_ 数据。

所以给定  
```C++
  std::promise<void> p;                 // promise for
                                        // communications channel
```  
_detecting task_ 的代码是简单的，  
```C++
  …                                     // detect event
  
  p.set_value();                        // tell reacting task
```  
_reacting task_ 的代码也是简单的：  
```C++
  …                                     // prepare to react
  
  p.get_future().wait();                // wait on future
                                        // corresponding to p
  
  …                                     // react to event
```

像使用 _flag_ 的方法一样，这个设计不需要 _mutex_，不管 _detecting task_ 是否会在 _reacting task_ 等待前先设置了它的 _std::promise_，都可以工作，并且不受 _spurious wakeup_ 的影响，只有条件变量才会被 _spurious wakeup_ 影响。像 _condvar-based_ 方法一样，_reacting task_ 调用 _wait_ 后 _reacting task_ 是真正被阻塞的，所以在等待的时候不会消耗系统资源。完美，对吧？

也不完全完美。是的，_future-based_ 方法没有了那些问题，但有其他的问题需要担忧。例如：[_Item 38_](#item-38-注意各种各样的线程-handle-的析构函数) 解释了：在 _std::promise_ 和 _future_ 之间的是 _shared state_，而这个 _shared state_ 通常是动态分配的。因此，你应该假设这个设计引入了 _heap-based_ 分配和释放的成本。

可能更重要的是，_std::promise_ 可能只能被设置一次。_std::promise_ 和 _future_ 之间的通信通道是 _one-shot_ 机制：它不可以被重复使用。这是和 _condvar-based_ 设计和 _flag-based_ 设计的重要的区别，它们两个可以被用来通信多次，_condvar_ 可以被重复通知，而 _flag_ 总是可以被清除和再次设置。

_one-shot_ 的约束可能并不像你想象的限制那么大。假定你想要创建一个处在挂起状态下的系统线程。也就是说，你想尽可能减少与线程创建所相关的所有开销，为的是：当你准备在线程上执行一些事情时，一般的线程创建的延迟可以被避免。或者你可能想要创建挂起的线程，为的是你可以在它运行前去配置它。这样的配置可能包括设置它的优先级或 _core affinity_。_C++_ 的并发 _API_ 并没有提供可以完成这些事情的方法，但是 _std::thread_ 对象提供了 _native_handle_ 成员函数，这个成员函数所返回的结果可以让你访问平台的底层线程 _API_，通常是 _POSIX_ 线程或 _Windows_ 线程。低级别的 _API_ 通常可以配置像优先级和 _affinity_ 这样的线程特性。

假设你想要的是只挂起某个线程一次，发生是在这个线程创建后，在它的线程函数运行前，使用 _void future_ 的设计是合理的选择。下面是这个设计的本质：  
```C++
  std::promise<void> p;

  void react();                                   // func for reacting task
  
  void detect()                                   // func for detecting task
  {
    std::thread t([]                              // create thread
                  {
                    p.get_future().wait();        // suspend t until
                    react();                      // future is set
                  });
    …                                             // here, t is suspended
                                                  // prior to call to react
  
  p.set_value();                                  // unsuspend t (and thus
                                                  // call react)
  
  …                                               // do additional work
  
  t.join();                                       // make t unjoinable
}                                                 // (see Item 37)
```

要把 _detect_ 外的所有路径上的 _t_ 都变为 _unjoinable_，这是非常重要的，使用像 [_Item 37_](#item-37-使-stdthread-在所有路径上都为-unjoinable) 的 _ThreadRAII_ 的 _RAII_ 类似乎是明智的。代码像是下面这样：  
```C++
  void detect()
  {
    ThreadRAII tr(                                // use RAII object
      std::thread([]
                  {
                    p.get_future().wait();
                    react();
                  }),
      ThreadRAII::DtorAction::join                // risky! (see below)
    );
    
    …                                             // thread inside tr
                                                  // is suspended here
    
    p.set_value();                                // unsuspend thread
                                                  // inside tr
    
    …
  
  }
```

这看起来比实际上要更安全。问题在于：如果在第一个 _..._ 区域中，注释 _thread inside tr is suspended here_ 的位置，一个异常被抛出了的话，那么 _set_value_ 将永远不会结束。这意味着运行 _lambda_ 的线程永远不会结束，这是一个问题，因为 _RAII_ 对象 _tr_ 已经被配置为在 _tr_ 的析构函数中对所对应的线程执行 _join_ 了。换句话说，如果在代码的第一个 _..._ 区域中抛出了一个异常的话，这个函数将会挂起，因为 _tr_ 的析构函数永远不会结束。

有方法可以解决这个问题，但是我将这个问题留给读者来解决，把它做为一个宝贵的练习。此处，我要展示的是原始代码，即为：没有使用的 _ThreadRAII_ 的代码，如何可以被扩展以去对 _reacting task_ 的挂起和解挂执行多次，而不是一次。这是简单的概括，因为关键是在 _react_ 代码中使用 _std::shared_future_ 来代替 _std::future_。一旦你知道了 _std::future_ 的 _share_ 成员函数会将它的 _shared state_ 的 _ownership_ 转移到 _share_ 所产生的 _std::shared_future_ 对象中，这个代码几乎可以自己写出来。这个唯一的细小差别是每个 _reacting task_ 都需要它自己的指向着所对应的 _shared state_ 的 _std::shared_future_ 的副本，所以根据 _share_ 所获取的 _std::shared_future_ 是通过那个运行在 _reacting task_ 上 _lambda_ 来进行 _by-value_ 捕获的：  
```C++
  std::promise<void> p;                           // as before

  void detect()                                   // now for multiple
  {                                               // reacting tasks
    
    auto sf = p.get_future().share();             // sf's type is
                                                  // std::shared_future<void>
    
    std::vector<std::thread> vt;                  // container for
                                                  // reacting threads
    for (int i = 0; i < threadsToRun; ++i) {
      vt.emplace_back([sf]{ sf.wait();            // wait on local
                            react(); });          // copy of sf; see
    }                                             // Item 42 for info
                                                  // on emplace_back
    
    …                                             // detect hangs if
                                                  // this "…" code throws!
    
    p.set_value();                                // unsuspend all threads
    
    …
    
    for (auto& t : vt) {                          // make all threads
      t.join();                                   // unjoinable; see Item 2
    }                                             // for info on "auto&"
  }
```  

使用了 _future_ 的设计可以实现这个工作，这个事实是值得注意的，这也是为什么你应该考虑将这个设计用于 _one-shot_ 事件通信。

### 需要记住的规则

* 对于简单的事件通信来说，_condvar-based_ 设计需要多余的 _mutex_、会对 _detecting task_ 和 _reacting task_ 之间的相互行进施加约束以及需要 _reacting task_ 去通知事件已经发生了。
* 利用了 _flag_ 的设计避免了这些问题，但是是基于轮询而不是阻塞的。
* _condvar_ 和 _flag_ 可以被一起使用，但是所产生的通信机制是有点呆板的。
* 使用 _std::promise_ 和 _future_ 避开了这些问题，但是这个方法使用了 _shared state_ 所对应的堆内存，只能用于 _one shot_ 通信。

## _Item 40_ 并发使用 _std::atomic_ 特殊内存使用 _volatile_

可怜的 _volatile_。如此被人误解。它不应该出现在本 _chapter_，因为它和并发编程无关。但是在其他的编程语言中，比如：_Jave_ 和 _C#_，它是有用的，甚至在 _C++_ 中，一些编译器也已经添加了 _volatile_ 语义，使得 _volatile_ 可以应用于并发软件，但是这是只有当使用那些编译器编译时。就算只是为了消除围绕着 _volatile_ 的困惑，在并发 _chapter_ 中讨论 _volatile_ 也是值得的。

程序员有时候会把 _volatile_ 和真正是属于本 _chapter_ 的 _C++_ 的特性 _std::atomic_ 模板搞混淆。这个模板的实例化，比如：_std::atomic&lt;int&gt;_、_std::atomic&lt;bool&gt;_ 和 _std::atomic&lt;Widget&gt;_ 等，提供了可以保证被其他线程视为原子操作的操作。一旦 _std::atomic_ 对象被构造出来，在这些对象上的操作表现地就好像它们是在 _mutex-protected_ 临界区中一样，但是这些操作通常是使用特殊机器指令实现的，比利用 _mutex_ 更加高效。

考虑使用了 _std::atomic_ 的代码：  
```C++
  std::atomic<int> ai(0);               // initialize ai to 0
  
  ai = 10;                              // atomically set ai to 10

  std::cout << ai;                      // atomically read ai's value
  
  ++ai;                                 // atomically increment ai to 11

  --ai;                                 // atomically decrement ai to 10
```  

在这些语句执行期间，其他读取 _ai_ 的线程只可能看到 _0_、_10_ 或 _11_。没有其他可能的值，当然，假设只有一个线程在更改 _ai_。

这个例子有两个方面值得注意。首先，在 _std::cout << ai;_ 语句中，_ai_ 是 _std::atomic_ 的事实只能保证读取 _ai_ 是原子的。不能保证整个语句是原子的。在 _ai_ 的值被读取和 _operator<<_ 被执行以去将 _ai_ 的值写入到标准输出之间，其他的线程可能已经修改了 _ai_ 的值了。这不会影响到这个语句的行为，因为 _int_ 所对应的 _operator<<_ 是使用 _by-value_ 的 _int_ 形参去输出的，因此所输出的值将会是从 _ai_ 所读出的值，重要的是理解在这个语句中的原子操作只有读取 _ai_ 的操作而已。

这个例子的第二个值得注意的方面是最后两个语句的行为，_ai_ 的增加和减少。这两个语句每一个都是 _read-modify-write_ _RMW_ 操作，它们是原子执行的。这是 _std::atomic_ 类型最好的特性之一，一旦 _std::atomic_ 对象被构造出来，它们的所有成员函数，包括这些 _RMW_ 操作，都保证被其他线程视为是原子的。

相反地，所对应的使用了 _volatile_ 的代码在多线程的上下文中几乎不保证任何事情：  
```C++
  volatile int vi(0);                   // initialize vi to 0
  
  vi = 10;                              // set vi to 10
  
  std::cout << vi;                      // read vi's value
  
  ++vi;                                 // increment vi to 11
  
  --vi;                                 // decrement vi to 10
```  
在这个代码执行期间，如果其他的线程正在读取 _vi_ 的值的话，那么它们可能看到任何东西，例如：_-12_、_68_、_4090727_。这样的代码将会有 _undefined behavior_，因为这些语句更改了 _vi_，所以如果其他的线程同时正在读取 _vi_ 的话，那么就是在同时读写同一块内存了，而这块内存既不是 _std::atomic_ 的，也没有 _mutex_ 所保护，这是数据竞争的定义。

做为一个 _std::atomic_ 和 _volatile_ 的行为在多线程编程是如何不同的具体例子，考虑两个被多线程所增加的简单的 _counter_。我们将它们两个初始化为 _0_：  
```C++
  std::atomic<int> ac(0);               // "atomic counter"
  
  volatile int vc(0);                   // "volatile counter"
```  
我们将在两个同时运行的线程中增加每一个 _counter_ 一次：  

![Alt text](image/image10.jpg)

当两个线程完成时，_ac_ 的值，即为：_std::atomic_ 的值，一定为 _2_，因为每个增加都是不可分割的操作。另一方面，_vc_ 的值不需要为 _2_，因为它的增加不可能原子地发生。它的每个增加都由下面这些所组成：读取 _vc_ 的值、增加被读取到的值以及将增加后的结果写回到 _vc_ 中。但是对于 _volatile_ 对象来说，这些操作不保证是原子的，所以 _vc_ 的两个增加的组成部分可能是像下面这样交错的：  
* _Thread 1_ 读取 _vc_ 的值，它是 _0_。
* _Thread 2_ 读取 _vc_ 的值，它仍然是 _0_。
* _Thread 1_ 增加它读到的 _0_ 为 _1_，然后将 _1_ 写到 _vc_ 中。
* _Thread 2_ 增加它读到的 _0_ 为 _1_，然后将 _1_ 写到 _vc_ 中。

因此 _vc_ 的最终值是 _1_，即使 _vc_ 是被增加了两次的。

这不是唯一可能的结果。_vc_ 的最终值通常是不可预测的，因为 _vc_ 涉及到了数据竞争。标准规定数据竞争会导致 _undefined behavior_，这意味着编译器可能生成做任意事情的代码。当然编译器不会利用这个余地来进行恶意操作。编译器宁愿去执行在没有数据竞争的程序中是有效的优化，这些优化会在存在有数据竞争的程序中产生意外的和不可预测的行为。

使用 _RMW_ 操作不是唯一 _std::atomic_ 是成功而 _volatile_ 是失败的情景。假定一个 _task_ 计算一个重要的且被另一个 _task_ 所需要的值。当第一个 _task_ 计算完成这个值时，必须要将这个值传递给第二个 _task_。[_Item 39_](#item-39-对于-one-shot-事件通信考虑-void-future) 解释了第一个 _task_ 将所期望的值的有效性传递给第二个 _task_ 的一个方法是使用 _std::atomic&lt;bool&gt;_。计算这个值的 _task_ 的代码看起来像下面这样：  
```C++
  std::atomic<bool> valAvailable(false);
  
  auto imptValue = computeImportantValue();       // compute value
  
  valAvailable = true;                            // tell other task
                                                  // it's available
```
当人们看到这个代码时，我们知道，_imptValue_ 的赋值是发生在 _valAvailable_ 的赋值之前的，这是重要的，但是编译器看到的所有只是一对独立变量的赋值而已。通常情况下，编译器被允许去重新排序这些不相关的赋值。也就是说，给定下面这样的赋值序列，其中 _a_、_b_、_x_ 和 _y_ 分别对应于独立的变量，  
```C++
  a = b;
  x = y;
```  
编译器可能会将它们乱序为下面这样：  
```C++
  x = y;
  a = b;
```  
即使编译器不会将它们重新排序，底层硬件仍然可能将它们重新排序，或可能让其他 _CPU_ 看起来好像是重新排序过的，因为有时候这可以让代码运行更快。

然而，使用 _std::atomic_ 会对代码的重新排序施加限制，这个限制是源代码中位于写 _std::atomic_ 变量之前的代码不得放在这个写 _std::atomic_ 变量之后，或可能让其他 _CPU_ 看起来好像是这样的。这意味着在我们的代码中，  
```C++
  auto imptValue = computeImportantValue();       // compute value
  
  valAvailable = true;                            // tell other task
                                                  // it's available
```  
编译器不仅必须保持 _imptValue_ 和 _valAvailable_ 赋值的顺序，还必须生成可以确保底层硬件也会这样做的代码。因此，声明 _valAvailable_ 为 _std::atomic_ 可以确保我们的关键排序需求是保持不变的：_imptValue_ 必须被所有线程视为不晚于 _valAvailable_ 的更改。

声明 _valAvailable_ 为 _volatile_ 不会对相同的代码的重新排序施加限制：  
```C++
  volatile bool valAvailable(false);
  
  auto imptValue = computeImportantValue();
  
  valAvailable = true;                            // other threads might see this assignment
                                                  // before the one to imptValue!
```  
此处，编译器可能会反转 _imptValue_ 和 _valAvailable_ 的赋值的顺序，即使不会，编译器也可能不能生成可以避免底层硬件使其他 _CPU_ 上的代码视为 _imptValue_ 是在 _imptValue_ 之前更改的机器代码。

这两个问题，不保证操作的原子性和没有在代码重新排序上提供限制，解释了为什么 _volatile_ 对于并发编程是无用的，但是没有解释 _volatile_ 对于什么是有用的。简单说，_volatile_ 用于告诉编译器它们正在处理不是常规表现的内存。

**_常规_** 内存的特性有：如果你把一个值写入到了某个内存位置上，这个值会保持在那里，直到进行了重写。所以如果我有一个正常的 _int_，  
```C++
  int x;
```  
并且编译器看到了下面这样的操作顺序的话，  
```C++
  auto y = x;                           // read x
  y = x;                                // read x again
```    
那么编译器可以通过消除 _y_ 的赋值来优化所生成的代码，因为 _y_ 的初始化是多余的。

**_常规_** 内存的特性还有：如果你把一个值写入到了某个内存位置上，永远不会读取它，并且又再次写了这个内存位置的话，那么第一次的写可以被清除，因为第一次的写永远不会被用到。所以给定两个相邻的语句，  
```C++
  x = 10;                               // write x
  x = 20;                               // write x again
```  
编译器可以清除第一个。这意味着：如果我们在源代码中有这样的代码的话，  
```C++
  auto y = x;                           // read x
  y = x;                                // read x again
  x = 10;                               // write x
  x = 20;                               // write x again
```  
编译器可以把它做为下面这样：  
```C++
  auto y = x;                           // read x
  
  x = 20;                               // write x
```

让你好奇谁会写执行这种多余的读和多余的写的代码，技术上被称为 _redundant load_ 和 _dead store_，答案是人们不会直接这样写，至少我们希望不会。然而，在编译器获取了看起来是合理的代码，并且执行了模板实例化、内联、各种各样的常见的重新排序优化后，在所得到的结果中包含有编译器可以避免的 _redundant load_ 和 _dead store_ 就非常常见了。

这样的优化只有当内存表现为 **_常规_** 时才是有效的。**_特殊_** 内存不是这样的。特殊内存最常见的类型可能是被用于 _memory-mapped I/O_ 的内存。这样内存中的位置实际上是在与外设，比如：外挂传感器或显示器、打印机以及网口等，进行通信，而不是在读或写常规内存，比如：_RAM_。在这样的上下文中，再一次考虑看起来是多余的读的代码：  
```C++
  auto y = x;                 // read x
  y = x;                      // read x again
```  
如果 _x_ 对应的是温度传感器所报告的值的话，那么第二次读 _x_ 就不是多余的了，因为温度可能在两个读取之间已经改变了。  

看起来是多余的写也是类似的情景。例如：在下面的代码中：  
```C++
  x = 10;                     // write x
  x = 20;                     // write x again
```  

如果 _x_ 对应的是 _radio transmitter_ 的控制端口的话，那么这可能是代码正在像这个 _ratio_ 发送命令，值 _10_ 所对应的命令和 _20_ 所对应的命令是不同的。优化了第一个赋值语句将会改变发往这个 _ratio_ 的命令的顺序。

_volatile_ 是我们告诉编译器我们正在处理的是特殊内存的方法。对于编译器来说，_volatile_ 的含义是 **_不要对这块内存上的操作执行优化_**。所以，如果 _x_ 对应的是特殊内存的话，它会被声明为 _volatile_：  
```C++
  volatile int x;
```  
考虑这对原来的代码的顺序所产生的影响：  
```C++
  auto y = x;                 // read x
  y = x;                      // read x again (can't be optimized away)

  x = 10;                     // write x (can't be optimized away)
  x = 20;                     // write x again
```
这正是当 _x_ 是 _memory-mapped_ 时或当 _x_ 被映射到一个进程间共享的内存位置上时我们想要做的。
 
突击测验！在最后的代码中，_y_ 的类型是什么：是 _int_ 还是 _volatile int_？

顺便说下，当处理特殊内存时，_redundant load_ 和 _dead store_ 必须被保留，这个事实解释了为什么 _std::atomic_ 对于这种类型的工作是不合适的。编译器被允许去清除 _std::atomic_ 所对应的多余的操作。这个代码不能按照 _volatile_ 那样的方式写，但是如果我们此时忽视这个警告，去关注编译器会被允许做什么的话，那么我们可以说概念上编译器可能接受这样的代码：  
```C++
  std::atomic<int> x;
  
  auto y = x;                 // conceptually read x (see below)
  y = x;                      // conceptually read x again (see below)
  
  x = 10;                     // write x
  x = 20;                     // write x again
```  
并且会优化这个代码为下面这样：  
```C++
  auto y = x;                 // conceptually read x (see below)
  x = 20;                     // write x
```  
对于特殊内存来说，这显然是不可接受的行为。  

现在，就像已经发生的，当 _x_ 是 _std::atomic_ 时，下面这两个语句都不会通过编译：  
```C++
  auto y = x;                 // error!
  y = x;                      // error!
```   
这是因为 _std::atomic_ 所对应的 _copy operation_ 被删除了，见 [_Item 11_](Chapter%203.md#item-11-首选-deleted-function-而不是-private-undefined-function)。理由很充分。如果使用 _x_ 来初始化 _y_ 是可以通过编译的话，那么考虑一下会发生什么。因为 _x_ 是 _std::atomic_，所以 _y_ 的类型也将会被推导为 _std::atomic_，见 [_Item 2_](Chapter%201.md#item-2-理解-auto-的类型推导)。我之前提到过关于 _std::atomic_ 的最好的事之一是它的所有操作都是原子的，为了让根据 _x_ 拷贝构造 _y_ 也成为原子的，编译器必须生成可以在同一个原子操作中进行读 _x_ 和写 _y_ 的代码。硬件通常不可以做这些，所以 _std::atomic_ 类型不支持拷贝构造。_copy assignment operator_ 也是被删除了的，原因相同，这也是为什么 _x = y_ 不可以通过编译。由于没有在 _std::atomic_ 中显式声明 _move operation_，每一个编译器所生成的特殊函数所对应的规则都在 [_Item 17_](Chapter%203.md#item-17-理解特殊成员函数的生成) 进行了描述，_std::atomic_ 没有提供 _move constructor_ 和 _move assignment operator_。

可以读取 _x_ 的值并将值给到 _y_，但是需要使用 _std::atomic_ 的成员函数 _load_ 和 _store_ 来完成。_load_ 成员函数会原子地读取 _std::atomic_ 的值，同时 _store_ 成员函数会原子地写入 _std::atomic_ 的值。先使用 _x_ 来初始化 _y_，然后再在 _y_ 中放入 _x_ 的值，这个代码必须被写成下面这样：  
```C++
  std::atomic<int> y(x.load());         // read x
  
  y.store(x.load());                    // read x again
```  
这个可以通过编译，通过 _x.load()_ 读取 _x_ 和初始化 _y_ 或存储到 _y_ 的是独立的的函数调用，这个事实表明没有理由去期待这两条语句中的任意一条是像单独的原子操作那样做为整体去执行的。

对于下面这样的代码，编译器可以通过在 _register_ 中存储 _x_ 的值而不是读取 _x_ 的值两次来进行 **_优化_**：  
```C++
  register = x.load();                  // read x into register
  
  std::atomic<int> y(register);         // init y with register value
  
  y.store(register);                    // store register value into y
```  

正如你看到的，结果只会从 _x_ 中读取一次，这种优化正是当处理特殊内存时必须被避免的。对于 _volatile_ 变量，这种优化不被允许。

因此情况应该清晰了：  
* _std::atomic_ 用于并发编程，但是不能用于访问特殊内存。
* _volatile_ 用于访问特殊内存，但是不能用于并发编程。

因为 _std::atomic_ 和 _volatile_ 满足于不同的目的，它们甚至可以被一起使用：  
```C++
  volatile std::atomic<int> vai;        // operations on vai are
                                        // atomic and can't be
                                        // optimized away
```  
如果 _vai_ 对应的是 _memory-mapped I/O_ 位置，并且这个位置会被多线程并发访问的话，那么这是有用的。

做为最后的注意事项，即使当 _std::atomic_ 的 _load_ 和 _store_ 成员函数不被需要时，一些开发者也喜欢使用它们，因为这会在代码中明确表明所涉及到的那些变量不是 **_常规_** 的。强调这个事实是合理的。访问 _std::atomic_ 通常是远远慢于访问 _non-std::atomic_ 的，我们已经看到了使用 _std::atomic_ 可以避免编译器去执行特定类型的代码重新排序，而使用 _non-std::atomic_ 是被允许去执行特定类型的代码重新排序的。因此调用 _std::atomic_ 的 _load_ 和 _store_ 有助于识别潜在的可扩展性瓶颈。从正确性的角度看，没有看到那些被用于传递信息给到其他线程的变量，比如：一个表明数据有效性的 _flag_，去调用 _store_ 就可能意味着：这些变量应该被声明为 _std::atomic_，但却没有。

然而，这在很大程度是一个风格问题，因此，这和 _std::atomic_ 和 _volatile_ 之间的选择是完全不同的。

### 需要记住的规则

* _std::atomic_ 用于在没有使用 _mutex_ 的情况下的多线程数据访问。它是一个开发并发软件的工具。
* _volatile_ 用于读写不应该被优化掉的内存。它是一个用于处理特殊内存的工具。