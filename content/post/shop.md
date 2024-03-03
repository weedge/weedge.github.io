---
author: "weedge"
title: "全栈开发"
date: 2022-01-02T10:26:23+08:00
tags: [
	"全栈",
]
categories: [
	"技术",
]

---

## 介绍

以前工作中有过前端的开发经验，使用后端模版Smarty(主要是外部需求，现在新项目中应该很少使用这个了)，前端模版Mustache，页面中使用 js jquery交互逻辑，直接服用一些开源的UI组件Bootstrap，开源的后台UI系统，主要是建设内部后台的时候使用，自动生成CRUD页面；前端的技术栈更新迭代相对后端快些，通过以全栈技术栈为切入点，通过一个简单的系统，学习下最新技术栈工具，主要目的如下：

根据需求，实现一个简单功能的购物系统，目的是为了学习ts->js on node.js全栈开发，了解整体开发构建工具，熟悉下工具开发流程，主要还是了解前端框架工具的使用，以及熟悉通过js运行时环境运行在后端服务上的应用开发工具，以便在后续开发后台系统的时候可以熟练使用这些技术进行开发，这些工具的设计思想可以借鉴到其他后端业务开发语言中。

<!--more-->

## 功能要求

1. ⽤户可以浏览商品，提供按商品名字搜索功能。 
2. ⽤户可以下单购买某样商品（⽀付过程可以省略），购买成功后系统需要异步通知下游系统（如仓库 系统、物流系统等，下游系统可简单实现）。 
3. 提供后台管理⻚⾯录⼊商品，包括商品名称、商品描述、商品价格、库存等。
4. 要求记录⽤户访问⽇志，供审计使⽤。 

## 实现要求

1. 前端使⽤Vue.js + elementUI，后台使⽤ AdonisJS 框架（AdonisJS 版本要求5.0以上）。 
2. 数据库使⽤ mysql，结合 AdonisJS 的 ORM 能⼒。 
3. 设计时需要考虑必要的系统安全性。 

## 设计实现

### 需求分析

整体交互主要是实现购买系统，以及用户购买商品的访问日志，以及用户行为日志，给审计系统，进行漏洞分析等，考虑到前期用户比较少，交互使用，5天设计开发时间，先提供v1简单版本，满足商家和买家用户的基本购买功能

买家基本需求：

1. 通过页面查看上架商品列表；查看商品详情页；通过商品名关键字搜索商品；
2. 购买商品，查看订单(为了简化需求，无购物车功能，假设用户订单只能购买一件商品，及用户和订单是一对多，订单和商品是多对一)；

卖家基本需求：

1. 后台录入查看修改删除商品(一个用户编辑多个商品项，一个商品项也可以给多个用户编辑，及用户和商品是多对多，为了简单考虑，不考虑协同的情况)；

后台需求：

1. 记录访问日志，审计需求；

### 功能设计

1. Shop-frontend  数据页面展现，满足商品列表展现；通过商品名搜索；详情页查看；
2. Shop-backend  商品后台，提供http RESTful api ，满足页面需求，以及相应auth权限验证，日志记录，购买订单发送；
3. Shop-admin-frontend 商品数据管理页面展现，操作商品增删查改功能页面；
4. Shop-admin-backend 商品数据管理后台，提供http RESTful api， 满足页面CRUD功能；
5. Shop-backend  记录用户访问日志， 适用pb格式记录日志，产生日志实时写入消息队列中，提供给审计系统使用
6. 其他服务；

整体架构设计如下图所示：

