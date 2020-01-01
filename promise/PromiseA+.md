# 如何根据 promsie A+ 实现一个 Promise

Date: 2019-12-19

## Abstract

这篇文章会根据 Promise A+ 规范实现一个 promise。为了不和原生的 Promise 搞混，我们把我们将要实现的 Promsie 叫做 PromiseJ。PromiseJ 实现 Promise A+ 规范中的 thenable 和 "The Promise Resolution Procedure"。

在这篇文章中：

_"promise"表示一个 Promise 实例_

_"Promise"表示一个 Promise 类_

## Promise 的用法

先来看一下 Promise 的用法。一个 Promise，首先要接受一个回调函数，这个回调函数接受两个参数分别是 resolve 和 reject。

```js
const p = new Promise((resolve, reject) => {
  //
});
```

这个时候我们可以很简单的写一个 demo，它是一个类，构造函数接受一个回调函数作为参数，而触发回调函数时，回调函数又接受 resolve 和 reject 作为参数。

```js
class PromiseJ {
  constructor(callback) {
    // new PromiseJ((resolve, reject) => {}), callback就是(resolve, reject) => {}
    // 在promsie机制中，传进来的回调会被立即执行
    callback(this._resolve, this.reject);
  }

  // 这里我们先不考虑Promise的静态resolve和reject方法，我用一个"_"前缀表示这是一个私有方法
  _resolve = () => {};
  _reject = () => {};
}
```

## Promise 的状态

Promise 有三种状态

- pending
- fulfilled
- rejected

关系如下图所示：

1. 当一个 Promsie 是 pending 的时候，那么这个 Promise 最终将变为 fulfilled 或者 rejected。
2. 当一个 Promsie 是 fulfilled 的时候，那么这个 Promise 有一个 value，且 value 不可更改。而 Promsie 的状态也不能改成 pending 或 rejected 了。fulfilled 是一个最终状态。
3. 当一个 Promise 是 rejected 的时候，那么这个 Promsie 有一个 reason，且 rejected 也是 Promise 的最终态，不能修改。

