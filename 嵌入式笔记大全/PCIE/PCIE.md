# PCI设备

## 1.主要信息

| 类别           | 信号                                                         | 描述 |
| -------------- | :----------------------------------------------------------- | ---- |
| 系统引脚       | CLK：给PCI设备提供时钟 RST#：用于复位PCI设备                 |      |
| 地址/数据引脚  | AD[31:00]：地址、数据复用 C/BE[3:0]：命令或者字节使能 PAR：校验引脚 |      |
| 接口控制       | FRAME#：PCI主设备驱动此信号，表示一个传输开始了、进行中 IRDY#：Initiator ready, 传输发起者就绪，一般由PCI主设备驱动此信号 TRDY#：Target ready，目标设备驱动，表示它就绪了 STOP#：目标设备驱动，表示它想停止当前传输 LOCK#：锁定总线，独占总线，有PCI桥驱动此信号 IDSEL：Initialization Device Select，配置设备时，用来选中某个PCI设备 DEVSEL#：Device Select，PCI设备驱动此信号，表示说：我就是你想访问的设备 |      |
| 仲裁引脚       | REQ#：申请使用PCI总线 GNT#：授予，表示你申请的PCI总线成功了，给你使用 |      |
| 错误通知引脚   | PERR#：奇偶校验错误 SERR#：系统错误                          |      |
| 中断引脚(可选) | INTA#、INTB#、INTC#、INTD#                                   |      |

![01_pci_sch](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\01_pci_sch.png)

## 2.PCI相关信息总结

### 2.1 PCI如何选择设备

​	像上面原理图的AD总线，是既可以传输数据，又可以传输地址的。不同PCI设备的**IDSEL**引脚会挂载到**AD31-AD11**这20个引脚上（具体根据设备数量而定，可能是桥设备，也有可能是PCI设备）。

​	如何判断是桥设备还是PCI设备，根据下图配置空间的header type来确定，**若为桥设备则为01，若为PCI设备则为00**

![11_headtype](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\11_headtype.png)

### 2.2 如何选中PCI设备或者桥设备

跟Root Bridge直接相连的PCI Agent、PCI Bridge，使用`Configuration Command type 0`

![image-20240112205122361](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112205122361.png)

桥之后的设备，Root Brdige使用`Configuration Command type 1`

![image-20240112205613949](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112205613949.png)

桥：接收到`Configuration Command type 1`后，有2个选择

* **对于跟它直连的PCI Agent**
  * 把`Configuration Command type 1`转换为`Configuration Command type 0`
  * 使用IDSEL选择设备
* **如果还需要通过下一级的桥才能访问此设备**
  * 转发`Configuration Command type 1`

### 2.3 PCI（Agent）设备的配置空间

![image-20240112205345262](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112205345262.png)

### 2.4 PCI桥设备的配置空间

![image-20240112205413056](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112205413056.png)

### 2.5 如何判断PCI发出的是读还是写信号

根据**C/BE[3:0]**的信号可以看出来

![1415a1ea43f9abd12879d43eec9b751f](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\1415a1ea43f9abd12879d43eec9b751f.png)

## 3.读写PCI设备主要流程

### 3.1 配置读写直连总线的PCI设备或者桥设备

1. CPU先进行配置读操作，以直连**Root Bridge**的PCI设备和桥为例，CPU发出**addr_cpu**信号经**Root Bridge**转化为addr_pci信号，也就是32位的**type0**信号，信号里面第**10~2数据位**有功能、寄存器的信息，而第**31~11位数据位**则包含了要选择的设备信息，相应设备的**IDSEL引脚**会挂载到某个AD总线位上，从而被选中，因此**type0信号不需要包含设备ID**。
2. CPU进行完配置读操作以后，PCI设备返回所需要的内存空间，CPU会进行内存空间的分配，并将分配完的地址写给相应的PCI设备，以后访问PCI设备或者桥直接访问相应的地址即可。

### 3.2配置读写总线下的桥设备以及PCI设备

![image-20240112214901980](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112214901980.png)

Type1上的总线、设备、功能、寄存器等信息会逐级转发，直到到达指定总线的范围下，经过桥设备转化为Type0信号，Type0信号就包含了要控制哪个IDSEL等信息，从而找到相应设备进行配置。

# PCIE设备

## 1.主要信息

**之所以用串行差分信号来传输数据，主要是因为PCI的并行虽然效率高，但是相互之间的干扰极强，不利于传输准确性**

PCI接口的引脚时并行的，PCEe的是串行的，每个方向的数据使用2条差分信号线来传输，发送/接收两个方向就需要4条线，这被称为1个Lane：

![image-20240112211419240](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112211419240.png)

PCIe设备的接口上，可以有多个Lane：

* 两个PCIe设备之间有一个Link
* 一个Link中有1对或多对"发送/接收"引脚，每对"发送/接收"引脚被称为"Lane"
* 一个Lane：有发送、接收两个方向，每个方向用2条差分信号线，所以1个Lane有4条线
* 一个Link最多可以有32 Lane

![image-20240112211457584](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112211457584.png)

## 2.PCIE的TLP包主要结构

1. 事务层(Tansaction Layer)：传输的是Transaction Layer Packet(TLP)

![image-20240112211714406](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112211714406.png)

2. 数据链路层(Data Link Layer)：传输的是Data Link Layer Packet(DLLP)

   在TLP加上前缀、后缀，得到DLLP

![image-20240112211806094](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112211806094.png)

3. 物理层(Physical Layer)：传输的是Physical Packet

   在DLLP加上前缀、后缀，得到的是Physical Packet

![image-20240112211833405](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112211833405.png)

![image-20240112211859625](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112211859625.png)



在PCI系统里，多个PCI设备都链接到相同总线上。

**在PCIe系统里，是点对点传输的：**

* **一条PCIe总线只能接一个PCIe设备**
* **要接多个PCIe设备，必须使用Switch进行扩展**

## 3.PCIE的配置相关信息

### 3.1如何判断PCIE是配置读配置写，还是IO读IO写，还是内存读内存写

​	根据TLP头部信息即可判断，如下图所示：

CfgRd0和CfgWr0用于Type0的配置读写，主要是配置读写直接连接的桥设备或者PCIE设备

CfgRd1和CfgWr1用于Type1的配置读写，主要是配置读写桥下的桥设备或者PCIE设备

![image-20240112212205463](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112212205463.png)

### 3.2 如何判断要读写的是PCIE设备还是桥设备

PCIE设备与桥设备配置寄存器的区别与PCI章节的基本一致，如下图所示：

![image-20240112212411672](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112212411672.png)

同时还能根据Header Type来判断是不是多功能设备：

![image-20240112212505396](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112212505396.png)

## 4.读写PCIE设备主要流程

### 4.1 配置在当前总线上的桥或者PCIE设备（type零）

由于无法像PCI设备一样通过IDSEL选中具体的设备，因此TLP包内必须含有**总线、设备、功能、寄存器**这四样数据。

同样，因为PCIE设备是点对点的，一条总线只有一个PCIE设备，因此设备号都是出厂的时候固定死了的，控制哪个设备就得写清楚设备号了。

我们在控制一个**PCIE**设备之前肯定是要先对他进行配置读的。CPU发出先发出**addr_cpu**经**Root** **Complex**转化为**addr_pcie**信号（**TLP**），然后**PCIE**控制器发送**TLP**数据包进行配置读。当CPU分配完地址空间后，对相应设备进行配置写。以后就可以直接通过读写分配好的地址空间直接对PCIE设备或者桥设备进行控制了。

### 4.2 配置的设备，不在当前总线上，但是在它下面的总线上，即(Type壹)：

* 目标设备的Bus号大于当前桥的Secondary Bus Number，
* 目标设备的Bus号小于或等于当前桥的Subordinate Bus Number，

那么在当前总线(即Secondary Bus Number)上传输的就是"Type 1 Configuration Request"：

* TLP格式如下图所示
* 会穿过桥
* 到达设备时，跟设备直接连接的桥会把它转换为"Type 0 Configuration Request"

![image-20240112214704309](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112214704309.png)

## 5.PCIE的总线配置流程（其实就是读写配置桥罢了，也算重点）

![image-20240112215220061](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112215220061.png)

**配置过程演示：**

下文中BDF表示Bus,Device,Function，用这三个数值来表示设备。

1. 软件设置Host/PCI Bridge的Secondary Bus Number为0，Subordinate Bus Number为255(先设置为最大，后面再改)。
2. 从Bus 0开始扫描：先尝试读到BDF(0,0,0)设备的Vendor ID，如果不成功表示没有这个设备，就尝试下一个设备BDF(0,1,0)。一个桥下最多可以直接连接32个设备，所以会尝试32次：Device号从0到31。**注意**：在Host/PCI Bridge中，这些设备的Device号是硬件写死的。
3. 步骤2读取BDF(0,0,0)设备(即使图中的A)时，发现它的Header Type是01h，表示它是一个桥、单功能设备
4. 发现了设备A是一个桥，配置它：
   * Primary Bus Number Register = 0：它的上游总线是Bus 0
   * Secondary Bus Number Register = 1：从它发出的总线是Bus 1
   * Subordinate Bus Number Register = 255：先设置为最大，后面再改
