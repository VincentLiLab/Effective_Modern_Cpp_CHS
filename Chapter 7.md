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
在这个调用中，被传递给 _std::async_ 的函数对象被认为是一个 _task_。

_task-based_ 方法通常是优于 _thread-basd_ 方法的，我们看到的代码已经描述展示了一些原因。此处，_doAsyncWork_ 会生成一个返回值，我们有理由假设调用 _doAsyncWork_ 的代码是对这个返回值感兴趣的。当使用 _thread-based_ 方法时，没有简单的方法获取到这个返回值。当使用 _task-based_ 方法时，就简单了，因为 _std::async_ 所返回的 _future_ 提供了 _get_ 函数。如果 _doAsyncWork_ 会抛出一个异常的话，那么 _get_ 函数就更重要了，因为 _get_ 函数也可以访问这个异常。当使用 _thread-based_ 方法时，如果 _doAsyncWork_ 会抛出异常的话，那么程序会挂掉，这是通过调用 _std::terminate_ 来完成的。

_thread-based_ 编程和 _task-based_ 编程之间更根本的区别是 _task-based_ 表现出了更高级别的抽象。这让你可以从线程管理的细节中解脱出来，一个观察让我想起了我需要总结并发 _C++_ 软件中的 **_线程_** 的三个含义：  
* 硬件线程是实际执行计算的线程。现代的机器架构在每个 _CPU_ 上都一个或多个硬件线程。
* 软件线程，也被称为 _OS_ 线程或系统线程，是操作系统在所有进程间管理并在硬件线程上调度执行的线程。通常，创建的软件线程可以比硬件线程多，因为，当一个软件线程被阻塞时，比如：等待 _I/O_ 或等待 _mutex_ 或 _condition variable_，可以通过执行其他非阻塞的线程来被提高吞吐率。
* _std::thread_ 是 _C++_ 进程中的对象，用来做为 _underlying_ 软件线程的 _handle_。一些 _std::thread_ 对象可以表现为 **_null_** _handle_，即为：没有所对应的软件线程，因为它们可能处在 _default-constructed_ 状态下，此时没有可以执行的函数、因为它们可能已经被移动了，此时所被移动到的 _std::thread_ 做为了 _underlying_ 软件线程的 _handle_、因为它们可能已经被 _join_ 了，此时所运行的函数已经结束了以及它们可能已经被 _detach_ 了，此时它们和它们所对应的 _underlying_ 软件线程之间的连接已经被分离了。

软件线程是一种有限资源。_如果你尝试创建的多于系统可以提供的话，_std::system_error_ 异常会被抛出。即使你要运行的函数不可以抛出异常，也是这样的。例如：即使 _doAsyncWork_ 是 _noexcept_ 的，  
```C++
  int doAsyncWork() noexcept;           // see Item 14 for noexcept
```  
下面这个语句也可能会导致一个异常：  
```C++
  std::thread t(doAsyncWork);           // throws if no more
                                        // threads are available
```

友好设计的软件必须以某种方式来处理这种可能性，但是如何呢？一种方法是在当前线程中运行 _doAsyncWork_，但是这可能会导致平衡的负载，而且如果当前线程是 _GUI_ 线程的话，那么会产生响应性问题。另一个选项是等待一些现存的软件线程结束，然后再去尝试创建新的 _std::thread_，但是那些现存的线程有可能正在等待这个 _doAsyncWork_ 所要执行的动作，比如：产生一个结果或者通知一个 _condition variable_。

即使你没有耗尽线程，你也可以遇到 _oversubscription_ 的问题。也就是当 _ready-to-run_ 的软件线程，比如：非阻塞的软件线程，多于硬件线程时。当这些发生时，通常是做为 _OS_ 一部分的线程调度器会对硬件上的软件线程进行 _time-slice_。当一个线程的 _time-slice_ 结束，另一个线程开始时，上下文切换会被执行。这些上下文切换增加了系统线程管理的整体负载，当调度某个软件线程的硬件线程所在的 _CPU_ 和这个软件线程在上一个 _time-slice_ 期间所对应的硬件线程所在的 _CPU_ 是不相同时，这些负载会尤其大。在这个场景下，（1）那个软件线程所对应的 _CPU_ 缓存通常是 _cold_，即为：这个 _CPU_ 缓存只包含少量对它是有用的数据和指令；（2）在这个 _CPU_ 上所运行 **_新_** 软件线程会 **_污染_** **_旧_** 软件线程所对应的 _CPU_ 缓存，这个 **_旧_** 软件线程已经在这个 _CPU_ 上运行过了，而且可能会再一次运行在这个 _CPU_ 上。 

