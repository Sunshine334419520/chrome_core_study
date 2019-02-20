## Chrome内核研究

### Chrome和Chromium介绍

#### 什么是chromium?
Chromium是一个建立在WebKit之上的浏览器开源项目，由Google发起的。该项目被创建以来发展迅速，很多先进的技术被采用，如跨进程模型，沙箱模型等等。同时，很多新的规范被支持，例如WebGL，Canvas2D，CSS3以及其他很多的HTML5特性，基本上每天你都可以看到它的变化，它的版本升级很快。在性能方面，其也备受称赞，包括快速启动，网页加载迅速等。
#### 什么是chrome?
Chrome是Google公司的浏览器产品，它基于chromium开源项目，一般选择稳定的版本作为它的基础，它和chromium的不同点在于chromium是开源试验场，会尝试很多新的东西，当这些东西稳定之后，chrome才会集成进来，这也就是说chrome的版本会落后于chromium。另外一个就是，chrome里面会加入一些私有的codec，这些仅在chrome中才会出现。再次，chrome还会整合Google的很多服务， 最后chrome还会有自动更新的功能，这也是chromium所没有的。


### 浏览器简述

#### 浏览器内核
浏览器内核当然就是浏览器最要的部分，浏览器最要或者说核心的部分是"Rednering Engine"(渲染引擎),这个渲染引擎就是浏览器的内核，它负责对网页的语法进行解释(如HTML，JavaScript)然后渲染并且显示出来，这个渲染引擎也就决定了如何显示网页的内容以及页面的格式以及页面的排榜.
#### 浏览器内核分类
比如trident(IE内核)、Gecko(Firefox内核)、Webkit(Safari内核，Chrome内核原型, 开源)、Blink(Chromium内核, 它是基于Webkit改造而来)

### Chromium多线程
Chromium是一个基于多进程模型的架构设计,而且每个进程里面也有很多的线程，特别是主进程(Browser进程),其中Browser进程分为UI线程IO线程以及DB线程(数据库),FILE线程等，在这里只列出以上重要的线程， 为什么要搞这么多线程？chrome官方给出的解释是为了UI能够快速等响应，不被一些费时等操作影响了用户的体验，比如file读写，socket读写，数据库读写. 那么chrome是怎么管理这么多线程的呢？这个问题就是下面我要说的，chrome的线程消息通信机制.
#### Chromium消息通信
现在我不仅仅只是介绍Chromium线程之间的消息通信机制，主要是要深入Chromium源码，测地的了解Chromium是如何实现消息通信，首先我还是介绍一下Chromium的消息通信原理,Chromium每一个线程都会有一个MessageLoop,这个MessageLoop就是用来处理与自己绑定的线程的任务.
#### CHromium多线程深度研究
现在来看看Chromium源码中是如何定义一个基础的Thread(首先声明,整个的代码并不是直接copy Chromium源码，而是右我自己参考它实现的代码，相比Chromium更加清晰，去除了一些没有用的代码，并且都是采用C++11特性，由于Chromium内部实现了很大属于他自己的东西，比如Callback，Task)

首先来看chromium多线程的启动过程，从下图可一看到, 这个启动流程从BrowserThread 开始,所有的Browser进程里面的线程都是基于这个BrwoserThread类实现或者重定义的.接下来我会按照下图的流程一个一个函数的进行分析.
##### chromium thread 启动流程

