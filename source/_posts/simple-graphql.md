---
title: 简单介绍一下GraphQL
date: 2018-09-03 20:02:00
categories: 技术学习
tags:
  - nodejs
  - graphql
  - restful
---

> 前段时间，莫名的有好多小朋友 `fork` 我的博客项目，然后想起了“噢，原来我还有一个博客”啊。但是不得不说的是，`fork` 博客这种行为是不提倡的，面试官一眼就看出来了，年轻的小朋友们，还是要踏踏实实的学习，不要搞歪风邪气！

> 本文是我上周在公司的个人分享里边的一些记录，因为也确实没有在实际项目中使用过[`GraphQL`](https://graphql.org/)，所以应该会有很多的不足之处，同样的，讲的也不够深入，仅供了解之用吧。

# 什么是GraphQL?

[`GraphQL`](https://graphql.org/)（Graph Query Language）本质上就是一个http接口的设计风格，一种接口查询语言，这一点跟 `restful` 是一样的。但它们又有本质上的区别： `restful` 是面向资源的，而[`GraphQL`](https://graphql.org/)则是面向全业务的。

# 为什么要使用GraphQL？

为了更好说明它与restful之间的区别，以及更好的说明[`GraphQL`](https://graphql.org/)，下面举一个简单的例子。

假如前端要写这样子的一个页面：展示用户界面时，除了要展示用户的email,phone等信息以外，还要把用户的好友的email，phone信息给展示出来。

对于后端人员而已，这可能就是数据库里边的一张表的事情:
```
Account: {
  name: String,
  email: String,
  phone: String,
  friends: [String<Account-id>]
}
```
使用 `restful` 接口，也就是暴露出来一个接口：`/accounts`来进行增删查改。

但是对于前端开发人员而言，这件事可能就复杂了一些：
1. 拿到用户ID信息，根据`/accounts/:ID`获取到用户信息；
2. 根据上边获取到的用户信息的`friends`字段，然后拿到里边的`friend_id`字段，发送请求`/accounts/:friend_id`来获取该用户的朋友信息；

问题估计大家应该能看出来了，对于前端开发人员而言，这种体验很不友好，这意味着，该用户的朋友用户越多，前端就要发送越多的请求来获取信息，然后拼凑出想要的数据。那么，我们接下来应该要怎么解决呢？

下边，我列举几个方案：
1. 忍
2. 买一杯星巴克，拍拍后端人员的肩膀，说声：“兄弟，那个展示用户的界面的接口啊...”
3. 自己撸起袖子，准备在前端跟后端之间构建一个业务层，把复杂的请求在这一层整合;

第二种方案，很明显很花钱，而且你要跟后端的人员关系很铁。而且从整个技术架构而言，这类的处理，放在 `restful api` 这一层是不是不是很好呢？

第三种方案，相信会是业务越来越复杂后，大家的普遍的选择。比较著名的案例应该就是淘宝的中途岛了。

那么，使用第三种方案，问题是不是已经解决了呢？基本上...是吧？但还是有瑕疵：假如你的客户端比较多，而每个端又都需要不同的数据，不同的字段。那这个中间层（proxy-middleware）的接口就会越来越多，以及越来越复杂。而有时候你为了偷懒，会把业务需求相似的接口合并成一个，这就会造成了“过载”的现象。（脑海里边脑补了萨满在水晶够用的时候，还疯狂的扔过载卡...）

而[`GraphQL`](https://graphql.org/)就是为了解决上边这些问题而提出来的。

根据官方建议，[`GraphQL`](https://graphql.org/)的接口只有一个：method为`POST`的`/graphql`接口。其他的查询的参数就全都放置在`query`或者`body`里边了。下边举例的话，这里采用的是`body`，毕竟用 `restful` 这么多年，思维定势了...以下皆采用`js/node.js`来编写代码演示。

## 前端请求

与 `restful` 容易造成“过载”不同的是，[`GraphQL`](https://graphql.org/)在前端发起请求的时候，就已经把需要的字段给安排好了。像上边查询用户的例子，请求时的字段大概就是这样子：

```
{
  # 查询参数为id：'123'的用户信息
  account(id: '123') {
    name
    email
    phone
    friends {
      name
      email
      phone
    }
  }
}
```

嗯嗯，是的，以上就是我们需要在body里边添加的内容，只需要把前端需要的字段信息填写上，就可以得到对应的字段信息进行渲染了。对比与 `restful` 的潜在的指数性爆发请求要轻松的多。

而除了查询以外，我们还需要增删改的操作，同样的[`GraphQL`](https://graphql.org/)提供了对应的操作，那就`mutation`，下边以添加一名用户为例：

```
mutation {
  createAccount (
    name: "lfz"
    email: 'classlfz@qq.com'
    phone: '12345678910'
  ) {
    name
  }
}
```

这里我们在创建完成后，只从创建成功后的新用户里边获取`name`字段。[`GraphQL`](https://graphql.org/)比较方便的一点就是指定你需要的字段信息，任何其他的字段信息都不应该出现的。

## 后端编写

### Schema

`Schema` 是 [`GraphQL`](https://graphql.org/) 里边一个比较重要的概念，我这里就翻译为模式或者图表吧。这个概念应该是[`GraphQL`](https://graphql.org/)的名称由来，在[`GraphQL`](https://graphql.org/)里边，查询其实按照图表来进行的。而 `Schema` 则是由众多的 `Type` 来构成的。

### Type

而 `Type` 则是 [`GraphQL`](https://graphql.org/)里边另外一个重要的概念，它允许我们自己创建一个数据结构作为一个类型来进行查询。上边查询用户的例子，在[`GraphQL`](https://graphql.org/)的 `Type` 来看，大概就是这样子：

```
type Account {
  name: String
  email: String
  phone: String
  friends: [Account]
}
```

### koa & koa-graphql

我们这里就采用[`koa`](https://koajs.com/)框架以及对应的[`koa-graphql`](https://github.com/chentsulin/koa-graphql)库来编写[`GraphQL`](https://graphql.org/)的服务端：

```js
import Koa from 'koa'
import mount from 'koa-mount'
import graphqlHTTP from 'koa-graphql'
import MyGraphqlSchema from './schema'

const app = new Koa()

// 配置graphql接口
app.use(mount('/graphql', graphqlHTTP({
  schema: MyGraphqlSchema,
  graphiql: true
})))

app.listen(4000)

console.log('graphQL server listen port: ' + 4000)
```

接下来，我们就需要编写 `schema.js` 来定义接口下的图表的内容：

```js
import {
  GraphQLObjectType,
  GraphQLNonNull,
  GraphQLSchema,
  GraphQLString,
  GraphQLList
} from 'graphql/type'

var accountType = new GraphQLObjectType({
  name: 'Account',
  description: 'Account creator',
  fields: () => ({
    name: {
      type: GraphQLString,
      description: 'The name of the account.',
    },
    email: {
      type: new GraphQLNonNull(GraphQLString),
      description: 'The email of the account.',
    },
    phone: {
      type: new GraphQLNonNull(GraphQLString),
      description: 'The phone of the account.',
    },
    friends: {
      type: new GraphQLList(accountType),
      description: 'The friends of the account, or an empty list if they have none.',
      resolve: (obj, args, context, info) => {
        console.log('*** Obj: ', obj)
        console.log('*** Args: ', args)
        console.log('*** Context: ', context)
        console.log('*** Info: ', info)
        // 假装自己又去数据库查询
        return [{id: '234', name: 'classlfz', email: 'classlfz@qq.com', phone: '12345678910'}]
      },
    }
  })
})

var schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: 'RootQueryType',
    fields: {
      account: {
        type: accountType,
        args: {
          id: {
            name: 'id',
            type: new GraphQLNonNull(GraphQLString)
          }
        },
        resolve: (root, args, context, info) => {
          // console.log(root, args, context, info)
          // 假装自己有去数据库查询
          return {id: '123', name: 'lfz', email: 'classlfz@qq.com', phone: '12345678910'}
        }
      }
    }
  }),

  // mutation
  mutation: new GraphQLObjectType({
    name: 'Mutation',
    fields: {
      createAccount: {
        type: accountType,
        args: {
          name: {
            name: 'name',
            type: GraphQLString
          },
          email: {
            name: 'email',
            type: GraphQLString
          },
          phone: {
            name: 'phone',
            type: GraphQLString
          }
        },
        resolve: (obj, {name}, source, info) => {
          // 假装自己有去数据库添加
          return {name: 'lfz', email: 'classlfz@qq.com', phone: '12345678910', friends: []}
        }
      }
    }
  })
})

export default schema
```

就这样，我们完成了一个简单的使用node.js编写的[`GraphQL`](https://graphql.org/)服务了。接下来，我们开启这个服务，再使用前端人员熟悉的 `postman` 工具来查看一下结果是如何的：

### 查询用户信息

![查询](/images/simple_graphql_query_postman.jpg)

### 创建用户信息

![创建用户](/images/simple_graphql_mutation_create_postman.jpg)


## 自检性

对于[`GraphQL`](https://graphql.org/)的这个特性，我本人是感觉有点黑科技的感觉的（原谅我的无知~）。它允许你通过graphql接口来查询整个图表的可查询字段！利用这一点，我们可以利用一些写好的工具来生成我们的接口文档，这对于厌倦写文档的人员，简直不要太好啊！

### 自检

![自检](/images/simple_graphql_self_inspection_postman.jpg)

下边演示一下，使用node的一个库graphdoc来生成接口文档

```sh
$ graphdoc -e http://localhost:4000/graphql -o ./doc/schema
```

这样，我们就在本地`./doc/schema`得到了一个`html`接口文档了~非常的方便。

# 为什么还不火？

上边说了那么多的[`GraphQL`](https://graphql.org/)好的地方，那么为什么[`GraphQL`](https://graphql.org/)没有火起来呢？不火总是有原因的嘛，优择略汰，下边这里罗列一些大家普遍认为的[`GraphQL`](https://graphql.org/)不足的地方：

1. 复杂度一直就在那里。是的，业务的复杂度一直都在那里，无论是前端人员忍受了那几十条的ajax请求（相信不会真有人这么能忍吧？），还是中间件来处理，亦或是后端编写[`GraphQL`](https://graphql.org/)接口，业务的复杂度都没有被消失。然后本来是前端的痛处，然后现在你特么让我后端来写？滚。

2. 好好好，你后端不写，没关系，谁还不会个后端语言啊，我用node.js来写一个中间件，在里边做一个转发，把你们的 `restful` 转换成graphql不就好了？嗯嗯，是的，这个或许应该是比较好的方案了，而且要比传统的 `restful` 中间件要好一些，因为一旦部署开来，graphQl的维护成本是要低于 `restful` 的，但是查询的速度跟效率其实并没有提高很多，而且比较容易在这一环节出现性能瓶颈，还有，项目就要维护多一个份代码的同时，还要求前端人员对后端有一定的了解...

3. 现在大家为了项目的可维护性，以及技术的更新迭代，更多的都会采用微服务的形式来编写，而 `restful` 天生就符合了微服务的形式，各自只需要维护好自己的接口就好了。而[`GraphQL`](https://graphql.org/)则是相反的，它设计出来是为了集合这些服务的，只暴露出一个接口，前端固然舒服，但后端服务的编写就不可避免的失去了一定程度的解耦性。

# 总结

总的来说，还是要以辨证的态度来看待这项技术吧。[`GraphQL`](https://graphql.org/)固然解决了 `restful` 的一些痛点，但有不可避免的引发了另外一些问题。但是比作其他的技术一样，只有最合适，没有最完美。