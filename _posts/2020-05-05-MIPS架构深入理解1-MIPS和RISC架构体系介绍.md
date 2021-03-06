---
layout:     post
title:      MIPS架构深入理解1-MIPS和RISC架构体系介绍
subtitle:   从宏观上对MIPS架构有一个基本理解，对MIPS架构和CISC架构差异有一个感官上的认识
date:       2020-05-05
author:     Tupelo Shen
header-img: img/post-bg-unix-linux.jpg
catalog:    true
tags:
    - MIPS
    - RISC
---

<h1 id="0">0 目录</h1>

* [1.1 流水线](#1.1)
    - [1.1.1 制约流水线效率的因素](#1.1.1)
    - [1.1.2 制约流水线效率的因素](#1.1.2)
* [1.2 MIPS架构5级流水线](#1.2)
* [1.3 RISC和CISC对比](#1.3)
* [1.4 MIPS架构的发展](#1.4)
* [1.5 MIPS架构和CISC架构的对比](#1.5)

---

众多RISC精简指令集架构中，MIPS架构是最优雅的"舞者"。就连它的竞争者也为其强大的影响力所折服。DEC公司的Alpha指令集（现在已被放弃）和HP的Precision都受其影响。虽说，优雅不足以让其在残酷的市场中固若金汤，但是，MIPS架构还是以最简单的设计成为每一代CPU架构中，执行效率最快的那一个。

作为一个从学术项目孵化而来的成果，简单的架构是MIPS架构商业化的需要。它拥有一个小的设计团队，制造和商业推广交给它的半导体合作伙伴们。结果就是，在半导体行业领域中，它拥有广泛的合作制造商：LSI逻辑、LSI、东芝、飞利浦、NEC和IDT等都是它的合作伙伴。值得一提的是，国内的龙芯买断了它的指令集架构，成为芯片国产化的佼佼者。

在低端CPU领域，MIPS架构昙花一现。目前主要应用于嵌入式行业，比如路由器领域，几乎占据了大部分的市场。

MIPS架构的CPU是RISC精简指令CPU中的一种，诞生于RISC学术研究最为活跃的时期。RISC（精简指令集计算机）这个缩略语是1986年-1989年之间诞生的所有新CPU架构的一个标签TAG，它为新诞生的这些高性能架构提供了想法上的创新。有人形象的说，"RISC是1984年之后诞生的所有计算机架构的基础"。虽说，略显夸张，但也是无可争议的事实，谁也无法忽略RISC精简指令集的那些先驱们所做的贡献。

MIPS领域最无法忽视的贡献者是Stanford大学的MIPS项目。之所以命名成MIPS，即可以代表`microcomputer without interlocked pipeline stages`-无互锁流水线的微处理器的英文意思，又可以代表`millions of instructions per second`每秒执行百万指令的意思，真是一语双关。看起来，MIPS架构主要研究方向还是CPU的流水线架构，让它如何更高效地工作。那接下来，我们就从流水线开始讲起。

> 流水线的互锁是影响CPU指令执行效率的关键因素之一。

<h2 id="1.1">1.1 流水线</h2>

假设有一家餐馆，我们称之为Evie的炸鱼薯条店。在其中，每一名顾客排队取餐（炸鳕鱼、炸薯片、豌豆糊、一杯茶）。店员装好盘子后，顾客找桌子坐下来就餐。这是，我们常见的餐馆就餐方式。

Evie的薯条是当地最好的小吃。所以，每当赶大集的时候，长长的队伍都要延伸到店外。所以，当隔壁的木屐店关门的时候，Evie盘下来，扩大了店面，增加了桌子。但是，这还是于事无补。因为忙碌的市民根本没空坐下来喝杯茶。（因为Evie还没有找到排长队的根本原因，......）

Evie炸鳕鱼和Bert的薯条是店里的招牌，来的顾客都会点这两样。但是他们只有一个柜台，所以，当有一名顾客执行点一杯茶的时候，而恰好他又在点鳕鱼和薯条的顾客后面，那他只能等待了.....。终于有一天，Evie想起了一个绝妙的主意：他们把柜台延长，Evie、Bert、Dionysus和Mary四名店员一字排开；每当顾客进来，Evie负责鳕鱼装盘，Bert添加薯条，Dionysus盛豌豆糊，Mary倒茶并结账。这样，一次可以有四名顾客同时被服务，多么天才的想法啊。

再也没有长长的排队人群，Evie的店收入增加了......。

这就是流水线，将重复的工作分成若干份，每个人负责其中一份。虽然每名顾客总的服务时间延长，但是，同时有四名顾客被服务，提高了整个取餐付账的效率。下图1-1就是Evie店里的流水线结构图。

<img src="https://raw.githubusercontent.com/tupelo-shen/my_test/master/doc/linux/mips-architecture/others/images/see_mips_run_1_1.PNG">

那么，我们将CPU执行一条指令分解成取指令、解码、查找操作数、执行运算、存储结果五步操作的话，是不是跟上面Evie的店里的流水线就极其类似了呢。本质上讲，等待执行的程序就是一个个指令的队列，等待CPU一个个执行。

流水线当然不是RSIC指令集的新发明，CSIC复杂指令集也采用流水线的设计。差异就是，RSIC重新设计指令集，使流水线更有效率。那到底什么是制约流水线效率的关键因素呢？

<h3 id="1.1.1">1.1.1 制约流水线效率的因素</h3>

我们都知道那个著名的"木桶效应"：决定木桶的装水量的是最短的那个木头，而不是最长的。同样的，如果我们保证指令执行的每一步都占用相同时间的话，那么这个流水线将是一个完美的高效流水线。但现实往往是残酷的，CPU的工作频率远远大于读写内存的工作频率（都不是一个量级）。

让我们回到Evie的店中。顾客Cyril是一个穷人，经常缺钱，所以在Mary没有收到他的钱之前，Evie就不会为他服务。那么，现在Cyril就卡在Evie的位置处，知道Mary处理完前面三名顾客，再给他结完账，Evie才会继续为他服务。所以，在这个取餐的流水线上，Cyril就是一个麻烦制造者，因为他需要一个他人正在使用资源（Mary结账）。（这种情况可以对应CPU存储指令使用锁总线的方式保证对内存的独占访问。）

有时候，Daphne和Lola分别买一部分食物，然后彼此之间共享。比如说，Lola买不买薯条，取决于Daphne是否购买一杯茶。因为光吃薯条不喝点饮料的话，也许Daphne会噎着或者齁着。那么，Lola就会在售卖员Bert处着急等待Daphne在Mary处买一杯茶，这中间就发生了时间上的空隙，我们将其称为流水线上的间隙。（这是不是很像条件分支？）

当然了，不是所有的依赖关系都是坏的结果。假设有一名顾客Frank，总是模仿Fred点餐，也许Frank是Fred的粉丝呢。这其实蕴涵着通过Cache命中提高存取内存和分支预测工作效率的基础。

你当然在想，把这些麻烦的顾客剔除出去，不就是一个效率超高的流水线吗？但是，Evie还要在这儿生活，怎么可能得罪顾客呢。计算机处理器的行业大佬Intel现在也面临着这样的问题：不可能不兼容以前的软件，完全另起炉灶搞一个新的架构的话，可能会流失很多客户。于是，只能在旧的架构上缝缝补补又十年啊。这也给了RSIC指令集发展的大好机会。

<h3 id="1.1.2">1.1.2 流水线和Cache</h3>

计算机CPU处理速度和内存读取速度的匹配问题是提高CPU工作效率的关键，也就是"木桶效应"的那个短板。So，为了加速对内存的访问，CPU设计中引入了Cache。所谓的Cache，就是一个小的高速内存，用来拷贝内存中的一段数据。Cache中的最小数据单元是line，每个line对应一小段内存地址（常见的line大小为64字节）。每个Line不仅包含从主内存读取的数据，还包括其地址信息（TAG）和状态信息。当CPU想要访问内存中的数据时，先由内存管理单元搜索Cache，如果数据存在，则立即返回给CPU，这称为Cache命中；如果不存在，则称为Cache未命中，此时，内存管理单元再去主内存中查找相关数据，返回给CPU并在Cache中留下备份。Cache当然不知道CPU下一步需要什么数据，所以它只能保留CPU最近使用过的数据。如果需要为新拷贝的主内存数据，它就会选择合适的数据丢弃，这涉及到Cache替换策略算法。

Cache大约9成时间能够提供CPU想要的数据，所以大大提高了CPU读取数据的速率，从而提高了流水线的工作效率。

因为指令不同于数据，是只读属性，所以，MIPS架构采用哈弗结构，将数据Cache和指令Cache分开。这样就可以同时读取指令和读写变量了。

<h2 id="1.2">1.2 MIPS架构5级流水线</h2>

<img src="https://raw.githubusercontent.com/tupelo-shen/my_test/master/doc/linux/mips-architecture/others/images/see_mips_run_1_2.PNG">

图1.2: MIPS-5级流水线

MIPS本身就是基于流水线优化设计的架构，所以，将MIPS指令分为5个阶段，每个阶段占用固定的时间，在此，固定的时间其实就是处理器的时钟周期（有2个指令花费半个时钟周期，所以，MIPS的5级流水线实际上占据4个时钟周期）。

所有的指令都是严格遵守流水线的各个阶段，即使某个阶段什么也不做。这样做的结果就是，只要Cache命中，每个时钟周期CPU都会启动一条指令。

让我们看看每一个阶段都做了什么：

* **取指令-IF**
    
    从I-Cache中取要执行的指令。

* **读寄存器-RD** 
 
    取CPU寄存器中的值。

* **算术、逻辑运算-ALU**

    执行算术或逻辑运算。（浮点运算和乘除运算在一个时钟周期内无法完成，我们以后再写文章专门讲解FPU相关的知识）

* **读写内存-MEM**

    也就是读写D-Cache。至于这儿为什么说读写数据缓存，是因为内存的读写速度实在太慢了，无法满足CPU的需要。所以出现了D-Cache这种高速缓存。但即便如此，在读写D-Cache期间，平均每4条指令就会有3条指令什么也做不了。但是，每条指令在这个阶段都应该是独占数据总线的，不然会造成访问D-Cache冲突。这就是内存屏障和总线锁存在的理由。

* **写回寄存器-Writeback**

    将结果写入寄存器。

当然，上面的流水线只是一个理论上的模型而已。但实际上，有一些MIPS架构的CPU会有更长的流水线或者其它的不同。但是，5级流水线架构确是这一切的出发点和基础。

流水线的严格规定，限制了指令可以做的事情。

首先，所有的指令具有相同的长度（32位），读取指令使用相同的时间。这降低了流水线的复杂度，比如，指令中没有足够的位用来编码复杂的寻址模式。

这种限制当然也有不利：基于X86架构的程序，平均指令长度也只有大约3个字节多一点。所以，MIPS架构占用更多的内存空间。

第二，MIPS架构的流水线设计排除了对内存变量进行任何操作的指令的实现。内存数据的获取只能在阶段4，这对于算术逻辑单元有点延迟。内存访问只能通过load或store指令进行。（MIPS架构的汇编也是最简单易懂的代码之一）

尽管有这些问题，但是MIPS架构的设计者也在思考，如何使CPU可以被编译器更加简单高效地优化。一些编译器高效优化的要求和流水线的设计要求是兼容的，所以MIPS架构的CPU具有32个通用寄存器，使用具有三个操作数的算术/逻辑指令。那些复杂的特殊目的的指令也是编译器不愿意产生的。通俗地讲，编译器能不用复杂指令就不用复杂指令。

<h2 id="1.3">1.3 RISC和CISC对比</h2>

我们如何区分RISC和CISC指令集定义上的区别。在我看来，RISC就是架构和指令集关系的描述。20世纪80年代中期，诞生了一批新的架构，在这些架构中，巧用指令集以最大化这些基于流水线实现的架构的效率。这些被巧用的指令集就被称为精简指令集，采用这些指令集的架构也就称为精简指令集计算机（RISC）。基于RSIC精简指令集设计的CPU架构有SPARC、MIPS、PowerPC、HP Precision、DEC Alpha和ARM。

与此形成鲜明对比的是，CISC复杂指令集计算机与流水线的实现没有多大关系。CISC的设计出发点主要是从代码的易用性上考虑的。1985年之后的计算机架构，基本上都是基于RISC实现的。CISC主要是1985年之前的架构使用。比如英特尔公司的X86架构和摩托罗拉公司的680x系列。

总结来说，RISC和CISC的共同点都是对指令集的描述，但是RISC对于CPU的流水线架构的实现影响比较大，而CISC指令集对于架构的影响不大。虽然，现在的X86架构大量借鉴了RISC的一些实现技巧，用来提升自己的性能。但其本质上还是复杂指令集计算机（CISC）架构。

<h2 id="1.4">1.4 MIPS架构的发展</h2>

纵观MIPS架构的近40年的发展历程，虽经历过辉煌，但现在也日渐式微。网上有许许多多关于MIPS架构的评论或者见解。笔者对于市场一窍不通，故不在此班门弄斧。但是我本人还是非常欣赏MIPS架构的设计理念：强调软硬件协同提高性能，同时简化硬件设计。

咱们在此提一下国内的龙芯公司，号称"国产芯"。它由于直接买断了MIPS指令集的授权，所以不受技术封锁的影响。而且，MIPS指令集的授权和ARM指令集的授权有着本质上的不同：MIPS授权后，允许设计厂商自行对架构或者指令集进行自定义；但是ARM授权不允许厂商对其授权的架构进行自定义（当然了，近些年，ARM也已经授权了苹果、高通等公司可以自行定义ARM授权的架构）。所以，龙芯选择MIPS是技术上的选择，也是时代的选择。虽然，最近几年RISC-V开源指令集非常火热，但是其上的软件生态同样需要布局。而龙芯已经在MIPS架构上花费了20年的人力物力，也已经有了一些技术上的沉淀。完全掉头转向RISC-V开源指令集也是不太现实的。希望龙芯能够在CPU领域继续深耕，逐步完善生态系统，实现真正的国产芯片自主化吧。

<h2 id="1.5">1.5 MIPS和CISC的对比</h2>

大部分的程序员对汇编语言的认知都来源于X86架构，毕竟是最早的CPU架构之一。但是，当你看见基于MIPS架构的汇编代码时，你还是得到一些惊喜。我个人的感觉就是，基于MIPS架构的汇编语言理解起来还是比较容易的，毕竟它是精简指令集。但是，它又有一些程序代码设计上的奇技淫巧，需要我们额外理解。我们将通过一下几个方面进行归纳总结：

1. 为了提高流水线的效率而对MIPS指令操作所施加的限制；
2. 极度简单的load/store操作；
3. 有意省略的一些操作；
4. 指令集的一些预想不到的特性；
5. 流水线操作中对程序员可见的那些点。

最初提出MIPS设想的斯坦福大学的研究小组，特别关注能够实现的简短的流水线架构。后来的事实也证明，他们的判断是完全正确的，由流水线而引申出的许多设计决定被证明，能够更容易、更快速地实现更高的性能。

<h3 id="1.5.1">1.5.1 MIPS指令集的限制</h3>

* 所有的指令都是32位长度：

    这意味着没有指令仅占用2个或3个字节的内存空间（也就是说，通常情况下，MIPS架构的二进制文件比X86架构大百分之二十或三十），也没有指令超过4个字节。

    这样的结果就是，只通过一条指令无法操作32位常数。因为一个32位指令，没有足够的位为操作数和目标寄存器进行编码。MIPS架构的设计者为两条指令保留了26位，这两条特殊的指令就是跳转jump指令，一个跳转到指定的目标地址，一个跳转到子程序。其它的指令都只有16位留给常数。于是，加载任意一个32位的常数，都需要2条指令才能实现，条件分支被限制到64K的作用范围。


* 指令操作必须适合流水线：

    指令的每一步操作都必须在流水线的正确阶段执行，且必须在一个时钟周期内完成。比如，寄存器写回操作只提供写一个值到寄存器中，所以指令在这个阶段只能改变某个寄存器的内容。

    乘除指令无法在一个时钟周期内完成。MIPS架构的CPU使用的策略就是，将这部分操作分配到单独的一个流水线上进行操作（我们在其它文章中，再讨论这个话题）。

* 3个操作数的指令：

    编译器偏爱三个操作数的运算，对于复杂的表达式能够有更大的优化空间。而算术/逻辑运算指令不需要存储操作，所以有足够的位表示两个源操作寄存器和一个目的寄存器。

* 32个通用寄存器：

    通用寄存器的个数是由软件需求驱动的，32个通用寄存器是现代计算机架构中常用的数量。如果使用16个寄存器并不能完全满足现代编译器的需要，而使用32个寄存器对于C编译器是完全足够的，足以覆盖最大最复杂的函数调用关系。但是，使用64个寄存器需要占用指令中更多的位去编码寄存器，也会增加上下文切换时的负荷（需要保存的寄存器更多）。

* 寄存器$0：

    寄存器$0总是返回一个0常数。0是最常用的一个常数，直接用一个寄存器表示，可以减少常数向寄存器的加载操作。

* 指令不含条件码:

    即使相比其它RISC架构，MIPS指令集也具有一个重要特性就是没有任何条件标志。许多架构使用进位、零等多个标志。像X86等CISC复杂指令集架构的指令中有一些位专门表示是否根据结果设置这些标志位。就是一些RISC指令集架构也保留了一些这样的标志位，比如说ARM，尽管通常只有比较指令可以设置这些标志位。

    MIPS架构决定使用寄存器保存这些信息：比较指令根据结果设置通用寄存器，条件分支指令检查判断这些通用寄存器。这样的操作，非常有利于流水线架构的实现，因为这样的机制，比较/分支指令不需要再依赖于算术/逻辑操作，也就是说，它们之间彼此都是独立的，流水线的实现也就更简单。它们之间的逻辑关系由软件实现，这也是MIPS架构的设计理念：强调软硬件结合，简化硬件设计。

    有效的条件分支指令要求，必须在半个时钟周期内做出是否要跳转的决定；MIPS架构通过尽可能简单地测试条件是否满足实现，比如，判断某个寄存器的值是否为符号位或者等于0，再比如，判断两个寄存器的值是否相等。

<h3 id="1.5.2">1.5.2 寻址和内存访问</h3>

* 访问内存都是先load/store到寄存器中：

    算术指令如果直接操作内存变量会破坏简化流水线设计的理念。所以，对内存变量进行操作的时候，先将其加载到寄存器中，然后再对寄存器进行算术逻辑操作。完成后，将将结果再存储到内存中对应的位置。

* 只有一种数据寻址模式，寄存器寻址：

    几乎所有的加载和存储都是通过寄存器基址加上16位的偏移实现的。

* 字节寻址：

    MIPS架构中的寄存器是一个整体，所有的操作都是对整个寄存器的操作。所以，无法实现字节或者半字这样的操作。但是，C语言之类的语法又要求可以按照字节或半字进行操作。MIPS架构采取的方式就是，提供一组load/store指令，分别加载字节、半字或WORD大小的内存变量。一旦数据加载到寄存器中，它就看作为一个寄存器长度大小的数据（比如说，32位架构就是32位整数，64位架构就被看作为64位整数）。所以，对于这些字节或半字的load操作，还需要考虑符号位。于是，又延伸出两种加载指令的形式：符号扩展或零扩展。

* load/store操作必须对齐：

    MIPS架构内存访问必须是按对齐方式进行的。字节可以是任何地址，但是半字就必须是偶数地址对齐，WORD必须是4字节对齐的方式。CISC指令集架构的微处理器可以从任意地址处读取一个4字节的数据，代价就是需要多花费一些时钟周期。

    但是，MIPS指令集一些特殊的指令，以简化未正确对齐的地址上load和store的工作。

* 跳转指令：

    指令的长度限制为32位，对于想要大范围跳转的分支指令是一个很大的问题。MIPS指令中最小的操作码域是6位，为跳转的目的地址保留了26位。因为内存中的指令代码都是4字节对齐的，也就是说，最低2位不需要保存，那么允许访问的程序范围就是2^28，等于256MB。这个地址不是相对于PC（程序计数器）的，而是被解释为256M的代码段中一个绝对地址。这样以来，对于大于256M的单个程序非常不便。虽然，可以使用寄存器保存跳转目标，然后再使用跳转指令跳转到32位地址的任何地方。

    条件分支指令只有16位的偏移量，对于4字节对齐的内存空间，其访问的范围是2^18B。但是这儿的地址可以解释为相对PC寄存器的正负范围。所以，编译器只有知道目标地址在分支指令前后128KB的范围内才能正确地编码条件分支指令。

<h3 id="1.5.3">1.5.3 MIPS没有的特性</h3>

* 没有字节或半字算术运算：

    所有的算术和逻辑操作都是基于32位完成的。操作字节或者半字要求更多的额外资源和更多的操作码，所以，一般不推荐使用。但是，如果程序中显式地使用short或者char类型的数据进行运算，支持MIPS架构的编译器必须额外地插入一些机器指令，保证结果能够像在真正的16位或8位机器上那样正确运行。

* 没有对堆栈寄存器的特定支持：

    虽然，传统意义上的MIPS汇编代码确定也会定义一个寄存器作为堆栈指针寄存器，但是，硬件上没有规定那个寄存器是特定的sp寄存器。而像ARM和X86架构有一个特定的sp寄存器。我们都知道，对于函数调用的实现，有一些约定俗成的格式，比如说`System V ABI`。有一种推荐的子程序调用时堆栈栈帧布局，这样可以混合使用汇编语言和C语言编程，使用不同的编译器选项进行编译。但是这一切和硬件都没有关系，需要人为实现。堆栈的pop操作不符合流水线的执行，因为它要写两个寄存器（来自堆栈的数据和增加的指针值）。

* 最少的子程序支持：

    跳转指令也与我们习惯上的认知有所不同：具有跳转（jump）和链接（link）跳转指令，把返回地址写入到一个固定的寄存器中。默认使用$31作为返回地址寄存器。

    这比把返回地址保存到堆栈中更为简单，而且还有许多优点。举两个例子让你对这种论断有一个直观感受：第一，它把分支指令和内存访问指令分离开来；第二，当调用不需要保存返回地址的子程序时，有助于提高效率。

* 最少的中断处理：

    很难看到比这做得更少的硬件了。它把程序重新运行的地址保存到一个特定寄存器中，修改机器状态，然后禁止中断。做完这些后，跳转到一段保存到低内存中的预定义好的程序，之后的工作完全由软件控制。

    其实，现在处理器对于中断都是基于能少则少的原则进行处理。硬件上，MIPS架构则是只保存了一个重新运行的地址，而像X86架构，还需要保存eflags、cs、eip、ss和esp等寄存器。所以，MIPS的中断处理更为简单。

* 最少的异常处理：

    异常的硬件处理其实同中断处理一样。MIPS架构把中断看作为异常的一种，MIPS的异常涵盖了CPU想要中断所有顺序的执行，调用软件处理程序所产生的所有事件。比如中断、试图访问物理地址不存在的虚拟内存或者其它事情都可以产生异常。还有比如故意植入的trap陷阱指令，像为了访问内核态程序的系统调用都是一种异常。所有的异常都导致CPU的控制权传递给一个固定的入口点。

    对于任何异常，MIPS架构的CPU不会存储任何东西到堆栈上，也不会写内存或者保存任何寄存器。一切都由你自己决定。这与ARM和X86架构都是不一样的。

    按照约定，MIPS架构也保留了2个通用寄存器，让异常程序可以自举（在MIPS架构的CPU上，不使用寄存器是无法工作的）。但是，对于一个旨在多架构上运行的、允许中断或陷阱（trap）的通用系统，这两个寄存器的值随时会发生变化，最后不要使用它们。

<h3 id="1.5.4">1.5.4 MIPS架构流水线的延迟</h3>

前面我们讨论的都是简化CPU的设计带来的一些结果。但是，为了使指令集对流水线更友好，也产生了一些奇怪的效应，想要理解它们并不容易：

* 分支延迟：

    <img src="https://raw.githubusercontent.com/tupelo-shen/my_test/master/doc/linux/mips-architecture/others/images/see_mips_run_1_3.PNG">

    如上图的流水线结构图所示，当一个jump指令在读取阶段时，又产生了新的PC寄存器值，jump指令后的指令也被启动了。MIPS架构规定，分支指令后的指令总是在分支目标指令之前执行。跟随在分支指令后的指令位置被称为`分支延迟槽`，具体物理意义有点抽象，对应上图的话，就是横向上的一格。对于分支延迟槽，如果硬件不做任何特殊处理的话，决定是否跳转以及跳转的目标地址等，这些工作就会在ALU阶段结束时才能完成，此时即使是在下下个流水线槽都来不及提供一个指令地址。

    但是分支指令的重要性足以给其特殊处理，从上图可以看出，通过特殊的处理，ALU阶段可以在半个时钟周期内就使目标地址可用。连同取指令提前的半个周期，刚好在下下个流水线槽得到分支目标地址作为指令开始执行。所以，CPU控制单元执行的顺序是，分支指令，分支延迟槽指令，然后是分支目标指令，中间没有延时。

    如何利用好这个分支延迟槽，就是编译器或者汇编程序编写者的责任了。可以适当安排位于分支延迟槽中的指令做些有用的工作。也可以把不影响执行顺序的指令安排到分支延迟槽中执行。

    对于条件分支指令，这个比较复杂，至少保证位于分支延迟槽的指令对两个分支都是无害的。如果是在没有可以安排的指令，可以添加一个nop指令。这也是我们经常在MIPS架构的汇编代码中看到的处理方式。

* 数据加载延迟（加载延时槽）：
    
    <img src="https://raw.githubusercontent.com/tupelo-shen/my_test/master/doc/linux/mips-architecture/others/images/see_mips_run_1_4.PNG">

    简化的流水线另一个结果就是，当下一条指令到达ALU阶段的时候，上一条load指令的数据才开始从cache或内存中到达。也就是说，load指令后的下一条指令还是不能使用数据的。

    那么load指令后的位置，就称为`加载延时槽`。带有优化的编译器总是尝试利用这个加载延时槽。有时候，编译器会把这个位置填充一个nop操作。最新的MIPS架构CPU上，load操作也是采用了互锁机制：如果你尝试过早使用这个数据，CPU会停止执行，等待这个数据到达。但是，在早期的CPU上，没有互锁机制，过早的使用这个数据，会产生不可预料的结果。

<div style="text-align: right"><a href="#0">回到顶部</a><a name="_label0"></a></div>