---
title: maven生成Manifest文件
date: 2021-1-22 18:00:00
---

# maven-jar-plugin常用

```xml
<build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.2.0</version>
        <configuration>
          <archive>
            <manifestFile>${project.build.outputDirectory}/META-INF/MANIFEST.MF</manifestFile>
          </archive>
        </configuration>
        ...
      </plugin>
    </plugins>
 </build>
```

# 常用参数

```xml
<archive>
  <addMavenDescriptor/>
  <compress/>
  <forced/>
  <index/>
  <pomPropertiesFile/>
 
  <manifestFile/>
  <manifest>
    <addClasspath/>
    <addDefaultEntries/>
    <addDefaultImplementationEntries/>
    <addDefaultSpecificationEntries/>
    <addBuildEnvironmentEntries/>
    <addExtensions/>
    <classpathLayoutType/>
    <classpathPrefix/>
    <customClasspathLayout/>
    <mainClass/>
    <packageName/>
    <useUniqueVersions/>
  </manifest>
  <manifestEntries>
    <key>value</key>
  </manifestEntries>
  <manifestSections>
    <manifestSection>
      <name/>
      <manifestEntries>
        <key>value</key>
      </manifestEntries>
    <manifestSection/>
  </manifestSections>
</archive>
```

## 存档

| 元素                                                         | 描述                                                         | 类型    | 自   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------- | ---- |
| addMavenDescriptor                                           | 创建的归档文件是否包含这两个Maven文件: 1. pom 文件, 位于归档文件中 META-INF/maven/g r o u p I d / {groupId}/*g**r**o**u**p**I**d*/{artifactId}/pom.xml 2. pom.properties 文件, 位于归档文件中 META-INF/maven/g r o u p I d / {groupId}/*g**r**o**u**p**I**d*/{artifactId}/pom.properties 默认值为true。 | boolean |      |
| compress                                                     | 为存档激活压缩。默认值为true。                               | boolean |      |
| forced                                                       | 是否强制重新创建存档(默认情况下)。将此选项设置为false，意味着归档程序应该将所包含文件的时间戳与目标归档的时间戳进行比较，并仅在后一个时间戳先于前一个时间戳的情况下重新构建归档。检查时间戳通常会提高性能(特别是，如果可以取消构建中的以下步骤，如果没有重新创建归档)，而不考虑您不时得到不准确结果的成本。特别是，不会检测到源文件的删除。 归档器不一定支持最新的检查。如果是，将该选项设置为true将被忽略。 默认值为true。 | boolean | 2.2  |
| index                                                        | 创建的存档是否包含 INDEX.LIST 文件。默认值为false。          | boolean |      |
| pomPropertiesFile                                            | 使用它来覆盖自动创建的 [pom.properties](https://maven.apache.org/shared/maven-archiver/#pom-properties-content) 文件(仅当addMavenDescriptor被设置为true时) | File    | 2.3  |
| manifestFile                                                 | 有了它，您可以提供自己的清单文件。                           | File    |      |
| [manifest](https://maven.apache.org/shared/maven-archiver/#class_manifest) |                                                              |         |      |
| manifestEntries                                              | 要添加到清单中的键/值对列表。                                | Map     |      |
| [manifestSections](https://maven.apache.org/shared/maven-archiver/#class_manifestSection) |                                                              |         |      |

## pom.properties 内容

自动创建 pom.properties 文件将包含以下内容：

```xml
artifactId=${project.artifactId}
groupId=${project.groupId}
version=${project.version}
```

## manifest

| 元素                            | 描述                                                         | 类型    | 自    |
| ------------------------------- | ------------------------------------------------------------ | ------- | ----- |
| addClasspath                    | 是否创建 Class-Path 清单项。默认值为false。                  | boolean |       |
| addDefaultEntries               | 如果清单将包含以下条目:                                      | boolean | 3.4.0 |
| addDefaultImplementationEntries |                                                              |         |       |
| addDefaultSpecificationEntries  |                                                              |         |       |
| addBuildEnvironmentEntries      |                                                              |         |       |
| addExtensions                   |                                                              |         |       |
| classpathLayoutType             | 格式化创建的 Class-Path 中的条目时要使用的布局类型。有效值是:simple、repository(与Maven类路径布局相同)和custom。 注意:如果指定 custom 类型，还必须设置 customClasspathLayout。默认值很简单。 |         |       |
| classpathPrefix                 | 将作为所有Class-Path条目前缀的文本。默认值为“”               |         |       |
| customClasspathLayout           | 。                                                           |         |       |
| mainClass                       | Main-Class清单条目                                           | String  |       |

### manifestSection



| Element         | Description                                       | Type   | Since |
| :-------------- | :------------------------------------------------ | :----- | :---- |
| name            | The name of the section.                          | String |       |
| manifestEntries | A list of key/value pairs to add to the manifest. | Map    |       |

# 参考

> https://maven.apache.org/shared/maven-archiver/examples/classpath.html
>
> https://maven.apache.org/shared/maven-archiver/
>
> https://blog.csdn.net/ksdb0468473/article/details/110833520