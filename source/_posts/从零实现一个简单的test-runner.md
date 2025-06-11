---
title: 从零实现一个简单的test runner【译】
author: Neil Ning
date: 2025-06-11 23:55:13
tags: ['test runner', '测试框架', 'JavaScript', 'Jest']
categories: 翻译
cover: bg.jpg
---
## 前言
本文翻译自[Building a JavaScript Testing Framework](https://cpojer.net/posts/building-a-javascript-testing-framework)，原文作者是[cpojer](https://cpojer.net/)，本文将介绍如何从零开始实现一个简单的测试运行器（test runner），并支持运行单元测试用例。

在日常工作中，我们经常需要编写测试用例来验证代码的正确性。他可以帮助我们发现代码中的错误和潜在问题，从而提高代码质量。为了运行这些测试用例，我们通常会使用诸如Jest、Mocha等测试框架或测试运行器（test runner）。他们都提供了丰富的功能来运行和管理测试用例，为了更好地理解test runner的工作原理。 我们从零开始实现一个简单的测试运行器，能够支持运行单元测试用例。

## 生成测试用例
首先使用yarn init --yes命令生成一个nodejs项目，然后编写一个简单的测试用例：
```bash
mkdir test-demo
cd test-demo
yarn init --yes
mkdir tests

echo "expect(1).toBe(2);" > tests/01.test.js
echo "expect(2).toBe(2);" > tests/02.test.js
echo "expect(3).toBe(4);" > tests/03.test.js
echo "expect(4).toBe(4);" > tests/04.test.js
echo "expect(5).toBe(6);" > tests/05.test.js
echo "expect(6).toBe(6);" > tests/06.test.js
touch index.mjs
```
上面的代码生成了一些测试用例文件，我们的目标是通过执行`node index.mjs`文件来运行这些测试用例，并输出测试结果。在使用Jest时，我们通过约定的方式正确的放置或命名测试用例文件，Jest会自动查找并运行这些文件。所以实现index.mjs文件第一步就是正确的查找测试用例文件，这里使用`glob`来查找所有的测试用例文件。

首先安装glob：
```bash
yarn add glob
```
然后在index.mjs中编写代码来查找测试用例文件：
```javascript
import glob from 'glob';
const testFiles = glob.sync('**/*.test.js');
console.log('Test files:', testFiles);
```
现在执行`node index.mjs`，应该可以看到输出的测试用例文件列表。Jest框架使用内部的一个包`jest-haste-map`来查找测试用例文件，这个包会缓存文件的查找结果，以提高性能，并返回文件列表的绝对路径。所以我们可以利用这个包改造上面的代码：
```javascript
import JestHasteMap from 'jest-haste-map';
import { cpus } from 'os';
import { dirname, join, relative } from 'path';
import { fileURLToPath } from 'url';

const root = dirname(fileURLToPath(import.meta.url));

const hasteMapOptions = {
  extensions: ['js'],
  maxWorkers: cpus().length,
  name: 'test-runner-demo',
  platforms: [],
  rootDir: root,
  roots: [root],
};

const hasteMap = new JestHasteMap.default(hasteMapOptions);
await hasteMap.setupCachePath(hasteMapOptions);

const testFiles = hasteFS.matchFilesWithGlob([
  process.argv[2] ? `**/${process.argv[2]}*` : '**/*.test.js',
]);
console.log('Test files:', testFiles);
```

现在执行`node index.mjs`，应该可以看到输出的测试用例文件列表，并且文件路径是绝对路径。另外为了支持命令行参数来指定测试用例文件的前缀，我们在代码中添加了一个判断，如果有传入参数，则只查找以该参数开头的测试用例文件。

## 运行测试用例
有了测试文件的路径之后，我们就可以运行这些测试用例了。为了利用多核CPU的优势，我们可以使用`worker_threads`模块来创建多个工作线程来并行运行测试用例。每个工作线程会加载测试用例文件，并执行其中的代码。这里我们使用`jest-worker`来创建工作线程。首先安装`jest-worker`：
```bash
yarn add jest-worker
```
然后创建一个`worker.js`文件来处理测试用例的执行：
```javascript
const fs = require('fs');

exports.runTest = async function (testFile) {
  const code = await fs.promises.readFile(testFile, 'utf8');

  return testFile + ':\n' + code;
};
```
在work.js中，我们定义了一个`runTest`函数来读取测试用例文件的内容，并返回文件名和内容。接下来在`index.mjs`中使用`jest-worker`来创建worker线程并运行测试用例：
```javascript
import JestWorker from 'jest-worker';

const worker = new Worker(join(root, 'worker.js'), {
  enableWorkerThreads: true,
});
 
await Promise.all(
  Array.from(testFiles).map(async (testFile) => {
    const testResult = await worker.runTest(testFile);
    console.log(testResult);
  }),
);
 
worker.end();
```
现在执行`node index.mjs`，应该可以看到输出的测试用例文件名和内容。这里我们只是简单地读取了测试用例文件的内容，并没有执行其中的测试代码。那要如何执行测试代码呢？我们先使用eval函数来执行获取到的代码字符串。
```javascript
const fs = require('fs');

exports.runTest = async function (testFile) {
  const code = await fs.promises.readFile(testFile, 'utf8');
  eval(code);
};
```

## 使用断言库
有的时候测试用例会有一些异常，为了捕获并报告这些异常，我们使用try...catch语句来包裹测试代码，并在catch中输出错误信息。首先定义一个testResult对象以便将测试结果从worker进程返回到主进程：
```javascript
exports.runTest = async function (testFile) {
  const code = await fs.promises.readFile(testFile, 'utf8');
  const testResult = {
    success: false,
    errorMessage: null,
  };
  try {
    eval(code);
    testResult.success = true;
  } catch (error) {
    testResult.errorMessage = error.message;
  }
  return testResult;
};
```

此时我们运行`node index.mjs`,会出现expect不存在的错误，因为测试用例中使用了expect函数来进行断言，但是我们并没有定义expect函数。为了实现这个功能，我们可以先自己实现一个简单的断言库
```javascript
const expect = (received) => ({
  toBe: (expected) => {
    if (received !== expected) {
      throw new Error(`Expected ${expected} but received ${received}.`);
    }
    return true;
  },
});
eval(code);
```

接着我们回到index.mjs中，修改将测试结果打印出来：
```javascript
import chalk from 'chalk';
import { relative } from 'path';

await Promise.all(
  Array.from(testFiles).map(async (testFile) => {
    const { success, errorMessage } = await worker.runTest(testFile);
    const status = success
      ? chalk.green.inverse.bold(' PASS ')
      : chalk.red.inverse.bold(' FAIL ');

    console.log(status + ' ' + chalk.dim(relative(root, testFile)));
    if (!success) {
      console.log('  ' + errorMessage);
    }
  }),
);
```
现在执行`node index.mjs`，应该可以看到测试结果的输出。成功的测试用例会显示绿色的PASS，失败的测试用例会显示红色的FAIL，并且会输出错误信息。

不过我们断言库的实现非常简单，只有一个`toBe`方法来进行相等断言。还有很多其他方法没有实现，比如`toEqual`、`toBeTruthy`等。为了更好地模拟Jest的断言库，我们可以使用`expect`包来提供更丰富的断言功能。
首先安装`expect`包：
```bash
yarn add expect
```
然后修改`worker.js`中的代码来使用`expect`包：
```javascript
const fs = require('fs');
const expect = require('expect');

exports.runTest = async function (testFile) {
  const code = await fs.promises.readFile(testFile, 'utf8');
  const testResult = {
    success: false,
    errorMessage: null,
  };
  try {
    eval(code);
    testResult.success = true;
  } catch (error) {
    testResult.errorMessage = error.message;
  }
  return testResult;
};
```
引入这个库之后，我们的测试用例就可以使用更丰富的断言方法了，创建一个测试文件`tests/mock.test.js`：
```javascript
const mock = require('jest-mock');

const fn = mock.fn();
expect(fn).not.toHaveBeenCalled();
fn();
expect(fn).toHaveBeenCalled();
```

这里用到了`jest-mock`包，所以我们还要执行`yarn add jest-mock`来安装这个包。现在执行`node index.mjs mcok.test.js`就可以执行这个文件了。

最后我们的test runner还有一个问题，测试用例出错之后，程序没有异常退出。为了解决这个问题，我们继续改造index.mjs文件：
```javascript
let hasFailed = false;
await Promise.all(
  Array.from(testFiles).map(async (testFile) => {
    const { success, errorMessage } = await worker.runTest(testFile);
    const status = success
      ? chalk.green.inverse.bold(' PASS ')
      : chalk.red.inverse.bold(' FAIL ');

    console.log(status + ' ' + chalk.dim(relative(root, testFile)));
    if (!success) {
      hasFailed = true; // Something went wrong!
      console.log('  ' + errorMessage);
    }
  }),
);
worker.end();
if (hasFailed) {
  console.log(
    '\n' + chalk.red.bold('Test run failed, please fix all the failing tests.'),
  );
  // Set an exit code to indicate failure.
  process.exitCode = 1;
}
```
上面的代码，我们定义了一个hasFailed变量来记录是否有测试用例失败，如果有测试用例失败，则在最后输出一条错误信息，并设置`process.exitCode`为1，表示程序异常退出。

## 实现describe和it函数
在使用Jest时，为了能更好的组织测试用例，我们通常会使用`describe`和`it`函数来定义测试套件和测试用例。为了实现这个功能，我们需要在worker.js中添加对这两个函数的支持。首先创建一个测试文件`tests/circus.test.js`：
```javascript
describe('circus test', () => {
  it('works', () => {
    expect(1).toBe(1);
  });
});

describe('second circus test', () => {
  it(`doesn't work`, () => {
    expect(1).toBe(2);
  });
});
```
接着，我门需要在worker.js中定义describe和it函数
```javascript
try {
  const describeFns = [];
  let currentDescribeFn;
  const describe = (name, fn) => describeFns.push([name, fn]);
  const it = (name, fn) => currentDescribeFn.push([name, fn]);
  eval(code);

  testResult.success = true;
} catch (error) {
  // …
}
```
以上代码在执行完eval(code)之后，只是执行了测试文件中的describe函数，还没有真正执行测试用例。为了执行测试用例，我们需要在worker.js遍历describeFns数组，并执行describe函数中的第二个参数（fn）：
```javascript
let testName; // Use this variable to keep track of the current test.
try {
  const describeFns = [];
  let currentDescribeFn;
  const describe = (name, fn) => describeFns.push([name, fn]);
  const it = (name, fn) => currentDescribeFn.push([name, fn]);
  eval(code);
  for (const [name, fn] of describeFns) {
    currentDescribeFn = [];
    testName = name;
    fn(); // 函数内部会执行上文的it函数

    currentDescribeFn.forEach(([name, fn]) => {
      testName += ' ' + name;
      fn(); // 执行it函数的第二个参数
    });
  }
  testResult.success = true;
} catch (error) {
  testResult.errorMessage = testName + ': ' + error.message;
}
```
到此为止，我们再次实现了一个简单版的describe和it函数。真正的测试框架还允许describe和it函数嵌套使用，并且可以在describe中定义beforeEach、afterEach等钩子函数，支持异步函数等。为了实现这些功能，我们还是老规矩，使用第三方已经实现好的库来完成。这里我们使用`jest-circus`包：
```bash
yarn add jest-circus
```
在代码中使用`jest-circus`来替换我们自己实现的describe和it函数：
```javascript
const fs = require('fs');
const expect = require('expect');
// Provide `describe` and `it` to tests.
const { describe, it, run, resetState } = require('jest-circus');

