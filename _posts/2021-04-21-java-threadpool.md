---
layout: post
title:  "Java线程池原理解析"
date:   2021-04-21 20:20:00 +0700
categories: [java]
---

## 前言
线程池在我们日常开发过程中我们经常使用到它, 为了对它有一个比较全面的认识, 我在这篇文章里将会把有关线程池的核心原理讲清楚.

## 线程池核心参数

参数名称|说明
-|-
核心线程数|线程池内的线程数如果小于这个值,那么新的任务到来时会优先创建线程而不是复用已有线程  
最大线程数|当大于核心线程数小于最大线程数时, 新来的任务都会进入到**任务队列**中存放,如果任务队列满了,这时才会创建新的线程,核心线程和非
核心线程的数量加在一起不能超过最大线程数  
任务队列|缓存任务的地方, 使用阻塞队列的API, 不同的*核心线程数:最大线程数*的配比会使用不同的任务队列.例如`newFixedThreadPool`创建的线程池使用的是`LinkedBlockingQueue`,而`newCachedThreadPool`创建的线程池使用的是`SynchronousQueue`.
keep-alive timeout|对非核心线程来说, 如果其空闲达到该参数设置的值那么会对线程进行销毁. 一般来说核心线程不会受此影响, 但如果对线程池显示调用`allowCoreThreadTimeOut`方法.  
Reject Policy(拒绝策略)|在任务队列满了而且当前线程数也达到了最大限制时,调用`execute`方法提交的任务会被线程池拒绝,有几种拒绝策略,默认的是AbortPolicy,该策略会抛出异常,其他的几个内置策略后面详述.

我们创建线程池一般是通过`Executors`提供的工具方法创建的, 最终调用的是`ThreadPoolExecutor`类的构造方法.
其实更加推荐的方式是我们手动使用`ThreadPoolExecutor`的构造器来创建,因为这样我们可以更加精细的控制线程池的行为,  
下面我们来看看这个构造器的各个参数的意思:  
参数名称|类型|含义  
-|-|-
corePoolSize|int|核心线程数  
maximumPoolSize|int|最大线程数
keepAliveTime|long|非核心线程的最大空闲时间  
workQueue|BlockingQueue|任务队列  
threadFactory|ThreadFactory|线程工厂,在这里我们可以控制创建Thread实例的方式,可以给Thread设置`UncaughtExceptionHandler`,即自定义异常捕获处理器  
handler|RejectedExecutionHandler|定义当到达线程上限并且队列已满的行为


另外对于任务入队列的策略有以下几种模式：  

模式名称|使用的数据结构|特点|缺点  
-|-|-|-
直接交付|SynchronousQueue|直接交付的队列主要应用于没有线程数上限的线程池, SynchronousQueue的特点是容量为0,即无论何时任何线程向其中放入任务都会失败,根据我们前面讲述的策略,队列已满时就要创建线程,因而这种策略的特点是创建尽可能多的线程满足要求,降低请求的响应时间,但是需要注意这种策略可能会加剧上下文切换的损耗反而会使响应时间提升. |需要注意资源利用情况可能超限,而且需要注意上下文切换的损耗   
无界队列|LinkedBlockingQueue|比较典型的是通过Executors创建的固定线程数线程池和单线程线程池.这种策略的特点是无论任务有多少,都不可能创建超过核心线程数的线程,因为是无界队列所以可以存放”无限“(当然受到JVM的设置影响,并不是真正的无限)的任务,对这种策略来说最大线程数这个参数是没有意义的.|这种策略的意义在于我们可以限制线程池使用的资源,但是要注意可能会增加请求的响应时间造成请求的积压,还有可能造成队列积压过多造成OOM  
有界队列|ArrayBlockingQueue|这种策略是上述两种的折衷.一般来说线程池中线程数配置的越大队列的长度设置的可以越小,可以更早的触发非核心线程的创建,而对于小的线程数来说,设置队列的容量可以偏大一些防止过早的触发拒绝策略|无,根据自己设置调整可以自由的优化

有如下几种拒绝策略(达到最大线程数和队列塞满了RejectedExecutionHandler)
- AbortPolicy:默认,抛出RejectedExecutionException异常.
- CallerRunsPolicy:调用者运行
- DiscardOldestPolicy: 抛出最先进入队列的任务
- DiscardPolicy: 抛弃当前试图加入队列的任务


## ThreadPoolExecutor内部结构
ThreadPoolExecutor内部通过一个整型表达线程数量和当前线程池的状态.

![avatar](/static/img/JAVA线程池——ctl结构.png)

关于runstate含义的解释如下
名称|含义
-|-
RUNNING|接收新任务并且处理队列中的任务  
SHUTDOWN|不接受新task，但是处理队列中的任务  
STOP|不接受新task，不处理队列中的任务，将队列中的任务取出使队列为空, 中断正在进行的任务  
TIDYING|所有任务全部中止，workerCount是0,然后调用terminated方法  
TERMINATED|terminated方法完成  

如下是线程池的状态流转:
![avatar](/static/img/JAVA线程池——线程池状态流转.png)


