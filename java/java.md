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

## springmvc @ModelAttribute注解

### 放在方法前

在每个handler方法执行之间，执行这个函数，并且把返回值作为value放入model，对应的key设置为注解的value属性，如果没有这个属性，则设置为返回值类型的小写。

### 放在方法参数前

在执行这个方法之前，寻找model中有没有注解中value属性的key（没有value属性则使用参数类型名的全小写）（显然是被之前的某个有modelattribute注解的方法返回的），如果有则绑定到参数，然后寻找请求中有没有可以注入的对象（比如form表单提交的参数），如果有则注入并绑定model，覆盖之前的值。

其实这个注解可以通过很多种方式绑定参数，比较复杂，上述的只是常见两种的情况。

## http编码

http标准规定必须是iso8859编码。这种编码每个字符只有一个字节（其实就完全可以当成字节流，因为最终解释成哪一种编码完全由服务器和浏览器决定），无法表示中文，所以为了折中，比如浏览器和服务器之间约定使用utf-8编码(服务器端可以在在java代码中控制，浏览器端在前端代码中控制)，但utf-8每个字符占1-3个字节，其实实际上会把utf-8编码的字符串拆成一个一个字节，然后假装成iso--8859编码，接着会使用iso8859传输http请求，然后再以utf-8解释。

```java
String downloadfile =  new String(filename.getBytes("UTF-8"),"iso-8859-1");
        // 以下载方式打开文件
        headers.setContentDispositionFormData("attachment", filename);
```

上图：因为java内部对String的存储使用utf16编码，但因为服务器和浏览器约定使用utf-8，所以先拿到utf-8编码的字节数组，事实上这个字节数组就完全可以作为iso8859来传输了，但我们需要将其放入String才能继续使用java，此时可以传入字节数组和编码方式，String内部将会认为bytes数组是以这个编码为标准的。（为什么不能传入其他编码方式？假如你传入的是utf8，那java将可能会认为三个字节为一个字符，以后和另一种编码（比如gbk）的操作系统交互时就会把这三个字节作为一组译成gbk编码，这是完全错误的，所以我们要告诉jvm正确的编码）



为什么需要告诉jvm编码呢？反正最后也是让utf8以iso8859来加入httpheader，而iso8859其实是字节流，并不会影响到内部的字节数组。

个人认为这可能是因为headers设计成只接受iso8859,并且会将不符合这样编码的参数自动转换为iso8859，比如java的String默认是utf16，所有字符都是两字节，此时两字节的中文字符因为没有对应的就会全部乱码，但两字节的英文字符因为可以转换所以可以正常显示。