exports.runTest = async function (testFile) {
  const code = await fs.promises.readFile(testFile, 'utf8');
  const testResult = {
    success: false,
    errorMessage: null,
  };
  try {
    resetState();
    eval(code);
    // Run jest-circus.
    const { testResults } = await run();
    testResult.testResults = testResults;
    testResult.success = testResults.every((result) => !result.errors.length);
  } catch (error) {
    testResult.errorMessage = error.message;
  }
  return testResult;
};
```
需要注意的是，在使用`jest-circus`时，我们需要在每次运行测试之前调用`resetState()`函数来重置状态。这样可以确保每个测试文件之间的状态是独立的。
接着在index.mjs中输出测试结果：
```javascript
await Promise.all(
  Array.from(testFiles).map(async (testFile) => {
    const { success, testResults, errorMessage } =
      await worker.runTest(testFile);
    const status = success
      ? chalk.green.inverse.bold(' PASS ')
      : chalk.red.inverse.bold(' FAIL ');

    console.log(status + ' ' + chalk.dim(relative(root, testFile)));
    if (!success) {
      hasFailed = true;
      // Make use of the rich `testResults` and error messages.
      if (testResults) {
        testResults
          .filter((result) => result.errors.length)
          .forEach((result) =>
            console.log(
              // Skip the first part of the path which is an internal token.
              result.testPath.slice(1).join(' ') + '\n' + result.errors[0],
            ),
          );
        // If the test crashed before `jest-circus` ran, report it here.
      } else if (errorMessage) {
        console.log('  ' + errorMessage);
      }
    }
  }),
);
```
以上代码会正确打印每个测试文件的测试结果，并且如果有测试文件中的用例失败，会输出详细的错误信息。
## 沙盒环境
通过100行左右的代码，我们已经实现了一个简单的测试运行器，能够支持运行单元测试用例，并输出测试结果。但是我们还在使用eval函数来执行测试用例，这样会有一些安全隐患，比如某个测试用例修改了全局变量，在另外的文件中全局变量的值就会被改变。为了避免这种情况，我们需要为每个测试文件创建沙盒环境，在Node中可以使用`vm`模块来创建沙盒环境。我们可以在worker.js中使用`vm.runInNewContext`来执行测试用例代码：
```javascript
const vm = require('vm');

