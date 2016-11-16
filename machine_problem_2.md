# Machine problem 2

##Supproting muti-programming in Nachos
I have to implement mutiprogramming by modifying page table(address space).   
Due to following:
* In Nachos, every program are mapping to the same page table.


###Implement
---------------------------------------------------------------------------
首先去查看address space，裡面會對應每個program的address。
可以看到
```C++
AddrSpace::AddrSpace()
{
    pageTable = new TranslationEntry[NumPhysPages];
    for (int i = 0; i < NumPhysPages; i++) {
        pageTable[i].virtualPage = i;   // for now, virt page # = phys page #
        pageTable[i].physicalPage = i;
        pageTable[i].valid = TRUE;
        pageTable[i].use = FALSE;
        pageTable[i].dirty = FALSE;
        pageTable[i].readOnly = FALSE;
    }

    // zero out the entire address space
    bzero(kernel->machine->mainMemory, MemorySize);
}

```
位修改前的addrspace會先新增一個pagetable，然後將virtual & physical page都動硬到同一個，並作初始化。   
在來是載入program的清況:
```C++
//----------------------------------------------------------------------
// AddrSpace::Load
//      Load a user program into memory from a file.
//
//      Assumes that the page table has been initialized, and that
//      the object code file is in NOFF format.
//
//      "fileName" is the file containing the object code to load into memory
//----------------------------------------------------------------------

bool
AddrSpace::Load(char *fileName)
{
    OpenFile *executable = kernel->fileSystem->Open(fileName);
    /*
    將program file先載入
    */
    NoffHeader noffH;
    unsigned int size;

    if (executable == NULL) {
        cerr << "Unable to open file " << fileName << "\n";
        return FALSE;
    }

    executable->ReadAt((char *)&noffH, sizeof(noffH), 0);
    /*
    將program file的header資訊放到noffH的資料結構當中
    */
    if ((noffH.noffMagic != NOFFMAGIC) &&
                (WordToHost(noffH.noffMagic) == NOFFMAGIC))
        SwapHeader(&noffH);
    ASSERT(noffH.noffMagic == NOFFMAGIC);

#ifdef RDATA
// how big is address space?
    size = noffH.code.size + noffH.readonlyData.size + noffH.initData.size +
           noffH.uninitData.size + UserStackSize;
                                                // we need to increase the size
                                                // to leave room for the stack
#else
// how big is address space?
    size = noffH.code.size + noffH.initData.size + noffH.uninitData.size
                        + UserStackSize;        // we need to increase the size
                                                // to leave room for the stack
   /*
    從program header中得到所需要的空間:
    text(code)、data(initdata)、BSS(uninitData) and stack space for function calls/local variables(UserStackSize)
    
    --------------------
    -       Stack      -
    -                  -
    -                  -
    --------------------
    -        ↓         -
    -                  -
    -        ↑         -
    --------------------
    -                  -
    -       Heap       -
    --------------------
    -       BSS        -   Block Started by Symbol   
    --------------------
    -       Data       -
    --------------------
    -       Text       -
    --------------------
    目前Linux的執行檔為ELF，詳情請自行學習。
    
    */
#endif
    numPages = divRoundUp(size, PageSize);
    size = numPages * PageSize;

    ASSERT(numPages <= NumPhysPages);           // check we're not trying
                                                // to run anything too big --
                                                // at least until we have
                                                // virtual memory

    DEBUG(dbgAddr, "Initializing address space: " << numPages << ", " << size);
// then, copy in the code and data segments into memory
// Note: this code assumes that virtual address = physical address
    if (noffH.code.size > 0) {
        DEBUG(dbgAddr, "Initializing code segment.");
        DEBUG(dbgAddr, noffH.code.virtualAddr << ", " << noffH.code.size);
        executable->ReadAt(
                &(kernel->machine->mainMemory[noffH.code.virtualAddr]),
                        noffH.code.size, noffH.code.inFileAddr);
        /*
        將code segments存放到noffH.code.virtualAddr位址。
        接下來也都是如法炮製。
        */
    }
    if (noffH.initData.size > 0) {
        DEBUG(dbgAddr, "Initializing data segment.");
        DEBUG(dbgAddr, noffH.initData.virtualAddr << ", " << noffH.initData.size);
        executable->ReadAt(
                &(kernel->machine->mainMemory[noffH.initData.virtualAddr]),
                        noffH.initData.size, noffH.initData.inFileAddr);
    }

#ifdef RDATA
    if (noffH.readonlyData.size > 0) {
        DEBUG(dbgAddr, "Initializing read only data segment.");
        DEBUG(dbgAddr, noffH.readonlyData.virtualAddr << ", " << noffH.readonlyData.size);
        executable->ReadAt(
                &(kernel->machine->mainMemory[noffH.readonlyData.virtualAddr]),
                        noffH.readonlyData.size, noffH.readonlyData.inFileAddr);
    }
#endif

    delete executable;                  // close file
    return TRUE;                        // success
}

```
很重要的一點是`virtual address = physical address`，所以會造成code segment會跟其他program重疊，所以我們必須要記錄每個program所使用到的pyhsical address，並發現此physical有被使用，就繼續往下搜尋可用的physical address其他的，這樣就可以保證不互相重疊。   
這邊會使用到的技巧是<a href="http://openhome.cc/Gossip/CppGossip/staticMember.html/">C++中的static</a>，因此在AddressSpace這個class底下，會共享一個`static ALLphysicalpage`，用來記錄所有pagetable的使用情況。

