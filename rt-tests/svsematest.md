

# 概述

启动两个线程，或fork两个进程，测量信号量SYSV的时延。

本程序的代码量为619行(不包含空行)。

# 使用

## 编译

当rt-tests测试套件安装完成后，本程序就被安装在/usr/local/bin目录下。

```c++
# 编译安装rt-tests测试套件
sudo apt-get install build-essential libnuma-dev    # 安装编译环境和必需的库
git clone git://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git
cd rt-tests
git checkout stable/v1.0    # master分支不是稳定版，所以要切换到stable分支
make all
make install
```
## 参数

```c++
-a [NUM] --affinity 在处理器#N上运行线程#N，如果可能的话
                           用NUM将所有线程钉在处理器NUM上
-b USEC --breaktrace=USEC 当延迟大于USEC时，发送中断跟踪命令
-d DIST --distance=DIST 线程间隔的距离，单位是us 默认=500
-f --fork 叉取新进程而不是创建线程
-i INTV --interval=INTV 线程的基本间隔，单位是us 默认=1000
-l LOOPS --loops=LOOPS 循环数：默认=0(无尽)
-p PRIO --prio=PRIO优先级
-S -smp SMP测试：选项-a -t和相同的优先级
                           的所有线程
-t --线程 每个可用的处理器有一个线程
-t [NUM] --threads=NUM 线程数。
                           没有NUM，线程数=max_cpus
                           没有-t默认=1
```
## 例子

```bash
svsematest -a -t -p99 -i100 -d25 -l1000000
# -a 指定了亲和性
# -t 让每个处理器有一个线程
# -p 指定最高优先级为99
# -i 指定线程间基本间隔为100us
# -d 指定线程间间隔为25us
# -l 指定循环1,000,000次
```
## 数据含义

```bash
#0: ID13110, P99, CPU0, I100; #1: ID13111, P99, CPU0, Cycles 1000000
#2: ID13112, P98, CPU1, I125; #3: ID13113, P98, CPU1, Cycles 813573
#4: ID13114, P97, CPU2, I150; #5: ID13115, P97, CPU2, Cycles 667285
#6: ID13116, P96, CPU3, I175; #7: ID13117, P96, CPU3, Cycles 591403
#1 -> #0, Min    1, Cur    2, Avg    2, Max   12
#3 -> #2, Min    1, Cur    3, Avg    2, Max   12
#5 -> #4, Min    1, Cur    3, Avg    3, Max   12
#7 -> #6, Min    1, Cur    2, Avg    3, Max   11
```
* Min - 最小时延
* Cur - 最后获取到的时延
* Avg - 平均时延
* Max - 最大时延
# 实现方法

## 定义

```c++
/* 核心数据结构的解释 */
struct params {        // 线程或进程的状态
    int num;
    int num_threads;    // 线程数
    int cpu;        // 亲和性对应的CPU核
    int priority;    // 优先级(注：除非SMP测试，否则不同实体优先级是递减的)
    int affinity;
    int semid;        // 信号量集合的描述符(每个实体2个信号量，值各为1；receiver共用sender的semid)
    int sender;        // 0, receiver; 1, sender
    int samples;    // 实际循环的次数(receiver上不加，而是加到sender上)
    int max_cycles;    // 要求的循环次数
    int tracelimit;    // 中断追踪条件
    int tid;        // 线程id(内核视角，通过gettid()获得)
    pid_t pid;        // 如为子进程，则记录pid
    int shutdown;    // 实体应该退出的标志(samples >= max_cycles 或 maxdiff > tracelimit)
    int stopped;    // 实体已经退出的标志
    struct timespec delay;    // 和interval、distance相关，是递增的
    unsigned int mindiff, maxdiff;
        // 分别是sender->diff.tv_usec的最小者和最大者
    double sumdiff;    // sender->diff.tv_usec的累加
    struct timeval unblocked, received, diff;
        // sender->received - receiver->unblocked = sender->diff
    pthread_t threadid;    // 如为线程，则记录线程号(POSIX视角)
    struct params *neighbor;    // 一对儿实体中的另一方
    char error[MAX_PATH * 2];    // 记录错误信息
};

/* 与进程相关的全局变量 */
// mustfork - 如在父进程此变量将置1。它的意思是需要创建子进程。因为用户输入的是不带参数的-f选项。
// wasforked - 如在子进程此变量将置1。它的意思是当前进程是子进程。因为主进程在for循环里fork出所有的子进程后，子进程执行execve()使用了带参数的-f选项，其参数形如r1, s1, r2, s2这样，s代表发送者，r代表接收者，r1和s1是一对使用信号量通信的进程。
// wasforked_sender - 1当前子进程是发送者子进程，0当前子进程是接收者子进程。
// wasforked_threadno - 子进程的编号，具有相同编号的一对子进程相互通信。
```
## glibc函数

