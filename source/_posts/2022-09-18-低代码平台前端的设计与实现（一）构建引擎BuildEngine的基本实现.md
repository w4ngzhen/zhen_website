---
title: 低代码平台前端的设计与实现（一）构建模块的基本实现
date: 2022-09-18
tags:
 - low-code
 - build
---
这两年低代码平台的话题愈来愈火，一眼望去全是关于低代码开发的概念，鲜有关于低代码平台的设计实现。本文将以实际的代码入手，逐步介绍如何打造一款低开的平台。

<!-- more -->

低开概念我们不再赘述，但对于低开的前端来说，至少要有以下3个要素：

1. 使用能被更多用户（甚至不是开发人员）容易接受的DSL（领域特定语言），用以描述页面结构以及相关UI上下文。
2. 内部具有构建引擎，能够将DSL JSON构建为React组件，交给React进行渲染。
3. 提供设计器（Designer）支持以拖拉拽方式来快速处理DSL，方便用户快速完成页面设计。

本文我们首先着眼于如何进行构建，后面的文章我们再详细介绍设计器的实现思路。

# DSL

对于页面UI来说，我们总是可以将界面通过树状结构进行描述：

```
1. 页面
    1-1. 标题
       1-1-1. 文字
    1-2. 内容面板
       1-2-1. 一个输入框
```

如果采用xml来描述，可以是如下的形式：

```xml
<page>
    <title>标题文字</title>
    <content>
        <input></input>
    </content>
</page>
```

当然，xml作为DSL有以下的两个问题：

1. 内容存在**较大的信息冗余**（page标签、title标签，都有重复的字符）。
2. 前端需要**引入单独处理xml的库**。

自然，我们很容易想到另一个数据描述方案：JSON。使用JSON来描述上述的页面，我们可以如下设计：

```json
{
    "type": "page",
    "children": [
        {
            "type": "title",
            "props": {
                "value": "标题文字"
            }
        },
        {
            "type": "content",
            "children": [
                {
                    "type": "input"
                }
            ]
        }
    ]
}
```

初看JSON可能觉得内容比起xml更多，但是在前端我们拥有原生处理JSON的能力，这一点就很体现优势。

回顾一下JSON的方案，我们首先定义一个基本的数据结构：元素节点（`ElementNode`），它至少有如下的内容：

1. **type**属性：表明当前节点所属的类型。
2. **children**属性：一个数组，存放所有的子节点。
3. **props**：该元素的属性列表，可以应用到当前的type，产生作用。

例如，对于一个页面（`page`），该页面有一个属性配置背景色（`backgroundColor`），该页面中有一个按钮（`button`），并且该按钮有一个属性配置按钮的尺寸（`size`），此外还有一个输入框（`input`）。

```json
{
    "type": "page",
    "props": {
        "backgroundColor": "pink", // page的 backgroundColor 配置
    },
    "children": [
        {
            "type": "button",
            "props": {
                "size": "blue" // button的size配置
            }
        },
        {
            "type": "input"
        }
    ]
}
```

在我们的平台中，我们定义如下的结构：

```typescript
export interface ElementNode {
    /**
     * Element 唯一类型type
     */
    type: string;
    /**
     * 组件的各种属性：
     * 扩展的、UI的
     */
    props: {
        [propsKey: string]: any;
    };
    /**
     * Element 的所有子元素
     */
    children?: ElementNode[]
}
```

# 构建

上文定义了我们低开平台的DSL，但是DSL数据如果没有转换构建为UI组件并渲染在界面上，是没有任何意义的。我们必须要有构建引擎支持将JSON转换为web页面的内容。

## 类型构建器（TypeBuilder）

首先我们需要定义基本的构建器：`TypeBuilder`。其作用是和`ElementNode.type`相绑定，一个type对应一个builder。

