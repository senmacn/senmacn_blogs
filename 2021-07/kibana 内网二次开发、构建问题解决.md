# kibana 内网二次开发、构建问题解决

Kibana 是为 Elasticsearch设计的开源分析和可视化平台。

笔者的公司起初是直接使用了 Kibana + Elasticsearch 的安装版，交给后端同事部署使用。在使用过程中其他部门提出了一些特殊的使用需求，并且 Kibana 的界面、功能无法满足完全满足使用者的需求（国人和外国人使用的思路确实有些不同。。。），领导也提出了后续会进一步整合 kibana 到整个工作系统的任务。

出于以上的原因，笔者和后端同事在前一段时间做了一些关于 Kibana 二次开发的工作。中间遇到了各种各样的问题。部分问题（比如版本、依赖下载）面向搜索引擎就能解决，然而笔者的公司对于内外网限制很严，在网上关于 Kibana 在内网下开发部署编译构建的内容极少，摸索很久，一个个小问题慢慢解决，才终于成功完成这项任务。在这里分享给大家。（由于 Kibana 版本的原因，大家在部署的过程中问题肯定不会一模一样，但其原理都差不多）

笔者二次开发的版本 @7.8 - @7.10。

## 工具准备

工欲善其事必先利其器。

1. 首先需要知道你需要那个版本的 Kibana 以及对应版本的 Elasticsearch。（注意：版本必须严格按照官网对应的来，否则————）到 github 去 clone 下对应版本的 Kibana 源码，Elasticsearch 使用安装版就行；
2. Kibana 要求版本的 nodejs（同样，必须一致），在 /.nvmrc 文件中可以找到，建议使用 nvm 管理多个nodejs 版本；
3. git 包含 gitbash；
4. yarn (最新版本就行，其实 npm 也行，依赖下载速度会非常感人)。

## Kibana 开发环境部署

这里有两种方法：
1. 直接在外网部署好，整个项目拷贝入内网。好处是暂时不需要管依赖的事，虽然后续 Kibana build 的过程还是需要解决以来下载的问题。
2. 源码移入内网，在内网解决依赖问题。

这里就按内网有私服，整个都放到内网解决吧。

源码移入内网后，按照 Kibana 推荐的部署开发环境的方式，使用 `yarn kbn bootstrap` 命令。

```js
  // package.json
  "scripts": {
    // ...
     "kbn": "node scripts/kbn",
  },
```

运行 kbn.js 文件，下载项目需要的所有依赖，内网显然是会有问题的，因为 yarn.lock 文件：

```
"@babel/cli@^7.12.10":
  version "7.12.10"
  resolved "https://registry.yarnpkg.com/@babel/cli/-/cli-7.12.10.tgz#67a1015b1cd505bde1696196febf910c4c339a48"
  integrity sha512-+y4ZnePpvWs1fc/LhZRTHkTesbXkyBYuOB+5CyodZqrEuETXi3zOVfpAQIdgC3lXbHLTDG9dQosxR9BhvLKDLQ==
```

npm 仓库指向的是外网 npm 仓库的下载源，这里笔者是将 

`"https://registry.yarnpkg.com/@babel/cli/-/cli-7.12.10.tgz#67a1015b1cd505bde1696196febf910c4c339a48"` 

整体替换成了内网私服的地址（注意可能会有多个源，都需要替换，仔细！）。当然删除 yarn.lock 文件，直接 `yarn install` 理论上也是行的，不推荐，依赖关系可能会改变造成构建失败。

可能出现的问题：
1. node-sass 下载失败。前端项目部署老生常谈的问题了，不被 node-sass 坑过，怎么能自称做过前端，这里就不过多赘述；

2. 网络超时。设置代理、改为淘宝镜像源等都可以解决；

3. kibana @7.10 后 增加了部分新的依赖，类似node-sass，除了请求 npm 仓库以外，还需要下载额外的依赖包，建议挂代理或者手动下载。

依赖下载完成后，进入kibana目录执行yarn kbn bootstrap（**最好在git bash窗口执行**）。很关键的点，第一次启动最好（必须）使用 gitbash（或者其他 shell）。推测是 Kibana 第一次构建的时候，需要运行某些 shell 命令，直接在 cmd 中运行会报找不到依赖的错误！

开发环境部署的步骤比较简单，只要耐心一点，不要求快，一般不会有什么大问题。

## Kibana build

二次开发完成后，需要 build 打包生成生产环境的 Kibana，这里也有一些关键的点。

首先直接运行 `yarn build`，直接报错。看一下 /package.json 文件：

```js
  "build": "node scripts/build --all-platforms",
  // 这里的 --all-platforms 命令可以改为你需要的平台，打包全平台会消耗大量的无用时间！
```

`yarn build` 命令运行了 script/build.js，只有两行代码。

```js
require('../src/setup_node_env'); // nodejs 环境，版本正确的话，不需要管
require('../src/dev/build/cli'); // 关键点
```

看看 /src/dev/build/cli.js 文件：

