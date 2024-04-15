---
title: RxJS Observables 单元测试 – 实用指南
date: 2024-04-15 13:34:19
tags: Rxjs Vitest
---

由于 RxJS 观察链的异步特性，对其进行单元测试可能具有挑战性。我们最新的技术博客着眼于简化这个过程。

目录表

* RxJS 可观察对象单元测试中的挑战
* 技术的先决条件
* 单元测试目标
* RxJS 观察链单元测试: 解决方案概述
* 单元测试 RxJS 观察链: 解决方案示例
* RxJS 观察链单元测试: 解决方案概述
* 进一步的资源
* 附录 A: 考虑单元测试 RxJS 观察链的替代解决方案

## RxJS 可观察对象单元测试中的挑战

Adaptive 为其资本市场客户构建的用户界面(ui)通常是“反应性的”，这意味着它们的视图会随着金融市场的变化而实时更新。在 Adaptive，我们倾向于使用 RxJS 的可观察对象来管理应用程序状态，将它们绑定到 UI 组件上，这样当可观察对象发出时，视图就会更新。在应用程序的状态层中，我们经常通过将操作符函数传递给 pipe()方法来定义重要的观察链。这些观察链定义了如何组合来自多个数据源的数据，并将其转换为 UI 组件要使用的视图模型对象。

这些观察链中逻辑的正确性通常对应用程序的行为至关重要，**但单元测试可能具有挑战性，因为随着时间的推移，可观察对象会异步发出多个值。**

那么，我们如何才能有效地对 RxJS 观察链进行单元测试，以验证它们的行为是否符合我们的预期呢?

## 技术的先决条件

假定读者熟悉 TypeScript、RxJS 和单元测试框架(如 Vitest 或 Jest)。本文将不关注任何特定的 UI 框架或视图/组件库。如果你正在使用 RxJS 以响应式的方式管理应用状态，那么无论你使用的是 React、Vue.js、Angular 还是其他框架，这些内容都应该是相关的。

本文中的示例代码可以在随附的 GitHub repo 中找到。在本文中，您可以使用 repo 中的标记跟踪示例的开发进度。

## 单元测试目标

Adaptive 团队希望能够测试由观察链创建的可观察对象发出的各种状态排列，并通过它所依赖的上游源可观察对象向所述链提供各种输入。

我们想要测试的单元是按原样导出的有状态可观察对象，或者是接受键(比如实体 ID)并返回可观察对象的函数。例如，下面的观察链，本文将演示如何进行单元测试。

Stage 1: state.ts

```javascript
import {
    Observable,
    map,
    mergeWith,
    scan,
    startWith
} from "rxjs";
import {
    pricesDto$,
    resetPrices$
} from "./service";
export const prices$: Observable < Record < string, number >> = pricesDto$.pipe(
    scan(
        (accum, current) => ({
            ...accum,
            [current.symbol]: current.price,
        }), {}
    ),
    mergeWith(resetPrices$.pipe(map(() => ({})))),
    startWith({})
);
```

在开发我们在这里展示的模式之前，我们探索并拒绝了附录 A: 为 RxJS 观察链单元测试考虑的替代解决方案中描述的一些方法。

## RxJS 观察链单元测试: 解决方案概述

正如我们将在下面演示的那样，我们通过使用 spy 函数订阅被测对象来实现我们的测试目标，这样我们就可以断言所发出的内容，并用 RxJS 主题模拟源可观察对象。我们通过一个非常小的实用函数 spyOnObservable()来实现这一点，它抽象了与监视观察者相关的样板文件。我们在这里使用的是 Vitest，但如果您使用的是 Jest，则语义非常相似。如果您使用不同的测试库(如 Mocha、Jasmine 或 Qunit)，语义可能会有点不同，但概念和功能应该很容易转移。

spyOnObservable.ts