```tsx
import {ReactNode} from "react";
import {ElementNode} from "../meta/ElementNode";

/**
 * 构建器构建上下文，至少包含ElementNode的相关数据
 */
export interface TypeBuilderContext {
    elementNode: Omit<ElementNode, ''>;
}

/**
 * 绑定Type的构建器
 */
export interface TypeBuilder {
    /**
     * 根据ElementNode上下文信息，得到ReactNode供React渲染
     * @param builderContext 构建器接受的数据上下文
     * @param childrenReactNode 已经完成构建的子节点的 ReactNode
     */
    build(
        builderContext: TypeBuilderContext,
        childrenReactNode?: ReactNode[],
    ): ReactNode;
}

/**
 * TypeBuilder构造函数类型
 */
export type TypeBuilderConstructor = new (args: any) => TypeBuilder;
```

这里的`TypeBuilder`只是接口抽象，具体的实现需要根据不同的type来编写不同的builder，每个builder中的build会根据有所差异。

这里我们先简单实现page、button和input：

```jsx
export class PageTypeBuilder implements TypeBuilder {

    build(builderContext: TypeBuilderContext,
          childrenReactNode?: ReactNode[]): ReactNode {
        const style: CSSProperties = {
            width: '100%',
            height: '100%',
            padding: '10px'
        }
        // 对于type = 'page'，就是用一个div作为UI组件
        // 注意，对于容器类组件，始终需要将传入的子元素放到对应的位置，控制子元素的展示
        return (
            <div style={style}>
                {childrenReactNode}
            </div>
        )
    }
}
```

```jsx
export class ButtonTypeBuilder implements TypeBuilder {
    build(builderContext: TypeBuilderContext,
          childrenReactNode?: ReactNode[]): ReactNode {
        const {elementNode} = builderContext;
        const {text = 'button'} = elementNode.props;
        // 直接使用antd的Button
        return (
            <Button
                type='primary'>
                {text}
            </Button>
        )
    }
}
```

```jsx
export class InputTypeBuilder implements TypeBuilder {
    build(builderContext: TypeBuilderContext,
          childrenReactNode?: ReactNode[]): ReactNode {
        // 使用antd的Input
        return (
            <Input/>
        )
    }
}
```

实际上，每个builder具体返回的组件，都可以根据要求进行任意定制开发，后续我们会深入介绍这一块的内容。但需要再次强调，正如上面`PageTypeBuilder`中的注释一样，**对于容器类组件，需要将`childrenReactNode`放到对应的节点位置，React才能正常渲染所有的子元素。**

实现了builder以后，为了方便管理，我们使用一个TypeBuilderManager（构建器管理器）来管理我们定义的所有的TypeBuilder：

```typescript
import {TypeBuilder, TypeBuilderConstructor} from "./TypeBuilder";
import {PageTypeBuilder} from "./impl/PageTypeBuilder";
import {ButtonTypeBuilder} from "./impl/ButtonTypeBuilder";
import {InputTypeBuilder} from "./impl/InputTypeBuilder";

/**
 * TypeBuilder管理器
 * 统一管理应用中所有已知的构建器
 * todo 后续可以支持多种方式加载
 */
class TypeBuilderManager {

    /**
     * 单实例
     * @private
     */
    private static instance: TypeBuilderManager;

    /**
     * 内存单例获取
     */
    static getInstance(): TypeBuilderManager {
        if (!TypeBuilderManager.instance) {
            TypeBuilderManager.instance = new TypeBuilderManager();
        }
        return TypeBuilderManager.instance;
    }

    /**
     * 单例，构造函数private控制
     * @private
     */
    private constructor() {
    }

    /**
     * 这里记录了目前所有的TypeBuilder映射，
     * 后续可以优化为程序进行扫描实现，不过是后话了
     * @private
     */
    private typeBuilderConstructors: Record<string, TypeBuilderConstructor> = {
        'page': PageTypeBuilder,
        'button': ButtonTypeBuilder,
        'input': InputTypeBuilder
    };

    /**
     * 根据元素类型得到对应构建器
     * @param elementType
     */
    getTypeBuilder(elementType: string): TypeBuilder {
        if (!this.typeBuilderConstructors.hasOwnProperty(elementType)) {
            throw new Error('找不到处理')
        }
        // 采用ES6的Reflect反射来处理对象创建，供后续扩展优化
        return Reflect.construct(this.typeBuilderConstructors[elementType], [])
    }

    /**
     * 添加专门处理某种elementType的TypeBuilder
     * @param elementType
     * @param typeBuilderConstructor
     */
    addTypeBuilder(elementType: string,
                   typeBuilderConstructor: TypeBuilderConstructor): void {
        if (this.typeBuilderConstructors.hasOwnProperty(elementType)) {
            console.warn(`当前TypeBuilderManager已经存在处理 elementType = ${elementType} 的Builder，本次添加对其覆盖。`);
        }
        this.typeBuilderConstructors[elementType] = typeBuilderConstructor;
    }

    /**
     * 移除处理指定elementType的Builder
     * @param elementType
     */
    removeTypeBuilder(elementType: string): void {
        delete this.typeBuilderConstructors[elementType];
    }

    /**
     * 获取当前能够处理的ElementType
     */
    getHandledElementTypes(): string[] {
        return Object.keys(this.typeBuilderConstructors);
    }
}

export {
    TypeBuilderManager
}
```

