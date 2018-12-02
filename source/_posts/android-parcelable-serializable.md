---
title: Android中的序列化：Parcelable和Serializable
date: 2018-11-30 22:44:44
tags: [Android]
---

### 概述

>序列化：将一个对象转换成`可存储`或`可传输`的状态。

<!--more-->

### Parcelable和Serializable的区别

#### 作用

> Serializable的作用是为了保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的。
>
> Parcelable的设计初衷是因为Serializable效率过慢，为了在程序内不同组件间以及不同Android程序间(AIDL)高效的传输数据而设计，这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。

#### 性能比较

- 在内存的使用中，Parcelable的性能方面要强于Serializable；
- Serializable序列化操作的时候会产生大量的临时变量(原因是使用了反射机制)，从而导致GC的频繁调用，因而性能比Parcelable差；
- Parcelable是以IBinder作为信息载体的。在内存上的开销比较小，因此内存直接进行数据传递的时候，Android推荐使用Parcelable；
- 在读写数据的时候，Parcelable是在内存中直接进行读写，而Serializable是通过IO流的形式将数据写入到硬盘上。

#### 选择

> Parcelable的性能比Serializable好，在内存开销方面较小，所以在内存间数据传输时推荐使用Parcelable，如Activity间传输数据，而Serializable可将数据持久化方便保存，所以在需要保存或网络传输数据时选择Serializable，因为Android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化。

### Serializable的使用

> 对于Serializable的使用，类只需要实现Serializable接口。

```java
public class User implements Serializable {
    
    /**
     * Java的序列化机制是通过判断类的serialVersionUID来验证版本一致性的。
     * 在进行反序列化时，JVM会把传来的字节流中的serialVersionUID与本地相应实体类的serialVersionUID进行比较，
     * 如果相同就认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常，即是InvalidCastException。
     * (如果没有显示定义，Java序列化机制会根据编译的Class自动生成一个serialVersionUID做为序列化版本比较用，如果Class文件没有发生变化，则serialVersionUID不变)
     */
    private static final long serialVersionUID = 123456789L;
    
    private String uid;
    private String userName;
    private int age;
}
```



### Parcelable的使用

> Parcelable的使用，需要实现`writeToParcel`、`describeContents`函数以及静态的`CREATOR`变量。

```java
public class User implements Parcelable {
    private String uid;
    private String userName;
    private int age;

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(this.uid);
        dest.writeString(this.userName);
        dest.writeInt(this.age);
    }

    public User() {
    }

    protected User(Parcel in) {
        this.uid = in.readString();
        this.userName = in.readString();
        this.age = in.readInt();
    }

    public static final Parcelable.Creator<User> CREATOR = new Parcelable.Creator<User>() {
        @Override
        public User createFromParcel(Parcel source) {
            return new User(source);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
}
```



### 参考链接

1. [Parcelable和Serializable的作用、效率、区别及选择](https://blog.csdn.net/ljx19900116/article/details/41699593)
2. [Android系统中Parcelable和Serializable的区别](https://greenrobot.me/devpost/android-parcelable-serializable/)
3. [序列化Serializable和Parcelable的理解和区别](https://www.jianshu.com/p/a60b609ec7e7)
4. [Android中Serializable和Parcelable序列化对象详解](https://www.cnblogs.com/yezhennan/p/5527506.html)
5. [Serializable](https://developer.android.com/reference/java/io/Serializable)
6. [Parcelable](https://developer.android.com/reference/android/os/Parcelable)
7. [java类中serialversionuid 作用 是什么?](https://www.cnblogs.com/duanxz/p/3511695.html)