5. 因为发现了桥A，执行"深度优先"的配置过程：先去枚举A下面的设备，再回来枚举跟A同级的B
6. 软件读取BDF(1,0,0)设备(就是设备C)的Vendor ID，成功得到Vendor ID，表示这个设备存在。
7. 它的Header Type是01h，表示这是一个桥、单功能设备。
8. 配置桥C：
   * Primary Bus Number Register = 1：它的上游总线是Bus 1
   * Secondary Bus Number Register = 2：从它发出的总线是Bus 2
   * Subordinate Bus Number Register = 255：先设置为最大，后面再改
9. 继续从桥C执行"深度优先"的配置过程，枚举Bus 2下的设备，从BDF(2,0,0)开始
10. 读取BDF(2,0,0)设备(就是设备D)的Vendor ID，成功得到Vendor ID，表示这个设备存在。
11. 它的Header Type是01h，表示这是一个桥、单功能设备。
12. 配置桥D：
    * Primary Bus Number Register = 2：它的上游总线是Bus 2
    * Secondary Bus Number Register = 3：从它发出的总线是Bus 3
    * Subordinate Bus Number Register = 255：先设置为最大，后面再改
13. 继续从桥D执行"深度优先"的配置过程，枚举Bus 2下的设备，从BDF(3,0,0)开始
14. 读取BDF(3,0,0)设备的Vendor ID，成功得到Vendor ID，表示这个设备存在。
15. 它的Header Type是80h，表示这是一个Endpoing、多功能设备。
16. 软件枚举这个设备的所有8个功能，发现它有Function0、1
17. 软件继续枚举Bus 3上其他设备(Device号1~31)，没发现更多设备
18. 现在已经扫描完桥D即Bus 3下的所有设备，它下面没有桥，所以桥D的Subordinate Bus Number等于3。扫描完Bus 3后，回退到上一级Bus 2，继续扫描其他设备，从BDF(2,1,0)开始，就是开始扫描设备E。
19. 读取BDF(2,1,0)设备(就是设备E)的Vendor ID，成功得到Vendor ID，表示这个设备存在。
20. 它的Header Type是01h，表示这是一个桥、单功能设备。
21. 配置桥E：
    * Primary Bus Number Register = 2：它的上游总线是Bus 2
    * Secondary Bus Number Register = 4：从它发出的总线是Bus 4
    * Subordinate Bus Number Register = 255：先设置为最大，后面再改
22. 继续从桥D执行"深度优先"的配置过程，枚举Bus 4下的设备，从BDF(4,0,0)开始
23. 读取BDF(4,0,0)设备的Vendor ID，成功得到Vendor ID，表示这个设备存在。
24. 它的Header Type是00h，表示这是一个Endpoing、单功能设备。
25. 软件继续枚举Bus 4上其他设备(Device号1~31)，没发现更多设备
26. 已经枚举完设备E即Bus 4下的所有设备了，更新设备E的Subordinate Bus Number为4。然后继续扫描设备E的同级设备：Bus=2，Device从2到31，发现Bus 2上没有这些设备。
27. 软件更新设备C即Bus 2的桥，把它的Subordinate Bus Number设置为4。然后继续扫描设备C的同级设备：Bus=1，Device从1到31，发现Bus 1上没有这些设备。
28. 软件更新设备A即Bus 1的桥，把它的Subordinate Bus Number设置为4。然后继续扫描设备A的同级设备：Bus=0，Device从1到31，发现Bus 0上的设备B。
29. 配置桥B：
    * Primary Bus Number Register = 0：它的上游总线是Bus 0
    * Secondary Bus Number Register = 5：从它发出的总线是Bus 5
    * Subordinate Bus Number Register = 255：先设置为最大，后面再改
30. 再从桥B开始，执行"深度优先"的配置过程。

## 6.PCIE的路由方式

所谓"路由"，就是怎么找到对方，PCIe协议中有三种路由方式：

* 基于ID的路由
* 基于地址的路由
* 隐式路由

TLP中怎么表示自己使用哪种路由？TLP头部就表明了：

![image-20240112223250233](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112223250233.png)

**消息报文中头部的Type字段里低3位表示隐式路由方式：**

* 000：路由到RC
* 001：使用地址路由(使用地址路由的消息不常见)
* 010：使用ID路由
* 011：来自RC的广播报文，底下的PCIe桥会向下游转发此消息
* 100：本地消息，消息到达目的设备后结束，不会再次转发
* 101：路由到RC，仅用于电源管理



**访问PCIe设备时，要先配置，才能读写数据：**

* 配置读、配置写：使用**基于ID的路由**，就是使用<Bus, Device, Function>来寻找对方。配置成功后，每个PCIe设备都有自己的PCIe地址空间了。
* 内存读、内存写或者IO读、IO写：
  * 发出报文给对方：使用**基于地址的路由**
  * 对方返回数据或者返回状态时：使用**基于ID的路由**
* 各类消息，比如中断、广播等：使用**隐式路由**

### 6.1基于ID的路由

TLP中含有<Bus number, Device number, Function number>，这就是ID。

配置成功后，每个PCIe设备，包括虚拟的PCIe桥，都分配到了地址空间：

* PCIe设备(Endpoint)：地址空间是自己的，别的设备访问这个地址范围时，就是想访问它
* PCIe桥：地址空间是下面的PCIe设备的，别的设备访问这个地址范围时，就是想访问它下面的设备

### 6.2 PCIe设备(Endpoint)的配置空间.

Endpoint的配置空间里有"Base Address Regiseters"：

* 一开始：这些寄存器用来声明：需要多大的地址空间、是内存空间还是IO空间
* 被配置后：系统软件为它分配出地址空间，把首地址写入这些寄存器

下图来自《PCI Express_ Base Specification Revision 4.0 Version 0.3 ( PDFDrive ).pdf》：

![image-20240112224206306](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112224206306.png)

### 6.3 PCIe桥的配置空间

PCIe桥的配置空间里有"Primary Bus Number、Secondary Bus Number、Subordinate Bus Number"：

* 配置PCIe桥的时候，系统软件会填充这些信息
* 对于ID路由，将根据这些信息转发报文

PCIe桥的配置空间里有"Memory Base、Prefetchable Memory Base、I/O Base"：

* 表示该桥下游所有设备使用的三组空间范围
  * 存储器空间
  * 可预取的存储器空间
  * I/O空间
* 对于地址路由，将根据这些信息转发报文
* **最重要的一点，比如桥A下面是桥B和桥C，B的Memory Base为1-1000，C的Memory Base为1000-2000，那么A的存储空间大小最起码要为1-2000，这样在type1信号传递的过程中就可以判断是不是在自己的地址范围下，如果是则往下传递，如果不是则不作出反应**

下图来自《PCI Express_ Base Specification Revision 4.0 Version 0.3 ( PDFDrive ).pdf》：

<img src="E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112224251369.png" alt="image-20240112224251369" style="zoom:80%;" />

### 6.3基于地址的路由

下图中 P1、P2、P3都是桥设备，判断TLP包里面的地址信息是不是自己处理的范围，如果是就转发，不是就不转发。（**小弟能处理就转发**）

<img src="E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\957b330012e6dc7c918cd77954b908a9.png" alt="957b330012e6dc7c918cd77954b908a9" style="zoom: 67%;" />

下图中当终端PCIE设备要发出TLP包给CPU时，P2判断他处理不了于是向上级转发TLP包（小弟处理不了就上报）

<img src="E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\e8754e8797f9644dafe3feb9a04ece90.png" alt="e8754e8797f9644dafe3feb9a04ece90" style="zoom: 80%;" />

**当我们要发起I/O读写的时候，TLP报文如下图所示：**
<img src="E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112230108645.png" alt="image-20240112230108645" style="zoom:80%;" />

**报文发送完成，PCIE发出回应报文如下图所示：**

<img src="E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112230300918.png" alt="image-20240112230300918" style="zoom:80%;" />

主设备要给EndPoint的内存写数据，它发出"内存写报文"，不需要对方回应。

主设备要读EndPoint的内存数据，它发出"内存读报文"，需要对方回应。

主设备要给EndPoint的IO写数据，它发出"IO写报文"，需要对方回应。

主设备要读EndPoint的IO数据，它发出"IO读报文"，需要对方回应。

* PCIe设备要回应时，回应谁？给"Requester ID"，使用**基于ID的路由**
* 发起PCIe传输的设备(主设备)，对每次传输都分配一个独一的Tag，并且在硬件内部保存当前TLP
* 接收到回应报文后，才会根据Tag清除掉内存中保存数据
* 如果没接收到回应，或者失败了：会把硬件中保存的TLP重新发送出去

