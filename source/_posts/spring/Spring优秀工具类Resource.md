---
title: Spring优秀工具类Resource
date: 2020-10-12
---

<http://www.blogjava.net/coolingverse/articles/149364.html>

文件资源的操作是应用程序中常见的功能，如当上传一个文件后将其保存在特定目录下，从指定地址加载一个配置文件等等。我们一般使用 JDK 的 I/O 处理类完成这些操作，但对于一般的应用程序来说，JDK 的这些操作类所提供的方法过于底层，直接使用它们进行文件操作不但程序编写复杂而且容易产生错误。相比于 JDK 的 File，Spring 的 Resource 接口（资源概念的描述接口）抽象层面更高且涵盖面更广，Spring 提供了许多方便易用的资源操作工具类，它们大大降低资源操作的复杂度，同时具有更强的普适性。这些工具类不依赖于 Spring 容器，这意味着您可以在程序中象一般普通类一样使用它们。

加载文件资源

Spring 定义了一个 org.springframework.core.io.Resource 接口，Resource 接口是为了统一各种类型不同的资源而定义的，Spring 提供了若干 Resource 接口的实现类，这些实现类可以轻松地加载不同类型的底层资源，并提供了获取文件名、URL 地址以及资源内容的操作方法。

**访问文件资源**

假设有一个文件地位于 Web 应用的类路径下，您可以通过以下方式对这个文件资源进行访问：

- 通过 FileSystemResource 以文件系统绝对路径的方式进行访问；
- 通过 ClassPathResource 以类路径的方式进行访问；
- 通过 ServletContextResource 以相对于Web应用根目录的方式进行访问。

相比于通过 JDK 的 File 类访问文件资源的方式，Spring 的 Resource 实现类无疑提供了更加灵活的操作方式，您可以根据情况选择适合的 Resource 实现类访问资源。下面，我们分别通过 FileSystemResource 和 ClassPathResource 访问同一个文件资源：


**清单 1. FileSourceExample**