---------------------------------------------------------------------------
因此我們先在class中新增一個`static ALLphysicalpage`   
```C++
class AddrSpace {
  public:
    AddrSpace();                        // Create an address space.
    ~AddrSpace();                       // De-allocate an address space

    static int ALLphysicaltable[NumPhysPages];
    /*
    machine/machine.h中有記錄const int NumPhysPages = 128;
    所以可以之都main memory中有多少個physical page。
    */
    bool Load(char *fileName);          // Load a program into addr space from
                                        // a file
                                        // return false if not found

    void Execute(char *fileName);               // Run a program
                                        // assumes the program has already
                                        // been loaded

    void SaveState();                   // Save/restore address space-specific
    void RestoreState();                // info on a context switch
    
    // Translate virtual address _vaddr_
    // to physical address _paddr_. _mode_
    // is 0 for Read, 1 for Write.
    ExceptionType Translate(unsigned int vaddr, unsigned int *paddr, int mode);

  private:
    TranslationEntry *pageTable;        // Assume linear page table translation
                                        // for now!
    unsigned int numPages;              // Number of pages in the virtual
                                        // address space

    void InitRegisters();               // Initialize user-level CPU registers,
                                        // before jumping to user code

};

```
並在.cc中增`int AddrSpace::ALLpyhsicaltable[NumPhysPages]={-1};`表示位使用過，也可用bool取代int來節省記憶體。   
並將AddrSpace::AddrSpace()中執行的是都移到load時才執行，因此AddrSpace::AddrSpace不會執行任何東西。

