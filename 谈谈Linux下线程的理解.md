最近深入研究了下Linux线程的问题，发现自己之前一直有些许误解，特记之……

关于Linux下的线程，各种介绍Linux的书籍都没有深入去解释的，或许真的如书上所述，Linux本质上不存在线程的概念！在某种程度上的确是这样，但是难道LInux就只有一种进程的东西么？？答案肯定是否定的！下面咱们慢慢分析

说起Linux下的线程，的确不如windows下来的直接，windwos中进程和线程有着明确的区分，各自有自己代表的数据结构、操作API，进程享有资源和线程参加调度。一切都是那么自然。LInux下进程和线程的区分就没有那么明显了。从最初的用户级线程模型，到后来内核级线程模型，Linux 对线程的支持虽然不如windows那么明显，但是却一直在努力。最初的用户级线程这里就不在多说，那么时代内核还没有加入对线程的支持，一切都是用户程序在YY，算不上是内核支持的线程。Linux支持线程恐怕要从LInuxThreads线程库被引入开始吧，后来基于LinuxThreadS线程库的各种缺点，NPTL线程库被引入进来。那么基于此，本篇文章介绍三个部分：内核级线程模型、LInuxThreas线程库、NPTL线程库。

### 1、内核级线程模型

关于线程模型，主要分为三种。用户级线程（1：N），内核级线程（1:1），还有另外一种混合线程（M :N）.因为种种原因，用户级线程和混合线程都没有进入内核，而正宗内核线程（1:1）确立了自己的正式地位。内核级线程，顾名思义是需要内核支持的，在早期用户线程模型，内核不晓得有线程的存在，对于内核来说，自己按照进程进行资源分配和调度。而用户程序在进程的基础上，实现了多线程。这样别的不说，线程的时间由用户程序分配，一方面更加灵活，而另一方面却大大增加了用户程序的复杂度。且这种情况下，总体时间由内核分配给进程，用户线程在平分进程得到的时间，宏观上讲，不同进程之间的线程没有公平可言。再者，面对多处理器，多线程更蹩脚了，进程A在处理器1上运行，那么进程A的所有线程都必须在处理器1上运行，试问这样的多处理器能发挥多大功效？而内核级线程就不同了，内核级线程是内核明确支持的，在内核中有其对应的数据结构，调度这点就不用用户操心了，由内核管理。既然内核明确支持多线程，线程调度是可以实现公平的，面对多处理器，内核线程更具备用户级线程所不具备的优势。那么既然线程思想已经引入内核，为何还说Linux中线程思想不明确呢？主要是在内核中，线程对应的结构和进程一样，都是对应task_struct结构，且进程创建和线程创建在底层的函数实现都是调用同一个AP Ido_fork,区别就在于参数不同，线程创建不会分配资源，会共享某个进程的资源。而实际上内核调度就是基于task_struct 结构的，这种情况下，你说调度到底是基于进程还是基于线程的？？

### 2、LinuxThread线程库

 LinuxThreas最先把多线程的概念引入Linux，但是其不适用于POSIX线程标准。但是Linux最初恐怕也没有思考那么多，也许LinuxThreads发起者们仅仅看到了多线程相对于多进程的巨大好处：1、同一进程内线程的切换比进程切换要快得多，因此多线程的程序比多进程的程序运行要快。2、多线程可以让切换见的间隙更小，从而实现更加真实的伪并行或者并行。基于上述目的，LinuxThreas便营运而生了。

LInuxthreads有以下特点：

1、管理线程的问题。每个进程拥有一个管理线程，进程内部线程的创建、销毁、同步都经过管理线程处理。通过管理线程，实现了下面对于多线程的要求：

* 系统必须有能力杀死整个进程。
* 线程栈的回收必须在对应线程结束后执行，所以线程不能自己处理。
* 将要终止的线程必须能够被等待，从而不至于成为僵尸。
* 线程数据区的回收需要对所有线程迭代执行，这个任务要管理线程完成。
* 如果主线程需要调用pthread_exit(),而进程并没有终止，主线程就睡眠，直到其他线程终止后，由管理线程唤醒主线程。
2、为了维护线程数据和内存，LinuxThreads使用进程地址空间的高位部分，即低于进程栈空间的部分。

3、基于信号机制实现线程同步。

4、LinuxThreads把每个线程实现为一个进程，拥有一个唯一的进程ID。

5、当进程收到终止信号，由管理线程负责用此信号杀死其他线程。

6、如果某个异步信号被发送，管理线程把信号传递给某个线程，如果该线程当前阻塞了该信号，则信号就是pending状态。

7、内核调度器实现对线程的调度。

 基于上述特点，LinuxThreads的缺点就很明显了：

1、其使用一个管理线程创建和协调进程中线程，这就增加了创建/销毁线程的开销。

2、既然其核心思想依赖于管理线程，那么必然要进行许多围绕管理线程的上下文切换，从而影响扩展性和性能。

3、由于管理线程只有一个，只能运行在一个CPU，在多处理情况下，同步问题是一个大问题。

4、使用信号实现同步，基于信号性能问题，会增加操作响应延迟，由于不能专门对进程发送信号，也不遵循POSIX标准。