```
`package com.baobaotao.io; import java.io.IOException; import java.io.InputStream; import org.springframework.core.io.ClassPathResource; import org.springframework.core.io.FileSystemResource; import org.springframework.core.io.Resource; public class FileSourceExample {     public static void main(String[] args) {         try {             String filePath =              "D:/masterSpring/chapter23/webapp/WEB-INF/classes/conf/file1.txt";             // ① 使用系统文件路径方式加载文件             Resource res1 = new FileSystemResource(filePath);              // ② 使用类路径方式加载文件             Resource res2 = new ClassPathResource("conf/file1.txt");             InputStream ins1 = res1.getInputStream();             InputStream ins2 = res2.getInputStream();             System.out.println("res1:"+res1.getFilename());             System.out.println("res2:"+res2.getFilename());         } catch (IOException e) {             e.printStackTrace();         }     } } `
```

 

在获取资源后，您就可以通过 Resource 接口定义的多个方法访问文件的数据和其它的信息：如您可以通过 getFileName() 获取文件名，通过 getFile() 获取资源对应的 File 对象，通过 getInputStream() 直接获取文件的输入流。此外，您还可以通过 createRelative(String relativePath) 在资源相对地址上创建新的资源。

在 Web 应用中，您还可以通过 ServletContextResource 以相对于 Web 应用根目录的方式访问文件资源，如下所示：

```
`<%@ page language="java" contentType="text/html; charset=utf-8" pageEncoding="utf-8"%> <jsp:directive.page import="     org.springframework.web.context.support.ServletContextResource"/> <jsp:directive.page import="org.springframework.core.io.Resource"/> <%     // ① 注意文件资源地址以相对于 Web 应用根路径的方式表示     Resource res3 = new ServletContextResource(application,          "/WEB-INF/classes/conf/file1.txt");     out.print(res3.getFilename()); %> `
```

 

对于位于远程服务器（Web 服务器或 FTP 服务器）的文件资源，您则可以方便地通过 UrlResource 进行访问。

为了方便访问不同类型的资源，您必须使用相应的 Resource 实现类，是否可以在不显式使用 Resource 实现类的情况下，仅根据带特殊前缀的资源地址直接加载文件资源呢？Spring 提供了一个 ResourceUtils 工具类，它支持"classpath:"和"file:"的地址前缀，它能够从指定的地址加载文件资源，请看下面的例子：


**清单 2. ResourceUtilsExample**

```
`package com.baobaotao.io; import java.io.File; import org.springframework.util.ResourceUtils; public class ResourceUtilsExample {     public static void main(String[] args) throws Throwable{         File clsFile = ResourceUtils.getFile("classpath:conf/file1.txt");         System.out.println(clsFile.isFile());          String httpFilePath = "file:D:/masterSpring/chapter23/src/conf/file1.txt";         File httpFile = ResourceUtils.getFile(httpFilePath);         System.out.println(httpFile.isFile());             } } `
```

 

ResourceUtils 的 getFile(String resourceLocation) 方法支持带特殊前缀的资源地址，这样，我们就可以在不和 Resource 实现类打交道的情况下使用 Spring 文件资源加载的功能了。

**本地化文件资源**

本地化文件资源是一组通过本地化标识名进行特殊命名的文件，Spring 提供的 LocalizedResourceHelper 允许通过文件资源基名和本地化实体获取匹配的本地化文件资源并以 Resource 对象返回。假设在类路径的 i18n 目录下，拥有一组基名为 message 的本地化文件资源，我们通过以下实例演示获取对应中国大陆和美国的本地化文件资源：


**清单 3. LocaleResourceTest**

```
`package com.baobaotao.io; import java.util.Locale; import org.springframework.core.io.Resource; import org.springframework.core.io.support.LocalizedResourceHelper; public class LocaleResourceTest {     public static void main(String[] args) {         LocalizedResourceHelper lrHalper = new LocalizedResourceHelper();         // ① 获取对应美国的本地化文件资源         Resource msg_us = lrHalper.findLocalizedResource("i18n/message", ".properties",          Locale.US);         // ② 获取对应中国大陆的本地化文件资源         Resource msg_cn = lrHalper.findLocalizedResource("i18n/message", ".properties",          Locale.CHINA);         System.out.println("fileName(us):"+msg_us.getFilename());          System.out.println("fileName(cn):"+msg_cn.getFilename());     } } `
```

 

虽然 JDK 的 java.util.ResourceBundle 类也可以通过相似的方式获取本地化文件资源，但是其返回的是 ResourceBundle 类型的对象。如果您决定统一使用 Spring 的 Resource 接表征文件资源，那么 LocalizedResourceHelper 就是获取文件资源的非常适合的帮助类了。

文件操作

在使用各种 Resource 接口的实现类加载文件资源后，经常需要对文件资源进行读取、拷贝、转存等不同类型的操作。您可以通过 Resource 接口所提供了方法完成这些功能，不过在大多数情况下，通过 Spring 为 Resource 所配备的工具类完成文件资源的操作将更加方便。

**文件内容拷贝**

第一个我们要认识的是 FileCopyUtils，它提供了许多一步式的静态操作方法，能够将文件内容拷贝到一个目标 byte[]、String 甚至一个输出流或输出文件中。下面的实例展示了 FileCopyUtils 具体使用方法：


**清单 4. FileCopyUtilsExample**

往往我们都通过直接操作 InputStream 读取文件的内容，但是流操作的代码是比较底层的，代码的面向对象性并不强。通过 FileCopyUtils 读取和拷贝文件内容易于操作且相当直观。如在 ① 处，我们通过 FileCopyUtils 的 copyToByteArray(File in) 方法就可以直接将文件内容读到一个 byte[] 中；另一个可用的方法是 copyToByteArray(InputStream in)，它将输入流读取到一个 byte[] 中。

如果是文本文件，您可能希望将文件内容读取到 String 中，此时您可以使用 copyToString(Reader in) 方法，如 ② 所示。使用 FileReader 对 File 进行封装，或使用 InputStreamReader 对 InputStream 进行封装就可以了。

FileCopyUtils 还提供了多个将文件内容拷贝到各种目标对象中的方法，这些方法包括：

| 方法                                                | 说明                                             |
| --------------------------------------------------- | ------------------------------------------------ |
| `static void copy(byte[] in, File out)`             | 将 byte[] 拷贝到一个文件中                       |
| `static void copy(byte[] in, OutputStream out)`     | 将 byte[] 拷贝到一个输出流中                     |
| `static int copy(File in, File out)`                | 将文件拷贝到另一个文件中                         |
| `static int copy(InputStream in, OutputStream out)` | 将输入流拷贝到输出流中                           |
| `static int copy(Reader in, Writer out)`            | 将 Reader 读取的内容拷贝到 Writer 指向目标输出中 |
| `static void copy(String in, Writer out)`           | 将字符串拷贝到一个 Writer 指向的目标中           |

在实例中，我们虽然使用 Resource 加载文件资源，但 FileCopyUtils 本身和 Resource 没有任何关系，您完全可以在基于 JDK I/O API 的程序中使用这个工具类。

**属性文件操作**

我们知道可以通过 java.util.Properties的load(InputStream inStream) 方法从一个输入流中加载属性资源。Spring 提供的 PropertiesLoaderUtils 允许您直接通过基于类路径的文件地址加载属性资源，请看下面的例子：

```
`package com.baobaotao.io; import java.util.Properties; import org.springframework.core.io.support.PropertiesLoaderUtils; public class PropertiesLoaderUtilsExample {     public static void main(String[] args) throws Throwable {             // ① jdbc.properties 是位于类路径下的文件         Properties props = PropertiesLoaderUtils.loadAllProperties("jdbc.properties");         System.out.println(props.getProperty("jdbc.driverClassName"));     } } `
```

 

一般情况下，应用程序的属性文件都放置在类路径下，所以 PropertiesLoaderUtils 比之于 Properties#load(InputStream inStream) 方法显然具有更强的实用性。此外，PropertiesLoaderUtils 还可以直接从 Resource 对象中加载属性资源：

| 方法                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `static Properties loadProperties(Resource resource)`        | 从 Resource 中加载属性                                       |
| `static void fillProperties(Properties props, Resource resource)` | 将 Resource 中的属性数据添加到一个已经存在的 Properties 对象中 |

**特殊编码的资源**

当您使用 Resource 实现类加载文件资源时，它默认采用操作系统的编码格式。如果文件资源采用了特殊的编码格式（如 UTF-8），则在读取资源内容时必须事先通过 EncodedResource 指定编码格式，否则将会产生中文乱码的问题。


**清单 5. EncodedResourceExample**

```
`package com.baobaotao.io; import org.springframework.core.io.ClassPathResource; import org.springframework.core.io.Resource; import org.springframework.core.io.support.EncodedResource; import org.springframework.util.FileCopyUtils; public class EncodedResourceExample {         public static void main(String[] args) throws Throwable  {             Resource res = new ClassPathResource("conf/file1.txt");             // ① 指定文件资源对应的编码格式（UTF-8）             EncodedResource encRes = new EncodedResource(res,"UTF-8");             // ② 这样才能正确读取文件的内容，而不会出现乱码             String content  = FileCopyUtils.copyToString(encRes.getReader());             System.out.println(content);       } } `
```



EncodedResource 拥有一个 getResource() 方法获取 Resource，但该方法返回的是通过构造函数传入的原 Resource 对象，所以必须通过 EncodedResource#getReader() 获取应用编码后的 Reader 对象，然后再通过该 Reader 读取文件的内容。