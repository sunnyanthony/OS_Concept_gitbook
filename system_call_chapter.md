# System Call Chapter

##### system call

* 交付給kernel執行，避免user直接操作
* 會發送trap\(soft interrupt\)，切換到kernel space
* 若有人問說為何要有user mode跟kernel mode，那最簡單的原因就是系統操作的安全。

##### system call request

* Process control - 控制process狀態，並給予process所需要的memory
* File management - 檔案的操作
* Device management - 對device進行操作
* Information maintenance - 維護系統資料，想是時間或是使用者
* Communications - 行程之間的溝通

補充  
在VM中critical instruction會讓VM處理inturrupt變複雜。Critical instruction主要指的是某些insturction的行為會因為user space或kernel space而不同。

