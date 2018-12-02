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
由上面的代码你明显可以看到Thread继承了PlatformThread::Delegate, 呆会我就会带大家去看看这个，并且在Thread类里面我们也明显的看到了MessageLoop这个消息循环，在之前我们说够每一个Chromium线程都会包含一个MessageLoop，现在看来是对的，但是现在我并不讨论这个消息循环，而是来看Start()函数，这个函数明显就是Chromium中每个线程的启动函数,现在看看这个函数里面做了什么
```c++
bool Thread::Start() {
	Options options;

	return StartWithOptions(options);
}
```
我们发现这个Start函数啥都没干，只是定义了一个Options变量，然后调用了StartWithOptions,那么疑问又来了,Options是干嘛的,StartWithOptions是干嘛的，不要急我们来一个一个分析,首先来看看Option的定义

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