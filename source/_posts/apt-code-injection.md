---
title: 可代替 ASM，使用 AnnotationProcessor 做代码插桩
---
# 1. 前言
说到代码插桩，你可能会想到 `AspectJ`、`Transfrom Api + ASM` 等等。

代码插桩的用处自不必说，可以做埋点、热修复、组件化路由等等。

然而，`AspectJ`感觉不好用，`ASM` 比较复杂，需要自定义 gradle 插件。好在前段时间，我遇到了新的方法 —— `AnnotationProcessor`。（下面简称为 `apt`）

`apt` 是否只能生成新的 java 文件？还是有什么方法可以直接插入代码，达到 ASM 的效果？

留个悬念，咱们接着往下看。


# 2. apt 与 ButterKnife
说到 apt，不得不说 ButterKnife。

通过注解生成`XXX_ViewBinding`的操作深入人心，然后`Javapoet`也逐渐家喻户晓。

回顾一下，以下是 `jdk` 中提供的 `apt` 相关的 api。

```
- javax
  - annotation.processing
    - AbstractProcessor       // 入口
    - ProcessingEnvironment   // 编译器环境，可理解为 Application
    - Filer                   // 文件读写 util
  - lang.model
    - element
      - Element               // 代码结构信息
    - type
      - TypeMirror            // 编译时的类型信息（非常类似 Class，但那是运行时的东西，注意现在是编译时） 
```

一个常规的注解处理器有这么几步：

1. 继承 `AbstractProcessor`
2. 根据注解获取相关 `Element`
3. 写入 `Filer`
4. 在`app/build/generated/source/apt/`下将生成相关 java 文件

然而，`Filer` 有局限性，只有 create 相关的接口。

```java
public interface Filer {
    JavaFileObject createSourceFile(CharSequence name,
                                    Element... originatingElements) throws IOException;
    ...
}
```

我们得寻找别的方式。

# 3. javac 与 重写 AST
让我们来思考一个问题：

1. AbstractProcessor.process() 这个入口是被什么东西所调用的呢?

当然是编译器啦，通常而言，我们一般用的是`javac`编译器。

现在，我们只需要通读一下 javac 的~~源码~~（[java 编译过程概览](http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html)），就会发现，编译流程大致如下：

1. **Parse and Enter**: `解析 .java 文件`，在内存中生成 **AST (抽象语法树)**、`填充符号表`
2. **Annotation Processing**: 调用 `AbstractProcessor.process()`，若有新的 java 文件生成，则回到步骤 1
3. **Analyse and Generate**: 依次执行`标注检查`、`数据及控制分析`、`解语法糖`、`生成并写入.class文件`

如此一来，我们知道了我们编写的`apt`代码执行在 java 编译过程中的第2步。

如果说，编译过程是 `.java -> AST -> .class` 的过程，那么我们可以在`apt`里修改`AST`这个中间产物，改变最终的`.class`，从而达到等同于`ASM`的效果。

具体而言，我们需要用到一些 `javac` 内部的 api，它们不属于 jdk 的`java/`或者`javax/`包下。而是在 `tools.jar` 的 `com.sun.tools.javac/` 下，具体不再展开。

> AST 详细介绍：[安卓AOP之AST:抽象语法树](https://www.jianshu.com/p/5514cf705666)

# 4. 一个例子，一行注解搞定单例
**设想，我现在有一个`UserManager`，想搞成单例。**

按照原本的生成新文件的方式肯定是不行的。不过现在我们可以插入代码。

1. 自定义一个注解`@Singleton`，以及一个注解处理器`SingletonProcessor`
2. 源代码加一行`@Singleton`:

```java
// UserManager.java
@Singleton
class UserManager {
}
```
apt 插桩后的代码，自动生成`getInstance()`，以及`InstanceHolder`，有没有很爽：

```java
// build 目录下，UserManager.class
@Singleton
class UserManager {

    public static UserManager getInstance() {
        return UserManager._InstanceHolder._sInstance;
    }

    UserManager() {
    }

    private static class _InstanceHolder {
        private static final UserManager _sInstance = new UserManager();

        private _InstanceHolder() {
        }
    }
}
```

实现细节请移步：[https://github.com/fashare2015/java-sugar](https://github.com/fashare2015/java-sugar)

# 5. 后记
作为 java 的忠实粉丝，希望搞几个语法糖出来。因此，胡乱捣鼓出了`java-sugar`这个项目。

其中实现了`单例`、`Builder`、`观察者`等几个常用的设计模式。

另外还做了自动生成`Getter`和`Setter`，这样一来，`java`应该不输给`kotlin`了吧（滑稽）。

也许，大致上可以把 `kotlin` 的语法糖都抄袭一遍？

# 6. 参考
[http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html](http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html)

[Java编译（二）Java前端编译： Java源代码编译成Class文件的过程](https://blog.csdn.net/tjiyu/article/details/53786262)

[Javac黑客指南](http://developer.51cto.com/art/201305/392858.htm)

[安卓AOP之AST:抽象语法树](https://www.jianshu.com/p/5514cf705666)

[Lombok](https://projectlombok.org/features/all)

