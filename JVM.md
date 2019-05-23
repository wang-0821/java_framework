## JVM Notes
* [1.About JVM](#1)
* [2.JVM Runtime Data Area](#2)
* [JVM Classloading Mechanism](#)
* [ Loading](#)
* [ Verification](#)
* [ Preparation](#)
* [ Resolution](#)
* [ Initialization](#)
* [ClassLoader](#)

<h2 id = "1">1.About JVM</h2>
&emsp;&emsp; Java程序设计语言、Java虚拟机、Java API类库统称为JDK，JDK是支持Java程序开发的最小环境。Java API中的Java SE API子集和Java虚拟机称为JRE。
Java SE：支持面向桌面级运用的Java平台，提供了完整的Java核心API。Java EE: 支持使用多层架构的企业运用的Java平台。

<h3 id = "2">2.JVM Runtime Data Area</h3>
&emsp;&emsp; JVM 运行时数据区划分为：方法区、虚拟机栈、本地方法栈、堆、程序计数器。
<br>
&emsp;&emsp; 程序计数器可看作当前线程执行的字节码的行号指示器。Jvm多线程是通过线程轮流切换分配处理器时间实现的，所以，每条线程都需要有一个程序计数器。
如果执行的是Native方法，那么这个计数器的值为空，是Jvm唯一没有规定任何OutOfMemoryError的区域。
<br>
&emsp;&emsp; Java虚拟机栈也是线程私有的，每个方法执行的同时会创建一个栈帧(Stack Frame)，用以存储局部变量表、操作数栈、动态链接、方法出口等。
一个方法的执行就对应着一个栈帧的入栈出栈。这个区域有两种异常：如果线程请求的栈深度大于虚拟机栈的深度，那么将抛出StackOverflowError。
如果虚拟机栈可以动态扩展，扩展时没有申请到足够的内存，那么就会抛出OutOfMemoryError。本地方法栈同虚拟机栈，不过本地方法栈服务于本地的Native方法。
<br>
&emsp;&emsp; Java堆是被所有线程共享的内存区域，虚拟机启动时创建，存放的都是对象实例。是垃圾收集器管理的主要区域。Java堆分为：新生代、老年代。
细分为：Eden区、From Survivor空间、To Survivor空间。
<br>
&emsp;&emsp; 方法区是被所有线程共享的内存区域，存储被已虚拟机加载的类信息、常量、静态变量、编译后的代码等数据。方法区被称为永久代，但本质两者不等同。
这一区域的回收主要是针对常量池的回收和对类型的卸载。当方法区无法满足内存分配需求时，抛出OutOfMemoryError异常。
<br>
&emsp;&emsp; 运行时常量池，是方法区的一部分，用以存放编译期生成的各种字面量和符号引用，类加载后进入方法区的运行时常量池中存放。
<br>
&emsp;&emsp; 直接内存，不是java虚拟机运行时数据区的一部分，使用Native函数直接分配堆外内存，然后通过一个Java堆中的DirectByteBuffer对象，
操作这块内存，避免了在Java堆和Native堆中来回复制数据。可能导致OutOfMemoryError异常。

<h2 id = "">JVM Classloading Mechanism</h2>
&emsp;&emsp; 类的整个生命周期包括7个阶段：加载(Loading)、验证(Verification)、准备(Preparation)、解析(Resolution)、
初始化(Initialization)、使用(Using)、卸载(Unloading)。其中验证(Verification)、准备(Preparation)、解析(Resolution)，
3个阶段统称为连接(Linking)。
<br>
&emsp;&emsp; 其中加载(Loading)、验证(Verification)、准备(Preparation)、初始化(Initialization)、卸载(Unloading)，
这5个阶段顺序是确定的，因为可能采用动态绑定。遇到new、getstatic、publicstatic、invokestatic，如果类没有初始化，那么要先触发初始化。
场景为：使用new实例化对象、读取或设置一个类的静态字段（被final修饰，在编译期已经把结果放入常量池的静态字段除外）、调用一个类的静态方法、
使用java.lang.reflect包的方法对类进行反射调用的时候、当初始化一个类的时候，其父类还没有初始化、虚拟机启动时，
会先初始化主类（包含main()的类）。这些行为被称作对一个类的主动引用。
<br>
&emsp;&emsp; 接口也会初始化，接口中不能使用静态语句块，但仍会有<clinit>()类构造器，接口在初始化时，并不要求父接口全部都完成了初始化，
只有在真正使用到父接口是，才会初始化。

<h3 id = ""> Loading</h3>
&emsp;&emsp; 加载阶段完成以下3件事：1.通过一个类的全限定名来获取定义该类的二进制字节流。
2.将字节流所代表的静态存储结构转化为方法区的运行时数据结构。3.在内存中生成一个代表这个类的Class对象，作为方法区这个类的各种数据入口。

<h3 id = ""> Verification</h3>
&emsp;&emsp; 验证是连接阶段的第一步，为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，不会危害虚拟机自身的安全。
会完成以下4个阶段的检验动作：文件格式验证、元数据验证、字节码验证、符号引用验证。
<br>
&emsp;&emsp; 文件格式验证是基于二进制字节流进行的，只有通过这个阶段的验证后，字节流才会进入内存的方法区中进行存储。后面的阶段都是基于方法区中的存储结构进行的，
不会再直接操作字节流。
<br>
&emsp;&emsp; 元数据验证，主要对类的元数据信息进行校验，确保不存在不符合Java语言规范的元数据信息。
<br>
&emsp;&emsp; 字节码验证，通过数据流和控制流分析，确保程序语义是合法的、符合逻辑的。将对类的方法体进行校验，保证不会作出危害虚拟机的事件。
<br>
&emsp;&emsp; 符号引用验证，判断引用的类、方法、字段等的引用合法性，发生在虚拟机将符号引用转化为直接引用时，在解析阶段发生。

<h3 id = ""> Preparation</h3>
&emsp;&emsp; 准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量使用的内存都在方法区中进行分配。
例如：public static int value = 123; 这个类变量在准备阶段之后，值为0，而不是123。因为这时候尚未开始执行任何Java方法，
真正赋值是在程序被编译后，执行类构造器<clinit>()方法时完成赋值。赋值动作将会在初始化阶段才会执行。
如果是常量，那么在准备阶段会被初始化为指定的值，如：public static final int value = 123; 此时准备阶段，会赋真值。

<h3 id = ""> Resolution</h3>
&emsp;&emsp; 解析阶段是将虚拟机常量池中的符号引用替换为直接引用的过程。符号引用只要能无歧义的定位到目标即可，引用的目标不一定已经加载到内存中。
直接引用可以是指向目标的指针、相对偏移量或一个能间接定位到目标的句柄。如果有了直接引用，那么引用的目标必定已经在内存中存在。

<h3 id = ""> Initialization</h3>
&emsp;&emsp; 初始化是类加载的最后一步，初始化阶段是执行类构造器<clinit>()方法的过程。
<clinit>() 方法是由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生的，静态语句块中只能访问到定义在静态语句块之前的变量，
定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。<clinit>() 方法与类的构造函数不同，不需要显式的调用父类构造器，
虚拟机会保证在子类<clinit>()方法执行前，父类的<clinit>()方法已经执行完毕。

<h2 id = "">ClassLoader</h2>
通过一个类的全限定名来获取描述此类的二进制字节流，这个动作被放到JVM外部实现，实现这个动作的代码模块称为类加载器。类加载器采用双亲委派模型，
如果一个类记载器收到了类加载的请求，首先不会自己去尝试加载这个类，而是把这个请求委派给父类来做，父加载器无法加载时，才会尝试自己去加载。