```javascript
import {
    Observable,
    Subscription
} from 'rxjs'
import {
    Mock
} from 'vitest'
/**

- Utility function for testing observables.
- Returns an object containing mock observer functions.
-
- To ensure your test does not cause a memory leak, assert that `complete`
- has been called; this will verify that this utility has unsubscribed
- from the observable under test. Alternatively, explicitly unsubscribe
- the subscription that is returned.
-
- Example usage:
-
- const { next, error, complete, subscription, latestEmission, emissionCount } =
-      spyOnObservable(observableToTest$.pipe(take(1)))
-
- expect(next).toHaveBeenCalledTimes(1)
- expect(next).toHaveBeenCalledWith(someValue)
- expect(latestEmission()).toBe(someValue)
- expect(error).not.toHaveBeenCalled()
- subscription.unsubscribe()
- expect(complete).toHaveBeenCalled()
  */
export function spyOnObservable(observable$: Observable < unknown > ) {
    const next: Mock < any, any > = vi.fn()
    const error: Mock < any, any > = vi.fn()
    const complete: Mock < any, any > = vi.fn()
    const emissionCount = () => next.mock.calls.length
    const latestEmission = () => {
        try {
            return next.mock.calls.at(-1) ![0]
        } catch (e) {
            throw new Error('expected next to have been called')
        }
    }
    let subscription: Subscription
    subscription = observable$.subscribe({
        next,
        error,
        complete: () => {
            subscription?.unsubscribe()
            complete()
        },
    })
    return {
        next,
        error,
        complete,
        subscription,
        latestEmission,
        emissionCount
    }
}
```

## 单元测试 RxJS 观察链: 解决方案示例

我们将使用以下虚构的用例来演示如何在现实生活中应用它。我们选择了一个用例，它比我们在现实生活中遇到的用例更简单，但具有足够的复杂性来说明问题以及我们将如何解决它。

假设我们有一个需要显示实时更新的金融工具价格的应用程序。为了做到这一点，我们需要一个从符号到价格的查找表，每个我们有价格的工具都有一个条目。我们有一个可观察对象，它将在每次价格更新时发出一个对象，其中包含该表的最新值。这是我们想要测试的可观察对象。

**被测试的可观察对象依赖于两个源可观察对象:**

* 第一个将发出一个对象，其中包含一个符号和价格的单个工具，每次有一个价格更新的工具。(在现实世界的应用中，这个可观察对象将是 WebSocket 流上的抽象。)
* 第二个表示一个事件，当触发时，应该将价格查找表的状态重置为其初始空状态。

这个可观察对象的初始实现如下所示。然而，在没有测试的情况下，我们不能确定它的行为是否符合预期。

Stage 1: state.ts

```javascript
import {
    Observable,
    map,
    mergeWith,
    scan,
    startWith
} from "rxjs";
import {
    pricesDto$,
    resetPrices$
} from "./service";
export const prices$: Observable < Record < string, number >> = pricesDto$.pipe(
    scan(
        (accum, current) => ({
            ...accum,
            [current.symbol]: current.price,
        }), {}
    ),
    mergeWith(resetPrices$.pipe(map(() => ({})))),
    startWith({})
);
```

有关其他上下文，请参见 service。t 和 model。Ts

让我们设置测试文件。

Stage 1: state.test.ts

```javascript
import {
    Subject
} from 'rxjs'
import {
    Price
} from './model'
// create subjects to mock out the source observables that our
// observable under test depends on
const mockPricesDto$ = new Subject < Price > ()
const mockResetPrices$ = new Subject < void > ()
// use doMock() rather than mock() so we can reference the
// variables containing the mock observables. mock() is hoisted
// so it does not allow referencing variables in the file scope.
// see https://vitest.dev/api/vi#vi-mock and
// https://vitest.dev/api/vi#vi-domock
vi.doMock('./service', () => ({
    pricesDto$: mockPricesDto$,
    resetPrices$: mockResetPrices$,
}))
// we need to dynamically import the observable under test
// after we call vi.doMock. see https://vitest.dev/api/vi#vi-domock
const {
    prices$
} = await import('./state')
describe('prices$', () => {
    // we are now ready to add our tests here
})
```

现在我们已经设置了测试文件的外壳，让我们添加一些基本断言。注意我们的 spyOnObservable()实用函数的用法。

Stage 2: state.test.ts

