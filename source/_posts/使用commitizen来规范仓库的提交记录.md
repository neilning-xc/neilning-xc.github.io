---
title: 使用commitizen来规范仓库的提交记录
author: Neil Ning
date: 2021-12-12 18:24:04
tags: ["git", "commitizen", "changelog"]
categories: 学习
cover: bg.jpeg
---
## 前言
在一个多人开发的项目中，开发者可能会因为各种各样的原因，写出很随意的git提交记录，或根据自己的习惯写出难以理解且没有意义的提交记录，众所周知提交记录的书写也是有规范的，目前大家使用较多的规范是[Angular团队推出的规范](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#-git-commit-guidelines)。该规范的大致格式如下：
```js
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```
该规范主要分为header，body，footer三个部分，三部分之间使用空行分隔。其中header分为type、scope、subject三部分，type为必填项，用几种固定的关键字来区分本次提交的目的，具体支持的type类型[参考这里](https://github.com/angular/angular.js/blob/master/DEVELOPERS.md#type)。scope为可选项，用于说明本次提交的影响范围，可以写模块名或者文件名。subject为必填项，用简短的文字说明本地提交的描述。如果是英文的话，首字母不需要大写，末尾不能加句号。body和footer部分也是可选的。

可以看到一份符合规范的提交记录的行文格式是较为繁琐的，如果只使用git commit命令来书写效率未免太过低下，得不偿失。

[commitizen](https://github.com/commitizen/cz-cli)工具就是为了解决这样的问题，当需要提交代码时，可以通过git cz命令代替原生的git commit，它会通过命令行交互的形式帮助开发者生成一份符合规范的优质提交记录。

## 本地项目使用commitizen
执行`npm install --save-dev commitizen`命令在本地项目中安装commitizen，安装完成之后需要为commitizen安装适配器，具体什么是适配器，后面会讲到。通过以下命令为当前项目初始化适配器。
```sh
npx commitizen init cz-conventional-changelog --save-dev --save-exact --force
```
或者通过
```sh
./node_modules/.bin/commitizen init cz-conventional-changelog --save-dev --save-exact --force
```
如果项目使用的是yarn，需要通过如下命令：
```
commitizen init cz-conventional-changelog --yarn --dev --exact
```
该命令执行完之后除了会安装一个npm包之外，还会在在package.json中添加一些配置：
```json
"config": {
  "commitizen": {
    "path": "./node_modules/cz-conventional-changelog"
  }
}
```
之后我们就能在项目中使用commitizen工具了，运行`./node_modules/.bin/cz`，会以交互式命令行的方式辅助完成提交记录书写。为了能够更加方便的使用该命令行，我们可以在package.json添加脚本。
```
"scripts": {
  "cm": "cz"
}
```
之后如果你想提交代码，你可以运行npm run cm来替代默认的git commit，该命令通过交互的形式，生成一份符合[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)规范（Angular规范）的提交记录。

## 全局使用commitizen
本地项目安装完之后，只能在该项目中通过npm script的方式使用cz命令，过程比较繁琐，所以更好的方式是全局安装commitizen工具，通过`npm install -g commitizen`全局安装。同样的，需要在全局安装适配器：
```sh
npm install -g cz-conventional-changelog
```
适配器安装完成之后，需要添加该适配器：
```
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```
该命令会在用户家目录生成.czrc文件，其的内容与本地项目安装时在package.json中的基本一致，目的是告诉commitizen使用该适配器。
全局安装之后，我们就可以在任何的git仓库运行git cz来代替git commit，git commit所支持的所有参数git cz也都支持。如git cz -a。如果项目中有设置了不同的commitizen适配器，将覆盖全局的适配器。

## commitizen适配器
我们知道每个项目的构建过程可能是不同的，所遵循的commit规范也可能是不同的，或需要定制交互式命令行的问题和选项，为了支持不同的需求，commitizen可以使用不同的适配器来定制commitizen，内置的适配器列表[点击这里](https://github.com/commitizen/cz-cli#adapters)，如果这些适配器都不满足要求，开发者还可以开发自己的适配器。我们在本地项目体验下cz-emoji适配器，该适配器会在type中加入emoji表情，先安装该适配器：
```sh
npm install --save-dev cz-emoji
```
在package.json中修改要使用的适配器：
```json
"config": {
  "commitizen": {
    "path": "./node_modules/cz-emoji"
  }
}
```
安装完成之后，运行git cz -a可以看到提示的type种类更加丰富了，并且每种type使用emoji表情代替。

![emoji.gif](emoji.gif)

## 重试上次提交
在使用commitizen的过程中，可能会频繁遇到需要重试上次提交的场景，例如你已经使用commitizen填写了一些列的字段，完成一次提交，但是单元测试失败了，或者发现一个小的拼写错误，修改完成之后也不想再增加一次新的历史提交记，即你想撤回上次的提交。虽然你可以使用`git reset --soft <LAST_COMMIT_ID>`来做到这一点，但是你再次提交时，不得不重新填写一边相同的内容。
所以commitizen提供了更加简便的方式来完成这个过程，执行`git cz --retry`，即可直接利用上次已填写的字段重新生成一次相同的提交记录。使用该命令时，如果是本地项目内安装了commitizen，并通过npm脚本的形式运行的cz(比如npm run cm)，则需要执行`npm run commit -- --retry`，如果是通过npx，则执行`npx cz --retry`。

## 通过git hooks使用commmitzen
通过全局安装或者本地安装的方式，开发者已经可以使用`git cz`或者`cz`命令了，但项目一般都是多人协作开发的，为了强制所有人都能遵循commitizen的提交规范，又要强制所有的开发者都安装commitizen，对于那些不熟悉commitizen的人来说又要花时间熟悉这个工具，所以commitizen提供了另外一种方式来强制提交规范——`git hooks`。不熟悉git hooks的[点击这里](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90)。
我们需要修改.git/hooks目录下的脚本文件，如果之前没有修改过该目录下的任何文件，可以通过如下命令复制一份新的脚本文件：
```
cp prepare-commit-msg.sample prepare-commit-msg
```
将prepare-commit-msg中的代码更新成如下代码：

```
#!/bin/bash
exec < /dev/tty && node_modules/.bin/cz --hook || true
```
保存后回到项目根目录下，再执行`git commit`时就会出现commitizen一样的交互式提交流程，需要强调的是这种方式要求本地项目安装commitizen和comitizen适配器。

以上方式是使用原生的git hooks来实现git commit改造的，但是我们都知道[husky](https://typicode.github.io/husky/#/)库也可以实现这个过程。以husky库7.x版本为例演示这个过程，husky4之前的版本与最新版的的husky库的使用方式有较大差异，不过原理是相同的。通过以下命令安装并启用husky：
```sh
npx husky-init && npm install
```

接着需要向husky中添加新的git hooks，执行以下命令：
```sh
npx husky add .husky/prepare-commit-msg
```

打开.husky/prepare-commit-msg文件将文件更新成以下代码：
```sh
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

exec < /dev/tty && node_modules/.bin/cz --hook || true
```

保存后在项目根目录运行git commit也会使用commitizen工具。以上方式两种方式本质上都是通过git的prepare-commit-msg hooks的原理来实现的。

不过需要特别强调的是，在实际项目中并不推荐这以上两种方式中的任何一种，因为他会修改`git commit`的默认行为，我们应该保留默认的git命令，并在npm脚本中添加自定义的命令来实现我们的目的，并配合下文提交到的commitlint来强制提交记录的规范性。其次如果使用standard-version来自动生成changelog时，会影响自动提交的行为，后文会详细讲解。

## 使用commitlint校验提交记录
通过以上方式，基本可以杜绝混乱的提交历史，如果还不放心，还可以使用commitlint + husky对提交历史进行校验。安装过程也十分简单，先安装commitlint工具，其中@commitlint/config-conventional包是根据Angular提交规范预定义的规则包：
```sh
npm install --save-dev @commitlint/{config-conventional,cli}
```
安装完成之后，在项目根目录创建`.commitlintrc.js`文件，来配置lint规则：
```
module.exports = {
  extends: ['@commitlint/config-conventional']
};
```
配置完成之后在项目根目录下运行`./node_modules/.bin/commitlint --edit`来验证最近一次提交记录是否符合要求，如果符合要求不会输出任何内容，反之则会出现报错信息。该命令正常运行之后，就可以配合husky来自动校验提交信息。安装husky的过程的过程可以[参考这里](https://typicode.github.io/husky/#/?id=install)，不做演示。
为husky添加新的commit-msg脚本：
```sh
# -V参数在命令行输出校验信息
npx husky add .husky/commit-msg "npx --no -- commitlint -V --edit $1"
```

添加完成之后在运行git cz会先根据commitizen生成合规的提交记录，然后执行commitlint命令来校验提交信息。但是如果有人使用`git commit`书写了不规范的历史记录，commitlint会报错并阻止本次提交，这样就达到了强制规范提交记录的目的。
需要说明的是，这里的commitlint的规则配置非常简单，用的都是默认值，如果项目使用不同的commitizen适配器，例如上文提到的cz-emoji适配器，需要根据情况修改`.commitlintrc.js`文件中的规则。完整的规则手册[参考这里](https://github.com/conventional-changelog/commitlint/blob/master/docs/reference-rules.md)。
## 根据提交记录生成changelog
上面我们已经安装了很多关于git提交的工具，此时就可以根据这些有价值的提交记录来生成changelog了，这里我们利用[standard-veriosn](https://github.com/conventional-changelog/standard-version)来生成changelog，该工具是根据[Angular提交规范](https://www.conventionalcommits.org/zh-hans/v1.0.0/)来生成changelog。首先安装该工具：
```sh
npm i --save-dev standard-version
```
安装完成之后，在package.json中添加脚本：
```json
{
  "scripts": {
    "release": "standard-version"
  }
}
```
之后我们就可以通过npm脚本的形式来生成changelog，如果是第一次执行该命令，则执行：
```sh
npm run release -- --first-release
# 或者通过npx执行
npx andard-version --first-release
```
注意第一个--是传递参数给npm，--first-release是sdandard-version接收的参数。后续发布再次生成changelog时，则不需要添加--first-release参数：
```sh
npm run release
```
该命令帮我们做了如下几件事：
1. 根据[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)规范生成或更新CHANGELOG.md文件
2. 更新package.json文件中版本号。第一次运行不会更新版本号
3. 根据提交规范，为以上更新生成一次新的提交，提交的type为chore，如chore(release): 1.1.1
4. 为当前分支打上tag，tag的值是上一步获得的版本号，版本号遵循[语义化规范](https://semver.org/)，如v1.2.0

该工具根据Conventional Commits和语义化版本规范帮我们自动生成版本号，如果想指定生成的版本号，需要额外的参数，如预发布时运行以下命令，会生成如1.0.1-0这样的版本号：
```
# npm run script
npm run release -- --prerelease
```
需要发布alpha版本时，可以传递如下参数，会生成1.0.1-alpha.0这样的版本号：
```sh
# npm run script
npm run release -- --prerelease alpha
```
如果只想更新次版本号（minor）时（主版本号为major，补丁版本号为patch）：
```sh
npm run release -- --release-as minor
```
或者想指定新的版本号：
```sh
npm run release -- --release-as 1.1.0
```
具体所支持的参数列表[参考这里](https://github.com/conventional-changelog/standard-version#cli-usage)。

以上我们根据默认的规则生成了CHANGELOG.md文件，该工具也支持丰富的配置，具体的配置方法[参考这里](https://github.com/conventional-changelog/standard-version#configuration)。
这里我们参考[掘金上的一篇文章](https://juejin.cn/post/6934292467160514567#heading-14)来为不同类型的log添加emoji表情作为演示，在项目根目录创建`.versionrc.js`文件，文件内容如下：
```js
module.exports = {
  "types": [
    { "type": "feat", "section": "✨ Features | 新功能" },
    { "type": "fix", "section": "🐛 Bug Fixes | Bug 修复" },
    { "type": "init", "section": "🎉 Init | 初始化" },
    { "type": "docs", "section": "✏️ Documentation | 文档" },
    { "type": "style", "section": "💄 Styles | 风格" },
    { "type": "refactor", "section": "♻️ Code Refactoring | 代码重构" },
    { "type": "perf", "section": "⚡ Performance Improvements | 性能优化" },
    { "type": "test", "section": "✅ Tests | 测试" },
    { "type": "revert", "section": "⏪ Revert | 回退" },
    { "type": "build", "section": "📦‍ Build System | 打包构建" },
    { "type": "chore", "section": "🚀 Chore | 构建/工程依赖/工具" },
    { "type": "ci", "section": "👷 Continuous Integration | CI 配置" }
  ]
};
```
此时运行npm run release -- --first-release命令生成的CHANGELOG.md会根据提交的类型生成各种有趣的表情。

**需要说明的是，由于该工具会利用`git commit`命令帮我们自动生成一次新的符合规范的提交记录，所以我们想利用该工具自动生成changelog就不能通过上文提到的git hooks来安装commitizen。因为它会修改`git commit`命令的行为，强制使默认的提交命令变成可交互式的，进而导致standard-version命令运行失败。**

最后需要强调的是，该工具使用[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)提交规范，所以如果使用的commitizen适配器不遵循该规范，则不能顺利生成changelog。

## 总结
上面介绍了很多工具，我们没有必要全都使用，这里推荐使用以下方案组合：
1. 全局安装，在所有的git仓库使用git cz命令来进行提交
2. 本地项目安装，以方便配合其他工具
3. 使用cz-conventional-changelog适配器
4. 通过npm脚本添加自定义的commitizen命令，如`npm run cm`
5. 使用commitlint校验提交记录，防止不规范的记录
6. 使用standard-version自动化生成changelog
7. 为standard-version添加自定义的脚本，如standard-version

日常的提交先使用`git add .`，再使用`git cz`或`npm run cm`代替`git commit`，如果使用git commit命令提交必须书写符合规范的提交记录。发布时可以运行`npm run release`来生成新的changelog。

## 参考资料
- [https://juejin.cn/post/6934292467160514567](https://juejin.cn/post/6934292467160514567)
- [https://github.com/commitizen/cz-cli](https://github.com/commitizen/cz-cli)