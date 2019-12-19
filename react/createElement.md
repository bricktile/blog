# React.createElement 分析

Date: 2019-12-15

## Abstract

打算开始阅读 react 源码。所以打算先从如何创建一个 react element 开始看起。这篇文章介绍了 React.createElement API 中做的一些事情，以及一个 react element 的结构是什么样的

## 准备一个简单的 Demo

首先准备一个 demo。这个可以通过 cra 或者 codesandbox 很轻易的做到。个人推荐使用 cra，本地调试比较方便

```js
function App() {
  return (
    <div className="App">
      <h1>Hello CodeSandbox</h1>
      <h2>Start editing to see some magic happen!</h2>
    </div>
  );
}

const rootElement = document.getElementById("root");
ReactDOM.render(
  <>
    <App />
  </>,
  rootElement
);
```

可以看到上面的代码，由于 Fragment 是不会产生真正的 dom 的，所以最终生成的 react tree 应该是这样的。

```
App
  |-- h1
  |-- h2

```

首先，JSX 只是一种语法糖。JSX 最终还是会被编译成 React.createElement。最终编译出来的结果和下面的类似。

> 注：react 官方是推荐使用 JSX 但并不强制要求。JSX 既可以像 template 一样书写代码，但是又具有 js 的特点。关于 JSX 的好处，可以另写文章讨论或者参考 react 官方。

```js
react.createElement(
  "Fragment",
  null,
  // 第三个参数往后是children
  react.createElement(App, props,
    __self: undefined
  })
);
```

我们可以看到，render 中有两个 Component，一个是 Fragment，Fragment 中嵌套了 App。所以转义出来的代码有两个 createElement。

那么 App 中的节点呢？ App 是一个 function，在这个 function 中也会调用 react.createElement 来创建 div、h1、h2 的 element。而 App 则是作为一个 function 被直接传入了 render。

## 接受的参数

react.createElement 接受多个参数。前两个参数分别是 type 和 props。第三个参数往后都是这个 element 的 children。

- type
- props
- children

## 1. 首先检查 type 是否合法

支持多种类型，对应着 react 中的多种组件。在开发模式下，react 会做 type 检查。在 production 模式下，会略过这一步。

类型：string | function | REACT_SYMBOLS | Object

有哪些 REACT_SYMBOLS 是合法的

```js
REACT_FRAGMENT_TYPE; // <></> 或者 <React.Fragment></React.Fragment>
REACT_CONCURRENT_MODE_TYPE; // concurrent mode is experimental
REACT_PROFILER_TYPE; // <React.Profiler id="xxx" onRender={() => {}} ></React.Profiler>
REACT_STRICT_MODE_TYPE; // <React.StrictMode></React.StrictMode>>
REACT_SUSPENSE_TYPE; // <React.Suspense fallBack={<Spinner />}>{...}<React.Suspense>
REACT_SUSPENSE_LIST_TYPE;
```

如果 type 是 object，type 不能是 null。 type.\$\$typeof 可以是以下

```js
type.$$typeof === REACT_LAZY_TYPE;
type.$$typeof === REACT_MEMO_TYPE; // React.memo
type.$$typeof === REACT_PROVIDER_TYPE; // React.createContext().Provider
type.$$typeof === REACT_CONTEXT_TYPE; // React.createContext().Consumer
type.$$typeof === REACT_FORWARD_REF_TYPE; // React.createContext().
type.$$typeof === REACT_FUNDAMENTAL_TYPE; // React.forwardRef
type.$$typeof === REACT_RESPONDER_TYPE;
type.$$typeof === REACT_SCOPE_TYPE;
```

## 2. 调用 createElement.apply(this, arguements)

在检查发现 type 合法后，执行 createElement。

```js
createElement.apply(this, arguements); // 等价于 createElement(type, props, children)
```

在这个函数中，主要会生成 element 所需要的 props，拿到 element 的 key 和 ref。

### 2.1. 生成 props

值得注意的一点是，生成的 props 是不包括 key 和 ref 的。请看一下例子：在 App 上，我们设置了 key，newprops 和 ref。但是生成的 props 中，只有 newProps 这一个属性。

