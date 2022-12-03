---
title: 低代码平台前端的设计与实现（二）构建引擎BuildEngine切面处理设计
date: 2022-12-03
tags:
 - lowcode
---

上一篇文章，我们介绍了如何设计并实现一个轻量级的根据JSON的渲染引擎，通过快速配置一份规范的JSON文本内容，就可以利用该JSON生成一个基础的UI界面。本文我们将回到低开的核心—页面拖拉拽，探讨关于页面拖拉拽的核心设计器Designer的一些**基本前置需求**，也就是构建引擎BuildEngine切面处理设计。

<!-- more -->

只要接触过低开平台的朋友都见过这样的场景，在设计器的画布中点击已经拖拉拽好的UI元素，会有一个边框，高亮显示当前的元素，还支持操作：

![010-border-show](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-12-03/010-border-show.png)

在上一篇文章我们介绍了创建的整个流程：由一个构建引擎（BuildEngine）通过读取JSON Schema的节点来匹配对应的节点类型来生成UI元素。

为了实现设计器画布选中边框的需求，首先想到的一个解决方案就是仿照BuildEngine做一个类似的DesignerBuildEngine，里面的流程和BuildEngine大致相同，只是在生成最终的ReactNode节点的时候，在其外围使用某个元素进行包裹，具备边框等功能：

```tsx
// DesignerBuileEngine伪代码
class DesignerBuileEngine {
   innerBuild() {
     // 在返回某个ReactNode前，使用一个div包裹
     const reactNode = xxx;
     return createElement(
       'div', { 
       // 边框样式等数据 
       }, 
       reactNode);
     }
}
```

但是这并不是一个很优雅的设计，因为如果我们衍生出一个新的DesignerRenderEngine，那么我们需要同时维护一个设计态一个云形态两个Engine，尽管他们的处理流程大致相同。

# 切面设计

## 组件构建切面处理

为了避免功能代码的冗余，也更方便后续的扩展性。我们考虑采用切面的设计方案。将整个处理流程的某些环节加入切面，以达到灵活处理的目的。切面的实现可以有很多种形式，例如一个回调函数，又或者传入一个对象实例（本质上还是回调）。作为一个轻量级低开模块，我们暂时设计一个简单的函数ComponentBuildAspectHandler（组件构建切面处理器）来进行切面处理。

![020-BuildEngine-handle-flow-with-build-aspect](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-12-03/020-BuildEngine-handle-flow-with-build-aspect.png)

该切面方法作为BuildEngine的一个实例属性存在，在BuildEngine进行构建的时候，我们切入进行处理：

```diff
// 伪代码
class BuildEngin {
// 新增的属性
+   componentBuildAspectHandle: ComponentBuildAspectHandler;

    innerBuild() {
        const reactNode = build()
        
+       // 在返回生成的React UI节点之前，调用切面处理器进行处理
+		    // const finalReactNode = ComponentBuildAspectHandler(reactNode, context);
        
        return reactNode;
    }
}
```

正如上图的黄色部分的，我们首先编写一个类型ComponentBuildAspectHandler，就是一个函数类型的处理方法。同时，为了封装一些处理的上下文，我们额外定义一个ComponentBuildAspectHandleContext来承载上下文数据。

```typescript
export interface ComponentBuildAspectHandleContext {
    /**
     * 当前构建的节点的path
     */
    path: string;
    /**
     * 当前构建的元素节点数据
     */
    elementNode: Omit<ElementNode, ''>;
}
/**
 * 构建切面
 * @param reactNode 通过typeBuilder构建出的reactNode
 * @param handleContext 封装的一些支持切面处理的上下文
 */
export type ComponentBuildAspectHandler =
    (reactNode: ReactNode,
     handleContext: ComponentBuildAspectHandleContext) => ReactNode
```

然后，我们为构建引擎添加一个实例字段并支持外部配置该handler：

```diff
 /**
  * 构建引擎
  */
 export class BuildEngine {

+    /**
+     * 引擎所持有的“组件构建切面处理器”
+     * @private
+     */
+    private _componentBuildAspectHandler?: ComponentBuildAspectHandler;
+
+    set componentBuildAspectHandler(value: ComponentBuildAspectHandler | undefined) {
+        this._componentBuildAspectHandler = value;
+    }

    ... ...
    ... ...
    
    private innerBuild(rootEleNode: ElementNode, rootPath: string): ReactNode | undefined  {

    ... ...
+
+        if (this._componentBuildAspectHandler) {
+            // BuildEngine使用者可以定义ReactNode切面处理，实现定制化
+            console.debug('进入组件构建切面处理')
+            return this._componentBuildAspectHandler(reactNode, {
+                path: rootPath,
+                elementNode: rootEleNode
+            })
+        }
+
         return reactNode;
     }
 }

```

如此，我们将构建引擎的中对于ReactNode节点的处理通过切面的方式，交给了外部调用者，方便进行灵活的定制开发。

回顾整个构建的流程，在运行时模式下（RuntimeMode），都是按照JSON Schema 节点（ElementNode）通过各种类型build处到一个又一个的ReactNode这个过程进行的；而当处于设计态（DesginMode）的时候，就可以通过`ComponentBuildAspectHandler`来进行一定的包裹，进而产生出设计态的效果。