// replace `eval(code);` with this:
const context = { describe, it, expect, mock };
vm.createContext(context);
vm.runInContext(code, context);
```
这样我们就通过`vm`模块创建了一沙盒环境，我们修改`tests/circus.test.js`中的测试用例来验证沙盒环境的效果：
```javascript
  
describe('second circus test', () => {
  it(`doesn't work`, async () => {
    await new Promise((resolve) => setTimeout(resolve, 2000));
    expect(1).toBe(2);
  });
});
```
当我们执行`node index.mjs circus.test.js`时，会出现setTimeout is not defined的错误。这是因为在沙盒环境中没有全局的setTimeout函数。为了让沙盒环境支持setTimeout等全局函数，我们可以将setTimeout等全局函数添加到沙盒环境中，但是把所有的JS全局对象都添加到沙盒环境中并不是一个好的做法。为了更好的模拟Jest的沙盒环境，我们可以使用`jest-environment-node`包来创建一个Node环境的沙盒。首先安装`jest-environment-node`：
```bash
yarn add jest-environment-node
```
然后在worker.js中使用`jest-environment-node`来创建沙盒环境：
```javascript
// Replace this code:
const context = { describe, it, expect, mock };
vm.createContext(context);

// With this:
const NodeEnvironment = require('jest-environment-node');
const environment = new NodeEnvironment({
  projectConfig: {
    testEnvironmentOptions: { describe, it, expect, mock },
  },
});
vm.runInContext(code, environment.getVmContext());
```

最后，我们还需要解决一个问题，就是require函数，之前使用eval函数来执行测试用例代码时，测试用例可以使用require函数来引入其他模块。但是在沙盒环境中，require函数是不可用的。为了让测试用例能够使用require函数，我们可以在沙盒环境中添加一个require函数。首先我们先创建一个测试用例
```javascript 
// tests/banana.js
module.exports = 'good';
```
修改circus.test.js来使用require函数：
```javascript
const banana = require('./banana.js');

