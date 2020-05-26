### Java基础

#### 1.java中==和equals和hashCode的区别

[链接](https://blog.csdn.net/justloveyou_/article/details/52464440)

1. == 该生产的是布尔结果，它计算的是操作数的值之间的关系。

   若操作数的类型是基本数据类型，那比较的值是否相等

   若操作数的类型是引用数据类型，则关系操作符比较的是两边的内存地址是否相同。

2. equals  Object的实例方法，比较两个对象的内容是否相同

   如String内部会重写Object的equals方法，主要分三个步骤

   - 比较引用是否相同（是否为同一对象）
   - 判断类型是否一致
   - 比较内容是否一致

3. hashCode Object 的native方法，获取对象的哈希值，用于确定对象在哈希表中的索引值，int整型。

#### 2. int、char、long各占多少字节数

1. 字节byte 位bit，int 4个字节32位 char 2个字节 16位，long 8个字节64位

#### 3.int与integer的区别

[链接](https://blog.csdn.net/chenliguan/article/details/53888018)

##### 基本使用对比

1. Integer是int的包装类型；int是基本数据类型
2. Integer变量实例化之后才能使用，int变量不需要。
3. Integer实际是对象的引用，指向此new的Interger对象，int直接储存数据值。
4. Integer的默认值是null，int的默认值是0。

##### 深入使用对比

1. new生成的Integer对象比较是false
2. Integer变量与int变量比较，只要变量值是相等的，结果为true。
3. 非new生成的Integer变量与new生成的Integer的变量是false，非new生成的变量指向常量池。内存引用地址不一样。
4. 两个非new生成的Integer的对象比较，如果两个值在-128-127之间，结果为true。

#### 5.谈谈对java多态的理解

[链接](https://blog.csdn.net/u013546115/article/details/78632492)

多态：指程序定义期间不知道具体的引用类型，只有在程序运行期间才确定引用变量指向哪一个类的实例对象。

#### 6.String、StringBuffer、StringBuilder区别

[链接](https://blog.csdn.net/rmn190/article/details/1492013)

##### 线程安全

StringBuffer:线程安全

StringBuilder：非线程安全

效率：

String：String是不可变的对象，每次改变Sting的类型其实就是在内存中新生成了一个对象，经常改变类型的不要用String，

StringBuffer：每次操作是对自己本身进行操作，对于经常改变类型的字符串，用该类，在多个线程中可以保证安全。

StringBuilder：与StringBUffer类似，但StringBuilder是非线程安全的，所以在单线程中操作时，可以选择该类。

#### 7.什么是内部类？内部类的作用

[链接](https://blog.csdn.net/mid120/article/details/53644539)

定义：出现在类内部的类就是内部类。

1. 实现隐藏
2. 共享外部类所有元素
3. 实现多重继承
4. 避免实现两个不同类的同名方法的调用。

#### 8.抽象类和接口区别

[链接](https://www.jianshu.com/p/038f0b356e9a)

#### 9.抽象类的意义

[链接](https://www.jianshu.com/p/4ed470d0cf38)

#### 10.抽象类与接口的应用场景

[链接](https://blog.csdn.net/xcbeyond/article/details/7667733)

#### 11.泛型中extends和super的区别

[链接](https://itimetraveler.github.io/2016/12/27/%E3%80%90Java%E3%80%91%E6%B3%9B%E5%9E%8B%E4%B8%AD%20extends%20%E5%92%8C%20super%20%E7%9A%84%E5%8C%BA%E5%88%AB%EF%BC%9F/)

1. <? extends T>表示上界通配符

   适用频繁往外读取

2. <? superT>表示下界通配符

   频繁往内插入

#### 12.父类的静态方法能不能被子类重写

不能。

静态方法在程序运行时已经分配好内存，所有引用到该对象的方法都指向的是同一块内存地址。子类若定义了同一个静态方法，其实是在内存中重新创建了一块内存地址。

#### 13.进程和线程的区别

[链接](https://blog.csdn.net/kuangsonghan/article/details/80674777)

#### 14.final，finally，finalize的区别

final：修饰变量方法，类。当final修饰父类的private方法时，子类不能集成该方法，但是可以重新定义该方法

finally：1. try finally语句中，只有try语句执行时，才会执行finally，另外finally会撤销之前的return。









