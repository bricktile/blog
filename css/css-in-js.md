# 为什么要用 css in js

Date: 2020-01-05

## Abstract

写这篇文章的原因是因为在面试中问到`css in js`的优势时，因为自己平时没有很好的总结，所以回答的磕磕巴巴。因此写这篇文章做一个总结。

第二个原因是，在尝试过`css in js`之后，我还是非常喜欢它的。所以想写一篇文章介绍。
这篇文章会主要介绍：

- 什么是 `css in js`
- Before `css in js`
- 为什么选择 `css in js

## 什么是 `css in js`

个人理解就是写在 js 里的 css（用 js 的方式管理 css）。

**举例：**

这是一个来自 [styled-components](https://www.styled-components.com/) 的代码片段。 Button 是一个样式组件。没错，styled-components 在这个创造了一个样式组件。看到这里，你可能会觉得非常疑惑，甚至觉得没有必要，因为在 css 中完全可以实现。

这个代码片段只是用来展示什么是`css in js`。文章之后会介绍`css in js`的一些好处。

```js
const Button = styled.a`
  /* This renders the buttons above... Edit me! */
  display: inline-block;
  border-radius: 3px;
  padding: 0.5rem 0;
  margin: 0.5rem 1rem;
  width: 11rem;
  background: transparent;
  color: white;
  border: 2px solid white;
  ${props =>
    props.primary &&
    css`
      background: white;
      color: palevioletred;
    `}
`;
```

## Before `css in js`

那我们来看一下不用 `css in js` 的一些 css 方案。

### 1. Pure css

纯手写 css。如果项目非常的小，直接写 css 确实非常的方便。可能一个几百行的 css 文件就搞定了。但是随着现在前端的发展，目前的项目如果是纯 css 的话，显然是不现实的。

以设计一个 UI 框架为例子（比如 Bootstrap），很难预防在升级版本中，设计师脑洞大开，要将所有的蓝色加深一点点。这个时候，如果是 css 的话，要将所有设计的蓝色都改动一遍是会让人非常崩溃的。

又比如说，有两个 button，区别在于一个背景是红色，一个背景是蓝色。这个时候，我们要么为每个 button 写两个类--'button-base button-color'，要么在 css 中重复书写样式。

在维护 css 的过程中会出现很多问题：样式的复用，css 选择器，兼容问题等等。所以就出现了 less 和 scss。

### 2. css preprocessor

css 预处理器用一句话解释就是：用一种新语法去描述样式，然后最后会被编译成 css 文件。

举例说明（以 scss 为例子）：

```scss
$mainblue: 5em;

button {
  background: $mainblue;
}
```

scss 支持设置一个全局变量，这个时候去改一个颜色非常简单。不仅如此，预编译器还支持很多其他功能，比如 mixin，嵌套，if-else，循环等等。

### 优势

- 提供变量、mixin 等语法支持
- 解决兼容性问题

### 缺点 1

css preprocessor 本身没有解决 css 类名重复的问题。如果一个大项目不采用一些措施来避免这个问题的话，造成的后果就是，类名越来越多，越来越长。

对于反对使用 css-in-js 的人来说，他们可能会提出很多方案来解决这个问题，比如 BEM 或者 css module。 从我的角度看，BEM 是一种因为无法解决 css 本身的缺陷所以只能定义规范的做法，个人并不认为这是一个什么很好的解决方案。

实际上，我个人非常不喜欢 BEM。BEM 不但在使用上让我每次取类名都非常纠结外，还会让类名越来越长。

我看到过一种解决类名过长的方案，很 tricky，但非常难以维护：

```html
<div class="block">
  <ul class="block__list">
    <li class="block__list-item"></li>
  </ul>
</div>
```

```scss
.block {
  &__list {
    &-item {
    }
  }
}
```

这个写法目前是我最讨厌的写法之一了。首先，它没有减少 html 中的任何类名长度。一个 xxx\_\_xxx_xxx-xxx 类名，还是得老实地写在 html 中（当然在 JSX 也是可以做一些拼接的）。其次，如果在一个大项目中，这样的写法会导致我根本找不到这个类的样式到底定义在哪里。试想一个 scss 文件内是一个非常深的嵌套格式，里面有非常多的 list 和 item 组合，找到对应的正确的 list 和 item 让人崩溃。

### 解决方案

这个时候，得益于 webpack 的 loader 机制， css-loader 支持 css modules，于是在 css 文件里，也可以定义本地样式了。但是这又引入了一种新的怪异的写法。

```js
import styles from "./styles.scss";

