---
title: Sinon.js使用最佳实践【译文】
author: Neil Ning
date: 2024-08-10 22:56:52
tags: ['test', 'sinon.js']
categories: 翻译
cover: bg.jpeg
---

## 前言
该文章翻译自【Best Practices for Spies, Stubs and Mocks in Sinon.js】，[点击这里](https://semaphoreci.com/community/tutorials/best-practices-for-spies-stubs-and-mocks-in-sinon-js)查看原文。以下是原文内容。

包含Ajax、网络请求、定时器、数据库或其他依赖的代码通常很难测试。例如，代码中用到Ajax或者网络请求，你需要一个能够响应请求的服务器。用到数据库时，为了使测试通过你还需要一个包含测试数据的数据库。
所有的这些使得编写和运行测试变得困难，因为你需要做额外的准备工作来设置测试环境，以保证测试能够运行成功。
幸运的是，我们可以使用Sinon.js避免这些问题。利用它提供的功能，仅仅用几行代码就可以将上面的例子简化。
然而，Sinon可能会让初学者感到困惑。它提供了很多函数如`spy`、`stub`、`mock`等，很难决定什么时候该用哪个函数。这些函数甚至有一些陷阱，所以你需要知道怎么做才能避免一些常见问题。
这篇文章中，我会向你展示spy、stub、mock之间的区别，以及何时该使用哪个函数。然后提供一些最佳实践来避免一些常见的陷阱。
## 函数示例
为了便于理解，下面是一个演示函数
```
function setupNewUser(info, callback) {
  var user = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };

  try {
    Database.save(user, callback);
  }
  catch(err) {
    callback(err);
  }
}
```
这个函数包含两个参数——一个需要保存的对象和一个回调函数，函数从info对象中读取变量，并将它保存入数据库，为了演示，save操作不是一个具体的操作，它可能表示一个Ajax请求，或者在Node.js中，它可能直接操作了数据库。具体是什么不重要，仅把它想象成一种数据保存操作。

## Spy、Stub、Mock
Spy、Stub、Mock被统称为测试替身（test doubles，doubles意为替身），类似于电影中的特技替身做一些危险的工作，我们使用测试替身来替代那些难以测试的依赖，这会使得测试代码更容易编写。
## 何时需要测试替身
为了更好的理解何时需要测试替身，我们先要了解函数的类型，我们将函数分为两个类型：
- 无副作用的函数
- 有副作用的函数

无副作用的函数很简单，这种函数的结果仅仅和函数的参数有关，相同的参数总是返回相同的输出。有副作用的函数可以定义为有外部依赖的函数，比如一些状态对象、当前时间、数据库调用，或者其他包含状态的方法。这种函数的执行结果不仅仅受函数的参数影响，还受这些状态的影响。
以上面的示例函数为例，函数内部调用了两个函数`toLowerCase`和`Database.save`。前者无副作用，它的输出仅仅依赖输入的值。后一个则是一个有副作用的函数，前面说过它是一类数据保存操作。`Database.save`的执行结果会受其他行为的影响。
如果我们想测试`setupNewUser`，我们就需要测试替身替换`Database.save`，因为该函数会产生副作用。或者换句话说，当函数包含副作用时，我们需要测试替身。
除了包含副作用的函数之外，还有一些场景需要用到测试替身，一个常见的例子是函数执行了一些耗时的计算或操作，这些计算会拖慢测试用例运行速度。不过，测试替身主要用在有副作用函数中。
## 何时使用spy
就像它的名字，spy用来监视函数，获取函数的调用信息。例如spy可以告诉我们函数被调用了多少次，每次的调用参数是什么，返回值是什么，或者抛出了什么异常等等。
因此，spy常用来验证发生了某个事情。结合Sinon的断言函数，我们可以检查很多不同的运行结果。最常见的场景包含以下两种：
- 检查某个函数调用了多少次
- 检查函数的调用参数

利用`sinon.assert.callCount, sinon.assert.calledOnce, sinon.assert.notCalled`等函数可以检查函数被调用了多少次。下面的例子验证`save`函数被调用：
```
it('should call save once', function() {
  var save = sinon.spy(Database, 'save');

  setupNewUser({ name: 'test' }, function() { });

  save.restore();
  sinon.assert.calledOnce(save);
});
```
`sinon.assert.calledWith`、`spy.lastCall`或`spy.getCall()`可以检查传入函数的参数。比如我们想确认之前提到的save函数接收到了正确的参数，可以使用以下代码：
```
it('should pass object with correct values to save', function() {
  var save = sinon.spy(Database, 'save');
  var info = { name: 'test' };
  var expectedUser = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };

  setupNewUser(info, function() { });

  save.restore();
  sinon.assert.calledWith(save, expectedUser);
});
```
spy能做的检查不仅仅只有这些，Sinon还提供其他断言函数，你可以使用它们检查各种不同的事情。而且相同的断言还可以用在stub函数上。
如果你使用spy监控其他函数，这个函数的行为不会受到影响。所以如果你想改变一个函数的行为，你需要stub。
## 何时使用stub
stub和spy很像，不过stub可以替换目标函数，自定义函数的行为，比如函数返回值和抛出的异常。它甚至还可以自动调用回调函数。
stub的常见使用场景如下：
- 使用stub替换容易出现问题的代码
- 使用stub触发特定的代码逻辑——如错误处理
- 使用stub测试异步代码

**stub可以替换容易出现问题的代码**，这些代码可能会让测试用例难以编写。通常是一些外部的依赖容易出现问题，如网络链接、数据库操作或者其他非JS的系统。这些外部依赖通常需要一些手动设置，比如需要在运行测试之前向数据库中填充测试数据，这些都会使编写和运行测试用例变得复杂。
如果能替换掉这些容易出现问题的代码，就能避免这些问题。上面的示例函数中调用了`Database.save`，如果没有正确的设置数据库可能出现问题。所以相比于使用spy，使用stub替换是一个更好的方案。
```
it('should pass object with correct values to save', function() {
  var save = sinon.stub(Database, 'save');
  var info = { name: 'test' };
  var expectedUser = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };

  setupNewUser(info, function() { });

  save.restore();
  sinon.assert.calledWith(save, expectedUser);
});
```
使用stub替换数据库操作相关的代码，我们不再需要实际的数据库。类似的方法也可以用在其他难以测试的代码上。

**使用stub可以触发特定的代码逻辑**。有时候我们想测试函数在异常状况下的行为——通常是捕获抛出的异常，我们可以使用stub来模拟抛出异常。
```
it('should pass the error into the callback if save fails', function() {
  var expectedError = new Error('oops');
  var save = sinon.stub(Database, 'save');
  save.throws(expectedError);
  var callback = sinon.spy();

  setupNewUser({ name: 'foo' }, callback);

  save.restore();
  sinon.assert.calledWith(callback, expectedError);
});
```
最后，**stub还可以简化异步代码的测试**，我们可以替换掉异步函数，强制让回调函数立即执行，使代码变成同步的，这样就移除了异步的代码。
```
it('should pass the database result into the callback', function() {
  var expectedResult = { success: true };
  var save = sinon.stub(Database, 'save');
  save.yields(null, expectedResult);
  var callback = sinon.spy();

  setupNewUser({ name: 'foo' }, callback);

  save.restore();
  sinon.assert.calledWith(callback, null, expectedResult);
});
```
stub是高度可配置的，可以做到还有[更多](https://sinonjs.org/releases/v18/stubs/)。但是使用方式基本是相同的。

## 何时使用mock
使用mock时，你应该特别注意。由于mock可以做很多事情，所以我们很容易忽视spy和stub。mock很容易使你的测试过于具体，这导致测试用例很脆弱。脆弱的测试用例很容易失败，只要代码发生更改。
mock的使用场景和stub相同，但如果你想确认更加具体的行为时可以使用mock。
例如，下面的例子可以验证数据库保存时的具体的场景。
```
it('should pass object with correct values to save only once', function() {
  var info = { name: 'test' };
  var expectedUser = {
    name: info.name,
    nameLowercase: info.name.toLowerCase()
  };
  var database = sinon.mock(Database);
  database.expects('save').once().withArgs(expectedUser);

  setupNewUser(info, function() { });

  database.verify();
  database.restore();
});
```
注意，使用mock时，我们提前定义了期望值。通常期望值是在断言函数之后调用。但是我们在定义mock函数时就直接定义了期望，然后调用`verify`。
这个例子中，我们使用`once`和`withArgs`定义一个mock，该mock同时检查了函数的调用次数和参数。如果使用stub，检查多个条件需要多个断言，这会使代码更容易理解。
由于mock在声明多个断言时很便捷，所以很容易使测试太过具体。并使得测试代码很难理解且容易失败，所以在使用mock时，应该时刻记得避免定义多个断言。

## 最佳实践
使用spy、stub、mock时，遵守下面的最佳实践可以避免一些常见的问题。

### 使用sinon.test
使用spy、stub和mock时，使用`sinon.test`函数，sinon可以自动执行清理函数。不使用时，在清理测试替身之前测试用例可能失败，这可能会引起级联失败：由于第一个失败引起的后续的测试失败。这种级联失败很容易掩盖最初的失败原因，所以应该尽可能的避免它。
使用`sinon.test`能够消除这种级联失败，下面是我们之前写的测试代码：
```
it('should call save once', function() {
  var save = sinon.spy(Database, 'save');

  setupNewUser({ name: 'test' }, function() { });

  save.restore();
  sinon.assert.calledOnce(save);
});
```
如果`setupNewUser`抛出了异常，spy就不会被清理。这可能会破坏后续的测试用例。使用`sinon.test`可以避免这样的问题：
```
it('should call save once', sinon.test(function() {
  var save = this.spy(Database, 'save');

  setupNewUser({ name: 'test' }, function() { });

  sinon.assert.calledOnce(save);
}));
```
注意上面代码的三个不同：第一行使用`sinon.test`包裹整个测试函数。第二行用`this.spy`替代了`sinon.spy`。最后，我们移除了`save.restore`。因为测试替身会被自动清除掉。
使用`sinon.test`时，三个测试替身函数的调用都会发生变化：
- `sinon.spy`变为`this.spy`
- `sinon.stub`变为`this.stub`
- `sinon.mock`变为`this.mock`

### 在sinon.test测试异步代码
使用`sinon.test`测时异步代码的时可能需要禁用模拟定时器，尤其是在**Mocha**中测试异步代码的时候。在**Mocha**中如果需要测试异步代码，你可以穿入一个额外的参数：
```
it('should do something async', function(done) {
```
不过在使用`sinon.test`时可能会导致失败：
```
it('should do something async', sinon.test(function(done) {
```
同时使用`sinon.test`和`done`可能会导致失败的测试用例显示的错误原因，测试超时。这是因为当使用`sinon.test`包裹测试代码时，模拟定时器默认是开启状态，所以你需要禁用他们。
禁用模拟定时器可以在代码中，或者配置文件中通过sinon.config设置：
```
sinon.config = {
  useFakeTimers: false
};
```
sinon.config控制一些函数的默认行为，比如`sinon.test`函数。
### 在beforeEach中调用stub
如果你想在所有的测试中替换某个函数，可以在`beforeEach`中调用。例如，我们需要用测试替身替换所有的`Database.save`，可以像下面这样：
```
describe('Something', function() {
  var save;
  beforeEach(function() {
    save = sinon.stub(Database, 'save');
  });

  afterEach(function() {
    save.restore();
  });

  it('should do something', function() {
    //you can use the stub in tests by accessing the variable
    save.yields('something');
  });
});
```
别忘了在`afterEach`中调用清理函数清理掉函数替身。如果没有清理掉测试替身，其他的测试用例可能会受到影响。
### 检查函数调用顺序和值
如果你想检查某个函数是否按顺序调用，你可以使用spy和stub，调用`sinon.assert.callOrder`:
```
var a = sinon.spy();
var b = sinon.spy();

a();
b();

sinon.assert.callOrder(a, b);
```
如果你想在某个函数调用之前检查变量的值，可以向stub函数的第三个参数中传入一个断言函数：
```
var object = { };
var expectedValue = 'something';
var func = sinon.stub(example, 'func', function() {
  assert.equal(object.value, expectedValue);
});

doSomethingWithObject(object);

sinon.assert.calledOnce(func);
```
stub内部的断言函数可以在测试替身调用之前检查某个变量的初始值是否正确。记得调用`sinon.assert.calledOnce`来确保测试替身函数被调用过，否则即使测试替身没有被调用，你的测试用例也不会失败。
## 结论
**Sinon**是个强大的工具，遵守上面的最佳实践，可以避免大多数开发者常见的问题。还有不需要忘记使用`sinon.test`，否则可能会导致级联失败。