---
title: 用JS实现一个极简编译器【译文】
author: Neil Ning
date: 2024-01-17 22:43:20
tags: ['AST', 'COMPILER', 'PARSER', 'TOKENIZER']
categories: 翻译
cover: bg.jpeg
---
## 前言
该文章翻译自【The super tiny compiler】，[点击这里](https://github.com/jamiebuilds/the-super-tiny-compiler/blob/d8d40130459d1537f6117a927947cd46c83182b0/the-super-tiny-compiler.js)查看原文。

今天我们来一起实现一个编译器，不过不是那种通用的编译器，而是一个超级简单的傻瓜编译器。编译的代码非常精简，如果移除代码注释，真正的代码只有大约200行。
**我们的目标是将lisp语言风格函数调用代码转换为C语言风格的函数调用。**
如果你不熟悉这两种语言，可以看下面的示例，假设我们有两个函数add和subtract，他们的写法如下：

|  | LISP | C |
| --- | --- | --- |
| 2 + 2 |`(add 2 2)`  |`add(2, 2)`  |
| 4 - 2 |`(subtract 4 2)`  |`subtract(4, 2)`  |
| 2 + (4 - 2)  |`(add 2 (subtract 4 2))`  | `add(2, subtract(4, 2))` |

看起来非常简单，对吧？
这就是我们的编译器要实现的功能。它不包含完整的LISP或者C语言的语法，但是以上示例已经足够可以演示现代编译器的基本原理。
大多数编译器的工作过程可以分为三个步骤：代码解析、语法转换和代码生成。
1. 代码解析将原始的代码转换为代码的抽象表示（AST）。
2. 语法转换指得是修改代码的抽象表示。
3. 代码生成将修改过的抽象代码生成新的代码。

## 代码解析
代码解析通常包含两个阶段：词法分析和语法分析
1. 词法分析将原始的代码分割成一个叫做tokens的东西，这个过程也叫tokenizer或lexer。
Tokens是一个数组，数组元素是描述语法的最小单位。

2. 语法分析将上面的tokens转换成抽象语法，这个抽象的语法描述了语法和各个tokens之间的关系，即我们熟悉的抽象语法树。
抽象语法树，Abstract Syntax Tree或简称AST，是一个嵌套的对象，属于描述语法信息的一种方式。

对于下面的代码语法：
```
(add 2 (subtract 4 2))
```
Tokens的形式大致如下：
```
[
  { type: 'paren',  value: '('        },
  { type: 'name',   value: 'add'      },
  { type: 'number', value: '2'        },
  { type: 'paren',  value: '('        },
  { type: 'name',   value: 'subtract' },
  { type: 'number', value: '4'        },
  { type: 'number', value: '2'        },
  { type: 'paren',  value: ')'        },
  { type: 'paren',  value: ')'        },
]
```
抽象语法树（AST）的形式如下：
```
{
  type: 'Program',
  body: [{
   type: 'CallExpression',
   name: 'add',
   params: [{
     type: 'NumberLiteral',
     value: '2',
   }, {
     type: 'CallExpression',
     name: 'subtract',
     params: [{
       type: 'NumberLiteral',
       value: '4',
     }, {
       type: 'NumberLiteral',
       value: '2',
     }]
   }]
 }]
}
```
## 语法转换
编译器的下一个阶段称之为语法转换，指得是对上一步得到的抽象语法树进行转换或者修改。这一步既可以是对同一种语言的抽象语法树的操作，也可以将它完全转换为另外一种语言的抽象语法树。我们来看一下如何转换抽象语法树。
你可能已经注意到AST由很多相似的元素组成，每个元素对象都有一个`type`属性。这些元素被称之为AST节点，这些节点使用不同的属性来描述语法树的每个部分。
我们可以用“NumberLiteral”描述一个数字：
```
{
  type: 'NumberLiteral',
  value: '2',
}
```
或者一个“CallExpression”节点：
```
{
  type: 'CallExpression',
  name: 'subtract',
  params: [...nested nodes....]
}
```
我们可以通过添加、删除、替换属性的方式来转换AST。如添加新的节点、移除不想要的节点。或者根据当前AST的结构创建一个全新的AST。
上文说过我们的目标是创建新的语言语法，所以接下来我们要重点关注如何创建一个新的抽象语法树。

### 遍历语法树
为了访问AST的每个节点，我们需要遍历他们。这个遍历AST所有节点的过程会使用深度优先的原则。
```
{
  type: 'Program',
  body: [{
    type: 'CallExpression',
    name: 'add',
    params: [{
      type: 'NumberLiteral',
      value: '2'
    }, {
      type: 'CallExpression',
      name: 'subtract',
      params: [{
        type: 'NumberLiteral',
        value: '4'
      }, {
        type: 'NumberLiteral',
        value: '2'
      }]
    }]
  }]
}
```
所以对于上面的AST，节点的遍历顺序如下：
1. Program - 首先访问AST的外层
2. CallExpression (add) - 移动到Program body属性的第一个元素
3. NumberLiteral (2) - 移动到CallExpression的params属性的第一个元素
4. CallExpression (subtract) - 移动到CallExpression的params属性的第二个元素
5. NumberLiteral (4) - 移动到CallExpression params属性的第一个元素
6. NumberLiteral (2) - 移动到CallExpression params属性的第二个元素

如果我们想直接操作当前的AST，而不是创建一个全新的独立AST。我们可能还需要介绍其他种类的抽象语法树。这里我们只是想实现上文提到的目标，仅仅访问每个节点已经足够。
我之所以使用“访问”这个词，是因为我们要使用访问者模式来操作语法树对象的每个元素。
### 访问者模式
为了实现访问者模式，我们会创建一个visitor对象，该对象包含不同的方法，每个方法只负责接收处理特定类型的节点。
```
var visitor = {
  NumberLiteral() {},
  CallExpression() {},
};
```
遍历抽象语法树时，当“进入”匹配的节点时会调用相对应的方法。
不过为了使方法内部实现更加便捷，我们一般也会把节点的父节点的引用也传入函数。
```
var visitor = {
  NumberLiteral(node, parent) {},
  CallExpression(node, parent) {},
};
```
不过有时我们也需要在“退出”节点时调用函数，如上面的树形结构：
```
 - Program
   - CallExpression
     - NumberLiteral
     - CallExpression
       - NumberLiteral
       - NumberLiteral
```
向下遍历时，我们会抵达每个分支的叶子节点。探索完当前分支后会退出该分支。因此当向下遍历时会“进入”节点，向上返回时会“退出”节点。
```
-> Program (enter)
  -> CallExpression (enter)
    -> Number Literal (enter)
    <- Number Literal (exit)
    -> Call Expression (enter)
       -> Number Literal (enter)
       <- Number Literal (exit)
       -> Number Literal (enter)
       <- Number Literal (exit)
    <- CallExpression (exit)
  <- CallExpression (exit)
<- Program (exit)
```
为了支持退出功能，最终我们的visitor对象是如下形式：
```
var visitor = {
  NumberLiteral: {
    enter(node, parent) {},
    exit(node, parent) {},
  }
};
```
## 代码生成
编译器的最后一个步骤是生成代码，有的编译器会做一些转换，但大多数情况下，代码生成是将AST转换为代码字符串。
代码生成器有几种不同的工作方式，有些编译器会利用之前生成的tokens，有些则会单独创建代码表示，以便可以线性的形式打印出每个节点。但是我可以确定的是大多数编译器会利用上面创建的AST，接下来会我们会探讨这一部分。
代码生成器知道如何高效的打印AST中各个类型的节点，它通过递归的方式打印嵌套的节点，直到所有的节点都被打印成字符串代码。

以上就是编译器的主要组成部分。但并不是所有的编译器都像上面描述的那样，由于编译器有各种不同的用途，有些编译器还需要其他的步骤。但是现在你应该已经知道了大多数编译器长什么样子了。
我已经向你详细介绍了编译器的工作原理，这是不是说你已经可以写出一个编译器了么？——开个玩笑，这是我下面需要演示的部分。
所以让我们开始吧……
## 词法分析
首先我们使用tokenizer开始我们的第一步——词法分析。
我们将字符串代码分割成由token组成的数组。将：
```
(add 2 (subtract 4 2))
```
转换成：
```
[{ type: 'paren', value: '(' }, ...]
```
代码及注释如下：
```
// 我们的函数接收字符串代码作为入参
function tokenizer(input) {

  // `current` 变量作为指针跟踪当前代码中位置。
  let current = 0;

  // `tokens` 数组存放生成的结果
  let tokens = [];

  // 首先创建一个while循环，在循环体中自增`current`变量。
  //
  // 之所以这么做是因为我们可以在一个循环里设置current变量任意次数，
  // 同时tokens数组的长度也是任意的
  while (current < input.length) {

    // 获取代码字符串的中当前位置的字符。
    let char = input[current];

    // 首先检查该字符串是不是左括号，待会儿CallExpression时我们会用到它，
    // 目前我们只需要关注字符
    //
    // 检查是否左括号:
    if (char === '(') {

      // 如果是，我们将生成一个`paren`类型的token，它的值是‘（’，并将它推入数组中
      tokens.push({
        type: 'paren',
        value: '(',
      });

      // `current`变量+1
      current++;

      // 继续进行下一轮循环
      continue;
    }

    // 接下来检查右括号，具体的步骤和上面的一样：检查右括号，创建一个新的类型的token。
    // 自增`current`变量，进行下一轮循环
    if (char === ')') {
      tokens.push({
        type: 'paren',
        value: ')',
      });
      current++;
      continue;
    }

    // 紧接着我们检查当前字符串是不是空格，我们关注空格是因为他会分割字符串，但是空格对
    // 我们不重要，所以我们不会为他创建新的token。
    //
    // 这里我们仅仅检查是否是空格，如果是我们进行下一轮循环。
    let WHITESPACE = /\s/;
    if (WHITESPACE.test(char)) {
      current++;
      continue;
    }

    // 下一个要处理的类型是数字，这个类型与之前所看到的有些不同，因为数字可能是任意长度的
    // 数字序列，我们需要获取完整的数字序列作为token的值。
    //
    //   (add 123 456)
    //        ^^^ ^^^
    //        两个数字类型的tokens
    //
    // 检查数字序列的第一个数字
    let NUMBERS = /[0-9]/;
    if (NUMBERS.test(char)) {

      // 创建一个新的`value`来存储整个数字序列
      let value = '';

      // 创建一个新的循环，依次检查是否是数字，直到遇到非数字的代码字符，我们才结束循环
      // 在循环内我们将字符串存入`value`变量，并自增`current`变量。
      while (NUMBERS.test(char)) {
        value += char;
        char = input[++current];
      }

      // 循环结束时候，新增一个`number`类型的token到`tokens`数组.
      tokens.push({ type: 'number', value });

      // 继续进行下一轮循环
      continue;
    }

    // 我们还需要在我们的语言中支持由双引号包裹的字符串类型
    //
    //   (concat "foo" "bar")
    //            ^^^   ^^^ 字符串tokens
    //
    // 首先检查左侧双引号:
    if (char === '"') {
      // 同样创建一个`value`来存储字符串的值
      let value = '';

      // 获取双引号的下一个位置的字符
      char = input[++current];

      // 创建一个新的while循环，直到遇到右双引号
      while (char !== '"') {
        value += char;
        char = input[++current];
      }

      // 循环结束之后跳过右侧双引号
      char = input[++current];

      // 追加一个字符串类型的token到`tokens`数组.
      tokens.push({ type: 'string', value });

      continue;
    }

    // 最后一个要处理的类型的关键字类型，他是一个特定的字符串序列。他是我们的lisp
    // 语法中的一个函数名称
    //
    //   (add 2 4)
    //    ^^^
    //    函数关键字token
    //
    let LETTERS = /[a-z]/i;
    if (LETTERS.test(char)) {
      let value = '';

      // 同样的，我们创建while循环迭代所有的字母并将它推入value变量
      // a value.
      while (LETTERS.test(char)) {
        value += char;
        char = input[++current];
      }

      // 循环结束之后，新增一个name类型的token。
      tokens.push({ type: 'name', value });

      continue;
    }

    // 最后，如果没有匹配的字符，我们抛出一个异常，退出程序。
    throw new TypeError('I dont know what this character is: ' + char);
  }

  // `tokenizer`的最后一步，返回tokens数组
  return tokens;
}
```

## 解析器
我们的解析器接收tokens数组作为入参，并将它转换为AST
```
[{ type: 'paren', value: '(' }, ...]   =>   { type: 'Program', body: [...] }
```
```
// 定义一个`parser`函数接收`tokens`数组作为参数
function parser(tokens) {

  // 同样地，我们创建一个`current`作为指针
  let current = 0;

  // 但是这次，我们需要使用递归来替代`while`循环，所以我们定一个`walk`函数
  function walk() {

    // walk函数内部，我们通过`current`指针获取当前token.
    let token = tokens[current];

    // 我们需要将不同类型的token放置到不同类型的代码路径中去，以`number`类型的token作为开始
    //
    // 检查当前token的类型是否是`number`
    if (token.type === 'number') {

      // 自增`current`变量
      current++;

      // 我们会直接返回一个`NumberLiteral`类型的AST节点，并将它的设置为当前token的值
      return {
        type: 'NumberLiteral',
        value: token.value,
      };
    }

    // 和number一样，如果token的类型是string，我们创建并返回一个`StringLiteral`类型的AST节点
    if (token.type === 'string') {
      current++;

      return {
        type: 'StringLiteral',
        value: token.value,
      };
    }

    // 接下来，我们需要查找调用表达式CallExpressions，首先检查当前token是否是左括号
    if (
      token.type === 'paren' &&
      token.value === '('
    ) {

      // 自增`current`指针来跳过左括号，因为在AST中我们不需要关注它
      token = tokens[++current];

      // 先创建一个`CallExpression`节点，将节点的name属性设置为token的值，
      // 因为左括号的下一个token就是函数的名称
      let node = {
        type: 'CallExpression',
        name: token.value,
        params: [],
      };

      // 再次自增`current`变量跳过name类型的token.
      token = tokens[++current];

      // 接下来我们需要依次迭代每一个token直到我们遇到右括号类型的token。
      // 这些token是`CallExpression`节点的`params`属性的值
      
      // 下面就是使用递归的地方，我们不直接解析可能嵌套多层的节点，而是使用递归来处理这种嵌套情况
      //
      // 为了进一步解释说明，我们假设有以下Lisp代码，可以看到add函数的参数是一个嵌套的
      // `CallExpression`类型的节点，这个节点有它自己的两个number类型的参数节点。
      //
      //   (add 2 (subtract 4 2))
      //
      // 你可能已经注意到我们的tokens数组有多个右括号类型的token
      //
      //   [
      //     { type: 'paren',  value: '('        },
      //     { type: 'name',   value: 'add'      },
      //     { type: 'number', value: '2'        },
      //     { type: 'paren',  value: '('        },
      //     { type: 'name',   value: 'subtract' },
      //     { type: 'number', value: '4'        },
      //     { type: 'number', value: '2'        },
      //     { type: 'paren',  value: ')'        }, <<< 右括号
      //     { type: 'paren',  value: ')'        }, <<< 右括号
      //   ]
      //
      // 我们嵌套调用walk函数来自增`current`变量来处理嵌套的`CallExpression`

      // 所以这里我们需要创建一个while循环直到遇到`paren`类型的token，
      // 并且他的值是右括号，或者非'paren'`类型的token
      while (
        (token.type !== 'paren') ||
        (token.type === 'paren' && token.value !== ')')
      ) {
        // 在循环体中调用`walk`函数，并将函数返回的节点推入`node.params`数组中
        node.params.push(walk());
        token = tokens[current];
      }

      // 最后，我们还需要自增`current`变量来跳过右括号
      current++;

      // 返回节点
      return node;
    }

    // 同样的，如果我们遇到未知类型的token，需要跑出异常
    throw new TypeError(token.type);
  }

  // 现在我们可以开始创建的AST了，AST有一个`Program`类型的根节点
  let ast = {
    type: 'Program',
    body: [],
  };

  // 接下来开始调用`walk`函数，并将函数返回的节点推入`ast.body`数组中
  //
  // 我们之所以在while循环中调用walk函数，是因为我们代码中可能会有多个语句`CallExpression`
  //
  //   (add 2 2)
  //   (subtract 4 2)
  //
  while (current < tokens.length) {
    ast.body.push(walk());
  }

  // 最后返回我们构建好的AST
  return ast;
}
```
## 遍历器
现在我们已经有AST了，接下来需要使用visitor对象来访问不同的节点，当遇到匹配的节点类型时，需要调用visitor对象对应类型的方法。
```
traverse(ast, {
  Program: {
    enter(node, parent) {
      // ...
    },
    exit(node, parent) {
      // ...
    },
  },
  
  CallExpression: {
    enter(node, parent) {
      // ...
    },
    exit(node, parent) {
      // ...
    },
  },
  
  NumberLiteral: {
    enter(node, parent) {
      // ...
    },
    exit(node, parent) {
      // ...
    },
  },
});
```
所以我们定义一个traverser函数，它接收AST和visitor对象作为参数。为了实现这个目标，还需要在函数内部定义两个函数。
```
function traverser(ast, visitor) {

  // `traverseArray`函数可以迭代访问数组，函数内部调用下面定义的`traverseNode`函数
  function traverseArray(array, parent) {
    array.forEach(child => {
      traverseNode(child, parent);
    });
  }

  // `traverseNode`函数接收当前节点`node`和他的父节点`parent`作为参数，
  // 以便他们能够传给visitor对象的函数
  function traverseNode(node, parent) {

    // 首先检查visitor对象中，有没有匹配当前节点类型的方法
    let methods = visitor[node.type];

    // 如果vistor对象中包含enter方法，使用`node`和`parent`作为参数调用该方法
    if (methods && methods.enter) {
      methods.enter(node, parent);
    }

    // 接下来我们处理不同的节点类型
    switch (node.type) {

      // 首先处理根节点`Program`，由于Program节点的body属性是一个数组，
      // 所以我们调用`traverseArray`函数向下遍历他的每一个节点
      // 注意`traverseArray`函数调用了`traverseNode`函数，所以我们会递归的遍历节点树的每个节点
      case 'Program':
        traverseArray(node.body, node);
        break;

      // 接下来的`CallExpression`类型的处理方式上面一样
      case 'CallExpression':
        traverseArray(node.params, node);
        break;

      // `NumberLiteral`和`StringLiteral`没有任何的子节点，所以我们直接跳过
      case 'NumberLiteral':
      case 'StringLiteral':
        break;

      // 同样地，如果没有匹配的类型抛出异常
      default:
        throw new TypeError(node.type);
    }

    // 最后如果存在`exit`方法，我们使用`node`和`parent`调用该方法
    if (methods && methods.exit) {
      methods.exit(node, parent);
    }
  }

  // 我们调用`traverseNode`方法启动递归遍历，AST根节点没有父节点，所以`parent`参数是null
  traverseNode(ast, null);
}
```
## 转换器
接下来实现转换器，我们的转换器需要将我们上面构建好的AST传入遍历器函数，并使用visitor对象创建一个新的AST。
```
 * ----------------------------------------------------------------------------
 *   Original AST                     |   Transformed AST
 * ----------------------------------------------------------------------------
 *   {                                |   {
 *     type: 'Program',               |     type: 'Program',
 *     body: [{                       |     body: [{
 *       type: 'CallExpression',      |       type: 'ExpressionStatement',
 *       name: 'add',                 |       expression: {
 *       params: [{                   |         type: 'CallExpression',
 *         type: 'NumberLiteral',     |         callee: {
 *         value: '2'                 |           type: 'Identifier',
 *       }, {                         |           name: 'add'
 *         type: 'CallExpression',    |         },
 *         name: 'subtract',          |         arguments: [{
 *         params: [{                 |           type: 'NumberLiteral',
 *           type: 'NumberLiteral',   |           value: '2'
 *           value: '4'               |         }, {
 *         }, {                       |           type: 'CallExpression',
 *           type: 'NumberLiteral',   |           callee: {
 *           value: '2'               |             type: 'Identifier',
 *         }]                         |             name: 'subtract'
 *       }]                           |           },
 *     }]                             |           arguments: [{
 *   }                                |             type: 'NumberLiteral',
 *                                    |             value: '4'
 * ---------------------------------- |           }, {
 *                                    |             type: 'NumberLiteral',
 *                                    |             value: '2'
 *                                    |           }]
 *  (sorry the other one is longer.)  |         }
 *                                    |       }
 *                                    |     }]
 *                                    |   }
 * ----------------------------------------------------------------------------
```

所以我们的转换器接收lisp代码的AST作为参数
```
function transformer(ast) {

  // 创建一个`newAst`变量，和之前一样它的根节点类型是program
  let newAst = {
    type: 'Program',
    body: [],
  };

  // 接下来我们需要使用一些特别的方式来实现transformer，在parent节点上创建一个`context`属性，
  // 然后将node推入parent节点的`context`数组中。通常会有其他方式来实现这个，但是这里为了演示方便，
  // 我们采用该方法。
  //
  // 需要注意context属性只是在老的AST上对新AST的引用
  ast._context = newAst.body;

  // 使用visitor对象调用traverser函数来遍历AST的每个节点
  traverser(ast, {

    // 处理所有`NumberLiteral`类型的节点
    NumberLiteral: {
      // 进入节点时会调用enter方法
      enter(node, parent) {
        // 创建一个`NumberLiteral`节点，并将它推入parent节点的context属性中，
        // NumberLiteral节点的父节点通常的是个CallExpression节点，后面处理该节点时可以看到
        // 他的context是新AST中CallExpression节点的arguments属性的引用，
        // 及parent._context -> expression.arguments
        parent._context.push({
          type: 'NumberLiteral',
          value: node.value,
        });
      },
    },

    // `StringLiteral`类型和上面的处理方式类似
    StringLiteral: {
      enter(node, parent) {
        parent._context.push({
          type: 'StringLiteral',
          value: node.value,
        });
      },
    },

    // 接下来处理`CallExpression`
    CallExpression: {
      enter(node, parent) {

        // 创建一个新的`CallExpression`节点，它包含一个嵌套的'Identifier'节点，
        // callee属性指向这个'Identifier'节点，该节点的name属性是老的
        // `CallExpression`节点的name属性（如：subtract函数名）
        let expression = {
          type: 'CallExpression',
          callee: {
            type: 'Identifier',
            name: node.name,
          },
          arguments: [],
        };

        // 接下来，在老的`CallExpression`节点上创建一个context属性指向新的CallExpression
        // 节点的arguments参数（参考上文中NumberLiteral节点的处理中，
        // 参数会被推入parent._context数组中）
        node._context = expression.arguments;

        // 然后检查当前节点的父节点是不是`CallExpression`类型来处理嵌套的CallExpression
        // 节点（上文中左半部分的示例）如果该节点父节点不是CallExpression类型的
        if (parent.type !== 'CallExpression') {

          // 如果不是，则说明该节点已经是最外层的CallExpression节点，
          // 我们需要使用ExpressionStatement类型的节点对他进行包裹，
          // 之所以这么做是因为在JavaScript代码中，顶层的CallExpression节点实
          // 际上是个ExpressionStatement节点（参考上面示例中右半部分）
          expression = {
            type: 'ExpressionStatement',
            expression: expression,
          };
        }

        // 最后将新的节点推入parent的context属性中（即ast._context属性，
        // 它指向了新AST的body属性）
        parent._context.push(expression);
      },
    }
  });

  // 在函数的最后，返回新创建的AST
  return newAst;
}
```
## 代码生成器
接下来我们实现最后一个阶段：代码生成，我们的代码生成器会递归地打印AST的每个节点并把他们合并起来生成一份完整的代码字符串。
```
function codeGenerator(node) {

  // 我们将按节点的类型进行细分处理
  switch (node.type) {

    // 如果是`Program`节点，我们需要处理`body`数组的每个元素，并将他们用换行符连接起来
    case 'Program':
      return node.body.map(codeGenerator)
        .join('\n');

    // 对与`ExpressionStatement`节点，我们调用codeGenerator方法处理嵌套的语句调用并在语句的末尾添加分号
    case 'ExpressionStatement':
      return (
        codeGenerator(node.expression) +
        ';' // << (...分号)
      );

    // 对于`CallExpression`节点，首先使用codeGenerator函数处理节点的callee属性，打印出被调用的函数名称，
    // 紧接着是个左小括号，然后处理节点的`arguments`数组，他们是函数的参数，需要用逗号连接，最后是右括号
    case 'CallExpression':
      return (
        codeGenerator(node.callee) +
        '(' +
        node.arguments.map(codeGenerator)
          .join(', ') +
        ')'
      );

    // 上面处理`CallExpression`节点打印函数名称时，调用的是codeGenerator(node.callee)，
    // 他是个`Identifier`类型，直接返回节点的名称作为函数名称
    case 'Identifier':
      return node.name;

    // `NumberLiteral`节点直接返回节点值
    case 'NumberLiteral':
      return node.value;

    // `StringLiteral`是个字符串类型，需要用双引号包裹节点值
    case 'StringLiteral':
      return '"' + node.value + '"';

    // 未知类型抛出异常
    default:
      throw new TypeError(node.type);
  }
}
```
## 编译器
最后我们来实现`compiler`函数，将上面的所有步骤连接起来
1. input  => tokenizer   => tokens
2. tokens => parser => ast
3. ast => transformer => newAst
4. newAst => generator => output

```
function compiler(input) {
  let tokens = tokenizer(input);
  let ast    = parser(tokens);
  let newAst = transformer(ast);
  let output = codeGenerator(newAst);

  // 将输出的代码返回
  return output;
}
```

最后一步导出所有函数
```
module.exports = {
  tokenizer,
  parser,
  traverser,
  transformer,
  codeGenerator,
  compiler,
};
```