## 元素节点解析切面处理

同样的，当进入到innerBuild的时候，我们就会解析JSON Schema节点，考虑到可能会存在希望在讲解析处的Schema节点交给TypeBuilder进行处理前，能够对节点进行一些编程开发，我们在此基础上再做一个扩展，设计暴露对该ElementNode节点处理的切面（ElementNodeResolveAspectHandler）。这样一来，就可以为后续可能存在的对于ElementNode需要进行特殊处理的场景进行支持。

同样的，我们暂时设计一个简单的函数ElementNodeResolveAspectHandler（组件构建切面处理器）来进行切面处理。

![030-BuildEngine-handle-flow-with-node-resolve-aspect.png](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-12-03/030-BuildEngine-handle-flow-with-node-resolve-aspect.png.png)

正如上图中的读取ElementNode切面处理部分，我们首先编写一个类型ElementNodeResolveAspectHandler，就是一个函数类型的处理方法。同时，为了封装一些处理的上下文，我们额外定义一个ElementNodeResolveAspectHandleContext来承载上下文数据。

```typescript
export interface ElementNodeResolveAspectHandleContext {
    /**
     * 当前构建的节点的path
     */
    path: string;
}

/**
 * 元素节点解析切面处理
 */
export type ElementNodeResolveAspectHandler =
    (elementNode: ElementNode, context: ElementNodeResolveAspectHandleContext) => ElementNode | undefined;
```

该方法也同样作为BuildEngine的一个实例属性，BuildEngine的持有者配置该handler：

```diff
export class BuildEngine {
+    /**
+    * 引擎所持有的"元素节点解析切面处理"
+    * @private
+    */
+   private _elementNodeResolveAspectHandler?: ElementNodeResolveAspectHandler;
+
+   set elementNodeResolveAspectHandler(value: ElementNodeResolveAspectHandler) {
+       this._elementNodeResolveAspectHandler = value;
+   }
    private innerBuild(rootEleNode: ElementNode, rootPath: string): ReactNode {
    
+       let resolvedRootEleNode: ElementNode;
+       if (this._elementNodeResolveAspectHandler) {
+           // BuildEngine使用者可以定义元素节点解析切面处理，实现定制化
+           console.debug('进入元素节点解析切面处理');
+           resolvedRootEleNode =
+               this._elementNodeResolveAspectHandler(rootEleNode, {
+                   path: rootPath
+               })
+       } else {
+           resolvedRootEleNode = rootEleNode;
+       }
    
        // innerBuild余下内容 ... ...
        // 需要注意的是，后续操作的ElementNode都需要使用resolvedRootEleNode
    
    }

}
```

至此，我们针对构建引擎BuildEngine设计了两个关键点的切面处理，为后续构建引擎支撑开发设计态提供了技术上的可能性。接下来，我们将屏蔽BuildEngine实例，不再直接暴露给用户进行使用，而是分别封装一个RuntimeBuildEngine以及DesignBuildEngine，在其内部会构造我们目前设计出的BuildEngine，并通过切面定制节点解析以及创建ReactNode的过程，来满足两种类型的差异。

对于本文来说，我们先编写一个RuntimeBuildEngine，用以展示切面的效果：

```typescript
import {BuildEngine} from "./BuildEngine";
import {ElementNode} from "../meta/ElementNode";

/**
 * 运行时BuildEngine
 */
export class RuntimeBuildEngine {
    private readonly _buildEngine: BuildEngine;

    constructor() {
        this._buildEngine = new BuildEngine();
        this._buildEngine.elementNodeResolveAspectHandler =
            (eleNode, ctx) => {
                console.debug(`[elementNodeResolveAspectHandler] current elementNode: ${eleNode.type}, path: ${ctx.path}`)
                // 务必返回节点
                return eleNode
            }
        this._buildEngine.componentBuildAspectHandler =
            (reactNode, ctx) => {
                console.debug(`[componentBuildAspectHandler] current reactNode: `, reactNode);
                // 务必返回节点
                return reactNode;
            }
    }

    /**
     * 内部代理BuildEngine.build
     * @param rootEleNode
     */
    build(rootEleNode: ElementNode) {
        return this._buildEngine.build(rootEleNode);
    }
}
```

注意我们当前RuntimeBuildEngine，对于切面处理的部分，我们只是通过console打印了信息到控制台。

最后，我们修改core模块的导出内容，不再到处BuildEngine，而是导出RuntimeBuildEngine供测试代码使用。

```diff
// core/src/index.ts
export {
-    BuildEngine
-} from './engine/BuildEngine';
+    RuntimeBuildEngine
+} from './engine/RuntimeBuildEngine';
```

# 基本测试

完成上述的切面处理代码以后，我们回到example项目中，重新使用RuntimeBuildEngine，通过打印的方式来验证切面的可行性。

![040-runtime-build-engine-aspect-output](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-12-03/040-runtime-build-engine-aspect-output.png)

# 附录

本文的所有内容已经提交至github仓库

[w4ngzhen/lite-lc (github.com)](https://github.com/w4ngzhen/lite-lc)

本章对应分支chapter_02
