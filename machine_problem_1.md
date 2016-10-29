# Machine problem 1

##Part I: console I/O system call
I have to implement ```PrintInt(int number)``` system call.

###Implement
---------------------------------------------------------------------------
先開啟userprog/syscall.h  
這邊先```#define```你system call進入exception所表示的編碼

```C++
 */

#ifndef SYSCALLS_H
#define SYSCALLS_H

#include "copyright.h"
#include "errno.h"
/* system call codes -- used by the stubs to tell the kernel which system call
 * is being asked for
 */
#define SC_Halt         0
. ..
#define SC_PrintInt     25
#define SC_Add          42
#define SC_MSG          100

#ifndef IN_ASM

/* The system call interface.  These are the operations the Nachos
 * kernel needs to support, to be able to run user programs.
 *
 * Each of these is invoked by a user program by simply calling the
 * procedure; an assembly language stub stuffs the system call code
 * into a register, and traps to the kernel.  The kernel procedures
 * are then invoked in the Nachos kernel, after appropriate error checking,
 * from the system call entry point in exception.cc.
 */
 /* Stop Nachos, and print out performance stats */
void Halt();

/*PrintINt*/
void PrintInt(int integer);
/*
 * Add the two operants and return the result
 */
```
並且定義system call的formal ```void PrintInt(int integer);```

---------------------------------------------------------

在test/start.S加入MIPS code
```c
        .globl PrintInt
        .ent PrintInt
PrintInt:
        addiu $2, $0,SC_PrintInt //將剛剛define的25(SC_PrintInt)放到register2
        syscall                  //MIPS machine會自動將帶入的參數存放到register4,5,6,7
        j       $31              //執行system call
        .end PrintInt
```
----------------------------
到userprog/exception.cc修改ExceptionHandler

加入SC_PrintInt這個case number(之前define的編碼)
```c
void
ExceptionHandler(ExceptionType which)
{
    int type = kernel->machine->ReadRegister(2);
        int val;
    int status, exit, threadID, programID;
        DEBUG(dbgSys, "Received Exception " << which << " type: " << type << "\n");
    switch (which) {
    case SyscallException:
        switch(type) {
        case SC_PrintInt:
                        DEBUG(dbgSys, "PrintInt\n"); // 使用Debug mode
                        val=kernel->machine->ReadRegister(4); //將MIPS machine自動存入的參數取出
                        SysPrintInt(val);                     //去執行kernel的system call，
                        {//以下是將Program counter+4，否則會一直執行同樣的instruction
                        // set previous programm counter (debugging only)
                        kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));

                        // set programm counter to next instruction (all Instructions are 4 byte wide)
                        kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);

                        // set next programm counter for brach execution
                        kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
                        }
                        return;
        case SC_Halt:
                        ...

        case SC_Exit:
                        DEBUG(dbgAddr, "Program exit\n");
            val=kernel->machine->ReadRegister(4);
            cout << "return value:" << val << endl;
                        kernel->currentThread->Finish();
            break;
        default:
                        cerr << "Unexpected system call " << type << "\n";
                        break;
                }
                break;
        default:
                cerr << "Unexpected user mode exception " << (int)which << "\n";
                break;
    }
    ASSERTNOTREACHED();//若執行到這邊，表示程式有錯誤
}

```
-------------------
ExceptionHandler執行的```SysPrintInt(val);```會去呼叫userprog/ksyscall.h
因此我們需要加上相對應的呼叫
 ```C
 /*kernel interface for system call*/
 void SysPrintInt(int n)
{
  kernel->interrupt->PrintInt(n);//kernel執行interrupt
}
```
----------------------------
可以到thread/kernel.cc中看到```interrupt = new Interrupt;// start up interrupt handling```和```interrupt->Enable();```開啟了interrupt的service
所以我們就直接去trace machine/interrupt.cc的code
這邊我們可以直接加上Interrupt::PrintInt(int n)的處理方法
```C++
void
Interrupt::PrintInt(int n)
{
 int m=0;
 int sign=0;
 char temp[64];
 int i=0;
 if(n<0){
      sign=1;
      n=-n;//n ~=n - 1
 }
 if(n==0)
        sign=2;
 while(n>0){
        //kernel->machine->WriteMem(int(temp+i),1,(int)(n%10+'0'));
        //i++;
      temp[i++]=n%10+'0';
      n=n/10;
 }
 if(sign==1)temp[i]='-';
 else
        i--;
 //s[i]='\0';
 int add=(int)temp;
 while(i>=0){
    //kernel->machine->ReadMem(int(add+i),1,&m);
    //kernel->synchConsoleOut->SynchConsoleOutput(NULL);
    //kernel->synchConsoleOut->PutChar(char(m));
    kernel->synchConsoleOut->PutChar(char(temp[i]));
    i--;
  }
if(sign==2)
        kernel->synchConsoleOut->PutChar(char('0'));
  kernel->synchConsoleOut->PutChar(char('\n'));
}
```
這邊使用的方法較簡單，先把integer轉為string，我是使用temp[64]佔存起來，然後在一個char一個char印出來，這邊要注意的是temp中的string跟原本的num是倒過來的，所以要從最後一個開始印出。
而```kernel->synchConsoleOut->PutChar();```可以在userprog/synchconsole.cc當中找到。
```synchConsoleOut->PutChar()```會呼叫在machine/console中的```ConsoleOutput::PutChar(char ch)
```執行將char寫入到output(default is stdout 我猜的)。


