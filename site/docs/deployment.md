# 启动和部署

Midway 提供了一个轻量的启动器，用于启动你的应用。我们为应用提供了多种部署模式，你既可以将应用按照传统的样子，部署到任意的服务上（比如自己购买的服务器），也可以将应用构建为一个 Serverless 应用，Midway 提供跨多云的部署方式。


## 本地开发


这里列举的主要是本地使用 `dev` 命令开发的方式，有两种。


### 快速启动单个服务


在本地研发时，Midway 在 `package.json` 中提供了一个 `dev` 命令启动框架，比如：
```json
{
  "script": {
    "dev": "midway-bin dev --ts"
  }
}
```
这是一个最精简的命令，他有如下特性：


- 1、使用 `--ts` 指定 TypeScript（ts-node）环境启动
- 2、使用内置的 API（@midwayjs/core 的 `initializeGlobalApplicationContext`）创建一个服务，不经过 `bootstrap.js`
- 3、单进程运行

在命令行运行下面的命令即可执行。
```bash
$ npm run dev
```



### 指定入口启动服务

由于本地的 dev 命令普通情况下和 `bootstrap.js` 启动文件初始化参数不同，有些用户担心本地开发和线上开发不一致，比如测试链路等。

这个时候我们可以直接传递一个入口文件给 `dev` 命令，直接使用入口文件启动服务。

```json
{
  "script": {
    "dev": "midway-bin dev --ts --entryFile=bootstrap.js"
  }
}
```



## 部署到普通服务器


### 部署后和本地开发的区别


在部署后，有些地方和本地开发有所区别。


**1、node 环境的变化**


最大的不同是，服务器部署后，会直接使用 node 来启动项目，而不是 ts-node，这意味着不再读取 `*.ts` 文件。


**2、加载目录的变化**


服务器部署后，只会加载构建后的 `dist` 目录，而本地开发则是加载 `src` 目录。

|  | 本地 | 服务器 |
| --- | --- | --- |
| appDir | 项目根目录 | 项目根目录 |
| baseDir | 项目根目录下的 src 目录 | 项目根目录下的 dist 目录 |


**3、环境的变化**


服务器环境，一般使用 `NODE_ENV=production` ，很多库都会在这个环境下提供性能更好的方式，例如启用缓存，报错处理等。


**4、日志文件**


一般服务器环境，日志不会打印到项目的 logs 目录，而是其他不会受到项目更新影响的目录，比如 `home/admin/logs` ，这样固定的目录，也方便其他工具采集日志。


### 部署的流程


整个部署分为几个部分，由于 Midway 是 TypeScript 编写，比传统 JavaScript 代码增加了一个构建的步骤，整个部署的过程如下。

