####java虚拟机运行时数据区说明

![](https://i.imgur.com/MRZD9Sb.png)

* **程序计数器** 一块很小的内存区域,当前线程所执行的字节码的行号指示器,如果执行的是native方法这个计数器是undefined,此内存区域是java虚拟机规范中没有任何OutOfMemoryError情况的区域
* **java虚拟机栈** 是线程私有的,与线程周期相同,描述的是java方法执行的内存模型,每个方法执行的同时会创建一个栈帧用来存储局部变量表,操作数栈,动态链接,方法出口等信息.每个方法从开始到执行完成的过程,都对应着一个栈帧在虚拟机栈中的入栈到出栈的过程,通常说的"栈"就是讲的虚拟机栈,或者说是虚拟机栈中的局部变量表部分,**局部变量表** 存储在编译期已知的基本类型(boolean,byte,int,long,short,float,double,char)以及对象引用(reference类型,它不等同于对象本身,可能是一个指向对象引用的地址,也可能是指向一个代表对象的句柄或者与该对象相关的位置)和returnAddress类型(指向一个字节码执行的地址).其中64位长度long,double会占用2个局部变量空间其余数据类型占用1个,局部变量所需要的内存空间在编译期间完成分配,当进入一个方法时,这个方法所需要在帧中分配多大的局部变量空间是完全确定的,在方法运行期间不会改变局部变量表的大小.在java虚拟机规范 **如果线程请求的栈的深度大于虚拟机所允许的深度将抛StackOverflowError异常,如果虚拟机栈可以动态扩展,如果在扩展时无法申请到足够的内存,就会抛OutOfMemoryError异常**(当前大部分的java虚拟机都可以动态扩展,只不过java虚拟机规范也允许固定长度的虚拟机栈)
* **本地方法栈** 与虚拟机栈作用相似,他们之间区别不过是虚拟机栈为虚拟机执行java方法服务,本地方法栈为虚拟机使用到的native方法服务(虚拟机规范中对本地方法栈所使用的语言,使用方法与数据结果没有强制规定,因此具体的虚拟机可以自由实现它,Sun HotSpot虚拟机就将本地方法栈与虚拟机栈合并成一个),与虚拟机栈一样 本地方法栈也会抛StackOverflowError及OutOfMemoryEerror
* **java堆** 是java虚拟机中所管理的内存中的最大一块,是所有线程共享的一块内存区域,在虚拟机启动时创建.该内存区域唯一作用:存放对象实例,几乎所有的对象实例都在这里分配内存.**(但随着JIT编译器的发展与逃逸分析技术的成熟,栈上分配,标量替换优化技术会导致一些微妙变化)** java堆是垃圾收集器管理的主要区域.因此也被成为GC堆.从内存回收的角度来看,由于现在收集器基本采用分带收集算法,所以java堆中还可以细分为新生代,老年代,再细一点的有Eden空间 From Survivor空间 To Survivor空间等.从内存分配角度来看,线程共享的java堆中可能划分为多个线程私有的分配缓冲区(Thread Local Allocation Buffer TLAB)
* **方法区** 与堆一样是各个线程共享的内存区域,它用于存储已经被虚拟机加载的类的信息,常量,静态变量,即时编译器编译后的代码等数据.根据虚拟机规范,当方法区无法满足内存分配需求时,将OutOfMemoryError异常
* **运行时常量区** 是**方法区的**一部分,class文件除了有类的版本,字段,方法,接口等描述信息外,还有常量池(Constant Pool Table),用来存放编译期生成各种字面量和符号引用,这部分在类加载后将在方法区的常量池中存放.java虚拟机对class文件的每一部分(包含常量池)的格式都有严格的规定

* **-XX:+/-UseTLAB**设置是否使用TLAB模式分配(每个线程在java堆中预先分配一小块内存,如果线程要分配内存就在自己所在TLAB上分配)
* **对象头** 包含两部分:一部分用于储存自身的运行时数据,如HashCode,GC分代年龄,锁状态标志,线程持有锁,偏向线程ID,偏向时间戳这部分数据的长度在32位和64位虚拟机中分别为32bit和64bit(称为Mark Word),对象需要存储的运行时的数据很多一般都超过32位或64位,因此Mark Word被设计成一个非固定的数据结构以便于在极小的空间存储尽量多的信息.另一部分是类型指针,即对象指向它的类元数据的指针,虚拟机通过这个指针来确定这个对象是那个类的实例
* **-Xms20m**最小堆内存
* **-Xmx20m** 最大堆内存 两个设置为一样即可避免堆自动扩展
* **-Xoss** 本地方法栈
* **-Xss** 设置栈的大小 由于HotSpot虚拟机将 本地方法栈及虚拟机栈合并,因此-Xoss是无效的
* **-Xmn10m** 堆里面新生代内存
* 操作系统对每个进程的内存是有限制的 window 32位系统最大分配是2G,因此减去Xmx(最大堆内存),再减去MaxPermSize(最大方法区容量),程序计数器很小可以忽略.如果虚拟机本身进程消耗的内存不计算内剩余的内存就被虚拟机栈与本地方法栈所占用,**每个线程分配的栈容量越大,可建立的线程数就越少,建立线程容易把剩余的内存消耗掉**
* **-XX:PermSize=20m** 设置方法区的最小值
* **-XX:MaxPermSize=20m** 设置方法区的最大值
* **-XX:MaxNewSize=2000** 新生代可分配内存最大值
* **-XX:NewSize=**  新生代初始化内存大小
* 在1.6以及之前的版本常量池分配在永久代内,因此可以通过上述两个设置来间接限制常量池的容量
* **-XX:MaxDirectMemorySize=100m** 指定DirectByteBuffer所能分配的空间,如果未设置 会被jvm默认为最大堆内存,一般在NIO中等使用
* **-XX:+PrintGCDetails** 收集GC日志
* **-verbose:gc** 收集GC日志 与上面类似
* **-XX:PretenureSizeThreshold** 令大于这个设置值的对象直接在老年代分配.这样做的目的避免在Eden区以及两个Survivor区之间发生大量的内存复制 **该值只对Serial与ParNew两种收集器有效** Parallel Scavenge收集器不认识该参数
* **-XX:SurvivorRatio=8** 堆中Eden与Survivor比例值
* **-XX:+PrintGCApplicationStoppedTime** 打印GC手机停顿时间
* **-XX:+PrintGCDateStamps** 打印GC收集时间
* **-Xloggc:gclog.log** 写入gc日志到文件中
* **-XX:+PrintReferenceGC** 用来跟踪系统内的(softReference)软引用，(weadReference)弱引用,(phantomReference)虚引用，显示引用过程
* **-XX:+DisableExplicitGC** 屏蔽掉显式调用System.gc();因为该操作大部分情况下会触发Full GC
* **-XX:+UseConcMarkSweepGC** 启用CMS收集器,新生代使用ParNew收集器,老年代使用CMS收集器 **默认关闭**
* **-XX:+UseParNewGC** 启用ParNew收集器,该收集器是在CMS启用后 默认新生代收集器(在启用CMS后可不用填写该参数) 老年代使用串行收集器
* **-XX:+UseSerialGC** 启用串行垃圾收集器 老年代新生代都是用 **默认关闭**
* **-XX:+UseParallelOldGC** 为老年代与新生代都使用并行清除的垃圾收集器 **默认关闭**
* **-XX:+UseParallelGC** 为新生代启用并行清除,老年代使用单线程Mark-Sweep-Compact的垃圾收集器
* **-XX:CompileThreshold=10000** 设置方法被调用多少次后会通知即时编译器提交一个该方法的代码编译请求![](https://i.imgur.com/RjrqtOo.png)
* **-XX:-UseCounterDecay** 不设置CompileThreshold,方法计数统计并不是方法被调用的绝对次数,而是一个相对的执行频率,即一段时间内方法被调用的次数,当超过一定的时间限度,如果方法调用次数仍然不足以让它提交给即时编译器编译,那么这个方法的调用计数器就会减半这个过程称之为方法计数器热度的衰减(CounterDecay),而这段时间就称之为此方法统计的半衰减周期(Counter Half Life Time).可以通过该参数来关闭热度衰减,让方法计数器统计方法被调用的绝对次数,这样,只要系统运行时间足够长,几乎所有的代码都会编译为本地代码.
* **-XX:CounterHalfLifeTime** 设置半衰减周期的时间 单位为秒
* **-XX:MetaspaceSize=256m** 指Metaspace扩容时触发FullGC的初始化阀值,也是最小阀值
* 
![](https://i.imgur.com/sUrYLSP.png)
* **-XX:-BackgroundCompilation**,如果上述编译达到条件,代码编译器未编译完成之前,都仍按照解释方式继续执行,而编译动作会在后台的编译线程中执行,设置该参数后将禁用后台编译.即一旦达到JIT编译条件,执行线程会向虚拟机提交编译请求后将会一直等待,直到编译完成后,开始执行编译器输出的本地代码
* **-XX:+PrintCompilation** 开启虚拟机在即时编译时,将编译为本地代码的方法打印出来 "%"为回边计数器触发OSR的编译
* **-XX:+PrintInlining** 输出方法内联信息 貌似需要开启 **-XX:+UnlockDiagnosticVMOptions** 参数才行
* **XX:CMSWaitDuration=2000** JVM中一个线程定时扫描old区,扫描间隔时间2s
* **XX:CMSInitiatingOccupancyFraction=75** (老年代使用率阀值)如果发现old区占比超过75%(CMS下默认是68%)会触发cms gc
####堆详解

* **Minor GC** 新生代GC 比较频繁
* **Major GC(Full GC) 老年代GC 出现Major GC时至少进行一次Minor GC(非绝对 Parallel Scavenge收集器会忽略),一般Major GC比Minor GC满10倍以上
* 在绝大多数情况下对象会在新生代Eden区中分配,当Eden区没有足够的空间进行分配时,虚拟机将发起一次Minor GC
* 虚拟机给每个对象定义了一个"age"计数器,如果对象生成在Eden区,经过Minor GC后扔存活,并且Survivor区能够容纳下,将会被移入Survivor区并且年龄加1,每经过一次Minor GC 年龄都会增加一岁,当增加到一定年龄后会被移入老年代(默认阀值是15,可通过设置**-XX:MaxTenuringThreshold=18**设置阀值)
* **PerNew** 收集器采用的是复制算法,该算法建立在绝大部分对象都是朝闻夕死的特性上,如果对象过多把这些对象复制到Survivor上 GC时间会比较长,而且一般空间比例Eden:Survivor为8:1
####JDK监控与故障处理工具

* **jps**  JVM Process Status 显示系统内所有的HotSpot虚拟机进程,参数说明:

  1. **-q** 只输出LVMID(进程的本地虚拟机唯一ID:Local Virtual Machine Identifier)省略主类的名称
  2. **-m** 输出虚拟机进程启动过时传递给主类main()函数的参数 
  3. **-l** 输出主类的全名,如果是jar的输出jar路径
  4. **-v** 输出虚拟机进程启动时JVM的参数
* **jstat** JVM Statistics Monitoring Tool用于收集HotSpot虚拟机各方面的运行数据,它可以显示本地或者远程虚拟机进程中的类装载,内存,垃圾收集,JIT编译等运行数据,在没有GUI图形界面的服务器环境上,该工具是运行期定位虚拟机性能问题的首选工具.参数格式 jstat [option vmid [interval[s|ms] [count]] ] **注意** 如果是本地虚拟机则vmid与LVMID一致,如果是远程虚拟机进程那么VMID格式应当**[protocol:][//]lvmid[@hostname[:port]]/servername** interval是查询间隔,count是查询次数 例如每250毫秒查询一次2764的垃圾回收情况 一共查询20次则:jstat -gc 2764 250 20 省略这两个参数则默认只查一次,option参数如下:
  1. **-class** 监控类装载,卸载数量,总空间及类装载所耗费的时间
  2. **-gc** 监控java堆状况,包括Eden区 两个Survivor区,老年代永久代的容量,已用空间,GC时间合计等信息
  3. **-gccapacity** 监控内容与-gc基本相同,但输出主要关注java堆各个区域使用到的最大,最小空间
  4. **-gcutil** 监控内容与-gc基本相同,但输出主要关注已使用空间占总空间的的百分比
  5. **-gccause** 与gcutil基本一样,但会额外增加输出导致上次GC产生的原因
  6. **-gcnew** 监控新生代GC状况
  7. **-gcnewcapacity** 与gcnew基本相同,输出主要关注使用到的最大,最小空间
  8. **-gcold** 监控老年代GC状况
  9. **-gcoldcapacity** 与gcold基本一样,输出主要关注使用到的最,最小空间
  10. **-gcpermcapacity** 输出永久代使用的最大,最小空间
  11. **-compiler** 输出JIT编译器编译过的方法,耗时等信息
  12. **-printcompiklation** 输出已经被JIT编译的方法
  
   总结:关于-gc 参数说明 

	* S0 S1代表新生代Survivor1 Survivor2区已使用的的百分比 
	* E 代表新生代Eden区 
	* O 代表老年代
	* P 永久代(Permanent) 
	* YGC 程序启动后发生的Minor GC次数 
	* YGCT Minor GC总耗时时间 
	* FGC 程序启动后总 GC次数 
	* FGCT Full GC总耗时时间 
	* GCT 所有GC总耗时时间
	* MC  Metaspace Capacity 
	* MU  Metaspace Used
  
* **jinfo** Configuration Info for Java 显示虚拟机配置信息 jinfo [option] pid

  * ** 查询虚拟机参数 jinfo -flag CMSInitiatingOccupancyFranction 144

* **jmap** Memory Map for Java生成虚拟机内存转储快照(heapdump文件)
* 
* **jhat** JVM Heap Dump Browser 用于分析heapdump文件,它会建立一个Http服务器,让用户在浏览器上查看分析结果
* **jstack** Stack Trace for Java 显示虚拟机的线程快照
* jmap -dump:format=b,file=*.dump [pid] 导出dump文件
* jmap -dump:format=b,file=*.hprof [pid]
* ps -p pid -o etime 查看JVM运行的总时间

####java类文件属性

* 头四个字节代表class属性文件,(十六进制0xCAFEBABE)称为魔数,用来辨别文件的类型(因为扩展名可以更改不准确)
* 5-6字节代表次版本号
* 7-8字节代表主版本号
* 9字节常量池数量 常量池主要存放两大类:字面量Literal与符号引用(Symbolic Reference),字面量比较接近java语言层面的常量概念,如文本字符串,声明为final的常量值,而符号引用属于编译原理方面的概念,包括一下三类常量:
   * 类与接口的全限定名
   * 字段的名称与描述符
   * 方法的名称与描述符

在Java语言中，可作为GC Roots的对象包括下面几种：
a) 虚拟机栈中引用的对象（栈帧中的本地变量表）；
b) 方法区中类静态属性引用的对象；
c) 方法区中常量引用的对象；
d) 本地方法栈中JNI（Native方法）引用的对象;