![Markdown](https://farm5.staticflickr.com/4848/39745484663_5b4f1d2dd8_k.jpg)

先来看看BrowserThread类的定义

```c++
class BrowserThread : public Thread {
 public:
	// ......
	bool Start() override;
	bool StartWithOptions(const Options& options) override;
	// ......
}
```
从BrowserThread类的定义可以看出它继承了Thread类并且重写了Start和StartWithOptions这两个方法. 下面看Start 和 StartWithOptions的实现

```c++ 
bool BrowserThread::Start() {
	// ......
	Options options;
	return StartWithOptions(options);
}
```

这个Start函数几乎什么都没有干，只是定义了一个Options结构然后调用了StartWithOptions.先来看看Options结构

```c++
struct BASE_EXPORT Options {
    typedef std::function<std::unique_ptr<
        MessagePump>()> MessagePumpFactory;
    Options();
    // 这个参数保存者真正做消息循环处理的地方.	 
    MessagePumpFactory message_pump_factory;

    // 线程stack大小
    size_t stack_size = 0;

    // 线程的优先级
    ThreadPriority priority = ThreadPriority::NORMAL;

    bool joinable = true;
}; 
```

这个Options也是非常的简单，主要保存了一些线程的信息，比如这个线程的消息循环处理类的Factory，和线程的stack size,以及线程的优先级，以及线程是否可以join.那现在对于我们来说最重要的应该是StartWithOption, 来看代码

```c++
bool BrowserThread::StartWithOptions(const Options& options) {
	// ......
	return Thread::StartWithOptions(options);
}
```

这个StartWithOption调用了Thread的StartWithOptions, Thread的定义再下面.


```c++    
class BASE_EXPORT Thread : PlatformThread::Delegate {
 public:
  // 中间有一些代码，但是现在对于我们来说并不重要....

  // 开始线程, 这个函数会调用到StartWithOptions.
  bool Start();
  // 停止线程.
  void Stop();
  
  // 省略大量代码...
 private:
  // 省略大量代码...

  // PlatformThread::Delegate methods:
  void ThreadMain() override;

  bool stopping_ = false;
  // The thread's handle.
  PlatformThreadHandle thread_;
  // The thread's id once it has started.
  PlatformThreadId id_ = kInvalidThreadId;

  // 省略大量代码...

  // The thread's MessageLoop and RunLoop. Valid only while the thread is alive.
  // Set by the created thread.
  MessageLoop* message_loop_ = nullptr;
  RunLoop* run_loop_ = nullptr;
};
```

由上面的代码你明显可以看到Thread继承了PlatformThread::Delegate, 呆会我就会带大家去看看这个，并且在Thread类里面我们也明显的看到了MessageLoop这个消息循环，在之前我们说够每一个Chromium线程都会包含一个MessageLoop，现在看来是对的，但是现在我并不讨论这个消息循环，而是来看StartWithOptions函数，这个函数明显就是Chromium中每个线程的启动函数,现在看看这个函数里面做了什么

```c++
bool Thread::StartWithOptions(const Options & options) {

	DCHECK(!message_loop_);
	DCHECK(!IsRunning());
	DCHECK(!stopping_);
	DCHECK(!is_thread_valid_);

	id_ = kInvalidThreadId;

	SetThreadWasQuitProperly(false);
	MessageLoop::Type type = options.message_loop_type;
	if (options.message_pump_factory)
		type = MessageLoop::TYPE_CUSTOM;

	message_loop_timer_slack_ = options.timer_slack;
	std::unique_ptr<MessageLoop> message_loop_owned =
		MessageLoop::CreateUnbound(type, options.message_pump_factory);
	message_loop_ = message_loop_owned.get();

	{
		std::lock_guard<std::mutex> lock(thread_mutex_);
		thread_ = options.joinable
			? PlatformThread::CreateWithPriority(options.stack_size,
												 this, options.priority)
			: PlatformThread::CreateNonJoinableWithPriority(
				options.stack_size, this, options.priority);
		is_thread_valid_ = true;
	}

	joinable_ = options.joinable;
	
	ignore_result(message_loop_owned_.release());

	DCHECK(message_loop_);
	return true;
}
```
我们可以看到这个函数一开始会做大量的DCHECK，DCHECK其实也就是Debug模式下进行一系列的assert，只有里面的条件为true时，才能接着执行.比如一开始的DCHECK(!message_loop_)意味者message_loop_必须为NULL，然后就是对options进行赋值,比如消息类型..., 之后创建了一个MessageLoop，这个MessageLoop我们之后会深入的讨论，现在我们只需要知道在开始线程之前，为这个线程创建了一个属于它的消息循环，之后重要的函数CreateWithPriority, CreateNonJoinableWithPriority(从函数名就可以看出这个函数是创建一个detch线程),在发现joinable为true时会调用CreateWithPriority().那么现在我们来看看CreateWithPriority函数
```c++
std::thread 
PlatformThread::CreateWithPriority(size_t stack_size, 
								   Delegate * delegate,
								   ThreadPriority priority) {
	return std::move(CreateThread(stack_size, true,
					 delegate, priority));
}
```
这个函数非常简单,只是调用CreateThread.
```c++
std::thread CreateThread(size_t stack_size,
                         bool joinable,
                         PlatformThread::Delegate* delegate,
                         ThreadPriority priority) {
	std::unique_ptr<ThreadParams> params(new ThreadParams);
	params->delegate = delegate;
	params->joinable = joinable;
	params->priority = priority;

	std::thread th(&ThreadFunc, params.get());
	
	if (!params->joinable)
		th.detach();
	return std::move(th);
}
```

这个函数就是最重要的部分了，首先创建了一个params，然后给这个params里面的参数赋值

```c++
struct ThreadParams {
	ThreadParams()
		: delegate(nullptr),
		  joinable(false),
		  priority(ThreadPriority::NORMAL) {}
		
	PlatformThread::Delegate* delegate;
	bool joinable;
	ThreadPriority priority;
};
```

ThreadParams结构保存着需要传递给thread的参数,其中最重要的是delegate,在Thread类的StartWithOptions函数调用的时候将this传递给了delegate，那现在这个delegate就是指向Thread类的一个指针. 接着上面的CreateThread函数说，在初始化params之后就创建了一个线程(线程函数ThreadFunc， 线程参数params.get())

```c++
void* ThreadFunc(void* params) {
	PlatformThread::Delegate* delegate = nullptr;

	{
		std::unique_ptr<ThreadParams> thread_params(
			static_cast<ThreadParams*>(params));

		delegate = thread_params->delegate;
	}
	
	delegate->ThreadMain();

	return nullptr;
}
```

可以看到ThreadFunc仅仅只是调用了一下delegate->ThreadMain()，之前有分析过，这个delegate其实就是指向Thread的，来看一下Thread::ThreadMain()

```c++
void Thread::ThreadMain() {
	// .......
	RunLoop run_loop;
	run_loop_ = &run_loop;
	Run(run_loop_);

	// .......
}
```

这个ThradMain会先创建一个RunLoop，这个RunLoop其实是一个帮助类，专门帮助chromium thread 运行消息循环的类, 然后调用了Run()函数

```c++
void Thread::Run(RunLoop * run_loop) {
	// ......
	run_loop_->Run();
}
```
```c++
void RunLoop::Run() {
	// ......

	const bool application_tasks_allowed =
		delegate_->active_run_loops_.size() == 1U ||
		type_ == Type::kNestableTasksAllowed;
	delegate_->Run(application_tasks_allowed);

	// ......
}
```

从之前的图中就可以看导这个delegate_->Run() 调用的是Run::Delegate::Run, 这个delegate是一个多态，由于MessageLoop(这个在之后讲消息循环的时候仔细讲)继承了Run::Delegate，并且重写了Run方法
所以最终这个delegate_->Run是调用到MessageLoop->Run函数，这个MessageLoop就是消息循环类, 每一个线程都回有一个消息循环，这个MessageLoop就是来管理线程中的每一个消息与事件, 在MessageLoop的Run方法中又调用了MessageLoopDefault的Run方法，所以最终走到了MessageLoopDefault::Run

```c++
void MessagePumpDefault::Run(Delegate* delegate) {
	for (;;) {
		bool did_work = delegate->DoWork();
		if (!keep_running_)
			break;

		did_work |= delegate->DoDelayedWork(delayed_work_time_);
		if (!keep_running_)
			break;

		if (did_work)
			// 作了延迟任务或者work.
			continue;

		// 没有做工作，就去做闲置的工作
		did_work = delegate->DoIdleWork();
		if (!keep_running_)
			break;

		// 做了闲置工作，contiune.
		if (did_work)
			continue;

		// ......
	}
}
```

走到这里整个chromium thread的启动大概流程就已成完成了，之后就会在这个for循环里无限的等待任务传来，然后处理任务.

##### chromium 线程间的任务传递
上面分析了线程的启动过程，接下了分析chromium如何再线程间传递任务, 一开始介绍chromium多线程时就说到过chromium有许多线程，chromium是如何再线程之间传递数据，以及处理数据，并且做到线程同步? chromium多线程遵循一个原则就是尽量避免锁的使用，线程之间尽量不要使用共享数据, 而是在线程中传递数据(Task), 传递的数据用Task代替，线程之间传递任务来达到线程之间的通信，如下图 ： 线程A发送一个Task到线程B.

1.  首先线程A调用PostTaskf发送一个Task到线程B
2.  此时线程B中有一个incoming queue用来接收这个任务
3.  接收到这个任务之后，线程B中的消息循环(每个线程都会有一个消息循环，这个之后会分析), 消息循环取出任务并且执行
4.  线程B封装一个Reply消息发送给线程A(大部分消息不需要这一步)
##### chromium thread message loop 研究
  > chromium 消息循环核心类

MessagePump::Delete : 这个类是一个纯基类，其中有3个方法，对应着不同任务的执行

MessageLoop : 这个类是chromium消息循环的主要的类，也是主消息循环类，chromium的消息类型分为3种(自定义消息，UI消息，IO消息)，其中MessageLoopForUI处理UI消息，MessageLoopForIO处理IO消息，MessageLoop可以处理自定义消息，在这里我们步分析MessageLoop的子类, 从图中可以看出MessageLoop继承了MessagePump::Delete和RunLoop::Delete

RunLoop : RunLoop这个类用来管理消息循环的运行以及停止，也是一个非常重要的类，里面包括的一个Delete类.

MessagePump : 一个虚基类，可以通过继承他实现不同的消息循环有不同的运行体.
![Markdown](https://farm8.staticflickr.com/7870/33215856348_fac437bb49_b.jpg)

作为一个消息循环，当然有一个用于存放任务的队列，chromium利用IncomingQueue来作为消息循环的队列

```c++
class IncomingTaskQueue {
 public:
		// .......

	 // 添加一个任务到incoming queue. 所有的任务都需要通过AddToIncomingQueue() or
	 // TryAddToIncomingQueue()，多个不同的线程可以同时提交.
	 // 如果成功返回true, 否则放回false，任务的所有权会被转移到调用的方法.
	 bool AddToIncomingQueue(const Location& from_here,
							 OnceClosure task,
							 std::chrono::milliseconds delay,
							 Nestable nestable);
		// ......
 private:

	 // 下面这三个队列，都是用于message loop中保存需要运行的消息的对应队列.
	 // 一个保存着普通的任务队列, 一个保存着延迟任务队列, 一个保存着闲置任务队列.
	 class TriageQueue : public ReadAndRemoveOnlyQueue {
	  public:
		  TriageQueue(IncomingTaskQueue* outer);
		  ~TriageQueue() OVERRIDE;

		  // ReadAndRemoveOnlyQueue:
		  // 如果队列时空的(调用clear除外), 那么下列方法就会从incoming 队列从新加载.
		  const PendingTask& Peek() OVERRIDE;
		  PendingTask Pop() OVERRIDE;

		  bool HasTasks() OVERRIDE;
		  void Clear() OVERRIDE;

	  private:
		  // 如果队列为空从incoming queue 重新加载.
		  void ReloadFromIncomingQueueIfEmpty();

		  IncomingTaskQueue* const outer_;
		  TaskQueue queue_;

		  DISALLOW_COPY_AND_ASSIGN(TriageQueue);
	 };

	 class DelayedQueue : public Queue {
	  public:
		  DelayedQueue(IncomingTaskQueue* outer);
		  ~DelayedQueue() OVERRIDE;

		  // Queue:
		  const PendingTask& Peek() OVERRIDE;
		  PendingTask Pop() OVERRIDE;

		  bool HasTasks() OVERRIDE;
		  void Clear() OVERRIDE;
		  void Push(PendingTask pending_task) OVERRIDE;

	  private:
		  IncomingTaskQueue* const outer_;
		  DelayedTaskQueue queue_;

		  DISALLOW_COPY_AND_ASSIGN(DelayedQueue);
	 };

	 class DeferredQueue : public Queue {
	  public:
		  DeferredQueue(IncomingTaskQueue* outer);
		  ~DeferredQueue() OVERRIDE;

		  // Queue:
		  const PendingTask& Peek() OVERRIDE;
		  PendingTask Pop() OVERRIDE;

		  bool HasTasks() OVERRIDE;
		  void Clear() OVERRIDE;
		  void Push(PendingTask pending_task) OVERRIDE;
		  
	  private:
		  IncomingTaskQueue* const outer_;
		  TaskQueue queue_;

		  DISALLOW_COPY_AND_ASSIGN(DeferredQueue);
	 };

	 

	 // 添加一个任务到 incoming queue. 调用者可以继续保持这个pending_task,
	 // 但是这个函数将会重置pending_task->task的值，因为这个函数需要保证这个
	 // pending_task->task的生命周期不会超过这个函数.
	 bool PostPendingTask(PendingTask* pending_task);

	 // 这个函数作真正的posting a pending task, 如果返回true，这个调用一你应该在
	 // 这个message loop 上面调用ScheduleWork() .
	 bool PostPendingTaskLockRequired(PendingTask* pending_task);


	 TriageQueue triage_tasks_;

	 DelayedQueue delayed_tasks_;

	 DeferredQueue deferred_tasks_;

	 // 这个队列里面保存的任务是还没有放到message loop 中的.
	 TaskQueue incoming_queue_;

	// ......
};
```
这个类有点大，先来看它的数据成员，其中incoming_queue_就是这个类的核心数据结构(TaskQueue就是c++ stl中的优先队列), 这个incoming_queue保存着所有的从别的线程发送到本线程消息循环中的任务，并且消息循环中不是直接从incoming_queue中取任务执行，而是从三个辅助队列中取数据这三个队列分别对应三种不同的任务，triage_tasks_(普通任务)，delay(延迟任务), deferred_tasks_(闲置任务)

InComingQueue的接口AddToIncomingQueue明显是这个类的核心接口，这个函数的作用就是将任务添加到IncomingQueeu，来看一下实现

```c++
bool IncomingTaskQueue::AddToIncomingQueue(const Location & from_here,
										   OnceClosure task,
										   std::chrono::milliseconds delay,
										   Nestable nestable) {
	CHECK(!task.is_null());

	PendingTask pending_task(from_here, std::move(task),
							 CalculateDelayedRuntime(delay), nestable);

	return PostPendingTask(&pending_task);
}
```
首先task必须不为空，然后创建了一个PendingTask对象，这个PendingTask结构仅仅只是对task的一个封装, 之后就条用PostPendingTask发送这个任务

```c++
bool IncomingTaskQueue::PostPendingTask(PendingTask * pending_task) {
	bool accept_new_tasks;
	bool schedule_work = false;

	{
		std::lock_guard<std::mutex> lock(incoming_queue_lock_);
		accept_new_tasks = accept_new_tasks_;
		if (accept_new_tasks) {
			schedule_work =
				PostPendingTaskLockRequired(pending_task);
		}
	}
	//....
	// 唤醒message loop 并且给他派遣工作
	if (schedule_work) {
		// 锁住message loop, 防止message loop被释放.
		std::lock_guard<std::mutex> lock(message_loop_lock_);
		if (message_loop_)
			message_loop_->SchedueWork();
	}

	return true;
}
```
PostPendingTask就是主要的添加任务的函数，其中在像incoming_queue_添加任务时，将队列进行了加锁，这个步骤时必须的（不同的线程可以操作这个队列)， 在PostPendingTaskLockRequered函数中，就将task加入到了incoming_queue，任务加入队列之后，需要唤醒消息循环，告诉消息循环来了任务，你可以进行处理了.

```c++
bool IncomingTaskQueue::PostPendingTaskLockRequired(PendingTask * pending_task) {
	//...
	pending_task->sequence_num = next_sequence_num_++;

	bool was_empty = incoming_queue_.empty();
	incoming_queue_.push(std::move(*pending_task));

	//...
}
```
首先将任务的序列号自增，然后将pending_task加入到incmoing_queue,incoming队列的添加操作以及分析完


我们最后分析MessageLoop，先从一些MessageLoop依赖类出发，首先分析MessageLoop继承的两个类RunLoop::Deletgate, MessagePump::Deletgate

```c++
class BASE_EXPORT MessagePump {
 public:
	 class BASE_EXPORT Delegate {
	  public:
		  virtual ~Delegate() {}

		  virtual bool DoWork() = 0;

		  virtual bool DoDelayedWork(
			  std::chrono::milliseconds& next_delayed_work_time) = 0;

		  virtual bool DoIdleWork() = 0;
	 };

	 MessagePump();
	 virtual ~MessagePump();

	 virtual void Run(Delegate* delegate) = 0;

	 virtual void Quit() = 0;

	 virtual void ScheduleWork() = 0;

	 virtual void ScheduleDelayedWork(
		 const std::chrono::milliseconds& delayed_time_work) = 0;
};
```
从MessagePump的发现它只是一个抽象内，Deletgate是一个代理内，用做给外部继承，其中DoWork()处理普通工作，DoDelayerdWork中处理延迟任务,
DoIdleWork中处理闲置任务, MessageLoop继承了这个类，并且实现了这个三个方法, 至于MessagePump，之前MessagePumpDefault继承了它并且实现了对应的虚函数, 来看一下最主要的Run函数

```c++
void MessagePumpDefault::Run(Delegate* delegate) {
	for (;;) {
		bool did_work = delegate->DoWork();
		if (!keep_running_)
			break;

		did_work |= delegate->DoDelayedWork(delayed_work_time_);
		if (!keep_running_)
			break;

		if (did_work)
			// 作了延迟任务或者work.
			continue;

		// 没有做工作，就去做闲置的工作
		did_work = delegate->DoIdleWork();
		if (!keep_running_)
			break;

		// 做了闲置工作，contiune.
		if (did_work)
			continue;

		std::unique_lock<std::mutex> lock(mutex_);
		if (delayed_work_time_.count() == 0) {
			event_.wait(lock);
		}
		else {
			// 在这里原本是准备使用wait_until的，可是由于C++11的wait_until接受的参数
			// 是time_point, 而我们保存的delayed_work_time_确实std::chrno::duration
			// 类型，这两个类型之间的转换我弄了半天也没发现如何转换,所有改成了wait_for,
			// 改成wait_for的话就需要计算等待的时长，而不是运行的时间.
			auto now = std::chrono::system_clock::now();		
			auto wait_time = 
				delayed_work_time_ - std::chrono::duration_cast<
				std::chrono::milliseconds>(now.time_since_epoch());
			if (wait_time.count() <= 0)
				continue;
			//event_.wait_until(lock, now);
			event_.wait_for(lock, wait_time);
		}

	}
}
```
首先就是一个for循环，无限的处理任务，deletgate其实就是MessageLoop(因为MessageLoop继承了MessagePump::Deletgate), 所以这里的deletgate->Dowork()就是调用的MessgeLoop的Dowork(), 在for循环里面
首先会执行没有延迟的任务(DoWork), 接着又会处理延迟任务。只要这两个其中一个执行成功，代表队列中是有任务的，可以接着处理任务, 没有任务执行可能是延迟任务的延迟时间没有到，如果只剩下延迟任务，就等待延迟时间然后接着执行延迟任务.




