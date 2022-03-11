---
category: Java
tag:
  - JVM
---

# 类加载器详解

## 回顾一下类加载过程

类加载过程：**加载->连接->初始化**。连接过程又可分为三步：**验证->准备->解析**。

![类加载过程](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/类加载过程.png)

一个非数组类的加载阶段（加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，这一步我们可以去自定义类加载器去控制字节流的获取方式（重写一个类加载器的 `loadClass()` 方法）。数组类型不通过类加载器创建，它由 Java 虚拟机直接创建。

所有的类都由类加载器加载，加载的作用就是将 `.class`文件加载到内存。

## 类加载器总结

JVM 中内置了三个重要的 ClassLoader，除了 BootstrapClassLoader 其他类加载器均由 Java 实现且全部继承自`java.lang.ClassLoader`：

1.  **BootstrapClassLoader(启动类加载器)** ：最顶层的加载类，由 C++实现，负责加载 `%JAVA_HOME%/lib`目录下的 jar 包和类或者被 `-Xbootclasspath`参数指定的路径中的所有类。
2.  **ExtensionClassLoader(扩展类加载器)** ：主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类，或被 `java.ext.dirs` 系统变量所指定的路径下的 jar 包。
3.  **AppClassLoader(应用程序类加载器)** ：面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。

## 双亲委派模型

### 双亲委派模型介绍

每一个类都有一个对应它的类加载器。系统中的 ClassLoader 在协同工作的时候会默认使用 **双亲委派模型** 。即在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则才会尝试加载。加载的时候，首先会把该请求委派给父类加载器的 `loadClass()` 处理，因此所有的请求最终都应该传送到顶层的启动类加载器 `BootstrapClassLoader` 中。当父类加载器无法处理时，才由自己来处理。当父类加载器为 null 时，会使用启动类加载器 `BootstrapClassLoader` 作为父类加载器。

![ClassLoader](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/classloader_WPS图片.png)

每个类加载都有一个父类加载器，我们通过下面的程序来验证。

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        System.out.println("ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader());
        System.out.println("The Parent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader().getParent());
        System.out.println("The GrandParent of ClassLodarDemo's ClassLoader is " + ClassLoaderDemo.class.getClassLoader().getParent().getParent());
    }
}
```

Output

```
ClassLodarDemo's ClassLoader is sun.misc.Launcher$AppClassLoader@18b4aac2
The Parent of ClassLodarDemo's ClassLoader is sun.misc.Launcher$ExtClassLoader@1b6d3586
The GrandParent of ClassLodarDemo's ClassLoader is null
```

`AppClassLoader`的父类加载器为`ExtClassLoader`，
`ExtClassLoader`的父类加载器为 null，**null 并不代表`ExtClassLoader`没有父类加载器，而是 `BootstrapClassLoader`** 。

其实这个双亲翻译的容易让别人误解，我们一般理解的双亲都是父母，这里的双亲更多地表达的是“父母这一辈”的人而已，并不是说真的有一个 Mother ClassLoader 和一个 Father ClassLoader 。另外，类加载器之间的“父子”关系也不是通过继承来体现的，是由“优先级”来决定。官方 API 文档对这部分的描述如下:

> The Java platform uses a delegation model for loading classes. **The basic idea is that every class loader has a "parent" class loader.** When loading a class, a class loader first "delegates" the search for the class to its parent class loader before attempting to find the class itself.

### 双亲委派模型实现源码分析

双亲委派模型的实现代码非常简单，逻辑非常清晰，都集中在 `java.lang.ClassLoader` 的 `loadClass()` 中，相关代码如下所示。

```java
private final ClassLoader parent;
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查请求的类是否已经被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//父加载器不为空，调用父加载器loadClass()方法处理
                        c = parent.loadClass(name, false);
                    } else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //抛出异常说明父类加载器无法完成加载请求
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                    //自己尝试加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