Pool重要结构如下表所示:
名称|作用
-|-
BlockingQueue<Runnable> workQueue|任务队列  
ReentrantLock mainLock|在修改内部状态时会用到,比如workers和completedTaskCount(已经完成的任务数量),我们在接下来的源码分析中会看到  
HashSet<Worker> workers|Worker集合

Worker重要结构
Worker继承自AQS，state用于保存当前是否被占用执行
```java
/** 执行任务的线程 */
final Thread thread;
/** 不从任务队列中拿而是初始化的时候就设置的任务 */
Runnable firstTask;
```

## ThreadPoolExecutor实现解析
首先我们来看execute方法的实现:
```java
public void execute(Runnable command) {
	if (command == null)
		throw new NullPointerException();
	int c = ctl.get();
	if (workerCountOf(c) < corePoolSize) {// 1
		if (addWorker(command, true))
			return;
		c = ctl.get();
	}
	if (isRunning(c) && workQueue.offer(command)) { // 2
		int recheck = ctl.get();
		if (! isRunning(recheck) && remove(command))
			reject(command);
		else if (workerCountOf(recheck) == 0)
			addWorker(null, false);
	}
	else if (!addWorker(command, false))
		reject(command);
}
```
1. 判断当前线程是否小于核心线程数，如果小于，那么创建一个线程，将firstTask设置为command直接返回。
2. 到了这里说明当前线程数大于等于核心线程数，所以尝试将任务放到队列中，下面根据成功、失败分为两种情况讨论。
3-1. 如果成功放入，这时候要确认线程池状态是否还是RUNNING，如果不是，那么尝试把放入队列的命令移除，然后调用RejectedExecutionHandler
进行拒绝
3-2. 如果队列已满，那么这时候才会新增线程，和之前非核心线程的新增条件可以对上。

我们接下来看一下添加线程的代码，addWorker主要可分为两大部分，
第一部分属于检查线程池状态的步骤，第二部分主要属于创建Worker并且进行启动
addWorker分析：
```java
retry:
for (;;) {
	int c = ctl.get();
	int rs = runStateOf(c);

	// Check if queue empty only if necessary.
	if (rs >= SHUTDOWN &&
		! (rs == SHUTDOWN &&
		   firstTask == null &&
		   ! workQueue.isEmpty())) // 1
		return false;

	for (;;) {
		int wc = workerCountOf(c);
		if (wc >= CAPACITY ||
			wc >= (core ? corePoolSize : maximumPoolSize)) // 2
			return false;
		if (compareAndIncrementWorkerCount(c)) // 3
			break retry;
		c = ctl.get();  // Re-read ctl
		if (runStateOf(c) != rs)
			continue retry;
		// else CAS failed due to workerCount change; retry inner loop
	}
}
```
1. 当线程池状态为STOP、TIDYING、TERMINATE时，addWOrker不会增加线程，
当线程池状态为SHUTDOWN的同时队列还有任务，这时候会允许加线程，否则也不会加。
对于RUNNING状态，则允许加线程。
2. 检查是否超过线程数（根据核心线程和非核心线程区别）上限，如果超过直接返回。
3. 尝试CAS更新wokerCount，如果成功进入下一步否则检查线程池状态，如果发生改变回到1，如果没有改变回到2.

第二部分：
```java
boolean workerStarted = false;
boolean workerAdded = false;
Worker w = null;
try {
	w = new Worker(firstTask); // @1
	final Thread t = w.thread;
	if (t != null) {
		final ReentrantLock mainLock = this.mainLock;
		mainLock.lock();
		try {
			// Recheck while holding lock.
			// Back out on ThreadFactory failure or if
			// shut down before lock acquired.
			int rs = runStateOf(ctl.get());

			if (rs < SHUTDOWN ||
				(rs == SHUTDOWN && firstTask == null)) {
				if (t.isAlive()) // precheck that t is startable
					throw new IllegalThreadStateException();
				workers.add(w);
				int s = workers.size();
				if (s > largestPoolSize)
					largestPoolSize = s;
				workerAdded = true;
			}
		} finally {
			mainLock.unlock();
		}
		if (workerAdded) {
			t.start();
			workerStarted = true;
		}
	}
} finally {
	if (! workerStarted)
		addWorkerFailed(w); // 启动失败了怎么办，见下方解析
}
return workerStarted;
```
1.创建新的Worker
2.mainLock锁住，因为要修改workers和检查线程池状态
3.检查线程池状态，如果处于RUNNING或者处于SHUTDOWN状态但没有firstTask那么将新的woker加入到workers中。
4.启动线程


Worker的start方法如下：
```java
public void run() {
	runWorker(this);
}
```
实际上调用了runWorker方法，接下来我们来看runWorker方法：

