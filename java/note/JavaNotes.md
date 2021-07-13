# java 

## 泛型

- 多个泛型函数嵌套调用时，第二层开始编译期间无法判断模板参数T的匹配类型，所以没有帮我们加上类型转换语句。

## 泛型可变参数

```java
<T> T[] test(T... objs)
```

### 泛型可变参数的原理

- 1. 若传入多个参数，则新建一个array装入，具体类型由各个参数类型匹配的共同基类决定。
- 2. 若传入单个参数，若为普通类型，则新建一个array装入。
- 3. 若传入单个参数且为数组类型，则不打包直接传入。

### 可能导致的问题

若代码如下所示，则不会有问题，因为pickTwo传入三个String，传入瞬间先打包为String[]，然后由K...接收，由于java泛型使用擦除法实现，所以在jvm看来，实际上pickTwo的参数是Object...，也就是Object[]，此时传入的参数是Object[]类型的引用，指向的是String[]对象，然后调用asArray，此时由于编译期不能确定K的类型，所以T为所以擦除法并不帮我们添加任何类型转换，由于参数只有一个且为数组，所以满足上述情况3，不新建array直接传入。asArray返回一个Object[]的引用类型，但其指向的仍然是一开始新建的String[]。pickTwo返回的也是这样的对象引用，回到main后由于擦除法把K匹配为String，所以帮我们在pickTwo调用后添加了转为String[]的语句，此时由于返回值虽然是Object[]类型的引用，但其实际指向一个String[]对象，所以转换成功，下图中注释部分即为编译器处理后的内容。

```java
public class Main {
    public static void main(String[] args) {
        var c= pickTwo("one","two","three");//String[] c = (String[])pickTwo("one", "two", "three");
    }

    static <K> K[] pickTwo(K...objs) {
        var a=asArray(objs);//K[] a = asArray(objs);
        return a;
    }

    static <T> T[] asArray(T... objs) {
        return objs;
    }

}

```

若代码如下所示，则会有问题，整体过程和上述大体一样，但调用pickTwo时因为参数是普通参数表，所以传入的k1，k2，k3分别是三个Object引用，指向三个String对象，然后调用asArray，此时满足上述的情况1，需要新建一个数组，但是jvm其实是根据引用类型判断建立的数组类型，而不是根据实际指向对象的类型，所以这时建立的是Object[]引用，指向一个Object[]数组，其中存储了三个Object引用，但指向三个String对象，接下来没有问题，因为jvm执行泛型函数时均把T和K当作Object，最后pickTwo返回后，返回的是一个Object[]引用，指向一个Object[]数组，其中存储了三个Object引用，但指向三个String对象，最后有趣擦除法在pickTwo前面添加了(String[])类型转换语句，但是此时返回的Object[]引用指向的是一个真正的Object[]数组，所以转换失败。抛出异常。

```java
public class Main {
    public static void main(String[] args) {
        var c= pickTwo("one","two","three");//String[] c = (String[])pickTwo("one", "two", "three");
    }

    static <K> K[] pickTwo(K k1, K k2,K k3) {
        var a=asArray(objs);//K[] a = asArray(objs);
        return a;
    }

    static <T> T[] asArray(T... objs) {
        return objs;
    }

}

```

### 问题原因

原因是java一般来讲

- 传递的参数若是int这种基础类型， 则参数是新建一个int，是传值。
- 传递的参数若是String这种对象，则参数是新建一个引用，是传引用。
- 若使用参数表，则是新建一个数组，把参数装进去，然后传递数组对象的引用。

问题就出在上述第三种情况，当多个的泛型函数嵌套调用时，若某一个函数使用普通参数列表，但他调用的函数使用了可变参数列表，此时无论传入的引用指向的对象的实际类型是什么，jvm只能看到引用的类型，由于jvm把所有泛型参数都当作Object，此时建立的可变参数数组一定是Object[]引用，指向一个Object[]数组,在这之后返回时，某一层由于可以被编译器判断具体类型而加上了类型转换语句，此时这个类型转换就会出错。