```c++
/* 功能：获取System V信号量集合的描述符。semop()和semctl()会使用它。
 * key：返回的信号量集合描述符是与本参数相关联的，一个key只对应一个固定的描述符。
 *        IPC_PRIVATE：私有的key。
 * nsems：信号量集合里的元素数。
 * semflg：
 *        IPC_CREATE：如果key不存在则创建之。(01000)
 *        IPC_EXCL：如果key存在则报错。(02000)
 *        IPC_NOWAIT：Return error on wait.(04000)
 * 返回值：成功则返回信号量集合的描述符(非负整数)；失败则返回-1。
 */
int semget(key_t key, int nsems, int semflg);

/* 功能：System V信号量集合的控制操作。
 * semid：要控制的信号量所在的信号量集合。
 * semnum：要控制的信号量在信号量集合中的序号。(从0开始编号。)
 * cmd：对信号量的控制操作。
 *        IPC_RMID - 0, 删除标识符
 *        IPC_SET - 1, 设置ipc_perm选项
 *        IPC_STAT - 2, 获取ipc_perm选项
 *        IPC_INFO - 3, See ipcs
 *         GETPID - 11, get sempid
 *         GETVAL - 12, get semval
 *         GETALL - 13, get all semval's
 *         GETNCNT - 14, get semncnt
 *         GETZCNT - 15, get semzcnt
 *         SETVAL - 16, set semval
 *         SETALL - 17, set all semval's
 * 第四个参数的类型为union semun，此参数是否存在取决于第三个参数cmd。
 */
int semctl(int semid, int semnum, int cmd, ...);

/* 功能：System V信号量操作。即P/V操作。
 * semid：信号量集合的描述符。执行semctl()函数可得。
 * sops：是一个数组指针，这个数组的元素是结构体sembuf。
 * nsops：数组spos里的元素要操作的元素的数量。
 */
int semop(int semid, struct sembuf *sops, size_t nsops);
```
# 代码解析

## 执行过程解析

1. 使用getenv()获取一个文件的名称myfile，用来在后面生成System V IPC关键字。
2. 使用process_options()处理参数。
    1. 如使用了-a选项，则修改全局变量affinity和setaffinity的值。setaffinity有三个值，AFFINITY_UNSPECIFIED表示没有使用-a选项，AFFINITY_SPECIFIED表示使用了-a选项且带参数，AFFINITY_USEALL表示使用了-a选项但没带参数(此时应用程序会指定一个核)。
    2. 如使用了-b选项，则先记录下用户输入的值，并在本函数后面的部分修改全局变量tracelimit，以设定中断跟踪的条件。
    3. 如使用了-d选项，则用全局变量distance记录用户输入的值。它表示线程之间的间隔，这个值最终是加到全局变量interval上的。
    4. 如使用了-f选项，则表示要创建进程。如指定了选项的参数(其形式为s<num>或r<num>, s代表sender，r代表receiver，<num>代表进程数)，则修改全局变量wasforked、wasforked_sender、wasforked_threadno；如没有指定选项的参数，则修改全局变量mustfork。
    5. 如使用了-i选项，则用全局变量interval记录用户输入。表示线程之间的基本间隔。
    6. 如使用了-l选项，则用全局变量max_cycles记录用户输入。表示循环次数。
    7. 如使用了-p选项，则用全局变量priority记录用户输入的优先级。
    8. 如使用了-S选项，则表示这是一个SMP测试。此时会设置全局变量smp、num_threads、setaffinity。
    9. 如果使用了-t选项，则会设置线程数。如用户指定了选项的参数，则使用num_threads记录用户的参数；如用户没有指定选项的参数，则线程数等同于处理器数量。
    10. 如果使用了?选项，则给error=1。这会使本函数后面的部分执行display_help()函数。
    11. 如果还没有创建进程(!wasforked)，则会检查亲和性、线程数、优先级的设置，给全局变量sameprio赋值(sameprio的意思是全部的线程都有同样的优先级)，并给tracelimit全局变量赋值。
    12. 如设置了error，则执行display_help()函数打印帮助信息并退出。
