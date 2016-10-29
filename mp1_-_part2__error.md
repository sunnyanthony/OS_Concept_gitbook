# 之前搞錯的MP1-part2(error

```C++
bool
FileSystem::Create(char *name, int initialSize)
{
    Directory *directory;
    PersistentBitmap *freeMap;
    FileHeader *hdr;
    int sector;
    bool success;

    DEBUG(dbgFile, "Creating file " << name << " size " << initialSize);

    directory = new Directory(NumDirEntries);
    directory->FetchFrom(directoryFile);

    if (directory->Find(name) != -1)
      success = FALSE;                  // file is already in directory
    else {
        freeMap = new PersistentBitmap(freeMapFile,NumSectors);
        sector = freeMap->FindAndSet(); // find a sector to hold the file header
        if (sector == -1)
            success = FALSE;            // no free block for file header
        else if (!directory->Add(name, sector))
            success = FALSE;    // no space in directory
        else {
            hdr = new FileHeader;
            if (!hdr->Allocate(freeMap, initialSize))
                success = FALSE;        // no space on disk for data
            else {
                success = TRUE;
                // everthing worked, flush all changes back to disk
                hdr->WriteBack(sector);
                directory->WriteBack(directoryFile);
                freeMap->WriteBack(freeMapFile);
            }
            delete hdr;
        }
        delete freeMap;
    }
    delete directory;
    return success;
}
```
我先來講解一下creat的操作，首先可以看到Filesystem會去directory當中檢查file是否存在，若沒有才會去新增。接下來```BitMap()```是用來去Disk上操作的，主要是找到空間放置file header。得到空間位置會去檢查disk中有沒有free block，還有檢查directory中有沒有空間放置檔案(不同的os會有不同限制，傳統的unix會限制15...映像中是這樣，會在去查證一下)，若是directory有空間放置檔案，救會將檔案名稱跟sector加入到directory當中。最後是新增一個file header，並且allocateu一塊disk空間存放file data，最後是將file寫入disk中(在unix中使用fflush指令可作到)。


再來是看一下我們最主要要使用到的```FileSystem::Open(char *name)```
```C
OpenFile *
FileSystem::Open(char *name)
{ 
    Directory *directory = new Directory(NumDirEntries);
    OpenFile *openFile = NULL;
    int sector;

    DEBUG(dbgFile, "Opening file" << name);
    directory->FetchFrom(directoryFile);
    sector = directory->Find(name); 
    if (sector >= 0) 		
	openFile = new OpenFile(sector);	// name was found in directory 
    delete directory;
    return openFile;				// return NULL if not found
}
```
Open主要是先得到sector，然後再去新增一個openfile的物件。我們主要就是透過openfile object來read和write檔案。然而我們可以透過回傳的openFile來進行操作。

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
        return (int)&(kernel->interrupt->OpenFile(filename));
}
```
這主要是取得openfile object的address並且轉成integer。

