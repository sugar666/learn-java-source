# java基础

## 基础
#### equals、== 和 hashCode 的区别是什么
- `==` 比较的不是值，比较的是地址引用
- `equals`比较的是hashCode(),`set`中比较的也是hashCode，equals相等，hashCode一定相等，hashCode相等，equals不一定相等


#### 问：列出一些你常见的运行时异常？
- ArithmeticException（算术异常）
- ClassCastException （类转换异常）
- IllegalArgumentException （非法参数异常）
- IndexOutOfBoundsException （下标越界异常）
- NullPointerException （空指针异常）
- SecurityException （安全异常） -- 比如在`java.lang`下自定义不存在的类，处于安全保护，就会报错

#### final有哪些用法
* 被final修饰的类不可以被继承
* 被final修饰的方法不可以被重写
* 被final修饰的变量不可以被改变.如果修饰引用,那么表示引用不可变,引用指向的内容可变.
*== 被final修饰的方法,JVM会尝试将其内联,以提高运行效率==
* 被final修饰的常量,==在编译阶段==会存入常量池中（和JVM联系起来）

#### static都有哪些用法?
* 静态变量
* 静态方法
* 静态块，多用于初始化
* 静态内部类.
* 静态导向，即`import static.import static`是在JDK 1.5之后引入的新特性,可以用来指定导入某个类中的静态资源,并且不需要使用类名.资源名,可以直接使用资源名,比如:

```java
import static java.lang.Math.sin;

public class Test{
    public static void main(String[] args){
        //System.out.println(Math.sin(20));传统做法
        System.out.println(sin(20)); // 静态导包
    }
}
```

#### error和exception有什么区别?
error通常表示恢复不是不可能但很困难的情况下的一种严重问题。比如说内存溢出、不可能指望程序能处理这样的情况；

exception通常表示一种设计或实现问题。也就是说，它表示如果程序运行正常，从不会发生的情况；

#### String对象的intern()是指什么?
intern()方法会首先从常量池中查找是否存在该常量值,如果常量池中不存在则现在常量池中创建,

如果已经存在则直接返回. 比如 

```java
String s1="aa"; 
String s2=s1.intern();
System.out.print(s1==s2);//返回true
```

#### final、finally、finalize的区别是什么？

- final 用于声明属性，方法和类，分别表示属性不可变，方法不可覆盖，类不可继承。
- finally是异常处理语句结构的一部分，表示总是执行。
- finalize是Object类的一个方法，在垃圾收集器执行的时候会调用被回收对象的此方法，可以覆盖此方法提供垃圾收集时的其他资源回收，例如关闭文件等。


## 面向对象
#### java中有没有`goto`关键字，怎么跳出多重循环
goto 是 Java 中的保留字，在目前版本的 Java 中没有使用

- break
- 设置标签
- 抛异常
- 改变flag

break + 标签。在最外层循环前加一个标签如 label，然后在最里层的循环使用用 break label。

```java
public static void main(String[] args) {
        label:    //标记
        for (int i = 0 ; i < 10; i++) {
            for (int j = 0; j < 10; j++) {
                    System.out.println("i = " + i + ", j = " + j);
                if(j == 5) {  //满中一定条件跳到某个标记
                    break label;
                }
            }
        }
    }
```
通过捕获异常。

```java
public static void main(String[] args) {
    try {
        for (int i = 0; i < 10; i++) {
            for (int j = 0; j < 10; j++) {
                System.out.println("i = " + i + ", j = " + j);
                if (j == 5) {// 满足一定条件抛异常
                    throw new RuntimeException("test exception for j = 5");
                }
            }
        }
    } catch (RuntimeException e) { //循环外层捕获异常
        e.printStackTrace();
    }
}
```
通过标置变量。

```java
public static void main(String[] args) {
    boolean flag = false; //初始化标置变量
    for (int i = 0; i < 10; i++) {
        for (int j = 0; j < 10; j++) {
            System.out.println("i = " + i + ", j = " + j);
            if (j == 5) {   //满足一定条件进行设置标置变量
                flag = true;
            }
            if (flag) { //内层循环判断标置变量
                break;
            }

        }
        if (flag) {//外层循环判断标置变量
            break;
        }
    }
}
```

==goto 会破坏程序的可读性，java 中没有大范围使用 goto 反而是好事，而且还应该避免使用带标签的 break 语句。==

#### switch语句能否作用在byte上,能否作用在long上,能否作用在String上
 * 基本类型的包装类（如：Character、Byte、Short、Integer）
 * switch可作用于char byte short int
 * switch可作用于char byte short int对应的包装类
 * **switch不可作用于long double float boolean，包括他们的包装类**
 * **switch中可以是字符串类型,String(`jdk1.7`之后才可以作用在string上)**
 * switch中可以是枚举类型