3. 使用check_privs()检查当前线程的调度策略是否是实时调度策略(SCHED_FIFO或SCHED_RR)，或者能改成SCHED_FIFO。当前线程要求是实时调度的，这样才能实时监控其它的线程或进程，若不能则直接退出程序。
4. 执行mlockall()锁定当前和以后的内存，防止被换页换出。
5. 如果定义了mustfork，则为fork模式创建共享内存。
    1. 计算共享内存totalsize的大小。
    2. 使用shm_open()创建共享内存对象。
    3. 使用ftruncate()把共享内存对象截断到所需的大小。
    4. 使用mmap()建立到共享内存对象的内存映射。
    5. 为发送者和接收者建立到共享内存的映射关系。
6. 如果没有定义mustfork，但定义了wasforked，则表示已经fork出来了进程。
    1. 使用shm_open()打开共享内存对象"/sigwaittest"。
    2. 使用fstat()获取共享内存对象的状态，进而得到共享内存的大小totalsize。
    3. 使用mmap()建立到共享内存对象的映射。
    4. 建立发送者和接收者到共享内存的映射关系，要求totalsize等于每个进程的结构体所需的空间大小。
    5. 如设置了wasforked_sender，则表示创建的是发送者进程，否则创建的是接收者进程，它们都会调用semathread()。
    6. 使用munmap()解除映射，执行return返回。
7. 执行signal()函数，捕获到SIGINT或SIGTERM后，则使用sighand()将全局变量mustshutdown置1.
8. 使用pthread_sigmask()设置线程信号屏蔽字。
9. 如果即没设置mustfork，也没设置wasforked，则表示要创建的是线程，则调用calloc()为发送者和接收者的相关数据`struct params`创建内存空间。
10. 将launchdelay和maindelay分别设置为10ms和50ms。
11. 使用for循环，启动所有的接收者线程和发送者线程：
    1. 执行ftok()利用myfile和当前线程在for循环里的编号i得到System V IPC的key。
    2. 执行semget()获取对应接收者线程的System V信号量集合描述符。
    3. 执行semctl()将对应接收者线程的信号集合里第１个信号量的semval设置为1。
    4. 执行semctl()将对应接收者线程的信号集合里第２个信号量的semval设置为1。
    5. 执行semop()，对信号量集合里的0号信号量执行-1操作。
    6. 执行semop()，对信号量集合里的1号信号量执行-1操作。
    7. 设置对应接收者线程的亲和性。如用户没有使用-a或-smp选项，则置-1表示没有亲和性。
    8. 设置对应接收者线程的优先级和中断追踪的条件。如不是进行SMP测试，则全局变量priority--，目的是让下个接收者线程的优先级降低。
    9. 使用全局变量interval设置对应接收者线程的延迟。全局变量interval在每个循环里都会增加distance，使得所有接收者线程的延迟是递增的关系。
    10. 使用全局变量max_cycles设置接收者线程的循环次数。
    11. 设置接收者的neighbor为对应的sender。
    12. 如定义了mustfork，则表示要创建进程，则设置接收者的pid和线程数，通过execvp()再次执行本程序。
    13. 如没有定义mustfork，则表示要创建线程，则通过pthread_create()创建线程，线程的代码是senathread，线程的参数是receiver[i]。
    14. 睡眠launchdelay的时间，等待接收者线程启动完成。
    15. 把接收者的数据结构通过memcpy()复制给发送者，即它们的大部分参数都是相等的。
    16. 设置发送者的neighbor为对应的receiver。
    17. 如定义了mustfork，则表示要创建进程，则设置接收者的pid和线程数，通过execvp()再次执行本程序。
    18. 没有定义mustfork，则表示要创建线程，则通过pthread_create()创建线程，线程的代码是senathread，线程的参数是sender[i]。
