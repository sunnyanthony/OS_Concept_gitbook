# Machine problem 3

####To implement a multilevel feedback queue for CPU scheduling.

```

       __________________________
CPU<---| L1:100~149             |--┐   /*Using a shortest job first scheduling*/
       |________________________|  |   /*The approximated equation: t(i)=0.5*T+0.5*t(i-1), t(0) =0*/
    _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _|   /*preemptive*/
    |  __________________________      
    └->| L2:50~99               |--┐   /*Using a priority scheduling*/
CPU<---|________________________|  |   
    _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _| 
    |  __________________________
    └->| L3:0~49                |      /*Using a round-boin scheduling*/
CPU<---|________________________|      /*The time quantum is 100 ticks instead of 500 ticks*/

```
* Priority value between 0 to 149
* 149 is highest priority
* An aging mechanism
  * Increased by 10 after waiting for every 1500 ticks
* Implementation of "-eq" argument for Nachos command line to initial processes/threads priority value


--------------------------------------------
首先先進入到thread/main.cc看到
```C++
kernel = new Kernel(argc, argv);
kernel->Initialize();
...
kernel->ExecAll();
```
之後可以看到thread/kernel.cc
```C++
void
Kernel::Initialize()
{
    // We didn't explicitly allocate the current thread we are running in.
    // But if it ever tries to give up the CPU, we better have a Thread
    // object to save its state.

    currentThread = new Thread("main", threadNum++);
    currentThread->setStatus(RUNNING);

    stats = new Statistics();           // collect statistics
    interrupt = new Interrupt;          // start up interrupt handling
    scheduler = new Scheduler();        // initialize the ready queue
    alarm = new Alarm(randomSlice);     // start up time slicing
    machine = new Machine(debugUserProg);
    ...
    interrupt->Enable();
}
```
kernel物件初始化時會開啟scheduler，並在Exec中新增thread到scheduler。並且初始化alarm，會設定machine/stats.h的TimerTicks到timer物件當中，用來設定rr的時間。
```C++
int Kernel::Exec(char* name)
{
        t[threadNum] = new Thread(name, threadNum);
        t[threadNum]->space = new AddrSpace();
        t[threadNum]->Fork((VoidFunctionPtr) &ForkExecute, (void *)t[threadNum]);
        threadNum++;

        return threadNum-1;
}
```
可以在`thread()`中，看到如何放入scheduler。檔案是在thread/thread.cc中。
```C++
void
Thread::Fork(VoidFunctionPtr func, void *arg)
{
    Interrupt *interrupt = kernel->interrupt;
    Scheduler *scheduler = kernel->scheduler;
    IntStatus oldLevel;

    DEBUG(dbgThread, "Forking thread: " << name << " f(a): " << (int) func << " " << arg);
    StackAllocate(func, arg);

    oldLevel = interrupt->SetLevel(IntOff);
    scheduler->ReadyToRun(this);        // ReadyToRun assumes that interrupts
                                        // are disabled!
    (void) interrupt->SetLevel(oldLevel);
}
```
kernel在main是global variable，所以其他物件都可以存取到kernel物件的位址。在Fork()當中，把thread(`this`)放到scheduler當中的readytorun。並且在執行scheduler前，會開啟interrupt，。可以在thread/scheduler.cc中看到
```C++
void
Scheduler::ReadyToRun (Thread *thread)
{
    ASSERT(kernel->interrupt->getLevel() == IntOff);
    DEBUG(dbgThread, "Putting thread on ready list: " << thread->getName());
        //cout << "Putting thread on ready list: " << thread->getName() << endl ;
    thread->setStatus(READY);
    readyList->Append(thread);
}
```
thread會被放到readyList當中，並且thread的status會被設為READY。  

-------------
了解大致上的運作之後，我們先從thread/thread.h跟thread.cc開始下手。
我相我們應該要先將thread增加priority/brust跟設定priority/brust的method。
```C++
private:
    // NOTE: DO NOT CHANGE the order of these first two members.
    // THEY MUST be in this position for SWITCH to work.
    int *stackTop;                       // the current stack 
    ...
    int priority;
    int CPUbrust;
public:
...
    // basic thread operations
    void setpriority(int p);
    int getpriority();
    void setbrust(int b){CPUburst = b;}
    int getbrust(){return CPUburst;}
```
這邊新增`priority`private變數，並透過`setpriority()`跟`getpriority()`做存取。在thread.cc中實作。
```C++
//--------------------
//Setting and getting priority.
//--------------------
void
Thread::setpriority(int p)
{
   priority = p;
}
int
Thread::getpriority()
{
   return priority;
}
```
接下來去修改thread/scheduler.h，新增三個queue。
```C++
int priority(Thread *a, Thread *b){
    int apriority = a->getpriority();
    int bpriority = b->getpriority();
    return ((apriority == bpriority) ? 0 : ((apriority > bpriority)? 1 : -1));
}
int sjfs(Thread *a, Thread *b){
    int apriority = a->getbrust();
    int bpriority = b->getbrust();
    return ((apriority == bpriority) ? 0 : ((apriority > bpriority)? 1 : -1));
}
Scheduler::Scheduler()
{
    L1_readyList = new Sorted<Thread *>(sjfs);
    L2_readyList = new SortedList<Thread *>(priority);
    L3_readyList = new List<Thread *>;
    toBeDestroyed = NULL;
}

```
之後更改`Scheduler::ReadToRun()`
```c++
void
Scheduler::ReadyToRun (Thread *thread)
{
    ASSERT(kernel->interrupt->getLevel() == IntOff);
    DEBUG(dbgThread, "Putting thread on ready list: " << thread->getName());
        //cout << "Putting thread on ready list: " << thread->getName() << endl ;
    thread->setStatus(READY);
    int priority;
    priority = thread->getpriority();
    if(priority>99){
        Thread *oldThread = kernel->currentThread;
        L1_readyList->Insert(thread);
        if(oldThread->getbrust() > thread->getbrust())
            kernel->currentThread->Yield();
    }
    else if(priority>49)
        L2_readyList->Insert(thread);
    else
        L3_readyList->Append(thread);
    //readyList->Append(thread);
}
```
將不同優先權的thread放到不同的queue當中，並且L1是可以搶佔的，所以在計算新近thread的approximated job execution time比較小時，就呼叫`Thread::Yield ()`產生interrupt，並且在`Thread::Yield () 當中的 kernel->scheduler->Run(nextThread, FALSE);`時，發生context switch，來完成。