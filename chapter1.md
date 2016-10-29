# Nachos Chapter


#<p><a href="https://homes.cs.washington.edu/~tom/nachos/">Nachos</a></p>

Nachos是一個教學用的簡易OS，使用C++撰寫讓使用者可以輕易上手。Program會被compile成MIPS，然後再透過Nachos來模擬並執行instruction。
主要執行內容如下:
* 執行nachos後會運行Nachos kernel，然後交付給Machine simulation去模擬MIPS Machine，此時User program跟NachOS的system call一起compile(使用gcc generates MIPS code)，然後Machine simulation會去Fetch instruction(剛剛compile後的MIPS code)。
* 當執行過程中遇到system call時，會先Throw exception，然後交付給Kernel中的Exception handler去處理interrupt。


#以下會介紹Nachos Directory Structure
<p><a href="http://lms.nthu.edu.tw/course/26730">reference</a></p>

####lib/
Utilities used by the rest of the Nachoscode
####machine/
The machine simulation.
####threads/
Nachos is a multi-threaded program. Thread support is found here. This directory also contains the main() routine of the nachos program, inmain.cc.
####test/
User test programs to run on the simulated machine. As indicated earlier, these are separate from the source for the Nachos operating system and workstation simulation. This directory contains its own Makefile. The test programs are very simple and are written in C rather thanC++.
####userprog/
Nachos operating system code to support the creation of address spaces, loading of user (test) programs, and execution of test programs on the simulated machine. The exception handling code is here, inexception.cc.
####network/
Nachos operating system support for networking, which implements a simple "post office" facility. Several independent simulated Nachos machines can talk to each other through a simulated network. Unix sockets are used to simulate network connections among the machines.
####filesys/
Two different file system implementations are here. The "real" file system uses the simulated workstation's simulated disk to hold files. A "stub" file system translates Nachos file system calls into UNIX file system calls.makefile.

上面這些介紹是從課堂上的ppt擷取下來的，可以在coding時方便查找。