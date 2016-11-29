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