#### 极客时间JVM笔记

##### 	第一章

  * 虚方法 

    子类重写父类的方法会被`Override`标示,当子类重写的方法被调用时JVM使用`invokevirtual`字节码指令来调用方法,简单来说所有可以被重写的方法都是虚方法.(`static`、`final`、`private`)除外

* 虚拟机运行字节码方式

  1. **解释执行**.条将字节码翻译成机器码并执行

  2. **即时编译**(Just-In-Time compilation)JIT.将一个方法中包含的所有字节码编译成机器码后再执行

* 编译器种类

  1. **C1**.又称为Client编译器,优化手段简单,编译时间短,效率一般

  2. **C2**.又称为Server编译器,优化手段复杂,编译时间长,代码执行效率高

  3. **Graal**.java10引入的实验性即时编译器

##### 第三章

* 加载器
  1. 启动类加载器.负责加载**Jre/lib**下的jar包中类及-Xbootclasspath指定的类
  2. 扩展类加载器.父加载器为启动类加载器.负责加载相对次要却又通用的类,`jre/lib/ext`目录下的jar包中的类及系统变量`java.ext.dirs`指定的类
  3. 应用类加载器.父类加载器为扩展类加载器,负责加载应用程序路径下的类 `-cp/ -classpath`  系统变量`java.class.path` 环境变量`CLASS_PATH`