```js
  // ...
  `
  options:
        --oss                   {dim Only produce the OSS distributable of Kibana}
        --no-oss                {dim Only produce the default distributable of Kibana}
        --skip-archives         {dim Don't produce tar/zip archives}
        --skip-os-packages      {dim Don't produce rpm/deb/docker packages}
        --all-platforms         {dim Produce archives for all platforms, not just this one}
        
  // 上面说的平台有关的选项，根据生产环境的需要修改即可
  `
  buildDistributables(log, buildOptions!)
  // 关键函数 buildDistributables
```
这里的 `build_distributables` 函数引用自 src/dev/build/build_distributables.js(ts)文件，包含了打包过程中的所有任务。打包过程中主要的问题均衡来自这里。接下来，展示我所遇到的几个问题的解法，其他问题类比即可。

### 无法下载 nodejs / 找不到对应版本的 nodejs

找到下面这两个任务：

```js
 await run(Tasks.Clean);
 await run(options.downloadFreshNode ? Tasks.DownloadNodeBuilds : Tasks.VerifyExistingNodeBuilds);
```
下面这个，会根据 downloadFreshNode 判断下载/验证 nodejs，建议直接注释掉此任务。（除非内网有或者自己搭建 node 库的下载源并且修改函数内部下载逻辑，没有这个必要）。注释后，打包还是会提示没有nodejs，当然，需要一份上文提及的相应版本的 nodejs，放到 Kibana 提示的文件夹，不需要 node_modules。

`yarn build` 还是提示错误！

这是因为上面的 `Tasks.Clean` 会删除上一次打包的文件，包括nodejs！在 /src/dev/build/tasks/clean_tasks.js(ts) 文件中可以看到其逻辑。

```js
export const Clean: GlobalTask = {
  global: true,
  description: 'Cleaning artifacts from previous builds',
  async run(config, log) {
    await deleteAll(
      [
        config.resolveFromRepo('build'),
        config.resolveFromRepo('target'),
        config.resolveFromRepo('.node_binaries'),
        // 在这里，需要把上面这行注释掉
      ],
      log
    );
  },
};
```
再次 `yarn build` ，nodejs 相关的问题应该就这些了（假如有问题，同样的处理，查看哪个任务环节的问题，并修改/注释掉即可）。

### 缺少 nodejs 的 gz 包（或是缺少一些其他的文件）

外网下载相应的gz包或者其他依赖文件，拷贝到 error 提示的位置就行。

### 无法下载依赖/网络错误（yarn）

build 过程中需要再下载一次依赖（yarn.lock 源已修改的应该不会再报错了），按上文解决 `yarn install` 问题的步骤去做就行（不建议直接注释任务）。

### 找不到、无法下载 chromium 相关（@7.10 版本以后）

`Tasks.InstallChromium` 任务需要去下载 chromium 文件，同样手动下载需要的版本，放到指定位置。
```js
  await run(Tasks.InstallChromium);
```

### PatchNativeModules 错误（无法下载 re2 文件）

`Tasks.PatchNativeModules` 内部会下载项目所需的 re2 文件：

```js
 await run(Tasks.PatchNativeModules);
```
```js
const packages: Package[] = [
  {
    name: 're2',
    version: '1.15.4',
    destinationPath: 'node_modules/re2/build/Release/re2.node',
    extractMethod: 'gunzip',
    archives: {
      darwin: {
        url: 'https://github.com/uhop/node-re2/releases/download/1.15.4/darwin-x64-64.gz',
        sha256: '595c6653d796493ddb288fc0732a0d1df8560099796f55a1dd242357d96bb8d6',
      },
      linux: {
        url: 'https://github.com/uhop/node-re2/releases/download/1.15.4/linux-x64-64.gz',
        sha256: 'e743587bc96314edf10c3e659c03168bc374a5cd9a6623ee99d989251e331f28',
      },
      win32: {
        url: 'https://github.com/uhop/node-re2/releases/download/1.15.4/win32-x64-64.gz',
        sha256: 'b33de62cda24fb02dc80a19fb79977d686468ac746e97cd211059d2d4c75d529',
      },
    },
  },
];

await download({
    log,
    url: archive.url,
    destination: downloadPath,
    sha256: archive.sha256,
    retries: 3,
  });
```
按照地址下载相应文件，放入提示的位置即可。

## 总结

笔者写这篇文章主要的目的是希望能帮助到开发经验没那么丰富，被迫（雾）对 Kibana 搞来搞去的大家（开发来开发去，已有功能其实都没怎么用。），当然也为了抚慰一下自己被公司内网和 Kibana 等东西整的心力憔悴的那一段时间。

Kibana 会不断的成长下去，目前就已经用 typescript 重构了一部分代码了，相信之后的改变会更大，但是希望大家不要被它庞大的代码量以及各式各样的错误所吓倒。部署打包的关键点，其实也就是搞清楚 `build_distributables` 里面的各种任务，根据自己开发的环境，具体任务具体分析，一个个小问题慢慢解决。前几个问题解决之后，了解其思路，举一反三，相信大家也能很快反应过来，就这啊^。