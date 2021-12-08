# hwlatdetect 文档
#  概述
本程序用来控制内核模块hwlat_detector.ko，这个内核模块是用来探测硬件时延的，注意这个时延和Linux内核自身无关。
本程序刚开始是用来探测x86上的SMIs（系统管理中断）。SMIs不由内核管理，内核一般也感知不到它。SMIs由BIOS设置和管理，一般用于关键事件，如热传感器和风扇的管理。有时，SMIs也用来管理那些占用时间过长的任务。
hwlat_detector.ko模块的工作原理：通过调用stop_machine()占用CPU的所有可配置时间，轮询TSC（时间戳计数器），然后查找TSC之间的空隙。由于机器已经停止了，中断也关闭了，所有的间隙都表示轮询被中断的时间，只有SMI能做到了。
本程序管理着debugsf的挂载、卸载，hwlat_detector.ko模块的加载、卸载。 

# 使用
##  编译
当rt-tests测试套件安装完成后，本程序就被安装在/usr/local/bin目录下。
```
# 编译安装rt-tests测试套件
sudo apt-get install build-essential libnuma-dev    # 安装编译环境和必需的库
git clone git://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git
cd rt-tests
git checkout stable/v1.0    # master分支不是稳定版，所以要切换到stable分支
make all
make install
```
## 参数
```
  -h, --help 
        显示此帮助信息并退出
  --duration=DURATION 
        测试SMI的总时间（<n>{smdw}）。
  --threshold=THRESHOLD
        超过该值即被视为SMI（微秒）。
  --interval=INTERVAL
        采样间隔时间（毫秒）。
  --sample_width=SAMPLE_WIDTH
        实际测量的时间(毫秒)
  --report=REPORT 
        样本数据的文件名
  --cleanup 
        强制卸载模块，umount debugfs。
  --debug 
        打开调试打印功能
  --quiet 
        关闭所有屏幕输出
```
## 例子

## 数据含义


## 性能指标


# 实现方法

## 定义

## 方法

## syscall

## 补充
x86_64平台常见的时钟硬件有以下这些

RTC(Real Time Clock)
实时时钟的独特之处在于，RTC是主板上一块电池供电的CMOS芯片(精度一般只到秒级)，RTC(Clock)吐出来的是”时刻”(例如”2014-2-22 23:38:44″)，而其他硬件时钟(Timer)吐出来的是”时长”(我走过了XX个周期，按照我的频率，应该是10秒钟)。

PIT(Programmable Interval Timer)
 PIT是最古老的时钟源，产生周期性的时钟中断(IRQ0)，精度在100-1000Hz，现在基本已经被HPET取代。
APIC Timer 这是PIT针对多CPU环境的升级，每个CPU上都有一个APIC Timer(而PIT则是所有CPU共享的)，但是它经常有BUG且精度也不高(3MHz左右)，所实际很少使用。

ACPI Timer(Power Management Timer)
它唯一的功能就是为每个时钟周期提供一个时间戳，用于提供与处理器速度无关的可靠时间戳。但其精度并不高(3.579545MHz)。

HPET(High Precision Event Timer)
HPET提供了更高的精度(14.31818MHz)以及更宽的计数器(64位)。HPET可以替代前述除RTC之外的所有时钟硬件(Timer)，因为它既能定时触发中断，又能维护和读取当前时间。一个HPET包含了一个固定频率的数值递增的计数器以及3-32个独立计数器，每个计数器又包含了一个比较器和一个寄存器，当两者数值相等时就会触发中断。HPET的出现将允许删除芯片组中的一些冗余的旧式硬件。2006年之后的主板基本都已支持HPET

TSC(Time Stamp Counter)
TSC是位于CPU里面的一个64位寄存器，与传统的周期性时钟不同，TSC并不触发中断，它是以计数器形式存在的单步递增性时钟。也就是说，周期性时钟是通过周期性触发中断达到计时目的，如心跳一般。而单步递增时钟则不发送中断，取而代之的是由软件自己在需要的时候去主动读取TSC寄存器的值来获得时间。TSC的精度(纳秒级)远超HPET并且速度更快，但仅能在较新的CPU(Sandy Bridge之后)上使用

# 实现分析


# 引用