runWorker解析
```java
final void runWorker(Worker w) {
	Thread wt = Thread.currentThread();
	Runnable task = w.firstTask;
	w.firstTask = null;
	w.unlock();// 这里调用解锁，实际上对应了初始化Worker时，给state设置的-1，在运行runworker之前不允许中断
	boolean completedAbruptly = true; // 当task内抛出异常时会被设置
	try {
		while (task != null || (task = getTask()) != null) { // 1
			w.lock();
			if ((runStateAtLeast(ctl.get(), STOP) ||
				 (Thread.interrupted() &&
				  runStateAtLeast(ctl.get(), STOP))) &&
				!wt.isInterrupted())
				wt.interrupt(); // 2
			try {
				beforeExecute(wt, task); // 钩子方法
				Throwable thrown = null;
				try {
					task.run(); // 3
				} catch (RuntimeException x) {
					thrown = x; throw x;
				} catch (Error x) {
					thrown = x; throw x;
				} catch (Throwable x) {
					thrown = x; throw new Error(x);
				} finally {
					afterExecute(task, thrown); // 钩子方法
				}
			} finally {
				task = null;
				w.completedTasks++;
				w.unlock();
			}
		}
		completedAbruptly = false; // 执行正常无异常
	} finally {
		processWorkerExit(w, completedAbruptly);
	}
}
```
1. 获取任务，具体的逻辑见下方，如果线程池状态正常的话，要么会阻塞在这里，要么有任务拿到任务
2. 如果worker被中断，比如shutdownnow造成的中断这里会打上中断标记
3. 执行任务，如果抛出异常的话这里会重新抛出，（但是submit方法提交的任务有所不同，详见下方）
4. 如果执行任务没有异常的话会再次循环取任务处理，否则由于抛出任异常，会进入processWorkerExit方法中，该方法会将当前线程
终止并且可能替换一个新的线程。

下面我们看processWorkerExit方法做了什么：
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
	if (completedAbruptly) 
		decrementWorkerCount();

	final ReentrantLock mainLock = this.mainLock;
	mainLock.lock();
	try {
		completedTaskCount += w.completedTasks; // 1
		workers.remove(w);
	} finally {
		mainLock.unlock();
	}

	tryTerminate(); // 2

	int c = ctl.get();
	if (runStateLessThan(c, STOP)) {
		if (!completedAbruptly) { // 3
			int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
			if (min == 0 && ! workQueue.isEmpty())
				min = 1;
			if (workerCountOf(c) >= min)
				return; // replacement not needed
		}
		addWorker(null, false);
	}
}
```
1. 将worker从wokerset中移除, 更新completedTasks, 由于这些操作不是原子的因此需要在加锁的情况下进行.
2. 因为我们这里减少了worker, 所以要检查一下当前的状态是否可以TERMINATED(队列、线程为空而且处于SHUTDWON或STOP状态)
3. 如果线程不是正常退出的, 那么这里要再替换一个新的线程. 同时这里有对allowCoreThreadTimeOut的处理,因为不涉及主流程就不展开说了.

我们必须知道只有当队列和workerset同时为空时才可以进入TERMINATE状态
下面我们看看tryTerminate方法的实现:
```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) || // 限制当前状态为STOP或者SHUTDOWN
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())) // 1
            return;
        if (workerCountOf(c) != 0) { // 2
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { // 3
                try {
                    terminated(); // 4
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0)); // 5
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```
1. 首先这里对线程池的状态做了一些校验, 具体如下:当前状态必须是STOP或者SHUTDOWN而且队列必须为空
2. 上述条件满足但是仍然有存活worker(可能有正在执行的任务), 这时候对其中一个线程进行中断.
3. mainLock加锁,因为接下来的要对内部状态做修改而且不是原子的,接下来修改了ctl,将状态修改为TIDYING,修改完成之后调用terminated钩子方法.
4. 调用完成之后就进入了TERMINATED状态,最终唤醒所有调用awaitTeminate方法的线程.

getTask方法解析
```java
private Runnable getTask() {
	boolean timedOut = false; // Did the last poll() time out?

	for (;;) {
		int c = ctl.get();
		int rs = runStateOf(c);

		// Check if queue empty only if necessary.
		if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) { // 1
			decrementWorkerCount();
			return null;
		}

		int wc = workerCountOf(c);

		// Are workers subject to culling?
		boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; // 2

		if ((wc > maximumPoolSize || (timed && timedOut)) // 3
			&& (wc > 1 || workQueue.isEmpty())) {
			if (compareAndDecrementWorkerCount(c))
				return null;
			continue;
		}

		try {
			Runnable r = timed ?
				workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
				workQueue.take(); // 4
			if (r != null)
				return r;
			timedOut = true;
		} catch (InterruptedException retry) {
			timedOut = false;
		}
	}
}
```
1. 首先校验当前线程池状态,必须是RUNNING或者SHUTDOWN并且队列不为空,否则返回null,返回null的话在runWorker中线程就会退出循环结束流程.
2. timed表示是否要判断超时,如果allowCoreThreadTimeOut为true的话表示核心线程也要判断超时.timeout在下面获取任务的地方会设置为true.
3. 这里是判断超时的地方,可以看到如果超时同样返回null.
4. 这里是最终从阻塞队列中拿任务的地方,如果是有时限的,那么如果在一定时间内没拿到会设置timedOut为true,在下一次循环的时候会在步骤3直接退出.