![promise-status](https://github.com/bricktile/blog/raw/promise/promise/promise-status.jpg)

给 PromiseJ 添加状态，初始状态为 pending

```js
const STATUS = {
  PENDING: "pending",
  FULFILLED: "fulfilled",
  REJECTED: "rejected"
};

class PromiseJ {
  constructor(callback) {
    // Promise状态
    this.state = STATUS.PENDING;
    // new PromiseJ((resolve, reject) => {}), callback就是(resolve, reject) => {}
    // 在promsie机制中，传进来的回调会被立即执行
    callback(this._resolve, this.reject);
  }
  // ...
}
```

### Promise 的状态是在什么时候变化的？

到这一步，PromiseJ 拥有了状态，但是没有状态转换的逻辑。在浏览器中运行一下下面的两个例子：

1. p1 作为一个 Promise 实例，p1 的状态是什么？

```js
p1 = new Promise(res => 3);
```

2. 这里的 p2 和第一题的 p1 状态是一样的吗？

```js
p2 = new Promise(res => {
  res(3);
});
```

**答案**

1. pending
2. resolved

由此可知，PromiseJ 的状态应该是在 PromiseJ.\_resolve 和 PromiseJ.\_reject 中被改变。

接着上面的代码，在 \_resolve 和 \_reject 方法中加入改变 PromiseJ 状态的逻辑。根据 Promise A+ 的规范，value 和 reason 都是不可修改的。fulfilled 和 rejected 不可以转变成其他状态。

```js
  _resolve(value) {
    if(this.state !== STATUS.PENDING) {
      return
    }
    this.state = STATUS.FULFILLED;
    // value一旦被设置，不可修改
    Object.defineProperty(this, "value", {
      value,
      writable: false
    });
  }

  _reject(reason) {
    if(this.state !== STATUS.PENDING) {
      return
    }
    this.state = STATUS.REJECTED;
    // reason一旦被设置，不可修改
    Object.defineProperty(this, "reason", {
      value: reason,
      writable: false
    });
  }
```

## Promise 中的 then

此时，PromiseJ 已经可以修改状态了。在修改完状态之后，就是要触发 then 中的回调。这一小节，我们来看一下 promise.then。

以下代码是 PromiseJ 应该要实现的功能：p 被 resolve 时要输出“p is fulfilled”；p 被 reject 时，要输出“p is rejected”。

```js
const p = new PromiseJ((resolve, reject) => {});
p1 = p.then(() => console.log("p is fulfilled")); // p 是fulfilled，触发 then 中的回调
p2 = p.then(
  () => {},
  () => console.log("p is rejected") // p 是rejected，触发 then 中的回调
);
```

### Promise then 的入参

then 有两个入参，分别是 onFulfilled, onRejected。规范中对这两个参数做了详细的解释。

```js
promise.then(onFulfilled, onRejected);
```

### then 的规范

首先，我们先来翻译一下 Promise A+关于 then 的规范。这里我们先逐字逐句的翻译，然后再一条一条解释。

1. onFulfilled 和 onRejected 都是可选的入参：

- 如果 onFulfilled 不是一个 function，忽略。
- 如果 onRejected 不是一个 function，忽略。

2. 如果 onFulfilled 是一个 function：

- 必须在 promise 被设置为 fulfilled 之后调用，第一个参数是 promise 的 value。
- 不能再 promise 被设置为 fulfilled 之前调用。
- 只允许调用一次。

3. 如果 onRejected 是一个 function：

- 必须在 promise 被设置为 rejected 之后调用，第一个参数是 promise 的 value。
- 不能再 promise 被设置为 rejected 之前调用。
- 只允许调用一次。

4. onFulfilled 或者 onRejected 必须是异步的。
5. onFulfilled 和 onRejected 中 this 的值不存在。
6. then 可以在同一个 promise 上多次调用：

- 当 promise 是 fulfilled, 所有多次 then 中的 onFulfilled 回调会按照顺序被执行。
- 当 promise 是 rejected, 所有多次 then 中的 onRejected 回调会按照顺序被执行。

7. then 必须返回一个 promise：

```js
promise2 = promise1.then(onFulfilled, onRejected);
```

- onFulfilled 或 onRejected 返回一个值 x, 执行 Promise Resolution Procedure [[Resolve]](promise2, x)。
- onFulfilled 或 onRejected 抛出错误, promise2 必须 reject 这个错误。
- onFulfilled 不是一个 function 且 promise1 是 fulfilled, promise2 必须把 promise1 的 value 作为自己的 value 来。
- onRejected 不是一个 function 且 promise1 是 rejected, promise2 必须把 promise1 的 reason 作为自己的 reason 来 reject。

以上 7 条，是 Promise A+ 规范关于 then 的全部解释。

首先，我们先判断 onFulfilled 和 onRejected 是不是函数。

```js
then(onFulfilled, onRejected) {
  if(isFunction(onFulfilled)) {}
  if(isFunction(onRejected)) {}
}
```

然后看一下第 2，3 条规则：onFulfilled 在 fulfilled 之后调用，onRejected 在 rejected 之后调用，且只能调用一次。

在介绍 Promise 状态的时候，我们知道 Promise 的状态改变是在 resolve 和 reject 方法中做的。那么是不是在 resolve 中改变状态之后直接调用 onFulfilled 和 onRejected 就可以了呢？答案是不行。

两个例子：

```js
// 先执行then，然后执行resolve(3)，最后执行 value => console.log(value)
p0 = new Promise(resolve => setTimeout(() => resolve(3))).then(value =>
  console.log(value)
);
// 先执行resolve(3)，然后执行then，最后执行 value => console.log(value)
p1 = new Promise(resolve => resolve(3)).then(value => console.log(value));
```

p0 在调用 resolve 的时候，then 已经被调用过了。在 p0 中，onFulfilled 和 onRejected 需要被存起来，等待当前 promise 状态发生变化后调用。

p1 在调用 resolve 的时候，还没有执行 then。在 p1 中，resolve 执行时，没有 onFulfilled 和 onRejected。所以在 p1 的 then 中，如果发现当前的 promise 状态不是 pending，then 中需要去调用 onFulfilled 和 onRejected。

所以 then 可以在 resolve 前也可以在 resolve 之后。那么 resolve 的时候，很有可能是没有 onFulfilled 和 onRejected 回调的。所以 then 中可能出现当前 promise 是 pending、fulfilled 和 rejected 三种状态。

**promise.then 中可能出现的情况**

![promise-then](https://github.com/bricktile/blog/raw/promise/promise/promise-then.jpg)

根据上面的描述，then 中和 resolve, reject 中都可能触发 onFulfilled 和 onRejected。

进一步完善 then：

```js
then(onFulfilled, onRejected) {
  const newPromise = new PromiseJ(() => {});
  // promise1 是 fullfiled
  if (this.state === STATUS.FULFILLED) {
    // onFulfilled 是一个function
    if (isFunction(onFulfilled)) {
      this._processOnfulfilled();
    } else {
    }
  } else if(this.state === STATUS.REJECTED) {
    if (isFunction(onRejected)) {
      this._processOnRejected();
    } else {

    }
  } else if (isFunction(onFulfilled)) {
    someWhereWeStoreOnFulfilled = onFulfilled;
  } else if(isFunction(onRejected)) {
    someWhereWeStoreOnRejected = onRejected;
  }

  return newPromise;
}
```

这里我们发现，在 promise 状态是 fulfilled 和 rejected 的情况下，onFulfilled 和 onRejected 不是函数的时候我们没有处理。但是 Promise A+ 已经给出了规范，参考 then 规范的第七条：

- onFulfilled 不是一个 function 且 promise1 是 fulfilled, promise2 必须把 promise1 的 value 作为自己的 value 来。
- onRejected 不是一个 function 且 promise1 是 rejected, promise2 必须把 promise1 的 reason 作为自己的 reason 来 reject。

所以代码可以加上：

```js
then(onFulfilled, onRejected) {
  const newPromise = new PromiseJ(() => {});
  // promise1 是 fullfiled
  if (this.state === STATUS.FULFILLED) {
    // onFulfilled 是一个function
    if (isFunction(onFulfilled)) {
      this._processOnfulfilled();
    } else {
      newPromise._resolve(this.value);
    }
  } else if(this.state === STATUS.REJECTED) {
    if (isFunction(onRejected)) {
      this._processOnRejected();
    } else {
      newPromise._reject(this.value);
    }
  } else if (isFunction(onFulfilled)) {
    someWhereWeStoreOnFulfilled = onFulfilled;
  } else if(isFunction(onRejected)) {
    someWhereWeStoreOnRejected = onRejected;
  }

  return newPromise;
}
```

现在我们已经完成了 then 规范的 1、2、3 部分和第 7 条的 3、4 部分，还缺少 4、5、6 部分，第 7 条的 1、2 部分。

在继续 4、5、6 部分之前，还有几个问题需要解决。

1. onFulfilled 和 onRejected 存放在哪里？

目前看来，可以放在当前的 promise 中。可以为当前的 promise 加两个属性--onFulfilledQueue 和 onRejectedQueue（作为一个 queue 的原因是同一个 promise 可以多次调用 then）。

2. then 必须返回一个全新的 promise 吗？

规范中说明了 promise2 可以和 promise1 相等。但是考虑到 promise 的状态一旦变成了 fulfilled 和 rejected 之后就不可以再修改，而 fulfilled 的 promise1.then 却是可以返回 pending 的 promise2 的，所以在这种情况下，promise2 和 promise1 一定是不相等的。而且在浏览器中测试 Promise 会发现，无论是否传入 onFulfilled 和 onRejected，无论这两个参数是否合法，then 都会返回一个全新的 promise。所以在这篇文章中，每个 then 都会返回一个新的 promise。

```js
// 创建一个pending的promise
a = new Promise(() => {}); // promsie<pending>
// 没有
b = a.then({}, {}); // promise<pending>，即使没有 onfulfilled和onRejected，也返回的是新的Promise
a === b; // false
```

#### onFulfilled 或者 onRejected 必须是异步的

这里的异步，其实规范里没有明确是用微任务还是宏任务。但是在 v8 中明确指出，promise task 是 microtask。所以这将会用 queueMicrotask 这个 api 来做异步任务。注意，node v11 以上版本才支持 queueMicrotask。

#### onFulfilled 和 onRejected 中 this 的值不存在

目前的想法：xxx.onFulFilled 这样的调用不可行，直接将 onFulfilled 取出调用。

#### 同一个 promise 上可以多次调用 then

- then 可以在同一个 promise 上多次调用

当前 promise 有多个 onFulfilled 和 onRejected。这里不贴全部代码了，逻辑是在存放 onFulfilled 和 onRejected 时，把他们存放到一个数组里。

```js
// ...
onFulfilledQueue.push(onFulfilled);
// ...
onRejectedQueue.push(onRejected);
// ...
```

因为第 7 条涉及到 resolution procedure，处理方式会放到 resolution procedure 中讲解。

## The Promise Resolution Procedure

promise.then 规则中的第 7 条提到了这一概念。

- onFulfilled 或 onRejected 返回一个值 x, 执行 Promise Resolution Procedure [[Resolve]](promise2, x)。

The Promise Resolution Procedure 可以理解为下面这段代码。一个 resolve function，有两个参数，x 和 promise，定义了在 x 为各种值的情况下，promise 应该是什么状态。

```js
[[Resolve]](promise, x);
```

我们需要一个 resolve resolution function

```js
function resolveResolution(promise, x) {
  //
}
```

### 跟着规范写 if-else

我们先跟着规范将 resolveResolution 写出来

#### 1. 如果 promise and x 是同一个引用, promise.reject(TypeError)

```js
function resolveResolution(promise, x) {
  if (promise === x) {
    promise._reject(new TypeError("promise should not be equal to x"));
  }
}
```

#### 2. 如果 x 是一个 promise

1. x 是 pending 的时候

- promise 等待 x resolve 或者 reject。

2. x 是 fulfilled 的时候

- promise.resolve(x.value)

3. x 是 rejected 的时候

- promise.\_reject(x.reason)

加入这段逻辑

```js
function resolveResolution(promise, x) {
  if (promise === x) {
    promise._reject(new TypeError("promise should not be equal to x"));
  } else if (x instanceof PromiseJ) {
    if (x.state === STATUS.PENDING) {
    } else if (x.state === STATUS.FULFILLED) {
      promise._resolve(x.value);
    } else {
      promise._reject(x.reason);
    }
  }
}
```

**提问：** 这个时候我们发现了一个问题，x 由 pending 变成 fulfilled 或者 rejected 的时候，如何通知 promise 也做同样的转变？

**思路：** promise 是链式调用的，每个 then 会产生一个新的 promise。会形成一个 promise chain。那么可以把 promise 接到 x 的后面，形成一个新的 promise chain。

```js
p0 = new Promise(resolve => {
  setTimeout(() => {
    resolve(3); // resolve 之后，会触发下一个 promise
  }, 1000);
});
p1 = p0.then();
p2 = p1.then();
```

#### 3. x 是一个 object 或者一个 function

1. 如果 x.then 存在，then = x.then
2. 如果在得到 x.then 的值的过程中报错，promise reject 这个错误

```js
x = {
  then: (function() {
    throw new Error("x.then will throw an error");
  })()
};
```

3. 如果 then 是一个 function（1 中得到的 then，也就是 x.then）, then.call(x, resolvePromise, rejectPromise)

4. then 不是一个 function，promise.\_resolve(x)

重点来看看一下第 3 点：then.call(x, resolvePromise, rejectPromise)。

**提问：** resolvePromise 和 rejectPromise 分别是什么？

我们知道，onFulfilled 返回一个 x 会触发 “The Promise Resolution Procedure”。所以我们在浏览器里测试一下以下代码：

第一个例子：

```js
p0 = new Promise((resolve, reject) => {
  resolve(100);
}).then(value => ({
  then: (resolvePromise, rejectPromise) => {}
}));
```

**提问**：p0 是 pending 还是 fulfilled？

第二个例子：

```js
p1 = new Promise((resolve, reject) => {
  resolve(100);
}).then(value => ({
  then: (resolvePromise, rejectPromise) => resolvePromise(value + 200)
}));
```

**提问**：p1 是 pending 还是 fulfilled？

**答案**

1. p0: pending
2. p1: fulfilled

**结论**

resolvePromise 和 rejectPromise 分别是某个 promise 的\_resolve 和 \_reject。那么这个时候，x 并不是一个 promise，当前的 promise 也不能修改状态，所以只能是传入进来的 promise 了。

#### 4. x 既不是 Object 和 function，也不是 Promise

promise.\_resolve(x)

## 如何设计一个 promise chain

上面两小节已经重点介绍了 promise 的两大核心规范。但是到目前为止我们还有一些问题。

我们要让 promise 能够像这样的链式调用：

![promise-chain](https://github.com/bricktile/blog/raw/promise/promise/promise-chain.jpg)

在文章介绍 then 规范的时候，提到了一个 promise 上可以多次的调用 then，假设这个 promise 叫做 promise1。每个 then 又传入了 onFulfilled 和 onRejected，所以我们将 onFulfilled 和 onRejected 分别存在了当前 promise1 的 onFulfilledQueue 和 onRejectedQueue 上。所以一开始我们设计的数据结构是这样的

```ts
interface PromiseJ {
  onFulfilledQueue: Array<Function>;
  onRejectedQueue: Array<Function>;
}
```

但是很快你就会发现这样的数据结构是有问题的，当上图的 promise1 被 resolve 了之后，promise1 因为存了 onFulfilled 和 onRejected，所以 promise1 能正确的触发 then 中的 onFulfilled 和 onRejected。那么这个时候我们怎么修改 promise2 的状态？我们没有存 promise2 的 resolve 和 reject。

这里有两种种处理方法：

**第一种方法：**
这是我在网上看到其他人实现的一种方法。具体原理是：

在 then 中返回一个全新的 promise2 时，我们是可以拿到这个 promise2 的 resolve 和 reject 的，为了方便理解，我们把 promise2 的 resolve 和 reject 叫做 resolve2 和 reject2，下一个 promise3 的叫做 resolve3 和 reject3。

现在来看一下 promise1 的三种状态下的情况。

1. fulfilled -- 直接 fulfilled promise2，不用存 onFulfilled2 和 onRejected2。
2. rejected -- 直接 rejected promise2， 不用存 onFulfilled2 和 onRejected2。
3. pending -- 利用 js 的闭包原理，我们存下一个 function，这个 function 在 promise2 改变状态为 fulfilled 的时候被触发，这个 function 主要触发 resolve2，这样 promise2 就能成功被 resolve 了。同时，这个 function 也触发 onFulfilled，然后执行[[Resolve]](promise2, x);

第三点用代码来解释比较清晰，以下代码只是展示一个大概思路，并不是可以直接运行的：

```js
then(onFulfilled, onRejected) {
  const {state, value, reason} = this;
  return new Promise((resolve, reject) => { // 这里拿到了 promise2 的 resolve 和 reject
    // 立即执行的回调
    const resolvePromise = (value) => {
      // 这里会产生闭包
      if(isFunction(onFulfilled)) {
        const x = onFulfilled(value);
        // promise resolution procedure，具体逻辑还需要根据规范来写
        resolution(promise, x);
      } else {
        resolve(value); // 闭包，可以访问 promise2 的resolve
      }
    }
    if(state === STATUS.PENDING) {
      // 如果 promise1 触发了 resolve1，那么 resove1 中去读取 callbackQueue 中的回调，就可以改变 promise2 的状态了。而promise2中也存了 promise5、promis6、promise7 的 resolve方法，这样就可以链式调用了
      this.callbackQueue.push(resolvePromise);
    } else if(state === STATUS.FULFILLED || state === STATUS.REJECTED) {
      // 这里可以直接触发 onFulfilled 或者 onRejected
    }
  })
}
```

**第二种方法：**

根据上面的图片，我们可以定义一个简单的关于 promise 的数据结构，我们通过这个数据结构来形成 Promise Chain。每个 promise 有自己的子 promise。

```ts
interface PromiseJ {
  nextPromiseQueue: Array<PromiseJ>;
}
```

看一下代码，这里我们先不考虑 reject 的情况。promise1 中存放了 promise2、promise3、promise4。每个通过 then 生成的 promise 中可能会有 onFulfilled 何 onRejected，这取决于它们是不是函数

```js
then(onFulfilled, onRejected) {
  //  这里总结下来应有五种情况，无论哪种情况，都需要返回新的promise
  const newPromise = new PromiseJ(() => {});

  // promise1 是 fullfiled
  if (this.state === STATUS.FULFILLED) {
    // onFulfilled 是一个function
    if (isFunction(onFulfilled)) {
      newPromise.onFulfilled = onFulfilled;
      this._processOnfulfilled(newPromise);
    } else {
      // onfulfilled 不存在
      newPromise._resolve(this.value);
    }
  } else if (this.state === STATUS.PENDING) {
    if (isFunction(onFulfilled)) {
      newPromise.onFulfilled = onFulfilled;
    }
    if (isFunction(onRejected)) {
      newPromise.onRejected = onRejected;
    }
    // 这里形成一个promise链
    this.nextPromiseQueue.push(newPromise);
  }
  return newPromise;
}
```

\_processOnfulfilled 中 异步执行了 “The Promise Resolution Procedure”。这符合 promise then 规范的第 7 点，可以往上回顾一下。

```js
_processOnfulfilled(newPromise) {
  // promise resolution procedure
  queueMicrotask(() => {
    try {
      let x = newPromise.onFulfilled(this.value);
      this._resolution(newPromise, x);
    } catch (e) {
      newPromise._reject(e);
    }
  });
}
```

然后我们要在 resolve 中加入遍历 nextPromiseQueue 的代码。

```js
_resolve(value) {
    // console.log("value", value, this);
    // 只有pending状态可以转变为fulfilled
    if (this.state !== STATUS.PENDING) {
      return;
    }
    this.state = STATUS.FULFILLED;
    // 这里有一个问题，如何保证value不被更改，现在想到的方法是用Object.defineProperty中设置writable
    Object.defineProperty(this, "value", {
      value,
      writable: false
    });
    while (this.nextPromiseQueue.length) {
      const promise = this.nextPromiseQueue.shift();
      if (promise && isFunction(promise.onFulfilled)) {
        this._processOnfulfilled(promise);
      }
    }
  }
```

这就是第二种思路的实现。实现 promise 链式调用的本质就是要拿到下一个 promise 的 resolve 和 reject。第一种思路利用闭包来访问当时的 promise 的 resolve 和 reject。第二种思路直接将 promise2 存在 promise1 中。

第二种思路有个问题：resolve 和 reject 应该是一个私有方法，不允许被外部像这样 promise.resolve() 调用。

但是个人感觉第二种思路结构更加清晰，直接使用 promise.resolve() 可读性也更加好，所以这篇文章使用第二种思路。所以这里我们把\_resolve 和\_reject 都改成 resolve 和 reject。

### 代码实现

其实到这一步，我们已经完成了 60%的工作了。现在我们有一个可以工作的 resolve，then，resolution procedure。 还差 reject 部分。

为什么在 then 中先忽略了 reject 呢？当我们把 onRejected 当做和 onFulfilled 一样处理的时候，我们会发现一个问题。

首先先来看一下下面例子在浏览器的输出：

```js
p1 = new Promise((res, rej) => rej(100)); // Uncaught (in promise) 100，Promise<rejected>
new Promise((res, rej) => rej(100)).then(
  () => {},
  reason => console.log(reason)
); // 100, Promise<resolved>
new Promise((res, rej) => rej(100)).then().then(
  () => {},
  reason => console.log(reason)
); // 100, Promise<resolved>
```

> 如果后续的 then 中没有 onRejected，那么就会抛出错误。

那么问题来了，抛出错误的时机是什么？在当前实现中，p1 中的 callback 会被立即执行，如果在 callback 中直接 reject 一个 reason（上面第一个例子），那么 reject 也是立即执行的。这个时候，p1 的状态被立即修改成了 rejected，但是 then 还没有发生，p1 也不知道是否需要抛出错误。所以在这个时候，处理抛出错误的情况必须等到所有的 then 都被执行了，当前 p1 才知道是否有合适的 onRejected。

resolve 就有所不同，在由 pending 变为 fulfilled 情况下，就算不知道后面是否有 then 也没关系。因为对于 p1 来说，不需要根据下一步的回调来产生不同的操作。

而 reject，如果不知道后面是否有 then 的 onRejected，reject 函数不知道是否需要抛错。而这个抛错是不能放在 then 中处理的, then 也不知道是否下一个 then 会有 onRejected 函数。

**结论：** 处理 onRejected，必须在所有 then 都执行完了之后。

回顾 then 的规范，里面提到一条 onFulfilled 和 onRejected 必须是异步的。上面的结论告诉我们，在 then 中不需要特殊处理 p1 是 rejected 的情况，将 onRejected 存起来，等待 p1 在执行完所有 then 之后再来判断要不要抛出错误。

那么我们还要不要在 then 中特殊处理 fulfilled 的情况？我们完全可以像处理 onRejected 一样，等待所有 then 执行完毕在来处理 Promise Chain 上的 onFulfilled。这样 then 的代码会变得非常简单。也没有我们前面所罗列出来的那么多情况。

到此为止，“onFulfilled 和 onRejected 必须是异步的。”这条规范变得非常合理。

then 只需要做一件事情：

```js
then(onFulfilled, onRejected) {
  //  这里总结下来应有五种情况，无论哪种情况，都需要返回新的promise
  const newPromise = new PromiseJ(() => {});
  if (isFunction(onFulfilled)) {
    newPromise.onFulfilled = onFulfilled;
  }
  if (isFunction(onRejected)) {
    newPromise.onRejected = onRejected;
  }
  this.nextPromiseQueue.push(newPromise);
  // 这里返回promise，让下一个then生成的promise挂到newPromise上
  return newPromise;
}
```

resolve 和 reject 中异步处理 nextPromiseQueue

```js
resolve(value) {
  // 只有pending状态可以转变为fulfilled
  if (this.state !== STATUS.PENDING) {
    return;
  }
  this.state = STATUS.FULFILLED;
  Object.defineProperty(this, "value", {
    value,
    writable: false
  });
  // _processOnfulfilled 遍历 nextPromiseQueue
  queueMicrotask(this._processOnfulfilled.bind(this));
}

reject(reason) {
  if (this.state !== STATUS.PENDING) {
    return;
  }
  this.state = STATUS.REJECTED;
  Object.defineProperty(this, "reason", {
    value: reason,
    writable: false
  });
  queueMicrotask(this._processOnRejected.bind(this));
}
```

## 完整代码

https://github.com/xiaojingzhao/j-promise

里面包括了 Promise A+ 测试。最新版本已全部通过测试。

### 遇到的问题

#### 问题 1：then 在异步中运行

这里出现了一个非常严重的问题，当我们看整份代码的时候，发现所有 then 中的回调 -- onFulfilled 和 onRejected 都在 resolve 之后执行了。到目前为止，我们觉得一切正常。但是来看一下下面的例子：

```js
const p = new PromiseJ(resolve => resolve("3333"));
setTimeout(() => p.then(value => console.log("value", value))); // 回调不执行
```

> 参考 tag v0.0.1 版本

Boom! 异步执行的 then 中的回调不执行了。思考一下，resolve 中异步处理了同步 then 中存下来的回调。但是如果 then 是异步的，那么这个时候 resolve 中的异步处理已经完成，所以异步 then 中的回调永远不会被触发。

这是为什么？当我们在 resolve 和 reject 中处理 promise chain 的时候，我们默认了一个前提条件：then 一定是同步执行的，或者说 then 一定发生在 \_processOnfulfilled 之前。

上面的例子说明，then 可能发生在异步的回调中 -- then 发生在 resolve 处理 onFulfilled 和 onRejected 之后。

那么又需要在 then 中判断当前 promise 状态了。这个时候，我们发现问题非常复杂，原因有以下几点：

##### 1. then 中无法判断这个 then 是异步触发的还是同步触发的

v0.01 版本的执行顺序是这样的：

```
resolve ---> change state and value  ---> invoke then multiple times synchronously ---> store onFulfilled and onRejected ---> invoke onFulfilled and onRejected travelling through all branches and chains
```

由此可以发现，如果是一个直接 resolve 的 promise，在执行第一个 then 的时候，promise 的状态已经改变，不管 then 是同步还是异步执行。

那么异步 then 中触发 onFulfilled 和 onRejected 的时机是什么？

**解决方案 1：** 如果是 fulfilled 或者 rejected 的 promise，不生成 promise chain 直接异步执行 onFulfilled 和 onRejected。

**带来的问题：** 无法处理 rejected promise 如果后面没有 onRejected 回调就报错的情况。回忆一下之前我们提到的关于为什么 reject 一个 promise 的时候要等所有的 then 执行完了再处理的原因。

**解决方案 2：** 接着方案 1，如果是 fulfilled 或者 rejected 的 promise，先储存 onFulfilled 和 onRejected，然后触发一个处理和 resolve 方法中一样的异步处理所有 onFulfilled 和 onRejected 的方法。

```js
queueMicrotask(this._processOnfulfilled.bind(this));
```

**带来的问题：** 多次调用 then 的话，会触发多次异步处理，但实际上多个同步的 then，只需要一次即可。例如下面的例子，只需触发两次异步处理方法，但实际上会触发四次。

```js
const p = new PromiseJ(resolve => resolve("3333")).then().then(); // 触发一次
setTimeout(() => p.then(value => console.log("value", value))); // 触发第二次
```

**解决方案 3：** 接着方案 2，多次触发，但只执行一次，很容易让人联想到函数防抖。利用函数防抖可以很好的解决上面的问题。

```js
then(onFulfilled, onRejected) {
    const newPromise = new PromiseJ(() => {});
    if (isFunction(onFulfilled)) {
      newPromise.onFulfilled = onFulfilled;
      if (this.state === STATUS.FULFILLED) {
        fulfilledTimer = clearTimer(fulfilledTimer);
        fulfilledTimer = setTimeout(this._processOnfulfilled.bind(this));
      }
    }
    if (isFunction(onRejected)) {
      newPromise.onRejected = onRejected;
      if (
        this.state === STATUS.REJECTED // TODO: 应该设置一个标记
      ) {
        rejectedTimer = clearTimeout(rejectedTimer);
        rejectedTimer = setTimeout(this._processOnRejected.bind(this));
      }
    }
    this.nextPromiseQueue.push(newPromise);
    // 这里返回promise，让下一个then生成的promise挂到newPromise上
    return newPromise;
  }
```

**带来的问题：** 会多触发一次异步处理 \_processOnfulfilled

```js
const p = new PromiseJ(resolve => resolve("3333")) // resolve 中 触发一次_processOnfulfilled
  .then()
  .then(); // 中异步回调触发一次
```

目前我们算是解决了异步 then 的问题。

#### 问题 2：reject 在 promises-aplus-tests 中不会报错。

这个问题直接影响了代码实现了思路。在设计 reject 的时候，正因为在浏览器中直接 reject 会抛出错误，所以才设计了先 reject 改变状态，后 then 同步执行，最后遍历 promise chain 查找是否有 onRejected 回调来确认是否要抛出错误这个流程。如果 reject 在 Promise A+规范中不抛错的话，那么我们可以换一个简单一点的实现方式。

- reject 时，不需要判断之后是否有 onRejected 回调，reject 不报错。
- 不用等所有的 then 都执行完再批量处理 onFulfilled 和 onReject。

问题 1 中可以采用第一种解决方案，不需要生成 promise chain。直接异步单个处理当前的 onFulfilled 和 onRejected。

具体代码参考 v0.0.3

#### 问题 3：一个 immediately resolve 的 promsie，如果直接 resolve 一个 promise2？

之前的实现，我们发现在 resolve 中，我们直接将 x 赋值给了 promise 的 value，这实际上是不正确的。如果 x 是一个 promise 呢？

其实 promise1.resolve(value)就是一个[[Resolution]](promise1, value)。

将代码修改为：

```js
  resolve(value) {
    // 只有pending状态可以转变为fullfilled
    if (this.state !== STATUS.PENDING) {
      return;
    }
    this._resolution(this, value);
    return this;
  }
```

以上三个是测试中遇到的比较严重的问题。目前 promise 大功告成~

## Future Work

跑完测试后，发现耗时大约 14s。这里有一份全部通过测试的 promise -- [my-promise](https://github.com/hax/my-promise)，作者是 Hax 耗时只需 8s。

后续可以看一下时间差距在哪里？