![simple-shop-system](https://raw.githubusercontent.com/weedge/mypic/master/shop-system.drawio.png)

### 数据库设计

单独数据库实例shop, 简单实现，未根据未来几年的数据量来评估表的容量，分库分表的情况等。

1. 商品表  tbl_shop_items 

   (Tips: 这里为了简化，没有定义复杂业务实体关系了，比如产品单元(CPU), 商品单元(SKU)，产品容器(Container) 等，不要脱离业务耍流氓)

| 字段        | 类型           | 描述         |
| ----------- | -------------- | ------------ |
| id          | int64          | 商品id       |
| name        | varchar(32)    | 商品名称     |
| desc        | varchar(1024)  | 商品描述     |
| price       | unsigned int   | 价格（分）   |
| stock       | unsigned int   | 库存数目     |
| sell_cn     | unsigned int   | 售卖数目     |
| is_released | bool           | 是否上架     |
| is_soldout  | bool           | 是否售匿     |
| is_del      | bool           | 是否删除     |
| owner_id    | unsigned int64 | 添加者用户id |
| create_at   | timestamp      | 创建时间     |
| update_at   | timestamp      | 修改时间     |
| ext         | varchar        | 扩展字段     |

2. 用户商品操作表 tbl_user_items

| 字段      | 类型           | 描述               |
| --------- | -------------- | ------------------ |
| id        | unsigned int64 | 操作id             |
| user_id   | unsigned int64 | 商品管理后台用户id |
| item_id   | unsigned int64 | 商品id             |
| create_at | timestamp      | 创建时间           |
| update_at | timestamp      | 修改时间           |
| ext       | varchar        | 扩展字段           |

3. 用户订单表  tbl_user_orders （为了简化，一个用户只能选一个商品直接下单购买，无购物车）

| 字段       | 类型           | 描述     |
| ---------- | -------------- | -------- |
| user_id    | unsigned int64 | 用户id   |
| item_id    | unsigned int64 | 商品id   |
| order_id   | unsigned int64 | 订单id   |
| pay_amount | unsigned int64 | 支付金额 |
| create_at  | timestamp      | 创建时间 |
| update_at  | timestamp      | 修改时间 |
| ext        | varchar        | 扩展字段 |

4. 用户表 tbl_users (users)

| 字段              | 类型           | 描述                    |
| ----------------- | -------------- | ----------------------- |
| id                | unsigned int64 | 用户id                  |
| name              | varchar        | 用户名                  |
| email             | varchar        | 邮件                    |
| password          | varchar        | 加密密码                |
| remember_me_token | varchar        | api Opaque Access Token |
| is_admin          | bool           | 是否是管理员            |
| created_at        | timestamp      | 创建时间                |
| updated_at        | timestamp      | 修改时间                |

ER关系： 

[tbl_shop_items] <-N---1-> (create) [tbl_users]

[tbl_shop_items] <-1---N-> [tbl_user_items] (record)   <-M---1-> [tbl_users]

[tbl_shop_items] <-1---N-> [tbl_user_orders] (order) <-M---1-> [users]

### 消息设计

订单生成后，通知下游系统，订阅消息，暂不考虑。

### 后端接口设计

因为逻辑简单，就没有画出接口功能的时序图和流程图。

#### Shop-backend  服务端口：2021

1. GET /shop/api/v1/itmes  商品列表
2. GET /shop/api/v1/items/:id 商品详情
3. GET /shop/api/v1/items/search?q=** 更具名称搜索商品
4. POST /shop/api/v1/order  商品下单
5. GET  /shop/api/v1/orders/:orderId  获取订单详情
6. GET /shop/api/v1/users/:uid/orders 用户商品订单列表

#### Shop-admin-backend 服务端口：2022

1. shop items CRUD RESTful api  for resource methods ([route RESTful resource](https://docs.adonisjs.com/guides/controllers#resourceful-routes-and-controllers))
2. POST /shopadmin/api/v1/items/:id/check 商品审核
3. GET /shopadmin/api/v1/items/:uid  获取用户创建的商品

### 前端页面交互设计

前端页面调试，本地起页面服务端口进行本地调试

#### Shop-frontend (vue route component page)

1. /     主页面/搜索结构页，列表页面 ，展示搜索框
2. /items/:id   详细页面， 展示购买按钮
3. /user/:id/orders 用户商品订单列表页面
4. /error  访问不到的页面

#### Shop-admin-frontend (CRUD edge template layout page)

CRUD RESTful api  for resource methods to render edge page

## 开发

初期整体使用前端js全栈开发技术栈；Typescript 静态语言，通过 tsc 将ts编译成js 减少错误；通过npm管理依赖包(vim vundle/python pip/php composer/go module/rust cargo)；使用VS Code IDE开发神器编码(All in ONE，可远程ssh连接安装了linux 内核虚拟服务调试学习，查看源码很方便)，安装所需插件。

具体开发工具如下： 

Typescript 编程语言(JavaScript超级 tsc 4.5.4 ts->js) + Node.js 运行时环境(v17.2 V8 JavaScript 引擎 from Google Chrome 的内核) + Adonis.js MVC后端框架(v5.0+) + Vue.js前端框架(v3.0+)  + Element-plus UI 前端页面UI模版(v1.3)

tips：业务模块设计尽量满足SOLID原则，最终满足**高内聚，低耦合**, 尽量让开发框架来满足。

### 后端服务开发

shop-backend, shop-admin-backend 服务逻辑， 使用ts语言，AdonisJS (MVC+IoC开发框架,适合后台开发框架) 5.0+开发环境：

[Model](https://docs.adonisjs.com/guides/database/introduction): app ER(entity relationship) data model -> table schema by migration -> create table -> make  table model 

[Controller](https://docs.adonisjs.com/guides/controllers): make controller for api logic

[Routing](https://docs.adonisjs.com/guides/routing): http RESTful api router (start preload routes)

[View](https://docs.adonisjs.com/guides/views/introduction): view  edge template reander frontend page

[Validator](https://docs.adonisjs.com/guides/validator/introduction): validate data schema

[Rules](https://docs.adonisjs.com/guides/validator/custom-rules): validate rules(common & DIY  start preload rules)

[Middleware](https://docs.adonisjs.com/guides/middleware):  Middleware are a series of functions that are executed during an HTTP request before it reaches the route handler. Every function in the chain has the ability to end the request or forward it to the `next` function. **~~server~~** Middleware, **Global** Middleware, **Naming** Middleware

[Authentication](https://docs.adonisjs.com/guides/auth/introduction): using **sessions**, **basic auth** or **API tokens(JWT, OAT)** 

[Events](https://docs.adonisjs.com/guides/events): define evens listener  to listen async event on(eventName,callback) , then emit event to tigger regist event's callback; eg: new user to emit event callback to send email (DIY events Listenner, start preload events)

通过adonis-ts-app demo 初始化一个app:	

```shell
#new project
npm init adonis-ts-app@latest shop-backend
npm init adonis-ts-app@latest shop-admin-backend
#install package for mysql 
npm i @adonisjs/lucid@alpha
#invoke package to select mysql for ace make:model and migration table
node ace configure(invoke) @adonisjs/lucid
#install mysql2
npm install mysql2
#modify config/database.ts use mysql client: mysql2

#install auth middleware
npm i @adonisjs/auth@alpha
#install Argon2 password hashing algorithm following the PHC string format
npm install phc-argon2
#invoke package to select lucid and api tokens
# https://docs.adonisjs.com/guides/auth/introduction
node ace invoke @adonisjs/auth
# rigister auth naming middleware in start/kernel.js  auth: 'App/Middleware/Auth',
# add auth middleware for router in start/router.js  
# use database-backed opaque access token
# demo: https://dev.to/tngeene/adonisjs-understanding-user-registration-and-authentication-2ojl
# for session auth
npm i @adonisjs/session 
# for CSRF token protection
npm i @adonisjs/shield 
#invoke add fhield global middleware in start/kernel.js () => import('@ioc:Adonis/Addons/Shield')
node ace invoke @adonisjs/shield

#change .env modify mysql conf
#migration table
node ace make:migration {table}
#migration create/rollback table
node ace migration:run / node ace migration:rollback

#then u can use ace to make a new model app/Models/***.ts
node ace make:model ***
#then u can use ace to make a new app/Controllers/Http/***Controller.ts
node ace make:controller ***
#if use edge tpl, u can use ace to make a new view to render tpl
node ace make:view ***
#if valide request param, u can use ace to make a new volidator
#https://docs.adonisjs.com/guides/validator/introduction
node ace make:validator ***
#or diy validate rules
node ace make:prldfile validator

#make events to gen start/events.ts for listen on(eventName,callback)
node ace make:prldfile events
#or make listener to listen on eventsList 
node ace make:listener ***


#make a middleware like auth middleware;
#transmit by ctx, ctx->ctx->ctx to next()
#eg: u can make a user action log middleware to analyse or A/B test for recommended system
#https://docs.adonisjs.com/guides/middleware
node ace make:middleware ***

# https://github.com/luin/ioredis/blob/master/API.md
# https://docs.adonisjs.com/guides/redis
#install redis
npm i @adonisjs/redis
node ace configure @adonisjs/redis

# https://www.npmjs.com/package/@elastic/elasticsearch
# https://www.elastic.co/guide/en/elasticsearch/client/javascript-api/current/typescript.html
npm i @elastic/elasticsearch


# watch run 
node ace serve --watch
```

### 前端开发

Shop-frontend, shop-admin-frontend 前端页面逻辑，使用TypeScript+ Vue.js(前端开发框架) + ElementUI组件 通过vite构建本地开发(CI过程通过webpack打包生成静态资源，通过CD部署到边缘CDN节点)，开发环境具体操作如下：

Vue.js 3 + TypeScript + Vite + ElementUI-plus

[vue router](https://next.router.vuejs.org/zh/): 通过 Vue.js，用组件组成应用；通过 Vue Router 将组件映射到路由上，让 Vue Router 知道在哪里渲染它们;

[axios](https://axios-http.com/zh/docs/intro): HTTP 网络client请求库; 

[vuex](https://next.vuex.vuejs.org/zh/index.html): state management pattern + library stat/view/actions like MVC;

[vee-validate](https://vee-validate.logaretm.com/v4/guide/overview): form validation;  [yup](https://www.npmjs.com/package/yup):  data schema validation;

```shell
# install vue cli
npm install -g @vue/cli
# u can use vue create app use vue.js v3
# vue create vue-app & cd vue-app
# vue add element-plus

# or use vite build vue.js v3 project app
npm init @vitejs/app shop-frontend 
npm init @vitejs/app shop-admin-frontend

# eg: shop-frontend
cd shop-frontend (shop-admin-frontend)
# choose vue-ts when install
npm install

# install vue-router4, https://next.router.vuejs.org/zh/
npm install vue-router@4

# install axios https://axios-http.com/zh/docs/intro
npm install axios

# install vuex, nice design(state management pattern + library stat/view/actions like MVC)  
# https://next.vuex.vuejs.org/zh/index.html
npm install vuex@next --save

# install vee-validate, https://vee-validate.logaretm.com/v4/guide/overview
npm i vee-validate@next --save


#install element-plus ui
npm install element-plus --save
npm i -D sass
#auto import 
npm install -D unplugin-vue-components unplugin-auto-import

# run local vue dev runtime app service (vite --port 5000 --host)
npm run dev

```

附具体代码(Monolith版本, 后端状态存储数据读写使用同一个数据源)：

1. [Shop-admin-backend](https://github.com/weedge/backend-adonisjs-api/tree/main/shop-admin-backend) : 后台管理系统，为了方便后续根据数据库表一键生成CURD代码，采用后端模版edge template；用户管理可以采用内部公司用户平台SSO单点登录获取ticket的方式，或者使用框架本身的身份认证功能进行后台模块的权限管理(web authentication) 
2. [Shop-backend-api](https://github.com/weedge/backend-adonisjs-api/tree/main/adonisjs-shop-backend-api)：后端接口，获取接口数据，需要登录验证，和token鉴权，记录用户行为日志，读写操作数据库和缓存等；
3. [Shop-frontend-vue](https://github.com/weedge/frontend-vue/tree/main/vue-shop-frontend)：前端页面，使用vue框架开发，访问后端API使用axios，状态管理使用vuex，页面路由使vue-router，表单验证和数据验证分别使用vee-valdate和yup，组件的使用需要结合文档使用；首页/搜索结果页展示商品列表，点击更多进入商品详细页面进行购买，购买需要登录/注册，登录后进入订单列表页，具体需求推动；

### 整体联调

这里前后端统一技术栈采用js on node  by (AdonisJS 5 + Vue 3 + Element-plus UI)， 如果熟悉node js  很方便调试，开发1人工全栈搞定就行，至于外部需求UI的调整美化由PM和设计师确定好，尽量通过工具自动化， 降低沟通成本；

当然当后续系统切成微服务，把1人工全栈的活分成多个人来维护，从公司和组织架构层面来说，虽然增加了开发沟通成本，但是对公司来说是有利的，人员离职对系统影响会降低很多吧(from: 郭老师的架构课,以心理学+认知等方面)；

## 总结

前端的技术迭代速度是快于后端的，主要是前端有大量的需求开发，不管是外部业务还是内部需求开发，所以需要满足业务场景下的高效迭代速度，衍生出高效的开发框架工具和UI组件复用，同时需要保持工具在项目中的迭代升级，而且在基于前端技术发展触及到后端开发，演变出前后技术栈统一的全栈开发，但是需求增多，需要更多的资源来承载用户请求，对于后端一些基础组件服务和性能优化的场景，还是需要根据**语言特性，人力，组织结构**等因素考虑；前端技术底层引擎和原理性基础知识是不易变的，可以在关注变化的过程中，深入浅出共性的底层逻辑，以不变应万变；工欲善其事，必先利其器；(不管是前端/后端开发，DevOps, 以及大数据开发，甚至AI，**熟练**利用好工具，工具是在应用场景下沉淀下来的，**理解**基础原理和工具设计原理，结合使用场景，快速定位(文档，问题)，输出最大化，技术工具没有银弹，应用在适用场景达到事半功倍效果，学会在巨人的肩膀上考虑问题，搞事情)。

## reference

1. ECMAScript: https://tc39.es/ecma262/
2. ts: https://www.typescriptlang.org/      https://typescript.bootcss.com/
3. why-ts: https://serokell.io/blog/why-typescript
4. Node.js: http://nodejs.cn/learn  **https://nodejs.org/dist/latest-v17.x/docs/api/**
5. V8 JavaScript/[WebAssembly](https://developer.mozilla.org/zh-CN/docs/WebAssembly/Concepts) Engine: **https://chromium.googlesource.com/v8/v8.git** **https://v8.dev/docs**
6. Adonis.js5: https://adonisjs.com/ 
7. Adonis.js5 + Edge template demo: https://masteringbackend.com/posts/adonisjs-tutorial-the-ultimate-guide
8. Adonis.js5 + Vue.js3 demo: https://www.section.io/engineering-education/build-a-ticketing-app-with-adonisjs-and-vuejs/
9. Vue.js3: https://v3.cn.vuejs.org/guide/introduction.html
10. Vite: https://cn.vitejs.dev/guide/
11. Vue CLI: https://cli.vuejs.org/zh/guide/
12. Element-Plus UI: https://element-plus.gitee.io/zh-CN/guide/installation.html
13. webpack: https://webpack.docschina.org/concepts/
14. web开发: **https://developer.mozilla.org/zh-CN/docs/Web** 

