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
在timer給予interrupt可以看`thread/alarm.cc`跟`machine/timer.*`
```c++
Alarm::Alarm(bool doRandom)
{
    timer = new Timer(doRandom, this);
}
```
這邊可以看到kernel新增一個alarm物件的時候其實是去設定timer，並且把alarm的物件給予timer(this是alarm物件)。
```c++
Timer::Timer(bool doRandom, CallBackObj *toCall)
{
    randomize = doRandom;
    callPeriodically = toCall;
    disable = FALSE;
    SetInterrupt();
}
```
timer就會設定發生time out的handler，也就是callPeriodically(alarm->CallBack())。最後就是把timer interrupt放到interrupt的schedule當中。並且可以由`machine/stat.h`得知interrupt的period為100 ticks。
```c++
void
Timer::SetInterrupt()
{
    if (!disable) {
       int delay = TimerTicks;

       if (randomize) {
             delay = 1 + (RandomNumber() % (TimerTicks * 2));
        }
       // schedule the next timer device interrupt
       kernel->interrupt->Schedule(this, delay, TimerInt);
    }
}
/*machine/stat.h
const int UserTick =       1;   // advance for each user-level instruction
const int SystemTick =    10;   // advance each time interrupts are enabled
const int RotationTime = 500;   // time disk takes to rotate one sector
const int SeekTime =     500;   // time disk takes to seek past one track
const int ConsoleTime =  100;   // time to read or write one character
const int NetworkTime =  100;   // time to send or receive one packet
const int TimerTicks =   100;   // (average) time between timer interrupts
*/
```
然後tick(onetick)作運算的時候，都會透過CheckIfDue去檢查interrupt當中的scheduler裡面是否有time out的情況。
```c++
bool
Interrupt::CheckIfDue(bool advanceClock)
{
    PendingInterrupt *next;
    Statistics *stats = kernel->stats;

    ASSERT(level == IntOff);            // interrupts need to be disabled,
                                        // to invoke an interrupt handler
    if (debug->IsEnabled(dbgInt)) {
        DumpState();
    }
    if (pending->IsEmpty()) {           // no pending interrupts
        return FALSE;
    }
    next = pending->Front();

    if (next->when > stats->totalTicks) {
        if (!advanceClock) {            // not time yet
            return FALSE;
        }
        else {                  // advance the clock to next interrupt
            stats->idleTicks += (next->when - stats->totalTicks);
            stats->totalTicks = next->when;
            // UDelay(1000L); // rcgood - to stop nachos from spinning.
        }
    }

    DEBUG(dbgInt, "Invoking interrupt handler for the ");
    DEBUG(dbgInt, intTypeNames[next->type] << " at time " << next->when);

    if (kernel->machine != NULL) {
        kernel->machine->DelayedLoad(0, 0);
    }

    inHandler = TRUE;
    do {
        next = pending->RemoveFront();    // pull interrupt off list
        next->callOnInterrupt->CallBack();// call the interrupt handler
        delete next;
    } while (!pending->IsEmpty()
                && (pending->Front()->when <= stats->totalTicks));
    inHandler = FALSE;
    return TRUE;
}

void
Interrupt::Schedule(CallBackObj *toCall, int fromNow, IntType type)
{
    int when = kernel->stats->totalTicks + fromNow;
    PendingInterrupt *toOccur = new PendingInterrupt(toCall, when, type);

    DEBUG(dbgInt, "Scheduling interrupt handler the " << intTypeNames[type] << " at time = " << when);
    ASSERT(fromNow > 0);

    pending->Insert(toOccur);
}
```
若有發現pending(interrupt scheduler)裡面的event有發生time out，就交給CallBack()去執行event interrupt。  

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
    int stick;
```
並且初始化Thread所需要的設定。
```C++
Thread::Thread(char* threadName, int threadID)
{
        ID = threadID;
    name = threadName;
    stackTop = NULL;
    stack = NULL;
    status = JUST_CREATED;
    stick = -1;//先將執行的時間設定為-1
    this.setbrust(0); //並且將t(0)=0 設定上去
    for (int i = 0; i < MachineStateSize; i++) {
        machineState[i] = NULL;         // not strictly necessary, since
                                        // new thread ignores contents
                                        // of machine registers
    }
    space = NULL;
}
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
        L1_readyList->Insert(thread);
        Thread *oldThread = kernel->currentThread;
        int oldbrust = 0.5*oldThread->getbrust()+0.5*(kernel->stats->userTicks-oldThread->stick);
        //只算usertick，因為我們scheduler只負責user thread的schedule
        //若包含systick(totalticks)，這樣會無法知道我們thread實際上使用CPU的時間
        if(oldbrust > thread->getbrust()){
            kernel->currentThread->Yield();
        }
    }

    else if(priority>49)
        L2_readyList->Insert(thread);
    else
        L3_readyList->Append(thread);
    //readyList->Append(thread);
}
```
將不同優先權的thread放到不同的queue當中，並且L1是可以搶佔的，所以在計算新近thread的approximated job execution time比較小時，就呼叫`Thread::Yield ()`產生interrupt，並且在`Thread::Yield () 當中的 kernel->scheduler->Run(nextThread, FALSE);`時，發生context switch，來完成。並且可以從`Yield()`當中看到，會先將本身的thread在被SWITCH之前，會先執行`ReadyToRun(this)`，也就是先放到scheduler後在進行switch。
```c++
void
Thread::Yield ()
{
    Thread *nextThread;
    IntStatus oldLevel = kernel->interrupt->SetLevel(IntOff);

    ASSERT(this == kernel->currentThread);

    DEBUG(dbgThread, "Yielding thread: " << name);

    nextThread = kernel->scheduler->FindNextToRun();
    if (nextThread != NULL) {
        kernel->scheduler->ReadyToRun(this);
        kernel->scheduler->Run(nextThread, FALSE);
    }
    (void) kernel->interrupt->SetLevel(oldLevel);
}
```
```c++
void
Scheduler::Run (Thread *nextThread, bool finishing)
{
    Thread *oldThread = kernel->currentThread;

    ASSERT(kernel->interrupt->getLevel() == IntOff);
    int oldbrust = 0.5*oldThread->getbrust()+0.5*(kernel->stats->userTicks-oldThread->stick);
    oldThread->setbrust(oldbrust);
    ...
    nextThread->stick = kernel->stats->userTicks;
    DEBUG(dbgThread, "Switching from: " << oldThread->getName() << " to: " << nextThread->getName());

    // This is a machine-dependent assembly language routine defined
    // in switch.s.  You may have to think
    // a bit to figure out what happens after this, both from the point
    // of view of the thread and from the perspective of the "outside world".

    SWITCH(oldThread, nextThread);//儲存oldthread，並讀取nextthread
    ...
}
```
在`Run()`中設定nextTread開始執行的時間，方便計算CPUBrust。也在這邊設定oldThread的brust。這裡不管thread從running狀態移到waiting或是ready，都可以設定到。包含preemtive或是I/O interrupt。