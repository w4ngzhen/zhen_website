---
title: 低代码平台前端的设计与实现（三）设计态画布DesignCanvas的设计与实现
date: 2023-02-04
tags: 
 - lowcode
---

上一篇文章，我们分析并设计了关于构建引擎BuildEngine的切面设计。本文我们将基于BuildEngine所提供的切面处理能力，在ComponentBuildAspectHandler中通过一些逻辑，来完成一个轻量级的设计器画布。

这个画布能够实现如下的一个简单的效果。对于所有渲染出来的元素，都会有一个灰色的边框，当我们选中某个元素的时候，就会高亮显示。

![010-wrapper-show](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-02-04/010-wrapper-show.gif)

<!-- more -->

# ElementNodeDesignWrapper

要做到上述效果，对于通过ElementNode创建出来的组件，我们可以使用一个元素来进行包裹，我们暂时对这个组件取名为`ElementNodeDesignWrapper`，它的作用就是能够给每一个元素添加边框。

这个wrapper组件，我们至少会设计以下几个属性：

- nodePath：一个基本信息，作为外部传入；
- isSelected：决定该wrapper是否被选中；
- onClick：wrapper组件被点击时候，触发的onClick事件；

有了`isSelected`和`onClick`以后，我们就可以让上层代码来控制多个元素究竟是哪个元素需要高亮。

```typescript
export type ElementNodeDesignWrapperProps = {
    /**
     * 标识当前节点path
     */
    nodePath: string;
    /**
     * 是否被选中
     */
    isSelected?: boolean;
    /**
     * 点击事件
     */
    onClick?: () => void;
}
```

对于这个wrapper，我们考虑使用div元素来包裹子元素，也就是说，wrapper的本质是div。这个div元素我们通过isSelected（是否选中）来控制其CSS中的`outline`样式配置。之所以选择`outline`，是因为outline在显示的时候，是不会影响元素的位置大小的，但缺点则是无论其元素是什么外形，`outline`总是矩形。

![020-wrapper-detail](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-02-04/020-wrapper-detail.png)

其次，我们还需要考虑这样一种问题，如果wrapper div包裹的实际HTML是`<button>`、`<a>`、`<span>`、`<b>`以及`<i>`元素，如果我们不将这个作为wrapper div的display设置为`inline-block`，那么wrapper div则会变成宽度占据一行的元素，会变成如下效果：

![030-outline-err-display](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-02-04/030-outline-err-display.png)

我们需要做的就是，检测wrapper div内部的元素是button、a、span、b或i元素的时候，则将wrapper div的样式中display属性置为`inline-block`，这样wrapper div就可以贴合这些元素。

那么，如何检测呢？我们可以采用这样一种方式：通过useRef这个Hook来创建一个ref，交给我们的wrapper div；然后，在useEffect的回调中，拿到类型为HTMLDivElement的ref.current。这个current我们可以通过访问firstChild就是div的唯一一个子元素，也就是wrapper包裹的元素。并且，我们可以访问firstChild.nodeName就能知道wrapper的HTML元素名称。存放到一个名为`targetNodeHtmlType`的state中；最后，我们按照上面的需求，让wrapper div的样式中的display属性，根据`targetNodeHtmlType`是否属于button、a、span、b或i元素中的一种来决定是否是`inline-block`。

![040-wrapper-html-ref-detail](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-02-04/040-wrapper-html-ref-detail.png)

最后，我们还需要对wrapper div的onClick事件进行“代理”，并阻止冒泡。

综合以上的分析，我们Wrapper div最终的样式核心代码：

```typescript
export const ElementNodeDesignWrapper: FC<PropsWithChildren<ElementNodeDesignWrapperProps>> = (props) => {

    const {
        nodePath,
        isSelected = false,
        children,
        onClick = () => {
        }
    } = props;

    const ref = useRef<HTMLDivElement | null>(null);

    const [
        targetNodeHtmlType,
        setTargetNodeHtmlType
    ] = useState<string>();

    useEffect(() => {
        if (!ref || !ref.current) {
            return;
        }
        const currentEle: HTMLDivElement = ref.current;
        const eleNodeName = currentEle.firstChild.nodeName;
        setTargetNodeHtmlType(eleNodeName);
    });

    const style: CSSProperties = useMemo(() => {
        // Wrapper内部以下实际的HTML元素在展示的过程中，需要使用inline-block
        // 否则会显示异常
        const inlineBlockEle = ['A', 'SPAN', 'BUTTON', 'B', 'I'];
        return {
            boxSizing: 'border-box',
            // 元素被选中，则使用蓝色高亮边框，否则使用灰色虚线
            outline: isSelected ? '2px solid blue' : '1px dashed gray',
            display: inlineBlockEle.includes(targetNodeHtmlType) ? 'inline-block' : '',
            padding: '3px',
            margin: '3px'
        }
    }, [isSelected, targetNodeHtmlType]);

    return (
        <div key={nodePath + '_wrapper_key'}
             style={style}
             ref={ref}
             onClick={(event) => {
                 event.stopPropagation();
                 onClick();
             }}>
            {children}
        </div>
    )
}
```

