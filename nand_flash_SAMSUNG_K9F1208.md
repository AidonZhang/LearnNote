
裸板驱动及其时序详解：
		
		[http://blog.csdn.net/liuhongwei123888/article/details/8642852](http://blog.csdn.net/liuhongwei123888/article/details/8642852)

SAMSUNG_K9F5608X0D(512Bytes(没有包括16Bytes spare)*32pages*2048blocks=32Mbytes)需要3个地址周期
![](http://i.imgur.com/jlEN50h.jpg)

列地址：A[7:0]第一个地址周期发出，对于X8的nandflash来说，如果要把A[8]也编进列地址里面去，那么整个地址传输需要4个周期完成，所以为了节省一个周期，K9F5608X0D把页内分为A(1 half array)区，B(2 half array)区。A区0－255字节，B区256－511字节。访问某页时必须通过A[8]选定特定的区，A[8]则由操作指令决定的，00h，在A区；01h在B区。所以传输地址时A[8]不需要传输。对于X16的nand flash来说，由于一个page的main area的容量为256word，仍相当于512byte。但是，这个时候没有所谓的1st halfpage 和2nd halfpage之分了，所以，bit8就变得没有意义了，也就是这个时候 A8 完全不用管，地址传递仍然和x8器件相同。除了这一点之外，x16的NAND使用方法和 x8 的使用方法完全相同。

行地址：A[24:9]由第2，3个地址周期发出；

SAMSUNG_K9F1208(512Bytes(不包括16Bytes spare)*32pages*4096blocks=64MBytes)需要4个地址周期

![](http://i.imgur.com/YFkR86a.jpg)


列地址：A[7:0]第一个地址周期发出，同上；

行地址：A[25:9]由第2，3，4个地址周期发出，A[25]以上的各位必须被设置为L；

SAMSUNG_K9F1G08U0B(2048Bytes(不包括64Bytes spare)*64pages*1024blocks=128MBytes)需要4个地址周期

这里有个问题2048不是用11bits就可以表示了吗？为什么还要用到A[11:0]这12bits？？？？？此处求解

解:在后面写代码才发现,其实列地址就用到了A[10:0]才对的,也就是说下面datasheet出错了,

A[26:11]是用来表示页地址的,对于A[27]其实是不存在的,只在跟大容量时才会有的.

![](http://i.imgur.com/nus87xV.jpg)

列地址：A[10:0]第1，2个地址周期发出，如图*L位要置为low；

行地址：A[26:11]由第3，4个地址周期发出；

 

 ④页寄存器：由于对nand flash读取和编程操作，一般最小单位是page。所以nand在硬件设计时候，对每一片都有一个对应的区域用于存放将要写入到物理存储单元中区的或者刚从存储单元中读取出来的一页的数据，这个数据缓存区就是页缓存，也叫页寄存器。所以实际上写数据只是写到这个页缓存中，只有等你发了对应的编程第二阶段的确认命令0x10之后，实际的编程动作才开始把页缓存一点点的写到物理存储单元中去，这也就是为什么发完0x10之后需要等待一段时间的原因。
![](http://i.imgur.com/jv8XJ3C.jpg)

    接下来解释一下一般nand flash（SAMSUNG_K9F5608X0D）管脚的说明(后面＃或前面n后上面－都表示低电平有效)：

I/O0~I/O7:        用于输入命令/地址/数据，输出数据

CE＃  :  片选管脚

CLE        : command latch enable ，命令锁存使能，在输入命令之前必须使其有效

ALE  :    Address latch enable ，地址锁存使能，在输入地址前必须使其有效

RE＃    : read enable，读使能，在读数据之前要先使其有效

WR＃   : Write enable，写使能，在写数据之前要先使其有效

WP＃   ：write Protect，写保护

R/B#   ：Ready/Busy Output ，就绪/忙，主要用于在发送完编程/擦除后，检测这些操作是否完成。

Vcc     ：Power，电源

Vss     ：Ground，接地

N.C     ：Non-Connection，未定义，未连接
![](http://i.imgur.com/ttpIjcJ.jpg)

以下是关SAMSUNG_K9F5608X0D（注意：其它型号命令与时序图可能会有所不同，具体区别可以查阅相关芯片的datasheet）命令设置表：
![](http://i.imgur.com/PxS0Uns.jpg)
Read status：读取状态寄存器命令；
![](http://i.imgur.com/3oK3oCW.jpg)
![](http://i.imgur.com/Pksy4A0.jpg)
ReadID：读芯片ID命令；ECh是制造厂商代码，Device code 是设备号
![](http://i.imgur.com/EzHxSWE.jpg)
![](http://i.imgur.com/zyZN17q.jpg)
Read1    ：读data field命令，(00h->1stfield,01h->2nd field)从写入地址开始读，读到此页最后一个byte（包括了sparefield）；
![](http://i.imgur.com/W0UsSjZ.jpg)
以下是针对连续row read的时序图，连续row read不会被中止直到CE管脚被拉高（sequential row read 只对于K9F5608U0D_P,F orK9F5608B0D_P有效）:
![](http://i.imgur.com/J7l8LaM.jpg)
Read2：读sparefield命令；
![](http://i.imgur.com/rGDxViN.jpg)
Reset  ： 复位命令，当nand flash还在忙状态(随机读取，写数据或者擦除期间)的时候，如果接收到复位命令则会中止当前这些操作，并且这些正在被改变的存储单元里面的内容不再有效。之后命令寄存器会被清除用于等待下一个命令。
![](http://i.imgur.com/9enNOHe.jpg)

Page program：写命令，从写入地址开始写，写到此页最后一个byte(包括了sparefield)；（这里需要注意：发写命令之前要先写入页内区域选择命令；00h->A区(默认)，01h->B区）
![](http://i.imgur.com/NK0xDzB.jpg)
Copy-Back program:   从源地址某个区开始处把数据快速有效地拷贝到目标地址的同一个区域直到页的最后一个byte而不需要通过外部memory。
![](http://i.imgur.com/ggNe1h7.jpg)
Block Erase：块擦除，从A[9](因为页内只有512个byte故列地址占用A[8:0])以上的地址发出，但是A[13:9]将不被理睬，应该块地址是从A[14]以上位才有效的。
![](http://i.imgur.com/9Y1CUey.jpg)