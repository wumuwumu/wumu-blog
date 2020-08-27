---
layout: odoo
title: 管理Odoo服务器实例
date: 2019-06-18 14:02:38
tags:
- python
- odoo
---

全书完整目录请见：[Odoo 12开发者指南（Cookbook）第三版](https://alanhou.org/odoo12-cookbook/)

本章中，我们将讲解如下内容：

- 配置插件路径
- 更新插件模块列表
- 标准化你的实例目录布局
- 安装并升级本地插件模块
- 对插件应用修改
- 应用及尝试建议的拉取请求

## 引言

在[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)中，我们看了如何使用与编辑器一同发布的标准核心插件来设置 Odoo 实例。本章集中讲解为 Odoo 实例添加非核心插件。Odoo中，你可以从多个目录中加载插件。此外，推荐你将第三方插件（如OCA模块）或你自定义的插件放在一个单独的文件夹中，这样可以避免与 Odoo 核心模块产生冲突。甚至Odoo 企业版也是一种类型的插件目录，你需要像普通插件目录一样加载它。

> ℹ️**有关用词 – 插件(add-on) vs. 模块(module)**
>
> 本书中，我们使用插件或插件模块来指代 Odoo 所预期安装的 Python 包。用户界面常使用应用（app）或模块的表达 ，但我们更愿意保留模块一词来表示Python模块或包，它们不一定是 Odoo 插件，而应用（app）来表示适当定义为应用的插件模块，表示它不是Odoo主菜单中的入口。

## 配置插件路径

通过addons_path参数的配置，你可以在 Odoo 中加载自己的插件模块。在Odoo初始化一个新数据库时，它会搜索在addons_path配置参数中给定的这些目录。addons_path会在这些目录中搜索潜在的插件模块。addons_path中所列出的目录预期应包含子目录，每个子目录是一个插件模块。在数据库初始化完成后，你将能够安装这些目录中所给出的模块。

### 准备工作

这一部分假定你已经准备好了实例并生成了配置文件，如在[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)中*在一个文件中存储实例配置*一节所描述。Odoo的源码存放在~/odoo-dev/odoo中，而配置文件存放在~/odoo-dev/myinstance.cfg中。

### 如何配置…

按如下步骤在实例的addons_path中添加~/odoo-dev/local-addons目录：

1. 编辑你的实例的配置文件，即 ~/odoo-dev/my-instance.cfg。

2. 定位到以addons_path =开头一行，默认，你会看到如下内容：


```
  addons_path = ~/odoo-dev/odoo/odoo/addons,~/odoo-dev/odoo/add-ons 
```


   译者注：

   当前默认生成的配置文件中为绝对路径，且仅包含xxx/odoo/addons

3. 修改该行，添加一个逗号（英文半角），并接你想想要添加为addons_的目录名称，如以下代码所示：



  ```
addons_path = ~/odoo-dev/odoo/odoo/addons,~/odoo-dev/odoo/addons,~/odoo-dev/local-addons 
  ```

4. 重启你的实例

   ```
   $ ~/odoo-dev/odoo/odoo-bin -c my-instance.cfg 
   ```

### 运行原理…

在重启 Odoo 时，会读取配置文件。addons_path变量的值应为一个逗号分隔的目录列表。可接受相对路径，但它们是相对于当前工作目录的，因此应在配置文件中尽量避免。

至此，~/odoo-dev/local-addons中包含的新插件尚不在该实例的可用模块列表中。为此，你需要执行一个额外的操作，在下一部分*更新插件模块列表*中会进行讲解。

### 扩展知识…

在第一次调用 odoo-bin脚本来初始化新数据库时，你可以传递一个带逗号分隔目录列表的–addons-path命令行参数。这会以所提供插件路径中所找到的所有插件来初始化可用插件模块列表。这么做时，你要显式地包含基础插件目录（odoo/odoo/addons）以及核心插件目录（odoo/addons）。

与前面稍有不同的是本地插件目录不能为空（**译者注：**请先阅读下面的小贴士），它必须要至少包含一个子目录，并包含插件模块的最小化结构。在[第四章 创建Odoo插件模块](https://alanhou.org/creating-odoo-add-on-modules/)中，我们会来看如何编写你自己的模块。同时，这里有一个生成内容来满足Odoo要求的快捷版黑科技：



```
$ mkdir -p ~/odoo-dev/local-addons/dummy$ touch ~/odoo-dev/local-addons/dummy/__init__.py$ echo '{"name": "dummy", "installable": False}' > \~/odoo-dev/local-addons/dummy/__manifest__.py 
```

你可以使用–save选项来保存配置文件的路径：



```
$ odoo/odoo-bin -d mydatabase \--add-ons-path="odoo/odoo/addons,odoo/addons,~/odoo-dev/local-addons" \--save -c ~/odoo-dev/my-instance.cfg --stop-after-init 
```

本例中，使用相对路径不会有问题，因为它们会在配置文件中转化为绝对路径。

> **小贴士：**因为Odoo仅当从命令行中设置路径时在插件路径的目录中查看插件，而不是在从配置文件中加载路径的时候，dummy已不再必要。因此，你可以删除它（或保留到你确定不需要新建一个配置文件时）。

## 更新插件模块列表

我们在前面的部分已经说到，在向插件路径添加目录时，仅仅重启Odoo服务是不足以安装其中一个新插件模块的。Odoo还需要有一个指定动作来扫描路径并更新可用插件模块的列表。

### 准备工作

启动你的实例并使用管理员账号连接它。然后，激活开发者模式（如果你不知道如何激活开发者模式，请参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)）。

### 如何更新…

要更新你实例中的可用插件模块列表，你需要执行如下步骤：

1. 打开Apps菜单
2. 点击Update Apps List：
   [![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050902052063.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050902052063.jpg)
3. 在弹出对话框中，点击Update按钮：
   [![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050902070776.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050902070776.jpg)
4. 在更新的最后，你可以点击Apps入口来查看已更新的可用插件模块列表。你将需要删除Apps搜索框中的默认过滤器来查看所有模块。

### 运行原理…

在点击了Update按钮之后，Odoo会读取插件路径配置变量。对于列表中的每一个目录，它会查看包含保存在插件模块目录中名为__manifest__.py的插件声明文件的直接子目录。Odoo读取声明内容，并预期查找其中的Python字典。除非声明内容中包含一个键installable的值为False，插件模块的元数据就会存储在数据库中。如果模块已存在，则更新相关信息。否则，会创建一条新记录。如果此前可用的插件模块未找到，则从列表中删除该记录。

> ℹ️仅在初始化数据库后添加了新的插件路径时才需要更新应用列表。如果你在初始化数据库之前在配置文件中添加了新插件路径，那么就无需手动更新模块列表。

## 标准化你的实例目录布局

我们推荐你在开发和生产环境都使用相似的目录布局。这一标准化会在你要执行运维时体现出用处，它也会缓解你日常工作的压力。

这一部分创建将相似生命周期或相似用途的文件分组放在标准化子目录中的目录结构。请自由按照自己的需求来调整这一结构，但请确保你将这一结构在某处进行记录存档。

### 如何标准化…

创建所推荐实例布局，你需要执行如下步骤：

**译者注：**读者也可直接使用 Alan 在 GitHub 上准备的[安装脚本](https://github.com/alanhou/odoo12-cookbook/tree/master/Chapter02)进行操作

1. 为实例创建一个目录：

   ```
   $ mkdir ~/odoo-dev/projectname$ cd ~/odoo-dev/projectname 
   ```

2. 在名为env/的子目录中创建一个Python虚拟环境：

  ```
  $ virtualenv -p python3 env 
  ```

3. 创建一些子目录，如下：

  ```
  $ mkdir src local bin filestore logs 
  ```

   这些子目录的功能如下：

   - src/：这包含Odoo本身的一个拷贝，以及一些第三方插件项目（我们在下一步中添加了Odoo源码）
   - local/：这用于保存你针对具体实例的插件
   - bin/：这包含各类帮助可执行shell脚本
   - filestore/：这用于文件存储
   - logs/（可选）：这用于存储服务日志文件

4. 克隆Odoo并安装所需依赖包（参见

   第一章 安装Odoo开发环境

   获取更多内容）：

   ```bash
$ git clone https://github.com/odoo/odoo.git src/odoo
$ env/bin/pip3 install -r src/odoo/requirements.txt 
   ```
5. 以bin/odoo保存如下shell脚本：

  ```bash
ROOT=$(dirname $0)/..
PYTHON=$ROOT/env/bin/python3
ODOO=$ROOT/src/odoo/odoo-bin
$PYTHON $ODOO -c $ROOT/projectname.cfg "$@"
exit $?
  ```

6. 让该脚本可执行：

  ```
$ chmod +x bin/odoo 
  ```

7. 创建一个空的本地模块dummy：

```
$ mkdir -p local/dummy
$ touch local/dummy/__init__.py
$ echo '{"name": "dummy", "installable": False}' >\local/dummy/__manifest__.py 
```

8. 为你的实例生成配置文件：



```
$ bin/odoo --stop-after-init --save \ --addons-path src/odoo/odoo/addons,src/odoo/addons,local \ --data-dir filestore 
```

9. 添加一个.gitignore文件，用于告诉GitHub排除这些给定目录，这样Git在提交代码时就会忽略掉这些目录，例如 filestore/, env/, logs/和src/：

```bash
# dotfiles, with exceptions:
.*
!.gitignore
# python compiled files
*.py[co]
# emacs backup files
*~
# not tracked subdirectories
/env/
/src/
/filestore/
/logs/
```

10. 为这个实例创建一个Git仓库并将已添加的文件添加到Git中：

```bash
$ git init
$ git add .
$ git commit -m "initial version of projectname"
```

### 运行原理…

我们生成了一个有明确标签目录和独立角色的干净的目录结构。我使用了不同的目录来存储如下内容：

- 由其它人所维护的代码（src/中）
- 本地相关的具体代码
- 实例的文件存储

通过为每个项目建一个virtualenv环境，我们可以确保该项目的依赖文件不会与其它项目的依赖产生冲突，这些项目你可能运行着不同的Odoo版本或使用了不同的第三方插件模块，这将需要不同版本的Python依赖。这当然也会带来一部分磁盘空间的开销。

以类似的方式，通过为我们不同的项目使用不同的Odoo拷贝以及第三方插件模块，我们可以让每个项目单独的进行推进并仅在需要时在这些实例上安装更新，因此也减少了引入回退的风险。

bin/odoo允许我们不用记住各个路径或激活虚拟环境就可以运行服务。这还为我们设置了配置文件。你可以在其中添加其它脚本来协助你的日常工作。例如，你可以添加一个脚本来检查运行实例所需的第三方项目。

有关配置文件，我们仅展示了这里需要设置的最小化选项，但很明显你可以设置更多，例如数据库名、数据库过滤器或项目所监听的端口。有关这一话题的更多信息，请参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)。

最后，通过在Git仓库中管理所有这些，在不同的电脑上复制这一设置及在团队中分享开发内容变得相当容易。

> **小贴士：**加速贴士
>
> 要加速项目的创建，你可以创建一个包含空结构的模板仓库，并为每个项目复制（fork）该仓库。这会省却你重新输入bin/odoo脚本、.gitignore及其它所需模板文件（持续集成配置、README.md、ChangeLog等等）所花费的时间。

### 参见内容

如果你喜欢这种方法，我们建议你尝试[第三章 服务器部署](https://alanhou.org/server-deployment/)中的使用 Docker 运行 Odoo 一部分的内容。

### 扩展知识…

复杂模块的开发要求有各类配置选项，在想要尝试任何配置选项时都会要更新配置文件。更新配置常常是一件头痛的事，避免它的一种方式是通过命令行传递所有配置选项，如下：

1. 手动激活虚拟环境：

```bash
$ source env/bin/activate
```

2. 进行Odoo源代码目录：

```bash
$ cd src/odoo
```

3. 运行服务：

```bash
./odoo-bin --addons-path=addons,../../local -d test-12 -i account,sale,purchase --log-level=debug
```

第三步中，我们直接通过命令行传递了一些参数。第一个是–addons-path，它加载Odoo的核心插件目录addons，以及你自己的插件目录local，在其中你可以放自己的插件模块。选项-d会使用test-12数据库或者在该数据库不存在时新建一个数据库。选项-i 会安装会计、销售和采购模块。接着，我们传递了log-level选项来将日志级别提升为debug，这样日志中会显示更多的信息。

> ℹ️通过使用命令行，你可以快速地修改配置选项。你也可以在Terminal中查看实时日志。所有可用选项可参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)，或使用-help命令来查看所有的选项列表及各个选项的描述。

## 安装并升级本地插件模块

Odoo 功能的核心来自于它的插件模块。Odoo自带的插件是你所拥有的财富，同时你也可以在应用商店下载一些插件模块或者自己写。

这一部分中，我们将展示如何通过网页界面及命令行来安装并升级插件模块。

对这些操作使用命令行的主要好处包含可以同时作用于一个以上的插件以及在安装或升级的过程中可以清晰地浏览到服务端日志，对于开发模式或编写脚本安装实例时都非常有用。

### 准备工作

确保你有一个运行中的 Odoo 实例，且数据库已初始化、插件路径已进行恰当地设置。在这一部分中，我们将安装/升级一些插件模块。

### 如何安装升级…

安装或升级插件有两种方法-可以使用网页界面或命令行。

#### 通过网页界面

可按照如下步骤来使用网页界面安装新的插件模块到数据库中：

1. 使用管理员账户连接实例并打开Apps菜单
   [![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906002399.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906002399.jpg)
2. 使用搜索框来定位你想要安装的插件。这里有一些帮助你完成该任务的操作指南：
   - 激活Not Installed过滤器
   - 如果你要查找一个具体的功能插件而不是广泛的功能插件，删除Apps过滤器
   - 在搜索框中输入模块名的一部分并使用它来作为模块过滤器
   - 你会发现使用列表视图可以阅读到更多的信息
3. 点击卡片中模块名下的Install按钮。

注意有些Odoo插件模块需要有外部Python依赖，如果你的系统中未安装该Python依赖，那么 Odoo 会中止安装并显示如下的对话框：

[![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906125210.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906125210.jpg)
**译者注：**按正常安装不会出现一错误，需通过 pip uninstall pyldap 才能复现这一错误

修复这一问题，仅需在你的系统中安装相关的Python依赖即可。

要升级已安装到数据库的模块，使用如下步骤：

1. 使用管理员账户连接到实例
2. 打开Apps菜单
3. 点击Apps:
   [![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906203077.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906203077.jpg)
4. 使用搜索框来定位你所安装的插件。有如下的小贴士：
   - 激活Installed过滤器
   - 如果你要查找一个具体的功能插件而不是广泛的功能插件，删除Apps过滤器
   - 在搜索框中输入部分插件模块的名称并按下 Enter 来使用它作为模块过滤器。例如，输入CRM并按下 Enter 来搜索CRM应用
   - 你会发现使用列表视图可以阅读到更多的信息
5. 点击卡片右上角的的三个点，然后点击Upgrade选项：

[![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906265357.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906265357.jpg)

激活开发者模式来查看模块的技术名称。如果你不知道如何激活开发者模式，请参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)：

[![Odoo 12开发者指南第二章 管理Odoo服务器实例](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906302261.jpg)](https://alanhou.org/homepage/wp-content/uploads/2019/05/2019050906302261.jpg)

在激活开发者模式之后，它会以红色显示模块的技术名称。如果你使用的是Odoo社区版，会看到一些带有Upgrade的附加应用。这些是Odoo企业版的应用，要想安装/使用它们，需要购买一个证书。

#### 通过命令行

要在你的数据库中安装新插件，可按照如下步骤：

1. 查找插件的名称。这是包含__manifest__.py文件的目录名，不带前面的路径。

2. 停止实例。如果你在操作生产数据库，请进行备份。

3. 运行如下命令：

   ```bash
   odoo/odoo-bin -c instance.cfg -d dbname -i addon1,addon2 --stop-after-init
   ```


   译者注：

   请将addon1,addon2替换为你所要安装的插件名



   > **小贴士：**你可以省略掉-d dbname，因为这在配置文件中进行了设置。

4. 重新启动实例

### 运行原理…

插件模块的安装和升级是两个紧密关联的操作，但有一些重要的区别，在下面两部分中进行了强调：

#### 插件安装

在你安装插件时，Odoo以提供的名称检查它的可用插件列表中未安装插件。它还会检查该插件的依赖，并且如果有依赖的话，它会在安装插件前递归安装这些依赖。

单个模块的安装包含如下步骤：

1. 如果存在，运行插件preinit钩子
2. 从Python源代码中加载模型定义并在必要时更新数据库结构（参见[第五章 应用模型](https://alanhou.org/application-models/)了解更多信息）
3. 加载插件的数据文件并在必要时更新数据库内容（参见[第七章 模块数据](https://alanhou.org/odoo12-module-data/)了解更多信息）
4. 如果实例中启用了演示数据则安装插件演示数据
5. 如果存在，运行插件postinit钩子
6. 运行对插件视图定义的验证
7. 如果启用了演示数据并启用了测试，运行该插件的测试（参见[第十八章 自动化测试用例](https://alanhou.org/automated-test-cases/)了解更多信息）
8. 在数据库中更新模块状态
9. 从插件的翻译文件中更新数据库中的翻译（参见[第十二章 国际化](https://alanhou.org/internationalization/)了解更多信息）

> ℹ️preinit和postinit钩子分别使用pre_init_hook和post_init_hook键名在__manifest__.py文件中定义。这些钩子用于在插件模块的安装之前及之后触发Python函数。参见[第四章 创建Odoo插件模块](https://alanhou.org/creating-odoo-add-on-modules/)了解更多有关 init 钩子的知识。

#### 插件升级

升级插件时，Odoo以给定的名称在可用的插件模块列表中检查已安装插件。它还会检查该插件的反向依赖（即依赖于所升级插件的那些插件）。如果存在，则也会对它们进行递归升级。

单个插件模块的升级过程包含如下步骤：

1. 如果有的话，先运行插件模块的预迁移步骤（参见[第七章 模块数据](https://alanhou.org/odoo12-module-data/)了解更多信息）
2. 从Python源码中加载模型定义并在必要时更新数据库结构（参见[第五章 应用模型](https://alanhou.org/application-models/)了解更多信息）
3. 加载插件的数据文件并在必要时更新数据库内容（参见[第七章 模块数据](https://alanhou.org/odoo12-module-data/)了解更多信息）
4. 如果实例中启用了演示数据更新插件演示数据
5. 如果模块有任何迁移方法的话，先运行插件模块的后置迁移步骤（参见[第七章 模块数据](https://alanhou.org/odoo12-module-data/)了解更多信息）
6. 运行对插件视图定义的验证
7. 如果启用了演示数据并启用了测试，运行该插件的测试（参见[第十八章 自动化测试用例](https://alanhou.org/automated-test-cases/)了解更多信息）
8. 在数据库中更新模块状态
9. 从插件的翻译文件中更新数据库中的翻译（参见[第十二章 国际化](https://alanhou.org/internationalization/)了解更多信息）

> ℹ️注意更新未安装的插件模块什么也不会做。但是安装已安装的插件模块会重新安装该插件，这会通过一些包含数据的数据文件产生一些预期外的问题，这些文件可能应由用户进行更新而非在常规的模块升级处理时进行更新（参见[第七章 模块数据](https://alanhou.org/odoo12-module-data/)中使用noupdate和forcecreate标记部分的内容）。通过用户界面不存在错误的风险，但通过命令行时则有可能发生。

### 扩展知识…

要当心依赖的处理。假定有一个实例你想要安装sale、sale_stock和sale_specific插件，sale_specific依赖于sale_stock，而sale_stock依赖于sale。要安装这三者，你只需要安装sale_specific，因为它会递归安装sale_stock和sale这两个依赖。要升级这两者，你需要升级sale，因为这样会递归升级其反向依赖，sale_stock和sale_specific。

管理依赖另一个比较搞的地方是在你向已经有一个版本安装了的插件添加依赖的时候。我们继续通过前例来理解这一问题。想像一下你在sale_specific中添加了一个对stock_dropshipping的依赖。更新sale_specific插件不会自动安装新的依赖，也会要求安装sale_specific。在这种情况下，你会收到非常糟糕的错误消息，因为插件的Python代码没有成功的加载，而插件的数据和模型表则存在于数据库中。要解决这一问题，你需要停止该实例并手动安装新的依赖。

## 从GitHub安装插件模块

GitHub是第三方插件的一个很好的来源。很多Odoo合作伙伴使用GitHub来分享他们内部维护的插件，而Odoo社区联盟（OCA）在GitHub上共同维护着几百个插件。在你开始编写自己的插件之前，确保查看是否已有可用的插件或者作为初始以继续扩展插件。

这一部分向你展示如何从GitHub上克隆OCA的partner-contact项目并让其中所包含的插件模块在我们实例中可用。

### 准备工作

假设你想要改变你的实例中地址的处理方式，你的客户需要在Odoo两个字段（街道和街道2）之外的第三个字段来存储地址。你肯定是可以编写自己的插件来为res.partne添加一个字段的，但如果想要让地址在发票上以合适的格式显示，问题就要比看上去麻烦一些了。所幸，你邮件列表上的某个人告诉了你partner_address_street3插件，由OCA作为partner-contact项目的一部分进行维护。

本部分中所使用的路径反映了我们在*标准化你的实例目录布局*一节中所推荐的布局。

### 如何安装…

按照如下步骤来安装partner_address_street3：

1. 进入你的项目目录：

```bash
$ cd ~/odoo-dev/my-odoo/src
```



2. 在src/目录中克隆partner-contact项目的12.0分支：

```bash
$ git clone --branch 12.0 \https://github.com/OCA/partner-contact.git src/partner-contact
```



3. 修改插件路径来包含该目录并更新你的实例中的插件列表（参见本章中的配置插件路径和更新插件模块列表一节）。instance.cfg中的addons_path一行应该是这样的：

   ```
   addons_path = ~/odoo-dev/my-odoo/src/odoo/odoo/addons, \~/odoo-dev/my-odoo/src/odoo/addons, \~/odoo-dev/my-odoo/src/, \~/odoo-dev/local-addons
   ```

4. 安装partner_address_street3插件（如果你不知道如何安装该模块，参见前面一节，安装并升级本地插件模块）

### 运行原理…

所有 Odoo社区联盟的代码仓库都将他们自己的插件放在单独的目录中，这与Odoo对插件路径中目录的预期是相一致的。因此，只需复制某处的仓库并将其添加到插件路径中就够了。

### 扩展知识…

有些维护者遵循不同的方法，每个插件模块一个仓库，放在仓库的根目录下。这种情况下，你需要创建一个新的目录，在这个目录中添加插件路径并克隆你所需的维护者的插件到该目录中。记住在每次添加一个新仓库拷贝时要更新插件模块列表。

## 对插件应用修改

GitHub上可用的大部分插件需要进行修改并且不遵循Odoo对其稳定发行版所强制的规则。它们可能收到漏洞修复或改善，包含你提交的问题或功能请求，这些修改可能会引入数据库模式的修改或数据文件和视图中的更新。这一部分讲解如何安装升级后的版本。

### 准备工作

假定你对partner_address_street3报告了一个问题并收到通知说该问题已在partner-contact项目12.0分支的最近一次修订中得以解决。这种情况下，你可以使用最新版本来更新你的实例。

### 如何修改…

要对GitHub的插件进行源的变更，需执行如下步骤：

1. 停止使用该插件的实例。

2. 如果是生产实例请做一个备份（参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)中*管理Odoo服务端数据库*一节）。

3. 进入克隆了partner-contact的目录：

```bash
$ cd ~/odoo-dev/my-odoo/src/partner-contact
```



4. 为该项目创建一个本地标签，这样万一出现了崩溃你可以进行回退：

```bash
$ git checkout 12.0$ git tag 12.0-before-update-$(date --iso)
```



4. 获取源码的最新版本：

```bash
$ git pull --ff-only
```



6. 在你的数据库中更新partner_address_street3插件（参见*安装并升级本地插件模块*一节）

7. 重启实例

### 运行原理…

通常，插件模块的开发者有时会发布插件的最新版本。这一更新一般包含漏洞修复及新功能。这里，我们将获取一个插件的新版本并在我们的实例中更新它。

如果git pull –ff-only失败的话，你可以使用如下命令回退到前一个版本：

```bash
$  git reset --hard 12.0-before-update-$(date --iso)
```



然后，你可以尝试git pull（不添加–ff-only），它会产生一个合并，但这表示你对插件做了本地修改。

### 扩展知识…

如果更新这一步崩溃了，参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)*从源码更新Odoo*一节获取恢复的操作指南。记住要总是在一个生产数据库的拷贝上先进行测试。

## 应用及尝试建议的拉取请求

在GitHub的世界中，拉取请求（PR）是由开发者所提交的请求，这样项目维护人员可以添加一些新的开发。比如一个 PR 可能包含漏洞修复或新功能。这里请求在拉取到主分支之前会进行审核和测试。

这一部分讲解如何对你的 Odoo 项目应用一个PR来测试漏洞修复的改进。

### 准备工作

在前一节中，假定你对partner_address_street3 报告了一个问题并收到一条通知在拉取请求中问题已修复，尚未合并到项目的12.0分支中。开发人员要求你验证PR #123中的修复状况。你需要使用这一分支更新一个测试实例。

你不应在生产数据库直接使用该分支，因此先创建一个带有生产数据库拷贝的测试环境（参见[第一章 安装Odoo开发环境](https://alanhou.org/installing-odoo-development-environment/)和[第三章 服务器部署](https://alanhou.org/server-deployment/)）。

### 如何操作…

应用并测试一个插件的GitHub拉取请求，你需要执行如下步骤：

1. 停止实例

2. 进入partner-contact所被克隆的目录：

```bash
$ cd ~/odoo-dev/my-odoo/src/partner-contact
```



3. 为该项目创建一个本地标签，这样万一出现崩溃时你可以回退：

```bash
$  git checkout 12.0$ git tag 12.0-before-update-$(date --iso
```



4. 拉取pull请求的分支。这么做最容易的方式是使用PR编号，在开发者与你沟通时你应该可以看到。在本例中，这个拉取请求编号是123：

```bash
$ git pull origin pull/123/head
```



5. 在你的数据库中更新partner_address_street3插件模块并重启该实例（如果你不知道如何更新该模块的话请参见*安装并升级本地插件模块*一节）

6. 测试该更新 – 尝试重现问题，或测试你想要的功能。

如果这不能运行，在GitHub的PR页面进行评论，说明你做了什么以及什么不能运行，这样开发者可以更新这个拉取请求。

如果它没有问题，也在PR页面说下；这是PR验证流程中非常重要的一部分；这会加速主分支中的合并。

### 运行原理…

我们在使用一个GitHub功能，使用pull/nnnn/head分支名称来通过编号进行拉取请求的拉取，其中nnnn是PR的编号。Git pull命令合并远程分支到我们的分支，在我们基础代码中应用修改。在这之后，我们更新插件模块、对其测试并向作者报回修改是成功或是失败。

### 扩展知识…

如果你想要同步测试它们，你可以针对相同仓库的不同拉取请求重复本节中的第4步。如果你对结果很满意，你可以创建一个分支来保留对应用了改变的结果的引用：

```bash
$ git checkout -b 12.0-custom
```



使用一个不同的分支会帮助你记住你没有从GitHub使用该版本，而是一个自定义的版本。

> ℹ️git branch命令可用于列出你仓库中的所有本地分支。

从这开始，如果你需要应用来自GitHub中12.0分支的最近一个审核版本，你需要不使用–ff-only来拉取它：

```bash
$ git pull origin 12.0
```



