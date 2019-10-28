---
title: hibernate_Embedded和@Embeddable
date: 2019-08-10 10:57:59
tags: java
---

在使用实体类生成对应的数据库表时，很多的时候都会遇到这种情况：在一个实体类中引用另外的实体类，一般遇上这种情况，我们使用@OneToOne、@OneToMany、@ManyToOne、@ManyToMany这4个注解比较多，但是好奇害死猫，除了这四个有没有别的使用情况，尤其是一个实体类要在多个不同的实体类中进行使用，而本身又不需要独立生成一个数据库表，这就是需要@Embedded、@Embeddable的时候了，下面分成4类来说明在一个实体类中引用另外的实体类的情况，具体的数据库环境是MySQL 5.7。

使用的两个实体类如下：

Address类
```java
public class Address implements Serializable{
    private static final long serialVersionUID = 8849870114128959929L;

    private String country;
    private String province;
    private String city;
    private String detail;
    
    //setter、getter}
```
Person类：
```java
@Entity
public class Person implements Serializable{
    private static final long serialVersionUID = 8849870114127659929L;

    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private Integer age;
    
    private Address address;
    
    //setter、getter
}
```
# 两个注解全不使用
当这两个注解都不使用时，那么两个实体类和上面的相同，那么生成的表结构如下： 

![](http://wumu.sciento.cn/img/20190810110112.png)


Address属性字段会映射成tinyblob类型的字段，这是用来存储不超过255字符的二进制字符串的数据类型，显然我们通常不会这么使用。

# 只使用@Embeddable
我们在Address实体类上加上@Embeddable注解，变成如下类：

```java
@Embeddable
public class Address implements Serializable{
    private static final long serialVersionUID = 8849870114128959929L;

    private String country;
    private String province;
    private String city;
    private String detail;
    
    //setter、getter
}
```
而Person实体类不变，生成的数据库表结构如下： 

![](http://wumu.sciento.cn/img/20190810110330.png)


可以看出这次是把Address中的字段映射成数据库列嵌入到Person表中了，而这些字段的类型和长度也使用默认值。如果我们在Address中的字段中设置列的相关属性，则会按照我们设定的值去生成，如下Address类：
```java
@Embeddable
public class Address implements Serializable{
    private static final long serialVersionUID = 8849870114128959929L;

    @Column(nullable = false)
    private String country;
    @Column(length = 30)
    private String province;
    @Column(unique = true)
    private String city;
    @Column(length = 50)
    private String detail;
    //setter、getter
}
```
生成的表结构如下：

 ![](http://wumu.sciento.cn/img/20190810110454.png)


我们在Address中配置的属性全部成功映射到Person表中。

# 只使用@Embedded
这里我们只在Person中使用@Embedded,如下：
```java
@Entity
public class Person implements Serializable{
    private static final long serialVersionUID = 8849870114127659929L;

    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private Integer age;
    
    @Embedded
    private Address address;
    
    //setter、getter
}
```
Adddress类和最开始的不同POJO类相同，此时生成的表结构如下： 

![](http://wumu.sciento.cn/img/20190810110619.png)


可以看出这个表结构和在Address中只使用@Embeddable注解时相同，在进入深一步试验，我们在Address中加入列属性，但是不使用@Embeddable注解会发生什么？ 
Address类如下：
```java
public class Address implements Serializable{
    private static final long serialVersionUID = 8849870114128959929L;

    @Column(nullable = false)
    private String country;
    @Column(length = 30)
    private String province;
    @Column(unique = true)
    private String city;
    @Column(length = 50)
    private String detail;
    //setter、getter
}
```
生成数据表结构如下： 

![](http://wumu.sciento.cn/img/20190810110728.png)


所以只使用@Embedded和只使用@Embeddable产生的效果是相同的。

# 两个注解全使用
既然单独使用@Embedded或者只使用@Embeddable都会产生作用，那么这两个都使用效果也一定是一样的，我们平时也是这么用的。所以在这部分我们就不演示和上面相同的效果了，而是说两个深入的话题。

## 覆盖@Embeddable类中字段的列属性
这里就要使用另外的两个注解@AttributeOverrides和@AttributeOverride，这两个注解是用来覆盖@Embeddable类中字段的属性的。

@AttributeOverrides：里面只包含了@AttributeOverride类型数组；
@AttributeOverride：包含要覆盖的@Embeddable类中字段名name和新增的@Column字段的属性；
使用如下： 
Person类如下：
```java
@Entity
public class Person implements Serializable{
    private static final long serialVersionUID = 8849870114127659929L;

    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private Integer age;
    
    @Embedded
    @AttributeOverrides({@AttributeOverride(name="country", column=@Column(name = "person_country", length = 25, nullable = false)),
                        @AttributeOverride(name="city", column = @Column(name = "person_city", length = 15))})
    private Address address;
    
    //setter、getter
}
```
Address类如下：
```java
@Embeddable
public class Address implements Serializable{
    private static final long serialVersionUID = 8849870114128959929L;

    @Column(nullable = false)
    private String country;
    @Column(length = 30)
    private String province;
    @Column(unique = true)
    private String city;
    @Column(length = 50)
    private String detail;
    //setter、getter
}
```
生成的数据表如下：

![](http://wumu.sciento.cn/img/20190810110901.png)

可以看出我们的@AttributeOverrides和@AttributeOverride两个注解起作用了。

## 多层嵌入实体类属性
上面所有的例子都是使用两层实体类嵌入，其实这种实体类的嵌入映射是可以使用多层的，具体的例子如下。 
我们新建立一个类Direction表示方位如下：
```java
@Embeddable
public class Direction implements Serializable{

    @Column(nullable = false)
    private Integer longitude;
    private Integer latitude;
}
```
Address如下：
```
@Embeddable
public class Address implements Serializable{
    private static final long serialVersionUID = 8849870114128959929L;

    @Column(nullable = false)
    private String country;
    @Column(length = 30)
    private String province;
    @Column(unique = true)
    private String city;
    @Column(length = 50)
    private String detail;
    
    @Embedded
    private Direction direction;
}
```
Person类如下：
```java
@Entity
public class Person implements Serializable{
    private static final long serialVersionUID = 8849870114127659929L;

    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private Integer age;
    
    @Embedded
    @AttributeOverrides({@AttributeOverride(name="direction.latitude", column=@Column(name = "person_latitude")),
                        @AttributeOverride(name="direction.longitude", column = @Column(name = "person_longitude"))})
    private Address address;
}
```
生成的数据表如下：

![](http://wumu.sciento.cn/img/20190810111050.png)

# 在上面需要注意如下几点：

在Person中定义Direction中的属性时，需要用”.”将所有相关的属性连接起来；
在Direction中longitude属性定义为not null，但是由于使用了@AttributeOverride注解，其中虽然没有定义null属性，但是这时使用的是默认的nullable属性，默认为true;

# 参考
> https://blog.csdn.net/lmy86263/article/details/52108130