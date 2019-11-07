# 谈一谈JAVA里的语法糖

上周在听大牛分享JVM编译优化时，提到了在编译阶段会进行的一个步骤：解语法糖。他提出了一个问题是：JAVA里有哪些语法糖，听到这个问题，似乎很容易回答，因为我们都知道java有很多语法糖，但话到嘴边，也就只能想起字符串拼接和foreach来，果然，没有经过系统的梳理，一些听起来简单的知识点也会难倒你，所以我来梳理一下，看java里到底有哪些语法糖。

### 概述

在搜狗百科中，语法糖含义解释如下：语法糖（Syntactic sugar），也译为糖衣语法，是由英国计算机科学家彼得·约翰·兰达（Peter J. Landin）发明的一个术语，指[计算机语言](https://baike.sogou.com/lemma/ShowInnerLink.htm?lemmaId=664318&ss_c=ssc.citiao.link)中添加的某种语法，这种语法对语言的功能并没有影响，但是更方便程序员使用。通常来说使用语法糖能够增加程序的可读性，从而减少程序代码出错的机会。

从概念上可以看出来，简单地说就是通过更简单的语法实现原有功能，对语言功能没有影响。以java来说就是某个语法可能JVM不支持，但是编译器可以将语法转换成基础语法以实现功能。相当于快捷方式。

### 有哪些语法糖

下面我们写一些语法糖使用，通过反编译看看他们是怎么实现的。

####增强for循环



``` java
    /**
     * 增强for循环
     */
    public void forTest() {
        List<String> stringList = Arrays.asList("A", "B");
        for (String str :stringList) {
            System.out.println(str);
        }
    }
```

#### 

反编译class文件后

```java
    public void forTest() {
        List<String> stringList = Arrays.asList("A", "B");
        Iterator var2 = stringList.iterator();

        while(var2.hasNext()) {
            String str = (String)var2.next();
            System.out.println(str);
        }

    }
```

通过Iterator实现，这里在分析fail-fast机制时提过，不能在增强for循环中进行集合的增删操作，否则会抛异常。

#### switch(String)

```java
/**
     * switch(String)
     */
    public int switchString(String str) {
        switch (str){
            case "A":
               return 1;
            case "B":
                return 2;
            default:
                return 0;
        }
    }
```

```java
public int switchString(String str) {
        byte var3 = -1;
        switch(str.hashCode()) {
        case 65:
            if (str.equals("A")) {
                var3 = 0;
            }
            break;
        case 66:
            if (str.equals("B")) {
                var3 = 1;
            }
        }

        switch(var3) {
        case 0:
            return 1;
        case 1:
            return 2;
        default:
            return 0;
        }
    }
```

可以看到switch(String)是转为了switch(byte)，之前单独分析过这里的机制。

#### 条件编译

```java
    /**
     * 条件编译
     */
    public void ifTest() {
        if (true)  {
            System.out.println("1");
        } else {
            System.out.println("2");
        }
    }
```

```java
    public void ifTest() {
        System.out.println("1");
    }
```

当条件为常量时，一定不会执行的分支会自动清除。

#### 可变参数

```java
    /**
     * 可变参数
     * @param strings
     */
    public void strings(String... strings) {
        for (String s : strings) {
            System.out.println(s);
        }
    }
```

```java
    public void strings(String... strings) {
        String[] var2 = strings;
        int var3 = strings.length;

        for(int var4 = 0; var4 < var3; ++var4) {
            String s = var2[var4];
            System.out.println(s);
        }

    }
```

可变参数实际是变长数组

#### 泛型

```java
    /**
     * 泛型
     */
    public void listTest() {
        List<String> list = new ArrayList<String>();
        int i = list.size();
    }
```

```java
    public void listTest() {
        List<String> list = new ArrayList();
        int i = list.size();
    }
```

可以看到，泛型参数已经被擦除了。

#### lambda表达式

```java
    /**
     * lambda表达式
     */
    public void lambdaTest() {
        List<String> list = new ArrayList<>();
        list = list.stream().distinct().collect(Collectors.toCollection(LinkedList::new));
    }
```

#### try with resource

```java
    /**
     * try with resource
     */
    public void tryResourceTest() throws IOException {
        try(StringWriter writer = new StringWriter()) {
            writer.write(1);
        }
    }
```

```java
public void tryResourceTest() throws IOException {
        StringWriter writer = new StringWriter();
        Throwable var2 = null;

        try {
            writer.write(1);
        } catch (Throwable var11) {
            var2 = var11;
            throw var11;
        } finally {
            if (writer != null) {
                if (var2 != null) {
                    try {
                        writer.close();
                    } catch (Throwable var10) {
                        var2.addSuppressed(var10);
                    }
                } else {
                    writer.close();
                }
            }

        }

    }
```



Try with resource是自动帮你加上finally，并且调用closeable接口里的close方法。