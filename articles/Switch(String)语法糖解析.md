# switch(String)语法糖解析

> Java1.5之前，switch语法结构仅支持int, byte, short, char这几个基本类型(及对应的包装类型)。
>
> 1.5后提供了enum枚举结构。
>
> Java7提供了switch(String)结构。

### 问题一

经常听到Java7中提供的switch(String)是Java语言的一个语法糖，实际JVM并不支持这个语法结构，但这个语法糖具体是怎么实现的，一直没有看过，今天比较有空，我来研究一下。

首先写一段switch结构的代码。

```java
public static void main(String[] args) {
        final String str = "F";

        switch (str) {
            case "A" :
                System.out.println("A");
                break;
            case "B" :
                System.out.println("B");
                break;
            default:
                System.out.println("F");
        }
    }
```

查看idea反编译后的源码。

```java
public static void main(String[] args) {
        String str = "F";
        String var2 = "F";
        byte var3 = -1;
        switch(var2.hashCode()) {
        case 65:
            if (var2.equals("A")) {
                var3 = 0;
            }
            break;
        case 66:
            if (var2.equals("B")) {
                var3 = 1;
            }
        }

        switch(var3) {
        case 0:
            System.out.println("A");
            break;
        case 1:
            System.out.println("B");
            break;
        default:
            System.out.println("F");
        }

    }
```

可以看到，switch结构中变为了String.hashcode()方法，利用其返回的int值进行判断，所以说编译后还是使用了switch(int)结构来实现的。大家都知道String的hashcode方法是有哈希冲突的风险的，所以在每个case条件中增加了equals作为补充判断，避免哈希冲突错误。

### 问题二

上面说增加了equals，这样能避免哈希冲突导致问题，但哈希冲突时，比如case("A") ，case("B")，加入A和B会计算出相同的hash值，那岂不是两个case的条件一样，那这样应该连编译都通不过呀。所以我找了一个哈希冲突的例子来试一下(找了半天没找到，看到一篇博客里找到了)

```java
public static void main(String[] args) {
        final String str = "test";
        switch (str) {
            case "AaAa":
                System.out.println("a");
                break;
            case "BBBB":
                System.out.println("b");
                break;
            case "AaBB":
                System.out.println("c");
                break;
            default:
                System.out.println("c");
                break;
        }
    }
```

反编译源码

```java
public static void main(String[] args) {
        String str = "test";
        String var2 = "test";
        byte var3 = -1;
        switch(var2.hashCode()) {
        case 2031744:
            if (var2.equals("AaBB")) {
                var3 = 2;
            } else if (var2.equals("BBBB")) {
                var3 = 1;
            } else if (var2.equals("AaAa")) {
                var3 = 0;
            }
        default:
            switch(var3) {
            case 0:
                System.out.println("a");
                break;
            case 1:
                System.out.println("b");
                break;
            case 2:
                System.out.println("c");
                break;
            default:
                System.out.println("c");
            }

        }
    }
```

果然反编译后的结构发生了变化，源代码里三个相同hash值的case，编译后只有一个case，里面使用了if else的结构来做判断，这样就万无一失了，确实解决了哈希冲突的场景。

### 问题三

看反编译后的代码，还会有一个问题，为什么一个switch编译后会拆分为两个switch结构，为什么要新增一个switch(byte)结构呢，这样貌似不是必要的，仅仅通过上面的switch和if else结构就足以应付各种场景了。增加一个结构反而会影响性能。那么到底是什么原因导致增加一个switch(byte)结构呢。这个答案是看一篇博客找到的。

看下面这种场景

```java
public static void main(String[] args) {
        final String str = "test";
        switch (str) {
            case "AaAa":
                System.out.println("a");
            case "BBBB":
                System.out.println("b");
                break;
            case "AaBB":
                System.out.println("c");
                break;
            default:
                System.out.println("c");
                break;
        }
    }
```

```java
public static void main(String[] args) {
        String str = "test";
        String var2 = "test";
        byte var3 = -1;
        switch(var2.hashCode()) {
        case 2031744:
            if (var2.equals("AaBB")) {
                var3 = 2;
            } else if (var2.equals("BBBB")) {
                var3 = 1;
            } else if (var2.equals("AaAa")) {
                var3 = 0;
            }
        default:
            switch(var3) {
            case 0:
                System.out.println("a");
            case 1:
                System.out.println("b");
                break;
            case 2:
                System.out.println("c");
                break;
            default:
                System.out.println("c");
            }

        }
    }
```

看到这里应该就明白了，是因为switch的特殊语法结构，自上而下地处理每个条件，在不加break的时候，会继续进行下一个条件的判断。在上面那种case + if else 时就处理不了这种情况，所以第一个switch结构只是为了解决哈希冲突的问题，唯一定位一个case，下面的switch(byte)才是执行。而且这样的写法会比较清晰。