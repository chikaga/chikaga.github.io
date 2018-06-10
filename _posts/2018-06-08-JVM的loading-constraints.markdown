---
layout: post
title: JVM的loading constraints
date: 2018-06-06 20:27:44
categories: JVM
---

参考：《深入Java虚拟机》第9章，略作修改

代码目录结构：

```
+---cracker
|   |   Cracker.class
|   |   Cracker.java
|   |
|   \---io
|       \---github
|           \---strang
|               \---greeter
|                       Spoofed.class
|
\---io
    \---github
        \---strang
            \---greeter
                    Delegated.class
                    Delegated.java
                    Greet.class
                    Greet.java
                    Greeter.class
                    Greeter.java
                    GreeterClassLoader.class
                    GreeterClassLoader.java
                    Spoofed.class
                    Spoofed.java
```

代码：

``` java
import io.github.strang.greeter.Greeter;
import io.github.strang.greeter.Spoofed;
import io.github.strang.greeter.Delegated;

public class Cracker implements Greeter {

	static {
		System.out.println("class Cracker loaded by: " + Cracker.class.getClassLoader());
	}

    public void greet() {

        Spoofed spoofed = Delegated.getSpoofed(); // 由System Class Loader加载

        System.out.println("secret val = "
            + spoofed.giveMeFive());
    }
}
```

``` java
package io.github.strang.greeter;

public class Greet {

    // Arguments to this application:
    //     args[0] - path name of directory in which class files
    //               for greeters are stored
    //     args[1], args[2], ... - class names of greeters to load
    //               and invoke the greet() method on.
    //
    // All greeters must implement the com.artima.greeter.Greeter
    // interface.
    //
    static public void main(String[] args) {

        if (args.length <= 1) {
            System.out.println(
                "Enter base path and greeter class names as args.");
            return;
        }

        GreeterClassLoader gcl = new GreeterClassLoader(args[0]);

        for (int i = 1; i < args.length; ++i) {
            try {

                // Load the greeter specified on the command line
                Class c = gcl.loadClass(args[i]);

                // Instantiate it into a greeter object
                Object o = c.newInstance();

                // Cast the Object ref to the Greeter interface type
                // so greet() can be invoked on it
                Greeter greeter = (Greeter) o;

                // Greet the world in this greeter's special way
                greeter.greet();
            }
            catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

``` java
package io.github.strang.greeter;

import java.io.*;
import java.util.Hashtable;

public class GreeterClassLoader extends ClassLoader {

    // basePath gives the path to which this class
    // loader appends "/.class" to get the
    // full path name of the class file to load
    private String basePath;

    public GreeterClassLoader(String basePath) {

        this.basePath = basePath;
    }

    public synchronized Class loadClass(String className,
        boolean resolveIt) throws ClassNotFoundException {

        Class result;
        byte classData[];

        // Check the loaded class cache
        result = findLoadedClass(className);
        if (result != null) {
            // Return a cached class
            return result;
        }

        // If Spoofed, don't delegate
        if (!className.contains("Spoofed")) {

            // Check with the system class loader
            try {
                result = super.findSystemClass(className);
                // Return a system class
                return result;
            }
            catch (ClassNotFoundException e) {
            }
        }

        // Don't attempt to load a system file except through
        // the primordial class loader
        if (className.startsWith("java.")) {
            throw new ClassNotFoundException();
        }

        // Try to load it from the basePath directory.
        classData = getTypeFromBasePath(className);
        if (classData == null) {
            System.out.println("GCL - Can't load class: "
                + className);
            throw new ClassNotFoundException();
        }

        // Parse it
        result = defineClass(className, classData, 0,
            classData.length);
        if (result == null) {
            System.out.println("GCL - Class format error: "
                + className);
            throw new ClassFormatError();
        }

        if (resolveIt) {
            resolveClass(result);
        }

        // Return class from basePath directory
        return result;
    }

