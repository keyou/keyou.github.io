---
title: Chromium 消息循环和线程池 (MessageLoop 和 TaskScheduler)
categories:
  - chromium
  - base
tags:
  - chromium
  - base
---

> Update:
>
> * 2020.6.7: 在最新的 chromium 代码中，`base::TaskScheduler` 被重命名为了 `base::ThreadPool`，功能不变。

Chromium 中的**多线程**机制由 base 库提供，要理解 Chromium 中的多线程机制，首先要理解的概念就是 `base::MessageLoop` 和 `base::TaskScheduler` ，它们两个是 Chromium 多线程的基础。

> 这篇文章重点在设计层面分析 `MessageLoop` 和 `TaskScheduler` ，不会过多讨论他们的各种调用接口，也不会过于强调内部的实现细节。关于使用方式的详细介绍可以参考 [Threading and Tasks in Chrome](https://chromium.googlesource.com/chromium/src/+/refs/tags/62.0.3175.0/docs/threading_and_tasks.md) 和 [Callback<> and Bind()](https://chromium.googlesource.com/chromium/src/+/refs/tags/62.0.3175.0/docs/callback.md)，关于内部实现细节参考公司内部文档 `Chromium 多线程task处理` 。

## MessageLoop

`base::MessageLoop` 代表消息循环，它不会主动创建新的线程，默认情况下它使用当前线程（你也可以手动把它 Bind 到指定的线程上），它只负责消息（任务）循环，它提供了 `task_runner()` 方法用于获取 `TaskRunner` 对象，你需要使用 `TaskRunner::PostTask*()` 方法来向该消息循环分发消息，默认情况下这些消息（任务）会在当前线程运行。因此，你可以在当前线程中创建 `MessageLoop` 并且在当前线程中向它 Post 消息，并且这些消息（任务）会在当前线程执行。

每一个通过`base::Thread`创建出来的线程都拥有一个 `MessageLoop`，但是主线程（main 函数所在的线程）不是通过`base::Thread`来创建的，它又需要处理消息循环，因此需要手动给主线程创建`MessageLoop`，这个过程一般在程序的入口处进行。

你可以使用 `base::MessageLoopCurrent::Get()` 静态方法获取当前线程的`MessageLoop`对象，从而使用 `base::MessageLoopCurrent::Get()->task_runner()→PostTask*()` 方法来创建任务。

一旦你创建了一个 `MessageLoop` 对象，它会自动 Bind 到当前线程（通过线程依赖的 ThreadLocal 机制来实现）。

> 这种每个线程包含一个 `MessageLoop` 的机制称为 `单线程异步多任务` 机制，可以理解为一种设计模式，在很多框架中都会使用，典型的比如网页中 js 的执行虽然可以是异步的，但是都在同一个线程中，还有包括 WPF 的 Dispatcher 机制，Android 的 Looper 机制等。这种机制可以大大简化线程的使用复杂度。一般通用线程池的任务队列也使用这种机制。

常规的使用方式如下：

```c++
#include "base/logging.h"
#include "base/message_loop/message_loop.h"
#include "base/message_loop/message_loop_current.h"
#include "base/task/post_task.h"
#include "base/task/single_thread_task_executor.h"
#include "base/task/thread_pool/thread_pool_impl.h"
#include "base/task/thread_pool/thread_pool_instance.h"
#include "base/threading/thread_task_runner_handle.h"
#include "base/timer/timer.h"

void Hello() {
  LOG(INFO) << "hello,demo!";
}

int main(int argc, char** argv) {
  // 创建消息循环
  base::MessageLoop message_loop;
  // 也可以使用下面的方法。它们的区别仅在于 MessageLoop 对外暴露了更多的内部接口。
  // 在当前线程创建一个可执行 task 的环境，同样需要使用 RunLoop 启动
  // base::SingleThreadTaskExecutor main_task_executer;

  base::RunLoop run_loop;

  // 使用 message_loop 对象直接创建任务
  message_loop.task_runner()->PostTask(FROM_HERE, base::BindOnce(&Hello));
  // 获取当前线程的 task runner
  base::ThreadTaskRunnerHandle::Get()->PostTask(FROM_HERE,
                                                base::BindOnce(&Hello));

  // 启动消息循环，即使没有任务也会阻塞程序运行。当前进程中只有一个线程。
  run_loop.Run();

  return 0;
}
```

### MessageLoop 的运行流程

![messageloop](/data/MessageLoop.svg)

### MessageLoop 的类图

![messageloop](/data/MessageLoop类图.svg)

类图中已经介绍了主要类的功能，这里不再赘述，简单总接一下就是：`MessageLoop` 创建消息/任务循环，并且绑定到当前线程，`RunLoop` 启动消息循环，调用者通过 `TaskRunner` 来创建任务。

> 调试技巧：每一个Post()创建的任务都会被包装进PendingTask类型中，你可以在gdb中通过up命令向上回朔堆栈到TaskAnnotator类中的RunTask()方法，然后可以使用p命令打印pending_task变量，该变量的posted_from属性记录了当前任务的创建者，这在调试异步任务时非常有用。

## TaskScheduler

`base::TaskScheduler` 直译为任务调度器，也可以叫做线程池，你需要使用它的 static 类型的 `Create*()` 相关方法来构造它，使用 `Start()` 方法来启动它，或者通过 `CreateAndStartWithDefaultParams()` 方法来同时创建并启动线程池。默认情况下他会创建 3 个线程，1 个 Service 线程，2 个 Worker 线程。Service 线程**只用来调度延时任务**，Worker 线程用来执行任务。Service 线程继承自 base::Thread ，因此它内部也包含了 MessageLoop（每一个 base::Thread 类创建出来的线程都有一个 MessageLoop）。Worker 线程是 TaskScheduler 直接使用 `PlatformThread::Create*()` 方法创建出来的，因此它不包含 MessageLoop。你可以使用 `base::PostTask*()` 全局方法来向线程池 Post 任务。

```c++
// TaskScheduler的一般用法：

#include <base/logging.h>
#include <base/message_loop/message_loop.h>
#include <base/task/post_task.h>
#include <base/task/task_scheduler/task_scheduler.h>

void Hello()
{
    LOG(INFO)<<"hello,demo!";
}

int main(int argc,char** argv)
{
  // 初始化线程池，会创建新的线程，在新的线程中会创建消息循环 MessageLoop
  base::TaskScheduler::CreateAndStartWithDefaultParams("Demo");

  // 通过以下方法创建任务
  base::PostTask(FROM_HERE, base::BindOnce(&Hello));
  // 或者通过创建新的TaskRunner来创建任务，TaskRunner可以控制任务执行的顺序以及是否在同一个线程中运行
  scoped_refptr<base::TaskRunner> task_runner_ =
    base::CreateTaskRunnerWithTraits({base::TaskPriority::USER_VISIBLE});
  task_runner_->PostTask(FROM_HERE,base::BindOnce(&Hello));

  // 不能使用以下方法创建任务，会导致程序崩溃，因为当前线程没有创建消息循环
  //base::MessageLoopCurrent::Get()->task_runner()->PostTask(FROM_HERE, base::BindOnce(&Hello));

  // 由于线程池默认不会阻塞程序运行，因此这里为了看到结果使用getchar()阻塞主线程。当前进程中共有4个线程，1个主线程，1个线程池Service线程，2个Worker线程。
  getchar();

  return 0;
}
```

通过工具查看以上程序的线程情况如下（主线程没有颜色，所以一共是4个）：

![threads](/data/image2019-6-11_19-19-53.png)

### TaskScheduler 的运行流程

![TaskScheduler1](/data/TaskScheduler流程图.svg)

### TaskScheduler 的类图

![TaskScheduler2](/data/TaskScheduler类图.svg)

## 总接

* 每一个 `base::Thread` 线程都拥有一个 `MessageLoop` 用来进行任务调度；
* 主线程如果需要消息循环，需要自行创建`MessageLoop` ；
* 由于 `MessageLoop` 中维护有 `TaskRunner`，因此你可以通过获取该线程的 `TaskRunner` 来给该线程 Post 任务；
* 线程池使用更底层的 `PlatformThread::Create*()` 来直接创建线程，从而避免每个线程都有 MessageLoop;
* 线程池中的任务是通过 `base::PostTask*()` 创建的；
* 线程池使用名为 `TaskSchedulerSe` 的线程来调度需要延时的任务，不需要延迟的任务会直接放入线程池的任务任务队列；
* 通过 `TaskRunner` 的 `PostTask*()` 方法会将任务Post到 `TaskRunner` 所在的 `MessageLoop`；
* 通过 `base::PostTask*()` 全局方法默认会将任务Post到线程池中；

------------

> 参考文档：
>
> * [Threading and Tasks in Chrome](https://chromium.googlesource.com/chromium/src/+/refs/tags/62.0.3175.0/docs/threading_and_tasks.md)
> * [Callback<> and Bind()](https://chromium.googlesource.com/chromium/src/+/refs/tags/62.0.3175.0/docs/callback.md)
> * [Thread and Task Profiling and Tracking - The Chromium Projects](https://www.chromium.org/developers/threaded-task-tracking)