回应的完成报文，可以含有数据，也可以不含数据，格式如下：

<img src="E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240112230341175.png" alt="image-20240112230341175" style="zoom:80%;" />

主设备进行IO读写的时候TLP包里面就已经有了上图红框中的内容了。PCIE设备接收到之后就可以原路返回发消息了。

## 7. PCIE设备的AXI总线

实际芯片中，CPU与外设之间的连接更加复杂，高速设备之间通过AXI总线连接。AXI总线总传输数据的双方分为Master和Slave，Master发起传输，Slave回应传输。Master和Slave是多对多的关系，它们之间读、写可以同时进行的，内部结构图如下：

其中 AWADDR表示写地址、ARADDR表示读地址，二者都是64位。

![image-20240114104533947](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114104533947.png)

**在AXI总线中，读写可以同时进行，有5个通道：**

* 读：
  * **读地址通道：传输读操作的地址**
  * **读数据通道：传输读到的数据**
* 写：
  * **写地址通道：传输写操作的地址**
  * **写数据通道：传输要写的数据**
  * **写响应通道：传输写操作的结果**

![image-20240114104929749](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114104929749.png)

![image-20240114104935517](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114104935517.png)

| 通道名称       | 通道功能                               | 数据流向   |
| -------------- | -------------------------------------- | ---------- |
| read address   | 读地址通道                             | 主机->从机 |
| read data      | 读数据通道（包括数据通道和读响应通道） | 从机->主机 |
| write address  | 写地址通道                             | 主机->从机 |
| write data     | 写数据通道                             | 主机->从机 |
| write response | 写响应通道                             | 从机->主机 |



## 8.CPU地址如何转化为PCIE的地址，发送地址和数据即可完成对设备的配置或读写

### 8.1 CPU访问PCIe控制器时，CPU地址空间可以分为：

* Client Register Set：地址范围 0xFD000000~0xFD7FFFFF，比如选择PCIe协议的版本(Gen1/Gen2)、电源控制等
* Core Register Set  ：地址范围 0xFD800000~0xFDFFFFFF，所谓核心寄存器就是用来进行设置地址映射的寄存器等
* Region 0：0xF8000000~0xF9FFFFFF , 32MB，用于访问外接的PCIe设备的配置空间
* Region 1：0xFA000000~0xFA0FFFFF，1MB，用于地址转换
* Region 2：0xFA100000~0xFA1FFFFF，1MB，用于地址转换
* ……
* Region 32：0xFBF00000~0xFBFFFFFF，1MB，用于地址转换

其中Region 0大小为32MB，Region1~31大小分别为1MB。



CPU访问Region 0的地址时，将会导致PCIe控制器发出读写配置空间的TLP。

CPU访问Region 1~32的地址时，将会导致PCIe控制器发出读写内存、IO空间的TLP。

### 8.2 寄存器介绍

 CPU访问一个地址，导致PCIe控制器发出TLP。TLP里含有PCIe地址、其他信息。

这些寄存器必定涉及这2部分：

* 地址转换：把CPU地址转换为PCIe地址
* 提供TLP的其他信息

Region0、Region1~32，每个Region都有类似的寄存器。

一个Region，可以用于读写配置空间，可以用于读写内存空间、可以用于读写IO空间，还可以用于读写消息。

这由Region对应的寄存器决定。

每个Region都有一样寄存器，以Region 0为例，有6个寄存器：

![image-20240114110421175](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114110421175.png)

**到底是发出哪种TLP，由Region对应的ob_desc0寄存器决定：**

| ob_desc0[3:0] | 作用                                |
| ------------- | ----------------------------------- |
| 1010          | 发出的TLP用于访问Type 0的配置空间   |
| 1011          | 发出的TLP用于访问Type 1的配置空间   |
| 0010          | 发出的TLP用于读写内存空间           |
| 0110          | 发出的TLP用于读写IO空间             |
| 1100          | 发出的TLP是"Normal Message"         |
| 1101          | 发出的TLP是"Vendor-Defined Message" |

CPU访问某个Region时，最终都是要发出TLP，TLP的内容怎么确定？

* 地址信息：ob_addr0/1把CPU地址转换为PCIe地址，提供TLP里面的地址信息

* 其他信息：ob_desc0/1/2/3提供TLP的其他信息

### 8.3如何读写配置空间

**Region0一般用于读写配置空间**，它对应的寄存器如下：

![image-20240114111418532](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114111418532.png)

#### 8.3.1定位：Bus/Dev/Function/Reg

![image-20240114112946892](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114112946892.png)

当Region 0的寄存器**ob_desc0[3:0]**被配置为读写配置空间时， CPU发出Region 0的地址，地址里面隐含有这些信息：

* Bus：cpu_addr[27:20]
* Dev：cpu_addr[19:15]
* Fun：cpu_addr[14:12]
* Reg：cpu_addr[11:0]

配置ob_desc[3:0]如下图所示：

![image-20240114111907771](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114111907771.png)

#### 8.3.2接下来就是地址转换了

比如我们可以设置bit[5:0]为27，意味着cpu_addr[27:0]这28条地址线都会传入TLP。

![image-20240114112529784](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114112529784.png)

#### 8.3.3 CPU读写Region 0的地址

Region 0的地址范围是：0xF8000000~0xF9FFFFFF。

CPU想访问这个设备：Bus=bus,Dev=dev,Fun=fun,Reg=reg，那么CPU读写这个地址即可：

```shell
0xF8000000 + (bus<<20) | (dev<<15) | (fun<<12) | (reg)
```

### 8.4如何读写IO或者内存空间

![63_me_io_rw_example](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\63_me_io_rw_example.png)

#### 8.4.1 如何配置Region 1用于内存或者IO读写

![image-20240114113105177](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114113105177.png)

#### 8.4.2如何配置Region 1地址转换

![image-20240114113137929](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114113137929.png)

addr0、addr1寄存器里保存的是PCIe地址，也就是CPU发出这个Region的CPU地址后，将会转换为某个PCI地址。

怎么转换？由addr0、addr1决定。

Region 1的CPU地址范围是：0xFA000000~0xFA0FFFFF，是1M空间。

我们一般会让PCI地址等于CPU地址，所以这样设置：

* addr0：
  * [5:0]等于19，表示CPU_ADDR[19:0]共20位地址传入TLP
  * [31:8]等于0xFA0000
* addr1：设置为0



如上设置后，CPU读写地址时0xFA0?????，就会转换为PCI地址：0xFA0?????，转换过程如下：

```shell
pci_addr = cpu_addr[19:0] | (addr0[31:20] << 20) | (addr1<<32)
         = 0x????? + (0xFA0 << 20) | (0 << 32)
         = 0xFA0?????
```

### 总结

CPU是要读写配置空间还是内存、IO空间需要读写不同的Region。

读写配置空间主要涉及到Region 0的配置，内存IO空间则涉及到Region 1~31的配置。

Region 1~31主要涉及到PCIE设备的内存及IO读写，Region 0则涉及到PCIE设备或桥的配置读写。

读写配置空间时Region 0的ob_desc0[3:0]主要用于判断是读写PCIE设备还是桥设备，其余的提供其他信息。ob_addr0和ob_addr1则用于将CPU的地址信息转换为TLP中的地址信息，主要为ob_addr0[5:0]的配置，一般是全部由CPU地址来填充。

读写内存或者IO空间时主要是配置Region 1~31的寄存器ob_desc0[3:0]，ob_desc0和ob_desc1的其余位则提供其他信息。ob_addr0和ob_addr1则用于将CPU的地址信息转换为TLP中的地址信息，主要为ob_addr0[5:0]的配置，我们一般会让PCI地址等于CPU地址所以这样设置：

* addr0：
  * [5:0]等于19，表示CPU_ADDR[19:0]共20位地址传入TLP
  * [31:8]等于0xFA0000
* addr1：设置为0

如上设置后，CPU读写地址时0xFA0?????，就会转换为PCI地址：0xFA0?????，转换过程如下：

```shell
pci_addr = cpu_addr[19:0] | (addr0[31:20] << 20) | (addr1<<32)
         = 0x????? + (0xFA0 << 20) | (0 << 32)
         = 0xFA0?????
```

# PCIE_Host驱动分析_地址映射（以RK3399为例）

其中 root_bus对应真实硬件，下面会扫描出PCI&PCIE设备或者PCI&PCIE桥，然后会经过右侧虚拟总线进行与驱动的匹配。

![51_pci_driver_block](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\51_pci_driver_block-1705200124686-1.png)

## 1.所谓`Host`，就是PCIe控制器，它的驱动做什么？

