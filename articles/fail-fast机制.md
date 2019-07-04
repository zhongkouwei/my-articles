# fail-fast机制解读

[TOC]

### 集合的增删

​	平时经常会有一些对集合的增删场景，尤其是在循环内进行删除，下面我们看下这几种场景。

#### 普通for循环

​	首先，使用 普通for循环可以对集合进行增删，但增删后由于普通for循环时是通过下标索引访问，因此有可能遇到某些数据读不到的问题。进行完全遍历时，由于集合长度已发生变化，会抛出IndexOutOfBoundsException下标越界异常。

​	看一个例子。

```java
            for (int i = 0; i <6 ; i++) {
                System.out.println("读取"+list.get(i));
                if (3 == i) {
                    list.remove(i);
                }
            }
```

上述代码输出了

```console
读取0
读取1
读取2
读取3
读取5
Exception in thread "Thread-1" java.lang.IndexOutOfBoundsException: Index: 5, Size: 5
```

读取了3之后直接跳到5，没有读取4，因为删除掉[3]之后，原本在[4]位置的4下标变为[3]，但循环已经跳过，所以漏了一个。此处可以通过手动控制在删除后i-1可以避免。但最后的下标越界异常是无法避免的，因此不要在for循环内进行超过1个的集合增删操作。

#### 增强for循环

​	在《阿里巴巴JAVA开发规范》中有这样一段话

> 【强制】不要在 foreach 循环里进行元素的 remove/add 操作。remove 元素请使用 Iterator 
>
> 方式，如果并发操作，需要对 Iterator 对象加锁。 

写代码尝试了一下，在foreach中进行任意的增删操作，均会抛出ConcurrentModificationException异常。我们写一段代码。

```java
						for (int j: list) {
                System.out.println("读取"+j);
                if (3 == j) {
                    list.remove(j);
                }
            }
```

然后看下由class文件反编译后的源码。

```java
						Iterator var0 = list.iterator();

            while(var0.hasNext()) {
                int j = (Integer)var0.next();
                System.out.println("读取" + j);
                if (3 == j) {
                    list.remove(j);
                }
            }
```

​	可以看到，foreach只是java语言的语法糖，本质上还是由Iterator来迭代的，但在删除时，调用的是list的remove方法，正是这里引发了修改异常。

​	之前在分析list源码的时候，提到过两个关键变量modCount和expectedModCount，在Itr中进行增删时，都会进行判断。

```java
				final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
```

​	跑个题，简单再说一下这两个变量的作用，首先modCount是AbstractList父类的一个属性，在集合进行结构变化时都会进行自增以记录修改次数。

```java
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```

​	在AbstractList中有一个私有内部成员类Itr，通过Iterator接口返回，在使用迭代器对集合迭代时候便会返回这个对象，Itr里面有一个属性，exceptedModCount，在Itr进行初始化时候，expectedModCount = modCount。

```java
private class Itr implements Iterator<E> {
        
        int expectedModCount = modCount;  // 赋值，等于modCount
        
        ..........
}
```

-  modCount记录的是集合真正的修改次数。

- expectedModCount记录的是使用当前迭代器时集合的修改次数。

  看到这里，大家应该就清楚了，每次开始迭代时候，用exceptedModCount记录下当前的modCount，这样，如果集合在其他地方进行了修改，两个值就会不一样，直接抛出异常。

那么它是怎么保证在当前迭代器进行修改不会有问题的呢？

#### 迭代器遍历

我们再使用迭代器进行遍历.

```java
            Iterator var0 = list.iterator();

            while(var0.hasNext()) {
                int j = (Integer)var0.next();
                System.out.println("读取" + j);
                if (3 == j) {
                    var0.remove();
                }
            }
```

```
读取0
读取1
读取2
读取3
读取4
读取5
```



这段代码和上面使用foreach的反编译源码相比，只有一个改动，那就是将list.remove()，改为了Iterator.remove，从上面的代码分析中我们知道了这是调用了Itr对象的remove方法，那这个方法怎么保证不会抛出异常呢？

```java
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount; // modCount此时已经修改，进行同步
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```

可以看到，在Itr中删除时，remove后会对exceptedModCount和modCount进行同步，从而保证了在当前迭代器内修改不会抛出异常。

引申一点，使用Iterator为什么没有出现使用普通for循环时的下标越界异常呢？因为Iterator遍历时候是通过指针操作，在增删时候会修改指针，避免了这个问题。

### fail-fast机制

​	通过分析上面知道了几种集合增删方式，可以看到，在多线程并发读取和修改集合时，也许并不会真正出问题，但为了防止这种情况导致的数据不一致性，通过记录集合的修改次数直接在这种情况出现时抛出异常，**实现了fail-fast机制**，确保同一时间只有一个线程修改或遍历线程。

​	它只是一种错误检测机制，做到了提前检测，但不一定会发生。

​	用线程的说法也不准确，因为通过上面的源码分析我们知道，fail-fast机制核心是保证了只在当前迭代器内修改，所以在单线程环境下，如果在迭代器外发生了修改（像上面的foreach），也会抛出异常。

#### 知识点

- 单线程环境和多线程环境都会发生ConcurrentModificationException异常，关键看是不是一个迭代器。
- 使用普通for循环可以少量删除，但可能会发生数据遗漏和数组下标越界异常。
- 增强for循环只是迭代器的语法糖，由于触发了Iterator的fail-fast机制，所以完全**无法**进行集合修改。

#### 我们应该怎么做

1. 在单线程环境下，使用Iterator进行集合的遍历修改。
2. 在多线程环境下，使用concurrent中的类来替换ArrayList和HashMap。不推荐使用同步锁，会额外造成阻塞。