### 双亲委派模型的好处

- 双亲委派模型保证了 Java 程序的稳定运行，
- 可以避免类的重复加载（JVM 区分不同类的方式不仅仅根据类名，相同的类文件被不同的类加载器加载产生的是两个不同的类），
- 也保证了 Java 的核心 API 不被篡改。

如果没有使用双亲委派模型，而是每个类加载器加载自己的话就会出现一些问题，比如我们编写一个称为 `java.lang.Object` 类的话，那么程序运行的时候，系统就会出现多个不同的 `Object` 类。

### 如果我们不想用双亲委派模型怎么办？

~~为了避免双亲委托机制，我们可以自己定义一个类加载器，然后重写 `loadClass()` 即可。~~

**🐛 修正（参见：[issue871](https://github.com/Snailclimb/JavaGuide/issues/871) ）** ：自定义加载器的话，需要继承 `ClassLoader` 。如果我们不想打破双亲委派模型，就重写 `ClassLoader` 类中的 `findClass()` 方法即可，无法被父类加载器加载的类最终会通过这个方法被加载。但是，如果想打破双亲委派模型则需要重写 `loadClass()` 方法

## 自定义类加载器

​		通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。**比如应用是通过网络来传输 Java类的字节码，为保证安全性，这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现**。自定义类加载器一般都是继承自`ClassLoader`类，从上面对`loadClass`方法来分析来看，我们只需要重写 findClass 方法即可。下面我们通过一个示例来演示自定义类加载器的流程：

```java
    package com.neo.classloader;
        
        import java.io.*;
        
        public class MyClassLoader extends ClassLoader {
            private String root;
        
            protected Class<?> findClass(String name) throws ClassNotFoundException {
                byte[] classData = loadClassData(name);
                if (classData == null) {
                    throw new ClassNotFoundException();
                } else {
                    return defineClass(name, classData, 0, classData.length);
                }
            }
        
            private byte[] loadClassData(String className) {
                String fileName = root + File.separatorChar
                        + className.replace('.', File.separatorChar) + ".class";
                try {
                    InputStream ins = new FileInputStream(fileName);
                    ByteArrayOutputStream baos = new ByteArrayOutputStream();
                    int bufferSize = 1024;
                    byte[] buffer = new byte[bufferSize];
                    int length = 0;
                    while ((length = ins.read(buffer)) != -1) {
                        baos.write(buffer, 0, length);
                    }
                    return baos.toByteArray();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                return null;
            }
        
            public String getRoot() {
                return root;
            }
        
            public void setRoot(String root) {
                this.root = root;
            }
        
            public static void main(String[] args)  {
        
                MyClassLoader classLoader = new MyClassLoader();
                classLoader.setRoot("E:\\temp");
        
                Class<?> testClass = null;
                try {
                    testClass = classLoader.loadClass("com.neo.classloader.Test2");
                    Object object = testClass.newInstance();
                    System.out.println(object.getClass().getClassLoader());
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } catch (InstantiationException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
```

自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在该类中对文件进行解密。由于这里只是演示，我并未对class文件进行加密，因此没有解密的过程。这里有几点需要注意：

- 1、这里传递的文件名需要是类的全限定性名称，即`com.paddx.test.classloading.Test`格式的，因为 defineClass 方法是按这种格式进行处理的。
- 2、最好不要重写loadClass方法，因为这样容易破坏双亲委托模式。
- 3、这类Test 类本身可以被 `AppClassLoader`类加载，因此我们不能把`com/paddx/test/classloading/Test.class`放在类路径下。否则，由于双亲委托机制的存在，会直接导致该类由`AppClassLoader`加载，而不会通过我们自定义类加载器来加载。

## 推荐阅读

- <https://blog.csdn.net/xyang81/article/details/7292380>
- <https://juejin.im/post/5c04892351882516e70dcc9b>
- <http://gityuan.com/2016/01/24/java-classloader/>
