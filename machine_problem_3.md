# Machine problem 3

####To implement a multilevel feedback queue for CPU scheduling.

```

       __________________________
CPU<---| L1:100~149             |<-┐   /*Using a shortest job first scheduling*/
       |________________________|  |   /*The approximated equation: t(i)=0.5*T+0.5*t(i-1)*/
    _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _| 
    |  __________________________      
    └--| L2:50~99               |<-┐   /*Using a priority scheduling*/
       |________________________|  |
    _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _| 
    |  __________________________
    └--| L3:0~49                |      /*Using a round-boin scheduling*/
       |________________________|      /*The time quantum is 100 ticks instead of 500 ticks*/

```
* Priority value between 0 to 149
* 149 is highest priority
* An aging mechanism
  * Increased by 10 after waiting for every 1500 ticks
* Implementation of "-eq" argument for Nachos command line to initial processes/threads priority value


--------------------------------------------
首先先進入到thread/main.cc看到
```C++

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
kernel物件初始化時會開啟scheduler，並在Exec中新增thread到scheduler。
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
kernel在main是global variable，所以其他物件都可以存取到kernel物件的位址。在Fork()當中，把thread(`this`)放到scheduler當中的readytorun。可以在thread/scheduler.cc中看到
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