* 类初始化触发条件
  1. 当虚拟机启动时，初始化用户指定的主类
  2. 当遇到用以新建目标类实例的 new 指令时，初始化 new 指令的目标类
  3. 当遇到调用静态方法的指令时，初始化该静态方法所在的类
  4. 当遇到访问静态字段的指令时，初始化该静态字段所在的类
  5. 子类的初始化会触发父类的初始化
  6. 如果一个接口定义了 default 方法，那么直接实现或者间接实现该接口的类的初始化，会触发该接口的初始化
  7. 使用反射 API 对某个类进行反射调用时，初始化这个类
  8. 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的类

##### 第四章

  * 字节码调用指令
        		1. **invokestatic** 调用静态方法
        		2. **invokespecial** 调用私有方法,构造方法,super方法或者构造方法,所有实现接口的默认方法
        		3. **invokevirtual** 调用非私有方法
        		4. **invokeinterface** 调用接口方法
        		5. **invokedynamic** 调用动态方法(主要是重写的方法)
* **动态绑定** 虚拟机根据调用者的动态类型,来确定虚方法调用的目标方法称为动态绑定.**invokedynamic**指令
* **静态绑定** 调用静态方法的**invokestatic**指令,用于构造方法,私有方法及超类私有方法调用的**invokespecial**指令,标示了**final**的虚方法