#### 你一般在哪种情况下使用抽象类，哪种情况下使用接口
（我觉得接口与抽象类最大的不同是设计思想的不同，接口是对行为的抽象，抽象类是对类/对象的抽象）
- ==抽象类和接口所反映出的设计理念不同==。抽象类表示的是对象 / 类的抽象，接口表示的是对行为的抽象。
    - 抽象类和接口的差异点很多，比如说子类实现二者的关键字不同（extends 和 implements）、抽象类不可以多重继承而接口可以，但是我觉得二者最重要的差异还是设计思想上的不同，抽象类表示的是一种对象的抽象，接口是 xxxx”。这表示候选人对接口和抽象类是有过思考，有自己的理解的。
- 抽象类不可以多重继承，接口可以多重继承。即一个类只能继承一个抽象类，却可以继承多个接口。
- 接口在 java8 中引入了一个重要变化**，就是可以在接口中实现方法**，这表示候选人是有跟踪学习比较新的技术，软实力中的主动性和学习能力都不错。

   
        如果一个类继承了某个抽象类，则子类必定是抽象类的种类，
        而接口实现则是有没有、具备不具备的关系，比如鸟是否能飞（或者是否具备飞行这个特点），能飞行则可以实现这个接口，不能飞行就不实现这个接口。

