## Java Concurrency In Practice Notes

* [1.Introduction](#1)
* [2.对象的组合](#2)
* [3.中断](#3)
* [4.Executor](#4)
* [5.活跃性、性能与测试](#5)
* [6.性能与可伸缩性](#6)
* [7.显式锁](#7)
* [8.构建自定义的同步工具](#8)
* [9.原子变量与非阻塞同步机制](#9)
* [10.Java内存模型](#10)

<h2 id="1">1.Introduction</h2>
&emsp;&emsp; Java中主要同步机制是关键字synchromized，还包括volatile类型的变量，显式锁，原子变量。多个线程访问同一个可变的状态变量时，
没有使用合适的同步，那么程序会出错，三种方式可修复问题：1.不在线程之间共享状态变量。2.将状态变量修改为不可变的变量。3.在访问状态变量时使用同步。

<br>
&emsp;&emsp; 无状态的对象一定是线程安全的，例如：Servlet。实际中尽量使用线程安全对象来管理类的状态，如AtomicLong。
Java提供了一种内置的锁机制来支持原子性：同步代码块（Synchronized Block）。同步代码块分两部分：一个作为锁的对象的引用，一个作为由这个锁保护的代码块。
以关键字synchronized来修饰的方法是一种横跨整个方法体的同步代码块，其中该同步代码块的锁就是方法调用所在的对象，静态synchronized方法以Class对象作为锁。

    <p>
    synchronized (lock) {
        // ops
    }
    </p>

<br>
&emsp;&emsp; 每个Java对象都可以用做一个实现同步的锁，这些锁被称为内置锁(Intrinsic Lock)或监视锁(Monitor Lock)。
线程在进入同步代码块之前会自动获得锁，并且无论正常退出还是抛异常退出同步代码块时，都会自动释放锁。获得内置锁唯一的方式就是进入这个锁保护的同步代码块或方法。
Java的内置锁是互斥锁，意味着最多只有一个线程能持有这种锁。

<br>
&emsp;&emsp; 内置锁是可重入的，如果一个线程试图获得一个已经由它自己持有的锁，那么这个请求就会成功。获取锁的操作的粒度是线程而不是调用。
重入的一种实现方法是为每个锁关联一个获取计数值和一个所有者线程。当计数值为0时，这个锁就被认为是没有被任何线程持有，当线程请求一个没有被持有的锁时，
JVM记录下锁的持有者并将计数器置1，如果同一个线程再次获取这个锁，那么计数值递增，当线程退出同步代码块时，计数器会相应的递减，当计数器值为0时，
这个锁将被释放。

<br>
&emsp;&emsp; 对于可能被多个线程同时访问的可变状态变量，在访问它时都需要持有同一个锁，在这种情况下，我们称状态变量是由这个锁保护的。
对象的内置锁与其状态之间没有内在的关联，大多数类都将内置锁用做一种有效的加锁机制，但对象的域并不一定要通过内置锁保护，当获取与对象关联的锁时，
并不能阻止其他线程访问该对象，某个线程在获得对象的锁之后，只能阻止其他线程获得同一个锁。之所以每个对象都有一个内置锁，只是为了避免显式地创建锁对象。
每个共享可变的变量都应该只由一个锁来保护。

<br>
&emsp;&emsp; 一种常见的加锁约定是，将所有可变状态都封装在对象内部，通过对象内置锁对所有访问可变状态的代码路径进行同步，使得在该对象上不会发生并发访问。
例如：Vector和其他同步集合类。如果在添加新的方法或代码路径时忘了使用同步，那么这种加锁协议很容易就被破坏。

<br>
&emsp;&emsp; 对于每个包含多个变量的不变性条件，其中涉及的所有变量都需要由同一个锁来保护。当执行时间较长的计算或者可能无法快速完成的操作时，
一定不要持有锁。当线程在没有同步的情况下读取变量时，可能得到一个失效值，这个值是由之前某个线程设置的值，而不是随机值，这种安全性保证被称为最低安全性。
最低安全性使用于绝大多数变量，但存在一个例外，非volatile类型的64位数值变量(double 和 long)。Java内存模型要求，变量的读和写操作都必须是原子操作。
但对于非volatile类型的long和double，JVM允许将64位的读或写操作，分解为两个32位的操作。在多线程中使用共享且可变的long和double，是不安全的，
除非用volatile声明或用锁保护起来。

<br>
&emsp;&emsp; 访问某个共享且可变的变量时要求所有线程在同一个锁上同步，用来保证某个线程写入该变量的值对于其他线程来说是可见的。

<br>
&emsp;&emsp; 当变量声明为volatile时，编译器与运行时不会将该变量上的操作与其他内存操作重排序。volatile变量不会被缓存在寄存器或其他处理器不可见的地方，
因此在读取colatile变量时总会返回最新写入的值。volatile变量是一种比synchronize关键字更轻量级的同步机制。不建议过度依赖volatile变量提供可见性，
在代码中过度依赖volatile控制状态的可见性，通常比锁更脆弱，也更难理解。volatile正确的使用方式包括：确保它们自身状态的可见性，
确保它们所引用对象的状态的可见性，以及标识一些重要的程序生命周期时间的发生。

<br>
&emsp;&emsp; 当且仅当满足以下条件时，才使用colatile变量：对变量的写入操作不依赖变量的当前值，或者能确保只有单个线程更新变量的值。
该变量不会与其他状态变量一起纳入不变性条件中。在访问变量时不需要加锁。

<br>
&emsp;&emsp; 发布(Publish) 一个对象，是指对象能够在当前作用域之外的代码中使用。发布对象最简单的方法是将对象的引用保存到一个公有的静态变量中，
以便任何类和线程都能看见该对象。

<br>
&emsp;&emsp; ThreadLocal能使线程中的某个值与保存值的对象关联起来。ThreadLocal提供了get set方法，这些方法为每个使用该变量的线程都留有一份独立的副本。
因此get总是返回由当前执行线程在调用set时设置的最新值。ThreadLocal 对象通常用于防止对可变的单实例变量或全局变量进行共享。
ThreadLocal用来维持线程封闭性。ThreadLocal变量类似于全局变量，能降低代码的可重用性，并在类之间引入隐含的耦合性，因此使用时要格外小心。

<br>
&emsp;&emsp; 不可变对象一定是线程安全的。不可变性不等于将对象的所有域声明为final，即使对象中所有域都是final类型的，对象仍然是可变的，
因为在final域中可以保存对可变对象的引用。满足以下条件时，对象才是不可变的：对象创建后状态不能修改，对象所有的域都是final，对象是正确创建的，
在对象创建期间，this引用没有逸出。如果final类型的域指向的是可变对象，那么在访问这些域所指向的对象的状态时，仍然需要使用同步。

<br>
&emsp;&emsp; 要安全的发布一个正确构造的对象，可以通过以下方式：在静态初始化函数中初始化一个对象引用；将对象的引用保存到volatile类型的域或AtomicReferance对象中；
将对象的引用保存到某个正确构造对象的final类型域中；将对象的引用保存到一个由锁保护的域中。

<br>
&emsp;&emsp; 线程安全库中的容器类提供了以下安全发布保证：1.通过将一个键或值放入HashTable、SynchronizedMap、或ConcurrentMap中，
可以安全的将它发布给任何从这些容器中访问它的线程。2.通过将某个元素放入Vector、CopyOnWriteArrayList、CopyOnWriteArraySet、SynchronizedList
或SynchronizedSet中，可以将该元素安全的发布到任何从这些容器中访问该元素的线程。3.通过将某个元素放入BlockingQueue或ConcurrentLinkedQueue中，
可以将该元素安全的发布到任何从这些队列中访问元素的线程。

<br>
&emsp;&emsp; 类库中其他数据传递机制（Future 和 Exchanger）同样能实现安全发布。通常发布一个静态构造的对象，最简单安全的方式是使用静态的初始化器，
静态初始化器由JVM在类的初始化阶段执行，由于JVM内部存在同步机制，因此通过这种方式初始化的任何对象都能被安全的发布。

    <p>
    public static Holder holder = new Holder(123);
    </p>
    
<br>
&emsp;&emsp; 如果对象从技术上来看是可变的，但其状态在发布后不会再改变，那么把这种对象称为事实不可变对象。
对象的发布需求取决于它的可变性：1.不可变对象可以通过任意机制来发布。2.事实不可变对象必须通过安全方式发布。3.可变对象必须通过安全方式发布，
并且必须是线程安全的或者由某个锁保护起来。

<br>
&emsp;&emsp; 在并发程序中使用和共享对象时，可以采用一些策略：1.线程封闭，线程封闭的类只能由一个线程拥有，对象被封闭在该线程中，并且只能由这个线程修改。
2.只读共享，在没有额外同步的情况下，共享的只读对象可以由多个线程并发访问，但任何线程不能修改它，共享的只读对象包括不可变对象和事实不可变对象。
3.线程安全共享，线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公有接口来进行访问而不需要进一步的同步。
4.保护对象，被保护的对象只能通过持有特定的锁来访问，保护对象包括封装在其他线程安全对象中的对象，以及已发布的并且由某个特定的锁保护的对象。

<h2 id="2">2.对象的组合</h2>
&emsp;&emsp; 为了防止多个线程在并发访问同一个对象时产生的相互干扰，这些对象应该要么是线程安全的对象，要么是事实不可变的对象，或者由锁来保护的对象。
将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时，总能持有正确的值。线程封闭通过确保对象只能由单个线程访问来实现。

<br>
&emsp;&emsp; 一些基本的容器类并不是线程安全的，如ArryList和HashMap，但通过包装器工厂方法，使得这些非线程安全的类可以在多线程的情况下使用，
这些工厂方法通过装饰器模式，将容器封装在一个同步的包装器对象中，包装器能将接口中的每个方法都实现为同步方法，并将调用请求转发到底层的容器对象上。
只要包装器对象拥有对底层容器对象的唯一引用，那么就是线程安全的。如：Collections.synchronizedList。

<br>
&emsp;&emsp; 遵循Java监视器模式的对象，会把对象的所有可变状态都封装起来，并由对象自己的内置锁保护。

<br>
&emsp;&emsp; 闭锁是一种同步工具，可以延迟线程的进度直到其到达终止状态。CountDownLatch是一种灵活的闭锁实现。可以使一个或多个线程等待一组事件的发生。
FutureTask 也可以用做闭锁，表示一种抽象的可生成结果的计算。FutureTask表示的计算是通过Callable实现的，相当于一种可生成结果的Runnable，
并且可处于以下三种状态：1.等待运行。2.正在运行。3.运行完成。执行完成表示计算的所有可能结束方式，包括正常结束、由于取消而结束和由于异常而结束。
当FutureTask进入结束状态后，会永远停止在这个状态。

<br>

### Semaphore
&emsp;&emsp; 可以使用Semaphore将任何一种容器变成有界阻塞容器，信号量的计数值会初始化为容器容量的最大值。栅栏类似闭锁，能阻塞一组线程直到某个事件发生，
栅栏与闭锁的关键区别在于，所有线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待事件，栅栏用于等待其他线程。

<br>

### CyclicBarrier
&emsp;&emsp; CyclicBarrier可以使一定数量的参与方反复地在栅栏位置汇集。在并行迭代算法中非常有用，通常将一个问题拆分成一系列相互独立的自问题。
当线程到达栅栏位置时将调用await方法，阻塞直到所有线程都到达栅栏的位置，如果所有线程到达了栅栏位置，栅栏将打开，所有线程被释放，栅栏被重置以便下次使用。
如果await调用超时，或者await阻塞被中断，那么栅栏被认为是打破了，所有阻塞的await调用都将终止，并抛出BrokenBarrierException。

<br>
&emsp;&emsp; 另一种形式的栅栏是Exchanger，是一种两方栅栏。

<h2 id="3">3.中断</h2>
&emsp;&emsp; 不要把不可靠的的取消操作至于阻塞操作中。否则可能会由于阻塞导致无法中断。如下程序，如果队列中元素已经满了，
那么在put时会一直阻塞，这时候，我们会发现，调用cancel()函数，并不会起任何作用。这时候，我们需要使用线程中断机制。

    <p>
        public class PrimeGenerator extends Thread {
            private final BlockingQueue<BigInteger> queue;
            private volatile boolean cancelled = false;
            
            public PrimeGenerator(BlockingQueue<BigInteger> queue) {
                this.queue = queue;
            }
            
            public void run() {
                try {
                    BigInteger p = BigInteger.ONE;
                    while(!cancelled) {
                        queue.put(p = p.nextProbablePrime());
                    }
                } catch(InterruptedException consumed) {
                    // do nothing;
                }
            }
            
            public void cancel() {
                this.cancelled = true;
            }
        }
        
        void consumePrime() {
            BlockingQueue<BigInteger> queue = new LinkedBlockingDeque<>();
            PrimeGenerator generator = new PrimeGenerator(queue);
            generator.start();
            try {
                while(needConsumeFlag) {
                    consume(queue.take());
                }
            } finally {
                generator.cancel();
            }
        }
    </p>
    
<br>
&emsp;&emsp; 线程分为两种：普通线程和守护线程。JVM启动时创建的所有线程中，除了主线程，其他的都是守护线程，当创建一个新的线程时，
新线程将继承创建它的线程的守护状态。因此主线程创建的线程都是普通线程。普通线程与守护线程的区别在于：当线程退出时，JVM会检查其他正在运行的线程，
如果这些线程都是守护线程，那么JVM会正常退出操作，当JVM停止时，所有仍然存在的守护线程都会被抛弃，既不会执行finally代码块，也不会执行回卷栈，
而JVM只是直接退出。所以尽可能少的试用守护线程，很少有操作能在不进行清理的情况下被安全的抛弃，当守护线程中包含IO操作任务，那么将是一种危险的行为，
守护线程最好用于执行内部任务，例如周期性的从内存的缓存中移除逾期的数据。

<br>
&emsp;&emsp; 当不需要内存资源时，可以通过垃圾回收器来回收，对于其他的资源如文件句柄或者套接字句柄，不需要时必须显式的交还给操作系统。
为了实现这个功能，垃圾回收器对那些定义了finalize方法的对象会进行特殊处理，在回收器释放它们后，调用它们的finalize方法，从而保证持久的资源被释放。
在大多数情况下，通过使用finally代码块和close方法，能够比使用终结器更好的管理资源。唯一的例外在于，当需要管理对象，并且该对象持有的资源，
是通过本地方法获得的。尽量避免编写或使用包含终结器的类，除非是平台库中的类。

<h2 id="4">4.Executor</h2>
&emsp;&emsp; 在单线程的Executor中，如果一个任务将另一个任务提交到同一个Executor，并且等待这个被提交任务的结果，那么将会引发死锁。
在更大的线程池中，如果所有正在执行任务的线程都由于等待其他仍处于工作队列中的任务而阻塞，那么会发生同样的问题，这个现象称为线程饥饿死锁。

<br>
&emsp;&emsp; 对于计算密集型的任务，在拥有N个处理器的系统上，当线程池大小为N + 1 时，通常能实现最优的利用率。
对于包含I／O操作或者其他阻塞操作的任务，由于线程并不会一直执行，因此线程池的规模应该更大。要正确设置线程池的大小，必须估算出任务的等待

<br>
&emsp;&emsp; N<sub>cpu</sub> = number of cpus;  U = target cpu utilization, 0 <= U <= 1; W/C = ratio of wait time to compute time;
要使处理器达到期望的使用率，线程池最优大小等于 N<sub>threads</sub> = N<sub>cpu</sub> * U * (1 + W/C);
CPU 数量可以通过如下方式获取：Runtime.getRuntime().availableProcessors();

<br>
&emsp;&emsp; 可以使用SynchronousQueue来避免任务排队，SynchronousQueue是一种在线程之间进行移交的机制，要将一个元素放入Synchronous中，
必须有另一个线程正在等待接受这个元素，如果没有线程等待并且线程池的大小小于最小值，那么ThreadPoolExecutor将创建一个新的线程，
否则根据饱和策略，这个任务将被拒绝。只有当线程池是无界的或者可以拒绝任务时，SynchronousQueue才有实际价值。newCachedThreadPool使用了它。

<br>
&emsp;&emsp; LinkedBlockingQueue或ArrayBlockingQueue这种FIFO队列，任务执行顺序与到达顺序相同，如果想进一步控制任务执行顺序，
可以使用PriorityBlockingQueue，任务顺序是根据自然顺序或者Comparator来定义的。

<br>
&emsp;&emsp; 只有当任务相互独立时，为线程池或工作队列设置界限才是合理的，如果任务之间有依赖性，那么有界的线程池会导致线程饥饿死锁问题。
此时应该使用无界的线程池，例如newCachedThreadPool。

<br>
&emsp;&emsp; 当有界队列被填满后，饱和策略开始发挥作用，ThreadPoolExecutor饱和策略可以通过调用setRejectedExecutionHandler来修改。
JDK提供了几种RejectedExecutionHandler实现：AbortPolicy、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy。

<br>
&emsp;&emsp; 中止(Abort) 是默认的饱和策略，该策略将抛出未检查的RejectedExecutionException。调用者捕获这个异常。
当新提交的任务无法保存到队列中等待执行时，抛弃(Discard)策略会抛弃改任务，抛弃最旧的策略将会抛弃下一个即将被执行的任务，然后尝试重新提交新的任务。
如果是优先队列，那么抛弃最旧的策略将导致抛弃优先级最高的任务，因此最好不要将抛弃最旧的饱和策略和优先级队列一起使用。

<br>
&emsp;&emsp; 调用者运行(CallerRuns)策略实现了一种调节机制。既不抛弃任务，也不抛出异常。而是将任务回退到调用者，从而降低新任务的流量。
它不会在线程池中执行新提交的任务，而是在一个调用了execute的线程中执行该任务。

<br>
&emsp;&emsp; 当串行循环中的各个操作之间彼此独立，并且每个迭代操作执行的工作量比管理一个新任务时带来的开销更多，那么这个串行循环就适合并行化。

<h2 id="5">5.活跃性、性能与测试</h2>
&emsp;&emsp; 如果在A线程持有锁L并想获得锁M的同时，B线程持有锁M并想获得锁L，那么A、B两个线程将会永远等待下去，这就是最简单的死锁。

<br>
&emsp;&emsp; 如果两个线程试图以不同的顺序来获得相同的锁，那么可能发生锁顺序死锁，如果所有线程以固定的顺序获得锁，那么在程序中不会出现锁顺序死锁。
    
    <p>
       A -> 锁住left -> 尝试锁住right -> 永久等待
       B -------> 锁住right -> 尝试锁住left -> 永久等待
    </p>
   
<br>
&emsp;&emsp; 如果在持有锁时调用某个外部方法，那么将出现活跃性问题，在这个外部方法中可能会获取其他锁(这可能会产生死锁)，或者阻塞时间过长，
导致其他线程无法及时获得当前被持有的锁。

<br>
&emsp;&emsp; 如果在调用某个方法时不需要持有锁，那么这种调用被称为开放调用。在程序中应该尽量使用开放调用，与那些在持有锁时调用外部方法的程序相比，
更易于对依赖于开放调用的程序进行死锁分析。

<h2 id="6">6.性能与可伸缩性</h2>
&emsp;&emsp; 线程最主要的目的就是提高程序的运行性能，线程可以使程序充分发挥系统的可用处理能力，从而提高系统的资源利用率。
线程还可以使程序在运行现有程序的情况下立即开始处理新的任务，从而提高系统的响应性。使用多个线程会引入额外的性能开销：线程之间的协调(如加锁、
触发信号、内存同步等)，增加的上下文切换、线程的创建和销毁、线程的调度等。如果过度的使用线程，这些开销甚至会超过由于提高吞吐量、响应性或者计算能力
所带来的性能提升。

<br>
&emsp;&emsp; 要想通过并发获得更好的性能，需要做好两件事情：更有效的利用现有处理资源，以及在出现新的处理资源时使程序尽可能利用这些新资源。

<br>

### Amdahl定律
&emsp;&emsp; Amdahl定律：在增加计算资源的情况下，程序在理论上能够实现最高加速比，这个值取决于程序中可并行组件与串行组件所占的比重。
假设F是必须被串行执行的部分，那么根据Amdahl定律，在包含N个处理器的机器中，最高的加速比为：

    <p>
        Speedup <= 1 / (F + (1 - F) / N)
    </p>

<br>
&emsp;&emsp; 当N趋近于无穷大的时候，最大的加速比趋近于 1 / F 。如果在程序中有10%的计算需要串行执行，那么最高的加速比将接近10。
在拥有10个处理器的系统中，如果程序有10%部分需要串行执行，那么最高的加速比为5.3，在拥有100个处理器的系统中，加速比可以达到9.2，
即使拥有无限多的CPU，加速比也不可能为10。

<br>

### 上下文切换
&emsp;&emsp; 如果主线程是唯一的线程，那么基本不会被调度出去。如果可运行的线程数大于CPU的数量，那么操作系统将会将某个正在运行的线程调度出来，
从而使其他线程能够使用CPU。这将导致一次上下文切换。

<br>
&emsp;&emsp; 切换上下文需要一定的开销，而在线程调度过程中需要访问由操作系统和JVM共享的数据结构。应用程序、操作系统以及JVM都使用一组相同的CPU。
在JVM和操作系统的代码中消耗越多的CPU时钟周期，应用程序的可用CPU时钟周期就越少。上下文切换的开销并不只是包含JVM和操作系统的开销。
当一个新的线程被切换进来时，它所需要的数据可能不在当前处理器的本地缓存中，因此上下文切换将导致一些缓存缺失，因而线程在首次调度运行时会更加缓慢。
这就是为什么调度器会为每个可运行的线程分配一个最小执行时间，即使有其他的线程正在等待执行：它将上下文切换的开销分摊到更多不会中断的执行时间上，
从而提高整体的吞吐量。

<br>
&emsp;&emsp; 当线程由于等待某个发生竞争的锁而被阻塞时，JVM会将这个线程挂起，并允许它被交换出去。在程序中发生越多的阻塞，与CPU密集型的程序，
就会发生越多的上下文切换，从而增加调度开销，并因此而降低吞吐量。无阻塞算法有助于减小上下文切换。上下文切换的开销相当于5000-10000个时钟周期，
也就是几微秒。UNIX系统的vmstat命令和Windows系统的perfmon工具都能报告上下文切换次数以及在内核中执行时间所占比例等信息。
如果内核占用率较高（超过10%），那么通常表示调度活动发生的很频繁，这很可能是由I／O或竞争锁导致的阻塞引起的。

<br>

### 内存同步
&emsp;&emsp; 同步操作的性能开销包括多个方面。在synchronized和volatile提供的可见性保证中可能使用一些特殊指令，即内训栅栏。
内存栅栏可以刷新缓存使缓存无效，刷新硬件的写缓冲，以及停止执行管道。内存栅栏可能同样会对性能带来间接的影响。因为它们将抑制一些编译器优化操作。
在内存栅栏中，大多数操作是不能被重排序的。

<br>
&emsp;&emsp; synchronized机制针对无竞争的同步做了优化，volatile通常是非竞争的。一个快速通道（Fast-Path）的非竞争同步将消耗20-250个时钟周期。
虽然无竞争同步的开销不为0，但是对于程序整体的影响微乎其微。现代的JVM会通过优化来去掉一些不会发生竞争的锁，从而减少不必要的同步开销。
如果一个锁对象只能由当前线程访问，那么JVM就可以通过优化来去掉这个锁获取操作。

<br>
&emsp;&emsp; 一些JVM能通过逸出分析找到不会发布到堆的本地对象引用（这个引用是线程本地的）。getStoogeNames会将Vector上锁获取释放四次。
但是一个智能的运行时编译器会分析调用，从而使stooges及其内部状态不会逸出，因此可以去掉这4次对锁获取操作。

    <p>
        public String getStoogeNames() {
            List<String> stooges = new Vector<String>();
            stooges.add("Moe");
            stooges.add("Larry");
            stooges.add("Curly");
            return stooges.toString();
         }
    </p>

<br>
&emsp;&emsp; 某个线程的同步可能影响到其他线程的性能，同步会增加共享内存总线上的通信量，总线的带宽是有限的，并且所有的处理器将共享这条总线，
如果有多个线程竞争同步带宽，那么所有使用了同步的线程都会收到影响。

<br>
&emsp;&emsp; 非竞争的同步可以在JVM中进行处理，竞争的同步可能需要操作系统的介入，从而增加开销。当线程锁竞争失败时，进入阻塞状态，JVM在阻塞时，
可以采用自旋等待(循环尝试获取锁，直到成功)，或者通过操作系统刮起被阻塞的线程。如果时间短采用自旋等待方式，如果等待时间长，采用挂起方式。
大多数JVM采用线程挂起方式。

<br>
&emsp;&emsp; 在线程无法获取某个锁，或者由于某个条件等待或者在I／O操作上阻塞时，需要被挂起，在这个过程中将包含两次额外的上下文切换，
以及所有必要的操作系统操作和缓存操作：被阻塞的线程在其执行时间片还未用完之前被交换出去，随后要获取的锁或其他资源可用时，被切换回来。

<br>

### 减少锁的竞争
&emsp;&emsp; 串行操作会降低可伸缩性，上下文切换会降低性能。在锁上发生竞争将同时导致这两种问题，因此减少锁的竞争能够提升性能和可伸缩性。
在并发程序中对可伸缩性的最主要威胁就是独占方式的资源锁。

<br>
&emsp;&emsp; 两个因素将影响在锁上发生竞争的可能性：锁的请求频率，以及每次持有该锁的时间。如果在锁上的请求量很高，
那么需要获取该锁的线程将被阻塞并等待。尽量减少锁的持有时间。

<br>
&emsp;&emsp; 很多人用对象池技术来循环利用对象，如果线程从对象池中请求一个对象，那么需要通过一个某种同步来协调对象池数据结构的访问，
从而使某个线程被阻塞。如果线程因为锁竞争而被阻塞，那么这种阻塞开销将是内存分配操作开销的数百倍。因此对象池看起来是性能优化技术，
实际上可能导致伸缩性问题。通常对象分配操作的开销，比同步的开销更低。

<br>

### 减少上下文切换的开销
&emsp;&emsp; 当任务在运行和阻塞这两个状态之间切换时，就相当于一次上下文切换。Amdahl定律告诉我们，程序的可伸缩性取决于在所有代码中必须被串行执行的代码比例。
Java中串行操作的主要来源是独占方式的资源锁，因此可以通过以下方式提升可伸缩性：减少锁的持有时间，降低锁的粒度，以及采用非独占的锁或非阻塞锁来代替独占锁。

<br>
&emsp;&emsp; 包括：吞吐量：一组并发任务中已完成任务所占的比例。响应性(延迟)：请求从出发到完成之间的时间。可伸缩性：指在增加更多资源的情况下，吞吐量的提升情况。

<h2 id="7">7.显式锁</h2>
&emsp;&emsp; Lock提供了一组抽象的加锁操作，Lock提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作，所有加锁和解锁的方法都是显式的。
ReentrantLock实现了Lock接口，提供了与synchronized相同的互斥性和内存可见性，而且是一种可重入锁。

<br>
&emsp;&emsp; 使用内置锁很难实现带有时间限制的操作。在ReentrantLock上的构造函数中，提供了两种公平性选择，创建一个非公平的锁或者一个公平的锁。
公平锁将按照线程发出请求的顺序来获得锁，非公平锁上，允许插队，当一个线程请求非公平锁时，如果该锁状态变为可用，那么这个线程将跳过队列中所有等待线程获得这个锁。
在Semaphore中同样可以选择采用公平或非公平的获取顺序。大多数情况下，非公平锁的性能高于公平锁。

<br>
&emsp;&emsp; 非公平锁的性能高于公平锁的原因是：恢复一个被挂起的线程与线程真正运行之间存在严重的延迟，假设A持有锁，B等待，A释放锁的时候，
B被唤醒，此时C请求这个锁，C可能在B被完全唤醒前获得释放这个锁，那么B完全唤醒后会获得锁，此时提升了吞吐量。如果持有锁时间较长或者请求锁的平均时间较长，
应该使用公平锁，此时插队带来的吞吐量提升可能不会出现。

<br>

### synchronized与ReentrantLock的选择
&emsp;&emsp; ReentrantLock危险性比synchronized同步机制高，如果忘记在finally中调用unlock，会很危险。因此，仅当内置锁不能满足需求时，才考虑使用ReentrantLock。
当使用高级功能时，才应该使用ReentrantLock，包括：可定时的、可轮询的、可中断的锁获取操作，公平队列，以及非块结构的锁。否则优先使用synchronized。
因为synchronized是JVM的内置属性，能执行一些优化，例如对线程封闭的锁对象的锁消除优化，通过增加锁的粒度来消除内置锁的同步，
而如果通过基于类库的锁实现这些功能，可能性不大。

<br>

### 读写锁(ReentrantReadWriteLock)
&emsp;&emsp; 读写锁能提升性能，一个资源可以被多个读操作访问，或者被一个写操作访问，当两者不能同时进行。
ReentrantReadWriteLock在构造时可以选择是一个非公平的锁，还是一个公平的锁。在公平锁中，等待时间最长的线程优先获得锁。如果锁由读线程持有，
另一个线程请求写入锁，那么其他读线程不能获得读取锁，直到写线程使用完并释放了写入锁。在非公平的锁中，线程获得访问许可的顺序是不确定的。
写线程降级为读线程是可以的，读线程不能升级为写线程。

<br>
&emsp;&emsp; 写入锁只能有唯一的所有者，并且只能由获得该锁的线程来释放。读取锁更类似一个semaphore而不是锁，只维护活跃的读线程的数量，
不考虑它们的标识。读写锁允许多个读线程并发的访问被保护的对象，当访问以读取操作为主的数据结构时，能提高程序的可伸缩性。

<h2 id="8">8.构建自定义的同步工具</h2>
&emsp;&emsp; 如果执行方法失败，重试可以采用休眠一段时间再调用的方式，或者直接重新调用该方法，重新调用称为忙等待或自旋等待。如果自旋等待，
可能消耗大量的CPU时间，导致CPU时钟周期浪费，如果休眠可能导致低响应性。此时可以通过Thread.yield，让出一定的时间使另一个线程运行。

<br>
&emsp;&emsp; 通过轮询与休眠来实现阻塞操作的过程，需要付出大量的努力，如果存在某种挂起线程的方法，并且能确保当某个条件成真时线程立即醒来，
那么将简化实现操作，这就是队列实现的操作。

<br>

### 条件队列
&emsp;&emsp; 可以通过wait、notifyAll实现条件队列，与休眠相比，条件队列并没有改变语义，只是在多个方面进行了优化：CPU效率、上下文切换开销、响应性。

<br>
&emsp;&emsp; wait方法将释放锁，阻塞当前线程，等待直到超时，然后通过线程中断或者一个通知被唤醒，唤醒后，wait在返回前要重新获取锁。
当线程从wait中被唤醒时，在重新请求锁时不具有任何特殊的优先级，而要与其他尝试进入同步代码块的线程一起正常的在锁上进行竞争。

<br>

### 显式的Condition对象
&emsp;&emsp; Condition是一种广义的内置条件队列。一个Condition和一个Lock关联在一起，Condition在每个锁上可以存在多个等待，
等待条件可以是可中断的或不可中断的、基于时限的等待以及公平的或非公平的队列操作。对于每个锁可以有任意数量的Condition对象。在condition中，
应该使用的方法分别为：await、signal、signalAll。

<br>
&emsp;&emsp; 与内置锁和条件队列一样，使用显式的Lock和Condition时，必须满足锁、条件谓词和条件变量之间的三元关系。
在条件谓词中包含的变量必须由Lock保护，并且在检查条件谓词及调用await和signal时，必须持有Lock对象。在显式的Condition和内置条件队列之间进行选择，
与在ReentrantLock和synchronized之间选择是一样的，如果需要高级功能，如使用公平的队列操作或者在每个锁上对应多个等待线程集，
那么优先使用Condition而不是内置条件队列。

<br>

### Synchronizer剖析
&emsp;&emsp; AbstractQueuedSynchronizer是许多其他同步类的基类。AQS用于构建锁和同步器的框架。不仅ReentrantLock和Semaphore是基于AQS的，
CountDownLatch、ReentrantReadWriteLock、SynchronousQueue和FutureTask也是基于AQS构建的。

<br>

### AbstractQueuedSynchronizer
&emsp;&emsp; 根据同步器的不同，获取操作可以是独占操作(ReentrantLock)，也可以是非独占操作(Semaphore和CountDownLatch)。
首先同步器判断是否允许获得操作，如果允许则执行，否则获取操作将阻塞或失败。对于锁来说，如果没有被某个线程持有，将能被成功获取，
对于闭锁，如果处于结束状态，也能被成功获取。

<br>
&emsp;&emsp; 如果同步器支持独占的获取操作，那么需要实现一些保护方法，包括：tryAcquire、tryRelease和isHeldExclusively等，
对于支持共享获取的同步器，应该实现tryAcquireShared、tryReleaseShared等。AQS中acquire、acquireShared、release、releaseShared，
这些带有try前缀的版本用来判断操作是否能执行。可以使用getState、setState、compareAndSetState来检查更新状态。

<h2 id="9">9.原子变量与非阻塞同步机制</h2>
&emsp;&emsp; 与基于锁的同步相比，非阻塞算法更复杂，但可伸缩性和活跃性有巨大的优势。如果线程在休眠或自旋的同时持有一个锁，那么其他线程将无法执行下去。
非阻塞算法不会收到单个线程失败的影响。原子变量可以用作一种更好的volatile类型变量。

<br>

### volatile
&emsp;&emsp; volatile提供了可见性保证，但不能用于构建原子的复合操作，比如++i，实际包含三个独立的操作，因此不能使用volatile来实现。

<br>

### 非阻塞算法
&emsp;&emsp; 如果存在某种算法，一个线程的失败或挂起不会导致其他线程也失败或挂起，那么这种算法被称为非阻塞算法。如果在算法的每个步骤中，
都存在某个线程能够执行下去，那么这种算法也被称为无锁算法。如果在算法中仅将CAS用于协调线程之间的操作，并且能正确的实现，那么它既是一种无阻塞算法，
也是一种无锁算法。非阻塞算法的特性为：某项工作具有不确定性，必须重新执行。

<h2 id="10">10.Java内存模型</h2>
&emsp;&emsp; 在编译器中生成的指令顺序，可以与源码中的顺序不同，编译器会把变量保存在寄存器而不是内存中。处理器可以采用乱序或并行等方式来执行指令，
缓存可能会改变将变量提交到主内存的次序。保存在处理器本地缓存中的值，对于其他处理器不可见。java规范要求JVM在线程中维护一种类似串行的语义：
只要程序的最终结果与在严格串行环境中执行的结果相同，那么所有上述操作是允许的。

<br>

### 重排序
&emsp;&emsp; 在没有充分同步的情况下，如果调度器采用不恰当的方式来交替执行不同线程的操作，那么将导致不正确的结果。
将缓存刷新到主内存中也可能出现不同的时序。内存级的重排序会使程序行为变得不可预测。同步将限制编译器、运行时和硬件对内存操作重排序的方式，
从而在实施重排序时不会破坏JMM提供的可见性保证。

<br>
&emsp;&emsp; 如果要保证执行操作B的线程看到操作A的结果，那么A和B之间必须满足Happens-Before关系。如果两个操作之间缺乏Happens-Before，
那么JVM可以任意的重排序。