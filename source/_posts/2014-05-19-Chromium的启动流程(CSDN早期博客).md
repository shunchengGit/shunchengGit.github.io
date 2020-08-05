---
layout: post_layout
title: Chromium的启动流程(CSDN早期博客)
date: 2014-05-19 00:00:00
location: 北京
pulished: true
abstract: Chromium的启动流程。
---

### 1引言

你点击了桌面上的Chrome图标，一个浏览器窗口出现了，输入网址就可以在Internet世界愉快玩耍。这一切是怎么实现的呢？Chromium这个多进程的程序是如何启动各个进程的呢？浏览器主进程（界面进程）启动了哪些线程？如何启动的呢？这些问题一直萦绕在心头，一起来看看源代码吧。本文主要针对Chromium for Mac的源代码，其它操作系统大同小异。

### 2背景知识

浏览器作为一个应用程序，是以进程的形式运行在操作系统上的。首先，Chromium是一个多进程的应用程序，我们需要了解Chromium的多进程框架。其次，Chromium中的每个进程又是多线程的，因而又需要了解Chromium的线程设计。接下来我们就从进程和线程入手介绍相关背景知识。

#### 2.1多进程框架

Chromium是个多进程程序，每当你点开Chromium图标，默认会启动多个进程。Google之所以这样设计，是怕你打开某个挫B网站而把它的整个浏览器全部弄崩掉。所以它利用现代操作系统多进程不同地址空间这个设计，把它整个浏览器分为多进程，这样即使一个进程崩掉了，也不一定会让整个浏览器崩掉。Chromium进程分为一般分为两类：第一类，Browser进程，它管理tabs和插件进程以及运行主界面UI。第二类，Renderer进程，它利用WebKit布局引擎来解释和布局HTML。Browser是老大，只有一个；Renderer是小弟可以有很多个，每个小弟都能跟老大相互通信，但是小弟之间不能相互通信。老大和小弟们之间的关系如下。

![1](/assets/img/postImage/Chromium的启动流程-CSDN早期博客/1.png)

 从上图可以看出每个Renderer进程都有一个RenderProcess对象，这个对象是用来管理整个Renderer进程的；与之对应的，Browser进程有许多RenderProcessHost对象，每个对象对应一个RenderProcess对象，也即是对应一个Renderer进程。

#### 2.2多线程框架

我们平时写程序也用多线程，其中一个常见的场景是把UI和费时处理过程分离，让他们拥有各自的线程。Chromium中的多线程基本是同理的，为了保证UI的快速响应，防止IO阻塞和费时计算而利用了多线程。但是我们在处理多线程共享资源的时候通常是利用锁，这容易引发各种莫名奇妙的问题，如死锁、优先级反转等。Chromium为了避免这些问题就少用锁，它有一些简单的约定：

1. 线程之间通过传递消息来交流(确切来说是传递任务)；
2. 对象只存在在一个线程上。

BrowserProcess类是Browser进程的服务管理者，它控制着大多数的线程。通常程序运行在UI线程，但是我们把某些类型的处理放到其他线程。这些线程如下：


 ui_thread：应用程序启动时的主线程。
 
 io_thread：这个线程的名字有点令人费解。它是处理browser进程和所有子进程之间通信的调度线程。它也负责所有的资源请求（网页加载）的分发（见多进程架构）。
 
file_thread：文件操作的通用线程。当你想执行阻塞文件系统的操作（例如，请求某种文件一个图标，把下载文件写到磁盘），会调用该线程。

 db_thread：数据库操作的线程。例如，Cookie服务在这个线程上使用SQLite操作。注意，历史数据库不使用该线程。
 
