# San 为什么会这么快

> 一个 MVVM 框架的性能进化之路


性能一直是 [框架选型](https://medium.freecodecamp.org/the-12-things-you-need-to-consider-when-evaluating-any-new-javascript-library-3908c4ed3f49) 最重要的考虑因素之一。[San](https://baidu.github.io/san/) 从设计之初就希望不要因为自身的短板（性能、体积、兼容性等）而成为开发者为难的理由，所以我们在性能上投入了很多的关注和精力，效果至少在从现在看来，还不错。


![San non-keyed performance](img/san-perf-non-keyed.png)



将近 2 年以前，我发了一篇 [San - 一个传统的MVVM组件框架](https://efe.baidu.com/blog/san-a-traditional-mvvm-component-framework/)。对 [San](https://baidu.github.io/san/) 设计初衷感兴趣的同学可以翻翻。我一直觉得框架选型的时候，了解它的调性是非常关键的一点。


不过其实，大多数应用场景的框架选型中，**知名度** 是最主要的考虑因素，因为 **知名度** 意味着你可以找到更多的人探讨、可以找到更多周边、可以更容易招聘熟手或者以后自己找工作更有优势。所以本文的目的并不是将你从三大阵营（[React](https://reactjs.org/)、[Vue](https://vuejs.org/)、[Angular](https://angular.io/)）拉出来，而是想把 [San](https://github.com/baidu/san/) 的性能经验分享给你。这些经验无论在应用开发，还是写一些基础的东西，应该会有所帮助。


在正式开始之前，惯性先厚脸皮求下 [Star](https://github.com/baidu/san/)。


## 视图创建


考虑下面这个还算简单的组件：

```js
const MyApp = san.defineComponent({
    template: `
        <div>
            <h3>{{title}}</h3>
            <ul>
                <li s-for="item,i in list">{{item}} <a on-click="removeItem(i)">x</a></li>
            </ul>
            <h4>Operation</h4>
            <div>
                Name:
                <input type="text" value="{=value=}">
                <button on-click="addItem">add</button>
            </div>
            <div>
                <button on-click="reset">reset</button>
            </div>
        </div>
    `,

    initData() {
        return {
            title: 'List',
            list: []
        };
    },

    addItem() {
        this.data.push('list', this.data.get('value'));
        this.data.set('value', '');
    },

    removeItem(index) {
        this.data.removeAt('list', index);
    },

    reset() {
        this.data.set('list', []);
    }
});
```

在视图初次渲染完成后，[San](https://github.com/baidu/san/) 会生成一棵这样子的树：

![Render Tree](img/render-tree.png)


那么，在这个过程里，[San](https://github.com/baidu/san/) 都做了哪些事情呢？


### 模板解析

在组件第一个实例被创建时，**template** 属性会被解析成 [ANode](https://github.com/baidu/san/blob/master/doc/anode.md)。

![ANode](img/anode.png)

[ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 的含义是抽象节点树，包含了模板声明的所有信息，包括标签、文本、插值、数据绑定、条件、循环、事件等信息。对每个数据引用的声明，也会解析出具体的表达式对象。


```json
{
    "directives": {},
    "props": [],
    "events": [],
    "children": [
        {
            "directives": {
                "for": {
                    "item": "item",
                    "value": {
                        "type": 4,
                        "paths": [
                            {
                                "type": 1,
                                "value": "list"
                            }
                        ]
                    },
                    "index": "i",
                    "raw": "item,i in list"
                }
            },
            "props": [],
            "events": [],
            "children": [
                {
                    "textExpr": {
                        "type": 7,
                        "segs": [
                            {
                                "type": 5,
                                "expr": {
                                    "type": 4,
                                    "paths": [
                                        {
                                            "type": 1,
                                            "value": "item"
                                        }
                                    ]
                                },
                                "filters": [],
                                "raw": "item"
                            }
                        ]
                    }
                },
                {
                    "directives": {},
                    "props": [],
                    "events": [
                        {
                            "name": "click",
                            "modifier": {},
                            "expr": {
                                "type": 6,
                                "name": {
                                    "type": 4,
                                    "paths": [
                                        {
                                            "type": 1,
                                            "value": "removeItem"
                                        }
                                    ]
                                },
                                "args": [
                                    {
                                        "type": 4,
                                        "paths": [
                                            {
                                                "type": 1,
                                                "value": "i"
                                            }
                                        ]
                                    }
                                ],
                                "raw": "removeItem(i)"
                            }
                        }
                    ],
                    "children": [
                        {
                            "textExpr": {
                                "type": 7,
                                "segs": [
                                    {
                                        "type": 1,
                                        "literal": "x",
                                        "value": "x"
                                    }
                                ],
                                "value": "x"
                            }
                        }
                    ],
                    "tagName": "a"
                }
            ],
            "tagName": "li"
        }
    ],
    "tagName": "ul"
}
```

[ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 保存着视图声明的数据引用与事件绑定信息，在视图的初次渲染与后续的视图更新中，都扮演着不可或缺的作用。

无论一个组件被创建了多少个实例，**template** 的解析都只会进行一次。当然，预编译是可以做的。但因为 **template** 是用才解析，没有被使用的组件不会解析，所以就看实际使用中值不值，有没有必要了。


### preheat

在组件第一个实例被创建时，[ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 会进行一个 **预热** 操作。看起来， **预热** 和 **template解析** 都是发生在第一个实例创建时，那他们有什么区别呢？

1. **template解析** 生成的 [ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 是一个可以被 JSON stringify 的对象。
2. 由于 1，所以 [ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 可以进行预编译。这种情况下，**template解析** 过程会被省略。而 **预热** 是必然会发生的。

接下来，让我们看看预热到底生成了什么？

```js
aNode.hotspot = {
    data: {},
    dynamicProps: [],
    xProps: [],
    props: {},
    sourceNode: sourceNode
};
```

上面这个来自 [preheat-a-node.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js) 的简单代码节选不包含细节，但是可以看出， **预热** 过程生成了一个 `hotspot` 对象，其包含这样的一些属性：

- data - 节点数据引用的摘要信息
- dynamicProps - 节点上的动态属性
- xProps - 节点上的双向绑定属性
- props - 节点的属性索引
- sourceNode - 用于节点生成的 HTMLElement



**预热** 的主要目的非常简单，就是把在模板信息中就能确定的事情提前，只做一遍，避免在 **渲染/更新** 过程中重复去做，从而节省时间。**预热** 过程更多的细节见 [preheat-a-node.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js)。在接下来的部分，对 `hotspot` 发挥作用的地方也会进行详细说明。

### 视图创建过程

![Render](img/anode-render.png)

视图创建是个很常规的过程：基于初始的 **数据** 和 [ANode](https://github.com/baidu/san/blob/master/doc/anode.md)，创建一棵对象树，树中的每个节点负责自身在 DOM 树上节点的操作（创建、更新、删除）行为。对一个组件框架来说，创建对象树的操作无法省略，所以这个过程一定比原始地 createElement + appendChild 慢。

因为这个过程比较常规，所以接下来不会描述整个过程，而是提一些有价值的优化点。


#### cloneNode

在 **预热** 阶段，我们根据 `tagName` 创建了 `sourceNode`。

```js
if (isBrowser && aNode.tagName
    && !/^(template|slot|select|input|option|button)$/i.test(aNode.tagName)
) {
    sourceNode = createEl(aNode.tagName);
}
```

[ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 中包含了所有的属性声明，我们知道哪些属性是动态的，哪些属性是静态的。对于静态属性，我们可以在 **预热** 阶段就直接设置好。See [preheat-a-node.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js#L117-L137)


```js
each(aNode.props, function (prop, index) {
    aNode.hotspot.props[prop.name] = index;
    prop.handler = getPropHandler(aNode.tagName, prop.name);

    // ......
    if (prop.expr.value != null) {
        if (sourceNode) {
            prop.handler(sourceNode, prop.expr.value, prop.name, aNode);
        }
    }
    else {
        if (prop.x) {
            aNode.hotspot.xProps.push(prop);
        }
        aNode.hotspot.dynamicProps.push(prop);
    }
});
```


在 **视图创建过程** 中，就可以从 `sourceNode` clone，并且只对动态属性进行设置。See [element-own-create.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/element-own-create.js#L28-L62)


```js
var sourceNode = this.aNode.hotspot.sourceNode;
var props = this.aNode.props;

if (sourceNode) {
    this.el = sourceNode.cloneNode(false);
    props = this.aNode.hotspot.dynamicProps;
}
else {
    this.el = createEl(this.tagName);
}

// ...

for (var i = 0, l = props.length; i < l; i++) {
    var prop = props[i];
    var propName = prop.name;
    var value = isComponent
        ? evalExpr(prop.expr, this.data, this)
        : evalExpr(prop.expr, this.scope, this.owner);

    // ...

    prop.handler(this.el, value, propName, this, prop);
    
    // ...
}
```

#### 属性操作

不同属性对应 DOM 的操作方式是不同的，属性的 **预热** 提前保存了属性操作函数（[preheat-a-node.jsL119](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js#L119)），属性初始化或更新时就无需每次都重复获取。

```js
prop.handler = getPropHandler(aNode.tagName, prop.name);
```

对于 `s-bind`，对应的数据是 **预热** 阶段无法预知的，所以属性操作函数只能在具体操作时决定。See [element-own-create.jsL42](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/element-own-create.js#L42)

```js
for (var key in this._sbindData) {
    if (this._sbindData.hasOwnProperty(key)) {
        getPropHandler(this.tagName, key)( // 看这里看这里
            this.el,
            this._sbindData[key],
            key,
            this
        );
    }
}
```

所以，`getPropHandler` 函数的实现也进行了相应的结果缓存。See [get-prop-handler.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/get-prop-handler.js#L247-L258)

```js
var tagPropHandlers = elementPropHandlers[tagName];
if (!tagPropHandlers) {
    tagPropHandlers = elementPropHandlers[tagName] = {};
}

var propHandler = tagPropHandlers[attrName];
if (!propHandler) {
    propHandler = defaultElementPropHandlers[attrName] || defaultElementPropHandler;
    tagPropHandlers[attrName] = propHandler;
}

return propHandler;
```

#### 创建节点

视图创建过程中，[San](https://github.com/baidu/san/) 通过 `createNode` 工厂方法，根据 [ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 上每个节点的信息，创建组件的每个节点。

[ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 上与节点创建相关的信息有：

- if 声明
- for 声明
- 标签名
- 文本表达式

节点类型有：

- IfNode
- ForNode
- TextNode
- Element
- Component
- SlotNode
- TemplateNode

因为每个节点都通过 `createNode` 方法创建，所以它的性能是极其重要的。那这个过程的实现，有哪些性能相关的考虑呢？

首先，**预热** 过程提前选择好 [ANode](https://github.com/baidu/san/blob/master/doc/anode.md) 节点对应的实际类型。See [preheat-a-node.jsL52](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js#L152) [preheat-a-node.jsL165](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js#L165) [preheat-a-node.jsL180](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js#L180) [preheat-a-node.jsL185](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/preheat-a-node.js#L185-L192)

在 `createNode` 一开始就可以直接知道对应的节点类型。See [create-node.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/create-node.js#L24-L26)

```js
if (aNode.Clazz) {
    return new aNode.Clazz(aNode, parent, scope, owner);
}
```

另外，我们可以看到，除了 Component 之外，其他节点类型的构造函数参数签名都是 `(aNode, parent, scope, owner, reverseWalker)`，并没有使用一个 Object 包起来，就是为了在节点创建过程避免创建无用的中间对象，浪费创建和回收的时间。

```js
function IfNode(aNode, parent, scope, owner, reverseWalker) {}
function ForNode(aNode, parent, scope, owner, reverseWalker) {}
function TextNode(aNode, parent, scope, owner, reverseWalker) {}
function Element(aNode, parent, scope, owner, reverseWalker) {}
function SlotNode(aNode, parent, scope, owner, reverseWalker) {}
function TemplateNode(aNode, parent, scope, owner, reverseWalker) {}

function Component(options) {}
```

而 Component 由于使用者可直接接触到，初始化参数的便利性就更重要些，所以初始化参数是一个 options 对象。


## 视图更新

### 从数据变更开始

在 [San](https://github.com/baidu/san/) 的组件中，当我们更改了数据，视图就会自动刷新。

```js
this.data.set('title', 'hello');
```

我们可以很容易的发现，`data` 是：

- 组件上的一个属性，组件的数据状态容器
- 一个对象，提供了数据读取和操作的方法。See [数据操作文档](https://baidu.github.io/san/tutorial/data-method/)
- Observable。每次数据的变更都会 `fire`，可以通过 `listen` 方法监听数据变更。See [data.js](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/runtime/data.js)

`data` 是变化可监听的，所以组件的视图变更就有了基础出发点。组件视图变更机制有几个关键的要素：

- 组件在初始化的过程中，创建了 `data` 实例并监听其数据变化。See [component.js#L256](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/component.js#L256)
- 视图更新是异步的。数据变化会被保存在一个数组里，在 `nextTick` 时批量更新。See [component.js#L779](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/component.js#L779-L784)
- 组件是个 `children` 属性串联的节点树，视图更新是个自上而下传递的过程：每个节点通过 `_update({Array}changes)` 方法接收数据变化信息，更新自身的视图，并向子节点传递数据变化信息。See [component.js#L685](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/component.js#L685-L687) [element.js#L254](https://github.com/baidu/san/blob/f0f3444f42ebb89807f03d040c001d282b4e9a48/src/view/element.js#L254-L256) ...


### 阻断

### 列表的操作

### keyed


## 吹毛求疵

在这个部分，我会列举一些大多数人觉得知道、但又不会这么去做的优化写法。这些优化写法貌似对性能没什么帮助，但是积少成多，带来的性能增益还是不可忽略的。


