---
title: nestjs搭建HTTP与WebSocket服务
date: 2022-11-22
tags:
 - nestjs
 - http
 - websocket
categories:
  - 技术
---

最近在做一款轻量级IM产品，后端技术栈框架使用了nodejs + nestjs作为服务端。同时，还需要满足一个服务同时支持HTTP服务调用以及WebSocket服务调用，此文主要记录本次搭建过程，以及基本的服务端设计。

<!-- more -->

# 基本环境搭建

node v14.17.5

nestjs 全局命令行工具（`npm i -g @nestjs/cli`）

本文不再详细介绍nestjs各种概念，请参考：[First steps | NestJS - A progressive Node.js framework](https://docs.nestjs.com/first-steps)

直接创建一个Demo项目：

```
nest new nest-http-socket-demo
```

## 目录划分设计

等待项目完成以后（这个过程可能会持续比较久，因为创建好目录结构以后还会进行包安装），结构如下：

```
nest-http-websocket-demo
├─ .eslintrc.js
├─ .gitignore
├─ .prettierrc
├─ README.md
├─ nest-cli.json
├─ node_modules
│    └─ ... ...
├─ package.json
├─ src
│    ├─ app.controller.spec.ts
│    ├─ app.controller.ts
│    ├─ app.module.ts
│    ├─ app.service.ts
│    └─ main.ts
├─ test
│    ├─ app.e2e-spec.ts
│    └─ jest-e2e.json
├─ tsconfig.build.json
├─ tsconfig.json
└─ yarn.lock
```

初始的目录结构可能不太符合我们的期望，我们对目录结构进行适当的调整。主要分为几个目录：

1. src/common。该目录存放服务端和客户端公共涉及的内容。方便后续拆分出单独的npm包供服务端和客户端公用；
2. src/base。该目录存放整个服务需要用到的一些基础内容，譬如拦截器、过滤器等；
3. src/module。后续存放按照不同的业务领域拆分出的子目录；
4. src/entity。存放数据定义等（本项目我们简化模型，认为数据传输的结构和服务中领域数据结构一致）。

调整后的src目录结构如下：

```
- src
  ├─ base
  ├─ common
  ├─ entity
  └─ module
```

## 基础类型定义

在规划API之前，我们先设计定义一些服务端基本数据结构。

### 服务端响应封装（ServerResponseWrapper）

众所周知，一般的服务端都会对原始返回数据进行一定的包装，增加返回码、错误消息等来明确的指出具体的错误内容，在我们的服务也不例外。于是，我们设计如下的结构体：

```typescript
export interface ServerResponseWrapper {
    /**
     * 服务端返回码
     */
    returnCode: string;
    /**
     * 错误信息（如有，例如返回码非成功码）
     */
    errorMessage?: string;
    /**
     * 返回数据（如有）
     */
    data?: any;
}
```

对于该结构来说，后续客户端也会使用相同的数据结构进行解析，所以我们可以考虑将该文件放在src/common中。

下面是一些常见的返回数据（纯样例）：

```json
// 获取用户基本信息成功
{
    "returnCode": "SUC00000",
    "data": {
        "username": "w4ngzhen",
        "lastLoginTime": "2022-11-22 11:50:22.000"
    }
}
// 获取用户名称出错（没有提供对应的userId）
{
    "returnCode": "ERR40000",
    "errorMessage": "user id is empty.",
}
// 获取服务端时间
{
    "returnCode": "SUC0000",
    "data": "2022-11-22 11:22:33.000"
}
```

### 返回码定义（ReturnCode）

为了统一返回码，我们在定义了一个ReturnCode实体类，由该类统一封装返回码。作为外部会涉及了解到的内容，我们也将该类放置于src/common中，且导出常用的错误码，代码如下：

```typescript
export class ReturnCode {

    private readonly _preCode: 'SUC' | 'ERR';
    private readonly _subCode: string;

    private readonly _statusCode: number;

    get codeString(): string {
        return `${this._preCode}${this._subCode}`;
    }

    get statusCode(): number {
        return this._statusCode;
    }

    constructor(prefix: 'SUC' | 'ERR', subCode: string, statusCode: number) {
        this._preCode = prefix;
        this._subCode = subCode;
        this._statusCode = statusCode;
    }
}

export const SUCCESS = new ReturnCode('SUC', '00000', 200);
export const ERR_NOT_FOUND = new ReturnCode('ERR', '40400', 404);
```

### 服务业务异常（BizException）

为了便于在服务调用过程中，能够按照具体的业务层面进行异常抛出。我们定义一个名为BizException的类来封装业务异常。对于外部系统来说，该异常并不可见，所以我们把该类放置于src/base中：

```typescript
import {ReturnCode} from "../common/return-code";

export class BizException {

    private readonly _errorCode: ReturnCode;
    private readonly _errorMessage: string;

    get errorCode(): ReturnCode {
        return this._errorCode;
    }

    get errorMessage(): string {
        return this._errorMessage;
    }

    protected constructor(errorEntity: ReturnCode, errorMessage: string) {
        this._errorMessage = errorMessage;
        this._errorCode = errorEntity;
    }

    static create(errEntity: ReturnCode, errMessage?: string): BizException {
        return new BizException(errEntity, errMessage);
    }
}
```

接下来，我们为服务器规划两个API，分别体现HTTP服务和WebSocket服务。

# HTTP服务开发

## 基础服务

首先，我们设计一个简单用户信息查询服务接口。该接口可以根据传递而来的用户ID（userId）返回对应的用户信息：

`GET /users?userId=${userId}`

为了实现上述接口，我们按照如下流程进行API搭建：

1. 在src/entity目录中，我们创建一个user目录，并在其中创建user.dto.ts文件专门用于定义用户User这个数据传输结构，内容如下：

```typescript
// src/entity/user/user.dto.ts
export interface UserDto {
    userId: string;
    username: string;
    age: number;
}
```

2. 在src/module创建一个user目录，划分用户user相关业务领域内容。同时，在其中创建user.service.ts，存放处理用户的相关服务代码，内容如下：

```typescript
// src/module/user/user.service.ts
import {Injectable} from '@nestjs/common';
import {UserDto} from "../../entity/user/user.dto";

@Injectable()
export class UserService {

    async getUserById(userId: string): Promise<UserDto> {
        // 测试数据
        const demoData: UserDto[] = [
            {
                userId: 'tom',
                username: 'Tom',
                age: 10
            },
            {
                userId: 'jerry',
                username: 'Jerry',
                age: 11
            }
        ];

        return demoData.find(u => u.userId === userId);
    }
}
```

3. 同样的，我们在src/module/user中创建User的Controller（`user.controller.ts`），增加`GET /users`接口，请求参数并调用服务：

```typescript
import {Controller, Get, Param, Query} from '@nestjs/common';
import {UserService} from './user.service';
import {UserDto} from "../../entity/user/user.dto";

@Controller("users")
export class UserController {
    constructor(private readonly userService: UserService) {
    }

    @Get()
    async getHello(@Query('userId') userId: string): Promise<UserDto> {
        return this.userService.getUserById(userId);
    }
}
```

4. 创建用户模块，将controller、service注册到用户模块中（`src/module/user/user.module.ts`）：

```typescript
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';

@Module({
  imports: [],
  controllers: [UserController],
  providers: [UserService],
})
export class UserModule {}
```

5. 将用户模块注册给全局总模块app.module.ts中：

```diff
 import { AppService } from './app.service';
+import {UserModule} from "./module/user/user.module";

 @Module({
-  imports: [],
+  imports: [UserModule],
   controllers: [AppController],
   providers: [AppService],
 })
```

完成上述操作以后，我们就可以启动服务进行验证了：

![010-http-api-origin-output](https://static-res.zhen.wang/images/post/2022-11-22/010-http-api-origin-output.png)

## 成功响应拦截器

上面的接口返回可以看出，Controller返回是什么样的结构体，前端请求到的数据就是什么结构，但我们希望将数据按照ServerResponseWrapper结构进行封装。在nestjs中，可以通过实现来自`@nestjs/common`中的`NestInterceptor`接口来编写我们自己的响应拦截，统一处理响应来实现前面的需求。按照我们之前规划，我们首先在src/base中创建interceptor目录，然后在里面创建`http-service.response.interceptor.ts`，内容如下：

```typescript
// src/base/interceptor/http-service.response.interceptor.ts
import {CallHandler, ExecutionContext, NestInterceptor} from "@nestjs/common";
import {map, Observable} from "rxjs";
import {ServerResponseWrapper} from "../../common/server-response-wrapper";
import {SUCCESS} from "../../common/return-code";

/**
 * 全局Http服务响应拦截器
 * 该Interceptor在main中通过
 * app.useGlobalInterceptors 来全局引入，
 * 仅处理HTTP服务成功响应拦截，异常是不会进入该拦截器
 */
export class HttpServiceResponseInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext,
              next: CallHandler):
        Observable<any> | Promise<Observable<any>> {
        return next.handle().pipe(map(data => {
            // 进入该拦截器，说明没有异常，使用成功返回
            const resp: ServerResponseWrapper = {
                returnCode: SUCCESS.codeString,
                data: data
            };
            return resp;
        }))
    }
}
```

创建完成后，我们在main入口中，需要将该响应拦截器注册到全局中：

```diff
 // src/main.ts
 async function bootstrap() {
   const app = await NestFactory.create(AppModule);
+
+  // 增加HTTP服务的成功响应拦截器
+  app.useGlobalInterceptors(new HttpServiceResponseInterceptor());
+
   await app.listen(3000);
 }
 bootstrap();
```

完成配置以后，我们可以再次调用API来查看结果：

![020-http-api-success-wrapper-output](https://static-res.zhen.wang/images/post/2022-11-22/020-http-api-success-wrapper-output.png)

可以看到，尽管我们的Controller返回的是一个实际数据结构（Promise也适用），但是经过响应拦截器的处理，我们完成了对响应体的包裹封装。

## 异常过滤器

上述我们完成一个调用，并对响应成功的数据进行了包裹，但面对异常情况同样适用吗？如果不适用又需要如何处理呢？

首先，我们增加一个专门处理字段错误的错误码ReturnCode：

```diff
// src/common/return-code.ts
 export const SUCCESS = new ReturnCode('SUC', '00000', 200);
+export const ERR_REQ_FIELD_ERROR = new ReturnCode('ERR', '40000', 400);
 export const ERR_NOT_FOUND = new ReturnCode('ERR', '40400', 404);
```

然后，我们在UserService中适当修改一下getUserById的实现，加入userId判空判断，并在为空的时候，抛出业务异常（这个过程我们顺便安装了lodash）：

```diff
+import * as _ from 'lodash';
+import {BizException} from "../../common/biz-exception";
+import {ERR_REQ_FIELD_ERROR} from "../../common/return-code";

 @Injectable()
 export class UserService {

     async getUserById(userId: string): Promise<UserDto> {
+        if (_.isEmpty(userId)) {
+            throw BizException.create(ERR_REQ_FIELD_ERROR, 'user id is empty');
+        }
         ... ...
     }
}
```

完成上述修改后，我们尝试发请求时候，故意不填写userId，得到如下的结果：

![030-http-api-error-origin-output](https://static-res.zhen.wang/images/post/2022-11-22/030-http-api-error-origin-output.png)

可以看到，尽管nestjs帮助我们进行一定的封装，但是结构体与我们一开始定义的ServerResponseWrapper是不一致的。为了保持一致，我们需要接管nestjs的异常处理，并转换为我们自己的wrapper结构，而接管的方式则是创建一个实现ExceptionFilter接口的类（按照路径划分，我们将这个类所在文件`http-service.exception.filter.ts`存放于src/base/filter目录下）：

```typescript
import {ArgumentsHost, Catch, ExceptionFilter, HttpException} from "@nestjs/common";
import {ServerResponseWrapper} from "../../common/server-response-wrapper";
import {BizException} from "../../common/biz-exception";

/**
 * 全局Http服务的异常处理，
 * 该Filter在main中通过
 * app.useGlobalExceptionFilter来全局引入，
 * 仅处理HTTP服务
 */
@Catch()
export class HttpServiceExceptionFilter implements ExceptionFilter {
    catch(exception: any, host: ArgumentsHost): any {
        // 进入该拦截器，说明http调用中存在异常，需要解析异常，并返回统一处理
        let responseWrapper: ServerResponseWrapper;
        let httpStatusCode: number;
        if (exception instanceof BizException) {
            // 业务层Exception
            responseWrapper = {
                returnCode: exception.errorCode.codeString,
                errorMessage: exception.errorMessage
            }
            httpStatusCode = exception.errorCode.statusCode;
        } else if (exception instanceof HttpException) {
            // 框架层的Http异常
            responseWrapper = {
                returnCode: 'IM9009',
                errorMessage: exception.message,
            }
            httpStatusCode = exception.getStatus();
        } else {
            // 其他错误
            responseWrapper = {
                returnCode: 'IM9999',
                errorMessage: 'server unknown error: ' + exception.message,
            };
            httpStatusCode = 500;

        }

        // 该拦截器处理HTTP服务的异常，所以手动切换到HTTP Host
        // 并获取响应response，进行HTTP响应的写入
        const httpHost = host.switchToHttp();
        const response = httpHost.getResponse();
        response.status(httpStatusCode).json(responseWrapper);
    }
}
```

该类的核心点在于，对捕获到的异常进行解析后，我们会通过参数ArgumentsHost来获取实际的HTTP Host，并从中获取response对象，调用相关支持的方法来控制响应response的内容（http状态码以及响应体内容）。

最后，我们依然在main里面进行注册配置：

```diff
+import {HttpServiceExceptionFilter} from "./base/filter/http-service.exception.filter";

 async function bootstrap() {
   const app = await NestFactory.create(AppModule);

   // 增加HTTP服务的成功响应拦截器
   app.useGlobalInterceptors(new HttpServiceResponseInterceptor());
+  // 增加HTTP服务的异常过滤器，进行响应包裹
+  app.useGlobalFilters(new HttpServiceExceptionFilter());

   await app.listen(3000);
 }
```

完成开发配置以后，我们重启服务，通过调用接口可以看到对应异常返回：

![040-http-api-error-wrapper-output](https://static-res.zhen.wang/images/post/2022-11-22/040-http-api-error-wrapper-output.png)

# WebSocket服务

在nestjs中想要集成WebSocket服务也很容易。

首先，我们使用一个装饰器`@WebSocketGateway()`来表明一个类是一个WebSocket的网关（Gateway），这个装饰器可以指定WebSocket服务的端口等信息。通常情况下，我们可以设置与HTTP服务不一样的端口，这样我们就可以在一个台服务上通过不同的端口暴露HTTP和WebSocket服务。当然，这不是必须，只是为了更好的区分服务。

其次，我们需要明白在nestjs可以使用ws或者socket.io两种具体实现的websocket平台。什么是具体平台？简单来讲，nestjs只负责设置一个标准的WebSocket网关规范，提供通用的API、接口、装饰器等，各个平台则是根据nestjs提供的规范进行实现。**在本例中，我们选择使用socket.io作为nestjs上WebSocket具体的实现，因为socket.io是一个比较著名websocket库，同时支持服务端和客户端，并且在客户端/服务端均内建支持了"请求 - 响应"一来一回机制。**

## 前置准备

**依赖安装**

nestjs中的websocket是一个独立的模块，且我们选取了socket.io作为websocket的实现，所以我们需要首先安装一下的基础模块：

```shell
yarn add @nestjs/websockets @nestjs/platform-socket.io
```

**网关创建**

websocket的相关内容，我们同样作为一种模块进行编写。于是，我们在src/module/目录中创建websocket文件夹，并在里面创建一个文件：my-websocket.gateway.ts，编写WS网关MyWebSocketGateway类的内容：

```typescript
import {WebSocketGateway} from "@nestjs/websockets";

@WebSocketGateway(4000, {
    transports: ['websocket']
})
export class MyWebSocketGateway {

}
```

一个简单的WebSocket网关就创建完成了。我们首先设定了WebSocket服务的端口号为4000（与HTTP服务的3000隔离开）；其次，需要特别提一下transports参数，可选择的transport有两种：

>  polling（HTTP长连接轮询）

该机制由连续的 HTTP 请求组成：

- 长时间运行的请求，用于从服务器接收数据`GET`
- 短运行请求，用于将数据发送到服务器`POST`

由于传输的性质，连续的发出可以在同一 HTTP 请求中连接和发送。

也就是说，polling本质上是利用HTTP请求+轮询来完成所谓的双工通讯，在某些古老的没有实现真正WebSocket协议的浏览器作为一种实现方案。

>  websocket（网络套接字）

WebSocket 传输由[WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) 连接组成，该连接在服务器和客户端之间提供双向和低延迟的通信通道。这是真正的长连接双工通讯协议。

所以，在通讯的过程中，服务端与客户端要保持相匹配的传输协议。

**模块创建注册**

同样的，我们在src/module/websocket中创建一个my-websocket.module.ts文件，内容如下：

```typescript
import {MyWebSocketGateway} from "./my-websocket.gateway";
import {Module} from "@nestjs/common";

@Module({
    providers: [MyWebSocketGateway]
})
export class MyWebSocketModule {

}
```

主要内容是将MyWebSocketGateway注册到模块中。

最后我们将MyWebSocket模块注册到根模块中：

```diff
+import {MyWebSocketModule} from "./module/websocket/my-websocket.module";

 @Module({
-  imports: [UserModule],
+  imports: [UserModule, MyWebSocketModule],
   controllers: [AppController],
   providers: [AppService],
 })
export class AppModule {}
```

## 基础服务

我们先设定这样一个场景：客户端连接上WebSocket服务后，可以给服务端发送一份JSON数据（内容加下方），服务端校验该数据后，在控制台打印数据。

```json
{
    "name": "w4ngzhen"
}
```

对于服务端来说，我们首先需要订阅事件（subscribe），假设发送JSON数据的事件为`hello`，那么我们可以通过如下的方式来进行订阅：

```typescript
export class MyWebSocketGateway {

    @SubscribeMessage('hello')
    hello(@MessageBody() reqData: { name: string }) {
        if (!reqData || !reqData.name) {
            throw BizException.create(ERR_REQ_FIELD_ERROR, 'data is empty');
        }
        console.log(JSON.stringify(reqData));
    }
    
}
```

测试WebSocket，可以使用postman来进行，只需要创建个一WebSocket的请求，在postman中按下CTRL+N（macOS为command+N），可以选择WebSocket请求：

![050-create-websocket](https://static-res.zhen.wang/images/post/2022-11-22/050-create-websocket.png)

创建后，需要注意，由于我们nestjs集成的WebSocket实现使用的socket.io，所以客户端需要匹配对应的实现（这点主要是为了匹配”请求-响应“一来一回机制）

![060-choose-socketio](https://static-res.zhen.wang/images/post/2022-11-22/060-choose-socketio.png)

完成配置后，我们可以采用如下的步骤进行事件发送：

![070-postman-websocket-send](https://static-res.zhen.wang/images/post/2022-11-22/070-postman-websocket-send.png)

发送完成后，就会看到postman的打印和nodejs服务控制台的打印，符合我们的预期：

![080-websocket-event-origin-output](https://static-res.zhen.wang/images/post/2022-11-22/080-websocket-event-origin-output.png)

当然，我前面提到过socket.io支持事件一来一回的请求响应模式。在nestjs中的WebSocket网关，只需要在对应的请求返回值即可：

```diff
     @SubscribeMessage('hello')
     hello(@MessageBody() reqData: { name: string }) {
         if (!reqData || !reqData.name) {
             throw BizException.create(ERR_REQ_FIELD_ERROR, 'data is empty');
         }
         console.log(JSON.stringify(reqData));
+        return 'received reqData';
     }
```

在postman的地方，我们需要发送的时候勾选上`Acknowledgement`：

![090-set-acknowledgement](https://static-res.zhen.wang/images/post/2022-11-22/090-set-acknowledgement.png)

完成以后，我们重新连接服务并发送数据，就可以看到一条完整的事件处理链路了：

![100-send-and-receive-origin](https://static-res.zhen.wang/images/post/2022-11-22/100-send-and-receive-origin.png)

至此，我们就完成了在Nestjs集成一个基础的WebSocket服务了。

当然，我们的工作还没有结束。在前面我们对HTTP服务编写了成功响应拦截器以及异常过滤器，接下来，我们按照同样的方式编写WebSocket的相关处理。

## 成功响应拦截器

对于集成在nestjs中的WebSocket服务，想要编写并配置一个成功响应拦截器并不复杂，没有什么坑。

首先，我们仿照着http-service.response.interceptor.ts，编写一个几乎完全一样的ws-service.response.interceptor.ts，与HTTP的成功响应拦截器放在相同目录src/base/interceptor中：

```typescript
// src/base/interceptor/ws-service.response.interceptor.ts
import {CallHandler, ExecutionContext, NestInterceptor} from "@nestjs/common";
import {map, Observable} from "rxjs";
import {ServerResponseWrapper} from "../../common/server-response-wrapper";
import {SUCCESS} from "../../common/return-code";

/**
 * 全局WebSocket服务响应拦截器
 * 该Interceptor在网关中通过装饰器 @UseInterceptors 使用
 * 仅处理WebSocket服务成功响应拦截，异常是不会进入该拦截器
 */
export class WsServiceResponseInterceptor implements NestInterceptor {
    intercept(context: ExecutionContext,
              next: CallHandler):
        Observable<any> | Promise<Observable<any>> {
        return next.handle().pipe(map(data => {
            // 进入该拦截器，说明没有异常，使用成功返回
            const resp: ServerResponseWrapper = {
                returnCode: SUCCESS.codeString,
                data: data
            };
            return resp;
        }))
    }
}

```

其次，与HTTP注册拦截器不同的是，nestjs中注册WebSocket的拦截器，需要在网关类上使用装饰器进行：

```diff
+ // 安装WebSocket成功响应拦截器
+ @UseInterceptors(new WsServiceResponseInterceptor())
  @WebSocketGateway(4000, {
      transports: ['websocket']
  })
  export class MyWebSocketGateway {
  ... ...
```

配置完成以后，我们重启服务，再次使用postman进行WebSocket事件请求，则会看到经过包装后的响应体：

![110-send-and-receive-success-wrapper](https://static-res.zhen.wang/images/post/2022-11-22/110-send-and-receive-success-wrapper.png)

## 异常过滤器

当然，我们尝试不发送任何的数据。理论上，则会进入校验流程不通过的场景，抛出BizException。在实际的发送中，我们会看到，postman无法接受到异常：

![120-send-and-receive-error-origin](https://static-res.zhen.wang/images/post/2022-11-22/120-send-and-receive-error-origin.png)

在服务端会看到一个异常报错：

![130-send-and-receive-error-server-output](https://static-res.zhen.wang/images/post/2022-11-22/130-send-and-receive-error-server-output.png)

对于这个问题，我们的需求是无论是否有异常，都需要使用ServerResponseWrapper进行包裹。与HTTP不同的是，WebSocket的异常过滤器需要实现`WsExceptionFilter`接口，实现该接口的catch方法：

```typescript
import {ArgumentsHost, Catch, ExceptionFilter, HttpException, WsExceptionFilter} from "@nestjs/common";
import {ServerResponseWrapper} from "../../common/server-response-wrapper";
import {BizException} from "../../common/biz-exception";

/**
 * 全局WebSocket服务的异常处理，
 * 该Filter在网关中通过 使用 @UseFilters 来进行注册
 * 仅处理WebSocket网关服务
 */
@Catch()
export class WsServiceExceptionFilter implements WsExceptionFilter {
    catch(exception: any, host: ArgumentsHost): any {
        // 进入该拦截器，说明http调用中存在异常，需要解析异常，并返回统一处理
        let responseWrapper: ServerResponseWrapper;
        if (exception instanceof BizException) {
            // 业务层Exception
            responseWrapper = {
                returnCode: exception.errorCode.codeString,
                errorMessage: exception.errorMessage
            }
        } else {
            // 其他错误
            responseWrapper = {
                returnCode: 'IM9999',
                errorMessage: 'server unknown error: ' + exception.message,
            };
        }
        // 对异常进行封装以后，需要让框架继续进行调用处理，才能正确的响应给客户端
        // 此时，需要提取到callback这个函数
        // 参考：https://stackoverflow.com/questions/61795299/nestjs-return-ack-in-exception-filter
        const callback = host.getArgByIndex(2);
        if (callback && typeof callback === 'function') {
            callback(responseWrapper);
        }
    }
}
```

这个Filter与HTTP服务中的异常过滤器差异点主要三点：

1）WebSocket中不存在HTTP状态码且不存在HTTP异常，所以我们只需要解析区分BizException与非BizException。

2）**WebSocket的异常过滤器中，想要继续后的数据处理，需要在方法返回前，从host中取到第三个参数对象（索引值为2），该值是一个回调函数，将处理后的数据作为参数，调用该callback方法，框架才能继续处理。—— WebSocket异常过滤器最终返回的关键点**。

```typescript
        // 对异常进行封装以后，需要让框架继续进行调用处理，才能正确的响应给客户端
        // 此时，需要提取到callback这个函数
        // 参考：https://stackoverflow.com/questions/61795299/nestjs-return-ack-in-exception-filter
        const callback = host.getArgByIndex(2);
        if (callback && typeof callback === 'function') {
            callback(responseWrapper);
        }
```

3）注册该异常过滤器同样和WebSocket的响应拦截器一样，需要在网关类上使用`@UseFilters`装饰器。

```diff
// 安装WebSocket成功响应拦截器
@UseInterceptors(new WsServiceResponseInterceptor())
+ // 安装WebSocket异常过滤器
+ @UseFilters(new WsServiceExceptionFilter())
@WebSocketGateway(4000, {
    transports: ['websocket']
})
```

完成该配置后，我们再次重启服务，使用postman，可以看到wrapper包装后的效果：

![140-send-and-receive-error-wrapper](https://static-res.zhen.wang/images/post/2022-11-22/140-send-and-receive-error-wrapper.png)

# 附录

本次demo已经提交至github

[w4ngzhen/nest-http-websocket-demo (github.com)](https://github.com/w4ngzhen/nest-http-websocket-demo)

同时，按照每一阶段进行了适配提交：

```
add: 添加WebSocket异常过滤器并注册到WebSocket网关中。
add: 添加WebSocket成功响应拦截器并注册到WebSocket网关中。
modify: 添加WebSocket的事件响应数据。
modify: 增减对事件”hello“的处理，并在控制台打印请求。
add: 创建一个基本的WebSocket网关以及将网关模块进行注册。
add: 增加nestjs websocket依赖、socket.io平台实现。
add: 添加HTTP服务异常过滤器，对异常进行解析并返回Wrapper包裹数据。
modify: 修改获取用户信息逻辑，加入userId判空检查。
add: 添加HTTP服务成功响应拦截器，对返回体进行统一Wrapper包裹。
modify: 注册user模块到app主模块。
add: 新增用户User模块相关的dto定义、service、controller以及module。
add: 添加ServerResponseWrapper作为服务端响应数据封装；添加返回码类，统一定义返回码；添加业务异常类，封装业务异常。
init: 初始化项目结构
```

我会逐步完善这个demo，接入各种常用的模块（数据库、Redis、S3-ECS等）。本文是本demo的初始阶段，已经发布于1.0版本tag。