it('tastes good', () => {
  expect(banana).toBe('good');
});
```
此时运行`node index.mjs circus.test.js`会报错，提示require is not defined。然后我们可以在worker.js中先实现一个require函数：
```javascript
const customRequire = (fileName) => {
  const code = fs.readFileSync(join(dirname(testFile), fileName), 'utf8');
  // Define a function in the `vm` context and return it.
  const moduleFactory = vm.runInContext(
    `(function(module) {${code}})`,
    environment.getVmContext(),
  );
  const module = { exports: {} };
  // Run the sandboxed function with our module object.
  moduleFactory(module);
  return module.exports;
};
```
但是以上代码只能require一次，如果我们修改circus.test.js文件：
```javascript
const apple = require('./apple.js');

it('tastes delicious', () => {
  expect(apple).toBe('delicious');
});
```

```javascript
// tests/apple.js
module.exports = 'delicious';
```
此时运行`node index.mjs circus.test.js`会报错，提示`Identifier 'module' has already been declared`。为了解决这个问题，我们进一步改造worker.js中的customRequire函数：
```javascript
const customRequire = (fileName) => {
  const code = fs.readFileSync(join(dirname(testFile), fileName), 'utf8');
  // Define a function in the `vm` context and return it.
  const moduleFactory = vm.runInContext(
    `(function(module, require) {${code}})`,
    environment.getVmContext(),
  );
  const module = { exports: {} };
  // Run the sandboxed function with our module object.
  moduleFactory(module);
  return module.exports;
};
```
现在我们就可以在多次调用require函数了。完整的代码如下：
```javascript
const fs = require('fs');
const expect = require('expect');
const mock = require('jest-mock');
const { describe, it, run, resetState } = require('jest-circus');
const vm = require('vm');
const NodeEnvironment = require('jest-environment-node');
const { dirname, basename, join } = require('path');

