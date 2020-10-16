# React Fiber Architecture

## 介绍

React Fiber 是 React 核心算法的一个正在进行的再实现。这是React团队两年多研究的成果。

React Fiber的目标是增加 React 对动画、布局和手势等领域的适应性。它的主要特性是**增量渲染**:能够将渲染工作分割成块，并将其分散到多个帧上。

其他关键特性包括在新的更新到来时暂停、中止或重用工作的能力;能够分配优先级的不同类型的更新;和新的并发原语（concurrency primitives）。

### 关于此文档

Fiber 引入了几个新的概念，仅通过查看代码是很难理解的。这个文档是我在React项目中执行Fiber时所做的笔记的集合。随着它的成长，我意识到它对其他人也可能是一个有用的资源。

我将尝试使用最简单的语言，并通过使用简要说明来避免术语。如果可能的话，我还会与外部资源紧密联系。

请注意，我不是 React 小组的成员，也不代表任何权威。这不是官方文件。我已经要求React团队的成员对其进行审核，以确保准确性。

这也是一项正在进行的工作。**Fiber 是一个正在进行的项目，在完成之前可能会经历重大的重构。**还在进行的是我在这里记录其设计的尝试。我们非常欢迎改进和建议。

我的目标是，在阅读完本文后，您将充分理解 Fiber (https://github.com/facebook/react/commits/master/src/renderers/shared/fiber)，并最终能够反馈给 React 。

### 预备知识

在理解 Fiber 之前，强烈建议掌握下面概念：

- 增量渲染：将渲染工作分成多个块并将其分布到多个帧中的能力。
- [React 组件、元素和实例](https://zh-hans.reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html)
- [协调](https://zh-hans.reactjs.org/docs/reconciliation.html)
- [React 基础理念](https://github.com/reactjs/react-basic)
- [React 设计原则](https://zh-hans.reactjs.org/docs/design-principles.html)，特别注意调度部分，它很好的解释了 React Fiber 产生的原因

### 什么是 reconciliation?

<dl>
  <dt>reconciliation</dt>
  <dd>该算法利用对比两个树之间的不同来确定哪些部分需要更改</dd>

  <dt>update</dt>
  <dd>用于呈现 React 应用程序的数据变化。通常是“setState”导致最终的重新渲染。</dd>
</dl>

React API的核心思想是，认为是更新导致的整个应用程序重新渲染。这允许开发人员声明式地推理，而不用担心如何有效地将应用程序从任何特定状态转换到另一种状态(A到B, B到C, C到A，等等)。

实际上，在每次更改时重新呈现整个应用程序只适用于小的应用程序;在一个真实的应用程序中，它的性能代价非常高。React进行了优化，在保持良好性能的同时创建了整个应用程序重新呈现的样子。这些优化的大部分是一个称为**协调（reconciliation）**的过程的一部分。


协调（Reconciliation） 是人们普遍理解的“虚拟DOM”背后的算法。高级描述大致是这样的:当您呈现一个React应用程序时，将生成描述该应用程序的节点树并保存在内存中。然后，这个树被刷新到渲染环境中——例如，对于浏览器应用程序，它被转换为一组DOM操作。当应用程序被更新时(通常通过' setState ')，一个新的树被生成。新的树与之前的树不同，可以计算需要哪些操作来更新呈现的应用程序。

尽管 Fiber 是对 reconciler 的重写，但高级算法[在React文档中描述](https://facebook.github.io/react/docs/reconciliation.html)将基本相同。重点是:

- 假设不同的组件类型会生成完全不同的树。React不会试图去区别它们，而是完全取代老树
- 列表的不同是使用 key 来执行的。key 应该是“稳定的、可预测且唯一的”。


### Reconciliation versus rendering

DOM 只是 React 可以渲染的环境之一，其他主要目标是通过 React native 实现的本地 iOS 和 Android 视图。(这就是为什么“虚拟DOM”有点用词不当的原因。)

它能够支持这么多目标的原因是因为 React 被设计成协调和渲染是分开的阶段。“协调器”负责计算树的哪些部分发生了变化;然后，渲染器使用这些信息来实际更新渲染的应用程序。

这种分离意味着 React DOM 和 React Native 可以使用它们自己的渲染器，同时共享 React core 提供的相同协调器。

Fiber 重新实现了协调。它主要与渲染无关，尽管渲染器需要进行更改以支持(并利用)新的体系结构。

### Scheduling

<dl>
  <dt>scheduling</dt>
  <dd>决定何时执行 work 的过程。</dd>

  <dt>work</dt>
  <dd>必须执行的计算。Work 通常是更新的结果(例如<code>setState)(e.g. <code>setState</code>).
  
</dl>

React's [Design Principles](https://facebook.github.io/react/contributing/design-principles.html#scheduling) document is so good on this subject that I'll just quote it here:

> In its current implementation React walks the tree recursively and calls render functions of the whole updated tree during a single tick. However in the future it might start delaying some updates to avoid dropping frames.
>
> This is a common theme in React design. Some popular libraries implement the "push" approach where computations are performed when the new data is available. React, however, sticks to the "pull" approach where computations can be delayed until necessary.
>
> React is not a generic data processing library. It is a library for building user interfaces. We think that it is uniquely positioned in an app to know which computations are relevant right now and which are not.
>
> If something is offscreen, we can delay any logic related to it. If data is arriving faster than the frame rate, we can coalesce and batch updates. We can prioritize work coming from user interactions (such as an animation caused by a button click) over less important background work (such as rendering new content just loaded from the network) to avoid dropping frames.

The key points are:

- In a UI, it's not necessary for every update to be applied immediately; in fact, doing so can be wasteful, causing frames to drop and degrading the user experience.
- Different types of updates have different priorities — an animation update needs to complete more quickly than, say, an update from a data store.
- A push-based approach requires the app (you, the programmer) to decide how to schedule work. A pull-based approach allows the framework (React) to be smart and make those decisions for you.

React doesn't currently take advantage of scheduling in a significant way; an update results in the entire subtree being re-rendered immediately. Overhauling React's core algorithm to take advantage of scheduling is the driving idea behind Fiber.

---

Now we're ready to dive into Fiber's implementation. The next section is more technical than what we've discussed so far. Please make sure you're comfortable with the previous material before moving on.

## 什么是 fiber?

我们将讨论React Fiber 的核心架构。Fiber 是一种比应用程序开发人员通常认为的低得多的抽象。如果你发现自己在试图理解它的过程中受挫，不要感到气馁。继续尝试，最终会有意义的。(当你最终明白了，请建议如何改进这部分。)

我们开始吧！

---
我们已经确定，Fiber 的一个主要目标是使 React 能够利用调度优势。具体来说，我们需要能够：

- 暂停 Work，稍后再返回。
- 为不同类型的 work 分配优先级。
- 重复使用以前完成的 work。
- 如果不再需要，则停止 work。


为了做到这一点，我们首先需要一种将 work 分解成单元的方法。在某种意义上，这就是 Fiber。一个 Fiber 表示一个**工作单元**。

更进一步，让我们回到[React 组件作为数据的函数]的概念(React components as functions of data](https://github.com/reactjs/react-basic#transformation)，通常表示为

```
v = f(d)
```
因此，渲染 React 应用程序类似于调用一个函数，该函数的主体包含对其他函数的调用，等等。在考虑 Fiber时，这个类比很有用.

计算机跟踪程序执行的典型方式是使用[调用堆栈](https://en.wikipedia.org/wiki/Call_stack)。当一个函数被执行时，一个新的**堆栈帧**被添加到堆栈中。该堆栈帧表示由该函数执行的 work。

在处理 UI 时，如果一次执行了太多 work，可能会导致动画帧卡顿。另外，一些 work 可能是不必要的，但它会被最先执行。这就是UI组件和函数之间的比较的失败之处，因为组件比一般函数有更具体的关注点。

新的浏览器(和React Native)实现了帮助解决这个问题的api: `requestIdleCallback` 调度一个低优先级函数在空闲期间被调用，而`requestAnimationFrame`调度一个高优先级函数在下一个动画帧被调用。问题是，为了使用这些api，您需要一种将 render work 分解为增量单元的方法。如果您只依赖于调用堆栈，它也将继续工作，直到堆栈为空。

如果我们可以自定义调用堆栈的行为来优化ui的呈现，并且我们可以随意中断并手动操作堆栈帧，岂不是更好？

这就是 React Fiber 的作用。Fiber 是堆栈的重新实现，专门用于 React 组件。您可以将单个 fiber 看作一个**虚拟堆栈帧**。

重新实现堆栈的好处是，您可以[在内存中保存堆栈帧](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)并根据需要执行它们(*无论何时*)。这对于完成我们的调度目标是至关重要的。

除了调度之外，手动处理堆栈帧可以开启并发和错误边界等潜在特性。我们将在以后的章节中讨论这些主题。

在下一节中，我们将进一步了解 Fiber 的结构。

### Fiber 的结构

*注:随着我们对实现细节的了解越来越详细，一些事情会发生变化的可能性也会增加。如发现任何错误或资料已过时，请提交申请*

具体地说，光纤是一个JavaScript对象，它包含关于组件、输入和输出的信息。

光纤对应于堆栈帧，它也对应组件的实例。

以下是一些属于 fiber 的重要字段。(这个列表并不详尽。)

#### `type` and `key`

fiber 的 type 和 key 与 React 元素（elements）的作用是一样的。(实际上，当从一个元素创建一个fiber时，这两个字段会被直接复制。)
fiber 的 type 描述了它所对应的组件。对于复合组件，type 是函数或类组件本身。对于宿主元素(' div '， ' span '等)，type 是一个字符串。

从概念上讲，type 是堆栈帧跟踪其执行的函数(如' v = f(d) ')。

key 是在协调期间使用，以确定 fiber 是否可以重复使用。

#### `child` and `sibling`

这些字段指向其他 fiber，描述 fiber 的递归树结构。

child fiber 对应于组件的`render`方法返回的值。在下面的例子中

```js
function Parent() {
  return <Child />
}
```

`Parent`的 child fiber 对应 `Child`。

sibling 字段说明`render`返回多个子项的情况（Fiber中的一项新功能！）：

```js
function Parent() {
  return [<Child1 />, <Child2 />]
}
```
child fiber 形成一个单链列表，其头是第一个子链。因此，在此示例中，`Parent`的子级为`Child1`，而`Child1`的兄弟级为`Child2`。
回到我们的功能类比，您可以将 child fiber 视为[tail-called function](https://en.wikipedia.org/wiki/Tail_call)。

#### `return`

return fiber 是程序在处理完当前 fiber 之后应返回的 fiber。从概念上讲，它与堆栈帧的返回地址相同。也可以将其视为 parent fiber。

If a fiber has multiple child fibers, each child fiber's return fiber is the parent. So in our example in the previous section, the return fiber of `Child1` and `Child2` is `Parent`.

如果 fiber 具有多个 child fiber，则每个 child fiber 的 return fiber 都是 parent fiber。因此，在上一节的示例中，`Child1`和`Child2`的 return fiber 为`Parent`。

#### `pendingProps` and `memoizedProps`

Conceptually, props are the arguments of a function. A fiber's `pendingProps` are set at the beginning of its execution, and `memoizedProps` are set at the end.

When the incoming `pendingProps` are equal to `memoizedProps`, it signals that the fiber's previous output can be reused, preventing unnecessary work.

#### `pendingWorkPriority`

A number indicating the priority of the work represented by the fiber. The [ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) module lists the different priority levels and what they represent.

With the exception of `NoWork`, which is 0, a larger number indicates a lower priority. For example, you could use the following function to check if a fiber's priority is at least as high as the given level:

```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

*This function is for illustration only; it's not actually part of the React Fiber codebase.*

The scheduler uses the priority field to search for the next unit of work to perform. This algorithm will be discussed in a future section.

#### `alternate`

<dl>
  <dt>flush</dt>
  <dd>To flush a fiber is to render its output onto the screen.</dd>

  <dt>work-in-progress</dt>
  <dd>A fiber that has not yet completed; conceptually, a stack frame which has not yet returned.</dd>
</dl>

At any time, a component instance has at most two fibers that correspond to it: the current, flushed fiber, and the work-in-progress fiber.

The alternate of the current fiber is the work-in-progress, and the alternate of the work-in-progress is the current fiber.

A fiber's alternate is created lazily using a function called `cloneFiber`. Rather than always creating a new object, `cloneFiber` will attempt to reuse the fiber's alternate if it exists, minimizing allocations.

You should think of the `alternate` field as an implementation detail, but it pops up often enough in the codebase that it's valuable to discuss it here.

#### `output`

<dl>
  <dt>host component</dt>
  <dd>The leaf nodes of a React application. They are specific to the rendering environment (e.g., in a browser app, they are `div`, `span`, etc.). In JSX, they are denoted using lowercase tag names.</dd>
</dl>

Conceptually, the output of a fiber is the return value of a function.

Every fiber eventually has output, but output is created only at the leaf nodes by **host components**. The output is then transferred up the tree.

The output is what is eventually given to the renderer so that it can flush the changes to the rendering environment. It's the renderer's responsibility to define how the output is created and updated.

## Future sections

That's all there is for now, but this document is nowhere near complete. Future sections will describe the algorithms used throughout the lifecycle of an update. Topics to cover include:

- how the scheduler finds the next unit of work to perform.
- how priority is tracked and propagated through the fiber tree.
- how the scheduler knows when to pause and resume work.
- how work is flushed and marked as complete.
- how side-effects (such as lifecycle methods) work.
- what a coroutine is and how it can be used to implement features like context and layout.

## Related Videos
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)
