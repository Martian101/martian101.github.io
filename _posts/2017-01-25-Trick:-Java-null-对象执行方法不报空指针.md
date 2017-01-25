运行下面的代码:

``` Java
public class RunNull {
    public static void main(String[] args) {
        Thread t = null;
        t.yield();
    }
}
```

`t`明明是null，但是程序并没有报空指针，为什么呢？

再看下面的代码:

``` Java
public class RunNull {
    public static void main(String[] args) {
        StaticObj obj = null;
        obj.say();
    }
}
```

``` Java
public class StaticObj {
    public static void say() {
        System.out.println("hello world");
    }
}
```

运行同样没有报空指针，并且函数正确执行，在控制台打印出了`hello world`。
思考一下，在Java里`null`可以被转化为任何类型的引用，尽管强制转换后的对象仍然是`null`，但是类型变了，这样执行类的static方法当然是可以的。初始化一个空的线程类，调用`yield()`方法没有报空指针也是这个原因。
