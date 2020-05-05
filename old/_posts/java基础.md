---
title: java基础整理
date: 2019-06-14 12:18:41
tags: java
---

java基础
===
1. [**序列化**](https://blog.csdn.net/yaomingyang/article/details/79321939)
---
>- 将***对象***的状态信息转化为可以存储或者传输的形式的过程
>- ***储存媒介***档案、记忆缓冲体；网络传输中为：字节或者xml文件
>- ***反序列化***存储形式还原为对象
>- 创建的对象在jvm的stack中，jvm停止对象消失，***序列化***持久对象需要的时候***反序列化***恢复
>- 应用场景: RMI(远程方法调用)及网络传输
>- API接口：java.io.Serializable && java.io.Externalizable
>- 序列化时，并不保存静态变量: 序列化保存的是对象的状态，静态变量属于类的状态，因此序列化并不保存静态变量
>- Transient关键字: 控制变量的序列化，在变量声明前加上该关键字，可以阻止该变量被序列化到文件中，在被反序列化后，transient 变量的值被设为初始值
>- Externalizable继承了Serializable，该接口中定义了两个抽象方法：writeExternal()与readExternal()。当使用Externalizable接口来进行序列化与反序列化的时候需要开发人员重写writeExternal()与readExternal()方法。若不定义这两个方法的实现细节，则输出的内容为空。
>- 在使用Externalizable进行序列化的时候，在读取对象时，会调用被序列化类的无参构造器去创建一个新的对象，然后再将被保存对象的字段的值分别填充到新对象中。所以，实现Externalizable接口的类必须要提供一个public的无参的构造器。

2. [**java编译&&运行**](https://blog.csdn.net/u012721013/article/details/53199854)
---
>- javac将java程序编译成二进制字节码class
>- jvm虚拟机：加载class 编译成机器码(JIT编译时运行动态编译，AOT先编译后运行) 内存管理
>- java为跨平台语言，运行在jvm虚拟机上，虚拟机本身处理不同操作系统平台问题
>- JIT编译&&运行节省空间效率低，编译跟运行共分资源影响运行；
>- AOT编译完运行，占用空间，运行效率高

[JIT VS AOT](https://blog.csdn.net/h1130189083/article/details/78302502)  
[java虚拟机](https://blog.csdn.net/zhangjg_blog/article/details/20380971)  
[java基础](https://www.javatpoint.com/nested-interface)

3. [jvm vs vmWare](https://stackoverflow.com/questions/861422/is-the-java-virtual-machine-really-a-virtual-machine-in-the-same-sense-as-my-vmw)
---
>- jvm是虚拟进程：程序计数 内存管理 虚拟cpu
>- 启动不同的java程序会启动不同的jvm进程
>- java程序依赖与jvm进程
>- vmWare为虚拟机器: 进程管理 i/o cpu 网络i/o
>- [vmWare vs docker] (https://stackoverflow.com/questions/16047306/how-is-docker-different-from-a-virtual-machine)
>- docker应用打包：资源隔离 应用打包 应用依赖 应用的环境变量等
>- [docker虚拟操作系统共享内核](https://juejin.im/post/5b260ec26fb9a00e8e4b031a)
>- 容器虚拟化的是操作系统而不是硬件，容器之间是共享同一套操作系统资源的。虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统。因此容器的隔离级别会稍低一些。
>- docker  

>> 1. Docker 是世界领先的软件容器平台。
>> 2. Docker 使用 Google 公司推出的 Go 语言  进行开发实现，基于 Linux 内核 的cgroup，namespace，以及AUFS类的UnionFS等技术，***对进程进行封装隔离***，属于***操作系统层面的虚拟化技术***。 由于隔离的进程独立于宿主和其它的隔离的进程，因此也称其为容器。Docke最初实现是基于 LXC.
>> 3. Docker 能够自动执行重复性任务，例如搭建和配置开发环境，从而解放了开发人员以便他们专注在真正重要的事情上：构建杰出的软件。
>> 4. 用户可以方便地创建和使用容器，把自己的应用放入容器。容器还可以进行版本管理、复制、分享、修改，就像管理普通的代码一样。

4. [java注解&&反射](https://blog.csdn.net/briblue/article/details/73824058)

>- 注解类似标签不是程序，通过反射可以获取注解，obj.getclass 利用class获取类中成员 方法 类本身等的注解
>- 注解可以定义期时间周期
>- 反射获取类信息 方法等，方法可以通过invoke执行对应的方法
>- [invoke示例](https://blog.csdn.net/cuiran/article/details/5302074)
>- 代码类似如下：

***主入口：***

```import tags.*;

import java.lang.annotation.Annotation;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;

@SuppressWarnings("deprecation")
@TestAnnotation()
public class MyAnnotation {

    @Check
    int a;
    @Perform
    public void testMethod(){};
    @SuppressWarnings("deprecation")
    public void test1(){
        Hero myHero = new Hero();
        myHero.speak();
        myHero.speak();
        int b = a;
        System.out.println(b);
    }
    private int test2(int c, int d){
        return c+d;
    }
    public MyAnnotation(int ...b ){
        a=b[0];
    }
    public static void main(String[] args){

        boolean ifAnnotation = MyAnnotation.class.isAnnotationPresent(TestAnnotation.class);
        if (ifAnnotation){
            TestAnnotation testAnnotation = MyAnnotation.class.getAnnotation(TestAnnotation.class);
            System.out.println("the id of annotation is: "+testAnnotation.id());
            System.out.println("the string of annnotaion is: " + testAnnotation.msg());

        }
        MyAnnotation myTest = new MyAnnotation(1,2);
        Class myClass = myTest.getClass();
        Field[] myFields = myClass.getDeclaredFields();
        for (Field myField : myFields){
            System.out.println(myField);
        }
        Method[] methods = myClass.getDeclaredMethods();
        // 示例method.invoke
        // methodXX.invoke(Object obj, Object ...args)利用指定的参数args执行指定对象obj中该方法，返回值为object型
        for (Method method : methods){
            if (method.getName()=="test2") {
                try {System.out.println("res of invoke: "+ method.invoke(myTest, new Object[] {new Integer(10), new Integer(20)}));
                }catch(Exception e){
                    e.printStackTrace();

                }
            }

        }

        try {
            Field a = MyAnnotation.class.getDeclaredField("a");
            a.setAccessible(true);
            Check check = a.getAnnotation(Check.class);
            if (check != null) {
                System.out.println(check.value());
            }
            Constructor[] test = MyAnnotation.class.getConstructors();
            int modifiers = test[0].getModifiers();
            System.out.println(Modifier.isPrivate(modifiers));
            for (Constructor tets : test){
                System.out.println("构造函数是否带参数: "+ tets.isVarArgs());

            }

        }catch (NoSuchFieldException e){
            e.printStackTrace();

        }catch (SecurityException e){
            e.printStackTrace();

        }


    }
}
```

***package tags***

```
package tags;

public @interface Check {
	//注解的属性 名为value类型为string默认值为"hi"
    String value() default "hi";
}

package tags;


public class Hero {
    @Deprecated()
    public void say(){
        System.out.println("Noting has to say!");
    }

    public void speak(){
        System.out.println("I have a dream!");
    }
}

package tags;

public @interface Perform {
}

package tags;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
//target为基本注解限定注解可以使用的范围
@Target(ElementType.TYPE)
// retention限定注解的生命周期
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation {
    int id() default 1;
    String msg() default "hi";
}
```