```javascript
import {
    Subject
} from 'rxjs'
import {
    Price
} from './model'
import {
    spyOnObservable
} from './spyOnObservable'
// create subjects to mock out the source observables that our
// observable under test depends on
const mockPricesDto$ = new Subject < Price > ()
const mockResetPrices$ = new Subject < void > ()
// use doMock() rather than mock() so we can reference the
// variables containing the mock observables. mock() is hoisted
// so it does not allow referencing variables in the file scope.
// see https://vitest.dev/api/vi#vi-mock and
// https://vitest.dev/api/vi#vi-domock
vi.doMock('./service', () => ({
    pricesDto$: mockPricesDto$,
    resetPrices$: mockResetPrices$,
}))
// we need to dynamically import the observable under test
// after we call vi.doMock. see https://vitest.dev/api/vi#vi-domock
const {
    prices$
} = await import('./state')
describe('prices$', () => {
    // spy on the observable under test, using the spyOnObservable utility
    const {
        latestEmission,
        error,
        subscription
    } = spyOnObservable(prices$)
    // ensure we unsubscribe when we are done to avoid memory leaks
    afterAll(() => {
        subscription.unsubscribe()
    })
    it('should initially emit empty object', () => {
        expect(latestEmission()).toEqual({})
    })
    it('should not error', () => {
        expect(error).not.toBeCalled()
    })
})
```

我们的测试通过了。

![alt text](image.png)

RxJS 可观察对象: 第二阶段——测试通过

到目前为止一切顺利。现在，让我们添加一些断言，说明源可观察对象 pricesDto$发出时的行为。

Stage 3: state.test.ts

```javascript
import {
    Subject
} from 'rxjs'
import {
    Price
} from './model'
import {
    spyOnObservable
} from './spyOnObservable'
// create subjects to mock out the source observables that our
// observable under test depends on
const mockPricesDto$ = new Subject < Price > ()
const mockResetPrices$ = new Subject < void > ()
// use doMock() rather than mock() so we can reference the
// variables containing the mock observables. mock() is hoisted
// so it does not allow referencing variables in the file scope.
// see https://vitest.dev/api/vi#vi-mock and
// https://vitest.dev/api/vi#vi-domock
vi.doMock('./service', () => ({
    pricesDto$: mockPricesDto$,
    resetPrices$: mockResetPrices$,
}))
// we need to dynamically import the observable under test
// after we call vi.doMock. see https://vitest.dev/api/vi#vi-domock
const {
    prices$
} = await import('./state')
describe('prices$', () => {
    // spy on the observable under test, using the spyOnObservable utility
    const {
        latestEmission,
        error,
        subscription
    } = spyOnObservable(prices$)
    // ensure we unsubscribe when we are done to avoid memory leaks
    afterAll(() => {
        subscription.unsubscribe()
    })
    it('should initially emit empty object', () => {
        expect(latestEmission()).toEqual({})
    })
    it('should emit object containing latest prices after pricesDto$ emits', () => {
        // call next() on the subject that mocks out the source observable
        // priceDto$ that the observable under test depends on to simulate
        // that observable emitting prices, and ensure the new price is
        // emitted as expected in the observable under test
        mockPricesDto$.next({
            symbol: 'XOM',
            price: 48.17
        })
        expect(latestEmission()).toEqual({
            XOM: 48.17
        })
        // add another instrument/price, ensure both instruments
        // appear in the resulting emission
        mockPricesDto$.next({
            symbol: 'BA',
            price: 218.93
        })
        expect(latestEmission()).toEqual({
            XOM: 48.17,
            BA: 218.93
        })
        // update the price of the first instrument, ensure the price is
        // updated in the resulting emission
        mockPricesDto$.next({
            symbol: 'XOM',
            price: 48.21
        })
        expect(latestEmission()).toEqual({
            XOM: 48.21,
            BA: 218.93
        })
    })
    it('should not error', () => {
        expect(error).not.toBeCalled()
    })
})
```

太好了，我们的测试仍然通过了

![alt text](image-1.png)

RxJS 可观察对象: 阶段 3——测试通过

现在让我们为重置事件添加一个断言。

Stage 4: state.test.ts