避免 _oversubscription_ 是困难的，因为软件线程与硬件线程的最佳比例是依赖于软件线程变成可运行状态的频繁程度的，这是可以动态改变的，比如：当程序从 _IO_ 密集型成为计算密集型时。软件线程与硬件线程的最佳比例还依赖于上下文切换的成本和软件线程使用 _CPU_ 缓存的效率。此外，硬件线程的数量和 _CPU_ 缓存的细节，比如：_CPU_ 缓存的大小和相对速度，都依赖于硬件架构，所以，即使你调整了你的应用程序，避免了平台上的 _oversubscription_，但如果此时硬件仍然是繁忙的话，那么仍然不能保证你的方案在其他类型的平台是好的。

如果你可以把这些问题推给别人的话，那么你的生活就轻松了，使用 _std::async_ 正是做的这些：  
```C++
  auto fut = std::async(doAsyncWork);             // onus of thread mgmt is
                                                  // on implementer of
                                                  // the Standard Library
```

这个调用将线程管理的责任转换给了 _C++_ 标准库的实现者。例如：接收到 _out-of-thread_ 异常的可能性显著降低了，因为这个调用可能永远不会产生 _out-of-thread_ 异常。你可能好奇，为什么呢？如果我要求的软件线程多于系统可以提供的软件线程的话，那么为什么这和是通过创建 _std::thread_ 来完成的还是通过创建 _std::async_ 来完成的会有关系呢？是有关系的，因为当 _std::async_ 按照这样的形式被调用时，即为：使用默认 _launch_ 策略时，见 [_Item 36_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)，它并不保证会创建出一个软件线程。它宁愿允许调度器将所指定的函数在这个例子中是 _doAsyncWork_ 安排到那个需要  _doAsyncWork_ 的结果的线程上，即为调用 _fut.get_ 或 _fut.wait_ 的线程上，如果系统是 _oversubscription_ 了或者是线程耗尽了的话，那么合理的调度器会利用这个自由。

在需要它的结果的线程上来运行它，如果你利用了这个技巧的话，那么我提醒你它可能会导致负载均衡的问题，而且这个问题不会轻易地消失，因为面对它们的是 _std::async_ 和运行调度器而不是你。然而，当谈到负载均衡时，关于在硬件上发生的事情，运行调度器可能比你了解的更全面，因为运行调度器管理着的是所有进程上的线程，而不是只有你的代码里所运行着的线程。

当使用 _std::async_ 时，_GUI_ 线程的响应性问题仍然存在，因为调度器没有方法知道你的哪个线程有严格的响应要求。在这种场景下，你需要将 _std::launch::async_ 的 _launch_ 策略给到 _std::async_。这将会确保你想要运行的函数真正地执行在一个不同的线程上，见 [_Item 36_](#item-36-如果异步是必须要的话那么指明-stdlaunchasync)。

_state-of-the-art_ 线程调度器利用了 _system-wide_ 线程池来避免 _oversubscription_，它通过 _work-stealing_ 算法提升了硬件核心之间的负载均衡。_C++_ 标准不需要使用线程池或 _work-stealing_，老实说，_C++11_ 的并发规范的一些技术特色使得利用这个技术变得很困难，比我们想象的要困难。然而，一些开发商在它们的标准库实现中利用了这个技术，我们有理由期待在这方面继续取得进展。如果你在你的并发开发中采用了 _task-based_ 方法的话，那么当这个技术变得更加普遍的时候，你就自动获得了这个技术的益处了。另一方面，如果你直接使用了 _std::thread_ 的话，你就要自己承担处理线程耗尽、_oversubscription_ 和负载均衡的责任，更不用说你的这些问题的解决方法如何与运行在同一台机器上的其他进程中的程序中所实现的方案所协调。

## _Item 36_ 如果异步是必须要的话，那么指明 _std::launch::async_

## _Item 39_ 对于 _one-shot event_ 通信考虑 _void future_ 

## _Item_ 40 并发使用 _std::atomic_ 特殊内存使用 _volatile_