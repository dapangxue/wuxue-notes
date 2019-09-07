# Java基础

## 一、String类

String被声明为final，表示String类是不能被继承的。

在JDK8中，String内部使用char数组存储数据，根据源码可以看出char数组也是final的，意味着value数组初始化之后就不能在引用其他数组，且作用范围是private，在String内部没有提供改变value数组的方法，保证了String的不可变性。

```JAVA
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
```

在JDK9之后采用了byte数组存储字符串。

### 不可变的优点

#### 1.可以缓存hash值

String的内容的不可变，保证了hashCode方法返回值的固定，用作HashMap的key。不可变的特性使得hash值也不可变，因此只需要进行一次计算。

#### 2.字符串常量池的需要

创建一个字符串，会在字符串常量池中保存字符串的字面量，String的不可变性才让字符串常量池可用。

#### 3.安全性

假设有一个方法采用String类型的变量作为参数，都会复制一份引用，该引用所指向的对象其实一直都待在单一的物理位置上，那么如果在方法中发生了修改，引用会重新指向新的字符串，不会影响参数字符串本身。

#### 4.线程安全

String不可变性天生具备线程安全的特性，可以在多个线程中安全的使用。
