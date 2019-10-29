---
title: npm版本管理
date: 2019-10-29 14:08:30
tags:
- node
---

在打包项目的时候，我们都要更新package.json的版本号，接着给给代码添加tag，最后push代码，这样的流程泰国麻烦有什么方法简化。

```
1. package.json`中修改递增`version
2. git add -A
3. git commit -m "update version"
4. git push
5. git tag <tag version>
6. git push --tag
7. npm publish
```

# 解决方法

我们可以使用`npm version`命令，从[文档](https://docs.npmjs.com/cli/version)上我们可以看到其依据[semver](https://semver.org/lang/zh-CN/)支持了大部分alias：

```bash
npm version [<newversion> | major | minor | patch | premajor | preminor | prepatch | prerelease | from-git]
```

> 例：初始版本为1.0.0
>
> `npm version prepatch`  //预备补丁版本号 v1.0.1-0
>
> `npm version prerelease`  //预发布版本号 v1.0.1-1
>
> `npm version patch` //补丁版本号 v1.0.2
>
> `npm version preminor` //预备次版本号 v1.1.0-0
>
> `npm version minor` //次版本号 v1.1.0
>
> `npm version premajor` //预备主版本号 v2.0.0-0
>
> `npm version major` //主版本号 v2.0.0

当在仓库中执行`npm version时`，会自动提交`git commit`并打上`git tag`。

> 当使用`-m`参数时，就可以自定义发布版本的信息，其中`%s`可以用来代替当前版本号
>
> ```
> npm version patch -m "upgrade to %s for reasons"
> 复制代码
> ```

这样以后版本迭代只需要以下步骤

- `npm version patch | minor | major | ...etc`
- `git push`
- `git push --tag`
- `npm publish`

npm version会同时创建时 `v版本号` 形式的tag，将tag push上去就可以自动触发构建了。

也可以简化这步操作，在npm version操作后自动 push

在 package.json中加入下面的代码，即可实现npm version操作后，自动push代码及tag，也就自动触发了 npm 发布操作。

```json
  "scripts": {
    "postversion": "git push --follow-tags"
  }
```

# 衍生问题

> 如何发布beta，rc，alpha版本呢？如果发布了，应该如何安装？

#### 解决方案

首先我们要理解这些版本的含义

- alpha：内部测试版本
- beta： 公开测试版本
- rc： 候选版本（Release Candidate）

然后将`package.json`的`version`改成`x.x.x-beta`

配合`npm publish --tag <tag>`，我们可以发布对应的`dist-tag`

> 举个例子：
>
> 使用`npm publish --tag beta`发布后，然后就可以使用`npm install <pkg>@beta`安装对应版本的包。

我们可以通过`npm dist-tag ls <pkg>`来查看包的`dist-tag`

```json
{
    latest: 1.0.1, // 这就是npm publish默认发布的tag
    beta: 1.0.1-beta
}
```

当我们的beta版本稳定后，可以使用`npm dist-tag add x.x.x-beta latest`设置为稳定版本。

# npm version与npm dist-tag

关于npm version prerelease的作用我这里不再赘述，你可以查看[这个文章](https://github.com/liangklfangl/npm-dist-tag/blob/master/NPM%E6%A8%A1%E5%9D%97%E7%9A%84TAG%E7%AE%A1%E7%90%86)。我只是记录一下关于npm version与npm dist-tag的使用：

第一步：发布第一个稳定版本

```
 npm publish//1.0.0
```

第二步：修改文件继续发布第二个版本

```
git add -A && git commit -m "c"
npm version patch
npm publish//1.0.1
```

第三步：继续修改文件发布一个prerelease版本

```
 git add -A && git commit -m "c"
 npm version prerelease
 npm publish --tag -beta//版本n-n-n-n@1.0.2-0
```

第四步：继续修改发布第二个prerelease版本

```
git add -A && git commit -m "c"
npm version prerelease
npm publish --tag -beta//版本n-n-n-n@1.0.2-1
```

第五步：npm info查看我们的版本信息

```json
{ name: 'n-n-n-n',
  'dist-tags': { latest: '1.0.1', '-beta': '1.0.2-1' },
  versions: [ '1.0.0', '1.0.1', '1.0.2-0', '1.0.2-1' ],
  maintainers: [ 'liangklfang <liangklfang@163.com>' ],
  time:
   { modified: '2017-04-01T12:17:56.755Z',
     created: '2017-04-01T12:15:23.605Z',
     '1.0.0': '2017-04-01T12:15:23.605Z',
     '1.0.1': '2017-04-01T12:16:24.916Z',
     '1.0.2-0': '2017-04-01T12:17:23.354Z',
     '1.0.2-1': '2017-04-01T12:17:56.755Z' },
  homepage: 'https://github.com/liangklfang/n#readme',
  repository: { type: 'git', url: 'git+https://github.com/liangklfang/n.git' },
  bugs: { url: 'https://github.com/liangklfang/n/issues' },
  license: 'ISC',
  readmeFilename: 'README.md',
  version: '1.0.1',
  description: '',
  main: 'index.js',
  scripts: { test: 'echo "Error: no test specified" && exit 1' },
  author: '',
  gitHead: '8123b8addf6fed83c4c5edead1dc2614241a4479',
  dist:
   { shasum: 'a60d8b02222e4cae74e91b69b316a5b173d2ac9d',
     tarball: 'https://registry.npmjs.org/n-n-n-n/-/n-n-n-n-1.0.1.tgz' },
  directories: {} }
```

我们只要注意下面者两个部分：

```
 'dist-tags': { latest: '1.0.1', '-beta': '1.0.2-1' },
  versions: [ '1.0.0', '1.0.1', '1.0.2-0', '1.0.2-1' ],
```

其中最新的稳定版本和最新的beta版本可以在dist-tags中看到，而versions数组中存储的是所有的版本。

第六步：npm dist-tag命令

```
npm dist-tag ls n-n-n-n
```

即npm dist-tag获取到所有的最新的版本，包括prerelease与稳定版本，得到下面结果：

```
-beta: 1.0.2-1
latest: 1.0.1
```

第七步：当我们的prerelease版本已经稳定了，重新设置为稳定版本

```
npm dist-tag add n-n-n-n@1.0.2-1 latest
```

此时你通过npm info查看可以知道：

```json
{ name: 'n-n-n-n',
  'dist-tags': { latest: '1.0.2-1', '-beta': '1.0.2-1' },
  versions: [ '1.0.0', '1.0.1', '1.0.2-0', '1.0.2-1' ],
  maintainers: [ 'liangklfang <liangklfang@163.com>' ],
  time:
   { modified: '2017-04-01T12:24:55.800Z',
     created: '2017-04-01T12:15:23.605Z',
     '1.0.0': '2017-04-01T12:15:23.605Z',
     '1.0.1': '2017-04-01T12:16:24.916Z',
     '1.0.2-0': '2017-04-01T12:17:23.354Z',
     '1.0.2-1': '2017-04-01T12:17:56.755Z' },
  homepage: 'https://github.com/liangklfang/n#readme',
  repository: { type: 'git', url: 'git+https://github.com/liangklfang/n.git' },
  bugs: { url: 'https://github.com/liangklfang/n/issues' },
  license: 'ISC',
  readmeFilename: 'README.md',
  version: '1.0.2-1',
  description: '',
  main: 'index.js',
  scripts: { test: 'echo "Error: no test specified" && exit 1' },
  author: '',
  gitHead: '03189d2cc61604aa05f4b93e429d3caa3b637f8c',
  dist:
   { shasum: '41ea170a6b155c8d61658cd4c309f0d5d1b12ced',
     tarball: 'https://registry.npmjs.org/n-n-n-n/-/n-n-n-n-1.0.2-1.tgz' },
  directories: {} }
```

主要关注如下:

```
 'dist-tags': { latest: '1.0.2-1', '-beta': '1.0.2-1' },
  versions: [ '1.0.0', '1.0.1', '1.0.2-0', '1.0.2-1' ]
```

此时latest版本已经是prerelease版本"1.0.2-1"了！此时用户如果直接运行npm install就会安装我们的prerelease版本了，因为版本已经更新了！

当然，我们的npm publish可以有很多tag的，比如上面是beta，也可以是stable, dev, canary等，比如下面你继续运行：

```
 git add -A && git commit -m "c"
 npm version prerelease
 npm publish --tag -dev
```

此时你运行npm info就会得到下面的信息：

```
{ name: 'n-n-n-n',
  'dist-tags': { latest: '1.0.2-1', '-beta': '1.0.2-1', '-dev': '1.0.2-2' },
  versions: [ '1.0.0', '1.0.1', '1.0.2-0', '1.0.2-1', '1.0.2-2' ],
  maintainers: [ 'liangklfang <liangklfang@163.com>' ],
  time:
   { modified: '2017-04-01T13:01:17.106Z',
     created: '2017-04-01T12:15:23.605Z',
     '1.0.0': '2017-04-01T12:15:23.605Z',
     '1.0.1': '2017-04-01T12:16:24.916Z',
     '1.0.2-0': '2017-04-01T12:17:23.354Z',
     '1.0.2-1': '2017-04-01T12:17:56.755Z',
     '1.0.2-2': '2017-04-01T13:01:17.106Z' },
  homepage: 'https://github.com/liangklfang/n#readme',
  repository: { type: 'git', url: 'git+https://github.com/liangklfang/n.git' },
  bugs: { url: 'https://github.com/liangklfang/n/issues' },
  license: 'ISC',
  readmeFilename: 'README.md',
  version: '1.0.2-1',
  description: '',
  main: 'index.js',
  scripts: { test: 'echo "Error: no test specified" && exit 1' },
  author: '',
  gitHead: '03189d2cc61604aa05f4b93e429d3caa3b637f8c',
  dist:
   { shasum: '41ea170a6b155c8d61658cd4c309f0d5d1b12ced',
     tarball: 'https://registry.npmjs.org/n-n-n-n/-/n-n-n-n-1.0.2-1.tgz' },
  directories: {} }
```

重点关注如下内容

```
 'dist-tags': { latest: '1.0.2-1', '-beta': '1.0.2-1', '-dev': '1.0.2-2' },
  versions: [ '1.0.0', '1.0.1', '1.0.2-0', '1.0.2-1', '1.0.2-2' ],
```

此时你会看到-beta版本最新是1.0.2-1，而-dev版本最新是1.0.2-2

# 参考

> <https://github.com/liangklfangl/npm-dist-tag>
>
> <https://juejin.im/post/5b624d42f265da0fa1223ffa>
>
> <https://docs.npmjs.com/cli/version>

