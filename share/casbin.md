### casbin

#### casbin 是什么
```
casbin是一个强大的、高效的开源访问控制框架，其权限管理机制支持多种访问控制模型，具有访问控制模型model和策略policy两个核心概念
```

#### 支持的model
```
  - ACL (Access Control List, 访问控制列表)
  - 具有 超级用户 的 ACL
  - 没有用户的 ACL: 对于没有身份验证或用户登录的系统尤其有用
  - 没有资源的 ACL: 某些场景可能只针对资源的类型, 而不是单个资源, 诸如 write-article, read-log等权限。 它不控制对特定文章或日志的访问。
  - RBAC (Role-Based Access Control，基于角色的访问控制)
  ABAC有时也被称为PBAC（Policy-Based Access Control）或CBAC（Claims-Based Access Control）。
  - 支持资源角色的RBAC: 用户和资源可以同时具有角色 (或组)。
  - 支持域/租户的RBAC: 用户可以为不同的域/租户设置不同的角色集
  - ABAC (Attribute-Based Access Control，基于属性的访问控制): 支持利用resource.Owner这种语法糖获取元素的属性。
  - RESTful: 支持路径, 如 /res/*, /res/: id 和 HTTP 方法, 如 GET, POST, PUT, DELETE。
  - 拒绝优先: 支持允许和拒绝授权, 拒绝优先于允许。
  - 优先级: 策略规则按照先后次序确定优先级，类似于防火墙规则
```

##### 工作原理
```
  在 Casbin 中, 访问控制模型被抽象为基于 PERM (Policy, Effect,
Request, Matcher) 的一个文件。因此，切换或升级项目的授权机制
与修改配置一样简单。您可以通过组合可用的模型来定制您自己的访
问控制模型。

例如，您可以在一个model中获得RBAC角色和ABAC属性，并共享一组policy规则。
```
- conf
```
// authz_rbac_with_not_deny_model
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act, eft

[role_definition]
g = _, _, _

[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && keyMatch(r.obj, p.obj) && r.act == p.act
// 对于过长的单行配置，您也可以通过在结尾处添加“\”进行断行
```

- model
```
p, 1646054d-d620-424c-99f9-dfc5752d5153, huanan1, /api/admin/series_description/*, GET, allow

g, 1646054d-d620-424c-99f9-dfc5752d5153, 6a8d84fe-38c1-4c20-b43b-ebd3f2efc771, huanan1
```
- 语法(这里以RBAC为例)

