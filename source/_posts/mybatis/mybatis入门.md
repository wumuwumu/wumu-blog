---
title: mybatis入门
date: 2020-9-19 14:00:00
---

# 添加依赖

```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>x.x.x</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.44</version>
</dependency>
```

# 创建配置文件

mybatis-config.xml

配置文件的标签顺序不能打乱，不然会报错。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties resource="top/sciento/wumu/jdbc/mybatis/db.properties"></properties>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${driver}"/>
                <property name="url" value="${url}"/>
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <package name="top.sciento.wumu.jdbc.mybatis.mapper"/>
    </mappers>
</configuration>
```

db.properties

```properties
driver=com.mysql.jdbc.Driver
url=jdbc:mysql://:3306/test
username=
password=
```

# 编写执行文件

```java

package top.sciento.wumu.jdbc.mybatis;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import top.sciento.wumu.jdbc.mybatis.entity.User;
import top.sciento.wumu.jdbc.mybatis.mapper.UserMapper;

import java.io.IOException;
import java.io.InputStream;
import java.io.Reader;
import java.util.List;

public class MybatisRunner {

    public static void main(String[] args) throws IOException {
        System.out.println(MybatisRunner.class.getResource(""));
        InputStream reader = MybatisRunner.class.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(reader);
        Configuration configuration  = sessionFactory.getConfiguration();
        // 默认是不会提交的，需要手动提交
        SqlSession sqlSession = sessionFactory.openSession();
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        List<User> userList = userMapper.selectList();
        System.out.println(userList);
        User user  = new User();
        user.setName("wumu");
        user.setAge(12);
        int id = userMapper.insert(user);
        System.out.println(id);
        System.out.println(user);
    }
}

```

```java
public interface UserMapper {
    List<User> list();

    @SelectProvider(value = UserSqlBuilder.class,method = "selectList")
    List<User> selectList();

    // 这里使用动态sql
    @InsertProvider(value = UserSqlBuilder.class,method = "insert")
//    @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
    @SelectKey(statement = "select last_insert_id()", keyProperty = "id", before = false, resultType = int.class)
    int insert(User user);
    
}

```

```java
public class UserSqlBuilder {
    public static String selectList() {
        return new SQL().SELECT("id","name","age").FROM("base_user").toString();
    }


    public static String insert(User user){
//        return new SQL().INSERT_INTO("base_user").INTO_COLUMNS("name","age")
//                .INTO_VALUES(user.getName(),String.valueOf(user.getAge())).toString();
        return new SQL().INSERT_INTO("base_user").VALUES("name","#{name}")
                .VALUES("age","#{age}").toString();
    }
}

```

```java
@Data
public class User {
    private Integer id;
    private String name;
    private Integer age;
}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!-- namespace属性是名称空间，必须唯一 -->
<mapper namespace="top.sciento.wumu.jdbc.mybatis.mapper.UserMapper">

    <!-- resultMap标签:映射实体与表
         type属性：表示实体全路径名
         id属性：为实体与表的映射取一个任意的唯一的名字
    -->
    <resultMap type="top.sciento.wumu.jdbc.mybatis.entity.User" id="UserMap">
        <!-- id标签:映射主键属性
             result标签：映射非主键属性
             property属性:实体的属性名
             column属性：表的字段名
        -->
        <id property="id" column="id"/>
        <result property="name" column="name"/>
        <result property="age" column="age"/>
    </resultMap>
    <select id="list" resultMap="UserMap">
        select * from base_user
    </select>