##### 字节码指令集

* 操作数栈(operand stack),用于运算
* 本地变量表(local variable table),又称为局部变量表
* 局部变量表->操作数栈
  * **iload index** 将指定的int值变量推入栈顶,index为下标
  * **iload_x** 将第x个int型的本地变量推入栈顶 x [0,3]
  * **aload_x** 将指定的**引用类型**本地变量表推入栈顶,index为下标
  * **aload_x** 将第x的**引用类型**本地变量推入栈顶 x [0,3]
* 操作数栈->局部变量表
  * **istore index**  将栈顶元素int的值存入本地变量表,index为下标,栈顶出栈 
  * **istore_x** 将栈顶int类型的数值存入第x个本地变量 x [0,3]
  * **astore index** 将栈顶**引用类型**存入指定本地变量,index为下标,栈顶出栈 
  * **astore_x** 将栈顶**引用类型**存入第x个本地变量 x [0,3]

* 常量->操作数栈

  * **bipush** 将单字节的常量[-128~127]推入栈顶
  * **sipush** 将一个短类型常量[-32768~32767]推入栈顶
  * **ldc** 将int,float,string型常量从常量池中推入栈顶
  * **ldc_w** 将int,float,string型常量从常量池中推入栈顶(宽引用)
  * **aconst_null** 将null推入到栈顶
  * **iconst_m1** 将int型-1推入栈顶
  * **iconst_x** 将int型x推入栈顶 x [0,5]