##Part II: File I/O system call
I have to implement four file I/O system call
* ######```OpenFileIdOpen(char*name);```
open a file with the name and return its fileId
* ######```int Write(char *buffer, int size, OpenFileIdid);```
write “size” characters from buffer into the file  
return number of characters actually written to the file
* ######```int Read(char *buffer, int size, OpenFileIdid);```
read “size” characters from the file and copy them into buffer  
return number of characters actually read from the file
* ######```int Close(OpenFileIdid);```
return 1 if successfully close the file, 0 otherwise

###Implement
---------------------------------------------------------------------------
* 首先先在userprog/syscall.h當中增加fileIO的case number以及function formal
* 再來是在test/start.S中增加system call的相關register MIPS code
* 到userprog/exception.cc增加ExceptionHandler當中的case
但我們需要先去查看filesys底下的openfile class跟filesys class，因為在處理interrupt時，fileIO的操作kernel是交付給file system處理的。因此我們現去處理filesystem在處理exception。
Note. 不過這時的filesys還沒有實作，所以是call底下linux的filesystem來操作。



由於我們並沒有真的實作出file system，因此是使用unix的system call完成。可以看下面的code:
```C++
#ifdef FILESYS_STUB             // Temporarily implement file system calls as
                                // calls to UNIX, until the real file system
                                // implementation is available
class FileSystem {
  public:
    FileSystem() { for (int i = 0; i < 20; i++) fileDescriptorTable[i] = NULL; }

    bool Create(char *name) {
        int fileDescriptor = OpenForWrite(name);

        if (fileDescriptor == -1) return FALSE;
        Close(fileDescriptor);
        return TRUE;
        }

    //OpenFile* Open(char *name) {
    int Open(char *name) {
          int fileDescriptor = OpenForReadWrite(name, FALSE);
          //先透過Unix的open()取得file descriptor
          if (fileDescriptor == -1) return NULL;
          //return new OpenFile(fileDescriptor);
          OpenFile* f = new OpenFile(fileDescriptor);
          fileDescriptorTable[fileDescriptor]=f; //將file pointer放入table中
          return fileDescriptor;
      }
    int Read(int fd, char* buf, int size){
       if(fd<0 || fd>19) //檢查是否有超過filedescriptorteble的界線
              return FALSE;
       OpenFile* fp =  fileDescriptorTable[fd];//取得table中的object
       if(fp==NULL)
              return FALSE;
       else
              return fp->Read(buf,size);//對object進行操作
    }
    bool Remove(char *name) { return Unlink(name) == 0; }

        OpenFile *fileDescriptorTable[20];

};

#else // FILESYS
```