--------------------------------------------------------
####在bool AddrSpace::Load(char*)中:   
我們在` numPages = divRoundUp(size, PageSize);//相除取上限`後，可以得到program需要多少個page。
並新增
```C++
pageTable = new TranslationEntry[numPages];
/*
將AddrSpace::AddrSpace()中的程式修改
*/
unsigned int j = 0;//使用unsigned是因為記憶體位址沒有正負號差別，避免特殊狀況造成bug加上，在Nachos這邊就算是用signed也不會有bug，因為總共只有128個pages
for(int i=0;i<numPages;i++)
{
  pageTable[i].virtualPage = i;   // for now, virt page # = phys page #
  while(j < NumPhysPages && AddrSpace::ALLphysicaltable[j] == 1 )
        j++;
  AddrSpace::ALLphysicaltable[j] = 1;
  pageTable[i].physicalPage = j;
  /*
  找到未使用的位址，將此physical page給予這個program使用。
  */
  pageTable[i].valid = TRUE;
  pageTable[i].use = FALSE;
  pageTable[i].dirty = FALSE;
  pageTable[i].readOnly = FALSE;

}
```
但是我們要如何轉換virtual跟physical呢? 我們可以參考:
```C++
ExceptionType
AddrSpace::Translate(unsigned int vaddr, unsigned int *paddr, int isReadWrite)
{
    TranslationEntry *pte;
    int               pfn;
    unsigned int      vpn    = vaddr / PageSize;
    unsigned int      offset = vaddr % PageSize;

    if(vpn >= numPages) {
        return AddressErrorException;
    }

    pte = &pageTable[vpn];

    if(isReadWrite && pte->readOnly) {
        return ReadOnlyException;
    }

    pfn = pte->physicalPage;

    // if the pageFrame is too big, there is something really wrong!
    // An invalid translation was loaded into the page table or TLB.
    if (pfn >= NumPhysPages) {
        DEBUG(dbgAddr, "Illegal physical page " << pfn);
        return BusErrorException;
    }

    pte->use = TRUE;          // set the use, dirty bits

    if(isReadWrite)
        pte->dirty = TRUE;

    *paddr = pfn*PageSize + offset;

    ASSERT((*paddr < MemorySize));

    //cerr << " -- AddrSpace::Translate(): vaddr: " << vaddr <<
    //  ", paddr: " << *paddr << "\n";

    return NoException;
}
```
可以得出physical address = pageTable[virtual/PageSize].physicalPage * PageSize + virtual % PageSize。   
因此我們知道在放置code segments時，該如何有效的放到physical page address。
```C++
// then, copy in the code and data segments into memory
// Note: this code assumes that virtual address = physical address
    if (noffH.code.size > 0) {
        DEBUG(dbgAddr, "Initializing code segment.");
        DEBUG(dbgAddr, noffH.code.virtualAddr << ", " << noffH.code.size);
        /*executable->ReadAt(
                &(kernel->machine->mainMemory[noffH.code.virtualAddr]),
                        noffH.code.size, noffH.code.inFileAddr);*/
        executable->ReadAt(
                &(kernel->machine->mainMemory[pageTable[noffH.code.virtualAddr/PageSize].physicalPage * PageSize + noffH.code.virtualAddr%PageSize]),
                        noffH.code.size, noffH.code.inFileAddr);
    }
    if (noffH.initData.size > 0) {
        DEBUG(dbgAddr, "Initializing data segment.");
        DEBUG(dbgAddr, noffH.initData.virtualAddr << ", " << noffH.initData.size);
        executable->ReadAt(
                &(kernel->machine->mainMemory[pageTable[noffH.initData.virtualAddr/PageSize].physicalPage * PageSize + noffH.initData.virtualAddr%PageSize]]),
                        noffH.initData.size, noffH.initData.inFileAddr);
    }
```
這個計算是先算出virtual address是在program本身pagetable中的哪一個page(也因為pageTable[i].virtualPage=i，所以可以直接透過virtual address得到pageTable對應的page內容)，然後再取得此page的physicalPage的id，在乘上page大小，以及加上offset，這樣就可以捯到physical address。   
這樣就能夠讓不同的program可以使用到不重疊的segments page。   
最後還需要在結束program時，釋放所佔用的page資源。透過修改`AddrSpace::~AddrSpace()`可得
```C++
AddrSpace::~AddrSpace()
{
   for(int i =0;i<numPages;i++){
        AddrSpace::ALLphysicaltable[pageTable[i].physicalPage] = -1;
   }
   delete pageTable;
}
```
最後能夠執行multiprobramming，但在output時，PrintInt()並沒有被lock，因此結果很亂，可能會有不同program的數字連在一起等問題，需要去改一下PrintInt()。