---
title: nexus仓库分类
abbrlink: eeeeee70
date: 2020-10-09 15:00:00
---

### Nexus仓库分类

Nexus包含了各种类型的仓库类型。在登录后的Nexus界面，单击左边的“Repositories”链接

四种仓库类型：

1）group（仓库组）

2）hosted（宿主）

3）proxy（代理）

4）virtual（虚拟）

说明：

1）每种类型的Format有Maven1或者Maven2，maven1是老版本，现在一般使用maven2。

2）仓库的Policy（策略）表示该仓库为发布（Release）版本还是快照（Snapshot）版本仓库。

3）虚拟仓库其实也是为maven1服务的，所以意义不大。

4）宿主仓库指的就是我们自己项目所构建组成的仓库。

5）代理仓库指的是远程仓库，比如中央仓库等，因为私服需要完全替代中央仓库，那么他必须拥有中央仓库的功能，所以nexus的仓库会有各种代理仓库

6）仓库组，他是整合以上所有的仓库于一体，那么他就是我们项目私服的地址，因为他把所有仓库都容纳为一个个体，所以我们下载资源时，他都能在对应的仓库中找到。

http://localhost:8081/nexus/content/groups/public/

![img](http://static.oschina.net/uploads/space/2015/0604/191316_hhia_1989321.png)

Nexus列出了默认的几个仓库：

1）Public Repositories：仓库组，将所有策略为Release的仓库聚合并通过一致的地址提供服务。

2）3rd party：一个策略为Release的宿主类型仓库，用来部署无法从公共仓库获得的第三方发布版本构件。

3）Apache Snapshots：策略为Snapshots的代理仓库，用来代理Apache Maven仓库的快照版本构件。

4）Central：该仓库代理Maven的中央仓库，策略为Release，只会下载和缓存中央仓库中的发布版本构件。

5）Central M1 shadow：maven1格式的虚拟类型仓库。

6）Codehaus Snapshots：代理Codehaus Maven仓库快照版本的代理仓库。

7）Release：策略为Release的宿主类型仓库，用来部署组织内部的发布版本构件。

8）Snapshots：策略为Snapshots的宿主类型仓库，用来部署组织内部的快照版本构件。

![img](http://static.oschina.net/uploads/space/2015/0604/191357_Rrwh_1989321.png)

仓库之间的关系

![img](http://static.oschina.net/uploads/space/2015/0604/191445_kmKi_1989321.jpeg)





### 2、Nexus的索引与构件搜索

点击列表上的“Central”行，在下方的“Configuration”中我们可以看到，在“Ordered Group Repositories”中包含了Release、Snapshots、3rd party、Central等仓库。为了构建Nexus的Maven中央库索引，首先需要设置Nexus中Maven Cencal代理仓库下载远程索引，将“Download Remote Indexes”的值从默认值false改为true。然而，由于其他索引库，因为他们要么依赖中央库，要么是本地库，所以，只需要右键update index即可。

![img](http://static.oschina.net/uploads/space/2015/0604/191532_xf3l_1989321.jpeg)

点击“Save”后，点击update now 更新索引，Nexus后台在下载Maven中央仓库的索引。

![img](http://static.oschina.net/uploads/space/2015/0604/191615_JXjl_1989321.png)

保存过后点击Browser Remote 然后看看远程索引库是否更新下来了

![img](http://static.oschina.net/uploads/space/2015/0604/191644_ybMm_1989321.png)

如果没有出现远程索引信息，那么要在“Public Repositories”行右击，点击“Update Index”

![img](http://static.oschina.net/uploads/space/2015/0604/191731_e2Xb_1989321.jpeg)