exports.runTest = async function (testFile) {
  const testResult = {
    success: false,
    errorMessage: null,
  };
  try {
    resetState();
    let environment;
    const customRequire = (fileName) => {
      const code = fs.readFileSync(join(dirname(testFile), fileName), 'utf8');
      const moduleFactory = vm.runInContext(
        // Inject require as a variable here.
        `(function(module, require) {${code}})`,
        environment.getVmContext(),
      );
      const module = { exports: {} };
      // And pass customRequire into our moduleFactory.
      moduleFactory(module, customRequire);
      return module.exports;
    };
    environment = new NodeEnvironment({
      projectConfig: {
        testEnvironmentOptions: {
          describe,
          it,
          expect,
          mock,
        },
      },
    });
    // Use `customRequire` to run the test file.
    customRequire(basename(testFile));
    const { testResults } = await run();
    testResult.testResults = testResults;
    testResult.success = testResults.every((result) => !result.errors.length);
  } catch (error) {
    testResult.errorMessage = error.message;
  }
  return testResult;
};
```
以上代码中moduleFactory变量的值就是runInContext函数的第一个参数，它是一个函数，这个函数接受两个参数：module和require。module是一个对象，它有一个exports属性，我们可以通过这个属性来导出模块的内容。require是一个函数，我们递归调用了customRequire函数来实现模块的加载。不过以上加载器不能加载Node.js的内置模块和第三方模块。

## 结语
通过以上的步骤，我们从零实现了一个简单的测试运行器，能够支持运行单元测试用例，并输出测试结果。我们使用了`jest-haste-map`来查找测试用例文件，使用`jest-worker`来并行运行测试用例，使用`jest-circus`来组织测试用例，并使用`jest-environment-node`来创建沙盒环境。虽然这个测试运行器还很简单，但它已经具备了基本的功能，可以作为学习和理解测试框架的基础。

## 参考资料
- [Building a JavaScript Testing Framework](https://cpojer.net/posts/building-a-javascript-testing-framework#user-content-fn-9)
- [Jest](https://jestjs.io/)