该构建器管理器维持了一个映射表，由ElementType映射到对应**TypeBuilder的构造函数**（注意，不是TypeBuilder实例，目的是为了可控懒创建TypeBuilder实例）。同时，该管理器还提供了对该映射表的增上查等API。

## 构建引擎（BuildEngine）

接下来是实现我们的构建引擎（`BuildEngine`，叫引擎高大上）。构建引擎的核心功能是读取由Schema转为的ElementNode，然后以递归深度遍历的方式不断读取ElementNode和ElementNode的子节点，根据ElementNode的类型type，从前面我们编写的TypeBuilderManager中获取对应Builder来将ElementNode构建为一个又一个ReactNode。

![010-BuildEngine-handle-flow](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-18/010-BuildEngine-handle-flow.png)

代码如下：

```ts
import {ElementNode} from "../meta/ElementNode";
import {TypeBuilderManager} from "../builder/TypeBuilderManager";
import {ReactNode} from "react"

/**
 * 构建引擎
 */
export class BuildEngine {

    /**
     * 构建：通过传入ElementNode信息，得到该节点对应供React渲染的ReactNode
     * @param rootEleNode
     */
    build(rootEleNode: ElementNode): ReactNode | undefined {
        return this.innerBuild(rootEleNode);
    }

    /**
     * 构建：通过传入ElementNode信息，得到该节点对应供React渲染的ReactNode
     * @param rootEleNode
     */
    private innerBuild(rootEleNode: ElementNode): ReactNode | undefined {
        if (!rootEleNode) {
            return undefined;
        }
        
        const {type, children} = rootEleNode;

        // 如果有子元素，则递归调用自身，获取子元素处理后的ReactNode
        const childrenReactNode =
            (children || []).map((childEleNode) => {
                return this.innerBuild(childEleNode)
            });

        // 通过 TypeBuilderManager 来统一查找对应ElementType的Builder
        const typeBuilder = TypeBuilderManager.getInstance().getTypeBuilder(type);
        if (!typeBuilder) {
            console.warn(`找不到type="${type}"的builder`)
            return undefined;
        }

        // 调用TypeBuilder的build，让其实例内部生成ReactNode
        const reactNode = typeBuilder.build(
            {
                elementNode: rootEleNode
            },
            childrenReactNode
        )
        return reactNode;
    }
}

```

需要注意，这个Engine的公共API是build，由外部调用，仅需要传入根节点ElementNode即可得到整个节点数的UI组件树（ReactNode）。为了后续我们优化内部的API结构，我们内部使用innerBuild作为内部处理的实际方法。

## 效果展示

建立一个样例项目，编写一个简单的样例：

