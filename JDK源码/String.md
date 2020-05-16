## 基于1.8版本

#### String.intern()

> 当调用intern()方法时,如果常量池已存在该字符串,则返回池中的字符串
>
> 如果不存在,则将此字符串添加到常量池,并返回**字符串的引用**

> 带有双引号的字符串,会调用dlc命令,将其添加到常量池中
>
> intern的作用是把new出来的字符串的引用添加到stringtable中，java会先计算string的hashcode，查找stringtable中是否已经有string对应的引用了，如果有返回引用（地址），然后没有把字符串的地址放到stringtable中，并返回字符串的引用（地址)

#### 双引号创建字符串

* 判断这个字符串是已经在常量池
  * 存在
    * 引用类型,返回引用地址指向的堆空间对象
    * 常量,直接返回常量池常量
  * 不存在
    * 在常量池创建该常量,并返回常量

```java
String a = "abc";
String b = "abc";
System.out.println(a == b); // true
```

#### new String创建字符串

* 首先在堆上创建对象(无论堆上是否存在相同字面的对象)
* 判断常量池上是否存在字符串的字面量
  * 不存在
    * 在常量池创建常量
  * 存在
    * 不做任何操作

```java
String a = new String("abc");
// a是堆对象的引用,a.intern()是常量池中对象,因为在new String("abc")时,已经将abc添加到常量池了
System.out.println(a == a.intern()); //false
```

#### 双引号相加

* 判断这两个常量,相加后的常量在常量池上是否存在
  * 不存在
    * 在常量池中创建相应的常量
  * 存在
    * 引用类型,直接返回引用地址指向的堆空间对象
    * 常量,直接返回常量池常量

```java
String a = "a" + "b"; // 会在常量池中创建三个对象 a,b,ab
String b = "ab"; // 常量池中已存在
System.out.println(a == b);//true 
```

#### 两个new String 相加

* 创建这两个对象及相加后的对象
* 判断常量池是否存在这两个对象的字面量常量
  * 存在, 不做任何操作
  * 不存在,在常量池创建常量

```java
//常量池中创建 aa bb
String a = new String("aa") + new String("bb");
//在创建对象b
String b = new String("a") + new String("a");
//b.intern()时,已经通过new String("aa") 创建了常量,所以直接返回常量值
System.out.println(b == b.intern()); // false
//如果把第一行代码注释,为true 因为b.intern时,常量池中没有,直接将当前对象b的引用放入到常量池,因此b=b.intern
```

#### 双引号与new String相加

* 创建两个对象,一个是new String的对象,一个是相加后的对象
* 判断双引号常量与new String相加的字面量是否在常量池中存在
  * 存在, 不做任何操作
  * 不存在,在常量池中创建常量

```java
String a = "ab";
String b = "a" + new String("b");
System.out.println(a == b.intern());//true
```

#### String.intern

* 判断这个常量是否存在常量池
  * 存在
    * 引用类型,返回引用地址指向的堆空间对象
    * 常量,直接返回常量池常量
  * 不存在
    * 将当前对象引用复制到常量池,并返回当前对象的引用

##### 只在常量池上创建常量

```java
String a1 = "AA";  
```

##### 只在堆上创建对象

```java
String a2 = new String("A") + new String("A");
```

##### 在堆上创建对象,在常量池上创建引用

```java
String b = new String("A") + new String("A");
b.intern();//在常量池上创建引用
```

#### 字符串常量池、Class常量池、运行时常量池