    private byte[] getTypeFromBasePath(String typeName) {
        System.out.println("get type: " + typeName + " from base path: " + basePath);

        FileInputStream fis;
        String fileName = basePath + File.separatorChar
            + typeName.replace('.', File.separatorChar)
            + ".class";

        try {
            fis = new FileInputStream(fileName);
        }
        catch (FileNotFoundException e) {
            return null;
        }

        BufferedInputStream bis = new BufferedInputStream(fis);

        ByteArrayOutputStream out = new ByteArrayOutputStream();

        try {
            int c = bis.read();
            while (c != -1) {
                out.write(c);
                c = bis.read();
            }
        }
        catch (IOException e) {
            return null;
        }

        return out.toByteArray();
    }
}
```

``` java
package io.github.strang.greeter;

public class Delegated {

    static {
        System.out.println("class Delegated loaded by: " + Delegated.class.getClassLoader());
    }

    public static Spoofed getSpoofed() {

        return new Spoofed();
    }
}
```

``` java
package io.github.strang.greeter;

public class Spoofed {

    private int secretValue = 42;

    public int giveMeFive() {
        return 5;
    }

    static {
        System.out.println("class: Spoofed loaded by " + Spoofed.class.getClassLoader());
    }
}
```

``` java
package io.github.strang.greeter;

public interface Greeter {

    void greet();
}
```

1.首先执行编译。

``` bash
javac io/github/strang/greeter/*.java cracker/*.java
```

2.将`io/github/strang/greeter/`目录下的`Spoofed.class`拷贝到`cracker/io/github/strang/greeter/`目录下。

3.运行。

从输出来看，Cracker被GreeterClassLoader加载，Delegated被系统类加载器加载，Delegated使用自己的定义类加载器（也就是系统类加载器）加载Spoofed。随后，GreeterClassLoader又试图加载Spoofed，却在GreeterClassLoader的第59行执行defineClass的时候抛出了java.lang.LinkageError。

```
 java io.github.strang.greeter.Greet cracker Cracker
get type: Cracker from base path: cracker
class Cracker loaded by: io.github.strang.greeter.GreeterClassLoader@4e25154f
class Delegated loaded by: sun.misc.Launcher$AppClassLoader@2a139a55
class: Spoofed loaded by sun.misc.Launcher$AppClassLoader@2a139a55
get type: io.github.strang.greeter.Spoofed from base path: cracker
Exception in thread "main" java.lang.LinkageError: loader constraint violation: loader (instance of io/github/strang/greeter/GreeterClassLoader) previously initiated loading for a different type with name "io/github/strang/greeter/Spoofed"
        at java.lang.ClassLoader.defineClass1(Native Method)
        at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
        at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
        at io.github.strang.greeter.GreeterClassLoader.loadClass(GreeterClassLoader.java:59)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        at Cracker.greet(Cracker.java:20)
        at io.github.strang.greeter.Greet.main(Greet.java:38)

```

4.结果分析。

需要重点关注的是`Cracker.java`。对`Cracker.class`反编译，`javap -v cracker/Cracker.class`，查看greet方法的字节码。执行到第6行时，JVM将解析常量池中指向Delegated类的getSpoofed方法的符号引用，由于Delegated的定义类加载器和当前类Cracker的不是同一个，而且getSpoofed方法的返回值类型为Spoofed，此时JVM将记录以下装载约束：

* 被系统类加载器记录为初始类加载器的Spoofed类必须和被GreeterClassLoader记录为初始类加载器的Spoofed类是同一个类(二者的定义类加载器必须是同一个)。

当字节码执行到第15行时，JVM尝试解析常量池中指向Delegated类的giveMeFive方法，而Spoofed并不处于当前类加载器（即GreeterClassLoader）的命名空间中，因此GreeterClassLoader试图加载Spoofed，此时违背了上述的装载约束。

```
  public void greet();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=2, args_size=1
         0: invokestatic  #2                  // Method io/github/strang/greeter/Delegated.getSpoofed:()Lio/github/strang/greeter/Spoofed;
         3: astore_1
         4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: new           #4                  // class java/lang/StringBuilder
        10: dup
        11: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
        14: ldc           #6                  // String secret val =
        16: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        19: aload_1
        20: invokevirtual #8                  // Method io/github/strang/greeter/Spoofed.giveMeFive:()I
        23: invokevirtual #9                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        26: invokevirtual #10                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        29: invokevirtual #11                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        32: return
      LineNumberTable:
        line 17: 0
        line 19: 4
        line 20: 20
        line 19: 29
        line 21: 32
```
