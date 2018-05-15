---
title: 也谈泛型，从 Java 到 Kotlin「Java篇」
---
# 1. 前言
私以为，泛型是 Java 语法中最难理解的一个特性。其一，它的某些特性比较反直觉。其二，Java 中的泛型是阉割版的。Java 只给出了 What，想要理解 Why，还要从 JVM 的其他语言中寻找答案。

本文会就`Raw类型`、`类型擦除`、`通配符`、`kotlin独有的特性`等话题展开探讨。

# 2. Java 中的泛型
>Java1.5 引入了泛型，同时为了兼容旧代码，保留了`Raw类型`。

## 2.1 Raw 类型
>举个例子，`List<Object>`为`泛型`，对应的，`List`为`Raw类型`。两者有些细微的区别。

在没有泛型的年代，多态是一个很好的泛化机制。无论什么对象都可以用 Object 来持有，丢进列表里。不过存在一些安全问题，我们来看一个简单例子：

```java
/**
 * 输出最大的学号
 * @param studentNoList 学号列表
 */
public static void printMaxStudentNo(List studentNoList){
    int maxOrderNo = 0;
    for(Object orderNo: studentNoList)
        if(maxOrderNo < (int)orderNo)
            maxOrderNo = (int)orderNo;
    System.out.println(maxOrderNo);
}

public static void main(String[] args) {
    printMaxStudentNo(Arrays.asList(999, 34, 354));	        // 输出 999
    printMaxStudentNo(Arrays.asList("999", "34", "354"));	// ClassCastException
}
```

我们很快意识到，`studentNoList`中学号的类型不明确，如果调用方不小心，以为学号是 String 类型，传入 String 的列表，有可能招致 Crash。关键点在于，我们无法保证`studentNoList.get()`返回的一定是数字。

引入泛型的其中一个目的是解决这样的类型安全问题。于是，我们得到了类型安全的版本：

```java
public static void printMaxStudentNo(List<Integer> studentNoList){...}

public static void main(String[] args) {
    printMaxStudentNo(Arrays.asList(999, 34, 354));	        // 输出 999
    printMaxStudentNo(Arrays.asList("999", "34", "354"));	// 编译错误
}
```

### 2.1.1 类型擦除
提到这个概念，有经验的 Java 码农总能举出一些例子。例如：

```java
System.out.println(new ArrayList<Integer>().getClass() == new ArrayList<String>().getClass());  // true
```

看一下反编译以后的样子，很好理解，`泛型`在编译以后擦除到了`Raw 类型`：

```java
System.out.println((new ArrayList()).getClass() == (new ArrayList()).getClass());
```
那么`类型擦除`就解释完了。。。才怪，还记得前面的编译错误吗（试图调用`printMaxStudentNo(List<String>)`，与参数类型`List<Integer>`不匹配）？

那么问题来了，既然`List<String>`与`List<Integer>`类型相同，为何又会类型不匹配呢？
有或者说，如何理解如下代码无法通过编译（它与`类型擦除`矛盾吗？）：

```java
ArrayList<Integer> list = new ArrayList<String>();
```
也许，你心中早有答案然后会心一笑，又或者先思考一番，待我慢慢道来。

### 2.1.2 擦除干净了吗
>上述问题先放一边，还有一个好玩的例子。

我们知道，对于泛型`List<T>`，我们不能`new T()`、`new T[]`、`List<T>.class`、`instanceof List<T>`。也就是说，由于擦除，我们失去了获取形参`T`实际类型的能力。

那么对于泛型`List<String>`，我们能否拿到实参`String`的类型呢？答案是肯定的：