```js
<App key="app" newProps="test" ref={ele => console.log(ref)} />
```

这样的写法很容易让人误会 key 和 ref 也会作为 props 传递到下一层。所以 react 在 props 上设置了 key 和 ref 的 getter。一旦通过 props.key 或者 props.ref 来获取这两个值，react 会做一个友情提示。

react 提供了一个 defaultProps 的 api 来让用户设置一个默认的 props。在生成 props 这个阶段，会把 props 和 defaultProps 做一个 merge，以此来保证 props 中没有设置的属性也存在一个默认值。

### 2.2. 生成 children

react 中，props 上还有一个属性就是 children。我们可以通过 children 来拿到被嵌套的组件。比如在 App 中，h1、h2 就是 App 的 children。

children 是以参数的形式传给 createElement API 的。可以是多个，也可以是一个。单个的情况也可以是一个数组，一个迭代器或者一个 node。

通过读取 arguments 第三个之后的参数，就可以获取到 children 了。

```js
// 单个
props.children = children;
```

```js
// 多个
var childArray = Array(childrenLength);

for (var i = 0; i < childrenLength; i++) {
  childArray[i] = arguments[i + 2];
}

{
  if (Object.freeze) {
    Object.freeze(childArray);
  }
}
props.children = childArray;
```

### 2.3. 返回一个 ReactElement

这个时候，我们得到了需要创建 react 的一系列条件。

