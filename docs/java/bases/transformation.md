# URL
  - https://www.cnblogs.com/xiaoyezideboke/p/10939219.html

Java 转型问题（向上转型和向下转型）
Java 转型问题其实并不复杂，只要记住一句话：父类引用指向子类对象。

什么叫父类引用指向子类对象？

从 2 个名词开始说起：向上转型(upcasting) 、向下转型(downcasting)。

举个例子：有2个类，Father 是父类，Son 类继承自 Father。

第 1 个例子：

Father f1 = new Son();   // 这就叫 upcasting （向上转型)
// 现在 f1 引用指向一个Son对象

Son s1 = (Son)f1;   // 这就叫 downcasting (向下转型)
// 现在f1 还是指向 Son对象
第 2 个例子：

Father f2 = new Father();
Son s2 = (Son)f2;       // 出错，子类引用不能指向父类对象
你或许会问，第1个例子中：Son s1 = (Son)f1; 问为什么是正确的呢。

很简单因为 f1 指向一个子类对象，Father f1 = new Son(); 子类 s1 引用当然可以指向子类对象了。

而 f2 被传给了一个 Father 对象，Father f2 = new Father(); 子类 s2 引用不能指向父类对象。

总结：

1、父类引用指向子类对象，而子类引用不能指向父类对象。

2、把子类对象直接赋给父类引用叫upcasting向上转型，向上转型不用强制转换吗，如：

Father f1 = new Son();
3、把指向子类对象的父类引用赋给子类引用叫向下转型(downcasting)，要强制转换，如：

f1 就是一个指向子类对象的父类引用。把f1赋给子类引用 s1 即 Son s1 = (Son)f1;

其中 f1 前面的(Son)必须加上，进行强制转换。

一、向上转型。
通俗地讲即是将子类对象转为父类对象。此处父类对象可以是接口。

1、向上转型中的方法调用：

实例
 转型
 

注意这里的向上转型：

Animal b=new Bird(); //向上转型
b.eat();
此处将调用子类的 eat() 方法。原因：b 实际指向的是 Bird 子类，故调用时会调用子类本身的方法。

需要注意的是向上转型时 b 会遗失除与父类对象共有的其他方法。如本例中的 fly 方法不再为 b 所有。

2、向上转型的作用

看上面的代码：

public static void doEate(Animail h) {
    h.sleep();
}
这里以父类为参数，调有时用子类作为参数，就是利用了向上转型。这样使代码变得简洁。不然的话，如果 doEate 以子类对象为参数，则有多少个子类就需要写多少个函数。这也体现了 JAVA 的抽象编程思想。

二、向下转型。
与向上转型相反，即是把父类对象转为子类对象。
public class Animail {
    private String name="Animail";
    public void eat(){
        System.out.println(name+" eate");
    }
}

class Human extends Animail{
    private String name="Human";
    public void eat(){
        System.out.println(name+" eate");
    }
}

class Main {
    public static void main(String[] args) {
        Animail a1=new Human();//向上转型
        Animail a2=new Animail();
        Human b1=(Human)a1;// 向下转型,编译和运行皆不会出错
 //       Human c=(Human)a2;//不安全的向下转型,编译无错但会运行会出错
    }
}

 
 实例
 

Animail a1=new Human();//向上转型
Human b1=(Human)a1;// 向下转型,编译和运行皆不会出错
这里的向下转型是安全的。因为 a1 指向的是子类对象。

而

Animail a2=new Animail();
Human c=(Human)a2;//不安全的向下转型,编译无错但会运行会出错
 
运行出错：

Exception in thread "main" java.lang.ClassCastException: study.转型实例.Animail cannot be cast to study.转型实例.Human
at study.转型实例.Main.main(Main.java:8)

向下转型的作用

向上转型时 b会遗失除与父类对象共有的其他方法；可以用向下转型在重新转回，这个和向上转型的作用要结合理解。

 

三、当转型遇到重写和同名数据
看下面一个例子，你觉得它会输出什么？

复制代码
public class A {
   public int i=10;
   void print(){
       System.out.println("我是A中的函数");
   }
}
class B extends A{
   public int i=20;
    void print(){
        System.out.println("我是B中的函数，我重写了A中的同名函数");
    }
    void speek(){
        System.out.println("向上转型时我会丢失");
    }

   public static void main(String[] args) {
        B b=new B();
        A a=b;//此处向上转型
        b.print();  System.out.println(b.i);
        b.speek();
        a.print();  System.out.println(a.i);
       ((B) a).speek();//a在创建时虽然丢失了speek方法但是向下转型又找回了

    }
}
复制代码
结果


我是B中的函数，我重写了A中的同名函数
20
向上转型时我会丢失
我是B中的函数，我重写了A中的同名函数
10
向上转型时我会丢失
我们发现同名数据是根据创建对象的对象类型而确定，而这个子类重写的函数涉及了多态，重写的函数不会因为向上转型而丢失

多态存在的三个必要条件
继承
重写
父类引用指向子类对象
所以不要弄混淆了，父类的方法在重写后会被子类覆盖，当需要在子类中调用父类的被重写方法时，要使用super关键字

　对于多态，可以总结它为：
　　一、使用父类类型的引用指向子类的对象；
　　二、该引用只能调用父类中定义的方法和变量；
　　三、如果子类中重写了父类中的一个方法，那么在调用这个方法的时候，将会调用子类中的这个方法；（动态连接、动态调用）
　　四、变量不能被重写（覆盖），”重写“的概念只针对方法，如果在子类中”重写“了父类中的变量，那么在编译时会报错。

   方法可重写，属性不可重写。父类的方法被子类覆盖，父类的属性不被子类覆盖。 

标签: java转型