[知乎: Java为什么要添加运行时获取泛型的方法？](https://www.zhihu.com/question/36645143/answer/68650398)

```java
// 先转成参数化类型 ParameterizedType
ParameterizedType paramType = (ParameterizedType) new ArrayList<String>(){}.getClass().getGenericSuperclass();
System.out.println(paramType.getActualTypeArguments()[0]);  // 拿到实参，输出 java.lang.String
```
短短两行，我们成功输出了`ArrayList<String>`中的`String`。

是的，泛型实参类型的信息还在，保留在某些角落里。这也正是`Gson`中`TypeToken`所使用的技术。另外，我们心爱的`Retrofit`也用到了同样的技术。

>PS:
>
>注意到代码中是匿名类`new ArrayList<String>(){}`，而非`new ArrayList<String>()`。其中的区别可以结合`TypeToken`细细体会，不展开讲。

## 2.2 「通配符」与 「变型(variance)」
>`变型`这个概念在 Java 中提及很少，几次在《Effective Java》中邂逅它，难免还是不理解。直到后来，在某个更完善的泛型体系中找到了解释，它便是 Scala。
>
>名词解释：总结自 《Scala编程》
>
>不变 (invariance)、协变 (covariance)、逆变 (contravariance)
>
>以`C<T>`为例，给定两个类型 Child 和 Parent，满足`Child extends Parent`，则`C<Child>`与`C<Parent>`之间存在三种关系：
>
> 1. 如果`C<Child>` extends `C<Parent>`，那么 C 是协变的;
> 2. 如果`C<Parent>` extends `C<Child>`，那么 C 是逆变的;
> 3. 如果`C<Child>`与`C<Parent>`毫无关系，那么 C 是不变的。
>
>它们统称为`变型(variance)`。

简而言之，`变型`描述了实参具有继承关系时，对于整体类型之间关系的影响。

Java 中的泛型是不变的，也可以说是阉割版的。后面我们会看到，kotlin 所谓的泛型新特性——`声明点变型`，不过是借鉴了 Scala 的成功经验。好在 kotlin 的泛型是完整的，用起来会更加舒服。


### 2.2.1 「泛型」与 「不变」
> 所谓`不变`，即`ArrayList<Object> list = new ArrayList<String>();`不合法。
>
> 泛型之所以设计成`不变`，是为了类型安全。

具体一点的例子，则有：

```java
List<String> strList = new ArrayList<>(Arrays.asList("haha"));
List<Object> objList = strList;	                    // 1. 企图用 List<Object> 持有它，然后加入数字
objList.add(999);
String lastItem = strList.get(strList.size()-1);    // 2. 获取列表最后一个字符串
```
假设泛型支持`协变`，即 1 处的赋值合法，则在 2 处会得到一个 `ClassCastException`。而设计成`不变`可以在编译时禁止 1 处的赋值，从而提前解决掉问题。这和 kotlin 中引入`空安全`是一个道理。

> PS:
> 
> 值得一提的是，数组被设计成`协变`的。即上述代码用数组来写，会在 2 处会得到一个 `ArrayStoreException`。
> 
> 故有，《Effective Java》第25条：列表优先于数组
> 
> 更多讨论参见 [知乎：java中，数组为什么要设计为协变？](https://www.zhihu.com/question/21394322)

### 2.2.2 「上/下界通配符」与「协/逆变」
>`不变`保证了安全，却降低了灵活性。某些场景下，我们需要`协变`和`逆变`，相应的也会有一些限制。

```java
class Parent{}
class Child extends Parent{}

List<? extends Parent> list = new ArrayList<Child>();	// List<父类>引用List<子类>, 可近似理解为协变 (上界通配符)
List<? super Child> list = new ArrayList<Parent>();	// List<子类>引用List<父类>, 可近似理解为逆变 (下界通配符)
```

它们有啥用呢？设想这样一个场景，先别看源码，考虑实现一个`Collections.copy(dest, src)`，即列表的拷贝。

```java
// 版本1，只支持 List<Object> 的 copy，局限性大
public static void copy(List<Object> dest, List<Object> src){...}

// 版本2，dest 和 src 的实参必须为相同类型，局限性大
public static <T> void copy(List<T> dest, List<T> src){...}

// 版本3，jdk实现。dest实参 可以是 src实参 的父类
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for(int i=0; i<src.size() && i<dest.size(); i++) 
        dest.set(i, src.get(i));
}
```
通过`协变`，src可以持有任何`T的子类型的List`；相应的，通过`逆变`，dest可以持有任何`T的父类型的List`。于是，api 设计上变的更为灵活。

### 2.2.3 「通配符」的限制
考虑到`2.2.1节`中`协变`带来了`objList.add(999)`的问题，最终不幸得到了`ClassCastException`。

具体一点，我们用`协变`改写一下`2.2.1节`的例子（使得 1 处能成功赋值）：

```java
List<String> strList = new ArrayList<>(Arrays.asList("haha"));
List<? extends Object> objList = strList;	    // 1. 企图用 List<? extends Object> 持有它，然后加入数字
objList.add(999);
String lastItem = strList.get(strList.size()-1);    // 2. 获取列表最后一个字符串
```

那么，Java 的设计者们如何应对这种情况呢？答案是，编译器直接禁止了`objList.add(999)`。

> 一般的，对通配符有如下限制：
> 
> 1. 对于协变`? extends T`，只能`get()`，即作为生产者(Producer)。
> 2. 对于逆变`? super T`，只能`set()`，即作为消费者(Consumer)。
> 
> 俗称`PECS`: `Producer-extends`，`Consumer-super`。

另外，Rxjava2 中，把 Action1<T> 改为 Consumer<T> 也是 `PECS` 的体现：

```java
// Rxjava1
public final Subscription subscribe(final Action1<? super T> onNext) 

// Rxjava2, 消费者用 ? super T
public final Disposable subscribe(Consumer<? super T> onNext) {...}
```

# 3. 小结
1. 泛型提供了编译时的类型安全，在运行时擦除到`Raw类型`
2. 擦除并非完全擦除掉泛型信息，某些情况下，可以用反射拿到实参的类型（如 TypeToken）。
3. 泛型列表是不变的，所以优于泛型数组。
4. Java 提供通配符来模拟协变和逆变，有 PECS 口诀。熟记它，看懂 Observable 中的各种 super、extends 不在话下。

# 4. Kotlin 中的泛型
考虑到篇幅过长，将另起一篇来介绍 kotlin 中的泛型。主要有这么几个话题：

1. 声明点变型：Java 中没有
2. 类型投影：对应 Java 的 super 和 extends
3. 星投影

# 5. 参考
《Thinking in Java》

《Effective Java》

《Scala编程》

[仔细说说Java中的泛型](https://zhuanlan.zhihu.com/p/31137677)

[深入理解 Java 泛型](https://blog.csdn.net/u011240877/article/details/53545041)

[知乎: Java 泛型 <? super T> 中 super 怎么 理解？与 extends 有何不同？](https://www.zhihu.com/question/20400700/answer/117464182)