self 和 source 是在 dev 环境下才需要的，我们可以先忽略。
关于 ReactCurrentOwner 的作用，作者目前也不太清楚，但是这个不影响创建 element。发现一片文章对 ReactCurrentOwner 做了详细的解释 -- [React ReactCurrentOwner](http://www.que01.top/2019/06/28/react-ReactCurrentOwner/)

```js
ReactElement(type, key, ref, self, source, ReactCurrentOwner.current, props);
```

#### 2.3 创建 element

通过 ReactElement 这个 function，可以得到一个结构如下的 element。里面记录了 element 的一系列信息。

其中有一个 validated 的属性被放在了\_store 中。 这个 validated 在之后用作判断 element 是否正确设置了 key，所以是一个 flag。但是整个 element 和 element 的 props 是被 freeze 住了的，所以，将这个 validated 放到\_store 中。如此一来，即使 element 被 freeze 了，也可以修改 validated。

```js
const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,

    // 以下三个属性都是在 dev 的时候设置
    __store: {
      validated: false // not enumerable,not configurable, not writable
    }
    __source: source, // not enumerable, not configurable
    __self: self, // not enumerable, not configurable
  };
```

回到最开始的 render 入口的话，可以发现代码其实是

```js
const fragmentElement = react.createElement(
  "Fragment",
  null,
  // 第三个参数往后是children
  element // App component
);
ReactDOM.render(fragmentElement, root);
```

## 3. 验证 children 的 key

在 react 中，如果 children 是一个数组，那么 react 会要求你为每一个 child 都设置一个 key。
在这一步，react 会校验需要 key 的 children 是否设置了 key，但是不会校验每个 key 是否唯一。

```js
for (var i = 2; i < arguments.length; i++) {
  validateChildKeys(arguments[i], type);
}
```

从 arguments[2]开始遍历每个 child。可以看到，在 validateChildKeys 内部，分为三种情况。

- 传入的 node 是一个数组
- 传入的 node 是单个节点
- 传入的 node 是一个迭代器

```js
function validateChildKeys(node, parentType) {
  if (typeof node !== "object") {
    return;
  }
  if (Array.isArray(node)) {
    for (let i = 0; i < node.length; i++) {
      const child = node[i];
      if (isValidElement(child)) {
        validateExplicitKey(child, parentType);
      }
    }
  } else if (isValidElement(node)) {
    // This element was passed in a valid location.
    if (node._store) {
      node._store.validated = true;
    }
  } else if (node) {
    const iteratorFn = getIteratorFn(node);
    if (typeof iteratorFn === "function") {
      // Entry iterators used to provide implicit keys,
      // but now we print a separate warning for them later.
      if (iteratorFn !== node.entries) {
        const iterator = iteratorFn.call(node);
        let step;
        while (!(step = iterator.next()).done) {
          if (isValidElement(step.value)) {
            validateExplicitKey(step.value, parentType);
          }
        }
      }
    }
  }
}
```

### 3.1 验证 node 是数组的 children 是否设置了 key

改造一下 App，在 App 中渲染一个 list。

```js
// react.createElement('div', props, [ele1, ele2, ele3, ..., ele8])
function App1() {
  const list = [1, 2, 3, 4, 5, 6, 7, 8];
  return (
    <div className="App">
      {list.map(item => (
        <li key={item}>{item}</li>
      ))}
    </div>
  );
}
```

这个时候在 react.createElement 中的第三个参数，是一个 children 数组，也就是说 arguments[2]是一个数组。

```js
react.createElement('div', props, [ele1, ele2, ele3, ..., ele8])
```

这时传入 validateChildKeys 中的是`[ele1, ele2, ele3, ..., ele8]`。 validateChildKeys 发现传入的是一个数组，则又对该数组遍历，如果数组中的每个 node 都设置了 key，那么说明没有问题。否则提示用户，在一个列表中，每个 child 都应该有一个 key。

### 3.2. 验证 node 既不是数组也不是 iterator 的情况

这种情况不需要设置 key，所以在这种情况下，只需要将 element 中的\_store.validated 设置为 true。

### 3.3. children 是一个 iterator

除了数组以外，还可以传入一个 iterator。iterator 目前不是一个 standard 标准。下面是一个来自 MDN 简单的迭代器。

修改一个 App，看看迭代器作为 children 的结果。如果 react 收到一个迭代器作为 children，react 会尝试去取 iterator[Symbal.iterator]或者 iterator['@@iterator']的结果作为 iteratorFn。通过 iterator.next 来遍历 node。上述的 node 没有设置 key，这时 react 会有一个 warning 来提示作者没有正确设置 key，但不影响渲染。

```js
// react.createElement('div', props, myIterable)
function App() {
  const myIterable = {
    *[Symbol.iterator]() {
      yield (<li>1</li>);
      yield (<li>2</li>);
      yield (<li>3</li>);
    }
  };
  return <div className="App">{myIterable}</div>;
}
```

### 3.4 如何区别这三种 children

现在我们知道了 children 的形式有三种。这三种形式的 children 会被转义成三种代码：

```js
// react.createElement(type, props, child1, child2, child3....)
// 不会校验是否设置了key
<App>
  <Child1 />
  <Child2 />
  <Child3 />
  ...
</App>

// react.createElement(type, props, [child1, child2, child3....])
// 需要设置key
<App>
  {
    children.map(child => <Child key={child.id} />)
  }
</App>

// react.createElement(type, props, iterator)
// 需要设置key
const myIterable = {
    *[Symbol.iterator]() {
      yield <li key={1}>1</li>
      yield <li key={2}>2</li>
      yield <li key={3}>3</li>
    },
  }
<div className="App">{myIterable}</div>
```

到目前为止，我们就创建好了 element 了

## 4 总结

1.  JSX 的语法会被转成 React.createElement
2.  children 既可以是数组也可以是 iterator 也可以只是 node。iterator 和数组作为 children 时 时，都需要设置 key
3.  props 中没有 key 和 ref。
4.  Fragement 只支持两种属性，一个是 key，一个是 props 中的 children。并不支持 ref，因为不会渲染出真正的节点
5.  一个 react element 是一个 Object。其中有 `type` 和 `$$type` 两个属性。`type`并不表示节点的 html 类型。例如 App 的`type`是一个创建 App 节点的 function -- 也就是 `function App() {...}`。`$$type`表示的是 react element 的类型。在上述的例子中，`$$type` 是一个 Symbol，值为 `Symbol.for('react.element')`

## 待解决的问题

1. REACT_SUSPENSE_LIST_TYPE 是什么类型的 Node？
2. REACT_RESPONDER_TYPE 和 REACT_SCOPE_TYPE 分别是什么？
3. ReactCurrentOwner 的作用？