# DesignCanvas

接下来，我们开始设计一个名为DesignCanvas的设计态画布，这个画布我们先暂时先不考虑比较复杂的功能，先考虑如何结合上面的Wrapper组件进行基本的效果呈现。考虑到对外屏蔽DesignCanvas的细节，我们只暴露一个属性，就是传入 JSON schema：

```typescript
interface DesignCanvasProps {
    /**
     * Schema JSON字符串
     */
    rootNodeSchemaJson: string;
}

export const DesignCanvas = (props: DesignCanvasProps) => {
    const {
        rootNodeSchemaJson
    } = props;

    // 1. 存储单机选中的path的state
    const [selectedNodePath, setSelectedNodePath] = useState<string>('');

    // 2. 经过切面绑定的buildEngine
    const buildEngine = ... ...

    // 3. 经过buildEngine + schema 创建的React组件（已经考虑的基本的异常处理）
    const renderComponent = ... ...

    return (
        <div style={{
            width: '100%',
            height: '100%',
            padding: '5px'
        }}>
            {renderComponent}
        </div>
    )
}
```

（1）selectedNodePath用以存储当前选中的path。在后续的切面处理中，构建元素节点的时候，如果切面正在处理的节点path与selectedNodePath一致，则wrapper组件需要高亮，否则虚线。

（2）buildEngine的代码具体如下：

```typescript
		// 经过切面绑定的buildEngine
    const buildEngine = useMemo(() => {
        const engine = new BuildEngine();
        engine.componentBuildAspectHandler = (reactNode, ctx) => {
            const {path} = ctx;

            const wrapperProps: ElementNodeDesignWrapperProps = {
                nodePath: path,
                isSelected: path === selectedNodePath,
                onClick: () => {
                    console.debug('wrapper onClick')
                    setSelectedNodePath(path)
                }
            }

            return (
                <ElementNodeDesignWrapper {...wrapperProps}>
                    {reactNode}
                </ElementNodeDesignWrapper>
            )
        }
        return engine;
    }, [selectedNodePath]);

```

上面的buildEngine中的componentBuildAspectHandler切面处理，我们编写了我们自己的实现，原本默认返回的组件，我们使用ElementNodeDesignWrapper进行包裹返回。其中：

1. isSelected属性来自于当前正处理节点path与第1点DesignCanvas组件存储的path的比对，如果当前正在处理及的几点就是已经选中的节点path，那么这个wrapper组件则被“选中”。

2. onClick属性的实现代码则是当wrapper组件点击后，更新selectedNodePath。

（3）renderComponent的实现：

```typescript
  // 经过buildEngine + schema 创建的React组件（已经考虑的基本的异常处理）
    const renderComponent = useMemo(() => {
        try {
            const eleNode = JSON.parse(rootNodeSchemaJson);
            return buildEngine.build(eleNode);
        } catch (e) {
            return <div>构建出错：{e.message}</div>
        }
    }, [rootNodeSchemaJson, selectedNodePath]);
```

对于这个渲染React组件，主要是将schema解析为ElementNode结构，并交给构建引擎build；如果报错则返回一个异常组件。

# 样例

在编写样例之前，我们先导出DesignCanvas，然后适当修改样例代码：

```tsx
import {ChangeEvent, useState} from "react";
import {Input} from 'antd';
import {DesignCanvas} from "@lite-lc/core";

export function SimpleExample() {

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
                <DesignCanvas rootNodeSchemaJson={elementNodeJson}/>
            </div>
        </div>
    );
}
```

效果：

![010-wrapper-show](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-02-04/010-wrapper-show.gif)

# 附录

本次相关代码已经提交至github，对应分支为chapter_03：

[w4ngzhen/lite-lc at chapter_03 (github.com)](https://github.com/w4ngzhen/lite-lc/tree/chapter_03)