```txt
  Model CONF 至少应包含四个部分: [request_definition], [policy_definition], [policy_effect], [matchers]，
  如果 model 使用 RBAC, 还需要添加[role_definition]部分。

 1.request定义
 
 sub, dom, obj, act 表示经典三元组: 访问实体 (Subject), 域(domain)，访问资源 (Object) 和访问方法 (Action)。
 2.policy 定义
 
 这里以model里例子来说比较形象
 p, 1646054d-d620-424c-99f9-dfc5752d5153, huanan1, /api/admin/series_description/*, GET, allow
 p => policy; 1646054d-d620-424c-99f9-dfc5752d5153 => 角色id;
 huanan1=> domain; /api/admin/series_description/* => obj; GET => action; allow => eft
 这里描述的意思是
 角色在华南地区可以get series_description 被允许
 3.role_definition 定义
 
 这里可以理解为角色的继承
 4.policy_effect 定义
 
 对policy生效范围的定义， 原语定义了当多个policy rule同时匹配访问请求request时,该如何对多个决策结果进行集成以实现统一决策。
 以上面的例子来说，意思就是有一个通过且没有任何一个拒绝就通过
 
 
 NOTE
1.Casbin 只存储用户角色的映射关系。
2.Cabin 没有验证用户是否是有效的用户，或者角色是一个有效的角色。 这应该通过认证来解决。
3.RBAC 系统中的用户名称和角色名称不应相同。因为Casbin将用户名和角色识别为字符串， 所以当前语境下Casbin无法得出这个字面量到底指代用户 alice 还是角色 alice。 这时，使用明确的 role_alice ，问题便可迎刃而解。
4.假设A具有角色 B，B 具有角色 C，并且 A 有角色 C。 这种传递性在当前版本会造成死循环。

角色层次
Casbin 的 RBAC 支持 RBAC1 的角色层次结构功能，如果 alice具有role1, role1具有role2，则 alice 也将拥有 role2 并继承其权限。
对于Casbin中的内置角色管理器, 可以指定最大层次结构级别。 默认值为10。 这意味着终端用户 alice 只能继承10个级别的角色。
 
```
##### 考虑设计
```
我现在使用的是RBAC的权限控制，在这基础上增加域策略和deny策略
域策略的出发点是在云上部署的时候可以根据地域或者医院的不同，隔离不同的权限
deny策略的出发点在于，猫被允许吃鱼，吃猫粮，但是对某只肥猫不给鱼吃某一条鱼，就可以设置p, Tom, /food?fish=yu1, GET, deny

RBAC的策略大致分为几种
[RBAC](http://directory.apache.org/fortress/user-guide/1.3-what-rbac-is.html)
RBAC1 带有角色继承的RBAC,角色继承就是指角色可以继承于其他角色,在拥
有其他角色权限的同时，自己还可以关联额外的权限这种设计可以给角色分组和分层。我现也是用的此种方案

RBAC2 静态职责分离，用户无法同时被赋予有冲突的角色。

RBAC3 动态职责分离，用户在一次会话（Session）中不能同时激活自身所
有的、互相有冲突的角色，只能选择其一

Casbin 支持 RBAC 系统的多个实例, 例如,用户可以具有角色及其继承关系
资源也可以具有角色及其继承关系。这两个 RBAC 系统不会互相干扰。
我们这里不区分用户和角色，他们都是字符串，但是为了方便识别，建议增加前缀，例如user-uuid role-uuid,我们可以轻松的为某一用户，某一角色，某一角色组增加policy

把权限划分为几级:
超级管理权限集合 /*
各平台权限集合 /api/admin/*
各模块权限集合 /api/admin/collection/*
各API权限集合 /api/admin/collection/:id (这里默认是具有全部参数权限)
API深入到参数的权限集合 /api/admin/collection/xxxx?a=1 (这里默认是对单个api设置deny Policy)
设置超时失效权限(暂不实现)

根据业务划分为几种权限组(或者称为角色)
超级管理员
基本权限组- 除了管理员都会继承该组，简称基本人权组
菜单权限组- 根据菜单划分权限
特殊权限组- 各种收费权限，重要权限


整体的权限系统设计
可作为模块引入
可单独部署，提供鉴权API
提供ACL, RBAC，ABAC鉴权策略
对实施工程师提供仅限于比较简单的菜单管理

```
##### 实践
```txt
用了casbin后，我们自己的项目启动的时候，会读取权限
数据库到内存（map），这期间的查询会只是查询内存，不用查数据库。
权限的存储和修改，casbin先改内存，然后同步修改数据库。
如果自己写，还是比较麻烦的,只要我们设计好策略逻辑，casbin就能很好的
实现我们所需要。

```
- 表
```
casbin_rule: {
    ptype: string
    v0: string,
    v1: string,
    v2: string,
    v3: string,
    v4: string,
    v5: string
}

```
- 使用casbin
```txt
1. 使用casbin 鉴权
2. 使用koa-authz 适配我们koa框架
3. 使用casbin-sequelize-adapter 实现casbin_rule的创建
4. 使用swagger内容build出权限内容policy（暂未实现完全）


demo实现代码如下:
const { NotAuthError } = require('../exceptions/user')
const loadUser = require('./load_user')
const compose = require('koa-compose')
const casbin = require('casbin')
const authz = require('koa-authz')
const BasicAuthorizer = require('koa-authz/BasicAuthorizer')
const { SequelizeAdapter } = require('casbin-sequelize-adapter')
const postgresConfig = require('../config').postgres

class MyAuthorizer extends BasicAuthorizer {
  // override function
  getUserName () {
    const { id } = this.ctx.state.loginUser
    return id
  }
}

const { NotAuthError } = require('../exceptions/user')
const loadUser = require('../middlewares/load_user')
const compose = require('koa-compose')
const casbin = require('casbin')
const authz = require('koa-authz')
const BasicAuthorizer = require('koa-authz/BasicAuthorizer')
const { SequelizeAdapter } = require('casbin-sequelize-adapter')
const postgresConfig = require('../config').postgres

class MyAuthorizer extends BasicAuthorizer {
  // override function
  getUserName () {
    const { id } = this.ctx.state.loginUser
    return id
  }
}

class CasbinToPVmed {
  constructor() {
    this.whiteList = ['/user/sign_in', '/user/demo']
    this.allow = false
    this.enforcer = null
  }

  checkWhiteList() {
    return async (ctx, next) => {
      const url = ctx.url
      const result = this.whiteList.find((item) => url.indexOf(item)!== -1)
      this.allow = Boolean(result)
      return next()
    }
  }


  userAuth() {
    return compose([loadUser, async (ctx, next) => {
      if (this.allow) {
        return next()
      }
      if (ctx.state.loginUser) {
        return next()
      } else {
        return ctx.throws(new NotAuthError())
      }
    }])
  }
  async loadPolicys() {
     // load the casbin model and policy from files, database is also supported.
     const policys = await SequelizeAdapter.newAdapter({
      username: postgresConfig.auth.user,
      password: postgresConfig.auth.pwd,
      port: postgresConfig.port,
      host: postgresConfig.host,
      database: 'test7',
      dialect: 'postgres'
    })

    return policys
  }
  async initRBACStrategy() {
    const policys = await this.loadPolicys()
    const enforcer = await casbin.newEnforcer(`${__dirname}/authz_rbac_with_not_deny_model.conf`, policys)
    enforcer.enableLog(true)
    return enforcer
  }

  RBAC() {
    return (ctx, next) => {
      if (this.allow) {
        return next()
      }
      return authz({
        newEnforcer: async() => {
          this.enforcer = await this.initRBACStrategy()
          return this.enforcer
        },
        authorizer: (ctx, option) => new MyAuthorizer(ctx, option)
      })(ctx, next)
    }  
  }
}
const casbinToPVmed = new CasbinToPVmed()

module.exports = (app) => {
  app.use(casbinToPVmed.checkWhiteList())
  app.use(casbinToPVmed.userAuth())
  app.use(casbinToPVmed.RBAC())
}
```

- ABAC策略的尝试（暂未实现）
```
abac.conf:
[request_definition]
r = input, obj, act, matcher

[policy_definition]
p = act, obj, op, p1, p2, p3, p4

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.act == p.act \
    && regexMatch(r.obj, p.obj) \
    && abacMatcher(r.matcher, p.act, p.obj, r.input, p.op, p.p1, p.p2, p.p3, p.p4)

abac.policy:
p, view, patient:.*:.*, =, input.user.id, context.patient.id
```
