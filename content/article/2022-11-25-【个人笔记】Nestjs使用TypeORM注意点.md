---
title: 【个人笔记】Nestjs使用TypeORM注意点
date: 2022-11-25
tags:
 - nestjs
 - typeorm
 - note
categories:
  - 技术
---

在Nestjs使用TypeORM还是有一些注意点。

<!-- more -->

# entities路径配置注意点

在nestjs中使用TypeORM，需要配置数据库连接（以MySQL为例）。需要特别注意的是配置参数里面的entities字段：

```json
{
    "type": "mysql",
    "host": "localhost",
    "port": 3306,
    "username": "root",
    "password": "root",
    "database": "zen-im",
    "entities": ["**/*.entity.{ts,js}"],
    "synchronize": true
}
```

entities字段的作用是根据提供的路径字符串，**在运行的时候**查找对应路径下的entity文件。

首先，我建议最好直接在使用 *TypeORM.forRoot* 来引入配置，就像下面一样：

```typescript
// app.module.ts
const entitiesPaths = [join(__dirname, '..', '**', '*.entity.{ts,js}')]
@Module({
    imports: [
        TypeOrmModule.forRoot({
                "type": "mysql",
                "host": "localhost",
                "port": 3306,
                "username": "root",
                "password": "root",
                "database": "zen-im",
                "entities": entitiesPaths,
                "synchronize": true
            }
        )],
    controllers: [AppController],
    providers: [AppService],
})
export class AppModule {
}
```

这样做的原因在于，能够控制entities的搜索路径。在上面例子中，我控制的路径是当前运行js路径（`__dirname`）的上一层（`..`）目录中的任意（`**`）子目录中，搜索所有的以`.entity.js`或`.entity.ts`作为后缀的文件作为扫描为entity文件。

之所以使用了上一层（`..`），是因为我的项目中，上面这个`app.module.ts`放在了`src/module`目录下，而我的所有entity.ts都在放在`src/entity`这个目录下：

```
src
 - module
   - app.module.ts
 - entity
   - user
     - user.entity.ts
```

最终生成出来的js代码，会放在`项目根目录/dist目录`下：

```
dist
 - module
   - app.module.js
 - entity
   - user
     - user.entity.js
```

所以在实际运行中，app.module.js中配置entities这个字段的时候，需要返回上一层（`..`），才能查找到。如果你的项目中，app.module.ts就在src目录下，entity存放路径就在app.module.ts所在的子目录，就可以直接配置成：

```js
join(__dirname, '**', "*.entity.{js,ts}")
```

如果这个路径配置不一致，运行的时候，会出现以下的错误：

- EntityMetadataNotFoundError: No metadata for "你的Entity" was found.

# Entity列配置注意点

这个地方比较细节，笔者编写代码的时候，按照曾经Java的MyBatis-Plus注解使用，给字段添加列定义的时候。不小心直接把名称字符串作为参数：

```typescript
import {Column, Entity, PrimaryColumn} from "typeorm";

@Entity('user')
export class UserPo {
    /**
     * 全局唯一ID
     */
    @PrimaryColumn('uid')
    uid: string;
    /**
     * 用户名
     */
    @Column('name')
    name: string;
}
```

运行的时候，就出现了：

- DataTypeNotSupportedError: Data type "uid" in "UserPo.uid" is not supported by "mysql" database.

原因在于装饰器`@PrimaryColumn`或者`@Column`的参数如果是一个字符串，则视为一个数据库的类型！要传一个对象，这个对象有个name字段，来表示列名：

```diff
import {Column, Entity, PrimaryColumn} from "typeorm";

@Entity('user')
export class UserPo {
    /**
     * 没有业务逻辑含义的全局唯一ID
     */
-    @PrimaryColumn('uid')
+    @PrimaryColumn({
+        name: 'uid'
+    })
    uid: string;
    /**
     * 用户名
     */
-    @Column('name')
+    @Column({
+       name: 'name'
+   })
    name: string;
}

```