```javascript
import {
    Subject
} from 'rxjs'
import {
    Price
} from './model'
import {
    spyOnObservable
} from './spyOnObservable'
// create subjects to mock out the source observables that our
// observable under test depends on
const mockPricesDto$ = new Subject < Price > ()
const mockResetPrices$ = new Subject < void > ()
// use doMock() rather than mock() so we can reference the
// variables containing the mock observables. mock() is hoisted
// so it does not allow referencing variables in the file scope.
// see https://vitest.dev/api/vi#vi-mock and
// https://vitest.dev/api/vi#vi-domock
vi.doMock('./service', () => ({
    pricesDto$: mockPricesDto$,
    resetPrices$: mockResetPrices$,
}))
// we need to dynamically import the observable under test
// after we call vi.doMock. see https://vitest.dev/api/vi#vi-domock
const {
    prices$
} = await import('./state')
describe('prices$', () => {
    // spy on the observable under test, using the spyOnObservable utility
    const {
        latestEmission,
        error,
        subscription
    } = spyOnObservable(prices$)
    // ensure we unsubscribe when we are done to avoid memory leaks
    afterAll(() => {
        subscription.unsubscribe()
    })
    it('should initially emit empty object', () => {
        expect(latestEmission()).toEqual({})
    })
    it('should emit object containing latest prices after pricesDto$ emits', () => {
        // call next() on the subject that mocks out the source observable
        // priceDto$ that the observable under test depends on, to simulate
        // that observable emitting prices, and ensure the new price is
        // emitted as expected in the observable under test
        mockPricesDto$.next({
            symbol: 'XOM',
            price: 48.17
        })
        expect(latestEmission()).toEqual({
            XOM: 48.17
        })
        // add another instrument/price, ensure both instruments
        // appear in the resulting emission
        mockPricesDto$.next({
            symbol: 'BA',
            price: 218.93
        })
        expect(latestEmission()).toEqual({
            XOM: 48.17,
            BA: 218.93
        })
        // update the price of the first instrument, ensure the price is
        // updated in the resulting emission
        mockPricesDto$.next({
            symbol: 'XOM',
            price: 48.21
        })
        expect(latestEmission()).toEqual({
            XOM: 48.21,
            BA: 218.93
        })
    })
    it('should emit empty object after resetPrices$ emits', () => {
        // call next() on the subject that mocks out the source observable
        // resetPrices$ that the observable under test depends on, to simulate
        // that observable emitting, and ensure that the prices lookup table
        // is reset to an empty object
        mockResetPrices$.next()
        expect(latestEmission()).toEqual({})
    })
    it('should not error', () => {
        expect(error).not.toBeCalled()
    })
})
```

太好了，测试又通过了。

![alt text](image-2.png)

RxJS 可观察对象: 阶段 4 -测试通过

现在，让我们添加一个断言，即在重置后流入的下一个价格中，查找表中只有新价格。
Stage 5: state.test.ts

```javascript
import {
    Subject
} from 'rxjs'
import {
    Price
} from './model'
import {
    spyOnObservable
} from './spyOnObservable'
// create subjects to mock out the source observables that our
// observable under test depends on
const mockPricesDto$ = new Subject < Price > ()
const mockResetPrices$ = new Subject < void > ()
// use doMock() rather than mock() so we can reference the
// variables containing the mock observables. mock() is hoisted
// so it does not allow referencing variables in the file scope.
// see https://vitest.dev/api/vi#vi-mock and
// https://vitest.dev/api/vi#vi-domock
vi.doMock('./service', () => ({
    pricesDto$: mockPricesDto$,
    resetPrices$: mockResetPrices$,
}))
// we need to dynamically import the observable under test
// after we call vi.doMock. see https://vitest.dev/api/vi#vi-domock
const {
    prices$
} = await import('./state')
describe('prices$', () => {
    // spy on the observable under test, using the spyOnObservable utility
    const {
        latestEmission,
        error,
        subscription
    } = spyOnObservable(prices$)
    // ensure we unsubscribe when we are done to avoid memory leaks
    afterAll(() => {
        subscription.unsubscribe()
    })
    it('should initially emit empty object', () => {
        expect(latestEmission()).toEqual({})
    })
    it('should emit object containing latest prices after pricesDto$ emits', () => {
        // call next() on the subject that mocks out the source observable
        // priceDto$ that the observable under test depends on, to simulate
        // that observable emitting prices, and ensure the new price is
        // emitted as expected in the observable under test
        mockPricesDto$.next({
            symbol: 'XOM',
            price: 48.17
        })
        expect(latestEmission()).toEqual({
            XOM: 48.17
        })
        // add another instrument/price, ensure both instruments
        // appear in the resulting emission
        mockPricesDto$.next({
            symbol: 'BA',
            price: 218.93
        })
        expect(latestEmission()).toEqual({
            XOM: 48.17,
            BA: 218.93
        })
        // update the price of the first instrument, ensure the price is
        // updated in the resulting emission
        mockPricesDto$.next({
            symbol: 'XOM',
            price: 48.21
        })
        expect(latestEmission()).toEqual({
            XOM: 48.21,
            BA: 218.93
        })
    })
    it('should emit empty object after resetPrices$ emits', () => {
        // call next() on the subject that mocks out the source observable
        // resetPrices$ that the observable under test depends on, to simulate
        // that observable emitting, and ensure that the prices lookup table
        // is reset to an empty object
        mockResetPrices$.next()
        expect(latestEmission()).toEqual({})
    })
    it('should emit object containing only the latest prices after pricesDto$ emits', () => {
        mockPricesDto$.next({
            symbol: 'HD',
            price: 332.12
        })
        mockPricesDto$.next({
            symbol: 'AA',
            price: 24.49
        })
        expect(latestEmission()).toEqual({
            HD: 332.12,
            AA: 24.49
        })
    })
    it('should not error', () => {
        expect(error).not.toBeCalled()
    })
})
```

