# JAVA 线程与 操作系统线程


## reason : 查看 jstack 输出时，看到有 tid 、 nid 的输出，
需要理清jvm 线程、linux 系统线程的关系

jstack 一个典型输出如下:

```shell
2021-02-07 20:59:10 
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.281-b09 mixed mode):

"Attach Listener" #52 daemon prio=9 os_prio=0 tid=0x00007efde0001000 nid=0x9dcb2 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"DestroyJavaVM" #51 prio=5 os_prio=0 tid=0x00007efe48012000 nid=0x9dc1f waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
 
"http-nio-8938-AsyncTimeout" #49 daemon prio=5 os_prio=0 tid=0x00007efe4a3ab800 nid=0x9dc64 waiting on condition [0x00007efe0eef1000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at org.apache.coyote.AbstractProtocol$AsyncTimeout.run(AbstractProtocol.java:1151)
    at java.lang.Thread.run(Thread.java:748)
 
"http-nio-8938-Acceptor-0" #48 daemon prio=5 os_prio=0 tid=0x00007efe4a56e000 nid=0x9dc63 runnable [0x00007efe0eff2000]
   java.lang.Thread.State: RUNNABLE
    at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)
    at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:424)
    at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:252)
    - locked <0x00000007802a88c8> (a java.lang.Object)
    at org.apache.tomcat.util.net.NioEndpoint$Acceptor.run(NioEndpoint.java:482)
    at java.lang.Thread.run(Thread.java:748)
 
```

其中有两个字段 tid、nid 看着都像是 thead id



## jstack 中的 nid
通过ps、pstack、top 等命令，和上面的nid对比可知道 jstack 每个线程里面的 nid 对应者linux本地的线程号. man ps 时，可以看到关于 linux 中线程称为为 "tid"(注意和jstack中的"tid"不是同一个意思),linux本地线程又叫 "tid"、"lwp"(light weight process)、"spid"

参考 openjdk 源码, 在 hotspot  的源码里grep "waiting on condition" 可找到 nid 相关信息

```c++
/** hotspot/src/share/vm/runtime/osThread.cpp **/

// Printing
void OSThread::print_on(outputStream *st) const {
  st->print("nid=0x%lx ", thread_id());
  switch (_state) {
    case ALLOCATED:               st->print("allocated ");                 break;
    case INITIALIZED:             st->print("initialized ");               break;
    case RUNNABLE:                st->print("runnable ");                  break;
    case MONITOR_WAIT:            st->print("waiting for monitor entry "); break;
    case CONDVAR_WAIT:            st->print("waiting on condition ");      break;
    case OBJECT_WAIT:             st->print("in Object.wait() ");          break;
    case BREAKPOINTED:            st->print("at breakpoint");               break;
    case SLEEPING:                st->print("sleeping");                    break;
    case ZOMBIE:                  st->print("zombie");                      break;
    default:                      st->print("unknown state %d", _state); break;
  }
}


// 可以看到是 OSThread 类的print_on方法打印的信息, 其中 nid 取值为 thread_id()返回值,可以看看 OSThread 类的头文件:


/** hotspot/src/share/vm/runtime/osThread.hpp **/

class OSThread: public CHeapObj<mtThread> {
  friend class VMStructs;

  // 省略 ...

 public:
  // 省略 ...
  thread_id_t thread_id() const                   { return _thread_id; }
  // 省略 ...

 private:
  // _thread_id is kernel thread id (similar to LWP id on Solaris). Each
  // thread has a unique thread_id (BsdThreads or NPTL). It can be used
  // to access /proc.
  thread_id_t _thread_id;

};
```

即jstack 中的 "nid"(大概率是 "native id") 即是 os 中的 内核线程 id, 到这里为止，除了得知 jstack "nid" 含义外，大概还是揣测 jvm 线程实现利用的是 内核线程，可能又叫做 light weight process，即轻量进程，从 jvm 线程调度、平时ps 中java的线程情况看，极有可能是 一个jvm 线程对应一个 内核线程, 具体对错需要再调研 jvm 的线程实现


## jstack 中的 tid
```
参考 openjdk 源码的实现，查找jstack 每个线程输出中的 "os_prio" 关键词, 代码大概在 hotspot 目录实现里, grep 一下即可找到源码位置，在 hotspot/src/share/vm/runtime/thread.cpp 里 
```

```c++

/** hotspot/src/share/vm/runtime/thread.cpp **/
void Thread::print_on(outputStream* st) const {
  // get_priority assumes osthread initialized
  if (osthread() != NULL) {
    int os_prio;
    if (os::get_native_priority(this, &os_prio) == OS_OK) {
      st->print("os_prio=%d ", os_prio);
    }
    st->print("tid=" INTPTR_FORMAT " ", this); // INTPTR_FORMAT 其实就是 0x%16llx, printf("%x", this) 的意思
    ext().print_on(st);
    osthread()->print_on(st);
  }
  debug_only(if (WizardMode) print_owned_locks_on(st);)
}

...
...

// INTPTR_FORMAT 宏定义
/** hotspot/src/share/vm/utilities/globalDefinitions.hpp **/
// Format pointers which change size between 32- and 64-bit.
#ifdef  _LP64
#define INTPTR_FORMAT "0x%016" PRIxPTR
#define PTR_FORMAT    "0x%016" PRIxPTR
#else   // !_LP64
#define INTPTR_FORMAT "0x%08"  PRIxPTR
#define PTR_FORMAT    "0x%08"  PRIxPTR
#endif  // _LP64


// PRIxPTR 宏定义, 这个有几个地方有定已，暂时没看出哪个定义生效
/** hotspot/src/share/vm/utilites/globalDefinitions_visCPP.hpp **/
#ifdef _LP64
#define PRIdPTR       "I64d"
#define PRIuPTR       "I64u"
#define PRIxPTR       "I64x"
#else
#define PRIdPTR       "d"
#define PRIuPTR       "u"
#define PRIxPTR       "x"
#endif


```
其实就是 jstack 中 tid 只是 表示当前jvm线程(区别于os)`Thread` 类的一个实体的指针地址