* 数组->操作数栈

  * **iaload index** 将int型数组下标为index的值推入栈顶
  * **aaload index** 将引用类型数组下标为index的值推入栈顶
  * **saload index** 将short型数组下标为index的值推入栈顶

* 操作数栈->数组

  * **iastore index** 将栈顶int值存入数组的index索引位置
  * **aastore index** 将栈顶引用类型值存入数组的index索引位置
  * **bastore index** 将栈顶boolean值存入数组的index索引位置

* 操作数栈其他命令

  * **pop** 弹出栈顶值(值不能为long或double)

  * **pop2** 弹出栈顶的两个值(或一个双精度值[double,long])

  * **dup** 复制栈顶元素并将复制的值推入栈顶(值不能为double或long) value → value,value

  * **dup_x1** 复制栈顶元素并将复制的值推入栈顶指定位置 dup_x1插入位置：1+1=2，即栈顶2个Slot下面. 

    > 例如:(栈尾)value2, value1(栈顶) → value1, value2, value1  (value1和value2不能为double或long)

  * **dup_x2** 复制栈顶元素并将复制的值推入栈顶指定位置 dup_x2插入位置：1+2=3，即栈顶3个Slot下面  

    > 例如 (栈尾)value3,value2,value1(栈顶)  → value1,value3,value2,value1 (value1不能为double或long),
    >
    > 且如果value2如果为双精度则会占用value3的位置 即 value3,value1,value2,value1

  * **dup2** 复制栈顶一个(double或long)或两个元素并将复制的值推入栈顶

    > 例如 {value2, value1}(栈顶)→ {value2,value1}, {value2,value1}

  * **dup2_x1** 复制栈顶元素并将复制的值推入栈顶指定位置  dup2_x1插入位置：2+1=3，即栈顶3个Slot下面

    > value3, {value2,value1} (栈顶)→{value2, value1},value3, {value2,value1}

  * **dup2_x2** 复制栈顶元素并将复制的值推入栈顶指定位置  dup2_x1插入位置：2+2=3，即栈顶4个Slot下面

    > 例如 {value4, value3},{value2, value1}→ {value2,value1}, {value4,value3}, {value2,value1}

  * **swap** 交换栈顶的两个Slot数值的位置,注意java虚拟机没有提供long或double交换的指令



* 运算相关
  * 加 **iadd ladd fadd dadd**
  * 减 **is fs ls ds**
  * 乘 **imul lmul fmul dmul**
  * 除 **idiv ldiv fdiv ddiv**
  * 余 **irem lrem frem drem**
  * 取负 **ineg lneg fneg dneg**
  * 位移 **ishl ishr lshr lshl**
  * 按位或 **ior lor**
  * 按位与 **iand land**
  * 按位异或 **ixor lxor**