…我们的测试不通过

![alt text](image-3.png)

RxJS 可观察对象: 第 5 阶段——测试失败

看来我们的测试发现了实现中的一个 bug ! 我们希望在 reset 可观察对象发出时将状态重置为空对象。但是我们传递给 scan()的 reducer 函数中的 accumulator 对象即使在重置之后也会保留状态。让我们通过在 scan()之前移动 mergeWith()来解决这个问题，并在 reducer 函数中添加一个条件，以便在重置事件中用空对象替换状态。(在这种情况下，流入 scan()的值将是 false 而不是 Price 对象。)

Stage 6: state.ts

```javascript
import {
    Observable,
    mergeWith,
    scan,
    startWith
} from "rxjs";
import {
    pricesDto$,
    resetPrices$
} from "./service";
import {
    Price
} from "./model";
export const prices$: Observable < Record < string, number >> = pricesDto$.pipe(
    mergeWith(resetPrices$),
    scan(
        (accum, current: Price | void) =>
        !current ?
        {} :
        {
            ...accum,
            [current.symbol]: current.price,
        }, {}
    ),
    startWith({})
);
```

我们的测试现在又通过了，这给了我们信心，我们的观察链做了我们想要它做的事情。

![alt text](image-4.png)

## RxJS 观察链单元测试: 解决方案概述

spyOnObservable 实用程序提供了一个易于使用的 API 来监视可观察对象，以方便断言它们的行为。这个实用程序与 RxJS Subjects 模拟上游可观察对象配对，定义了一个可扩展的模式，用于编写语法愉悦的单元测试，模拟观察链在现实世界中的行为，通过上游源可观察对象提供各种输入。

你可以随意复制 spyOnObservable，根据你认为合适的需要进行调整，并使用它而不注明出处。

## 进一步的资源

如果您有兴趣了解更多关于如何实现响应式编程原则的信息，请参阅 reactive Trader® 及其开源代码库。Reactive Trader® 是 Adaptive 的实时外汇(FX)和信用交易平台，旨在展示整个应用程序堆栈的反应性编程原则。

如果你想了解 React-RxJS，一个提供绑定来将 RxJS 可观察对象与 React 集成的库，请参阅我们 2020 年的文章 Why React RxJS。

## 附录 A: 考虑单元测试 RxJS 观察链的替代解决方案

Marble 测试非常适合测试 RxJS 操作符(如 map()、first()等)。例如，创建操作符函数的工厂函数。事实上，RxJS 代码库中 RxJS 操作符的单元测试使用大理石测试。当测试一个接受一个可观察对象并返回另一个可观察对象的函数时，大理石测试也很有用。

然而，对于我们测试有状态观察链的用例，大理石测试似乎并不适合。此外，大理石测试对整个产生的排放流起作用。我们希望能够以交互的方式测试游戏中排放的内容，这样我们就可以在每次排放后通过可观测源验证预期结果。

我们简要探讨的另一种方法是订阅可观察对象，并在观察者中进行测试断言(例如，将下一个处理程序传递给 subscribe())。这对于发出多个值的可观察对象来说效果很差，它只在你知道可观察对象将完成时才有效，并且需要样板文件来处理在可观察对象完成时实现异步测试的承诺。在某种程度上，您可以通过使用 take()等操作符强制完成(但是您需要确切地知道要关注多少次释放)，以及 toArray()来检查释放的多个值，从而解决这个问题。我们发现这种方法并不令人满意。

Author: Bruce Harris

[原文链接：https://weareadaptive.com/2024/01/09/unit-testing-rxjs-observables-a-practical-guide/](https://weareadaptive.com/2024/01/09/unit-testing-rxjs-observables-a-practical-guide/)