主要事先透過unix提供的open取得file descriptor，然後在給OpenFile操作。
```C++
int
OpenForReadWrite(char *name, bool crashOnError)
{
    int fd = open(name, O_RDWR, 0);

    ASSERT(!crashOnError || fd >= 0);
    return fd;
}
```

------
補充 `extern "C" {}`是在Cpp當中跟c聯接所使用的，這是避免使用c的library時被cpp給更改。
`#ifdef __cplusplus`也可以被用來保護cpp的程式

------


再來是看一下我們使用到的```FileSystem::Open(char *name)```
```C
#ifdef FILESYS_STUB                     // Temporarily implement calls to
                                        // Nachos file system as calls to UNIX!
                                        // See definitions listed under #else
class OpenFile {
  public:
    OpenFile(int f) { file = f; currentOffset = 0; }    // open the file
    ~OpenFile() { Close(file); }                        // close the file

    int ReadAt(char *into, int numBytes, int position) {
                Lseek(file, position, 0);
                return ReadPartial(file, into, numBytes);
                }
    int WriteAt(char *from, int numBytes, int position) {
                Lseek(file, position, 0);
                WriteFile(file, from, numBytes);
                return numBytes;
                }
    int Read(char *into, int numBytes) {
                int numRead = ReadAt(into, numBytes, currentOffset);
                currentOffset += numRead;
                return numRead;
                }
    int Write(char *from, int numBytes) {
                int numWritten = WriteAt(from, numBytes, currentOffset);
                currentOffset += numWritten;
                return numWritten;
                }

    int Length() { Lseek(file, 0, 2); return Tell(file); }

  private:
    int file;
    int currentOffset;
};

#else // FILESYS

```
這邊會維護一個file object的position，在每次讀取實都會進行更新。主要是透過unix的read跟write來進行操作，因為沒有實作OpenFile object。

--------------------------------------------------------
那我們就回到ExceptionHandler去處理open的case
```C
 case SC_Open:
                        val = kernel->machine->ReadRegister(4);
                        {
                        char *filename = &(kernel->machine->mainMemory[val]);
                        status = SysOpen(filename);
                        kernel->machine->WriteRegister(2, (int) status);//file address
                        }
                        kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
                        kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
                        kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
                        return;
                        ASSERTNOTREACHED();
                        break;
 case SC_Read:
                        val = kernel->machine->ReadRegister(4);
                        {
                        int s = kernel->machine->ReadRegister(5); //把size從reg5讀取出來
                        int fid = kernel->machine->ReadRegister(6); //把fileID從reg6讀取出來
                        char *buf = &(kernel->machine->mainMemory[val]); //把要存入的buffer的位址取出
                        status = SysRead(fid,buf,s);//將參數傳送給ksyscall
                        kernel->machine->WriteRegister(2, (int) status);//file address
                        }
                        kernel->machine->WriteRegister(PrevPCReg, kernel->machine->ReadRegister(PCReg));
                        kernel->machine->WriteRegister(PCReg, kernel->machine->ReadRegister(PCReg) + 4);
                        kernel->machine->WriteRegister(NextPCReg, kernel->machine->ReadRegister(PCReg)+4);
                        return;
                        ASSERTNOTREACHED();
                        break;

```
先將file name從register4讀取出來，再傳送給sysopen處理，最後我們會得到status(openfloe object的address)，然後把需要return的參數寫入到register2當中。到這邊就結束open的exception處理。

------------------------------
SysOpen需要去kernel的system call中處理，開啟userprog/ksyscall.h，並加上
```C
int SysOpen(char *filename)
{
        // return value
        // negtive: failed
        // larger than 0: openfilw object address
        return kernel->OpenFile(filename);//這邊直接交給kernel去處理，不使用interrupt
}
```