<div className={styles["item-list"]} />;
```

```scss
:local(.item-list) {
  color: grey;
}
```

webpack 的 css-loader 的配置也非常简单。个人感受的话是目前大部分前端样式的解决方案都是通过 scss + css-loader 来解决的。

### 缺点 2

整个项目有两种维护方式，一种是 js 模块，一种是 scss 管理方式（一些全局变量的 scss 文件，各种 scss 文件混在其他有 js 文件的目录下）。 scss 有自己的变量，js 也有自己的模块。

## 为什么选择 `css in js`

从上一章节来看，css processor 已经有很多功能了，为什么还要选择`css in js`？ `css in js` 解决了一个十分重要的问题就是 css 到 dom 的 mapping 问题。

### css 到 js/dom 的映射关系

比如下面的例子，只能通过`item-list`来对应 scss 文件中的类名`styles` 变成了一个 object。在 scss 文件中，是不会限制类名中使用`-`的，所以在 dom 中，还是需要通过 key 的 mapping 来指定对应的样式。

```js
import styles from "./styles.scss";

<div className={styles["item-list"]} />;
```

```scss
:local(.item-list) {
  color: grey;
}
```

其实到这一步就非常明显了，`styles`是一个 object，那为啥不直接写一个 object，反而需要去配置 loader 来转成 object 呢？

```js
export const itemList = "color:xxx";
```

就以上例子来说，直接用 js 的好处就是：

- 类名直接跳转（因为是一个变量了）。
- 本身就是一个模块，不需要关心命名冲突，懒加载等问题。
- 不再需要依赖字符串去找到对应的样式了。

css 到 dom 的映射消失是非常重要的。如果不用 `css in js` 怎么写？看一下以下例子，如果在一个项目中，有很多的 default button：

1. 好像没有必要单独为了样式去封装一个组件，但是每次都要重新书写一遍 className 也让人烦恼，
2. 而且可能会有 typo 的错误。

又如果我只有在几个地方需要用到 default button，为了不重复书写 css，我必须把"default-button"的样式放到 global 的 css 或者 scss 中去。

```jsx
<div className="default-button">{this.props.children}</div>
```

试想一下这个目录结构：

```
- global-scss
    |- styles.scss // 有 default-button 样式
- views
    |- pages
        |- page1
            |- index.jsx
            |- styles.scss // 有 default-button 样式，覆盖global下的default-button样式
        |- page2
            |- index.jsx
            |- styles.scss
```

有太多 scss 文件了，当我们想要修改样式的时候，一个一个去 scss 文件找显然是不现实的。而下面这种写法，又会让全局搜索无法找到全部的样式

```scss
// 有时为了方便会这么写
.default {
  &-button {
  }
  &-text {
  }
}
```

而且在 UI 中我们可能会遇到这样的写法：

```jsx
let type = condition ? "default" : "danger";
<div className={`${type}-button`}>{this.props.children}</div>;
```

上面两个例子，在全局搜索中是非常容易被漏掉的。

**总结**

个人理解，这就是 css 到 js 的映射关系。是通过字符串来指定样式名的。而在 js 和 scss 中，字符串可以被拼接。这使得之后想要找的对应的类名非常不方便。

### css 是一个变量/组件

`css in js`除了解决映射的问题，还解决了代码重用的问题。react 和 vue 这些框架的出现，前端就开始多了很多组件，就是为了代码重用。现在一个样式就可以是一个组件。重用样式不需要去污染全局命名了。样式可以像一个模块/变量一样。当我们需要重复使用这个样式的时候，我们不需要在全局写一个'default-button' 的样式。我们只需要 'import' 这个样式模块就可以了。

更重要的是，这样的写法，在下一次需要修改的时候，至少在全局搜索这一步，不会漏掉一些代码。

以 styled-component 为例子：

```js
const DefaultButton = styled.a`
  /* This renders the buttons above... Edit me! */
  display: inline-block;
  border-radius: 3px;
  padding: 0.5rem 0;
  margin: 0.5rem 1rem;
  width: 11rem;
  background: transparent;
  color: white;
  border: 2px solid white;
  ${props =>
    props.primary &&
    css`
      background: white;
      color: palevioletred;
    `}
`;

// jsx
<DefaultButton primary />;
```

### 优势总结

`css in js` 带来的优势是非常明显的。

1. 不需要额外的配置来支持 css mudules，在 js 中，变量/组件本身就是一个模块。
2. 不需要为重名担心。
3. 更方便的 css 管理（例如不用的 css 变量很可能只需要 lint 就能检查出来）。
4. 动态的样式。
5. auto-prefix。
6. 统一用 js 的方式管理，不存在 scss 变量和 js 变量。
7. ...

这些好处是因为在 js 中，我们可以利用 js+webpack 提供的模块化，js 本身就是一个可执行的脚本语言，不需要额外的依赖(css preprocessor，css module)等来为 css 做很多优化。之后随着前端的发展，个人觉得`css in js`可以做的事更多。

如果 react 提供了 virtual dom，那么 styled-component 也完全可以提供 "virtual-styled-dom"。

## Reference

[stop-using-css-in-javascript-for-web-development](https://hackernoon.com/stop-using-css-in-javascript-for-web-development-fa32fb873dcc)

https://hackernoon.com/all-you-need-to-know-about-css-in-js-984a72d48ebc

https://github.com/cssinjs

https://www.styled-components.com/docs/basics#motivation

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