5、信号的处理时基于线程而不是基于进程，且信号的传递是串行化的，意味着如果某个线程阻塞了信号，信号不会被传递给其他线程直到该线程解除阻塞。

6、基于线程本质上是一个进程，那么那么同一进程下的线程user和groupID信息或许会不一致。对于setuid和setgid来讲会成为问题。

7、如果进程中某个线程发生crash，那么dump文件仅仅针对该线程而不是针对其所属进程（后期的LinuxThreads版本解决了这个问题）

8、由于每个线程也是一个进程，/proc文件系统会变得异常拥挤。且对于应用的线程数量也会受到限制。

9、线程数据的访问依赖于栈地址，不能直接获取，因此访问线程数据的速度较慢，且用户不能随便指定栈的大小，以防发生冲突。

 

### 3、NPTL线程库

NPTL或者成为Native POSIX Thread Library，是Linux下线程库的全新实现，克服了LinuxThreads的缺点，且遵循POSIX标准。在可扩展性和性能方面，NPTL有了质的提升，不过NPTL也是基于1:1的内核线程模型实现的。

NPTL的设计目标如下：

1、应该遵守POSIX标准

2、在多处理器系统上应该工作良好。

3、NPTL线程库应该和LInuxThreads二进制兼容。

4、NPTL应该充分利用NUMA的支持。

基于上述目标，NPTL的实现有如下特点：

 NPTL相对于LinuxThreads的提升，最最主要的两个方面是1、取消了管理线程 2、增加了futex实现同步

取消管理线程的好处：

1）不使用管理线程，线程的创建/终止，发送信号操作均有内核负责，另外内核还负责回收线程的栈空间，在线程终止的时候，会让父线程等待哦其所有孩子终止。

2）因为不适用管理线程，管理线程就不成为瓶颈，创建和终止效率更高，且减少了很多线程切换。

3）不使用管理线程，就不需要管理线程作为中介传递信号，在多处理器上可以更加方便的调度、通信。

futex带来的好处：

futex成为快速互斥，用于线程之间的同步。LInuxThreads模型使用信号机制，该机制下势必涉及线程的睡眠和唤醒，而睡眠和唤醒操作都是在内核完成的。futex实现了在用户空间操作资源，有内核实现仲裁。可以显著提高同步效率。

除此之外，信号处理可以基于进程，比如发送终止信号，可以直接针对进程发送，这样进程中的所有线程均收到信号。而对于信号的传递，进程可用线程可实现接收，而之前如果线程阻塞了，那么信号就pending了。

/proc文件系统只列举初始线程的信息。

资源使用报告基于进程而不是某个线程。

NPTL和LinuxThreads保持二进制兼容。

### 补充：

最后关于多线程在做下补充，现代不管是操作系统还是应用程序都是多线程的，许多人也都知道多线程可以让程序运行更快、更流畅，但是为何多线程会对程序有这么大的影响，许多人可能就不太明白，我前些时间在知乎上看到一个讨论多线程效率的问题，大牛们纷纷发表自己的意见，那么这里我就借花献佛，在这里对于多线程做下介绍，当然这里仅仅涉及当前应用的线程模型-----内核线程模型。

1、多线程为何能提高程序运行效率

关于这个问题，大多数人应该都明白，线程把一个进程划分成一个个相对独立的代码段，且线程取代进程参与调度，即CPU按照线程来分配时间。从时间上来讲，之前进程参与调度的时代，如果进程等待某个资源被阻塞，那么整个进程就都阻塞了，必须切换其他进程运行，而进程之间的切换开销较大。多线程情况下，如果一个线程阻塞，在没有同步关系的情况下完全可以调度其他的线程运行，粒度更小，相当于间隙更小，这样程序运行势必更快。但是这点并不是多线程高效的主要原因，其主要原因恐怕还要归结于多处理器带来的巨大优势。现代计算机的CPU数目与日俱增，如何充分利用多处理器的优势，让处理器尽可能的忙起来呢？做个比喻，一个固定大小的容器，如果放置大小为A的不规则石头，可放置10块，如果把石头敲碎，就可以放20块，因为在大块石头下，有很多间隙，好比内存中的碎片，没办法利用，而缩小粒度的话，就可以填充的比较实。这里某种程度上多线程也是这个道理。且多处理下，线程的调度更加灵活，一个进程的不同线程可能同时在不同的处理器上运行，这样就实现了真并行，程序运行自然更快。

2、单处理器下的多线程有意义么？

这就是知乎上这个问题的本质？对于这个问题，还真需要辩证的分析。先说没有意义，在单处理器下，任意时刻只能由一个线程运行，此时即使是拿线程调度，一个进程的所有时间加起来最多也就等于进程调度方式下进程获得的时间，且还不算线程调度下的切换时间。从这一个角度，多线程似乎没有意义。

但是为何说起有意义呢？主要是因为理想和现实的差距。事实上，不管是线程还是进程，任意时刻绝大部分都是睡眠的，即现代计算机大多数是IO密集型，涉及IO必然要等待，进程调度下，进程一旦等待，必然要切换另一个进程执行，而在线程调度下，一个线程等待，可以调度另一个线程运行，，从这点看，多线程即使是在单处理器仍然是有意义的！当然在CPU密集型的系统就令说了！