</mapper>
```

# 知识分析

## 返回主键

1、使用options

options可以配置sql的大部分属性，对应着我们标签`<select>`上写的相关属性。

| --         | --     |                | 描述                                                         |
| ---------- | ------ | -------------- | ------------------------------------------------------------ |
| `@Options` | `方法` | 映射语句的属性 | 该注解允许你指定大部分开关和配置选项，它们通常在映射语句上作为属性出现。与在注解上提供大量的属性相比，`Options` 注解提供了一致、清晰的方式来指定选项。属性：`useCache=true`、`flushCache=FlushCachePolicy.DEFAULT`、`resultSetType=DEFAULT`、`statementType=PREPARED`、`fetchSize=-1`、`timeout=-1`、`useGeneratedKeys=false`、`keyProperty=""`、`keyColumn=""`、`resultSets=""`, `databaseId=""`。注意，Java 注解无法指定 `null` 值。因此，一旦你使用了 `Options` 注解，你的语句就会被上述属性的默认值所影响。要注意避免默认值带来的非预期行为。 The `databaseId`(Available since 3.5.5), in case there is a configured `DatabaseIdProvider`, the MyBatis use the `Options` with no `databaseId` attribute or with a `databaseId` that matches the current one. If found with and without the `databaseId` the latter will be discarded.         注意：`keyColumn` 属性只在某些数据库中有效（如 Oracle、PostgreSQL 等）。要了解更多关于 `keyColumn` 和 `keyProperty` 可选值信息，请查看“insert, update 和 delete”一节。 |

2、使用SelectKey

对应着SelectKey标签

| --           | --     | --            | -                                                            |
| ------------ | ------ | ------------- | ------------------------------------------------------------ |
| `@SelectKey` | `方法` | `<selectKey>` | 这个注解的功能与 `<selectKey>` 标签完全一致。该注解只能在 `@Insert` 或 `@InsertProvider` 或 `@Update` 或 `@UpdateProvider` 标注的方法上使用，否则将会被忽略。如果标注了 `@SelectKey` 注解，MyBatis 将会忽略掉由 `@Options` 注解所设置的生成主键或设置（configuration）属性。属性：`statement` 以字符串数组形式指定将会被执行的 SQL 语句，`keyProperty` 指定作为参数传入的对象对应属性的名称，该属性将会更新成新的值，`before` 可以指定为 `true` 或 `false` 以指明 SQL 语句应被在插入语句的之前还是之后执行。`resultType` 则指定 `keyProperty` 的 Java 类型。`statementType` 则用于选择语句类型，可以选择 `STATEMENT`、`PREPARED` 或 `CALLABLE` 之一，它们分别对应于 `Statement`、`PreparedStatement` 和 `CallableStatement`。默认值是 `PREPARED`。 The `databaseId`(Available since 3.5.5), in case there is a configured `DatabaseIdProvider`, the MyBatis will use a statement with no `databaseId` attribute or with a `databaseId` that matches the current one. If found with and without the `databaseId` the latter will be discarded. |

描述：

@SelctKey(statement="SQL语句",keyProperty="将SQL语句查询结果存放到keyProperty中去",before="true表示先查询再插入，false反之",resultType=int.class)
其中：

- statement是要运行的SQL语句，它的返回值通过resultType来指定
- before表示查询语句statement运行的时机
- keyProperty表示查询结果赋值给代码中的哪个对象，keyColumn表示将查询结果赋值给数据库表中哪一列
- keyProperty和keyColumn都不是必需的，有没有都可以
- before=true，插入之前进行查询，可以将查询结果赋给keyProperty和keyColumn，赋给keyColumn相当于更改数据库
- befaore=false，先插入，再查询，这时只能将结果赋给keyProperty
- 赋值给keyProperty用来“读”数据库，赋值给keyColumn用来写数据库
- selectKey的两大作用：1、生成主键；2、获取刚刚插入数据的主键。
- 使用selectKey，并且使用MySQL的last_insert_id()函数时，before必为false，也就是说必须先插入然后执行last_insert_id()才能获得刚刚插入数据的ID。

## maven打包xml文件

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>8</source>
                <target>8</target>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-war-plugin</artifactId>
            <version>2.1.1</version>
        </plugin>
    </plugins>
    <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>false</filtering>
        </resource>
        <resource>
            <directory>src/main/resources</directory>
        </resource>
    </resources>
</build>
```