12. 如没有定义mustshutdown，则不停while循环等待线程结束以打印统计信息：
    1. 遍历所有线程的数据结构，如shutdown置位则给mustshutdown置位。
    2. 如果第一个接收者的samples大于oldsamples或mustshutdown置位，则
        * 在一个for循环里打印发送者和接收者的相关信息(id，优先级，亲和性，Cycles)。
        * 在一个for循环里打印接收者获得的统计信息(Min, Cur, Avg, Max)。
        * printed置1，表示已打印
    3. 如果走的else分支，则表示没有打印输出信息，则printed置0。
    4. 使用pthread_sigmask()将信号屏蔽字设置为SIGTERM和SIGINT。
    5. 睡眠maindelay的时间。
    6. 使用pthread_sigmask()将信号屏蔽字置空。
    7. 如果printed置位而mustshutdown没有置位，则表明线程记录到了错误，则打印统计信息。
13. 在for循环里将所有的接收者和发送者的shutdown置1.
14. 使用nanosleep()睡眠第一个接收者的delay时间。
15. 在for循环里确保每个发送者和接收者都结束了。
16. 在for循环里删除所有接收者的信号量集合标识符。这也是`nosem`标志的位置，所有信号量出问题都会跳转到此处。
17. 如果mustfork置位，则表示创建的是进程，需要取消内存映射，删除对应的共享内存对象"/sigwaittest"。
18. 执行return 返回。
## semathread解析

1. 使用pthread_sigmask()清空当前进程的信号屏蔽字。
2. 使用sched_setscheduler()为当前进程设置实时调度SCHED_FIFO和优先级。
3. 如果用户没有指定亲和性，则使用sched_setaffinity()为线程设定亲和性。
4. 如果用户指定了亲和性，将mustgetcpu置1。
5. 如果wasforked为0，则表明创建的是进程，使用gettid()获取进程号。
6. 如果par->shutdown为0，表明当前线程没执行完毕，继续在while()循环里执行该线程的动作：
    1. 如果是发送者，则
        1. 用gettimeofday()记录当前时刻。
        2. 用semop()为信号量集里编号为SEM_WAIT_FOR_SENDER的信号量执行+1操作
        3. 用par->samples++记录本次信号量的操作(即循环次数)
        4. 如用户指定了循环次数，且本线程的循环达标，则par->shutdown置1，此时满足退出while循环的条件。
        5. 如mustgetcpu为1，则用get_cpu()为par->cpu赋值。
        6. 用semop()为信号量集里编号为SEM_WAIT_FOR_RECEIVER的信号量执行-1操作
        7. 用semop()为信号量集里编号为SEM_WAIT_FOR_SENDER的信号量执行-1操作
    1. 如果是接收者，则
        1. 用neighbor代表接收者线程的数据结构。如wasforked为1代表创建的是进程，则通过发送者线程的数据结构par加上它们之间的距离par->num_threads来定位到neighbor；如wasforked为0代表创建的是线程，则通过发送者线程的par->neighbor定位到neighbor。
        2. 用semop()为信号量集里编号为SEM_WAIT_FOR_SENDER的信号量执行-1操作。
        3. 用gettimeofday()记录当前的时刻。
        4. 用par->samples++记录本次信号量的操作(即循环次数)
        5. 判断是否满足为par->shutdown置1的条件。
        6. 如mustgetcpu为1，则用get_cpu()为par->cpu赋值。
        7. 用timersub()记算(par->received - neighbor->unblocked)的值，结果保存在par->diff。
        8. 计算par->mindiff，par->maxdiff, par->sumdiff的值。
        9. 如果用户定义了-b选项进行中断追踪，且满足条件，则向追踪使能文件里写入0，并给这一对发送者和接收者的shutdown都置1。
        10. 使用semop()为信号量集合里编号为SEM_WAIT_FOR_RECEIVER的信号量执行+1操作。
        11. 使用nanosleep()睡眠par->delay的时间。
        12. 使用semop()为信号量集合里编号为SEM_WAIT_FOR_SENDER的信号量执行+1操作
7. 如par->sender为1，则表明当前线程是发送者线程，则使用semop()在信号量集里的两个信号量上执行+1操作。
8. 将par->stopped置1，表示线程即将结束。
9. return返回，线程结束。
