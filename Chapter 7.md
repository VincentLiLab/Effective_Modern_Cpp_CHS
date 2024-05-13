- [_Chapter 7_ _concurrency API_](#chapter-7-concurrency-api)
  - [_Item 35_ 首选 _task-based_ 编程而不是 _thread-based_ 编程](#item-35-首选-task-based-编程而不是-thread-based-编程)
    - [需要记住的规则](#需要记住的规则)
  - [_Item 36_ 如果异步是必须要的话，那么指明 _std::launch::async_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)
    - [需要记住的规则](#需要记住的规则-1)
  - [_Item 39_ 对于 _one-shot event_ 通信考虑 _void future_](#item-39-对于-one-shot-event-通信考虑-void-future)
  - [_Item 40_ 并发使用 _std::atomic_ 特殊内存使用 _volatile_](#item-40-并发使用-stdatomic-特殊内存使用-volatile)


# _Chapter 7_ _concurrency API_

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
* _The std::launch::deferred launch policy_ 意味着：只有当 _std::async_ 所返回的 _future_ 的 _get_ 或 _wait_ 被调用时，_f_ 才可以运行。也就是说 _f_ 的执行会被推迟到所对应的 _get_ 或 _wait_ 被执行时。当所对应的 _get_ 或 _wait_ 被调用时，_f_ 将会同步执行，即为：调用发将会阻塞，直到 _f_ 结束运行。如果 _get_ 和 _wait_ 都没被调用时，_f_ 将永远不会运行。

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

## _Item 39_ 对于 _one-shot event_ 通信考虑 _void future_ 

## _Item 40_ 并发使用 _std::atomic_ 特殊内存使用 _volatile_