---
title: Java中的transient关键字
date: 2018-12-01 00:28:17
tags: [Java]
---

### 概述

>在Java中，一个类只要实现Serializable接口，这个类的对象就可以被序列化，这种序列化模式为开发者提供了很多便利，我们可以不必关心具体序列化的过程，只要这个类实现了Serializable接口，这个类的所有属性都会自动序列化。但是有时我们需要让类的某些属性不被序列化，如密码这类信息，为了安全起见，不希望在网络操作中被传输或者持久化到本地。只要在相应的属性前加上`transient`关键字，就可以实现部分属性不被序列化，该属性的生命周期仅存于调用者的内存中而不会写入到磁盘持久化。

<!--more-->

### transient的使用

```java
public class TransientTest {

    public static void main(String[] args) {

        User user = new User();
        user.setUsername("Github");
        user.setPassword("123456");

        System.out.println("read before Serializable: ");
        System.out.println("username: " + user.getUsername());
        System.err.println("password: " + user.getPassword());

        try {
            ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream("user.txt"));
            os.writeObject(user); // 将User对象写进文件
            os.flush();
            os.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            ObjectInputStream is = new ObjectInputStream(new FileInputStream("user.txt"));
            user = (User) is.readObject(); // 从流中读取User的数据
            is.close();

            System.out.println("\nread after Serializable: ");
            System.out.println("username: " + user.getUsername());
            System.err.println("password: " + user.getPassword());

        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

public class User implements Serializable {
    private static final long serialVersionUID = 1234567890L;

    private String username;
    private transient String password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

**运行结果**：

![runResult](java-transient/runResult.png)

### transient修饰静态变量

```java
public class TransientTest {

    public static void main(String[] args) {

        User user = new User();
        user.setUsername("Github");
        user.setPassword("123456");

        System.out.println("read before Serializable: ");
        System.out.println("username: " + user.getUsername());
        System.err.println("password: " + user.getPassword());

        try {
            ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream("user.txt"));
            os.writeObject(user); // 将User对象写进文件
            os.flush();
            os.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        try {
            // 在反序列化前盖板username的值
            user.setUsername("Tom");

            ObjectInputStream is = new ObjectInputStream(new FileInputStream("user.txt"));
            user = (User) is.readObject(); // 从流中读取User的数据
            is.close();

            System.out.println("\nread after Serializable: ");
            System.out.println("username: " + user.getUsername());
            System.err.println("password: " + user.getPassword());

        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

public class User implements Serializable {
    private static final long serialVersionUID = 1234567890L;

    private static String username;
    private transient String password;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

**运行结果**：

![static](java-transient/static.png)

### 总结

1. 一旦变量被transient修饰，变量将不再是对象持久化的一部分，该变量内容在序列化后无法获得访问。
2. transient关键字只能修饰变量，而不能修饰方法和类。**本地变量是不能被transient关键字修饰的。**变量如果是用户自定义类变量，则该类需要实现Serializable接口。
3. 一个**静态变量**不管是否被transient修饰，均不能被序列化。

### 参考链接

1. [Java transient关键字使用小记](http://www.cnblogs.com/lanxuezaipiao/p/3369962.html)