本文也主要是讨论Browser上的线程何时启动的。更多关于Chromium进程和线程框架请参考：
[http://www.chromium.org/developers/design-documents/multi-process-architecture](http://www.chromium.org/developers/design-documents/multi-process-architecture)

[http://www.chromium.org/developers/design-documents/threading](http://www.chromium.org/developers/design-documents/threading)

### 3Browser多线程的创建和启动

 双击浏览器图标，我们启动了一个Chromium进程，这就是Browser进程。用C++编写的程序启动函数是main，Chromium当然也不例外。具体流程如下图所示:

![2](/assets/img/postImage/Chromium的启动流程-CSDN早期博客/2.png)

其中,RunNamedProcessTypeMain是个很重要的函数
该函数定义于"content/public/app/content_main_runner.cc"文件。

函数原型如下：

```cpp
int RunNamedProcessTypeMain(
    const std::string& process_type,
    const MainFunctionParams& main_function_params,
    ContentMainDelegate* delegate)
```

该函数内部定义了个静态结构体数组：

```cpp
  static const MainFunction kMainFunctions[] = {
#if !defined(CHROME_MULTIPLE_DLL_CHILD)
    { "",                            BrowserMain },
#endif
#if !defined(CHROME_MULTIPLE_DLL_BROWSER)
#if defined(ENABLE_PLUGINS)
    { switches::kPluginProcess,      PluginMain },
    { switches::kWorkerProcess,      WorkerMain },
    { switches::kPpapiPluginProcess, PpapiPluginMain },
    { switches::kPpapiBrokerProcess, PpapiBrokerMain },
#endif  // ENABLE_PLUGINS
    { switches::kUtilityProcess,     UtilityMain },
    { switches::kRendererProcess,    RendererMain },
    { switches::kGpuProcess,         GpuMain },
#endif  // !CHROME_MULTIPLE_DLL_BROWSER
  };
```

该结构体有两个成员，一个是`const char process_type[]`表示进程类型，它从启动进程传入的参数解析得到；另一个是函数指针，表示该进程对应的主函数。对于Browser进程而言，它的启动参数是空的，解析出来`process_type`也是空的。我们从上面的结构体数组可以看到，空参数表示该进程的主函数是`BrowserMain`。我们再来看看`BrowserMain`中做了啥。主要代码如下所示：

```cpp
scoped_ptr<BrowserMainRunner> main_runner(BrowserMainRunner::Create());

int exit_code = main_runner->Initialize(parameters);
if (exit_code >= 0)
return exit_code;

exit_code = main_runner->Run();
```

我们可以看出该函数主要是创建了一个BrowserMainRunner对象，然后Initialize，紧接着Run。BrowserMainRunner的相关类图如下所示。

![3](/assets/img/postImage/Chromium的启动流程-CSDN早期博客/3.png)

从类图可以看出BrowserMainRunner是个接口类，该类包含3个接口方法：Initialize、Run、Shutdown。其实现类是BrowserMainRunnerImpl。该类管理了一个BrowserMainLoop对象，通过控制BrowserMainLoop来控制该进程。我们可以发现BrowserMainLoop对象管理了许多线程对象，如：`main­­_thread`、`db_thread` 、`file_thread`等。这就是Browser进程中的主要线程对象。相关序列图如下图所示。

![4](/assets/img/postImage/Chromium的启动流程-CSDN早期博客/4.bmp)

main_runner在Initialize的时候创造了BrowserMainLoop对象，并调用了该对象的Initialize函数和CreateStartupTasks函数。而CreateStartupTask调用的时候创建了一个StartupTaskRunner对象，利用该对象来管理Startup Tasks，比如添加任务、执行任务。在此函数中就利用了AddTask来添加任务，其中一个重要的任务是CreateThreads，即把所有他管理的线程对象创建起来。该函数具体如下。
     
```cpp
void BrowserMainLoop::CreateStartupTasks() {
    //other code
    StartupTask pre_create_threads =
        base::Bind(&BrowserMainLoop::PreCreateThreads, base::Unretained(this));
    startup_task_runner_->AddTask(pre_create_threads);

    StartupTask create_threads =
        base::Bind(&BrowserMainLoop::CreateThreads, base::Unretained(this));
    startup_task_runner_->AddTask(create_threads);

    StartupTask browser_thread_started = base::Bind(
        &BrowserMainLoop::BrowserThreadsStarted, base::Unretained(this));
    startup_task_runner_->AddTask(browser_thread_started);

    StartupTask pre_main_message_loop_run = base::Bind(
        &BrowserMainLoop::PreMainMessageLoopRun, base::Unretained(this));
    startup_task_runner_->AddTask(pre_main_message_loop_run);

    startup_task_runner_->RunAllTasksNow();
}
```

以上就是Browser各种主要线程创建的时机。

### 4多进程的创建和启动

上文我们已经提到，Chromium是一种多进程的程序，那除了双击启动的Browser进程，其它的小弟进程Renderer在哪里启动的呢？通过上文，我们知道每个Renderer都有个RenderProcess对象，相应的Browser上有于之对应的RenderProcessHost对象。追踪源码我们可以发现，在RenderProcessHostImpl类的Init函数中，有这样几行代码：

```cpp
child_process_launcher_.reset(new ChildProcessLauncher(
#if defined(OS_WIN)
        new RendererSandboxedProcessLauncherDelegate,
#elif defined(OS_POSIX)
        renderer_prefix.empty(),
        base::EnvironmentMap(),
        channel_->TakeClientFileDescriptor(),
#endif
        cmd_line,
        GetID(),
        this));
```

该函数new了一个ChildProcessLauncher对象，再看看对应的构造函数，里面有如下代码：

```cpp
context_ = new Context();
  context_->Launch(
#if defined(OS_WIN)
      delegate,
#elif defined(OS_ANDROID)
      ipcfd,
#elif defined(OS_POSIX)
      use_zygote,
      environ,
      ipcfd,
#endif
      cmd_line,
      child_process_id,
      client);
```

Launch函数的内部又做了什么呢？代码如下：

```cpp
BrowserThread::PostTask(
    BrowserThread::PROCESS_LAUNCHER, FROM_HERE,
    base::Bind(
        &Context::LaunchInternal,
        make_scoped_refptr(this),
        client_thread_id_,
        child_process_id,
        use_zygote,
        environ,
        ipcfd,
        cmd_line));
```

可以看出它PostTask到BrowserThread::PROCESS_LAUNCHER，让其执行Context ::LaunchInternal任务。再来看看这个函数，里面有一行：

```cpp
bool launched = base::LaunchProcess(*cmd_line, options, &handle);
```

这个函数调用fork()然后execvp ()启动了子进程。子进程会执行上文的RunNamedProcessTypeMain函数，但是这次它是带执行参数的，所以不会进入BrowserMain函数，而是进入其它函数，如：RendererMain等。以上就是子进程的启动流程了。

### 5总结

Browser多线程启动

1. Browser主函数是BrowserMain，该函数创建了一个BrowserMainRunnerImpl对象，同时执行了该对象的Initialize函数；
2. BrowserMainRunnerImpl的Initialize创建了一个BrowserMainLoop对象，同时执行了该对象的Init和CreateStartupTasks函数；
3. BrowserMainLoop的CreateStartupTasks添加了一堆任务，其中一个任务就是CreateThreads函数；
4. 在CreateThreads函数中会初始化各种线程对象，并且StartWithOptions。

Chromium多进程

1. 双击Chromium图标启动第一个进程，即Browser进程；
2. Browser进程中RenderProcessHostImpl对象的Init函数会new一个ChildProcessLauncher对象，该对象来管理子进程的启动；
3. ChildProcessLauncher构造过程中会借助其Context对象，Post一个Task到PROCESS_LAUNCHER线程；
4. Task会执行base::LaunchProcess,该函数调用fork()然后execvp ()启动了子进程；
5. 启动的子进程是带参数的，这次在执行RunNamedProcessTypeMain的时候不会进入BrowserMain，而是进入其对应的主函数。
 