[接口与抽象类的比较](https://www.cnblogs.com/dolphin0520/p/3811437.html)

#### 面向对象的特点
1. 抽象：抽象就是忽略一个主题中与当前目标无关的那些方面，以便更充分地注意与当前目标有关的方面。抽象并不打算了解全部问题，而只是选择其中的一部分，暂时不用部分细节。抽象包括两个方面，一是过程抽象，二是数据抽象。 
2. 继承：
3. 封装：封装是把过程和数据包围起来，只提供需要访问的接口
4. 多态：
    1. 接口实现
    2. 继承父类重写方法
    3. 同一类中进行方法重载
    
-------

        面向过程 和 面向对象 的基本概念
        
        1、面向过程：---侧重于怎么做？
        
        1.把完成某一个需求的 所有步骤 从头到尾 逐步实现
        2.根据开发要求，将某些功能独立的代码封装成一个又一个函数
        3.最后完成的代码，就是顺序的调用不同的函数
    
        特点：
        1.注重步骤与过程，不注重职责分工
        2.如果需求复杂，代码会变得很复杂
        3.开发复杂项目，没有固定的套路，开发难度很大
        
        
        面向对象：--谁来做?
        相比较函数，面向对象是更大的封装，根据职责在一个对象中封装多个方法
        1.在完成某一个需求前，首先确定职责--要做的事(方法)
        2.根据职责确定不同的对象，在对象内部封装不同的方法(多个)即我们所说的函数。
        3.最后完成代码，就是顺序的让不同的对象调用不同的方法
        特点：
        1.注重对象和职责，不同的对象承担不同的职责
        2.更加适合对复杂的需求变化，是专门应对复杂项目的开发，提供的固定套路
        3.需要在面向过程的基础上，再学习一些面向对象的语法


#### 使用final关键字修饰一个变量时，是引用不能变，还是引用的对象不能变?
- 指的是引用不能变，引用指向的对象的值还是可以改变的

#### 什么是不可变对象？
不可变对象指对象一旦被创建，状态就不能再改变。任何修改都会创建一个新的对象，如 String、Integer及其它包装类。

#### 深拷贝与浅拷贝
在浅拷贝中，基本数据类型是重新的去创建内存，引用类型只是进行引用传递，浅拷贝中的数据中的引用类型改变后，会导致所有的数据都变化
一般使用的`Object A = B`

==深拷贝需要去覆写clone方法，对每一个引用类型都是去使用clone方法进行复制，这样才算完成了深拷贝==

==在覆写clone方法的时候，需要去继承Cloneable接口==

```java
public class Student implements Cloneable {

    //引用类型
    private Subject subject;
    //基础数据类型
    private String name;
    private int age;

    public Subject getSubject() {
        return subject;
    }

    public void setSubject(Subject subject) {
        this.subject = subject;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    /**
     *  重写clone()方法
     * @return
     */
    @Override
    public Object clone() {
        //深拷贝
        try {
            // 直接调用父类的clone()方法
            Student student = (Student) super.clone();
            student.subject = (Subject) subject.clone();
            return student;
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }

    @Override
    public String toString() {
        return "[Student: " + this.hashCode() + ",subject:" + subject + ",name:" + name + ",age:" + age + "]";
    }
}
```

[java中的浅拷贝与深拷贝](https://www.jianshu.com/p/94dbef2de298)

==序列化是深拷贝==

#### 创建对象的方式
- 使用new关键字，涉及到对象创建的步骤，对象中包含的内容（JVM）的相关的知识
    - 1. 初始化类加载器
    - 2. 分配内存空间
        - 规整：指针碰撞
        - 不规整：维护一个内存列表
    - 3. 进行并发的处理
    - 4. 默认初始化
    - 5. 创建对象头
    - 6. 进行init初始化
    - 对象中的信息
        - 对象头
            - 运行时元数据
                - hashCode值
                - GC的年龄
                - 锁的状态
            - 类型指针（确定该对象所属的类型，指向方法区中的对象信息）
        - 实例数据
        - 对齐填充
- Class中的newInstance()方法，默认调用空参的构造器。
- Constructor中的newInstance()方法，可以调用有参的，也可以调用空参的
- 使用clone()，设计到浅拷贝与深拷贝，需要继承Cloneable()接口
- 使用反序列化（==序列化是深拷贝==）

## 进阶面试题
### 反射
==java程序在运行的过程中，可以知道这个类的全部的属性与方法，这种动态的获取信息以及调用信息的方法称为反射==

#### 优点
- 在运行期间装载JVM中的类，根据业务的需求动态的执行，提高了代码的灵活性和扩展性
- 可以提高代码的复用率

#### 缺点
- 性能较差，效率低于一般的java代码
- 可维护性差，代码与一般的代码交织在一起

#### 实际的应用场景
实际的使用：
- java中的bean注入（框架中的注解注入）
- jdbc中数据库驱动的加载

#### 为什么反射的性能较差，有没有什么方法可以使得反射的性能变快
java中的反射需要去解析字节码，同时在进行调用的时候，还包含了一些参数的拼接

改进的方法：
- 关闭反射中的安全检查: `m.setAccessible(true);`
- 用缓存将反射得到的元数据保存起来；
- 利用一些高性能的反射库，如ReflectASM ReflectASM 使用字节码生成的方式实现了更为高效的反射机制。执行时会生成一个存取类来 set/get 字段，访问方法或创建实例。一看到 ASM 就能领悟到 ReflectASM 会用字节码生成的方式，而不是依赖于 Java 本身的反射机制来实现的，所以它更快，并且避免了访问原始类型因自动装箱而产生的问题。

### 怎么样去实现动态代理

什么是动态代理:==动态代理指的是在程序运行的过程中生成代理类==

==现在很多框架都是利用类似机制来提供灵活性的扩展性，比如用来包装 RPC 调用，面向切面编程（AOP）等。==

利用反射来实现的

#### 什么时候需要动态代理
想要对一些类的内部的一些方法，在执行前和执行后做一些共同的的操作，而在方法中执行个性化操作的时候--用动态代理

#### 动态代理有两种实现的方式
- JDK中原生的动态代理，需要继承接口来实现，利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用 InvokeHandler 来处理
- 字节码的实现，比如cglib，基于继承业务类的子类实现的

> JDK动态代理是通过接口中的方法名，在动态生成的代理类中调用业务实现类的同名方法；

> CGlib动态代理是通过继承业务类，生成的动态代理类是业务类的子类，通过重写业务方法进行代理；


## java中的四种引用类型（默认的引用类型是强引用）
- 强引用：强引用的对象不会被垃圾回收器回收，只能先进行显示的中断，复制为null，然后在去交给JVM进行回收
    - 只要强引用的对象是可触及的，垃圾收集器就永远不会回收掉被引用的对象。
- 软引用：软引用在内存空间充足的时候，不会被进行垃圾回收，只有在内存空间不足的时候才会进行垃圾回收
- 弱引用：在进行垃圾回收的时候，只要是发现了弱引用的垃圾，就会进行垃圾回收
- 虚引用：形同虚设，如果一个对象仅持有虚引用，那么它相当于没有引用，在任何时候都可能被垃圾回收器回收；

除强引用外，其他3种引用均可以在java.lang.ref包中找到它们
==JVM中的垃圾回收机制对于4中引用都是适用的==

#### 为什么需要四种引用类型
可以按照人为的需求去控制垃圾回收

## JVM是如何实行多态的
==动态绑定技术（动态调用，调用在方法区的方法）==：在执行期间去判断具体的类型

静态调用：在编译期间就确定下来

## 3*0.1==0.3返回值是什么
false，浮点数不能进行精确的计算，一般需要去限制float的计算的长度。

## a=a+b与a+=b的区别
==a += b==会进行隐式的类型转化，而a=a+b则不会自动进行类型转换。

举个例子，如： byte a = 23; byte b = 22; b = a + b;//编译出错 而 b += a; // 编译OK

## int 和Integer谁占用的内存更多?
Integer,Integer是一个对象，需要去储存对象的元数据

==java泛型类在编译期间就确定了数据的类型==

## 引用类型占用的大小
在64位的hotspot中，占用8个字节，32位的占用4个字节（默认的null对象）