```tsx
import {BuildEngine} from "@lite-lc/core";
import {ChangeEvent, useState} from "react";
import {Input} from 'antd';

export function SimpleExample() {

    // 使用构建引擎
    const [buildEngine] = useState(new BuildEngine());

    // 使用state存储一个schema的字符串
    const [elementNodeJson, setElementNodeJson] = useState(JSON.stringify({
        "type": "page",
        "props": {
            "backgroundColor": "pink", // page的 backgroundColor 配置
        },
        "children": [
            {
                "type": "button",
                "props": {
                    "size": "blue" // button的size配置
                },
            },
            {
                "type": "input"
            }
        ]
    }, null, 2))

    let reactNode;
    try {
        const eleNode = JSON.parse(elementNodeJson);
        reactNode = buildEngine.build(eleNode);
    } catch (e) {
        // 序列化出异常，返回JSON格式出错
        reactNode = <div>JSON格式出错</div>
    }

    return (
        <div style={{width: '100%', height: '100%', padding: '10px'}}>
            <div style={{width: '100%', height: 'calc(50%)'}}>
                <Input.TextArea
                    autoSize={{minRows: 2, maxRows: 10}}
                    value={elementNodeJson}
                    onChange={(e: ChangeEvent<HTMLTextAreaElement>) => {
                        const value = e.target.value;
                        // 编辑框发生修改，重新设置JSON
                        setElementNodeJson(value);
                    }}/>
            </div>
            <div style={{width: '100%', height: 'calc(50%)', border: '1px solid gray'}}>
                {reactNode}
            </div>
        </div>
    );
}
```

![](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-09-18/020-simple-example-show.gif)

## 设计优化

### 路径设计

目前为止，我们已经设计了一个简单的构建引擎。但是还有两个需要解决的问题：

1. 循环创建的ReactNode数组没有添加key，会导致React渲染性能问题。
2. 构建的过程中，无法定位当前ElementNode的所在位置。

我们先讨论问题2。对于该问题具体是指：**TypeBuilder.build方法接受的入参可以知道当前ElementNode节点自身的信息，但是却无法知道ElementNode所在的位置具体处于整体的哪个位置。**

```json
{
    "type": "page",
    "children": [
        {
            "type": "panel",
            "children": [
                {    
                    "type": "input"
                },
                {
                    "type": "button",
                }
            ]
        },
        {    
            "type": "input"
        }
    ]
}
```

对于上述的每一个type，都应当有其标志其唯一的一个key。可以知道，每一个元素的路径是唯一的：

- page：/page
- panel：/page/panel@0
- 第一个input：/page/panel@0/input@0。page下面有个panel（面板）元素，位于page的子节点第0号位置（基于0作为起始）。panel下面有个input元素，位于panel的子节点第0号位置。
- button：/page/panel@0/button@1
- 第二个input：/page/input@1

也就是说，路径由`'/'`拼接，每一级路径由`'@'`分割type和index，type表明该节点类型，index表明该节点处于上一级节点（也就是父级节点）的children数组的位置（基于0起始）。

那么，如何生成这样一个路径信息呢？逐级遍历ElementNode即可。其实遍历的这个动作，我们已经在之前构建引擎的innerBuild地方进行过了（递归），现在只需要进行简单的修改方法：

```diff
// BuildEngine.ts代码
-    private innerBuild(rootEleNode: ElementNode): ReactNode | undefined {
+    private innerBuild(rootEleNode: ElementNode, rootPath: string): ReactNode | undefined {
         if (!rootEleNode) {
             return undefined;
         }
// ... ...
         // 递归调用自身，获取子元素处理后的ReactNode
         const childrenReactNode =
-            (children || []).map((childEleNode) => {
-                return this.innerBuild(childEleNode)
+            (children || []).map((childEleNode, index) => {
+                // 子元素路径：
+                // 父级路径（也就是当前path）+ '/' + 子元素类型 + 子元素所在索引
+                const childPath = `${rootPath}/${childEleNode.type}@${index}`;
+                return this.innerBuild(childEleNode, childPath);
             });
// ... ...
```

首先，我们修改了innerBuild方法入参，增加了参数`rootPath`，用以表示当前节点所在的路径；其次，在生成子元素ReactNode的地方，将`rootPath`作为基准，根据上述规则`"${elementType}@${index}"`，来生成子元素节点的路径，并传入到的递归调用的innerBuild中。

