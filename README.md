# ThinkJS3 JWT 鉴权实践

JWT(JSON Web Tokens)一个提供基于JSON格式安全认证的token。

## 一、JWT 组成
JWT 由三部分组成，分别是 header(头部)，payload(载荷)，signature(签证) 这三部分以小数点连接起来。

本例中使用名为jwt-token的cookie来存储JWT例如：

jwt-token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoibHVzaGlqaWUiLCJpYXQiOjE1MzI1OTUyNTUsImV4cCI6MTUzMjU5NTI3MH0.WZ9_poToN9llFFUfkswcpTljRDjF4JfZcmqYS0JcKO8;

其中：

*   header = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9;
*   payload = eyJuYW1lIjoibHVzaGlqaWUiLCJpYXQiOjE1MzI1OTUyNTUsImV4cCI6MTUzMjU5NTI3MH0;
*   signature = WZ9_poToN9llFFUfkswcpTljRDjF4JfZcmqYS0JcKO8;


### 1. header
header 是对类型和哈希算法进行base64Encode之后得到的。对于比例中的header进行base64Decode可以得到：
{
  "alg":"HS256”,
  "typ":”JWT"
}

### 2. payload
payload 是对我们需要传输的信息进行base64Encode之后得到的。对于本例中的payload进行base64Decode可以得到：
{
  "name":"lushijie”,
  "iat":1532595255, // JWT 发布的时间
  "exp”:1532595270 // JWT 过期的时间，15秒后过期
}
本例中的iat, exp 是 koa-jwt 中的默认字段，初此之外 JWT 标准中注册的非强制使用的声明还有 jti，iss等，有兴趣的小伙伴可以查看更多的相关标准。
由于 payload 可以在客户端解码获得，所以不建议在 payload 中存放敏感信息，例如用户的密码。

### 3. signature
signature 包含了 header，payload 和 密钥，计算公式如下：

```
const encodedString = base64Encode(header) + "." + base64Encode(payload);
let signature = HMACSHA256(encodedString, '密钥');
```
这里密钥是保存在服务端的，客户端是不知道的。

## 二、JWT 验证

对于验证一个 JWT 是否有效也是比较简单的，服务端根据前面介绍的计算方法计算出 signature，和要校验的JWT中的 signature 部分进行对比就可以了，如果 signature 部分相等则是一个有效的 JWT。

## 三、 JWT 在 ThinkJS3 中的实践

下面我们在 ThinkJS3 中(单模块项目)实现使用 JWT 实现只有在登陆后才能访问一个接口。
ThinkJS3 兼容 koa2 的所有middleware，那就找个现成的 jwt 插件吧，koa-jwt!
koa-jwt 代码没有几行，大家可以稍微读一下，简单易懂，接下来我们开始使用它~
首先我们要在 ThinkJS3 中配置 koa-jwt:

### 1. 公共配置

[project path]/src/config/config.js:

```
  module.exports = {
    // ...
    jwt: {
      secret: 'lushijie-password',
      cookie: 'jwt-token',
      expire: 30 // 秒
    },
  }
```
因为这三个参数在不同的位置会用到，为了统一管理我们提取到了公共的 config 中。


### 2. 中间件配置

[project path]/src/config/middleware.js:

```
const jwt = require('koa-jwt');
const isDev = think.env === 'development';

module.exports = [
  // ...
  {
    handle: jwt,
    // match(ctx) {
      // return !/^\/index\/login/.test(ctx.path);
    // },
    options: {
      cookie: think.config('jwt')['cookie'],
      secret: think.config('jwt')['secret'],
      passthrough: true
    }
  },

  // payload 这里配置因为本例中 jwt 并没有用到 request 解析后的参数
];
```

起初我想通过配置 match 参数来决定某个 URL 是否需要登陆认证，后来发现这样需要配置好多的正则，比较麻烦；
其次 koa-jwt 没有提供无权访问自定义错误的钩子，所以放弃了 match 的方案。

这里采用了 koa-jwt 提供的配置 `passthrough: true`，这个参数让我们不管权限验证通过与否都可以继续执行后面的中间件，只是在当前的 ctx 上设置了 payload。

我们错误处理需要在 logic 层进行，而不应该在 controller 层，否则会出现以下问题：如果 logic 层有有参数校验不通过同时无权访问，会先报参数校验不通过信息，然后再报无权访问，这显然是不符合要求的。

### 3. 扩展 think.Controller 
 
这里我们对 think.Controller 做了扩展，这里没有对 think.Logic 上进行扩展是因为 think.Logic 继承自 think.Controller。

[project]/src/extend/controller.js:

```
const jsonwebtoken = require('jsonwebtoken');
module.exports = {
  authFail() {
    return this.fail('JWT 验证失败');
  },

  checkAuth(target, name, descriptor) {
    const action = descriptor.value;
    descriptor.value = function() {
      console.log(this.ctx.state.user);
      const userName = this.ctx.state.user && this.ctx.state.user.name;
      if (!userName) {
        return this.authFail();
      }
      this.updateAuth(userName);
      return action.apply(this, arguments);
    }
    return descriptor;
  },

  updateAuth(userName) {
    const userInfo = {
      name: userName
    };
    const {secret, cookie, expire} = this.config('jwt');
    const token = jsonwebtoken.sign(userInfo, secret, {expiresIn: expire});
    this.cookie(cookie, token);
    return token;
  }
}
```

其中 authFail 是 JWT 验证失败的操作；updateAuth 是更新 JWT，此处使用 jsonwebtoken 生成 JWT 并种 cookie；checkAuth 使用了 deractor 方式实现，当然你也可以使用你喜欢的方式。
    
此处使用 cookie 的方式记录生成的JWT, 当初也可以采用别的方式储存，koa-jwt 提供了 getToken 让我们能够自由的获取 JWT, 此处不再详述。


### 4. controller 业务逻辑

[project path]/src/controller/jwt1.js

```
const userList = {
  lushijie: '123123',
  xiaoming: '456456'
};

module.exports = class extends think.Controller {
  async userAction() {
    const userInfo = this.ctx.state.user;
    if (userInfo) {
      return this.success(userInfo);
    } else {
      return this.fail('获取用户信息失败');
    }
  }

  loginAction() {
    const {name, password} = this.get();
    if (userList[name] && password === userList[name]) {
      const token = this.updateAuth(name);
      return this.success(token);
    } else {
      return this.fail('登陆失败');
    }
  }

  logoutAction() {
    this.updateAuth(null);
    return this.success('退出登录成功');
  }
}
```
jwt1 这个简单的 controller 包含了三个简单的功能，登陆、退出与获取用户信息，其中获取用户信息要求必须登陆之后才可以访问。
这里的登陆只是进行了一个简单的模拟，真实项目中的用户验证会比这个复杂一些，原理是一致的。


### 5. Logic 权限验证

[project path]/src/logic/jwt1.js

```
let checkAuth = think.Controller.prototype.checkAuth;
module.exports = class extends think.Logic {
  @checkAuth
  userAction(){
    // 正常的参数验证逻辑
  }
}
```

这样一个验证就完成了！ 如果该 Logic 中的所有 action 都需要进行验证可以给 __before 加 deractor 就可以了！


