---
title: Java反射学习记录
date: 2020-05-13 11:38:32
categories:
    - Java基础
tags: 
    - Java
cover: https://i.loli.net/2020/05/26/vblkX6NYjFLBH3h.png
---
# Java反射学习记录

### 1. 文章结构
![Java的反射机制.png](https://i.loli.net/2020/05/26/vblkX6NYjFLBH3h.png)

### 2. 反射是什么？

反射是提供了能够动态操作Java代码的工具集程序。有一下几种能力
1. 在运行时分析类的能力
2. 在运行时查看对象
3. 实现通用的数组操作代码
参考：《[Java核心技术：卷1](https://book.douban.com/subject/26880667/)》第190页-反射章节

下面的简单的反射代码例子可以感受一下（摘取自《[Thinking in Java](https://book.douban.com/subject/2130190/)》反射章节）

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Method;
import java.util.regex.Pattern;

class Person {
    private String name;
    private int age;
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void sayHello() {
        System.out.println("My name is " + name + " and " + age +" years old");
    }

    public String getName() {
        return name;
    }

    public String getAge() {
        return age;
    }
}

public class ShowMethods {
    private static final String usage = "usage: \n"+
    "ShowMethods qualified.class.name\n"+
    "To Show all methods in class or:" +
    "ShowMethods qualified.class.name word\n" +
    "To search for methods involving word";

    private static final Pattern pattern = Pattern.compile("\\w+\\.");
    public static void main(String[] args) {
        if (args.length < 1) {
            System.out.println(usage);
            System.exit(0);
        }

        int lines = 0;
        try {
            Class<?> c = Class.forName(args[0]);
            Method[] methods = c.getMethods();
            c.getMethod(name, parameterTypes)
            Constructor[] ctors = c.getConstructors();
            if (args.length == 1) {
                System.out.println("===========下面是方法===========");
                for (Method method : methods) {
                    System.out.println(pattern.matcher(method.toString()).replaceAll(""));
                }
                System.out.println("===========下面是构造器===========");
                for(Constructor ctor : ctors) {
                    System.out.println(pattern.matcher(ctor.toString()).replaceAll(""));
                }
                lines = methods.length + ctors.length;
            }else {
                System.out.println("===========下面是方法===========");
                for (Method method : methods) {
                    if (method.toString().indexOf(args[1]) != -1) {
                        System.out.println(pattern.matcher(method.toString()).replaceAll(""));
                        lines++;
                    }
                }
                System.out.println("===========下面是构造器===========");
                for(Constructor ctor : ctors) {
                    if (ctor.toString().indexOf(args[1]) != -1) {
                        System.out.println(pattern.matcher(ctor.toString()).replaceAll(""));
                    }
                    lines++;
                }

            }
        }catch(ClassNotFoundException e) {
            System.out.println("No such class: " + e);
        }
    }
}
```

### 3. 反射的应用

反射在平时的业务开发中其实自己去实现的频率并不高，更多的是用注解的形式去使用反射机制，而且大部分开发常用的注解都框架都已经实现，只有很少部分都注解需要自己去实现。如果你学习偏向于业务程序的开发，而不是库的开发。可以选择性的学习反射的相关知识。

有如下的应用：

1. 用于自定义注解的读取
2. 外部的包和库没有源码的时候，可以用反射的方式去间接的查看类的定义
3. 工具，库，框架的开发过程中经常使用



### 4. 反射中常用到的中间类以及类上面的方法

使用反射常见的过程是：

1. 获取目标对象
2. 功过getClass()方法获取改对象实例的Class类
3. 在Class上通过getDeclaredFields()，getDeclaredMethods(), getConstructor()等方法获取你感兴趣的部分
4. 通过getName()获取目标部分定义的名字，Field.getType()获取定义的类型等
5. 通过获取到感兴趣的类结构信息后将信息做相对应的处理

#### 4.1. Construct对象

使用反射的第一个入口就是获取实例对象的Class类对象。

class.getContructs()获取构造器列表



#### 4.2 Method对象

class.getMethods()和class.getDeclaredmethods()可以获取改类上面的定义的方法，类上定义的方法等类型是Method对象

getMetheds() 和 getDeclaredMethods()的区别是，前者只会返回权限修饰符是public的方法，而后者是访问类上定义的所有的方法，但不包括父类上面的方法。



#### 4.3. Field对象

class.getFields()或class.getDeclaredFields()。返回类上定义的属性列表。两者却别和上面相同。通过Field对象我们可以访问到类上属性的定义

Field.getType()方法可以获取改域定义的数据类型



#### 4.4. Modifier对象

此对象为访问修饰符对象，常用的静态方法是`toString(int modifiers)`，返回构造器，域，方法上的访问修饰符的字符串形式。在构造器，域，方法上获取访问修饰符的通用方法是getModifiers()

```java
int modifiers = Class.getMethods()[0].getModifiers();
Modifier.toString(modifiers); // => "public final"之类的
```

还是有 isPublic(), isPrivate()，。。。。以此类推。

#### 4.4. Construct，Field，Method上的通用方法

- getName()方法，此方法可以获取定义的对象的代码名字字符串。
- getModifiers()方法，获取访问修饰符



### 5. 使用案例

[参考原文](https://www.bilibili.com/read/cv4802402)

需求描述：定义一个注解 `@Length(int min, int max, String message)`用在类的字符串属性上，定义该字符串的长度在定义的范围内。

#### 5.1. 第一步：先定义注解

```java
@Target({ ElementType.Field })
@Retention(RetentionPolicy.RUNTIME)
public @interface Length {
  int min;
  int max;
  String message;
}
```

因为是用在类属性上，所以注解的的@Target元注解为Field，还有因为注解需要在运行时被读取，所以@Retention元注解要为runtime。



#### 5.2.  第二步：获取注解并进行验证

 ```java
 public static String validate(Object target) throws IllegalAccessException {
   // 为了方便演示，健壮性代码就不写了
   Field[] declaredFields = target.getClass().getDeclaredFields();
   for (Field field : declaredFields) {
     boolean isLengthAnnotation = field.isAnnotationPresent(Length.class);
     if (isLengthAnnotation) {
       Length length = field.getAnnotation(Length.class);
       field.setAccessible(true); // 为了也能够获取访问修饰符不为public的属性的值
       int valueActuallyLength = ((String)field.get(target)).length();
       if (valueActuallyLength < length.min() || valueActuallyLength > length.max()) {
         return length.message();
       }
     }
   }
   return null;
 }
 ```



### 参考资料

- [《Java核心技术：卷1》](https://book.douban.com/subject/26880667/)

- [《Thinking in Java》](https://book.douban.com/subject/2130190/)

- [ 注解案例](https://www.bilibili.com/read/cv4802402)