当然，build内部调用innerBuild的时候，需要构造一个起始节点的path，传入innerBuild。

```diff
// BuildEngine.ts代码
     build(rootEleNode: ElementNode): JSX.Element | undefined {
-        return this.innerBuild(rootEleNode);
+        // 起始节点，需要构造一个起始path传入innerBuild
+        // 注意，根节点由于不属于某一个父级的子元素，所以不存在'@${index}'
+        return this.innerBuild(rootEleNode, '/' + rootEleNode.type);
     }
```

另外，为了让每一个builder能够获取到需要构建的ElementNode的路径信息这个上下文，我们在TypeBuilderContext中添加path属性：

```diff
/**
 * 构建器构建上下文，至少包含ElementNode的相关数据
 */
export interface TypeBuilderContext {
+   /**
+    * path：让每个TypeBuilder知道当前构建的节点所在的路径
+    */
+    path: string;
    elementNode: Omit<ElementNode, ''>;
}
```

同时，innerBuild中也要进行一定的修改，需要在调用`TypeBuilder.build`方法的时候把path传入：

```diff
        // innerBuild函数
        // ...
         const reactNode = typeBuilder.build(
             {
+                path: rootPath,
                 elementNode: rootEleNode
             },
        // ...
```

这样一来，每个builder的build方法里面，都可以从BuilderContext中获取到当前实际要构建转换的ElementNode唯一具体路径path。在后续的优化中，我们就可以利用该path做一些事情了。

现在，如何处理**问题1：key值未填写**的问题呢？其实，当我们解决了问题2以后，我们现在知道path是唯一的，那么我们可以将path作为每个元素的key，例如：

Button构建器：

```diff
export class ButtonTypeBuilder implements TypeBuilder {
    build(builderContext: TypeBuilderContext,
          childrenReactNode?: ReactNode[]): ReactNode {
-        const {elementNode} = builderContext;
+        const {path, elementNode} = builderContext;
         const {text = 'button'} = elementNode.props;
         // 直接使用antd的Button
         return (
             <Button
+                key={path}
                 type='primary'>
                 {text}
             </Button>)
     }
}
```

Input构建器：

```diff
export class InputTypeBuilder implements TypeBuilder {
    build(builderContext: TypeBuilderContext,
          childrenReactNode?: ReactNode[]): ReactNode {
+        const {path} = builderContext;
         // 使用antd的Input
         return (
-            <Input/>
+            <Input key={path} />
         )
     }
 }
```

page构建器可以不用，因为它只会生成一个组件，不会参与循环生成中。

**我们只需要将所有的组件使用path作为key即可**。

# 关于构建的总结

目前为止，我们设计了一套十分精简的根据Schema节点转换为ReactNode的构建引擎，以一套基于antd组件的组件构建引擎，通过接收JSON遍历节点构建出ReactNode，再交给React渲染出对应结构的页面。该构建引擎需要考虑，React渲染时候元素的上下文，所以在遍历元素节点的时候，需要把相关的上下文进行封装并交给对应的构建Builder作为key使用。当然，渲染部分还有很多很多的处理以及各种基本UI元素的创建还有很多的方法（譬如CDN挂载基础类型等），但是基于本系列，我们由浅入深逐步建立整个低代码平台。下篇文章，笔者将开始介绍设计器Designer的实现。

# 附录

本章内容对应代码已经推送到github上

[w4ngzhen/lite-lc (github.com)](https://github.com/w4ngzhen/lite-lc)

main分支与最新文章同步，chapter_XX对应本系列的第几章，本文在分支chapter_01上体现。

且按照文章里各段介绍顺序完成了提交：

```
modify: BuildEngine递归增加path；TypeBuilderContext添加path。
add: 添加样例文件，用以展示效果。
add: 添加一个基础的构建引擎实现；导出核心库提供的一些类型与对象。
add: 添加TypeBuilder管理器用以管理所有已知的TypeBuilder.
add: 添加TypeBuilder以及其实现.
add: ElementNode 映射 JSON schema
init: core and example basic files.
```