![image.png](https://img.alicdn.com/imgextra/i3/O1CN01wSpCuM27pWGTDeDyK_!!6000000007846-2-tps-2212-242.png)
由于部署和平台、环境非常相关，下面我们都将以 Linux 来演示，其他平台可以视情况参考。


### 编译代码和安装依赖


由于 Midway 项目是 TypeScript 编写，在部署前，我们先进行编译。在示例中，我们预先写好了构建脚本，执行 `npm run build` 即可，如果没有，在 `package.json` 中添加下面的 `build` 命令即可。
```json
// package.json
{
  "scripts": {
    "build": "midway-bin build -c"
  },
}
```

:::info
虽然不是必须，但是推荐大家先执行测试和 lint。
:::


一般来说，部署构建的环境和本地开发的环境是两套，我们推荐在一个干净的环境中构建你的应用。


下面的代码，是一个示例脚本，你可以保存为 `build.sh` 执行。

```bash
## 服务器构建（已经下载好代码）
$ npm install                                       # 安装开发期依赖
$ npm run build																			# 构建项目
$ npm prune --production												    # 移除开发依赖

## 本地构建（已经安装好 dev 依赖）
$ npm run build
$ npm prune --production														# 移除开发依赖
```

:::info
一般安装依赖会指定 `NODE_ENV=production` 或 `npm install --production` ，在构建正式包的时候只安装 dependencies 的依赖。因为 devDependencies 中的模块过大而且在生产环境不会使用，安装后也可能遇到未知问题。
:::


执行完构建后，会出现 Midway 构建产物 `dist` 目录。
```bash
➜  my_midway_app tree
.
├── src
├── dist                # Midway 构建产物目录
├── node_modules        # Node.js 依赖包目录
├── test
├── package.json
└── tsconfig.json
```


### 打包压缩


构建完成后，你可以简单的打包压缩，上传到待发布的环境。




### 上传和解压


有很多种方式可以上传到服务器，比如常见的 `ssh/FTP/git` 等。也可以使用 [OSS](https://www.aliyun.com/product/oss) 等在线服务进行中转。


### 启动项目

Midway 构建出来的项目是单进程的，不管是采用 `fork` 模式还是 `cluster` 模式，单进程的代码总是很容易的兼容到不同的体系中，因此非常容易被社区现有的 pm2/forever 等工具所加载，


我们这里以 pm2 来演示如何部署。


项目一般都需要一个入口文件，比如，我们在根目录创建一个 `bootstrap.js` 作为我们的部署文件。
```
➜  my_midway_app tree
.
├── src
├── dist                # Midway 构建产物目录
├── test
├── bootstrap.js        # 部署启动文件
├── package.json
└── tsconfig.json
```


Midway 提供了一个简单方式以满足不同场景的启动方式，只需要安装我们提供的 `@midwayjs/bootstrap` 模块（默认已自带）。

```bash
$ npm install @midwayjs/bootstrap --save
```

然后在入口文件中写入代码，注意，这里的代码使用的是 `JavaScript` 。

```javascript
const { Bootstrap } = require('@midwayjs/bootstrap');
Bootstrap.run();
```

虽然启动文件的代码很简单，但是我们依旧需要这个文件，在后续的链路追踪等场景中需要用到。

注意，这里不含 http 的启动端口，如果你需要，可以参考文档 修改。

- [修改 koa 端口](extensions/koa#修改端口)

这个时候，你已经可以直接使用 `NODE_ENV=production node bootstrap.js` 来启动代码了，也可以使用 pm2 来执行启动。

我们一般推荐使用工具使用工具来启动 Node.js 项目，下面有一些文档可以进阶阅读。

- [pm2 使用文档](extensions/pm2)
- [cfork 使用文档](extensions/cfork)



### 启动参数

在大多数情况下，不太需要在 Bootstrap 里配置参数，但是依旧有一些可配置的启动参数选项，通过 `configure` 方法传入。

```typescript
const { Bootstrap } = require('@midwayjs/bootstrap');
Bootstrap
  .configure({
  	imports: [/*...*/]
  })
  .run();
```



| 属性           | 类型                                                         | 描述                                                         |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| appDir         | string                                                       | 可选，项目根目录，默认为 `process.cwd()`                     |
| baseDir        | string                                                       | 可选，项目代码目录，研发时为 `src`，部署时为 `dist`          |
| imports        | Component[]                                                  | 可选，显式的组件引用                                         |
| moduleDetector | 'file' \| IFileDetector \| false                             | 可选，使用的模块加载方式，默认为 `file` ，使用依赖注入本地文件扫描方式，可以显式指定一个扫描器，也可以关闭扫描 |
| logger         | Boolean \| ILogger                                           | 可选，bootstrap 中使用的 logger，默认为 consoleLogger        |
| ignore         | string[]                                                     | 可选，依赖注入容器扫描忽略的路径，moduleDetector 为 false 时无效 |
| globalConfig   | Array<{ [environmentName: string]: Record<string, any> }> \| Record<string, any> | 可选，全局传入的配置，如果传入对象，则直接以对象形式合并到当前的配置中，如果希望传入不同环境的配置，那么，以数组形式传入，结构和 `importConfigs` 一致。 |



**示例，传入全局配置（对象）**

```typescript
const { Bootstrap } = require('@midwayjs/bootstrap');
Bootstrap
  .configure({
  	globalConfig: {
      customKey: 'abc'
    }
  })
  .run();
```



**示例，传入分环境的配置**

```typescript
const { Bootstrap } = require('@midwayjs/bootstrap');
Bootstrap
  .configure({
  	globalConfig: [{
      default: {/*...*/},
      unittest: {/*...*/}
    }]
  })
  .run();
```






## 使用 Docker 部署

### 编写 Dockerfile，构建镜像


步骤一：在当前目录下新增Dockerfile

```dockerfile
FROM node:12

WORKDIR /app

ENV TZ="Asia/Shanghai"

COPY . .

# 如果各公司有自己的私有源，可以替换registry地址
RUN npm install --registry=https://registry.npm.taobao.org

RUN npm run build

# 如果端口更换，这边可以更新一下
EXPOSE 7001

CMD ["npm", "run", "online"]
```


步骤二: 新增 `.dockerignore` 文件（类似 git 的 ignore 文件），可以把 `.gitignore`  的内容拷贝到 `.dockerignore` 里面


步骤三：当使用 pm2 部署时，请将命令修改为 `pm2-runtime start` ，pm2 行为请参考 [pm2 容器部署说明](https://www.npmjs.com/package/pm2#container-support)。


步骤四：构建 docker 镜像

```bash
$ docker build -t helloworld .
```

步骤五：运行 docker 镜像

```bash
$ docker run -itd -P helloworld
```

运行效果如下：
![image.png](https://cdn.nlark.com/yuque/0/2020/png/187105/1608882492099-49160b6a-601c-4f08-ba65-b95a1335aedf.png#height=33&id=BtUCB&margin=%5Bobject%20Object%5D&name=image.png&originHeight=45&originWidth=1024&originalType=binary&ratio=1&size=33790&status=done&style=none&width=746)

然后大写的 `-P` 由于给我们默认分配了一个端口，所以我们访问可以访问 `32791`  端口（这个 `-P` 是随机分配，我们也可以使用 `-p 7001:7001` 指定特定端口）

![image.png](https://cdn.nlark.com/yuque/0/2020/png/187105/1608882559686-031bcf0d-2185-42cd-a838-80f008777395.png#height=94&id=dfag9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=188&originWidth=578&originalType=binary&ratio=1&size=24488&status=done&style=none&width=289)

关于别的推送到 dockerhub 或者 docker 的 registry，可以大家搜索对应的方法。


**优化**

我们看到前面我们打出来的镜像有1个多G，可优化的地方：
- 1、我们可以采用更精简的 docker image 的基础镜像：例如 node:12-alpine，
- 2、其中的源码最终也打在了镜像中，其实这块我们可以不需要。

我们可以同时结合 docker 的 multistage 功能来做一些优化，这个功能请注意要在 `Docker 17.05` 版本之后才能使用。


```dockerfile
FROM node:12 AS build

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

FROM node:12-alpine

WORKDIR /app

COPY --from=build /app/dist ./dist
COPY --from=build /app/bootstrap.js ./
COPY --from=build /app/package.json ./

RUN apk add --no-cache tzdata

ENV TZ="Asia/Shanghai"

RUN npm install --production

# 如果端口更换，这边可以更新一下
EXPOSE 7001

CMD ["npm", "run", "start"]
```

当前示例的结果只有 `207MB`。相比原有的 `1.26G` 省了很多的空间。

### 结合 Docker-Compose 运行

在 docker 部署的基础上，还可以结合 docker-compose 部署一些跟自己服务相关的服务。


**步骤一**

按照 Docker 方式部署的方式新增 dockerfile


**步骤二**

新增 `docker-compose.yml` 文件，内容如下：（此处我们模拟我们的 midway 项目需要使用redis）

```yaml
version: "3"
services:
  web:
    build: .
    ports:
      - "7001:7001"
    links:
      - redis
  redis:
    image: redis

```


**步骤三：构建**

使用命令：

```bash
$ docker-compose build
```

**步骤四：运行**

```bash
$ docker-compose up -d
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/187105/1608884158660-02bd2d3c-08b4-4ecc-a4dd-a18d4b9d2c12.png#height=44&id=jWw4i&margin=%5Bobject%20Object%5D&name=image.png&originHeight=62&originWidth=1054&originalType=binary&ratio=1&size=47727&status=done&style=none&width=746)
那么redis比如怎么用，因为 docker-compose 里面加了一个 redis，并且 link 了。

关于更多关于 docker-compose 的详情，可以查看网上关于 docker-compose 的使用方法。



## 单文件构建部署

在某些场景，将项目构建为单文件，部署的文件可以更小，可以更容易的分发部署，在一些场景下特别的高效，如：

- Serverless 场景，单文件可以更快的部署
- 私密场景，单文件可以更容易的做加密混淆

Midway 从 v3 开始支持将项目构建为单文件。

不支持的情况有：

- egg 项目（@midwayjs/web）
- 入口处 `importConfigs` 使用的路径形式引入配置的应用，组件
- 未显式依赖的包，或者包里有基于约定的文件

:::info

当前，还处于测试阶段。

:::

### 前置依赖

单文件构建有一些前置依赖条件。

```bash
## 用于生成入口
$ npm i @midwayjs/bundle-helper --save-dev

## 用于构建单文件
## 装到全局
$ npm i @vercel/ncc -g
## 或者装到项目
$ npm i @vercel/ncc --save-dev
```



### 代码部分

必须将项目引入的配置调整为 "对象模式"。

Midway 的官方组件都已经调整为该模式，如果有自己编写的组件，也请调整为该模式才能构建为单文件。

:::tip

Midway v2/v3 均支持配置以 "对象模式" 加载。

:::

```typescript
// src/configuration.ts
import { Configuration } from '@midwayjs/decorator';
import { join } from 'path';

import * as DefaultConfig from './config/config.default';
import * as LocalConfig from './config/config.local';

@Configuration({
  importConfigs: [
    {
      default: DefaultConfig,
      local: LocalConfig
    }
  ]
})
export class ContainerLifeCycle {
}
```



### 构建流程

单文件构建的编译需要几个步骤：

- 1、准备单文件构建入口
- 2、将项目 ts 文件构建为 js
- 3、使用额外编译器，将所有的 js 文件打包成一个文件

**步骤一**

修改入口 `bootstrap.js`  为下列代码。

```typescript
const { Bootstrap } = require('@midwayjs/bootstrap');

// 显式以组件方式引入用户代码
Bootstrap.configure({
  // 这里引用的是编译后的入口，本地开发不走这个文件
  imports: require('./dist/index'),
  // 禁用依赖注入的目录扫描
  moduleDetector: false,
}).run()

```

**步骤二**

`package.json` 中增加下面的脚本。

```json
  "scripts": {
    // ...
    "bundle": "bundle && npm run build && ncc build bootstrap.js -o build",
    "bundle_start": "NODE_ENV=production node ./build/index.js"
  },
```

包含三个部分

- `bundle` 是将所有的项目代码以组件的形式导出，并生成一个 `src/index.ts` 文件
- `npm run buid` 是基础的 ts 项目构建，将 `src/**/*.ts` 构建为 `dist/**/*.js`
- `ncc build bootstrap.js -o build` 以 `bootstrap.js` 为入口构建为一个单文件，最终生成到 `build/index.js`

**步骤三**

执行命令。

```bash
$ npm run bundle
```

:::tip

注意，构建过程中可能有错误，比如 ts 定义错误，入口生成语法不正确等情况，需要手动修复。

:::

**步骤四**

启动项目。

```bash
$ npm run bundle_start
```

如果启动访问没问题，那么你就可以拿着构建的 build 目录做分发了。