* 解析设备树，根据设备树确定：寄存器地址、CPU空间地址、PCI空间地址、中断信息
* 记录资源：CPU空间地址、PCI空间地址
* 初始化PCIe控制器本身，建立CPU地址和PCI地址的映射
* 扫描识别当前PCIe控制器下面的PCIe设备

## 2.设备树文件解析及CPU地址空间

RK3399访问PCIe控制器时，CPU地址空间可以分为：

* Client Register Set：地址范围 0xFD000000~0xFD7FFFFF，比如选择PCIe协议的版本(Gen1/Gen2)、电源控制等
* Core Register Set  ：地址范围 0xFD800000~0xFDFFFFFF，所谓核心寄存器就是用来进行设置地址映射的寄存器等
* Region 0：0xF8000000~0xF9FFFFFF , 32MB，用于访问外接的PCIe设备的配置空间
* Region 1：0xFA000000~0xFA0FFFFF，1MB，用于地址转换
* Region 2：0xFA100000~0xFA1FFFFF，1MB，用于地址转换
* ……
* Region 32：0xFBF00000~0xFBFFFFFF，1MB，用于地址转换

其中Region 0大小为32MB，Region1~31大小分别为1MB。

在设备树里都有体现(下列代码中，其他信息省略了)：

* reg属性里的0xf8000000：Region 0的地址
* reg属性里的0xfd000000：PCIe控制器内部寄存器的地址
  * Client Register Set：地址范围 0xFD000000~0xFD7FFFFF
  * Core Register Set  ：地址范围 0xFD800000~0xFDFFFFFF
* ranges属性里
  * **第1个0xfa000000：Region1~30的CPU地址空间首地址，用于内存读写**
  * **第2个0xfa000000：Region1~30的PCI地址空间首地址，用于内存读写**
  * **第1个0xfbe00000：Region31的CPU地址空间首地址，用于IO读写**
  * **第2个0xfbe00000：Region31的PCI地址空间首地址，用于IO读写**
* **Region32呢？在.c文件里用作"消息TLP"**

```shell
        pcie0: pcie@f8000000 {
                compatible = "rockchip,rk3399-pcie";
                #address-cells = <3>;
                #size-cells = <2>;
                ranges = <0x83000000 0x0 0xfa000000 0x0 0xfa000000 0x0 0x1e00000
                          0x81000000 0x0 0xfbe00000 0x0 0xfbe00000 0x0 0x100000>;
                reg = <0x0 0xf8000000 0x0 0x2000000>,
                      <0x0 0xfd000000 0x0 0x1000000>;
                reg-names = "axi-base", "apb-base";
        };
```

## 3.Region0和寄存器地址

下图中的rockchip_pcie_parse_dt函数在probe函数内被调用，主要用于解析设备树的信息。

下图中的axi_base其实就是涉及到前文中提到CPU发出的AXI总线信号给PCIE控制器设备，apb_base则为PCIE控制器的寄存器地址（APB代表外设的意思）

![image-20240114165119979](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114165119979.png)

0xF8000000就是RK3399的Region0地址，用于 ECAM：[PCIe ECAM介绍](https://zhuanlan.zhihu.com/p/176988002)。

即：只写读写0xF8000000这段空间，就可以只写读写PCIe设备的配置空间。

0xFD000000即使RK3399 PCIe控制器本身的寄存器基地址。

Region0用与读写配置空间，它对应的寄存器要设置用于产生对应的TLP，函数调用关系如下：

```c
rockchip_pcie_probe
    err = rockchip_pcie_init_port(rockchip);
```

在读写PCIE设备之前，需要配置好Region0的寄存器，如下图所示：

![image-20240114165346239](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114165346239.png)

## 4.确定CPU/PCI地址空间

在内核`arch/arm64/boot/dts`下搜：`grep "rockchip,rk3399-pcie" * -nr`

​	找到设备树文件：`arch/arm64/boot/dts/rk3399.dtsi`，代码如下：

```shell
        pcie0: pcie@f8000000 {
                compatible = "rockchip,rk3399-pcie";
                #address-cells = <3>;
                #size-cells = <2>;
                aspm-no-l0s;
                clocks = <&cru ACLK_PCIE>, <&cru ACLK_PERF_PCIE>,
                         <&cru PCLK_PCIE>, <&cru SCLK_PCIE_PM>;
                clock-names = "aclk", "aclk-perf",
                              "hclk", "pm";
                bus-range = <0x0 0x1f>;
                max-link-speed = <1>;
                linux,pci-domain = <0>;
                msi-map = <0x0 &its 0x0 0x1000>;
                interrupts = <GIC_SPI 49 IRQ_TYPE_LEVEL_HIGH 0>,
                             <GIC_SPI 50 IRQ_TYPE_LEVEL_HIGH 0>,
                             <GIC_SPI 51 IRQ_TYPE_LEVEL_HIGH 0>;
                interrupt-names = "sys", "legacy", "client";
                #interrupt-cells = <1>;
                interrupt-map-mask = <0 0 0 7>;
                interrupt-map = <0 0 0 1 &pcie0_intc 0>,
                                <0 0 0 2 &pcie0_intc 1>,
                                <0 0 0 3 &pcie0_intc 2>,
                                <0 0 0 4 &pcie0_intc 3>;
                phys = <&pcie_phy>;
                phy-names = "pcie-phy";
                ranges = <0x83000000 0x0 0xfa000000 0x0 0xfa000000 0x0 0x1e00000
                          0x81000000 0x0 0xfbe00000 0x0 0xfbe00000 0x0 0x100000>;
                reg = <0x0 0xf8000000 0x0 0x2000000>,
                      <0x0 0xfd000000 0x0 0x1000000>;
                reg-names = "axi-base", "apb-base";
                resets = <&cru SRST_PCIE_CORE>, <&cru SRST_PCIE_MGMT>,
                         <&cru SRST_PCIE_MGMT_STICKY>, <&cru SRST_PCIE_PIPE>,
                         <&cru SRST_PCIE_PM>, <&cru SRST_P_PCIE>,
                         <&cru SRST_A_PCIE>;
                reset-names = "core", "mgmt", "mgmt-sticky", "pipe",
                              "pm", "pclk", "aclk";
                status = "disabled";
                pcie0_intc: interrupt-controller {
                        interrupt-controller;
                        #address-cells = <0>;
                        #interrupt-cells = <1>;
                };
        };
```

在PCIe设备树里有一个属性`ranges`，它里面含有多个range，每个range描述了：

* flags：是内存还是IO
* PCIe地址
* CPU地址
* 长度（也就是大小，多少M）

先提前说一下怎么解析这些range，函数为`for_each_of_pci_range`，解析过程如下：

![image-20240114171541785](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\image-20240114171541785.png)

从probe函数开始分析，完整的代码流程及主要解析信息如下图：

```c
rockchip_pcie_probe
	resource_size_t io_base;
    LIST_HEAD(res); // 资源链表

	// 解析设备树获得PCI host bridge的资源(CPU地址空间、PCI地址空间、大小)
	err = of_pci_get_host_bridge_resources(dev->of_node, 0, 0xff, &res, &io_base);
		// 解析 bus-range
		// 设备树里:  bus-range = <0x0 0x1f>;
		// 解析得到: bus_range->start= 0 , 
		//          bus_range->end = 0x1f, 
		//          bus_range->flags = IORESOURCE_BUS;
		// 放入前面的链表"LIST_HEAD(res)"
		err = of_pci_parse_bus_range(dev, bus_range);  
			pci_add_resource(resources, bus_range);

		// 解析 ranges
		// 设备树里: 
        //        ranges = <0x83000000 0x0 0xfa000000 0x0 0xfa000000 0x0 0x1e00000
        //                  0x81000000 0x0 0xfbe00000 0x0 0xfbe00000 0x0 0x100000>;
    	of_pci_range_parser_init
    		parser->range = of_get_property(node, "ranges", &rlen);
		for_each_of_pci_range(&parser, &range) {// 解析range            
            // 把range转换为resource
            // 第0个range
            // 		range->pci_space = 0x83000000,
            //		range->flags     = IORESOURCE_MEM,
            //		range->pci_addr  = 0xfa000000,
            //		range->cpu_addr  = 0xfa000000,
            //		range->size      = 0x1e00000,
            // 转换得到第0个res：
            // 		res->flags = range->flags = IORESOURCE_MEM;
            // 		res->start = range->cpu_addr = 0xfa000000;
            // 		res->end = res->start + range->size - 1 = (0xfa000000+0x1e00000-1);
            // ---------------------------------------------------------------
            // 第1个range
            // 		range->pci_space = 0x81000000,
            //		range->flags     = IORESOURCE_IO,
            //		range->pci_addr  = 0xfbe00000,
            //		range->cpu_addr  = 0xfbe00000,
            //		range->size      = 0x100000,
            // 转换得到第1个res：
            // 		res->flags = range->flags = IORESOURCE_MEM;
            // 		res->start = range->cpu_addr = 0xfbe00000;
            // 		res->end = res->start + range->size - 1 = (0xfbe00000+0x100000-1);
            err = of_pci_range_to_resource(&range, dev, res); 

            // 在链表中增加resource
            // 第0个resource：
            //		注意第3个参数: offset = cpu_addr - pci_addr = 0xfa000000 - 0xfa000000 = 0
            // 第1个resouce
            //		注意第3个参数: offset = cpu_addr - pci_addr = 0xfbe00000 - 0xfbe00000 = 0
            pci_add_resource_offset(resources, res,	res->start - range.pci_addr);

        }

    /* Get the I/O and memory ranges from DT */
    resource_list_for_each_entry(win, &res) {
        rockchip->io_bus_addr = io->start - win->offset;   // 0xfbe00000, cpu addr
        rockchip->mem_bus_addr = mem->start - win->offset; // 0xfba00000, cpu addr
        rockchip->root_bus_nr = win->res->start; // 0
    }

	bus = pci_scan_root_bus(&pdev->dev, 0, &rockchip_pcie_ops, rockchip, &res);

	pci_bus_add_devices(bus);
```

## 5.建立CPU/PCI地址空间的映射

调用关系如下（rockchip_cfg_atu中的atu为adress transition unit，地址转换单元）：

```c
rockchip_pcie_probe
   	err = rockchip_cfg_atu(rockchip);
				/* MEM映射: Region1~30 */ 
                for (reg_no = 0; reg_no < (rockchip->mem_size >> 20)/*0x1e00000右移二十位即为0x1e，也就是30个Region*/; reg_no++) {
                    err = rockchip_pcie_prog_ob_atu(rockchip, reg_no + 1,
                                    AXI_WRAPPER_MEM_WRITE,
                                    20 - 1,
                                    rockchip->mem_bus_addr +
                                    (reg_no << 20),
                                    0);
                    if (err) {
                        dev_err(dev, "program RC mem outbound ATU failed\n");
                        return err;
                    }
                }
                
				/* IO映射: Region31 */
                offset = rockchip->mem_size >> 20;
                for (reg_no = 0; reg_no < (rockchip->io_size >> 20)/*0x1e00000右移二十位即为0x1e，也就是30个Region*/; reg_no++) {
                    err = rockchip_pcie_prog_ob_atu(rockchip,
                                    reg_no + 1 + offset,
                                    AXI_WRAPPER_IO_WRITE,
                                    20 - 1,
                                    rockchip->io_bus_addr +
                                    (reg_no << 20),
                                    0);
                    if (err) {
                        dev_err(dev, "program RC io outbound ATU failed\n");
                        return err;
                    }
                }

                /* 用于消息传输: Region32 */
                rockchip_pcie_prog_ob_atu(rockchip, reg_no + 1 + offset,
                              AXI_WRAPPER_NOR_MSG,
                              20 - 1, 0, 0);

                rockchip->msg_bus_addr = rockchip->mem_bus_addr +
                                ((reg_no + offset) << 20);

```

**那究竟是如何建立映射的呢？**

以下面MEM空间映射为例，之后以此类推，之所以CPU地址和PCIE地址设置成相同的，主要是为了方便控制。

MEM空间映射：

reg_no为0，即开始Region1的内存映射。Region1的CPU地址为0xFA000000，mem_bus_addr为0xFA000000，mem_bus_addr加上reg_no左移20位还是0xFA000000。Region1的PCIE地址为0xFA000000。

reg_no为1，即开始Region2的内存映射。Region2的CPU地址为0xFA100000，mem_bus_addr为0xFA000000，mem_bus_addr加上reg_no左移20位还是0xFA100000。Region2的PCIE地址为0xFA100000。

```c
	// rockchip->mem_bus_addr = 0xfa000000
	// rockchip->mem_size     = 0x1e00000
	// 设置Region1、2、……30的映射关系
	for (reg_no = 0; reg_no < (rockchip->mem_size >> 20)/*0x1e00000右移二十位即为0x1e，也就是30个Region*/; reg_no++) {
		err = rockchip_pcie_prog_ob_atu(rockchip, reg_no + 1,
						AXI_WRAPPER_MEM_WRITE,
						20 - 1,
						rockchip->mem_bus_addr +
						(reg_no << 20),
						0);
```



IO空间映射：

```c
	// rockchip->io_bus_addr = 0xfbe00000
	// rockchip->io_size     = 0x100000
	// 设置Region31的映射关系
	offset = rockchip->mem_size >> 20;
	for (reg_no = 0; reg_no < (rockchip->io_size >> 20)/*0x1e00000右移二十位即为0x1e，也就是30个Region*/; reg_no++) {
		err = rockchip_pcie_prog_ob_atu(rockchip,
						reg_no + 1 + offset,
						AXI_WRAPPER_IO_WRITE,
						20 - 1,
						rockchip->io_bus_addr +
						(reg_no << 20),
						0);
		if (err) {
			dev_err(dev, "program RC io outbound ATU failed\n");
			return err;
		}
	}
```



Message空间映射：

```c
	/* Region32：assign message regions */
	rockchip_pcie_prog_ob_atu(rockchip, reg_no + 1 + offset,
				  AXI_WRAPPER_NOR_MSG,
				  20 - 1, 0, 0);

	rockchip->msg_bus_addr = rockchip->mem_bus_addr +
					((reg_no + offset) << 20);
```

# RK3399_PCIe_Host驱动分析_设备枚举

参考资料：

* 《PCI Express Technology 3.0》，Mike Jackson, Ravi Budruk; MindShare, Inc.
* [《PCIe扫盲系列博文》](http://blog.chinaaet.com/justlxy/p/5100053251)，作者Felix，这是对《PCI Express Technology》的理解与翻译
* 《PCI EXPRESS体系结构导读 (王齐)》
* 《PCI Express_ Base Specification Revision 4.0 Version 0.3 ( PDFDrive )》
* 《NCB-PCI_Express_Base_5.0r1.0-2019-05-22》
* [SOC中AXI总线是如何连接的](https://zhuanlan.zhihu.com/p/157137488)
* [AXI总线整理总结](https://blog.csdn.net/tristan_tian/article/details/89393045)
* [PCIe中MSI和MSI-X中断机制](https://blog.csdn.net/pieces_thinking/article/details/119431791)

开发板资料：

* https://wiki.t-firefly.com/zh_CN/ROC-RK3399-PC-PLUS/

本课程分析的文件：

* `linux-4.4_rk3399\drivers\pci\host\pcie-rockchip.c`



### 1. PCIe控制器的资源

上节视频我们分析了PCIe控制器驱动程序`pcie-rockchip.c`，它解析了设备树，得到了如下资源：

* 总线资源：就是总线号，从0到0x1f
* 内存资源：CPU地址基地址为0xfa000000，PCI地址基地址为0xfa000000，大小为0x1e00000
* IO资源：CPU地址基地址为0xfbe00000，PCI地址基地址为0xfbe00000，大小为0x100000

这3类资源记录在链表中：

![image-20220106102119348](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\79_res.png)



解析设备树时，把资源记录在这个链表里：

![image-20220106102303015](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\80_get_res.png)



res链表中记录的资源最终会放到pci_bus->bridge->windows链表里，如下图记录：

![image-20220106162022018](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\81_rk3399_pci_res.png)

### 2. 设备配置空间

本节内容参考：`PCI_SPEV_V3_0.pdf`

使用PCI/PCIe的目的，就是为了简单地访问它：像读写内存一样读写PCI/PCIe设备。

提问：

* 使用哪些地址读写设备？
* 这些地址的范围有多大？
* 是像内存一样访问它，还是像IO一样访问它？



每个PCI/PCIe设备都有配置空间，就是一系列的寄存器，对于普通的设备，它的配置空间格式如下。

里面有：

* 设备ID
* 厂家ID
* Class Code：哪类设备？存储设备？显示设备？等待
* 6个Base Address Register：

![image-20211117140817516](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\28_typ0_configuration_space.png)



#### 2.1 设备信息

* Vendor ID：厂家ID，PCI SIG组织给每个厂家都分配了一个独一的ID
* Device ID：厂家给自己的某类产品分配一个Device ID
* Revision ID：厂家自定义的版本号，可以认为是Device ID的延伸
* Header Type：
  * b[7]: 1-它是一个多功能设备("multi-function")，0-它是单功能设备("single-function")
  * b[6:0]: 00h-普通设备, 01h-桥设备，这个取值也决定了配置空间中偏移地址10h开始处的含义
    ![image-20220106092929107](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\74_config_space_two_types.png)

* Class Code：这是只读的寄存器，它含有3个字节，用来表明设备的功能，它分为3部分
  * 最高字节：表示"base class"，用来表示它属于内存卡、显卡等待
  * 中间字节：表示"sub-class"，再细分一下类别
  * 最低字节：用来表示寄存器级别的编程接口"Interface"
  * 示例如下：Base Class为01h时，表示它是一个存储设备，但是还可以继续使用sub-class、Interface细分
    ![image-20220106094233596](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\75_base_class_01h.png)



#### 2.2 基地址(Base Address)

普通的PCI/PCIe设备有6个基地址寄存器，简称为BAR：

![image-20220106094822299](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\76_base_register_reg.png)

BAR用于：

* 声明需要什么类型的空间：内存、IO、32位地址、64位地址？
* 声明需要的空间有多大
* 保存主控分配给它的PCI空间基地址



地址空间可以分为两类：内存(Memory)、IO：

* 对于内存，写入什么值读出就是什么值，可以提前读取
* 对于IO，它反应的是硬件当前的状态，每个时刻读到的值不一定相同



BAR的格式如下：

* 用于内存空间

![image-20220106100241585](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\77_bar_for_mem.png)

* 用于IO空间：

![image-20220106100318426](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\78_bar_for_io.png)



BAR怎么表示它想申请多大的空间？以32位地址为例：

* 软件往BAR写入0xFFFFFFFF
* 软件读BAR
* 读出的数值假设为0xFFF0,000?，忽略最低的4位，就得到：0xFFF0,0000
  * 这表示BAR中可以写入的"Base Address"只有最高的12位
  * 也就表示了最低的20位是可以变化的范围，所以这个空间大小为2^20=1M Byte

如果BAR表示它使用32位的地址，那么BAR0~BAR5可以分别表示6个地址空间。

如果BAR表示它使用64位的地址，那么BAR0和BAR1、BAR2和BAR3、BAR4和BAR5分别表示3个地址空间：

* 低序号的BAR表示64位地址的低32位
* 高序号的地址表示64位地址的高32位。



### 3. 扫描设备的过程

#### 3.1 核心: 构造pci_dev

扫描PCIe总线，对每一个PCIe桥、PCIe设备，都构造出对应的pci_dev：

* 填充pci_dev的各项成员，比如VID、PID、Class等
* 分配地址空间、写入PCIe设备



pci_dev结构体如下：

![image-20220106112538571](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\82_pci_dev.png)



对应pci_dev结构体里的设备信息：读取PCI设备的配置空间即可获得。

对应pci_dev结构体里的资源，本节课程先不分析irq。对于resource结构体，每个成员对应一个BAR。

resource结构体如下，要注意的是：里面记录的start、end等，是基于CPU角度看待的。也就是说，如果记录的是内存地址、IO地址，那么是CPU地址，不是PCI地址。并且这些地址是物理地址，要在软件中使用它们要先执行ioremap。

![image-20220106112818930](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\83_resource.png)

#### 3.2 代码分析

我们要找到这4个核心代码：

* 分配pci_dev
* 读取PCIe设备的配置空间，填充pci_dev中的设备信息
* 根据PCIe设备的BAR，得知它想申请什么类型的地址、多大？
* 分配地址，写入BAR



关键代码分为两部分：

* 读信息、得知PCIe设备想申请多大的空间

  ```shell
  rockchip_pcie_probe
      bus = pci_scan_root_bus(&pdev->dev, 0, &rockchip_pcie_ops, rockchip, &res);
  		pci_scan_root_bus_msi
              pci_scan_child_bus
              	pci_scan_slot
              		dev = pci_scan_single_device(bus, devfn);
  						dev = pci_scan_device(bus, devfn);
  							struct pci_dev *dev;
  							dev = pci_alloc_dev(bus);
  							pci_setup_device
                                  pci_read_bases(dev, 6, PCI_ROM_ADDRESS);	
                          pci_device_add(dev, bus);
  ```

  

* 分配空间

```c
rockchip_pcie_probe
	pci_bus_size_bridges(bus);
	pci_bus_assign_resources(bus);
		__pci_bus_assign_resources
            pbus_assign_resources_sorted
            	/* pci_dev->resource[]里记录有想申请的资源的大小, 
            	 * 把这些资源按对齐的要求排序
            	 * 比如资源A要求1K地址对齐，资源B要求32地址对齐
            	 * 那么资源A排在资源B前面, 优先分配资源A
            	 */
                list_for_each_entry(dev, &bus->devices, bus_list)
                    __dev_sort_resources(dev, &head);
				// 分配资源
				__assign_resources_sorted
                    assign_requested_resources_sorted(head, &local_fail_head);
```



##### 3.2.1 分配pci_dev结构

![image-20220106113805402](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\84_alloc_pci_dev.png)



##### 3.2.2 读取设备信息

![image-20220106114243209](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\85_pci_setup_device.png)



在`pci_scan_device`函数中，会先尝试读取VID、PID，成功的话才会继续调用`pci_setup_device`：

![image-20220106114505761](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\86_pci_scan_device.png)



在`pci_setup_device`内部，会继续读取其他信息：

![image-20220106114658658](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\87_pci_setup_device_code.png)



##### 3.2.3 读BAR

![image-20220106115246161](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\88_pci_read_bar.png)

![image-20220106115501793](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\89_pci_read_bar_code1.png)



`pci_read_bases`函数代码分析：

![image-20220106115858880](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\90_pci_read_bases.png)



`pci_read_bases`函数又会调用`__pci_read_base`，`__pci_read_base`只是读BAR，算出想申请的空间的大小：

* 读BAR，保留原值
* 写0xFFFFFFFF到BAR
* 在读出来，解析出所需要的地址空间大小，记录在pci_dev->resource[ ]里
  * pci_dev->resource[ ].start = 0;
  * pci_dev->resource[ ].end = size - 1;



把前面讲过的贴出来，有助于理解代码：

BAR怎么表示它想申请多大的空间？以32位地址为例：

* 软件往BAR写入0xFFFFFFFF
* 软件读BAR
* 读出的数值假设为0xFFF0,000?，忽略最低的4位，就得到：0xFFF0,0000
  * 这表示BAR中可以写入的"Base Address"只有最高的12位
  * 也就表示了最低的20位是可以变化的范围，所以这个空间大小为2^20=1M Byte



以下是`__pci_read_bases`函数的代码分析。

* 得到大小(原始数据，需要进一步解析)：比如下列代码中sz被赋值为0xFFF0,000?，需要进一步解析

![image-20220106151603798](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\91_get_size.png)



##### 3.2.4 分配地址空间

这部分代码的函数调用非常深，我们抓住2个问题即可：

* 从哪里分配得到地址空间？
  * 在设备树里指明了CPU地址、PCI地址的对应关系，这些作为"资源"记录在pci_bus里
  * 读BAR时，在pci_dev->resource[]里记录了它想申请空间的大小
* 分配得到的基地址，要写入BAR



代码调用关系如下：

* 把要申请的资源, 按照对齐要求排序，然后调用assign_requested_resources_sorted，代码如下：

  ```shell
  /* 把要申请的资源, 按照对齐要求排序
   * 然后调用assign_requested_resources_sorted
   */
  
  rockchip_pcie_probe
  	pci_bus_size_bridges(bus);
  	pci_bus_assign_resources(bus);
  		__pci_bus_assign_resources
              pbus_assign_resources_sorted
              	/* pci_dev->resource[]里记录有想申请的资源的大小, 
              	 * 把这些资源按对齐的要求排序
              	 * 比如资源A要求1K地址对齐，资源B要求32地址对齐
              	 * 那么资源A排在资源B前面, 优先分配资源A
              	 */
                  list_for_each_entry(dev, &bus->devices, bus_list)
                      __dev_sort_resources(dev, &head);
  				// 分配资源
  				__assign_resources_sorted
                      assign_requested_resources_sorted(head, &local_fail_head);
  ```

* `assign_requested_resources_sorted`函数做两件事

  * 分配地址空间

  * 把这块空间对应的PCI地址写入PCIe设备的BAR

  * 代码如下：

    ```shell
    assign_requested_resources_sorted(head, &local_fail_head);
        pci_assign_resource
            ret = _pci_assign_resource(dev, resno, size, align);
            	// 分配地址空间
                __pci_assign_resource
                    pci_bus_alloc_resource
                        pci_bus_alloc_from_region
                            /* Ok, try it out.. */
                            ret = allocate_resource(r, res, size, ...);
                                err = find_resource(root, new, size,...);
                                    __find_resource
                                        
                                        // 从资源链表中分配地址空间
                                        // 设置pci_dev->resource[]
                                        new->start = alloc.start;
                                        new->end = alloc.end;
                // 把对应的PCI地址写入BAR
                pci_update_resource(dev, resno);
                    pci_std_update_resource
                        /* 把CPU地址转换为PCI地址: PCI地址 = CPU地址 - offset 
                         * 写入BAR
                         */
                        pcibios_resource_to_bus(dev->bus, &region, res);
                        new = region.start;
                        reg = PCI_BASE_ADDRESS_0 + 4 * resno;
                        pci_write_config_dword(dev, reg, new);
    ```

    

# INTx_MSI_MSIX三种中断机制分析

参考资料：

* 《PCI_SPEV_V3_0.pdf》6.8节

* [PCIe中MSI和MSI-X中断机制](https://blog.csdn.net/pieces_thinking/article/details/119431791)

* [PCIe学习笔记之MSI/MSI-x中断及代码分析](https://blog.csdn.net/yhb1047818384/article/details/106676560/)

* [msix中断分析](https://blog.csdn.net/weijitao/article/details/46566789)

* [MSI中断与Linux实现](https://www.cnblogs.com/gunl/archive/2011/06/09/2076892.html)

* [ARM GICv3中断控制器](https://blog.csdn.net/yhb1047818384/article/details/86708769)

开发板资料：

* https://wiki.t-firefly.com/zh_CN/ROC-RK3399-PC-PLUS/

本课程分析的文件：

* `linux-4.4_rk3399\drivers\pci\host\pcie-rockchip.c`

### 1. PCI设备的INTx中断机制

传统PCI设备的引脚中有4条线：INTA#、INTB#、INTC#、INTD#，"#"表示低电平有效，如下图所示：

![image-20220114144708980](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\96_pci_intx.png)



PCI设备就像普通的设备一样，通过物理引脚发出中断信号：

![image-20220114150723805](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\97_pci_intx_example.png)



在PCI设备的配置空间，它声明：通过INTA#、INTB#、INTC#还是INTD#发出中断。

![image-20220114152410023](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\98_pci_config_reg_for_int.png)

配置空间有2个寄存器：Interrupt Pin、Interrupt Line，作用如下：

* Interrupt Pin：用来表示本设备通过哪条引脚发出中断信号，取值如下

  | Interrupt Pin取值 | 含义              |
  | ----------------- | ----------------- |
  | 0                 | 不需要中断引脚    |
  | 1                 | 通过INTA#发出中断 |
  | 2                 | 通过INTB#发出中断 |
  | 3                 | 通过INTC#发出中断 |
  | 4                 | 通过INTD#发出中断 |
  | 5~0xff            | 保留              |

* Interrupt Line：给软件使用的，PCI设备本身不使用该寄存器。软件可以写入中断相关的信息，比如在Linux系统中，可以把分配的virq(虚拟中断号)写入此寄存器。软件完全可以自己记录中断信息，没必要依赖这个寄存器。



INTx中断是电平触发，处理过程如下：

* PCI设备发出中断：让INTx引脚变低
* 软件处理中断，清除中断：写PCI设备的某个寄存器，导致PCI设备取消中断
* PCI设备取消中断：让INTx引脚变高





### 2. PCIe设备的INTx中断机制

PCIe设备的配置空间也同样有这2个寄存器：Interrupt Pin、Interrupt Line，它们的作用跟PCI设备完全一样。

PCI总线上有INTA#~INTD#这些真实存在的引脚，但是PCIE总线上并没有这些引脚，PCIe设备怎么兼容INTx中断机制？

PCIe设备通过"INTx模拟"(PCI Compatible INTx Emulation)来实现传统的INTx中断，当设备需要发出中断时，它会发出特殊的TLP包：

![image-20220114155205690](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\99_tlp_for_int.png)

TLP头部中，Message Code被用来区分发出的是哪类TLP包，为例"INTx模拟"，有两类TLP包：

* Assert_INTx
  ![image-20220114155744259](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\100_Assert_INTx.png)
* Deassert_INTx
  ![image-20220114160030443](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\101_Deassert_INTx.png)



跟传统PCI设备类似，这个"INTx模拟"的处理过程也是一样的：

* PCIe设备发出中断：发出Assert_INTx的TLP包
* 软件处理中断，清除中断：写PCIe设备的某个寄存器，导致PCIe设备取消中断
* PCIe设备取消中断：发出Deassert_INTx的TLP包



硬件框图如下：

![image-20220114160521777](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\102_pcie_intx_example.png)

对于软件开发人员来说，他感觉不到变化：

* PCI设备通过真实的引脚传输中断
* PCIe设备通过TLP实现虚拟的引脚传输中断



PCIe控制器内部接收到INTx的TLP包后，就会向GIC发出中断，最终导致CPU接收到中断信号。

对应的中断程序执行时，会读取PCIe控制器的寄存器，分辨发生的是INTA#~INTD#这4个中断的哪一个。



### 3. MSI中断机制

在PCI系统中，使用真实的引脚发出中断已经带来了不方便：

* 电路板上需要布线
* 只有4条引脚，多个PCI设备共享这些引脚，中断处理效率低。

在PCI系统中，就已经引入了新的中断机制：MSI，Message Signaled Interrupts。

在初始PCI设备时，可以告诉它一个地址(主控芯片的地址)、一个数据：

* PCI设备想发出中断时，往这个地址写入这个数据就可以触发中断
* 软件读到这个数据，就知道是哪个设备发出中断了



流程及硬件框图如下：

* 写哪个地址可以触发中断？可能是PCI/PCIe控制器里面的某个地址，也可能是GIC的某个地址
* 初始化PCI/PCIe设备时，把该地址(cpu_addr)转换为pci_addr，告知PCI/PCIe设备(写入它的配置空间)
* PCI/PCIe设备要发出中断时，会发起一个"写内存传输"：往pci_addr写入数据value
* 这导致cpu_addr被写入数据value，触发中断

![image-20220114165335847](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\104_msi_msix.png)

上图中的"pci_addr/value"保存在哪里？保存在设备的配置空间的capability里。



#### 3.1 capability  

capability的意思是"能力"，PCI/PCIe设备可以提供各种能力，所以在配置空间里有寄存器来描述这些capability：

* 配置空间里有第1个capability的位置：Capabilities Pointer
* 它指向第1个capability的多个寄存器，这些寄存器也是在配置空间里
* 第1个capability的寄存器里，也会指示第2个capability在哪里

![image-20220114163327615](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\103_capability.png)



Capability示例图如下：

* 配置空间0x34位置，存放的是第1个capability的位置：假设是 A4H
* 在配置空间0xA4位置，找到第1个capability，capability的寄存器有如下约定
  * 第1个字节表示ID，每类capability都有自己的ID
  * 第2个字节表示下一个capability的位置，如果等于0表示这是最后一个capability
  * 其他寄存器由capability决定，所占据的寄存器数量由capability决定
* 第1个capability里面，它表示下一个capability在5CH
* 在配置空间0x5C位置，找到第2个capability
  * 第1个字节表示ID，第2个字节表示下一个capability的位置(图里是E0H)
  * 其他字节由capability本身决定
* 在配置空间0xE0位置，找到第3个capability
  * 第1个字节表示ID
  * 第2个字节表示下一个capability的位置，这里是00H表示没有更多的capability了
  * 其他字节由capability本身决定

![image-20220117103544744](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\105_capabilities.png)



#### 3.2 MSI capability

一个PCI设备是否支持MSI，需要读取配置空间的capability来判断: 有MSI capability的话就支持MSI机制。

在配置空间中，MSI capability用来保存：pci_addr、data。表示：PCI设备往这个pci_addr写入data，就可以触发中断。

有如下问题要确认：

* pci_addr是32位、还是64位？
* 能触发几个中断？通过地址来分辨，还是通过数据来分辨？
* 这些中断能否屏蔽？
* 能否读出中断状态？
* 这个些问题，都由capability里面的"Message Control"决定。

MSI capability格式如下：

![image-20220117115002363](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\106_msi_capability.png)

#### 3.3 格式解析

MSI Capability格式的含义如下：

* Capability ID：对于MSI capability，它的ID为05H

* Next Pointer：下一个capability的位置，00H表示这是最后一个capability

* Message Control

  | 位   | 域                         | 描述                                                         |
  | ---- | -------------------------- | ------------------------------------------------------------ |
  | 8    | Per-vector masking capable | 是否支持屏蔽单个中断(vector)：<br />1: 支持<br />0: 不支持<br />这是只读位。 |
  | 7    | 64 bit address capable     | 是否支持64位地址：<br />1: 支持<br />0: 不支持<br />这是只读位。 |
  | 6:4  | Multiple Message Enable    | 系统软件可以支持多少个MSI中断？<br />PCI设备可以表明自己想发出多少个中断，<br />但是到底能发出几个中断？<br />由系统软件决定，它会写这些位，表示能支持多少个中断：<br />000: 系统分配了1个中断<br />001: 系统分配了2个中断<br />010: 系统分配了4个中断<br />011: 系统分配了8个中断<br />100: 系统分配了16个中断<br />101: 系统分配了32个中断<br />110: 保留值<br />111: 保留值<br />这些位是可读可写的。 |
  | 3:1  | Multiple Message Capable   | PCI设备可以表明自己想发出多少个中断：<br />000: 设备申请1个中断<br />001: 设备申请2个中断<br />010: 设备申请4个中断<br />011: 设备申请8个中断<br />100: 设备申请16个中断<br />101: 设备申请32个中断<br />110: 保留值<br />111: 保留值<br />这些位是只读的。 |
  | 0    | MSI Enable                 | 使能MSI：<br />1: 使能<br />0: 禁止                          |

* Message Address/Message Uper Address：地址

  * 32位地址保存在Message Address中
  * 64位地址：低32位地址保存在Message Address中，高32位地址保存在Message Uper Address中
  * 这些地址是系统软件初始化PCI设备时分配的，系统软件把分配的地址写入这些寄存器
  * 这些地址属于PCI地址空间

* Message Data：数据

  * 这个寄存器只有15位，PCI设备发出中断时数据是32位的，其中高16位数据为0
  * 这个寄存器的数值是系统软件初始化设备时写入的
  * 当PCI设备想发出中断是会发起一次写传输：
    * 往Message Address寄存器表示的地址，写入Message Data寄存器的数据
    * 如果可以发出多个中断的话，发出的数据中低位可以改变
    * 比如"Multiple Message Enable"被设置为"010"表示可以发出4个中断
    * 那么PCI设备发出的数据中bit1,0可以修改

* Mask Bits/Pending Bits: 屏蔽位/挂起位，这是可选的功能，PCI设备不一定实现它

  * Mask Bits：每一位用来屏蔽一个中断，被系统软件设置为1表示禁止对应的中断
  * Pending Bits：每一位用来表示一个中断的状态，这是软件只读位，它的值为1表示对应中断发生了，待处理



### 4. MSI-X中断机制

MSI机制有几个缺点：

* 每个设备的中断数最大是32，太少了
* 需要系统软件分配连续的中断号，很可能失败，也就是说设备想发出N个中断，但是系统软件分配给它的中断少于N个
* 通过MSI发出中断时，地址是固定的

于是引入了MSI-X机制：Enhanced MSI interrupt support，它解决了MSI的缺点：

* 可以支持多大2048个中断
* 系统软件可以单独设置每个中断，不需要分配连续的中断号
* 每个中断可以单独设置：PCI设备使用的"地址/数据"可以单独设置



假设MSI-X可以支持很多中断，每个中断的"地址/数据"都不一样。提问：在哪里描述这些信息？

* "地址/数据"：
  * 不放在配置空间，空间不够
  * 放在PCI设备的内存空间：哪个内存空间？哪个BAR？内存空间哪个位置(偏移地址)？
  * 系统软件可以读写这些内存空间
* 中断的控制信息
  * 使能/禁止？
  * 地址是32位还是64位？
  * 这些控制信息也是保存在PCI设备的内存空间
* 中断的状态信息(挂起？)
  * 这些信息也是保存在PCI设备的内存空间



#### 4.1 MSI-X capability

一个PCI设备是否支持MSI-X，需要读取配置空间的capability来判断: 有MSI-X capability的话就支持MSI-X机制。

MSI-X capability格式如下：

![image-20220117142556453](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\107_msi-x_capability.png)



#### 4.2 MSI-X capability格式解析

格式解析如下：

* Capability ID：对于MSI-X capability，它的ID为11H

* Next Pointer：下一个capability的位置，00H表示这是最后一个capability

* Message Control

  | 位   | 名            | 描述                                                         |
  | ---- | ------------- | ------------------------------------------------------------ |
  | 15   | MSI-X Enable  | 是否使能MSI-X：<br />1: 使能<br />0: 禁止<br />注意: MSI-X和MSI不能同时使能。 |
  | 14   | Function Mask | 相当于MSI-X中断总开关：<br />1: 所有中断禁止<br />0: 有各个中断的Mask位决定是否使能对应中断 |
  | 13   | 保留          |                                                              |
  | 10:0 | Table Size    | 系统软件读取这些位，算出MSI-X Table的大小，也就是支持多少个中断<br />读出值为"N-1"，表示支持N个中断 |

* Table Offset/Table BIR ：BIR意思为"BAR Indicator register"，表示使用哪个BAR。

  | 位   | 域           | 描述                                                         |
  | ---- | ------------ | ------------------------------------------------------------ |
  | 31:3 | Table Offset | MSI-X Table保存在PCI设备的内存空间里，<br />在哪个内存空间？有下面的"Table BIR"表示。<br />在这个内存空间的哪个位置？由当前这几位表示。 |
  | 2:0  | Table BIR    | 使用哪个内存空间来保存MSI-X Table？<br />也就是系统软件使用哪个BAR来访问MSI-X Table？<br />取值为0~5，表示BAR0~BAR5 |

* PBA Offset/PBA BIR：PBA意思为"Pending Bit Array"，用来表示MSI-X中断的挂起状态。

  | 位   | 域         | 描述                                                         |
  | ---- | ---------- | ------------------------------------------------------------ |
  | 31:3 | PBA Offset | PBA保存在PCI设备的内存空间里，<br />在哪个内存空间？有下面的"PBA BIR"表示。<br />在这个内存空间的哪个位置？由当前这几位表示。 |
  | 2:0  | PBA BIR    | 使用哪个内存空间来保存PBA？<br />也就是系统软件使用哪个BAR来访问PBA？<br />取值为0~5，表示BAR0~BAR5 |



#### 4.3 MSI-X Table

PCI设备可以往某个地址写入某个数据，从而触发MSI-X中断。

这些"地址/数据"信息，是由系统软件分配的，系统软件要把"地址/数据"发给PCI设备。

PCI设备在哪里保存这些信息？

* 在PCI设备的内存空间
* 在哪个内存空间？由MSI-X capability的"Table BIR"决定
* 在这个内存空间的哪个位置？由MSI-X capability的"Table Offset"决定

MSI-X Table格式如何？如下图所示：

![image-20220117144509485](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\108_msi-x_table.png)

上图中，Msg Data、Msg Addr Msg Upper Addr含义与MSI机制相同：PCI设备要发出MSI-X中断时，往这个地址写入这个数据。如果使用32位地址的话，写哪个地址由Msg Addr寄存器决定；如果使用64位地址的话，写哪个地址由Msg Addr和Msg Upper Addr这两个寄存器决定。

Vector Control寄存器中，只有Bit0有作用，表示"Mask Bit"。系统软件写入1表示禁止对应中断，写入0表示使能对应中断。



#### 4.4 PBA

PBA意思为"Pending Bit Array"，用来表示MSI-X中断的挂起状态。它的格式如下：

这些"挂起"信息，是由PCI设备设置，系统软件只能读取这些信息。

PCI设备在哪里保存这些信息？

* 在PCI设备的内存空间
* 在哪个内存空间？由MSI-X capability的"PBA BIR"决定
* 在这个内存空间的哪个位置？由MSI-X capability的"PBA Offset"决定

PBA格式如下：每一位对应一个中断，值为1表示中断发生了、等待处理。

![image-20220117145232528](E:\嵌入式笔记大全\嵌入式笔记大全\操作系统\嵌入式笔记大全\PCIE\PCIE.assets\109_pba.png)



### 5. MSI/MSI-X操作流程

#### 5.1 扫描设备

扫描设备，读取capability，确定是否含有MSI capability、是否含有MSI-X capability。



#### 5.2 配置设备

一个设备，可能都支持INTx、MSI、MSI-X，这3中方式只能选择一种。

##### 5.2.1 MSI配置

系统软件读取MSI capability，确定设备想申请多少个中断。

系统软件确定能分配多少个中断给这个设备，并把"地址/数据"写入MSI capability。

如果MSI capability支持中断使能的话，还需要系统软件设置MSI capability来使能中断。

注意：如果支持多个MSI中断，PCI设备发出中断时，写的是同一个地址，但是数据的低位可以发生变化。

比如支持4个MSI中断时，通过数据的bit1、bit0来表示是哪个中断。



##### 5.2.2 MSI-X配置

MSI-X机制中，中断相关的更多信息保存在设备的内存空间。所以要使用MSI-X中断，要先配置设备、分配内存空间。

然后，系统软件读取MSI-X capability，确定设备需要多少个中断。

系统软件确定能分配多少个中断给这个设备，并把多个"地址/数据"写入MSI-X Table。

注意：PCI设备要发出MSI-X中断时，它会往"地址"写入"数据"，这些"地址/数据"一旦配置后是不会变化的。MSI机制中，数据可以变化，MSI-X机制中数据不可以变化。

使能中断：设置总开关、MSI-X Table中某个中断的开关。



注意：MSI-X Table中，每一项都可以保存一个"地址/数据"，Table中"地址/数据"可以相同，也就是说：PCI设备发出的中断可以是同一个。



#### 5.3 设备发出中断

PCI设备发出MSI中断、MSI-X中断时，都是发起"数据写"传输，就是往指定地址写入指定数据。

PCI控制器接收到数据后，就会触发CPU中断。



#### 5.4 中断函数

系统软件执行中断处理函数。





















































